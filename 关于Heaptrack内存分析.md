# Heaptrack 堆内存分析指南

本文介绍如何使用 Heaptrack 分析 Linux C/C++ 程序的动态堆内存，重点说明 `heaptrack_gui` 图形界面中每个页面、指标和操作的含义，并给出当前项目在 Linux x86_64/WSL2 和 SSD202 ARM 设备上的推荐工作流。

> 工具的官方 GUI 名称是 `heaptrack_gui`，不是 `heaptrack_ui`。不同发行版打包的 Heaptrack 版本可能略有差异，界面文字和可用页签以本机版本为准。

## 1. Heaptrack 能解决什么问题

Heaptrack 会跟踪程序中的动态内存分配和释放事件，并为这些事件记录调用栈。它适合回答以下问题：

- 程序动态堆内存的峰值是多少；
- 峰值出现在哪个时间点；
- 哪些调用路径对堆峰值贡献最大；
- 程序结束时哪些动态内存仍未释放；
- 哪些函数频繁调用分配函数；
- 哪些位置产生大量“分配后立即释放”的临时分配；
- 某个操作前后，堆内存、分配次数和泄漏量发生了什么变化；
- 两次运行或两个版本之间的动态堆行为有什么差异。

Heaptrack 主要拦截 `malloc`、`calloc`、`realloc`、`free`、C++ `new/delete` 等分配函数，并记录分配大小、释放事件和调用栈。它不是完整的进程内存或 CPU 分析器。

### 1.1 Heaptrack 不能直接统计什么

Heaptrack 默认不能完整统计：

- 线程栈和普通局部变量；
- 可执行文件的 `.text`、`.rodata`、`.data`、`.bss`；
- 绕过 `malloc` 的自定义内存池内部使用情况；
- 直接使用 `mmap()` 获得的全部映射；
- 共享库、文件映射、显存、设备驱动和硬件缓冲区占用；
- CPU 热点、锁竞争和调度延迟；
- 在 attach 之前已经存在的动态分配；
- 子进程的完整分配行为。

因此，Heaptrack 的 `peak heap memory consumption` 与操作系统看到的 RSS/PSS 不会相等。分析完整进程内存还需要结合：

```bash
grep -E 'VmRSS|VmHWM|RssAnon|RssFile|RssShmem|Threads' /proc/<PID>/status
cat /proc/<PID>/smaps_rollup
pmap -x <PID>
```

CPU 热点应使用 `perf`，非法内存访问应使用 ASan 或 Valgrind Memcheck。

## 2. 当前项目需要特别注意的内存

当前项目是 C11、LVGL、FFmpeg、Baresip 和多线程 Linux 程序，支持 Linux x86_64/WSL2 和 SSD202 ARM Linux。

### 2.1 LVGL 内置内存池不会完整显示为 Heaptrack 分配

`lvgl_cfg/lv_conf.h` 当前配置：

```c
#define LV_MEM_CUSTOM 0
#define LV_MEM_SIZE (40U * 1024U * 1024U)
```

这表示 LVGL 使用约 40 MiB 的内置静态内存池。该内存池本身不是通过一次普通业务 `malloc()` 建立，因此：

- Heaptrack 不会把 LVGL 池内部每次 `lv_mem_alloc()` 都当作 libc 堆分配；
- LVGL 内存池可能反映在 ELF 的 `.bss`、匿名页和 RSS 中；
- Heaptrack 的堆峰值可能明显小于进程 RSS；
- 分析 LVGL 对象和碎片还应结合 `lv_mem_monitor()` 或 `LV_USE_MEM_MONITOR`。

### 2.2 FFmpeg、OpenSSL 和第三方库

Heaptrack 可以看到 `av_malloc()`、`CRYPTO_zalloc()` 等最终调用到动态分配器的路径，但如果第三方动态库已被 strip，可能只能看到导出符号或 `<unresolved function>`。

### 2.3 设备和硬件缓冲区

SSD202 的 MI SDK、显示、解码器和设备节点可能使用：

- 内核缓冲区；
- DMA/连续物理内存；
- 设备驱动管理的帧池；
- `mmap()` 映射；
- 硬件解码器内部资源。

这些内容不能仅凭 Heaptrack 判断，应同时查看平台 SDK 统计、`/proc`、驱动日志和系统内存。

## 3. 安装与检查

Ubuntu/Debian 常见安装方式为：

```bash
sudo apt update
sudo apt install heaptrack heaptrack-gui
```

部分发行版将 CLI 和 GUI 放在同一个 `heaptrack` 包中。如果找不到 `heaptrack-gui`，先执行：

```bash
apt-cache search heaptrack
```

确认工具：

```bash
heaptrack --version
heaptrack --help
heaptrack_gui --version
heaptrack_print --version
```

GUI 依赖 Qt/KDE Frameworks。官方构建中，`Consumed`、`Allocations`、`Temporary Allocations` 和 `Sizes` 图表还依赖 KDiagram/KChart；发行版构建缺少这些可选依赖时，相应页签可能不存在。

## 4. 准备适合分析的程序

### 4.1 调试符号

为了得到函数名、源码文件和行号，建议使用：

```text
-g
-Og
-fno-omit-frame-pointer
```

并确保程序和要分析的动态库没有被 strip。

检查主程序：

```bash
file ./app
readelf -S ./app | grep -E 'debug_info|debug_line|symtab'
```

理想结果应包含：

```text
with debug_info, not stripped
.debug_info
.debug_line
.symtab
```

当前项目的 Linux Debug 构建命令为：

```bash
cmake --preset linux-amd64-debug
cmake --build --preset linux-amd64-debug --parallel 8
```

默认产物为：

```text
$HOME/.cache/hdl_lv_4_inch_screen/build/linux_amd64/debug/app
```

SSD202 Debug 构建命令为：

```bash
cmake --preset ssd202-debug
cmake --build --preset ssd202-debug --parallel 8
```

默认产物为：

```text
$HOME/.cache/hdl_lv_4_inch_screen/build/ssd202/debug/app
```

### 4.2 保留完全一致的二进制和动态库

