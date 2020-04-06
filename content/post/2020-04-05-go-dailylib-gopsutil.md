---
layout:    post
title:    "Go 每日一库之 gopsutil"
subtitle: 	"每天学习一个 Go 库"
date:		2020-04-05T21:30:23
author:		"darjun"
image:	"img/post-bg-2015.jpg"
tags:
    - Go 每日一库
URL: "2020/04/05/godailylib/gopsutil"
categories: [
	"Go"
]
---

## 简介

`gopsutil`是 Python 工具库[`psutil`](https://github.com/giampaolo/psutil) 的 Golang 移植版，可以帮助我们方便地获取各种系统和硬件信息。`gopsutil`为我们屏蔽了各个系统之间的差异，具有非常强悍的可移植性。有了`gopsutil`，我们不再需要针对不同的系统使用`syscall`调用对应的系统方法。更棒的是`gopsutil`的实现中没有任何`cgo`的代码，使得交叉编译成为可能。

## 快速使用

先安装：

```golang
$ go get github.com/shirou/gopsutil
```

由于`gopsutil`库用到了`golang.org/x/sys`，后者在墙外，如果有类似下面的报错：

```golang
cannot find package "golang.org/x/sys/windows"
```

可使用下面的命令下载`golang.org/x/sys`在 GitHub 上的镜像：

```golang
$ git clone git@github.com:golang/sys.git $GOPATH/src/golang.org/x/sys
```

使用：

```golang
package main

import (
  "fmt"

  "github.com/shirou/gopsutil/mem"
)

func main() {
  v, _ := mem.VirtualMemory()

  fmt.Printf("Total: %v, Available: %v, UsedPercent:%f%%\n", v.Total, v.Available, v.UsedPercent)

  fmt.Println(v)
}
```

`gopsutil`将不同的功能划分到不同的子包中：

* `cpu`：CPU 相关；
* `disk`：磁盘相关；
* `docker`：docker 相关；
* `host`：主机相关；
* `mem`：内存相关；
* `net`：网络相关；
* `process`：进程相关；
* `winservices`：Windows 服务相关。

想要使用对应的功能，要导入对应的子包。例如，上面代码中，我们要获取内存信息，导入的是`mem`子包。`mem.VirtualMemory()`方法返回内存信息结构`mem.VirtualMemoryStat`，该结构有丰富的字段，我们最常使用的无外乎`Total`（总内存）、`Available`（可用内存）、`Used`（已使用内存）和`UsedPercent`（内存使用百分比）。`mem.VirtualMemoryStat`还实现了`fmt.Stringer`接口，以 JSON 格式返回内存信息。语句`fmt.Println(v)`会自动调用`v.String()`，将返回信息输出。程序输出：

```golang
Total: 8526921728, Available: 3768975360, UsedPercent:55.000000%
{"total":8526921728,"available":3768975360,"used":4757946368,"usedPercent":55,"free":0,"active":0,"inactive":0,"wired":0,"laundry":0,"buffers":0,"cached":0,"writeback":0,"dirty":0,"writebacktmp":0,"shared":0,"slab":0,"sreclaimable":0,"sunreclaim":0,"pagetables":0,"swapcached":0,"commitlimit":0,"committedas":0,"hightotal":0,"highfree":0,"lowtotal":0,"lowfree":0,"swaptotal":0,"swapfree":0,"mapped":0,"vmalloctotal":0,"vmallocused":0,"vmallocchunk":0,"hugepagestotal":0,"hugepagesfree":0,"hugepagesize":0}
```

单位为字节，我的电脑内存 8GB，当前使用百分比为 55%，可用内存 3768975360B（即 3.51GB）。

## CPU

我们知道 CPU 的核数有两种，一种是物理核数，一种是逻辑核数。物理核数就是主板上实际有多少个 CPU，一个物理 CPU 上可以有多个核心，这些核心被称为逻辑核。`gopsutil`中 CPU 相关功能在`cpu`子包中，`cpu`子包提供了获取物理和逻辑核数、CPU 使用率的接口：

* `Counts(logical bool)`：传入`false`，返回物理核数，传入`true`，返回逻辑核数；
* `Percent(interval time.Duration, percpu bool)`：表示获取`interval`时间间隔内的 CPU 使用率，`percpu`为`false`时，获取总的 CPU 使用率，`percpu`为`true`时，分别获取每个 CPU 的使用率，返回一个`[]float64`类型的值。

例如：

```golang
func main() {
  physicalCnt, _ := cpu.Counts(false)
  logicalCnt, _ := cpu.Counts(true)
  fmt.Printf("physical count:%d logical count:%d\n", physicalCnt, logicalCnt)

  totalPercent, _ := cpu.Percent(3*time.Second, false)
  perPercents, _ := cpu.Percent(3*time.Second, true)
  fmt.Printf("total percent:%v per percents:%v", totalPercent, perPercents)
}
```

上面代码获取物理核数和逻辑核数，并获取 3s 内的总 CPU 使用率和每个 CPU 各自的使用率，程序输出（注意每次运行输出可能都不相同）：

```golang
physical count:4 logical count:8
total percent:[30.729166666666668] per percents:[32.64248704663213 26.94300518134715 44.559585492227974 23.958333333333336 36.787564766839374 20.3125 38.54166666666667 28.125]
```

### 详细信息

调用`cpu.Info()`可获取 CPU 的详细信息，返回`[]cpu.InfoStat`：

```golang
func main() {
  infos, _ := cpu.Info()
  for _, info := range infos {
    data, _ := json.MarshalIndent(info, "", " ")
    fmt.Print(string(data))
  }
}
```

为了方便查看，我使用 JSON 输出结果：

```golang
{
 "cpu": 0,
 "vendorId": "GenuineIntel",
 "family": "198",
 "model": "",
 "stepping": 0,
 "physicalId": "BFEBFBFF000906E9",
 "coreId": "",
 "cores": 8,
 "modelName": "Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz",
 "mhz": 3601,
 "cacheSize": 0,
 "flags": [],
 "microcode": ""
}
```

由结果可以看出，CPU 是 Intel 的 i7-7700 系列，频率 3.60GHz。上面是我在 Windows 上运行的返回结果，内部使用了`github.com/StackExchange/wmi`库。在 Linux 下每个逻辑 CPU 都会返回一个`InfoStat`结构。

### 时间占用

调用`cpu.Times(percpu bool)`可以获取从开机算起，总 CPU 和 每个单独的 CPU 时间占用情况。传入`percpu=false`返回总的，传入`percpu=true`返回单个的。每个 CPU 时间占用情况是一个`TimeStat`结构：

```golang
// src/github.com/shirou/gopsutil/cpu/cpu.go
type TimesStat struct {
  CPU       string  `json:"cpu"`
  User      float64 `json:"user"`
  System    float64 `json:"system"`
  Idle      float64 `json:"idle"`
  Nice      float64 `json:"nice"`
  Iowait    float64 `json:"iowait"`
  Irq       float64 `json:"irq"`
  Softirq   float64 `json:"softirq"`
  Steal     float64 `json:"steal"`
  Guest     float64 `json:"guest"`
  GuestNice float64 `json:"guestNice"`
}
```

* `CPU`：CPU 标识，如果是总的，该字段为`cpu-total`，否则为`cpu0`、`cpu1`...；
* `User`：用户时间占用（用户态）；
* `System`：系统时间占用（内核态）；
* `Idle`：空闲时间；
* ...

例如：

```golang
func main() {
  infos, _ := cpu.Times(true)
  for _, info := range infos {
    data, _ := json.MarshalIndent(info, "", " ")
    fmt.Print(string(data))
  }
}
```

为了方便查看，我用 JSON 输出结果，下面是其中一个输出：

```golang
{
 "cpu": "cpu0",
 "user": 674.46875,
 "system": 1184.984375,
 "idle": 7497.1875,
 "nice": 0,
 "iowait": 0,
 "irq": 75.578125,
 "softirq": 0,
 "steal": 0,
 "guest": 0,
 "guestNice": 0
}
```

## 磁盘

子包`disk`用于获取磁盘信息。`disk`可获取 IO 统计、分区和使用率信息。下面依次介绍。

### IO 统计

调用`disk.IOCounters()`函数，返回的 IO 统计信息用`map[string]IOCountersStat`类型表示。每个分区一个结构，键为分区名，值为统计信息。这里摘取统计结构的部分字段，主要有读写的次数、字节数和时间：

```golang
// src/github.com/shirou/gopsutil/disk/disk.go
type IOCountersStat struct {
  ReadCount        uint64 `json:"readCount"`
  MergedReadCount  uint64 `json:"mergedReadCount"`
  WriteCount       uint64 `json:"writeCount"`
  MergedWriteCount uint64 `json:"mergedWriteCount"`
  ReadBytes        uint64 `json:"readBytes"`
  WriteBytes       uint64 `json:"writeBytes"`
  ReadTime         uint64 `json:"readTime"`
  WriteTime        uint64 `json:"writeTime"`
  // ...
}
```

例如：

```golang
func main() {
  mapStat, _ := disk.IOCounters()
  for name, stat := range mapStat {
    fmt.Println(name)
    data, _ := json.MarshalIndent(stat, "", "  ")
    fmt.Println(string(data))
  }
}
```

输出包括所有分区，我这里只展示一个：

```golang
C:
{
  "readCount": 184372,
  "mergedReadCount": 0,
  "writeCount": 42252,
  "mergedWriteCount": 0,
  "readBytes": 5205152768,
  "writeBytes": 701583872,
  "readTime": 333,
  "writeTime": 27,
  "iopsInProgress": 0,
  "ioTime": 0,
  "weightedIO": 0,
  "name": "C:",
  "serialNumber": "",
  "label": ""
}
```

注意，`disk.IOCounters()`可传入可变数量的字符串参数用于标识分区，**此参数在 Windows 上无效**。

### 分区

调用`disk.PartitionStat(all bool)`函数，返回分区信息。如果`all = false`，只返回实际的物理分区（包括硬盘、CD-ROM、USB），忽略其它的虚拟分区。如果`all = true`则返回所有的分区。返回类型为`[]PartitionStat`，每个分区对应一个`PartitionStat`结构：

```golang
// src/github.com/shirou/gopsutil/disk/
type PartitionStat struct {
  Device     string `json:"device"`
  Mountpoint string `json:"mountpoint"`
  Fstype     string `json:"fstype"`
  Opts       string `json:"opts"`
}
```

* `Device`：分区标识，在 Windows 上即为`C:`这类格式；
* `Mountpoint`：挂载点，即该分区的文件路径起始位置；
* `Fstype`：文件系统类型，Windows 常用的有 FAT、NTFS 等，Linux 有 ext、ext2、ext3等；
* `Opts`：选项，与系统相关。

例如：

```golang
func main() {
  infos, _ := disk.Partitions(false)
  for _, info := range infos {
    data, _ := json.MarshalIndent(info, "", "  ")
    fmt.Println(string(data))
  }
}
```

我的 Windows 机器输出（只展示第一个分区）：

```golang
{
  "device": "C:",
  "mountpoint": "C:",
  "fstype": "NTFS",
  "opts": "rw.compress"
}
```

由上面的输出可知，我的第一个分区为`C:`，文件系统类型为`NTFS`。

### 使用率

调用`disk.Usage(path string)`即可获得路径`path`所在磁盘的使用情况，返回一个`UsageStat`结构：

```golang
// src/github.com/shirou/gopsutil/disk.go
type UsageStat struct {
  Path              string  `json:"path"`
  Fstype            string  `json:"fstype"`
  Total             uint64  `json:"total"`
  Free              uint64  `json:"free"`
  Used              uint64  `json:"used"`
  UsedPercent       float64 `json:"usedPercent"`
  InodesTotal       uint64  `json:"inodesTotal"`
  InodesUsed        uint64  `json:"inodesUsed"`
  InodesFree        uint64  `json:"inodesFree"`
  InodesUsedPercent float64 `json:"inodesUsedPercent"`
}
```

* `Path`：路径，传入的参数；
* `Fstype`：文件系统类型；
* `Total`：该分区总容量；
* `Free`：空闲容量；
* `Used`：已使用的容量；
* `UsedPercent`：使用百分比。

例如：

```golang
func main() {
  info, _ := disk.Usage("D:/code/golang")
  data, _ := json.MarshalIndent(info, "", "  ")
  fmt.Println(string(data))
}
```

由于返回的是磁盘的使用情况，所以路径`D:/code/golang`和`D:`返回同样的结果，只是结构中的`Path`字段不同而已。程序输出：

```golang
{
  "path": "D:/code/golang",
  "fstype": "",
  "total": 475779821568,
  "free": 385225650176,
  "used": 90554171392,
  "usedPercent": 19.032789388496106,
  "inodesTotal": 0,
  "inodesUsed": 0,
  "inodesFree": 0,
  "inodesUsedPercent": 0
}
```

## 主机

子包`host`可以获取主机相关信息，如开机时间、内核版本号、平台信息等等。

### 开机时间

`host.BootTime()`返回主机开机时间的时间戳：

```golang
func main() {
  timestamp, _ := host.BootTime()
  t := time.Unix(int64(timestamp), 0)
  fmt.Println(t.Local().Format("2006-01-02 15:04:05"))
}
```

上面先获取开机时间，然后通过`time.Unix()`将其转为`time.Time`类型，最后输出`2006-01-02 15:04:05`格式的时间：

```golang
2020-04-06 20:25:32
```

### 内核版本和平台信息

```golang
func main() {
  version, _ := host.KernelVersion()
  fmt.Println(version)

  platform, family, version, _ := host.PlatformInformation()
  fmt.Println("platform:", platform)
  fmt.Println("family:", family,
  fmt.Println("version:", version)
}
```

在我的 Win10 上运行输出：

```golang
10.0.18362 Build 18362
platform: Microsoft Windows 10 Pro
family: Standalone Workstation
version: 10.0.18362 Build 18362
```

### 终端用户

`host.Users()`返回终端连接上来的用户信息，每个用户一个`UserStat`结构：

```golang
// src/github.com/shirou/gopsutil/host/host.go
type UserStat struct {
  User     string `json:"user"`
  Terminal string `json:"terminal"`
  Host     string `json:"host"`
  Started  int    `json:"started"`
}
```

字段一目了然，看示例：

```golang
func main() {
  users, _ := host.Users()
  for _, user := range users {
    data, _ := json.MarshalIndent(user, "", " ")
    fmt.Println(string(data))
  }
}
```

## 内存

在[快速开始](#快速开始)中，我们演示了如何使用`mem.VirtualMemory()`来获取内存信息。该函数返回的只是物理内存信息。我们还可以使用`mem.SwapMemory()`获取交换内存的信息，信息存储在结构`SwapMemoryStat`中：

```golang
// src/github.com/shirou/gopsutil/mem/
type SwapMemoryStat struct {
  Total       uint64  `json:"total"`
  Used        uint64  `json:"used"`
  Free        uint64  `json:"free"`
  UsedPercent float64 `json:"usedPercent"`
  Sin         uint64  `json:"sin"`
  Sout        uint64  `json:"sout"`
  PgIn        uint64  `json:"pgin"`
  PgOut       uint64  `json:"pgout"`
  PgFault     uint64  `json:"pgfault"`
}
```

字段含义很容易理解，`PgIn/PgOut/PgFault`这三个字段我们重点介绍一下。交换内存是以**页**为单位的，如果出现缺页错误(`page fault`)，操作系统会将磁盘中的某些页载入内存，同时会根据特定的机制淘汰一些内存中的页。`PgIn`表征载入页数，`PgOut`淘汰页数，`PgFault`缺页错误数。

例如：

```golang
func main() {
  swapMemory, _ := mem.SwapMemory()
  data, _ := json.MarshalIndent(swapMemory, "", " ")
  fmt.Println(string(data))
}
```

## 进程

`process`可用于获取系统当前运行的进程信息，创建新进程，对进程进行一些操作等。

```golang
func main() {
  var rootProcess *process.Process
  processes, _ := process.Processes()
  for _, p := range processes {
    if p.Pid == 0 {
      rootProcess = p
      break
    }
  }

  fmt.Println(rootProcess)

  fmt.Println("children:")
  children, _ := rootProcess.Children()
  for _, p := range children {
    fmt.Println(p)
  }
}
```

先调用`process.Processes()`获取当前系统中运行的所有进程，然后找到`Pid`为 0 的进程，即操作系统的第一个进程，最后调用`Children()`返回其子进程。还有很多方法可获取进程信息，感兴趣可查看文档了解~

## Windows 服务

`winservices`子包可以获取 Windows 系统中的服务信息，内部使用了`golang.org/x/sys`包。在`winservices`中，一个服务对应一个`Service`结构：

```golang
// src/github.com/shirou/gopsutil/winservices/winservices.go
type Service struct {
  Name   string
  Config mgr.Config
  Status ServiceStatus
  // contains filtered or unexported fields
}
```

`mgr.Config`为包`golang.org/x/sys`中的结构，该结构详细记录了服务类型、启动类型（自动/手动）、二进制文件路径等信息：

```golang
// src/golang.org/x/sys/windows/svc/mgr/config.go
type Config struct {
  ServiceType      uint32
  StartType        uint32
  ErrorControl     uint32
  BinaryPathName   string
  LoadOrderGroup   string
  TagId            uint32
  Dependencies     []string
  ServiceStartName string
  DisplayName      string
  Password         string
  Description      string
  SidType          uint32
  DelayedAutoStart bool
}
```

`ServiceStatus`结构记录了服务的状态：

```golang
// src/github.com/shirou/gopsutil/winservices/winservices.go
type ServiceStatus struct {
  State         svc.State
  Accepts       svc.Accepted
  Pid           uint32
  Win32ExitCode uint32
}
```

* `State`：为服务状态，有已停止、运行、暂停等；
* `Accepts`：表示服务接收哪些操作，有暂停、继续、会话切换等；
* `Pid`：进程 ID；
* `Win32ExitCode`：应用程序退出状态码。

下面程序中，我将系统中所有服务的名称、二进制文件路径和状态输出到控制台：

```golang
func main() {
  services, _ := winservices.ListServices()

  for _, service := range services {
    newservice, _ := winservices.NewService(service.Name)
    newservice.GetServiceDetail()
    fmt.Println("Name:", newservice.Name, "Binary Path:", newservice.Config.BinaryPathName, "State: ", newservice.Status.State)
  }
}
```

注意，调用`winservices.ListServices()`返回的`Service`对象信息是不全的，我们通过`NewService()`以该服务名称创建一个服务，然后调用`GetServiceDetail()`方法获取该服务的详细信息。不能直接通过`service.GetServiceDetail()`来调用，因为`ListService()`返回的对象缺少必要的系统资源句柄（为了节约资源），调用`GetServiceDetail()`方法会`panic`！！！

## 错误和超时

由于大部分函数都涉及到底层的系统调用，所以发生错误和超时是在所难免的。几乎所有的接口都有两个返回值，第二个作为错误。在前面的例子中，我们为了简化代码都忽略了错误，在实际使用中，建议对错误进行处理。

另外，大部分接口都是一对，一个不带`context.Context`类型的参数，另一个带有该类型参数，用于做上下文控制。在内部调用发生错误或超时后能及时处理，避免长时间等待返回。实际上，不带`context.Context`参数的函数内部都是以`context.Background()`为参数调用带有`context.Context`的函数的：

```golang
// src/github.com/shirou/gopsutil/cpu_windows.go
func Times(percpu bool) ([]TimesStat, error) {
  return TimesWithContext(context.Background(), percpu)
}

func TimesWithContext(ctx context.Context, percpu bool) ([]TimesStat, error) {
  // ...
}
```

## 总结

`gopsutil`库方便了我们获取本机的信息，且很好地处理了各个系统间的兼容问题，提供了一致的接口。还有几个子包例如`net/docker`限于篇幅没有介绍，感兴趣的童鞋可自行探索。

大家如果发现好玩、好用的 Go 语言库，欢迎到 Go 每日一库 GitHub 上提交 issue😄

## 参考

1. gopsutil GitHub：[https://github.com/shirou/gopsutil](https://github.com/shirou/gopsutil)
2. Go 每日一库 GitHub：[https://github.com/darjun/go-daily-lib](https://github.com/darjun/go-daily-lib)

## 我

我的博客：[https://darjun.github.io](https://darjun.github.io)

欢迎关注我的微信公众号【GoUpUp】，共同学习，一起进步~

![](/img/wxgzh8.jpg#center)