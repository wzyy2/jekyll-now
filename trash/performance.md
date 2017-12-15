## 1 介绍

​	perf是从Linux 2.6开始引入的一个profiling工具，通过访问包括pmu在内的软硬件性能计数器来分析性能，支持多架构，是目前kernel的主要性能检测手段，和kernel代码一起发布，所以兼容性良好。

## 2 功能

​	性能瓶颈如果要分类的话，大致可以分为几个大类：cpu／gpu／mem／storage，其中gpu用perf没法探测（这个目前比较好用的工具就只有ds5），storage只能用tracepoint来统计。总的说来，perf还是侧重于分析cpu的性能，其他功能都不是很好用。

```markdown
perf

 usage: perf [--version] [--help] COMMAND [ARGS]

 The most commonly used perf commands are:
  annotate        Read perf.data (created by perf record) and display annotated code
  archive         Create archive with object files with build-ids found in perf.data file
  bench           General framework for benchmark suites
  buildid-cache   Manage <tt>build-id</tt> cache.
  buildid-list    List the buildids in a perf.data file
  diff            Read two perf.data files and display the differential profile
  inject          Filter to augment the events stream with additional information
  kmem            Tool to trace/measure kernel memory(slab) properties
  kvm             Tool to trace/measure kvm guest os
  list            List all symbolic event types
  lock            Analyze lock events
  probe           Define new dynamic tracepoints
  record          Run a command and record its profile into perf.data
  report          Read perf.data (created by perf record) and display the profile
  sched           Tool to trace/measure scheduler properties (latencies)
  script          Read perf.data (created by perf record) and display trace output
  stat            Run a command and gather performance counter statistics
  test            Runs sanity tests.
  timechart       Tool to visualize total system behavior during a workload
  top             System profiling tool.

 See 'perf help COMMAND' for more information on a specific command.
```

​	其中比较常用的功能有几个：

* record：收集profile数据
* report：根据profile数据生成统计报告
* stat：打印性能计数统计值
* top：cpu占有率实时统计

## 3 在Android平台使用

##### 3.1 准备工作

###### 1. 首先按google或芯片厂商的指导，构建一个完整的android和kernel的编译环境（如果不关心kernel可以忽略）, 这样分析的时候符号表才能匹配上。

###### 2. 编译perf

    ​```shell
# setup your build environment first
~$ . build/envsetup.sh
~$ lunch
~$ mmm external/linux-tools-perf
~$ adb root
~$ adb remount
~$ adb push perf /system/bin/
~$ adb shell sync
    ​```

###### 3. 准备符号文件

符号文件可以简单分为三类：

a. 平台native代码，这部分代码在编译的过程中会自动生成符号表，不需要我们干预

b. 平台java代码，对于art虚拟机来说（老版本的dalvik就不说了）最终的编译结果是oat文件，这也是正规的elf文件，但是默认是不带debug信息。而新版本的android也提供了自动生成java符号表的工具：

  ```shell
bash art/tools/symbolize.sh
  ```

c. 第三方apk，如果是来自开源社区，则可以通过修改makefile和套用android提供的java符号表工具来生成符号表文件，然后拷贝到android的符号表目录，==注意路径必须要和设备上的完全一致==，可以通过showmap来获取设备上的路径。

  ```shell
~$ adb shell showmap apk_pid
38540    36296    36296        0        0    36216       80        0    3 /data/app/com.android.webview-2/lib/arm/libwebviewchromium.so
~$ cp libwebviewchromium.so $ANDROID_PRODUCT_OUT/symbols/data/app/com.android.webview-2/lib/arm/libwebviewchromium.so
  ```

  如果是商业的apk，基本上已经做过混淆和strip，除非开发商能配合，不然就没招。

###### 4. 稍微新一点的android都开起了kernel的指针保护，这也会影响perf的record，所以需要临时关闭保护：

  ```shell
~$ adb shell echo 0 > /proc/sys/kernel/kptr_restrict
  ```

###### 5. 为了方便分析，一般会把record的数据pull到host端，在host端做分析，所以需要在设备端也安装一下perf工具，ubuntu下安装命令如下：

```shell
~$ sudo apt-get install linux-tools-common
```

###### 6. 目前大部分的android平台默认perf功能都是打开的，所以一般不需要重新配置kernel，如果碰到perf被关闭的情况，可以打开下面几个配置