符号解析必须使用采集时实际运行的版本。建议每份 profile 同时归档：

- Heaptrack 数据文件；
- 对应的未剥离 `app`；
- Build ID；
- 对应动态库或独立 debug 文件；
- Git 提交号和构建配置；
- 测试步骤和运行时长。

可记录 Build ID：

```bash
readelf -n ./app | grep -A1 'Build ID'
```

## 5. 采集方式

## 5.1 从程序启动开始采集，推荐

从启动开始采集可以看到初始化、稳定运行和正常退出的完整生命周期。

当前项目依赖相对资源路径，应先进入项目根目录：

```bash
cd /data/code/source
mkdir -p /tmp/heaptrack-source

heaptrack \
    --record-only \
    -o /tmp/heaptrack-source/app \
    -- /root/.cache/hdl_lv_4_inch_screen/build/linux_amd64/debug/app
```

说明：

- `--` 是 Heaptrack 参数和被测程序之间的分隔符，必须与程序路径之间有空格；
- `--record-only` 表示只完成采集和解释，不自动打开 GUI；
- `-o` 指定输出文件前缀；
- Heaptrack 会根据构建能力追加 `.gz` 或 `.zst`；
- 不同版本的默认输出目录不同，应以终端打印的 `heaptrack output will be written to ...` 为准。

如需向应用传递参数：

```bash
heaptrack --record-only -o /tmp/heaptrack-source/app -- ./app --config test.json
```

`--` 后的全部内容都会作为被测程序及其参数。

### 输出文件名占位符

官方脚本支持在 `-o` 中使用：

- `%h`：主机名；
- `%p`：Heaptrack 启动脚本的进程标识，具体结果以工具输出为准。

例如：

```bash
heaptrack --record-only -o /tmp/heaptrack/%h/%p/app -- ./app
```

父目录不存在时，官方脚本会尝试创建。

## 5.2 Attach 到已运行进程

```bash
heaptrack \
    --record-only \
    -o /tmp/heaptrack-source/attached-app \
    --pid <PID>
```

Attach 模式需要 GDB，并通过 GDB 向目标进程注入 Heaptrack。官方明确警告该模式不稳定，注入或停止跟踪可能导致应用崩溃。

还应注意：

- 只能看到 attach 之后发生的分配和释放；
- 无法重建 attach 之前已经存在的分配；
- attach 结果不适合直接判断整个进程生命周期的泄漏；
- 目标程序通常需要链接 `libdl`；
- Linux Yama `ptrace_scope` 可能阻止 attach；
- 不应为了方便在生产设备上永久降低 ptrace 安全限制。

能重新启动程序时，优先使用从启动开始采集。

## 5.3 控制调用栈深度

默认每次分配最多记录 64 层调用栈。高频分配程序中，可以降低深度以减少开销：

```bash
heaptrack \
    --unwind-depth 32 \
    --record-only \
    -o /tmp/heaptrack-source/app \
    -- ./app
```

取值范围为 1～64。层数过小会丢失业务调用者，建议先使用 32 或 64，只在开销确实过高时继续降低。

该选项只对新启动的进程生效，不适用于 `--pid` attach。

## 5.4 与 ASan 一起运行

官方提供：

```bash
heaptrack --asan --record-only -o /tmp/heaptrack-source/app -- ./app
```

限制：

- 只支持 GCC 动态链接的 `libasan.so`；
- 官方说明当前不支持 Clang ASan 运行库；
- `--asan` 隐含启用实验性的 inject 拦截方式；
- ASan 和 Heaptrack 都会增加开销，结果时序与内存峰值可能明显偏离生产环境。

通常应分别进行 Heaptrack 和 ASan 实验，只有确有需要时才叠加。

## 5.5 正常结束采集

判断泄漏时必须让程序执行完整反初始化。推荐：

```bash
kill -TERM <PID>
```

等待应用完成：

- 停止定时器和回调；
- 停止播放器和解码线程；
- 清空帧队列；
- 停止 SIP、MQTT 和网络线程；
- 销毁 LVGL 页面和对象；
- 关闭数据库和文件；
- 释放 OpenSSL、FFmpeg 等上下文。

不要用以下方式生成正式泄漏结论：

```bash
kill -9 <PID>
```

崩溃、强制关闭终端或在业务仍活动时终止，也会让本可在退出阶段释放的对象显示为 `Leaked`。

## 6. SSD202 嵌入式采集

官方推荐在缺少完整符号的嵌入式设备上先记录 raw 数据，再在具有 SDK、sysroot 和未剥离二进制的开发机上解释。

## 6.1 设备端采集 raw profile

设备端 Heaptrack 必须为目标 ARM 架构构建，并与设备 libc 兼容。

```bash
cd /data/sourcet
mkdir -p /data/sourcet/tmp

heaptrack \
    --raw \
    --record-only \
    -o /data/sourcet/tmp/heaptrack.app \
    -- ./app
```

输出通常类似：

```text
/data/sourcet/tmp/heaptrack.app.raw.zst
```

或：

```text
/data/sourcet/tmp/heaptrack.app.raw.gz
```

`--raw` 只跳过设备端符号解释，并不能消除分配拦截和调用栈采集开销。

将 raw 文件复制到开发机：

```bash
scp root@<设备地址>:/data/sourcet/tmp/heaptrack.app.raw.zst /tmp/
```

## 6.2 开发机解释 raw profile

示例：

```bash
heaptrack --interpret /tmp/heaptrack.app.raw.zst \
    --sysroot /opt/toolchain/gcc-arm-8.2-2018.08-x86_64-arm-linux-gnueabihf/arm-linux-gnueabihf/libc \
    --extra-paths \
        /root/.cache/hdl_lv_4_inch_screen/build/ssd202/debug \
        /data/code/source/lib/ssd20x/lib
```

参数含义：

- `--sysroot`：目标系统根目录，用于还原目标机 `/lib`、`/usr/lib` 等路径；
- `--extra-paths`：侧载程序或不在 sysroot 中的未剥离二进制/动态库目录；
- `--debug-paths`：保存独立 debug symbol 文件的额外目录。

