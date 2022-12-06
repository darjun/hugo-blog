---
layout:    post
title:    "一起用Go来做一个游戏(中)"
subtitle: "每天学习一个 Go 库"
date:     2022-11-18T22:39:30
author:   "darjun"
image:    "img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2022/11/18/godailylib/ebiten2"
categories: [
  "Go"
]
---

## 限制飞船的活动范围

上一篇文章还留了个尾巴，细心的同学应该发现了：飞船可以移动出屏幕！！！现在我们就来限制一下飞船的移动范围。我们规定飞船可以左右超过半个身位，如下图所示：

![](/img/in-post/godailylib/ebiten12.png#center)

很容易计算得出，左边位置的x坐标为：

```golang
x = -W2/2
```

右边位置的坐标为：

```golang
x = W1 - W2/2
```

修改input.go的代码如下：

```golang
func (i *Input) Update(ship *Ship, cfg *Config) {
  if ebiten.IsKeyPressed(ebiten.KeyLeft) {
    ship.x -= cfg.ShipSpeedFactor
    if ship.x < -float64(ship.width)/2 {
      ship.x = -float64(ship.width) / 2
    }
  } else if ebiten.IsKeyPressed(ebiten.KeyRight) {
    ship.x += cfg.ShipSpeedFactor
    if ship.x > float64(cfg.ScreenWidth)-float64(ship.width)/2 {
      ship.x = float64(cfg.ScreenWidth) - float64(ship.width)/2
    }
  }
}
```

运行结果如下：

![](/img/in-post/godailylib/ebiten13.gif#center)

## 发射子弹

我们不用另外准备子弹的图片，直接画一个矩形就ok。为了可以灵活控制，我们将子弹的宽、高、颜色以及速率都用配置文件来控制：

```json
{
  "bulletWidth": 3,
  "bulletHeight": 15,
  "bulletSpeedFactor": 2,
  "bulletColor": {
    "r": 80,
    "g": 80,
    "b": 80,
    "a": 255
  }
}
```

新增一个文件bullet.go，定义子弹的结构类型和New方法：

```golang
type Bullet struct {
  image       *ebiten.Image
  width       int
  height      int
  x           float64
  y           float64
  speedFactor float64
}

func NewBullet(cfg *Config, ship *Ship) *Bullet {
  rect := image.Rect(0, 0, cfg.BulletWidth, cfg.BulletHeight)
  img := ebiten.NewImageWithOptions(rect, nil)
  img.Fill(cfg.BulletColor)

  return &Bullet{
    image:       img,
    width:       cfg.BulletWidth,
    height:      cfg.BulletHeight,
    x:           ship.x + float64(ship.width-cfg.BulletWidth)/2,
    y:           float64(cfg.ScreenHeight - ship.height - cfg.BulletHeight),
    speedFactor: cfg.BulletSpeedFactor,
  }
}
```

首先根据配置的宽高创建一个rect对象，然后调用`ebiten.NewImageWithOptions`创建一个`*ebiten.Image`对象。

子弹都是从飞船头部发出的，所以它的横坐标等于飞船中心的横坐标，左上角的纵坐标=屏幕高度-飞船高-子弹高。

随便增加子弹的绘制方法：

```golang
func (bullet *Bullet) Draw(screen *ebiten.Image) {
  op := &ebiten.DrawImageOptions{}
  op.GeoM.Translate(bullet.x, bullet.y)
  screen.DrawImage(bullet.image, op)
}
```

我们在Game对象中增加一个map来管理子弹：

```golang
type Game struct {
  // -------省略-------
  bullets map[*Bullet]struct{}
}

func NewGame() *Game {
  return &Game{
    // -------省略-------
    bullets: make(map[*Bullet]struct{}),
  }
}
```

然后在`Draw`方法中，我们需要将子弹也绘制出来：

```golang
func (g *Game) Draw(screen *ebiten.Image) {
  screen.Fill(g.cfg.BgColor)
  g.ship.Draw(screen)
  for bullet := range g.bullets {
    bullet.Draw(screen)
  }
}
```

子弹位置如何更新呢？在`Game.Update`中更新，与飞船类似，只是飞船只能水平移动，而子弹只能垂直移动。

```golang
func (g *Game) Update() error {
  for bullet := range g.bullets {
    bullet.y -= bullet.speedFactor
  }
  // -------省略-------
}
```

子弹的更新、绘制逻辑都完成了，可是我们还没有子弹！现在我们就来实现按空格发射子弹的功能。我们需要在`Input.Update`方法中判断空格键是否按下，由于该方法需要访问Game对象的多个字段，干脆传入Game对象：

```golang
func (i *Input) Update(g *Game) {
  if ebiten.IsKeyPressed(ebiten.KeyLeft) {
    // -------省略-------
  } else if ebiten.IsKeyPressed(ebiten.KeyRight) {
    // -------省略-------
  } else if ebiten.IsKeyPressed(ebiten.KeySpace) {
    bullet := NewBullet(g.cfg, g.ship)
    g.addBullet(bullet)
  }
}
```

给Game对象增加一个`addBullet`方法：

```golang
func (g *Game) addBullet(bullet *Bullet) {
  g.bullets[bullet] = struct{}{}
}
```

目前有两个问题：

* 无法一边移动一边发射，仔细看看`Input.Update`方法中的代码，你能发现什么问题吗？
* 子弹太多了，我们想要限制子弹的数量。

下面来逐一解决这些问题。

第一个问题很好解决，因为在KeyLeft/KeyRight/KeySpace这三个判断中我们用了if-else。这样会优先处理移动的操作。将KeySpace移到一个单独的if中即可：

```golang
func (i *Input) Update(g *Game) {
  // -------省略-------
  if ebiten.IsKeyPressed(ebiten.KeySpace) {
    bullet := NewBullet(g.cfg, g.ship)
    g.addBullet(bullet)
  }
}
```

第二个问题，在配置中，增加同时最多存在的子弹数：

```json
{
  "maxBulletNum": 10
}
```

```golang
type Config struct {
  MaxBulletNum      int        `json:"maxBulletNum"`
}
```

然后我们在`Input.Update`方法中判断，如果目前存在的子弹数小于`MaxBulletNum`才能创建新的子弹：

```golang
if ebiten.IsKeyPressed(ebiten.KeySpace) {
  if len(g.bullets) < cfg.MaxBulletNum {
    bullet := NewBullet(g.cfg, g.ship)
    g.addBullet(bullet)
  }
}
```

再次运行：

![](/img/in-post/godailylib/ebiten15.gif#center)

数量好像被限制了，但是不是我们配置的10。原来`Input.Update()`的调用间隔太短了，导致我们一次space按键会发射多个子弹。我们可以控制两个子弹之间的时间间隔。同样用配置文件来控制（单位毫秒）：

```json
{
  "bulletInterval": 50
}
```

```golang
type Config struct {
  BulletInterval    int64      `json:"bulletInterval"`
}
```

在`Input`结构中增加一个`lastBulletTime`字段记录上次发射子弹的时间：

```golang
type Input struct {
  lastBulletTime time.Time
}
```

距离上次发射子弹的时间大于`BulletInterval`毫秒，才能再次发射，发射成功之后更新时间：

```golang
func (i *Input) Update(g *Game) {
  // -------省略-------
  if ebiten.IsKeyPressed(ebiten.KeySpace) {
    if len(g.bullets) < g.cfg.MaxBulletNum &&
      time.Now().Sub(i.lastBulletTime).Milliseconds() > g.cfg.BulletInterval {
      bullet := NewBullet(g.cfg, g.ship)
      g.addBullet(bullet)
      i.lastBulletTime = time.Now()
    }
  }
}
```

运行：

![](/img/in-post/godailylib/ebiten16.gif#center)

又出现了一个问题，10个子弹飞出屏幕外之后还是不能发射子弹。我们需要把离开屏幕的子弹删除。这适合在`Game.Update`函数中做：

```golang
func (g *Game) Update() error {
  g.input.Update(g)
  for bullet := range g.bullets {
    if bullet.outOfScreen() {
      delete(g.bullets, bullet)
    }
  }
  return nil
}
```

为了`Bullet`添加判断是否处于屏幕外的方法：

```golang
func (bullet *Bullet) outOfScreen() bool {
  return bullet.y < -float64(bullet.height)
}
```

再次运行：

![](/img/in-post/godailylib/ebiten17.gif#center)

## 外星人来了

外星人图片如下：

![](/img/in-post/godailylib/alien.png#center)

同飞船一样，编写Alien类，添加绘制自己的方法：

```golang
type Alien struct {
  image       *ebiten.Image
  width       int
  height      int
  x           float64
  y           float64
  speedFactor float64
}

func NewAlien(cfg *Config) *Alien {
  img, _, err := ebitenutil.NewImageFromFile("../images/alien.png")
  if err != nil {
    log.Fatal(err)
  }

  width, height := img.Size()
  return &Alien{
    image:       img,
    width:       width,
    height:      height,
    x:           0,
    y:           0,
    speedFactor: cfg.AlienSpeedFactor,
  }
}

func (alien *Alien) Draw(screen *ebiten.Image) {
  op := &ebiten.DrawImageOptions{}
  op.GeoM.Translate(alien.x, alien.y)
  screen.DrawImage(alien.image, op)
}
```

游戏开始时需要创建一组外星人，计算一行可以容纳多少个外星人，考虑到左右各留一定的空间，两个外星人之间留一点空间。在游戏一开始就创建一组外星人：

```golang
type Game struct {
  // Game结构中的map用来存储外星人对象
  aliens  map[*Alien]struct{}
}

func NewGame() *Game {
  g := &Game{
    // 创建map
    aliens:  make(map[*Alien]struct{}),
  }
  // 调用 CreateAliens 创建一组外星人
  g.CreateAliens()
  return g
}

func (g *Game) CreateAliens() {
  alien := NewAlien(g.cfg)

  availableSpaceX := g.cfg.ScreenWidth - 2*alien.width
  numAliens := availableSpaceX / (2 * alien.width)

  for i := 0; i < numAliens; i++ {
    alien = NewAlien(g.cfg)
    alien.x = float64(alien.width + 2*alien.width*i)
    g.addAlien(alien)
  }
}
```

左右各留一个外星人宽度的空间：

```golang
availableSpaceX := g.cfg.ScreenWidth - 2*alien.width
```

然后，两个外星人之间留一个外星人宽度的空间。所以一行可以创建的外星人的数量为：

```golang
numAliens := availableSpaceX / (2 * alien.width)
```

创建一组外星人，依次排列。

同样地，我们需要在`Game.Draw`方法中添加绘制外星人的代码：

```golang
func (g *Game) Draw(screen *ebiten.Image) {
  // -------省略-------
  for alien := range g.aliens {
    alien.Draw(screen)
  }
}
```

运行：

![](/img/in-post/godailylib/ebiten18.png#center)

再创建两行：

```golang
func (g *Game) CreateAliens() {
  // -------省略-------
  for row := 0; row < 2; row++ {
    for i := 0; i < numAliens; i++ {
      alien = NewAlien(g.cfg)
      alien.x = float64(alien.width + 2*alien.width*i)
      alien.y = float64(alien.height*row) * 1.5
      g.addAlien(alien)
    }
  }
}
```

让外星人都起来，同样地还是在`Game.Update`方法中更新位置：

```golang
func (g *Game) Update() error {
  // -------省略-------
  for alien := range g.aliens {
    alien.y += alien.speedFactor
  }
  // -------省略-------
}
```

![](/img/in-post/godailylib/ebiten19.gif#center#center)

## 射击！！

当前子弹碰到外星人直接穿过去了，我们希望能击杀外星人。这需要检查子弹和外星人之间的碰撞。我们新增一个文件collision.go，并编写检查子弹与外星人是否碰撞的函数。这里我采用最直观的碰撞检测方法，即子弹的4个顶点只要有一个位于外星人矩形中，就认为它们碰撞。

```golang
// CheckCollision 检查子弹和外星人之间是否有碰撞
func CheckCollision(bullet *Bullet, alien *Alien) bool {
  alienTop, alienLeft := alien.y, alien.x
  alienBottom, alienRight := alien.y+float64(alien.height), alien.x+float64(alien.width)
  // 左上角
  x, y := bullet.x, bullet.y
  if y > alienTop && y < alienBottom && x > alienLeft && x < alienRight {
    return true
  }

  // 右上角
  x, y = bullet.x+float64(bullet.width), bullet.y
  if y > alienTop && y < alienBottom && x > alienLeft && x < alienRight {
    return true
  }

  // 左下角
  x, y = bullet.x, bullet.y+float64(bullet.height)
  if y > alienTop && y < alienBottom && x > alienLeft && x < alienRight {
    return true
  }

  // 右下角
  x, y = bullet.x+float64(bullet.width), bullet.y+float64(bullet.height)
  if y > alienTop && y < alienBottom && x > alienLeft && x < alienRight {
    return true
  }

  return false
}
```

接着我们在`Game.Update`方法中调用这个方法，并且将碰撞的子弹和外星人删除。

```golang
func (g *Game) CheckCollision() {
  for alien := range g.aliens {
    for bullet := range g.bullets {
      if CheckCollision(bullet, alien) {
        delete(g.aliens, alien)
        delete(g.bullets, bullet)
      }
    }
  }
}

func (g *Game) Update() error {
  // -------省略-------

  g.CheckCollision()

  // -------省略-------
  return nil
}
```

注意将碰撞检测放在位置更新之后。运行：

![](/img/in-post/godailylib/ebiten20.gif#center#center)

## 增加主界面和结束界面

现在一旦运行程序，外星人们就开始运动了。我们想要增加一个按下空格键才开始的功能，并且游戏结束之后，我们也希望能显示一个Game Over的界面。首先，我们定义几个常量，表示游戏当前所处的状态：

```golang
const (
  ModeTitle Mode = iota
  ModeGame
  ModeOver
)
```

Game结构中需要增加mode字段表示当前游戏所处的状态：

```golang
type Game struct {
  mode    Mode
  // ...
}
```

为了显示开始界面，涉及到文字的显示，文字显示和字体处理起来都比较麻烦。ebitengine内置了一些字体，我们可以据此创建几个字体对象：

```golang
var (
  titleArcadeFont font.Face
  arcadeFont      font.Face
  smallArcadeFont font.Face
)

func (g *Game) CreateFonts() {
  tt, err := opentype.Parse(fonts.PressStart2P_ttf)
  if err != nil {
    log.Fatal(err)
  }
  const dpi = 72
  titleArcadeFont, err = opentype.NewFace(tt, &opentype.FaceOptions{
    Size:    float64(g.cfg.TitleFontSize),
    DPI:     dpi,
    Hinting: font.HintingFull,
  })
  if err != nil {
    log.Fatal(err)
  }
  arcadeFont, err = opentype.NewFace(tt, &opentype.FaceOptions{
    Size:    float64(g.cfg.FontSize),
    DPI:     dpi,
    Hinting: font.HintingFull,
  })
  if err != nil {
    log.Fatal(err)
  }
  smallArcadeFont, err = opentype.NewFace(tt, &opentype.FaceOptions{
    Size:    float64(g.cfg.SmallFontSize),
    DPI:     dpi,
    Hinting: font.HintingFull,
  })
  if err != nil {
    log.Fatal(err)
  }
}
```

`fonts.PressStart2P_ttf`就是ebitengine提供的字体。创建字体的方法一般在需要的时候微调即可。将创建外星人和字体封装在Game的init方法中：

```golang
func (g *Game) init() {
  g.CreateAliens()
  g.CreateFonts()
}

func NewGame() *Game {
  // ...
  g.init()
  return g
}
```

启动时游戏处于ModeTitle状态，处于ModeTitle和ModeOver时只需要在屏幕上显示一些文字即可。只有在ModeGame状态才需要显示飞船和外星人：

```golang
func (g *Game) Draw(screen *ebiten.Image) {
  screen.Fill(g.cfg.BgColor)

  var titleTexts []string
  var texts []string
  switch g.mode {
  case ModeTitle:
    titleTexts = []string{"ALIEN INVASION"}
    texts = []string{"", "", "", "", "", "", "", "PRESS SPACE KEY", "", "OR LEFT MOUSE"}
  case ModeGame:
    g.ship.Draw(screen)
    for bullet := range g.bullets {
      bullet.Draw(screen)
    }
    for alien := range g.aliens {
      alien.Draw(screen)
    }
  case ModeOver:
    texts = []string{"", "GAME OVER!"}
  }

  for i, l := range titleTexts {
    x := (g.cfg.ScreenWidth - len(l)*g.cfg.TitleFontSize) / 2
    text.Draw(screen, l, titleArcadeFont, x, (i+4)*g.cfg.TitleFontSize, color.White)
  }
  for i, l := range texts {
    x := (g.cfg.ScreenWidth - len(l)*g.cfg.FontSize) / 2
    text.Draw(screen, l, arcadeFont, x, (i+4)*g.cfg.FontSize, color.White)
  }
}
```

在`Game.Update`方法中，我们判断在ModeTitle状态下按下空格，鼠标左键游戏开始，切换为ModeGame状态。游戏结束时切换为GameOver状态，在GameOver状态后按下空格或鼠标左键即重新开始游戏。

```golang
func (g *Game) Update() error {
  switch g.mode {
  case ModeTitle:
    if g.input.IsKeyPressed() {
      g.mode = ModeGame
    }
  case ModeGame:
    for bullet := range g.bullets {
      bullet.y -= bullet.speedFactor
    }

    for alien := range g.aliens {
      alien.y += alien.speedFactor
    }

    g.input.Update(g)

    g.CheckCollision()

    for bullet := range g.bullets {
      if bullet.outOfScreen() {
        delete(g.bullets, bullet)
      }
    }
  case ModeOver:
    if g.input.IsKeyPressed() {
      g.init()
      g.mode = ModeTitle
    }
  }

  return nil
}
```

input.go中增加`IsKeyPressed`方法判断是否有空格或鼠标左键按下。

## 判断游戏胜负

我们规定如果击杀所有外星人则游戏胜利，有3个外星人移出屏幕外或者碰撞到飞船则游戏失败。

首先增加一个字段`failCount`用于记录移出屏幕外的外星人数量和与飞船碰撞的外星人数量之和：
```golang
type Game struct {
  // -------省略-------
  failCount int // 被外星人碰撞和移出屏幕的外星人数量之和
}
```

然后我们在`Game.Update`方法中检测外星人是否移出屏幕，以及是否与飞船碰撞：

```golang
for alien := range g.aliens {
  if alien.outOfScreen(g.cfg) {
    g.failCount++
    delete(g.aliens, alien)
    continue
  }

  if CheckCollision(alien, g.ship) {
    g.failCount++
    delete(g.aliens, alien)
    continue
  }
}
```

这里有一个问题，还记得吗？我们前面编写的`CheckCollision`函数接受的参数类型是`*Alien`和`*Bullet`，这里我们需要重复编写接受参数为`*Ship`和`*Alien`的函数吗？不用！

我们将表示游戏中的实体对象抽象成一个`GameObject`结构：

```golang
type GameObject struct {
  width  int
  height int
  x      float64
  y      float64
}

func (gameObj *GameObject) Width() int {
  return gameObj.width
}

func (gameObj *GameObject) Height() int {
  return gameObj.height
}

func (gameObj *GameObject) X() float64 {
  return gameObj.x
}

func (gameObj *GameObject) Y() float64 {
  return gameObj.y
}
```

然后定义一个接口`Entity`：

```golang
type Entity interface {
  Width() int
  Height() int
  X() float64
  Y() float64
}
```

最后让我们游戏中的实体内嵌这个`GameObject`对象，即可自动实现`Entity`接口：

```golang
type Alien struct {
  GameObject
  image       *ebiten.Image
  speedFactor float64
}
```

这样`CheckCollision`即可改为接受两个`Entity`接口类型的参数：

```golang
// CheckCollision 两个物体之间是否碰撞
func CheckCollision(entityA, entityB Entity) bool {
  top, left := entityA.Y(), entityA.X()
  bottom, right := entityA.Y()+float64(entityA.Height()), entityA.X()+float64(entityA.Width())
  // 左上角
  x, y := entityB.X(), entityB.Y()
  if y > top && y < bottom && x > left && x < right {
    return true
  }

  // 右上角
  x, y = entityB.X()+float64(entityB.Width()), entityB.Y()
  if y > top && y < bottom && x > left && x < right {
    return true
  }

  // 左下角
  x, y = entityB.X(), entityB.Y()+float64(entityB.Height())
  if y > top && y < bottom && x > left && x < right {
    return true
  }

  // 右下角
  x, y = entityB.X()+float64(entityB.Width()), entityB.Y()+float64(entityB.Height())
  if y > top && y < bottom && x > left && x < right {
    return true
  }

  return false
}
```

如果游戏失败则切换为ModeOver模式，屏幕上显示"Game Over!"。如果游戏胜利，则显示"You Win!"：

```golang
if g.failCount >= 3 {
  g.overMsg = "Game Over!"
} else if len(g.aliens) == 0 {
  g.overMsg = "You Win!"
}

if len(g.overMsg) > 0 {
  g.mode = ModeOver
  g.aliens = make(map[*Alien]struct{})
  g.bullets = make(map[*Bullet]struct{})
}
```

注意，为了下次游戏能顺序进行，这里需要清楚所有的子弹和外星人。运行：

![](/img/in-post/godailylib/ebiten21.gif#center#center)

## 总结

本文接着上篇文章，介绍了发射子弹，检测碰撞，增加主界面和游戏结束界面等内容。至此一个简答的游戏就做出来了。可以看出使用ebitengine做一个游戏还是很简单的，非常推荐尝试呢！上手之后，建议看看官方仓库examples目录中的示例，会非常有帮助。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)
2. ebitengine 官网：[https://ebitengine.org/](https://ebitengine.org/)
3. Python 编程（从入门到实践）：[https://book.douban.com/subject/35196328/](https://book.douban.com/subject/35196328/)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxsearch.png#center)