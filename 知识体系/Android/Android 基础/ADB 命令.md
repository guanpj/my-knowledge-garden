# ADB 命令

# shell

adb shell 常用的命令有 ：

- am（activity manager）

可执行 启动 activity，service，broadcast，杀死进程等操作。

- pm（package manager）

可以执行 安装/卸载 应用，输出 apk 路径等操作。

- dpm（device policy manager）

可以执行设备管理器相关命令，激活设备管理员，设置设备管理器 owner 等。

- screencap （截屏）

使用该命令可以快速截屏并将图片保存至指定路径，截屏过程用户无感知。

# ps

adb shell ps --help

查看帮助

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/ADB/clipboard_20230323_041900.png)

## 常用参数

- -p 后面加 进程 id ，可以显示进程为指定 id 的信息

```apache
adb shell ps -p 1

USER            PID   PPID     VSZ    RSS WCHAN            ADDR S NAME                       
root              1      0 10782796  4384 0                   0 S init
```

- -P 后面加 父进程 id，可以显示父进程为指定 id 的信息

```apache
adb shell ps -P 1

USER            PID   PPID     VSZ    RSS WCHAN            ADDR S NAME                       
root            133      1 10761096  2540 0                   0 S init
root            285      1 13495412 127540 0                  0 S zygote64
root            286      1 1840080 116252 0                   0 S zygote
```

- -u 后面加 user 名，可以显示指定 user 的信息

```perl
adb shell ps -u system

USER            PID   PPID     VSZ    RSS WCHAN            ADDR S NAME                       
system          164      1 10759476  4296 0                   0 S servicemanager
system          551    285 13891404 262940 0                  0 S system_server
// 省略...
```

- -T 将线程信息也显示出来

## 常用属性

使用 `adb shell ps -o HELP` 可以查看 ps 命令可以查看的所有属性信息

```sql
Command line field types:

  ARGS    CMDLINE minus initial path     CMD     Thread name (/proc/TID/stat:2)
  CMDLINE Command line (argv[])          COMM    EXE filename (/proc/PID/exe)
  COMMAND EXE path (/proc/PID/exe)       NAME    Process name (PID's argv[0])

Process attribute field types:

  S       Process state:
      R (running) S (sleeping) D (device I/O) T (stopped)  t (trace stop)
      X (dead)    Z (zombie)   P (parked)     I (idle)
      Also between Linux 2.6.33 and 3.13:
      x (dead)    K (wakekill) W (waking)

  SCH     Scheduling policy (0=other, 1=fifo, 2=rr, 3=batch, 4=iso, 5=idle)
  STAT    Process state (S) plus:
      < high priority          N low priority L locked memory
      s session leader         + foreground   l multithreaded
  %CPU    Percentage of CPU time used    %MEM    RSS as % of physical memory
  %VSZ    VSZ as % of physical memory    ADDR    Instruction pointer
  BIT     32 or 64                       C       Total %CPU used since start
  CPU     Which processor running on     DIO     Disk I/O
  DREAD   Data read from disk            DWRITE  Data written to disk
  ELAPSED Elapsed time since PID start   F       Flags 1=FORKNOEXEC 4=SUPERPRIV
  GID     Group ID                       GROUP   Group name
  IO      Data I/O                       LABEL   Security label
  MAJFL   Major page faults              MINFL   Minor page faults
  NI      Niceness (static 19 to -20)    PCY     Android scheduling policy
  PGID    Process Group ID               PID     Process ID
  PPID    Parent Process ID              PR      Prio Reversed (dyn 39-0, RT)
  PRI     Priority (dynamic 0 to 139)    PSR     Processor last executed on
  READ    Data read                      RES     Short RSS
  RGID    Real (before sgid) Group ID    RGROUP  Real (before sgid) group name
  RSS     Resident Set Size (DRAM pages) RTPRIO  Realtime priority
  RUID    Real (before suid) user ID     RUSER   Real (before suid) user name
  SHR     Shared memory                  STIME   Start time (ISO 8601)
  SWAP    Swap I/O                       SZ      4k pages to swap out
  TCNT    Thread count                   TID     Thread ID
  TIME    CPU time consumed              TIME+   CPU time (high precision)
  TTY     Controlling terminal           UID     User id
  USER    User name                      VIRT    Virtual memory size
  VSZ     Virtual memory size (1k units) WCHAN   Wait location in kernel
  WRITE   Data written                   
```

这里列举一些常用的属性

命令行属性：

- NAME 进程名称
- CMD 线程名

进程属性：

- S 代表着进程状态，其中 R 代表运行中（running），S 代表休眠中（sleeping），X 代表死亡（dead）
- USER 代表 User 名称
- PID 进程 id
- PPID 父进程 id
- VSZ 进程虚拟地址空间大小
- PCY Android 系统调度策略
- TID 线程 id

ps 命令默认显示 USER，PID，PPID，VSZ，RSS，WCHAN，ADDR，S，NAME 几个属性

可以使用 adb shell ps -o + 欲显示的属性名，逗号隔开 来自定义输出信息，例如输出 USER,UID,PID,PPID,PGID,PCY,NAME 几个属性，且进程 id 为 285 的进程信息

```apache
adb shell ps -o USER,UID,PID,PPID,PGID,PCY,NAME  -p 285

USER           UID    PID   PPID  PGID PCY NAME                       
root             0    285      1   285  ta zygote64
```

# dympsys

## 服务总览

dumpsys service 名称

## 参数查看

- adb shell dumpsys activity -h
- adb shell dumpsys window -h
- adb shell dumpsys meminfo -h
- adb shell dumpsys package -h
- adb shell dumpsys batteryinfo -h

dumpsys activity containers

直观地查看 Activity 返回栈

dumpsys activity lastanr

查看自开机以来出现的最新的 ANR，即：关机失效，最新的会覆盖上一次

dumpsys activity starter

查看 Activity 的启动者

adb shell dumpsys activity activities | grep mResumedActivity

查看栈顶 Activity

adb shell dumpsys activity top | grep mParent

查看栈顶 activity 的所有 fragment，不过很多时候 activity 中会有一些不可见的 fragment 。例如：用于分发 Lifecycle 的 ReportFragment，Glide 中的空 fragment。因此我们可以使用 `grep` 将其过滤掉

<strong>adb shell dumpsys meminfo -s [process] </strong>其中 process 输入 pid 和 applicationId 均可

按比例分摊的内存大小 (Proportional Set Size - <strong>PSS</strong>)

应用使用的 <strong>非共享页数量 + 共享页均匀分摊数量</strong>（例如，如果三个进程共享 3MB，则每个进程的 PSS 为 1MB）

adb shell dumpsys activity o

查询 OOM 相关的信息

adb shell dumpsys activity p

查看每个进程详细的信息

adb shell am force-stop 包名

强制杀死应用