```shell
CONFIG_PERF_EVENTS=y
CONFIG_HW_PERF_EVENTS=y
```

##### 3.2 获取当前平台支持的事件

```shell
rk3399:/data/local # ./perf list                                                                                                                                                                               

List of pre-defined events (to be used in -e):
  cpu-cycles OR cycles                               [Hardware event]
  instructions                                       [Hardware event]
  cache-references                                   [Hardware event]
  cache-misses                                       [Hardware event]
  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]

  cpu-clock                                          [Software event]
  task-clock                                         [Software event]
  page-faults OR faults                              [Software event]
  context-switches OR cs                             [Software event]
  cpu-migrations OR migrations                       [Software event]
  minor-faults                                       [Software event]
  major-faults                                       [Software event]
  alignment-faults                                   [Software event]
  emulation-faults                                   [Software event]
  dummy                                              [Software event]

  L1-dcache-loads                                    [Hardware cache event]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-stores                                   [Hardware cache event]
  L1-dcache-store-misses                             [Hardware cache event]
  L1-dcache-prefetch-misses                          [Hardware cache event]
  L1-icache-loads                                    [Hardware cache event]
  L1-icache-load-misses                              [Hardware cache event]
  dTLB-load-misses                                   [Hardware cache event]
  dTLB-store-misses                                  [Hardware cache event]
  iTLB-load-misses                                   [Hardware cache event]
  branch-loads                                       [Hardware cache event]
  branch-load-misses                                 [Hardware cache event]
```

实际上android移植的perf还不完整，tracepoint的事件还不支持，例如：block事件，所以如果想要抓去一些内核子系统的性能信息就无法满足。android 7.0开始已经去掉了perf工具，替代它的是simpleperf[^1]工具，对tracepoint的支持比原来的好很多。

[^1]: 后面也会简单介绍一些simpleperf

##### 3.3 获取系统热点进程

perf中的top工具可以列出当前cpu的热点，还可以附加kernel的符号表让信息可方便分析。命令如下：

```shell
$ adb shell mkdir -p /data/local/symbols
$ adb push vmlinux /data/local/symbols/vmlinux
$ adb shell
# perf top --vmlinux=/path/to/vmlinux -d 2
```

结果输出如下：

<img src="https://ws3.sinaimg.cn/large/006tNc79gy1flgi2ejdcfj30xu0dgmzu.jpg"></img>

perf top还可以只抓取指定进程的pid，这一般是用在要优化某个程序是非常有用，命令如下：

```shell
perf top --vmlinux=/path/to/vmlinux -d 2 -p pid_of_prog
```

perf top还和系统的top一样可以指定刷新间隔[^2], 以上命令中的-d选项就是这个功能，单位是秒。

[^2]: 这个是指top统计信息的刷新间隔而不是采样间隔

##### 3.4 获取进程的统计信息

perf stat用于获取进程某个时间段内的pmu统计信息，命令如下：

```shell
 # ./perf stat -p 1415  
```

ctrl+c退出，或发信号让perf进程退出都可以看到统计结果，例如：

<img src="https://ws2.sinaimg.cn/large/006tKfTcgy1flgio6j61zj30vy0b241d.jpg"></img>

一些明显的异常值会被标注为红色，例如上图是浏览器跑fishtank时候抓的统计信息，可以看到分支预测的失败率非常高，结合perf的热点分析工具可以进一步缩小范围找到分支预测失败的原因。

##### 3.5 收集进程的profile数据

perf record用于记录详细的profile数据，可以指定记录某个进程，还可以记录调用栈，命令如下：

```shell
# perf record -g -p pid -o /data/local/perf.data
```

也可以指定只抓取某个事件，事件列表可以通过上面的perf list得到，例如：

```shell
# ./perf record -e cache-misses -p 1415 
```

##### 3.6 分析profile数据

perf report用户分析抓到的profile数据，一般会先把数据发到pc上再分析，命令如下：

```shell
$ adb pull /data/local/perf.data
$ perf report --objdump=aarch64-linux-android-objdump --vmlinux=/path/to/vmlinux --symfs $ANDROID_PRODUCT_OUT/symbols -i perf.data
```

结果如图：

<img src="https://ws4.sinaimg.cn/large/006tKfTcgy1flgm0hrrh9j31080nodpz.jpg"></img>

 上图有‘+’的地方可以用‘enter’键来遍历其调用关系。

###### 3.7 FlameGraph