如果有单独的调试符号目录：

```bash
heaptrack --interpret /tmp/heaptrack.app.raw.zst \
    --sysroot <目标sysroot> \
    --extra-paths <app构建目录> <项目动态库目录> \
    --debug-paths <独立debug文件目录>
```

解释完成后会生成非 raw 的 `.gz` 或 `.zst` 文件，再打开：

```bash
heaptrack --analyze /tmp/heaptrack.app.zst
```

### 6.3 为什么嵌入式 profile 建议 raw/interpret 分离

如果直接在设备端解释：

- 设备上通常没有未剥离二进制和第三方调试符号；
- sysroot 和独立 debug 文件不完整；
- 大量调用栈会成为 `<unresolved function>`；
- 解释过程会消耗设备 CPU、内存和存储空间。

raw/interpret 分离可以让开发机使用完整工具链、sysroot 和构建产物解析 ARM 地址。

## 7. 打开和分析数据

## 7.1 GUI

```bash
heaptrack_gui /tmp/heaptrack-source/app.zst
```

或者：

```bash
heaptrack --analyze /tmp/heaptrack-source/app.zst
```

`heaptrack --analyze` 会优先打开 `heaptrack_gui`，GUI 不可用时回退到 `heaptrack_print`。

## 7.2 命令行报告

```bash
heaptrack_print /tmp/heaptrack-source/app.zst | less
```

默认主要输出：

- `MOST CALLS TO ALLOCATION FUNCTIONS`；
- `PEAK MEMORY CONSUMERS`；
- `MOST TEMPORARY ALLOCATIONS`。

显示泄漏并扩大结果数量：

```bash
heaptrack_print \
    --print-leaks true \
    --peak-limit 30 \
    --sub-peak-limit 10 \
    /tmp/heaptrack-source/app.zst | less
```

只看调用栈中包含指定函数的分配：

```bash
heaptrack_print \
    --filter-bt-function player \
    --print-leaks true \
    /tmp/heaptrack-source/app.zst | less
```

生成 FlameGraph.pl 输入：

```bash
heaptrack_print \
    --flamegraph-cost-type leaked \
    --print-flamegraph /tmp/heaptrack-leaked.stacks \
    /tmp/heaptrack-source/app.zst
```

`--flamegraph-cost-type` 可选：

- `allocations`；
- `temporary`；
- `leaked`；
- `peak`。

还可以输出 Massif 兼容数据或分配大小直方图，具体参数以本机 `heaptrack_print --help` 为准。

## 8. GUI 打开页

未加载数据时，GUI 打开页通常包含以下项目。

### 8.1 Profile Data

主分析文件，通常命名为：

```text
heaptrack.<应用名>.<PID>.gz
heaptrack.<应用名>.<PID>.zst
```

raw 文件必须先 `--interpret`，不能直接当作完整分析文件使用。

### 8.2 Compare to

可选基线 profile。启用后，GUI 使用：

```text
主 profile 成本 - 基线 profile 成本
```

适合比较：

- 修改前与修改后；
- 空闲基线与执行场景；
- 重复 1 次与重复 20 次；
- 旧版本与新版本。

差分结果可能出现负数：

- 正数：主 profile 比基线更多；
- 负数：主 profile 比基线更少；
- 接近 0：变化不明显。

差分只有在两次运行的操作、时长、输入和退出方式可比较时才有意义。官方 GUI 的 Flame Graph 在差分模式下可能被禁用。

### 8.3 Suppressions

可选的 LSAN 风格泄漏抑制文件，用于隐藏已确认且暂不需要处理的已知泄漏。不要用 suppression 掩盖尚未查明的项目泄漏。

## 9. Summary 页

`Summary` 是总览页，分为顶部统计和四个热点列表。

## 9.1 顶部左侧

### debuggee

被分析程序及其参数。Attach 模式会标记为 attached。

### total runtime

采集的总运行时间。

启用时间过滤后，还会显示：

- 原始总运行时间；
- 过滤开始时间；
- 过滤结束时间；
- 过滤区间长度。

### total system memory

采集系统的物理内存总量，用于提供环境背景。它不是应用可独占内存，也不是应用实际使用量。

## 9.2 顶部中间

### calls to allocation functions

调用受跟踪内存分配函数的总次数，并显示平均每秒次数。

高分配次数不等于泄漏，但可能带来：

- 分配器 CPU 开销；
- 多线程分配锁竞争；
- 堆碎片；
- UI 或音视频时延抖动。

### temporary allocations

官方定义非常具体：一次分配之后，下一次相关内存事件就是该对象的释放，中间没有发生其他分配，则记为 temporary allocation。

它不是泛指“生命周期短”，而是“分配后直接跟随释放”。界面还显示：

- 临时分配总次数；
- 占全部分配的百分比；
- 平均每秒临时分配次数。

大量临时分配常见于：

- 字符串格式化；
- 高频日志；
- JSON 临时节点；
- 每帧创建/销毁缓冲区；
- 加密库临时对象；
- 事件循环中反复构造小对象。

## 9.3 顶部右侧

### peak heap memory consumption

Heaptrack 跟踪到的动态堆最大同时存活量，并显示峰值出现时间。

该指标考虑释放事件，不是“历史累计申请字节数”。它不包含 Heaptrack 看不到的栈、静态区、自定义内存池内部、设备缓冲区等。

### peak RSS including heaptrack overhead

采集期间观测到的进程 RSS 峰值，明确包含 Heaptrack 自身开销。

不能把它当作生产版本真实 RSS，也不能将它与 `peak heap` 的差值直接解释为泄漏。要测真实 RSS，应在不使用 Heaptrack 的情况下运行同一场景并读取 `/proc/<PID>`。

### total memory leaked

完整 profile 中，程序结束时仍未释放的受跟踪字节数。

它的正确解释取决于退出方式：

