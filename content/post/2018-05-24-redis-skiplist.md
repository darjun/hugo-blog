---
layout:     post
title:      "Redis源码阅读-skiplist"
subtitle:   "\"Learn from redis\""
date:       2018-05-24T09:07:00
author:     "darjun"
image: "/img/background1.jpg"
tags:
    - redis-source
URL: "/2018/05/24/redis-skiplist/"
categories: [ Tech ]
---

## <span id="概述">概述</span>
跳跃表是`zset`（有序集合）的基础数据结构。跳跃表可以高效地保持元素有序，并且实现相比平衡树简单、直观。Redis的跳跃表是基于William Pugh在[《Skip lists: a probabilistic alternative to balanced trees》][1]中描述的算法实现的。做了以下几点改动：
1. 允许重复分数。
2. 比较不仅会涉及键，还可能涉及节点数据（键相等时）。
3. 有一个后退指针，所以是一个双向链表。便于实现`ZREVRANGE`等命令。

## <span id="实现结构">实现结构</span>
实现在`redis.h`和`t_zset.c`两个文件中。

跳跃表的基本结构如下（来自William Pugh的论文）：
![skiplist结构](/img/in-post/redis-skiplist/skiplist-structure.png)

* 表头：维护跳跃表各层的指针。
* 中间节点：维护节点数据和各层的前进后退指针。
* 层：保存指向该层下一个节点的指针和与下个节点的间隔（span）。为了提高查找效率，程序总是先从高层开始访问，然后随着范围的缩小慢慢降低层次。
* 表尾：全部为NULL，表示跳跃表各层的末尾。

跳跃表的一个节点用结构`zskiplistNode`表示：
```
typedef struct zskiplistNode {
    // redis中通用对象，用来保存节点数据
    robj *obj;
    // 分数
    double score;
    // 后退指针，只有第0层有效
    struct zskiplistNode *backward;
    // 各层的前进指针及与下一个节点的间隔
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplist;
```

跳跃表用结构`zskiplist`表示：
```
typedef struct zskiplist {
    // 跳跃表的头部和尾部
    struct zskiplistNode *header, *tail;
    // 跳跃表的长度
    unsigned long length;
    // 层高
    int level;
} zskiplist;
```

## <span id="跳跃表操作">3.跳跃表操作</span>
### <span id="创建跳跃表">3.1.创建跳跃表</span>
调用`zslCreate`创建一个空的跳跃表。
```
// 创建一个层高为level的跳跃表节点，以score为分数，obj为数据
zskiplistNode *zslCreateNode(int level, double score, robj *obj) {
    zskiplistNode *zn = zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->obj = obj;
    return zn;
}

zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    // 分配空间
    zsl = zmalloc(sizeof(*zsl));
    // 层高初始化为1，插入数据时可能会增加
    zsl->level = 1;
    // 长度为0
    zsl->length = 0;
    // 分配头部节点，层高ZSKIPLIST_MAXLEVEL（当前值为32，对于2^32个元素足够了）
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);
    // 初始化头部各层的指针和间隔
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

### <span id="销毁跳跃表">3.2.销毁跳跃表</span>
跳跃表结构使用完后，需要调用`zslFree`来释放。
```
void zslFreeNode(zskiplistNode *node) {
    // obj是redis对象，使用引用计数管理的
    decrRefCount(node->obj);
    zfree(node);
}