还可以通过一些脚本来方便分析调用关系，Flame Graph就是一个比较好用的可视化分析工具。

下载：

```shell
$ git clone https://github.com/brendangregg/FlameGraph.git
```

生成图形：

```shell
perf script --vmlinux=<kernel_folder>/vmlinux --symfs $ANDROID_PRODUCT_OUT/symbols -i perf.data | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > flamegraph.html
```



## 在Linux平台使用

arm版本的linux发行版很多都没有提供perf的包，所以需要自己手动编译一个perf，由于perf依赖的elfutils/binutils/zlib，所以实际上需要交叉编译四个东西。

首先编译zlib，[源码地址](http://zlib.net/zlib-1.2.11.tar.gz "zlib")

```shell
$ CC=aarch64-linux-gnu-gcc ./configure --prefix=/home/cmc/workspace/linaro/toolchain/armlinux/aarch64/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu/aarch64-linux-gnu/libc/usr
$ make && make install
```

==Note: prefix要指向你的交叉编译工具的库目录==

编译elfutils，我直接用的最新的版本的：

```shell
$ git clone git://sourceware.org/git/elfutils.git
```

配置：

```shell
$ cd /path/to/elfutils
$ mkdir build
$ ./configure --enable-maintainer-mode --host=aarch64-linux-gnu --prefix=/home/cmc/workspace/linaro/elfutils/build
```

修改Makefile： 删除elfutils根目录下Makefile里面的libcpu

修改backends/Makefile： 删除backends/Makefile中的libebl_i386和libebl_x86_64有关的所有东西

编译：

```shell
$ make && make install
```

编译binutils，这个要考虑和gcc版本的兼容，我用的2.28.1的版本，[源代码地址](http://ftp.gnu.org/gnu/binutils/binutils-2.28.1.tar.bz2 "binutils")

```shell
$ cd /path/to/binutils
$ mkdir build
$ ../configure --target=aarch64-linux-gnu --host=aarch64-linux-gnu --prefix=/home/cmc/workspace/linaro/binutils-2.28.1/build
$ make && make install
```

编译perf，perf是kernel一起发布的，所以直接下载一个kernel就有了，但是交叉编译的话，需要改一些东西：

修改Makefile.perf，在前面加入：

```shell
EXTRA_CFLAGS=-I/path/to/elfutils/build/inclue -L/path/to/elfutils/build/lib -I/path/to/binutils/build/include -L/path/to/binutils/build/lib
WERROR=0
NO_LIBPERL=1
NO_LIBPYTHON=1
```

编译

```shell
$ cd /path/to/kernel/tools/perf
$ make -f Makefile.perf perf ARCH=arm64 CROSS_COMPILE=/home/cmc/workspace/linaro/toolchain/armlinux/aarch64/gcc-linaro-6.3.1-2017.02-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu- -j8
```

理论上在arm的linux发行版上直接编译perf应该也是可以的，但是我没有试过。用法的话和android是一样的，这里就不叙说了。

## Simpleperf使用

android 7.0开始提供了一个更完整的perf版本simpleperf：

```shell
$ source build/envsetup.sh
$ lunch 
$ mmma system/extras/simpleperf
```

simpleperf相对之前google移植的perf有以下改进

- 支持剖析apk中兼容的共享库，从 .gnu_debugdata 段读取符号表和调试信息
- 提供更方便分析的脚本
- 纯静态，所以和android版本无关，只要指令集兼容都能跑

ndk r13开始就提供了simpleperf工具，所以也可以直接下载编译好的工具：

```shell
$ git clone https://aosp.tuna.tsinghua.edu.cn/platform/prebuilts/simpleperf
```

用法上和perf是类似的，命令基本通用，可以直接参考上面perf的命令。

simpleperf更多信息， 特别是调试java程序的方法，请参考[官方手册](https://android.googlesource.com/platform/system/extras/+/master/simpleperf/doc/README.md "simpleperf")






## 1. 介绍

​     streamline是ds5提供的一个性能分析工具，通过抓取cpu和gpu的内部硬件计数器和kernel的一些软件tracepoint来实现性能分析，功能比较强大。

## 2. 抓取数据

- 准备工具

  1. 首先需要下载ds5，一些新的cpu和gpu可能需要较新的ds5才能支持，我目前用的是5.26版本。ds5是需要license的，不过可以先申请一个30天的全功能试用license，下载安装和申请license的流程就不在这里介绍了。
  2. 设备端需要和host建立连接，目前支持两种连接方式：adb和网络
  3. 需要设备端的root权限，不然没法在设备端运行gatord
  4. 需要和固件匹配的符号表，方便后续的分析

- gatord

  设备端需要运行一个守护进程来和host端做交互，早期版本的mali驱动或ds5版本还需要重新编译kernel的gpu驱动来启用gatord，不过目前新版本的ds5已经没有这个问题了。

  ```shell
  $ adb push /path/to/ds5/sw/streamline/bin/$ARCH/gatord /data/local/
  # cd /data/local/
  # ./gatord &
  ```

- 建立连接

  gatord跑起来之后就可以在host端打开streamline，在左上角点击<img src="https://ws1.sinaimg.cn/large/006tKfTcgy1flhpwhifzaj301k00w744.jpg"></img>按钮，会弹出如下对话框：

  <img src="https://ws4.sinaimg.cn/large/006tNc79ly1flhq18n0ysj30rq0m2tbj.jpg"></img>

  选中你要调试的设备，点“select”即可。

- 配置计数器

  streamline支持的计数器和设备相关，在连接完成后，可以点击<img src="https://ws4.sinaimg.cn/large/006tNc79gy1flhq5ycczdj301800w3yd.jpg"></img>按钮，会弹出对话框，列出所有支持的计数器：

  <img src="https://ws1.sinaimg.cn/large/006tNc79ly1flhq92ewy2j30ts0nmn48.jpg"></img>

  左边是可选的计数器，右边是已选的计数器，在可选计数器上双击即可移动到已选计数器，完成后点“save”按钮即可，全部重选可以点击“load defaults"先恢复默认设置。

- 抓取数据

  计数器选好以后，可以点击<img src="https://ws4.sinaimg.cn/large/006tNc79gy1flhqimyv5dj301000s3yc.jpg"></img>开启抓取，结束时点击<img src="https://ws3.sinaimg.cn/large/006tKfTcgy1flhqlf0jx0j301g00y3ye.jpg"></img>按钮，这时候会自动跳转到分析界面。

## 3. 分析结果

streamline相对于传统的profile工具的优势在于：丰富的硬件计数器支持，可以很方便的看到cache，bus和gpu内部状态。

- 加载符号表

  右键单击左侧的我们抓取到的数据名称，在弹出的菜单中点击”analyze"，会弹出如下对话框：

  <img src="https://ws2.sinaimg.cn/large/006tKfTcly1flhratuog8j30ss0h4q77.jpg"></img>

  点击红色箭头指向的按钮就可以添加带符号信息的elf文件了。不用全部添加，根据热点添加即可。

- Heat Map

  这个视图可以很方便的找到热点线程，点击左下角的<img src="https://ws2.sinaimg.cn/large/006tNc79gy1flhri3amkij305o014dft.jpg"></img>

  点击▶️可以展开各个进程，点击每个线程的名字，可以只显示这个线程的统计值，例如线程的cpu占有率，线程的miss rate等。

  时间轴上有个滑块，可以拉伸和移动，以显示某个时间段内的统计数据，类似下图：

  <img src="https://ws3.sinaimg.cn/large/006tNc79gy1flhrtwsy52j318g0p0qbu.jpg"></img>

- Core Map

  这个视图可以看到每个线程各个时刻都在那个core上跑，对于看调度问题比较方便，比如不合理的cpu迁移。

- Cluster Map

  这个视图可以看到每个线程在当前跑在哪个cluster，可以分析是否有不合理的大小核迁移导致性能下降。

- Samples

  这个视图可以分析每个时间片内函数的cpu占比，时间片可以通过上面时间轴上的滑块控制，对于分析热点函数比较有用。

- Processes

  这个视图可以分析热点进程的cpu占比。

- Functions

  点击上部的”functions"选项卡，可以看到整个抓取周期内的函数热点统计，如下图所示：

  <img src="https://ws3.sinaimg.cn/large/006tNc79ly1flhs5r8pafj318g0p07dy.jpg"></img>

- Call Paths

  如果想看函数的调用关系，可以切换到“call paths”选项卡



# 1. 介绍

​    systrace是目前android上最主要的性能调试手段，有以下优点：

- 完全免费，安装和使用都比较简便
- 由于不需要在设备端运行监控程序，所以不需要root权限[^1]
- 界面友好

​    同时也有一些缺点：

- 基于tracepoint，所以只会收集你加过trace的函数信息，android在大部分模块的重要函数里都加了trace了，所以大部分情况下还是够用，同时android也提供了几个函数方便添加自己的trace。
- 看不到pmu计数器的信息，也看不到gpu和memory的信息（理论上内核驱动如果定时收集这些信息并加到trace里，systrace应该也能看到）

### 2. 用法

​    为了更方便介绍systrace，我这里举一个实际的性能分析例子：fishtank在1000只鱼的情况下帧率很低

#### 准备工作

​    获取systrace有三种方式：

1. 下载[android_sdk_tool](https://dl.google.com/android/repository/sdk-tools-linux-3859397.zip "sdk-tools-linux")

   路径：/path_to_sdk/platform-tools/systrace

2. 下载[android studio](https://dl.google.com/dl/android/studio/ide-zips/3.0.0.18/android-studio-ide-171.4408382-linux.zip "android_studio_ide_linux")

   提供了图形化抓取功能，实际上也是嗲用sdk里的systrace

3. 直接用android源码里的

   路径：/path_to_android/external/chromium-trace

#### 抓取数据

​    systrace的命令格式：

```shell
$ cd external/chromium-trace/
$ python ./systrace.py -h
Usage: systrace.py [options] [category1 [category2 ...]]

Example: systrace.py -b 32768 -t 15 gfx input view sched freq

Options:
  -h, --help            show this help message and exit
  -o FILE               write trace output to FILE
  -t N, --time=N        trace for N seconds
  -l, --list-categories
                        list the available categories and exit
  -j, --json            write a JSON file
  --link-assets         (deprecated)
  --from-file=FROM_FILE
                        read the trace from a file (compressed) rather
                        thanrunning a live trace
  --asset-dir=ASSET_DIR
                        (deprecated)
  -e DEVICE_SERIAL_NUMBER, --serial=DEVICE_SERIAL_NUMBER
                        adb device serial number
  --target=TARGET       chose tracing target (android or linux)
  --timeout=TIMEOUT     timeout for start and stop tracing (seconds)
  --collection-timeout=COLLECTION_TIMEOUT
                        timeout for data collection (seconds)

  Atrace and Ftrace options:
    -b N, --buf-size=N  use a trace buffer size  of N KB
    --no-fix-threads    don't fix missing or truncated thread names
    --no-fix-tgids      Do not run extra commands to restore missing thread to
                        thread group id mappings.
    --no-fix-circular   don't fix truncated circular traces

  Atrace options:
    --atrace-categories=ATRACE_CATEGORIES
                        Select atrace categories with a comma-delimited list,
                        e.g. --atrace-categories=cat1,cat2,cat3
    -k KFUNCS, --ktrace=KFUNCS
                        specify a comma-separated list of kernel functions to
                        trace
    -a APP_NAME, --app=APP_NAME
                        enable application-level tracing for comma-separated
                        list of app cmdlines
    --no-compress       Tell the device not to send the trace data in
                        compressed form.
    --boot              reboot the device with tracing during boot enabled.The
                        report is created by hitting Ctrl+C after the
                        devicehas booted up.

  Battor trace options:
    --battor-categories=BATTOR_CATEGORIES
                        Select battor categories with a comma-delimited list,
                        e.g. --battor-categories=cat1,cat2,cat3
    --hubs=HUB_TYPES    List of hub types to check for for BattOr mapping.
                        Used when updating mapping file.
    --serial-map=SERIAL_MAP
                        File containing pregenerated map of phone serial
                        numbers to BattOr serial numbers.
    --battor_path=BATTOR_PATH
                        specify a BattOr path to use
    --update-map        force update of phone-to-BattOr map
    --battor            Use the BattOr tracing agent.

  Ftrace options:
    --ftrace-categories=FTRACE_CATEGORIES
                        Select ftrace categories with a comma-delimited list,
                        e.g. --ftrace-categories=cat1,cat2,cat3
```

systrace支持的atrace类别有：

```shell
$ adb root 
$ python ./systrace.py -l
         gfx - Graphics
       input - Input
        view - View System
     webview - WebView
          wm - Window Manager
          am - Activity Manager
          sm - Sync Manager
       audio - Audio
       video - Video
      camera - Camera
         hal - Hardware Modules
         app - Application
         res - Resource Loading
      dalvik - Dalvik VM
          rs - RenderScript
      bionic - Bionic C Library
       power - Power Management
          pm - Package Manager
          ss - System Server
    database - Database
     network - Network
       sched - CPU Scheduling
        freq - CPU Frequency
        idle - CPU Idle
        load - CPU Load
  memreclaim - Kernel Memory Reclaim
  binder_driver - Binder Kernel driver
  binder_lock - Binder global lock trace

NOTE: more categories may be available with adb root
```

Note: ==有写事件需要设备的root权限才能操作，所以最好先切到root权限==

除了支持android在ftrace基础上扩展的atrace，systrace也是支持kernel原生的ftrace的，还支持单独抓取某个kernel函数，当然前提是这个函数本身有tracepoint，具体可以参见上面的命令帮助信息。还可以直接用trace文件做输入，这种离线分析功能应该在分析android引导过程的时候比较有用。

在抓取前要先大致确定这个场景涉及到哪些模块，再回到我们这次要分析的场景是：浏览器跑fishtank中开启1000只鱼的时候帧率很低；第一时间能想到的模块有：gfx webview sched freq load workq disk

先在设备上重现问题，然后在host端执行如下命令：

```shell
$ cd external/chromium-trace
$ python ./systrace.py -t 10 -o fishtank.html gfx webview sched freq load workq disk
```

这个fishtank.html即我们抓到的数据。为了方便和本文对照，我上传到[网盘](https://pan.baidu.com/s/1slJUibN "fishtank.html")了。

### 3. 分析

​    分析数据需要chrome浏览器，版本最好要新一些，太旧可能会有兼容问题，因为这个html并不符合w3c的标准。

​    用chrome打开以后，界面如下：

<img src="https://ws3.sinaimg.cn/large/006tNc79gy1fliti6e9laj31kw0q8dld.jpg"></img>

左列是抓取的线程名或trace名，既然是绘制问题，我们第一个要看肯定是绘制的线程，android 5.0以前是在ui线程做绘制的，以后的版本都是在render线程做绘制，所以我们先拉到render线程，可以看到如下：

<img src="https://ws2.sinaimg.cn/large/006tNc79gy1flitp8f9fxj31kw0k30w4.jpg"></img>

点击右边的有四个按键，分别对应四种模式如下：

<img src="https://ws2.sinaimg.cn/large/006tNc79gy1flitrkea2xj309u08caag.jpg"></img>

先用”时间线“模式拉个1s左右的时间线，然后切到“选择”模式，选择这段时间线内render线程的区域，会自动在下方列出这个区域的函数统计：

<img src="https://ws1.sinaimg.cn/large/006tNc79gy1fliu1ik579j31kw0cc78t.jpg"></img>

可以看到帧率确实很低，这段时间的绘制只有9次（drawgl次数），平均耗时29ms，平均间隔是111ms，所以主要原因是绘制间隔太大。继续往下分析就要根据浏览器的渲染模型了，我们知道chromium里是由光栅化和canvas线程完成实际绘制的（内部叫paint），而ui线程或render线程来完成贴图（内部叫draw）。因为这个网页用的canvas，所以我们先用“时间线”模式拉出绘制间隔，然后顺着时间线往下找绘制线程如下：

<img src="https://ws4.sinaimg.cn/large/006tNc79gy1fliuh03rrqj31co0k0q7f.jpg"></img>

可以看到最近这一次的绘制耗时124ms，拉开看一下具体耗时：

<img src="https://ws3.sinaimg.cn/large/006tNc79gy1fliuoifkdfj31kw0mkwla.jpg"></img>

刚好有1000个绘制，这里会不会就是那1000只鱼，通过查看网页源码，可以确认：

<img src="https://ws4.sinaimg.cn/large/006tNc79gy1fliuurki0sj30pu0gw77e.jpg"></img>

<img src="https://ws4.sinaimg.cn/large/006tNc79gy1fliuust89vj30tk09qgo0.jpg"></img>

javascript是单线程运行的，所以这里无法用到多核，javascript worker技术是让js跑多线程，但是这个网页并没有用到这个技术。

要解决这个问题，要么改网页代码，启用javascript worker技术，这样应该能让帧率提升不少；还有一种办法就是启用chromium的gpu光栅化技术，即不调用skia做2d绘制，直接用gpu来绘制，但是目前这个技术缺陷较多，会导致某些场景下闪屏。