- 正常完整退出后仍存活：更可能是真泄漏或生命周期未关闭；
- `kill -9`、崩溃、强制中止：只能称为“结束时仍存活”，不能直接判定泄漏；
- 单例、进程级缓存和第三方库全局对象：可能设计为进程结束时由操作系统回收，但仍会显示为未释放。

如果应用了 suppression，还会显示被抑制的泄漏字节数。

### memory consumption delta

当 GUI 过滤到一个时间区间时，Summary 不再把区间末尾值称为完整生命周期泄漏，而显示区间内存变化量：

```text
区间结束存活堆 - 区间开始存活堆
```

正数表示该时间段内存净增长，负数表示净释放。它非常适合分析“进入页面到退出页面”或“开始播放到停止播放”的内存变化。

## 9.4 Peak Contributions

列出对全局动态堆峰值贡献最大的函数/位置。

用途：

- 找最大工作集；
- 找大帧、大缓存和批量对象；
- 找峰值发生时仍存活的主要内存；
- 判断高峰来自 UI、视频、网络还是数据库。

`Peak` 不是某函数历史累计申请量，而是该位置对最大同时存活堆的贡献，并考虑释放。

聚合调用树中的父节点和子节点可能代表同一批分配在不同栈层级上的归属，不能把父子行简单相加。

## 9.5 Largest Memory Leaks

列出结束时未释放字节数最多的函数/位置。

摘要通常显示靠近分配点的函数，例如 `calloc` 包装器、`av_malloc` 或 `CRYPTO_zalloc`。真正负责资源生命周期的往往是其上层调用者，应通过 `Bottom-Up`、`Caller / Callee` 和 `Stacks` 找到项目函数。

## 9.6 Most Memory Allocations

列出调用分配函数次数最多的位置。

适合查：

- 高频小对象；
- 频繁 JSON 构造；
- 日志格式化；
- 每帧/每 tick 分配；
- 可以复用的临时缓冲区。

高频分配即使全部释放，也可能造成 CPU 和碎片问题。

## 9.7 Most Temporary Allocations

列出 temporary allocation 次数最多的位置。

优先检查“高频且大部分都是 temporary”的项目调用路径。例如一个函数共有 5000 次分配，其中 4800 次是 temporary，通常说明它在反复申请和立即释放可复用对象。

## 10. Bottom-Up 页

Bottom-Up 从靠近分配调用点的位置出发，向上展开调用者。

示意：

```text
malloc/av_malloc/项目分配包装器
└── 创建对象或缓冲区的函数
    └── 业务函数
        └── 页面、定时器或线程入口
```

它最适合回答：

> 这批分配最终是谁发起的？

例如搜索 `av_malloc` 后逐层展开，直到看到：

```text
av_malloc
└── av_frame_get_buffer
    └── driver_decode_frame
        └── player_refresh
```

### 10.1 过滤框

通常提供：

- `filter by function`：按函数名过滤；
- `filter by file`：按源码文件过滤；
- `filter by module`：按可执行文件或动态库过滤。

适合输入：

```text
player
src/drivers/video
app
libavutil
```

### 10.2 表格列

#### Location

发生分配或调用栈中的函数位置。悬停通常可看到：

- 完整函数名；
- 模块名；
- 模块路径；
- 该位置各项成本及占比；
- 部分调用栈摘要。

缺少调试信息时会出现 `<unresolved function>` 或只显示地址/模块。

#### Peak

该位置对全局动态堆峰值的贡献字节数，考虑释放。

#### Leaked

完整 profile 中，该位置分配且结束时未释放的字节数。时间过滤后应结合 `memory consumption delta` 理解为过滤区间的净变化贡献，而不是完整进程泄漏结论。

#### Allocations

从该位置触发分配函数的次数。

#### Temporary

从该位置触发的 temporary allocation 次数。官方定义是分配后直接跟随对应释放，中间没有其他分配事件。

### 10.3 使用建议

- 查泄漏：按 `Leaked` 降序；
- 查峰值：按 `Peak` 降序；
- 查抖动：按 `Temporary` 降序；
- 查分配热点：按 `Allocations` 降序；
- 不要只停留在 `malloc`、`calloc`、`av_malloc`；
- 一直展开到出现项目自己的资源所有者或业务入口。

## 11. Top-Down 页

Top-Down 从调用栈入口向下展开，逐步走向分配位置。

示意：

```text
main/线程入口/事件循环
└── 业务模块
    └── 页面或播放器
        └── 分配函数
```

它最适合回答：

> 某个入口、线程或业务场景总共带来了多少分配？

常见用途：

- 从 `main` 看初始化阶段；
- 从线程入口看 MQTT/SIP/播放器线程；
- 从 UI timer 看刷新逻辑；
- 从页面创建函数看其全部下游分配。

Top-Down 的过滤框和 `Location/Peak/Leaked/Allocations/Temporary` 列含义与 Bottom-Up 相同，但树的展开方向相反。

### Bottom-Up 与 Top-Down 如何选择

| 问题 | 推荐视图 |
|---|---|
| `av_malloc` 的上层是谁 | Bottom-Up |
| 哪个业务入口导致最多内存 | Top-Down |
| 谁应负责释放某对象 | Bottom-Up 后切 Caller/Callee |
| 页面创建的全部下游分配 | Top-Down |
| 某低层包装器被哪些路径调用 | Bottom-Up |

## 12. Stacks 面板

在 Bottom-Up 或 Top-Down 选择聚合行时，GUI 会显示 `Stacks` 面板。

同一个聚合函数可能对应多条不同调用栈，因此面板包含：

- `Selected Stack`：当前选中的调用栈编号；
- `/ N`：该聚合项包含的调用栈总数；
- 栈帧列表：具体函数、文件和行号。

用途：

- 查看被聚合热点的完整调用链；
- 区分同一个分配函数的不同业务来源；
- 找到真正负责释放资源的上层函数；
- 复制栈帧用于缺陷记录。

如果一个函数有很多栈，不要只看第 1 条，应切换多个 `Selected Stack`，优先检查成本最大的路径。

## 13. Caller / Callee 页

该页面把调用关系按函数聚合，适合围绕一个函数分析“谁调用它”和“它调用了谁”。