void zslFree(zskiplist *zsl) {
    // 跟踪第0层（最下面一层）的forward指针可以遍历所有元素
    zskiplistNode *node = zsl->header->level[0].forward, *next;

    // 释放头部节点
    zfree(zsl->header);
    while(node) {
        next = node->level[0].forward;
        // 释放当前节点
        zslFreeNode(node);
        node = next;
    }
    // 释放zskiplist结构
    zfree(zsl);
}
```

### <span id="插入节点">3.3.插入节点</span>
向跳跃表中插入节点时，以Redis对象`obj`和与之关联的分数`score`调用`zslInsert`。

```
zskiplistNode *zslInsert(zskiplist *zsl, double score, robj *obj) {
    // update记录的是各层新节点插入位置的前一个节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    // 分数不能是NaN
    redisAssert(!isnan(score));
    x = zsl->header;
    // 从最高层向下处理，每次遇到使score和obj介于其中的x和x->forward，则新节点必定在x和x->forward之间
    // 到下一层更精准的定位插入位置。下一层继续从位置x前进查找，因为由上一层得知新节点必然在x之后。
    // 记录节点x在跳跃表中的rank。
    for (i = zsl->level-1; i >= 0; i--) {
        // x节点的rank由上一层计算得到
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        // score相同时，比较两个obj
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0))) {
            // 计算rank
            rank[i] += x->level[i].span;
            // 移向下一个继续判断
            x = x->level[i].forward;
        }
        // 新节点要插入x与x->level[i]->forward之间，x->level[i]需要做相应调整（span，forward等字段）
        update[i] = x;
    }

    // 因为允许重复score，所以这里并没有做重复判断。应该由调用者测试新插入的对象是否已经存在。

    // 随机新节点层高
    level = zslRandomLevel();
    if (level > zsl->level) {
        // 新节点比当前层高大
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            // 没有节点，span是整个跳跃表的长度
            update[i]->level[i].span = zsl->length;
        }
    }
    // 创建跳跃表节点
    x = zslCreateNode(level,score,obj);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
        // 更新span
        // (rank[0] - rank[i])是第i层中新节点与前一个节点的距离（严格说应该是第0层新节点的前一个节点与第i层新节点的前一个节点间距离）
        // 新节点将update[i]后面的间隔分成两部分。
        x->level[i].span = update[i]->level[i].span - （rank[0] - rank[i]);
        // +1算上新节点
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 未更新的层需要增加新节点之前的节点span，因为后面多了一个新节点
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 更新后退节点
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```

随机层高算法：
```
int zslRandomLevel(void) {
    int level = 1;
    // level每次循环有1/4概率增加层高
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

### <span id="范围">3.4.范围</span>
#### <span id="数值范围">3.4.1.数值范围</span>
`zslValueGteMin`判断`value`是否大于等于（由`minex`决定是否取等于）范围`spec`中的最小值：
```
static int zslValueGteMin(double value, zrangespec *spec) {
    return spec->minex ? (value > spec->min) : (value >= spec->min);
}
```

`zslValueGteMax`判断`value`是否小于等于（由`maxex`决定是否取等于）范围`spec`中的最大值：
```
static int zslValueLteMax(double value, zrangespec *spec) {
    return spec->maxex ? (value < spec->max) : (value <= spec->max);
}
```

`zslIsInRange`判断跳跃表中是否有节点处于范围`range`中。这里有个前提，所有节点的`score`都相等。
```
int zslIsInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;

    // 范围大小为0
    if (range->min > range->max || 
            (range->min == range->max && (range->minex || range->maxex)))
        return 0;
    x = zsl->tail;
    // 跳跃表中最大值不大于范围range的最小值
    if (x == NULL || !zslValueGteMin(x->score,range))
        return 0;
    // 跳跃表中最小值不小于范围range的最大值
    x = zsl->header->level[0].forward;
    if (x == NULL || !zslValueLteMax(x->score,range))
        return 0;
    return 1;
}
```

`zslFirstInRange`返回跳跃表中处于范围`range`中的第一个节点：
```
zskiplistNode *zslFirstInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;
    int i;

    // 跳跃表中没有节点处于范围range内
    if (!zslIsInRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 找到最后一个小于range最小值的节点
        while (x->level[0].forward &&
            !zslValueGteMin(x->level[i].forward->score,range))
                x = x->level[i].forward;
    }

    // x为小于range最小值的最大节点，x->level[0].forward为大于range最小值的最小节点
    x = x->level[0].forward;
    redisAssert(x != NULL);

    if (!zslValueLteMax(x->score,range)) return NULL;
    return x;
}
```

`zslLastInRange`返回跳跃表中处于范围`range`中的最后一个节点：
```
zskiplistNode *zslLastInRange(zskiplist *zsl, zrangespec *range) {
    zskiplistNode *x;
    int i;

    // 跳跃表中没有节点处于范围range内
    if (!zslIsInRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 找到小于等于range最大值的最后一个节点
        while (x->level[0].forward &&
            !zslValueLteMax(x->level[i].forward->score,range))
                x = x->level[i].forward;
    }

    redisAssert(x != NULL);

    if (!zslValueGteMin(x->score,range)) return NULL;
    return x;
}
```

#### <span id="字典序范围">3.4.2.字典序范围</span>
`zslLexValueGteMin`判断`value`是否大于等于（由`minex`决定是否取等于）范围`spec`的最小值。通过字典序来比较。
```
static int zslLexValueGteMin(robj *value, zlexrangespec *spec) {
    return spec->minex ?
        (compareStringObjectsForLexRange(value,spec->min) > 0) :
        (compareStringObjectsForLexRange(value,spec->min) >= 0);
}
```

这里使用`compareStringObjectsForLexRange`来比较字典序：
```
int compareStringObjectsForLexRange(robj *a, robj *b) {
    if (a == b) return 0;

    // minstring和maxstring是Redis预定义的用于表示最小和最大字符串的
    if (a == shared.minstring || b == shared.maxstring) return -1;
    if (a == shared.maxstring || b == shared.minstring) return 1;
    return compareStringObjects(a,b);
}
```

`zslLexValueLteMax`判断`value`是否小于等于（由`maxex`决定是否取等于）范围`spec`的最大值。通过字典序来比较。
```
static int zslLexValueLteMax(robj *value, zlexrangespec *spec) {
    return spec->maxex ?
        (compareStringObjectsForLexRange(value,spec->max) < 0) :
        (compareStringObjectsForLexRange(value,spec->max) <= 0);
}
```

`zslIsInLexRange`判断跳跃表中是否有节点的数据`obj`处于范围`range`中。这里有个前提，所有节点的`score`都相等。
```
int zslIsInLexRange(zskiplist *zsl, zlexrangespec *range) {
    zskiplistNode *x;

    // 范围为空
    if (compareStirngObjectsForLexRange(range->min,range->max) > 1 ||
            (compareStringObjects(range->min,range->max) == 0 &&
            (range->minex || range->maxex)))
        return 0;
    x = zsl->tail;
    // 跳跃表最大值小于范围range的最小值
    if (x == NULL || !zslLexValueGteMin(x->obj,range))
        return 0;
    x = zsl->header->level[0].forward;
    // 跳跃表最小值大于范围range的最大值
    if (x == NULL || !zslLexValueLteMax(x->obj,range))
        return 0;
    return 1;
}
```

`zslFirstInLexRange`返回跳跃表处于范围`range`中的第一个节点：
```
zskiplistNode *zslFirstInLexRange(zskiplist *zsl, zlexrangespec *spec) {
    zskiplistNode *x;
    int i;

    // 跳跃表没有处于范围range中的节点
    if (!zslIsInLexRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 找到最后一个小于range最小值的节点
        while (x->level[i].forward &&
            !zslLexValueGteMin(x->level[i].forward->obj,range))
                x = x->level[i].forward;
    }

    // x为最后一个小于range最小值的节点，x->level[0].forward为第一个大于range最小值的节点
    x = x->level[0].forward;
    redisAssert(x != NULL);

    // x是第一个大于range最小值的节点，必须满足小于range的最大值
    if (!zslLexValueLteMax(x->obj,range)) return NULL;
    return x;
}
```

`zslLastInLexRange`返回跳跃表处于范围`range`中的最后一个节点：
```
zskiplistNode *zslLastInLexRange(zskiplist *zsl, zlexrangespec *range) {
    zskiplistNode *x;
    int i;

    // 跳跃表没有处于范围range中的节点
    if (!zslIsInLexRange(zsl,range)) return NULL;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        // 找到小于等于range最大值的最后一个节点
        while (x->level[i].forward &&
            zslLexValueLteMax(x->level[i].forward->obj,range))
                x = x->level[i].forward;
    }

    redisAssert(x != NULL);

    // x是小于range最大值的最后一个节点，必须满足大于range最小值这个限制
    if (!zslLexValueGteMin(x->obj,range)) return NULL;
    return x;
}
```

### <span id="删除节点">3.5.删除节点</span>
Redis中实现了跳跃表的多种删除方式。下面依次介绍：

#### <span id="删除指定score和obj的节点">3.5.1.删除指定score和obj的节点</span>
这是一种最基本的删除方式，由分数`score`和对象`obj`指定需要删除的节点。调用`zslDelete`完成这个任务。

```
int zslDelete(zskiplist *zsl, double score, robj *obj) {
    // 各层需要修改的节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    // 与zslInsert操作一致
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0)))
            x = x->level[i].forward;
        update[i] = x;
    }

    // 可能有多个节点拥有相同score，所以这里需要比较对象
    x = x->level[0].forward;
    if (x && x->score == score && equalStringObjects(x->obj,obj)) {
        // 从跳跃表中删除该节点
        zslDeleteNode(zsl, x, update);
        // 释放节点
        zslFreeNode(x);
        return 1;
    }
    // 没找到
    return 0;
}
```

`zslDelete`调用通用的节点删除函数`zslDeleteNode`，`zslDeleteNode`被其他几个删除函数使用。
```
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    for (i = 0; i < zsl->level; i++) {
        if (update[i]->level[i].forward == x) {
            // 待删除的节点在该层存在，调整forward指针和span
            // 两个间隔合并为一个，同时减少了一个节点
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            // 该层没有这个节点，只需要减少span
            update[i]->level[i].span -= 1;
        }
    }
    if (x->level[0].forward) {
        // 更新前进节点的后退指针
        x->level[0].forward->backward = x->backward;
    } else {
        // x是跳跃表尾部
        zsl->tail = x->backward;
    }
    // 如果某一层只有该节点，该节点删除后已经没有节点了，直接移除这一层。但是跳跃表至少有一层。
    while (zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL) {
        zsl->level--;
    }
    zsl->length--;
}
```

#### <span id="删除一个score范围内的节点">3.5.2.删除一个score范围内的节点</span>
删除分数介于`min`和`max`之间的所有节点。函数`zslDeleteRangeByScore`完成这个功能。范围由`zrangespec`指定：
```
typedef struct {
    // 最小最大值
    double min, max;
    // 是否包括最小值/最大值
    int minex, maxex;
} zrangespec;
```

```
// 删除的节点也需要从dict中删除
unsigned long zslDeleteRangeByScore(zskiplist *zsl, zrangespec *range, dict *dict) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned long removed = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (range->minex ?
            x->level[i].forward->score <= range->min :
            x->level[i].forward->score < range->min))
                x = x->level[i].forward;
        update[i] = x;
    }

    // x是最后一个 < 或 <= min的节点了。
    x = x->level[0].forward;

    // 依次删除每个处于范围内的节点。因为是依次向后删除，所以update都是适用的。
    while (x &&
           (range->maxex ? x->score < range->max : x->score <= range->max))
    {
        zskiplistNode *next = x->level[0].forward;
        // 使用通用节点删除函数zslDeleteNode
        zslDeleteNode(zsl,x,update);
        // 从dict中删除这个对象
        dictDelete(dict,x->obj);
        zslFreeNode(x);
        removed++;
        x = next;
    }
    return removed;
}
```

#### <span id="删除一个字典序范围内的节点">3.5.2.删除一个字典序范围内的节点</span>
函数`zslDeleteRangeByLex`删除以字典序表示的范围内所有节点。这个方法有个前提条件，**跳跃表中的所有score都是相同的**。在概述中提到过，如果`score`相同，会比较`obj`。

字典序范围用结构`zlexrangespec`表示：
```
typedef struct {
    // 最小最大值
    robj *min, *max;
    // 是否包括最小/大值
    int minex, maxex;
} zlexrangespec;
```

```
unsigned long zslDeleteRangeByLex(zskiplist *zsl, zlexrangespec *range, dict *dict) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned long removed = 0;
    int i;

    x = zsl->header;
    // 除了使用zslLexValueGteMin比较，其他与函数zslDeleteRangeByScore基本相同
    // zslLexValueGteMin判断obj是否大于等于zlexrangespec的最小值
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            !zslLexValueGteMin(x->level[i].forward->obj,range))
                x = x->level[i].forward;
        update[i] = x;
    }

    // x是最后一个 < 或 <= min的节点了
    x = x->level[0].forward;

    while (x && zslLexValueLteMax(x->obj,range)) {
        zskiplistNode *next = x->level[0].forward;
        // 使用通用节点删除函数zslDeleteNode
        zslDeleteNode(zsl,x,update);
        // 从dict中删除这个对象
        dictDelete(dict,x->obj);
        zslFreeNode(x);
        removed++;
        x = next;
    }
    return removed;
}
```

#### <span id="删除一个rank范围内的节点">3.5.4.删除一个rank范围内的节点</span>
`zslDeleteRangeByRank`可以删除一个rank范围内的所有节点。
```
unsigned long zslDeleteRangeByRank(zskiplist *zsl, unsigned int start, unsigned int end, dict *dict) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned long traversed = 0, removed = 0;
    int i = 0;

    x = zsl->header;
    // 排行直接通过span判断即可
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (traversed + x->level[i].span) < start) {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    traversed++;
    // x->level[0].forward是rank为start的节点（因为上面比较是 <start）
    x = x->level[0].forward;
    // 删除节点之后span会改变，使用累计的traversed与end判断
    while (x && traversed <= end) {
        zskiplistNode *next = x->level[0].forward;
        // 使用通用节点删除函数zslDeleteNode
        zslDeleteNode(zsl,x,update);
        // 从dict中删除这个对象
        dictDelete(dict,x->obj);
        zslFreeNode(x);
        removed++;
        traversed++;
        x = next;
    }
    return removed;
}
```

### <span id="查询">3.6.查询</span>
#### <span id="查找score和obj的rank">3.6.1.查找score和obj的rank</span>
查找跳跃表中由`score`和`obj`指定的节点的rank。rank从1开始，没有该节点时返回0。
```
unsigned long zslGetRank(zskiplist *zsl, double score, robj *o) {
    zskiplistNode *x;
    unsigned long rank = 0;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                 compareStringObjects(x->level[i].forward->obj,o) < 0))) {
            rank += x->level[i].span;
            x = x->level[i].forward;
        }

        // x可能等于表头
        if (x->obj && equalStringObjects(x->obj,o)) {
            return rank;
        }
    }
    return 0;
}
```

#### <span id="查找某个rank的节点">3.6.2.查找某个rank的节点</span>
函数`zslGetElementByRank`查找某个rank的节点。
```
zskiplistNode *zslGetElementByRank(zskiplist *zsl, unsigned long rank) {
    zskiplistNode *x;
    unsigned long traversed = 0;
    int i = 0;

    x = zsl->header;
    // rank可以使用span计算
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward && (traversed + x->level[i].span) <= rank) {
            traversed += x->level[i].span;
            x = x->level[i].forward;
        }
        if (traversed == rank) {
            return x;
        }
    }
    return NULL;
}
```

## <span id="总结">4.总结</span>
跳跃表实现简单，插入、查找、删除等操作性能都不错。Redis基于William Pugh的算法按照自己的需求做了一些扩展，如允许相同分数、增加后退指针等。

[1]: https://www.epaperpress.com/sortsearch/download/skiplist.pdf