### 13.1 主表列

#### Location

被聚合的函数。官方定义中，它代表调用分配函数的父符号；缺少调试信息时可能无法解析。

#### Allocations (Self)

该函数直接触发分配函数的次数，不包含下游函数。

#### Temporary (Self)

该函数直接产生的 temporary allocation 次数。

#### Peak (Self)

直接源自该函数的分配对峰值的贡献。

#### Leaked (Self)

直接在该函数位置分配且未释放的字节数。

#### Allocations (Incl.)

该函数自身及所有下游调用共同产生的分配次数。

#### Temporary (Incl.)

该函数自身及下游调用共同产生的 temporary allocation 次数。

#### Peak (Incl.)

该函数自身及下游调用共同贡献的峰值字节数。

#### Leaked (Incl.)

该函数自身及下游调用共同关联的未释放字节数。

### 13.2 Self 与 Inclusive 的区别

假设：

```text
page_create
└── widget_create
    └── calloc
```

可能出现：

- `page_create` 的 Self 很小或为 0；
- `page_create` 的 Inclusive 很大；
- `widget_create` 的 Self 或 Inclusive 较大。

因此：

- Self 高：函数本身直接分配多；
- Inclusive 高而 Self 低：主要成本来自下游；
- 两者都高：函数自身和下游都产生大量分配。

父子函数的 Inclusive 成本可能包含同一批分配，不能将它们直接相加。

### 13.3 下方三个视图

选择主表函数后，通常显示：

- `Caller`：哪些函数调用了当前函数；
- `Callee`：当前函数调用了哪些函数；
- Source/Location：成本映射到哪些源码文件和行。

双击 Caller 或 Callee 可以继续导航到该函数。源码路径存在且配置了 Code Navigation 时，可以从源码位置打开编辑器。

### 13.4 典型用法

1. 在 Summary 或 Bottom-Up 选中 `av_malloc` 上层项目函数；
2. 右键选择 `View Caller/Callee`；
3. 查看 Caller，确认由页面、timer、解码线程还是网络回调触发；
4. 查看 Callee，确认具体分配路径；
5. 对比 Self/Inclusive，判断资源所有者和优化位置。

## 14. Flame Graph 页

火焰图将调用树按成本宽度可视化。

基本规则：

- 矩形越宽，所选成本越大；
- 父框包含下游聚合成本；
- 上下方向表示调用栈层级；
- 不同颜色主要用于区分，不代表固定模块或严重程度；
- 单击某个框可放大该分支；
- 右键可进入 Caller/Callee 或复制描述；
- `Esc` 可重置视图；
- 前进/后退按钮用于浏览放大历史。

## 14.1 Cost Source

### Memory Peak

按对动态堆峰值的贡献绘制。适合找：

- 大对象；
- 帧缓冲；
- 峰值时同时存活的缓存；
- 页面创建期间的最大工作集。

### Leaked

按结束时未释放字节数绘制。适合找泄漏调用链。只有正常退出的完整 profile 才能作为强泄漏证据。

### Allocations

按分配次数绘制。适合找高频分配热点，即使这些分配最终都释放。

### Temporary Allocations

按 temporary allocation 次数绘制。适合找分配后立即释放的内存抖动。

## 14.2 Bottom-Up View

未勾选时默认是 Top-Down 火焰图，从入口向分配点看。

勾选后切换为 Bottom-Up，以分配位置为起点聚合调用者。查某个底层分配函数的业务来源时更方便。

## 14.3 Collapse Recursion

折叠函数递归调用产生的重复栈帧。

- 勾选：递归函数合并，图更紧凑；
- 取消：每层递归单独显示，适合分析递归深度和每层成本。

## 14.4 Cost Threshold

设置相对总成本的显示阈值。低于该百分比的节点不绘制，以加快大型 profile 的火焰图生成并降低 GUI 内存占用。

- profile 很大或 GUI 卡顿：提高阈值；
- 需要看小热点：降低阈值；
- 需要完整细节：设为 0，但可能显著增加生成时间和 GUI 内存。

## 14.5 Search

按函数名或模块名进行不区分大小写搜索：

- 直接匹配项高亮；
- 不匹配项变淡；
- 下方显示匹配成本及其占总成本比例。

可搜索：

```text
player
av_malloc
logger_write
libcrypto
```

## 15. Consumed 页

该页显示动态堆随时间的变化，横轴为运行时间，纵轴为内存字节数。

它最适合回答：

- 峰值在什么时候发生；
- 进入页面后内存是否回落；
- 播放停止后帧缓冲是否释放；
- 内存是一次性增长后稳定，还是持续阶梯式增长；
- 某段操作造成多少净增长。

### 15.1 Total cost graph

显示整个程序的总动态堆曲线。

### 15.2 Detailed cost graph

显示主要函数/位置贡献的堆曲线，通常以堆叠方式展示。

### 15.3 Show legend

显示曲线颜色对应的函数或数据集。

### 15.4 Show total cost graph

显示或隐藏总成本曲线。

### 15.5 Show detailed cost graph

显示或隐藏主要热点的详细堆叠曲线。

### 15.6 Stacked diagrams

控制显示多少条主要详细数据集。数值越大，显示的热点函数越多，但图例和曲线也越复杂。

### 15.7 Export As

将图表导出为：

- PNG；
- JPEG；
- TIFF；
- SVG。

SVG 适合放入设计文档或缺陷报告，放大后仍清晰。

## 16. Allocations 页

该页显示累计分配次数随时间的变化。

观察重点：

- 曲线斜率越陡，单位时间分配越频繁；
- 空闲时仍持续线性增长，说明后台有周期性分配；
- 某个操作出现陡升，说明该场景创建了大量对象；
- 操作结束后曲线不会下降，因为它统计的是累计分配次数，而不是当前存活对象数。

适合定位：

- 高频 timer；
- 日志循环；
- JSON 解析；
- 网络消息处理；
- 每帧临时对象；
- 重复页面重建。

工具栏、图例、详细曲线、导出和时间选择操作与 `Consumed` 相同。

## 17. Temporary Allocations 页

该页显示累计 temporary allocation 次数随时间的变化。

temporary 的官方定义仍是：分配后直接跟随其对应释放，中间没有其他分配事件。

观察重点：

- 斜率持续较高：存在稳定的临时分配抖动；
- 只在页面切换时陡升：页面构建有大量临时对象；
- 播放期间斜率很高：可能每帧申请/释放缓冲；
- 与 `Allocations` 曲线形状接近：大部分分配都很短暂。

## 18. 时间区间选择和过滤

在 `Consumed`、`Allocations` 或 `Temporary Allocations` 图表中：

1. 按住鼠标左键横向拖动；
2. 选择一个时间区间；
3. Tooltip 显示开始、结束和 Delta；
4. 右键选择 `Filter In On Selection`；
5. GUI 重新解析该时间区间。

三个图表的时间选择会同步。

过滤后：

- Summary 显示过滤范围；
- `total memory leaked` 改为 `memory consumption delta`；
- Bottom-Up、Top-Down、Caller/Callee 按区间重新计算；
- 可以准确分析某一次页面进入/退出或某段播放过程。

恢复完整 profile：

```text
Filter → Reset Filter
```

或在图表右键选择 `Reset Filter`。

### 18.1 推荐的时间过滤实验

在日志中记录操作时间，或按固定步骤执行：

```text
0～10 秒：启动并稳定
10～20 秒：进入页面
20～40 秒：执行目标操作
40～50 秒：退出页面并等待回收
50 秒后：正常退出
```

然后分别过滤：

- 10～20 秒：页面创建成本；
- 20～40 秒：业务运行成本；
- 40～50 秒：页面销毁后的净变化。

## 19. Sizes 页

`Sizes` 是请求分配大小的直方图：

- 横轴：Requested Allocation Size；
- 纵轴：Number of Allocations。

它回答：

- 程序主要申请多大的对象；
- 是否存在海量小对象分配；
- 是否频繁申请固定大小帧或消息；
- 大对象是否来自少数调用路径。

悬停详细柱时通常显示：

- 该大小区间的分配次数；
- 来源函数；
- 累计申请字节数；
- 平均每次分配大小。

典型判断：

- 大量 16～128 B：小对象/节点/字符串碎片；
- 固定几 KiB：网络包、JSON 缓冲或日志缓冲；
- 数百 KiB～数 MiB：图像、音视频帧或大缓存。

Sizes 统计请求大小和次数，不代表这些对象同时存活，也不能单独证明泄漏。

## 20. 图表上的时间选择数值

拖动选择区间后，Tooltip 会显示：

- `Start`：区间开始时间和值；
- `End`：区间结束时间和值；
- `Delta`：结束值减开始值。

不同页签中 Delta 含义为：

| 页签 | Delta 含义 |
|---|---|
| Consumed | 该时间段动态堆净增长/净释放字节数 |
| Allocations | 该时间段新增分配调用次数 |
| Temporary Allocations | 该时间段新增 temporary allocation 次数 |

## 21. Suppression 泄漏抑制

Heaptrack 支持 LSAN 风格的一行一条 suppression：

```text
# 已确认的第三方库进程级缓存
leak:third_party_cache_init

# 通配符
leak:*known_library_cache*
```

官方实现支持：

- 空行；
- `#` 开头注释；
- `leak:` 前缀；
- `*` 通配符；
- `^` 起始锚点；
- `$` 结束锚点。

例如：

```text
leak:^known_init_function$
leak:*third_party_namespace*
```

### 21.1 GUI suppression

在打开 profile 时通过 `Suppressions` 选择文件。Summary 会显示：

- 未抑制的泄漏；
- suppressed 字节数；
- 展开的 Suppressions 区域中可查看匹配次数和被抑制字节数。

### 21.2 内置与嵌入 suppression

官方 GUI 设置中通常有：

- `Disable Embedded Suppressions`：禁用程序通过 `__lsan_default_suppressions()` 嵌入的 suppression；
- `Disable Builtin Suppressions`：禁用 Heaptrack 对部分 libc、动态链接器、GLib 已知分配的内置 suppression。

排查第三方库或验证 suppression 是否掩盖问题时，可以暂时禁用；日常分析通常保留默认设置。

### 21.3 使用原则

只有满足以下条件才应加入 suppression：

1. 已展开完整调用栈；
2. 已确认不是项目所有权错误；
3. 已确认是第三方库或进程级有意常驻对象；
4. 已记录原因和适用版本；
5. suppression 尽量精确，不使用过宽模式。

## 22. 差分分析

差分比单次 profile 更适合验证优化和回归。

### 22.1 基线与场景

生成空闲基线：

```bash
heaptrack --record-only -o /tmp/heaptrack/baseline -- ./app
```

生成目标场景：

```bash
heaptrack --record-only -o /tmp/heaptrack/scenario -- ./app
```

GUI 中：

```text
Profile Data: scenario
Compare to:   baseline
```

### 22.2 差分实验必须控制变量

尽量保持一致：

- 相同二进制和动态库；
- 相同运行时长；
- 相同初始化等待时间；
- 相同网络输入；
- 相同页面操作次数；
- 相同退出方式；
- 相同系统环境。

否则差异可能只是场景噪声。

### 22.3 如何解释负值

差分是“主 profile 减基线”：

- `Peak = -1 MiB`：主 profile 峰值贡献比基线少约 1 MiB；
- `Allocations = -1000`：主 profile 少约 1000 次分配；
- `Leaked = +200 KiB`：主 profile 多约 200 KiB 未释放/净增长。

## 23. `<unresolved function>` 的原因和处理

常见原因：

- `app` 或 `.so` 被 strip；
- 没有 `-g`；
- profile 与二进制版本不一致；
- GUI/解释器找不到采集时加载的动态库；
- 嵌入式 profile 没有使用正确 `--sysroot`；
- 侧载程序不在 sysroot，也没有通过 `--extra-paths` 提供；
- 独立 debug 文件目录没有通过 `--debug-paths` 提供；
- 厂商库只保留少量导出符号；
- 汇编、JIT 或特殊代码没有普通 DWARF 信息；
- 调用栈展开失败。

### 23.1 处理顺序

1. 检查 `file app` 是否 `with debug_info, not stripped`；
2. 检查 `.debug_info/.debug_line/.symtab`；
3. 确认 profile 和二进制 Build ID 一致；
4. 确认动态库版本一致；
5. SSD202 使用 `--raw` 后在开发机 `--interpret`；
6. 设置正确 `--sysroot`；
7. 为项目程序和侧载库设置 `--extra-paths`；
8. 为独立 debug 文件设置 `--debug-paths`；
9. 检查调用栈中 `<unresolved>` 上下是否还有已解析的项目函数。

即使最底层未解析，上层已解析函数仍可能足以定位业务路径。

### 23.2 debuginfod

官方解释器默认会避免意外使用 debuginfod，防止符号解析产生不可预期的网络延迟。确实需要时，可以按官方说明显式启用：

```bash
export HEAPTRACK_ENABLE_DEBUGINFOD=1
```

启用前应确认网络、隐私、下载量和符号服务器来源。

## 24. 如何判断是否真正泄漏

单次截图不足以证明泄漏，应做可重复实验。

### 24.1 空闲基线

```text
启动
→ 等待初始化完成
→ 空闲 60 秒
→ 正常退出
```

记录：

- Peak；
- Leaked；
- Allocations；
- Temporary；
- 退出时调用栈。

### 24.2 页面一次循环

```text
启动
→ 进入目标页面
→ 操作一次
→ 退出页面
→ 等待异步任务和 timer 停止
→ 正常退出
```

### 24.3 页面重复循环

```text
进入页面
→ 退出页面
→ 重复 20 次
→ 等待回收
→ 正常退出
```

判断：

| 现象 | 常见解释 |
|---|---|
| 第一次增长，后续稳定 | 单例、缓存或延迟初始化 |
| 每次操作近似固定增长 | 高概率生命周期泄漏 |
| 运行时增长，退出页面后回落 | 正常工作集 |
| 停止播放后不回落，重复播放继续增长 | 播放器、帧或队列释放问题 |
| 只有 `kill -9` 时显示大量 Leaked | 不能证明正常退出也泄漏 |
| 正常退出后项目对象仍大量存活 | 需要检查反初始化顺序和所有权 |

### 24.4 泄漏调用栈检查原则

不要停在：

```text
malloc
calloc
utils_calloc
av_malloc
CRYPTO_zalloc
```

这些只是分配执行者。应继续向上找到：

- 谁创建对象；
- 谁保存对象；
- 所有权属于哪个模块；
- 正常释放函数是什么；
- 释放函数是否执行；
- 释放前是否仍有线程、timer 或回调使用对象。

## 25. 当前项目的推荐排查流程

## 25.1 页面内存

1. 启动并等待稳定；
2. 在 `Consumed` 记录进入页面前值；
3. 进入页面；
4. 退出页面并等待 3～5 秒；
5. 重复 20 次；
6. 正常退出；
7. 用时间过滤分析单次和全部循环；
8. 在 Bottom-Up 按 `Leaked` 和 `Peak` 排序；
9. 搜索页面创建、私有数据分配和销毁函数；
10. 同时使用 LVGL `lv_mem_monitor()` 检查内置池。

## 25.2 FFmpeg/视频播放

分别采集：

- 不播放；
- 播放一次后停止；
- 重复播放/停止；
- 播放中强制退出，仅作为对照；
- 播放停止后正常退出。

重点搜索：

```text
av_malloc
av_frame
av_packet
player
publish_frame
video_frame
decode
```

检查配对：

- `av_malloc()` / `av_free()`；
- `av_frame_alloc()` / `av_frame_free()`；
- `av_packet_alloc()` / `av_packet_free()`；
- `av_packet_ref()` / `av_packet_unref()`；
- codec context 打开/关闭；
- swscale/swresample context 创建/释放；
- 队列清空时每个节点的 payload；
- 替换最新帧时旧帧释放；
- timer、回调停止与线程 join 的顺序。

## 25.3 日志和临时分配

在 Summary 和 Flame Graph 选择 `Temporary Allocations`，搜索：

```text
logger_write
printf
vasprintf
cJSON
CRYPTO
```

重点检查：

- 每帧日志；
- 每 10 ms/100 ms timer 日志；
- 不变状态重复打印；
- 每次日志创建临时 JSON 或字符串；
- 可以改为状态变化时记录的路径。

## 25.4 OpenSSL、MQTT、SIP

如果 `CRYPTO_zalloc`、ASN.1 或 TLS 路径显示未释放：

1. 在 Bottom-Up 展开到 MQTT、curl、Baresip 或 SIP 项目调用者；
2. 确认网络线程已退出；
3. 确认连接、证书、SSL context 和库级资源按正确顺序释放；
4. 区分第三方进程级缓存和项目反初始化遗漏；
5. 不要仅凭 `CRYPTO_zalloc` 名称修改 OpenSSL 内部代码。

## 26. 性能开销和结果偏差

Heaptrack 相比 Valgrind Massif 通常开销更低：

- 多线程应用不会因 Heaptrack 而被整体串行化；
- 主要在发生分配时付出采集成本；
- 纯 CPU 计算段受影响较小。

但它仍会改变：

- 分配延迟；
- 线程时序；
- RSS；
- 峰值发生时间；
- 实时音视频表现；
- 生成的 profile 文件大小。

因此：

- `peak RSS including heaptrack overhead` 不能作为生产内存指标；
- 实时性结论应在不带 Heaptrack 的环境复测；
- 采集时只执行目标场景，避免无关长时间运行；
- 超长 profile 应拆成多个短 profile。

官方说明，小时级大型 profile 加载和缩放会很慢，因为数据不能高效随机 seek。建议每隔若干分钟重新采集，而不是生成一个超大文件。

## 27. Heaptrack 与 Valgrind Massif 的区别

Heaptrack 的优势：

- 通常时间和内存开销更低；
- 多线程程序不会被整体串行化；
- 保留分配次数；
- 可以识别 temporary allocation；
- GUI 调用树、时间图和火焰图更适合交互分析。

Massif 的优势：

- 能将某些页面分配/内存池作为堆统计；
- 能统计栈内存；
- 对绕过普通 `malloc` 的内存池场景可能更有帮助。

当前项目的 LVGL 内置池就是典型边界：Heaptrack 适合分析 libc 堆和第三方库动态分配，LVGL 池内部应使用 LVGL 自身监控；必要时再补充 Massif。

## 28. 常见问题

### 28.1 `--/path/to/app` 报 unknown option

错误：

```bash
heaptrack ... --/path/to/app
```

正确：

```bash
heaptrack ... -- /path/to/app
```

`--` 必须是独立参数。

### 28.2 GUI 中大量 `<unresolved function>`

优先检查符号、Build ID、动态库版本、sysroot、`--extra-paths` 和 `--debug-paths`。嵌入式设备优先 raw/interpret 分离。

### 28.3 Leaked 很大，但程序是强制停止的

重新执行一次正常 `SIGTERM` 退出并等待反初始化完成。强制结束结果只能表示“结束时仍存活”。

### 28.4 Peak heap 很小，但 RSS 很大

RSS 还包含：

- LVGL 40 MiB 内置池；
- ELF 静态区；
- 共享库；
- 线程栈；
- 文件映射；
- Heaptrack 自身；
- 设备和第三方库其他映射。

结合 `/proc/<PID>/smaps_rollup` 和 `pmap -x` 分析。

### 28.5 Attach 失败

检查：

- GDB 是否安装；
- 用户权限；
- `ptrace_scope`；
- 目标是否链接 `libdl`；
- 是否处于容器或受限环境。

官方警告 attach 不稳定，能重启时改用启动采集。

### 28.6 ASan 程序启动报 runtime 顺序错误

对 GCC `libasan.so` 构建使用：

```bash
heaptrack --asan -- ./app
```

Clang ASan 当前不在官方支持范围内。

### 28.7 GUI 没有时间图和 Sizes

发行版构建 Heaptrack GUI 时可能缺少可选的 KDiagram/KChart 依赖。检查软件包或重新构建带图表支持的 GUI。

### 28.8 Heaptrack GUI 自身占用很高

大型 profile 包含大量事件、调用栈和时间数据。可以：

- 缩短采集时间；
- 降低 `--unwind-depth`；
- 分场景采集；
- 提高 Flame Graph `Cost Threshold`；
- 使用 `heaptrack_print` 先做命令行筛选。

### 28.9 old gold linker 导致坏调用栈

官方已知问题指出，旧版 gold linker 可能让 libunwind 产生错误回溯，表现为：

- `heaptrack_print` 中调用栈乱码；
- GUI 解析时内存暴涨甚至崩溃。

遇到此问题可尝试使用 `ld.bfd` 重新链接应用和 libunwind。

## 29. 推荐的标准分析记录模板

每次分析建议记录：

```text
测试目的：
Git 提交：
构建 Preset：
Build ID：
Heaptrack 版本：
运行平台：
程序路径：
动态库来源：
采集命令：
退出方式：
测试步骤：
运行时间：
Peak heap：
Peak RSS（含 Heaptrack）：
Leaked/Delta：
Allocations：
Temporary：
最大 Peak 调用栈：
最大 Leaked 调用栈：
最大 Temporary 调用栈：
结论：
后续验证：
```

## 30. 快速分析清单

1. 使用带 `-g` 且未 strip 的程序和库；
2. 从应用启动开始采集；
3. 明确执行一个可重复场景；
4. 使用正常退出，不用 `kill -9`；
5. Summary 先看 Peak、Leaked、Allocations、Temporary；
6. Consumed 找峰值时间和是否回落；
7. Bottom-Up 从分配点追到业务调用者；
8. Top-Down 从场景入口检查全部下游；
9. Caller/Callee 比较 Self 与 Inclusive；
10. Stacks 查看聚合项的具体调用栈；
11. Flame Graph 分别查看 Peak、Leaked、Allocations、Temporary；
12. Sizes 判断小对象抖动和大块分配；
13. 对目标时间区间使用 `Filter In On Selection`；
14. 用 baseline/scenario 做差分；
15. 对 `<unresolved>` 补齐 sysroot 和符号；
16. 用 `/proc` 校验真实 RSS/PSS；
17. 用 LVGL 监控补充内置 40 MiB 内存池；
18. 修复后用相同场景和时长复测。

## 31. 官方资料

本文主要依据以下官方资料和实现整理，访问日期为 2026-11-03：

- KDE Heaptrack 产品页：<https://apps.kde.org/heaptrack/>
- KDE Heaptrack 官方仓库：<https://invent.kde.org/sdk/heaptrack>
- 官方 README：<https://invent.kde.org/sdk/heaptrack/-/raw/master/README.md>
- 官方启动脚本与 CLI 帮助实现：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/track/heaptrack.sh.cmake>
- GUI 主窗口及 Summary/页签实现：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/mainwindow.cpp>
- GUI 布局和控件说明：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/mainwindow.ui>
- Bottom-Up/Top-Down 列定义：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/treemodel.cpp>
- Caller/Callee Self/Inclusive 定义：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/callercalleemodel.cpp>
- 时间图交互和过滤实现：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/chartwidget.cpp>
- Flame Graph 控件和成本定义：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/flamegraph.cpp>
- Sizes 直方图实现：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/gui/histogramwidget.cpp>
- raw 数据符号解释器：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/interpret/heaptrack_interpret.cpp>
- 命令行分析器：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/print/heaptrack_print.cpp>
- suppression 语法实现：<https://invent.kde.org/sdk/heaptrack/-/raw/master/src/analyze/suppressions.cpp>

不同发行版可能提供较旧版本。若本机界面或参数与本文不同，以以下命令输出和本机 GUI Tooltip 为准：

```bash
heaptrack --help
heaptrack_print --help
heaptrack --version
```
