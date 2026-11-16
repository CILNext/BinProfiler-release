# BinProfiler

BinProfiler 是一个基于 Frida 的二进制插桩工具，用于性能分析和切片制作；对应平台的二进制包可从本仓的 GitHub Releases 页面获取。

当前支持范围：

| CPU ISA \\ OS | `Ubuntu 22.04+` | `openEuler 22.03+` |
| --- | --- | --- |
| AArch64 | `trace`、`elfgen` | `trace`、`elfgen` |
| x86_64 | `elfgen` | `elfgen` |

## 目录

- [下载与安装](#下载与安装)
- [环境要求](#环境要求)
- [运行原理](#运行原理)
- [Trace 功能说明](#trace-功能说明)
- [Elfgen 功能说明](#elfgen-功能说明)
- [指令数统计功能说明](#指令数统计功能说明)
- [常见问题与排查](#常见问题与排查)

## 下载与安装

从 [GitHub Releases](https://github.com/CILNext/BinProfiler-release/releases) 页面下载对应系统的压缩包，以及同版本的 sha256 校验文件：

- `binprofiler-ubuntu-22.04-aarch64.tar.gz`（Ubuntu 22.04+）
- `binprofiler-openeuler-22.03-aarch64.tar.gz`（openEuler 22.03+）
- `binprofiler-vX.Y.Z-sha256.txt`（与压缩包放在同一目录，用于校验）

```sh
# 1. 校验压缩包。sha256 文件里列出了两个系统的包，
#    --ignore-missing 表示只校验本次实际下载的压缩包。
sha256sum -c --ignore-missing binprofiler-vX.Y.Z-sha256.txt

# 2. 解压。注意：tar 包内没有顶层目录，直接解压会把文件释放到当前目录，
#    建议先建一个单独目录。
mkdir binprofiler-vX.Y.Z
tar -xzf binprofiler-ubuntu-22.04-aarch64.tar.gz -C binprofiler-vX.Y.Z

# 3. 确认包内包含两个文件
ls binprofiler-vX.Y.Z
# binprofiler  libbinprofiler-agent.so
```

`binprofiler` 默认从自身所在目录加载 `libbinprofiler-agent.so`，两个文件需放在同一目录；如需分离存放，用 `--agent` 指定路径（见 Trace 功能说明）。

## 环境要求

- **架构与系统**：见开头的支持范围表。`trace` 采集仅支持 AArch64（x86_64 构建不含 trace 模块，运行 `trace` 子命令会直接报错）；`elfgen` 也可在 x86_64 Linux 上离线运行。
- **`trace` / 指令数统计**：需要对目标进程的 `ptrace` 附加权限；受 `ptrace_scope` 限制时的配置方法见[运行原理](#运行原理)。
- **回放切片 ELF**：回放环境的 SVE Vector Length 需与采集环境一致（见 SVE Vector Length 说明）；若切片要在 Linux 7.0 之前的内核上运行，生成时需使用 `--merge-elf`（见 Elfgen 功能说明）。

## 运行原理

`binprofiler` 负责附加到目标进程、注入 `agent`，并通过控制通道下发启动参数和任务请求；`agent` 常驻于目标进程内，执行采集并回传结果。

在 Linux 上 `agent` 的注入依赖 `ptrace` 权限。如果本机启用了较严格的 `ptrace_scope` 限制，即使是同用户进程，也可能无法附加成功。

使用 `trace` 采集前建议确认：

- 当前用户对目标进程具备附加权限
- 目标架构为 AArch64
- 运行环境允许通过 `ptrace` 注入动态库
  - 临时放宽限制：`echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope`
  - 持久配置：在 `/etc/sysctl.d/10-ptrace.conf` 中写入
    `kernel.yama.ptrace_scope = 0`，然后重新加载 sysctl 配置

生产环境或共享机器上不建议长期放宽 `ptrace_scope`；如果只做一次采集，优先使用临时配置，采集结束后再恢复原值。

## Trace 功能说明

`trace` 子命令用于采集指定进程中指定线程的动态指令流。采集结果可通过 `elfgen`
生成可执行 ELF。基本调用形式如下：

```sh
./binprofiler [--output-dir <dir>] trace <pid> [trace-options]
```

默认情况下：

- 输出目录是 `./out`
- 未指定线程时，默认抓取主线程，也就是 Linux 上通常 `tid == pid`
- `binprofiler` 默认从自身所在目录加载 `libbinprofiler-agent.so`

如需显式指定 agent 路径：

```sh
./binprofiler --agent /path/to/libbinprofiler-agent.so trace <pid>
```

### 常见用法

```sh
# Trace 成功后自动生成 ELF。默认采集 5,000 万条指令，
# 并优先将指定的 End PC 作为切片结束位置。
./binprofiler --output-dir /tmp/binprofiler-replay trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf
```

<details>
<summary>选项说明与 Trace 输出</summary>

### 选项说明

- `-p/--tid`：指定目标线程（默认是主线程）
- `--instruction-limit`：限制采集指令数，默认 5,000 万条。实际数量会高于该值：停止按内部批次粒度进行，实测可能超出设定值 2 倍以上。
- `--warmup-ms`：设置预热时间。初始翻译会将原始二进制转换为带插桩的代码，可能影响目标程序性能；预热期间不采集 Trace，以减少失真。默认值为 1 ms。
- `--trigger-pc`（别名 `--start-pc`）/ `--trigger-hit`：在指定 PC 第 N 次命中后开始录制，推荐用于精确控制 Trace 的采集窗口。
- `--trigger-timeout-s`：等待 Trigger PC 命中的最长时间，默认 10 s。
- `--end-pc`：为 `elfgen` 提供优先的切片结束位置。若 End PC 位于热点循环中，会产生大量结束判断和补偿代码，可能影响切片代表性；建议选择大函数末尾等低频位置。
- `--trace-context-sve`：记录 SVE 寄存器初始值，默认关闭；目标程序包含 SVE 指令时需要开启。
- `--trace-context-neon`：记录 NEON/FP 寄存器初始值，默认开启；仅在确认目标程序只含标量指令时才可关闭。
- `--trace-jit`：用于存在自修改行为的目标程序，例如 Python 程序；尚未充分验证。
- `--emit-elf`：Trace 成功后自动生成 ELF；产物在 `48bit/` 目录下，含 `orr` / `brk` /
  `nofinal` 三个变体（见 Elfgen 输出文件说明）
- `--merge-elf`：Trace 成功后自动调用 `elfgen` 时生效，额外生成合并 PT_LOAD 变体
  （`48bit-merge/`）；完整说明见 [Elfgen 功能说明](#elfgen-功能说明)的选项说明
- `--gen39bit`：Trace 成功后自动调用 `elfgen` 时，在 48-bit 产物之外额外生成 39-bit VA
  切片（`39bit/` 目录）（**开发中的实验特性**）；完整说明见 [Elfgen 功能说明](#elfgen-功能说明)

### Trace 输出文件

Trace 成功后会生成原始 `.bpraw` 和 `host.log`。文件名规则如下：

```text
<output-dir>/binprofiler-trace-<pid>-<tid>.bpraw
<output-dir>/host.log
```

- `.bpraw`：原始 Trace 文件，包含 catalog、动态执行记录、内存 payload 和必要的 context 信息，是后续 `elfgen` 的主要输入。
- `host.log`：本次 Trace 的主机侧日志，包含参数、`agent` 注入过程、实际解析出的 trigger/end PC、输出文件路径和错误信息。

启用 `--emit-elf` 时，Trace 会在成功后自动调用 Elfgen；生成的 ELF 和 map 文件见下方 Elfgen 输出说明。

</details>

<details>
<summary>Trigger PC 与 End PC 设置</summary>

`--trigger-pc` 和 `--end-pc` 都支持两种写法：

```text
0xADDR
MODULE:0xADDR
```

含义分别是：

- `0xADDR`：动态运行时 PC。这个地址必须是目标进程当前运行时的真实虚拟地址。
- `MODULE:0xADDR`：模块内静态 PC。`0xADDR` 是 `objdump` / `readelf` 看到的
  ELF 静态地址，`MODULE` 用来指定这个地址属于哪个已加载模块。

优先使用 `MODULE:0xADDR`，尤其是目标程序或动态库启用了 PIE/ASLR 时。这样无需手动
计算运行时基址：BinProfiler 会在目标进程内读取 `/proc/self/maps` 和 ELF program header，
将静态 PC 转换为实际的运行时 PC。Trace 完成时，`host.log` 会记录解析后的地址，例如：

```text
trace finished: actual trigger pc=0xaaaad77f08d8 actual end pc=0xaaaad77f08fc instructions=3256
trace stats: catalog=35 dynamic=594 dropped_catalog=0 dropped_dynamic=0 output_bytes=70672 cleanup_pending=true
```

### 使用动态运行时 PC

如果你已经知道目标进程里的实际 PC，可以直接传裸地址：

```sh
./binprofiler trace <pid> \
  --trigger-pc 0x7f0012345678 \
  --end-pc 0x7f00123489ab \
  --emit-elf
```

注意：裸 `0xADDR` 永远被当作动态运行时 PC，不会自动按静态 PC 解析。

### 使用 `objdump` 获取静态 PC

通常先对目标二进制或动态库做反汇编，找到想要的静态 PC：

```sh
objdump -d /path/to/demo_app | grep -n '<target_function>'
objdump -d /usr/lib/libfoo.so | grep -n '<target_function>'
```

假设得到：

```text
00000000000008d8 <trigger_func>:
00000000000008fc <end_func>:
```

`MODULE` 通常可以直接写模块文件名（basename）。例如 `/path/to/demo_app` 可写成
`demo_app`，`/usr/lib/libfoo.so` 通常写成 `libfoo.so`：

```sh
./binprofiler trace <pid> \
  --trigger-pc demo_app:0x8d8 \
  --end-pc demo_app:0x8fc \
  --emit-elf
```

如果 PC 来自动态库：

```sh
./binprofiler trace <pid> \
  --trigger-pc libfoo.so:0x1234 \
  --end-pc libfoo.so:0x4568 \
  --emit-elf
```

如果不确定模块在目标进程中的名称，或者 basename 匹配失败、匹配到多个模块，可查看
`/proc/<pid>/maps` 中的可执行映射来确认实际路径：

```sh
cat /proc/<pid>/maps | awk '$2 ~ /x/ { print }'
```

示例输出：

```text
aaaaddaa0000-aaaaddaa1000 r-xp 00000000 00:32 6730 /tmp/demo_app
ffff9a100000-ffff9a200000 r-xp 00000000 00:32 9001 /usr/lib/libfoo.so
```

`MODULE` 支持以下匹配方式：

- maps 中的完整路径，例如 `/usr/lib/libfoo.so:0x1234`
- basename，例如 `libfoo.so:0x1234`
- 路径后缀，例如 `aarch64-linux-gnu/libfoo.so:0x1234`
- 主程序没有 `main:` / `exe:` 等特殊别名，也需按 maps 中的路径或文件名填写

若 basename 匹配到多个模块，BinProfiler 会报错；此时改用更完整的路径即可。

### Trigger PC 与 End PC 可以来自不同模块

`--trigger-pc` 和 `--end-pc` 会分别解析，因此可以分别来自主程序和动态库：

```sh
./binprofiler trace <pid> \
  --trigger-pc demo_app:0x8d8 \
  --end-pc libfoo.so:0x4568 \
  --emit-elf
```

### 常见注意事项

- 如果使用 `MODULE:0xADDR`，`MODULE` 必须能在目标进程 maps 中唯一匹配。
- 如果使用裸 `0xADDR`，需自行确保它是运行时 PC。
- 不要把 `objdump` 里看到的小地址直接写成裸地址，例如 `--trigger-pc 0x8d8`。
  这会被当作运行时地址 `0x8d8`，通常不会命中。
- `end-pc` 仅为 `elfgen` 提供优先结束位置；Trace 本身仍由
  `--instruction-limit` 停止。

</details>

<details>
<summary>End PC 选择原则</summary>

基于 Trace 生成的 ELF 不是常规程序，没有天然的 `exit` 位置。BinProfiler 会在
`end-pc` 对应的位置插入结束判断和补偿逻辑，用它作为 `elfgen` 优先选择的切片结束点。

选择 `end-pc` 时最重要的原则是：**选择执行次数少、语义稳定、靠近目标片段结尾的指令**。
不要将 `end-pc` 放在热点循环中；同一 PC 的命中次数越多，`elfgen` 需要区分和补偿的动态实例
越多，`final_pc` 类补偿代码的开销也越高，生成的切片越容易失真。

若仅在模拟器中运行 39-bit nofinal 版本，可按模拟指令数截断执行，无需指定 final PC。

建议优先选择：

- 目标函数或目标阶段末尾附近的普通计算指令
- 循环外、仅执行少数几次的指令
- 已确认会在预期 Trace 窗口之后执行到的位置

避免选择：

- 热点循环内的指令
- `ret`、`br`、`b`、`bl`、条件跳转等控制流跳转指令
- 可能在多个不相关路径中频繁命中的公共代码位置

出于稳定性考虑，当前不允许将控制流跳转指令设为 `end-pc`。若配置的 `end-pc`
属于跳转、调用或返回指令，Trace 启动时会直接报错；请改选其前后附近的非控制流指令。

</details>

<details>
<summary>指令支持情况</summary>

当前 `trace` / `elfgen` 对 AArch64 内存相关指令的支持情况如下：

| 范围 | 代表类别 | Trace probe | Elfgen | 当前状态 |
| --- | --- | --- | --- | --- |
| 标量普通 ld/st | 普通单寄存器 load/store | 支持 | 支持 | 稳定 |
| 标量 pair | `ldp/stp`、`ldnp/stnp` | 支持 | 支持 | 稳定 |
| 标量 literal load | literal `ldr` | 支持 | 支持 | 稳定 |
| 标量 ordered / RCPC | acquire/release、RCPC 单寄存器 | 支持 | 支持 | 稳定 |
| 标量 exclusive | exclusive load/store | 支持 | 支持 | 稳定 |
| 标量 LSE atomic | LSE atomic RMW / CAS | 支持 | 支持 | 稳定 |
| NEON/FP 单寄存器 | `B/H/S/D/Q` 单寄存器 load/store | 支持 | 支持 | 稳定 |
| NEON/FP pair | `S/D/Q` pair load/store | 支持 | 支持 | 稳定 |
| NEON/FP literal load | `S/D/Q` literal load | 支持 | 支持 | 稳定 |
| NEON full-register structure | full-register `ld1..ld4/st1..st4` | 支持 | 支持 | 稳定 |
| pair ordered / pair RCPC | ordered / RCPC pair | 不支持 | 不支持 | 无支持计划 |
| NEON/FP RCPC | SIMD/FP RCPC load/store | 不支持 | 不支持 | 无支持计划 |
| SVE memory | SVE contiguous、tuple、gather/scatter、fault-first 等 | 支持 | 支持 | 稳定 |
| SVE 未支持 | 部分 SVE 地址形态与少数剩余指令族 | 不支持 | 不支持 | 按需支持 |
| PRFM | prefetch hint | 支持 | 支持 | 稳定 |
| MOPS | 专用 memory 流水指令 | 不支持 | 不支持 | 无支持计划 |

### SVE Vector Length 说明

采集 Trace 时，不同 load/store 指令可以具有不同的 Vector Length；生成 ELF 时，整份 ELF
必须使用固定的 Vector Length。原因是生成的 ELF 仅包含用户态指令流，不包含用于切换
Vector Length 的系统调用。

</details>

## Elfgen 功能说明

如果已有 `.bpraw` Trace，可单独运行 `elfgen` 生成切片 ELF。基本调用形式如下：

```sh
./binprofiler [--output-dir <dir>] elfgen <trace.bpraw> [elfgen-options]
```

默认情况下：

- 输出目录是 `./out`
- 输入文件是已有的 `.bpraw`
- 生成的 ELF 会根据 trace 内容自动命名并写入输出目录

### 常见用法

```sh
# 1. 从已有 Trace 生成 ELF
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw

# 2. 生成 39-bit VA 切片
./binprofiler --output-dir /tmp/binprofiler-39bit elfgen \
  ./out/binprofiler-trace-12345-12345.bpraw \
  --gen39bit

# 3. 合并 segment，降低 Program Header 数量（适用条件见选项说明）。
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw \
  --merge-elf
```

<details>
<summary>选项说明</summary>

- `--ignore-end-pc`：忽略 trace 里的 `end-pc` 标记，让 `elfgen` 只按 instruction-limit 回退策略选择结束位置（**强烈不建议使用**）
- `--merge-elf`：额外生成一份合并 PT_LOAD 的变体，输出在 `48bit-merge/` 目录，与 `48bit/`
  常规产物并存（合并 pass 的 PT_LOAD 目标数固定为 70）。用于满足 Linux 7.0 之前内核对
  Program Header Table 大小不超过 4 KiB 的限制；Linux 7.0 及以上无需使用此选项。
  可通过 `uname -r` 查看当前内核版本。
- `--gen39bit`：48-bit 发射完成后运行 39-bit 生成序列，产出 `39bit/` 目录；配合
  `--merge-elf` 时另有 `39bit-merge/`（见下方 39-bit VA 切片功能）（**开发中的实验特性**；
  旧名 `--to39bit` 已移除，传入会报未知选项）
- `--compensate-dczva`：把 `DC ZVA` 替换为可移植的软件清零 store
- `-q/--quiet`：抑制终端输出，`host.log` 仍完整记录

</details>

<details>
<summary>补偿代码</summary>

`elfgen` 生成的 replay ELF 并非简单按顺序重放 Trace 中的原始指令。对于初始寄存器状态、
系统调用、系统寄存器读取、部分内存访问和最终退出点，直接重放要么无法恢复 Trace 时的精确状态，
要么会产生不应出现的外部副作用。补偿代码是 `elfgen` 额外插入的 runtime stub：它会根据
`.bpraw` 中的上下文、内存事件和专用 payload 修正运行时状态，再跳回原始指令流或终止执行。

补偿统计会同时输出到终端与 `host.log`，例如：

```text
48bit statistics: compensation instructions=938854 (240.01%) without finish compensation=45 (0.01%)
48bit statistics details: base=899699 load=0 stxr-atomic=0 svc=0 mrs=0 finish=39117 init=38
```

第一行是补偿指令总量（相对 trace 指令数的百分比）与扣除 finish 补偿后的数量；`details`
行是分类明细，各字段表示该类补偿 stub 生成并实际计入的指令量：

| 项 | 目的 | 备注 |
| --- | --- | --- |
| `base` | 全部补偿桩共享的机构成本：帧保存/恢复、scratch 寄存器、回跳，以及 occurrence 检查机制（游走、select、游标比较、条目路由）等。 | 刻意按单数聚合，不按族细分。 |
| `init` | 在 replay ELF 入口从 trace 的 initial context 恢复初始寄存器状态，包括 PC、SP、通用寄存器、NZCV、FPCR/FPSR，以及可选的 NEON/SVE 上下文。 | 通常每个 replay ELF 有一个，是进入 trace 指令流前的状态基线。 |
| `svc` | 避免重新执行可能产生外部副作用或上下文依赖的 `svc`；改为从 runtime table 选择该动态实例的 context snapshot，恢复后跳到记录的后继 PC。 | `restore_context` 也归入 `svc` 统计，用于部分 SVE first-fault / no-fault load 等需要完整上下文恢复的场景。 |
| `mrs` | 避免重新读取 replay 时可能变化的系统寄存器；改为把 trace 捕获到的读值写回目标寄存器。 | 常见于计数器、状态寄存器等系统寄存器读值。 |
| `load` | 普通内存指令补偿的数据面成本：读取记录/条目值并写出交付值，覆盖 load 值回填、store 效果修正、`DC ZVA` zero range 等。 | 如果 shadow memory 已能证明直接重放安全，可能不会生成对应补偿。 |
| `stxr-atomic` | 修正 exclusive load/store 与 atomic/LSE 序列，保证 store-exclusive 的成功/失败结果和必要的内存副作用与 trace 一致。 | 对应旧版统计中的 `exclusive`。 |
| `finish` | 标记 replay 应该停止的动态位置（final PC）的结束判断与补偿代码。 | `orr` / `brk` 两个变体共享同一份 final PC 选择，只是结束信号不同；`nofinal` 变体不插入这部分。 |
| `dczva` | 把 `DC ZVA` 替换为可移植的软件清零 store。 | 仅开启 `--compensate-dczva` 时该字段出现（关闭时整个字段省略）。 |

`details` 行按固定字段顺序全显式（零值不省略），各分类之和与补偿指令总量守恒。
39-bit 管线的 `details` 行额外包含 `address`、`cond`、`indirect` 三个字段，对应
39-bit 地址投影与控制流方向的补偿。

生成的 ELF 不含 section header；各代码区域改由对应的 `_map.txt` 描述：`original-code`
（原始业务代码）、`stub-0x...`（补偿桩）、`init-stub`（入口初始化）、`runtime`（运行时表）、
`literal-*`（literal 岛）、`mem`（数据区）、`private-stack`（私有栈）。调试 replay 差异或
统计补偿占比时，结合 map 区域与上述统计查看。

</details>

<details>
<summary>39-bit VA 切片功能</summary>

`--gen39bit` 用于将 48-bit VA Linux 环境采集的 `.bpraw` Trace 转换为可在 39-bit VA
环境运行的切片 ELF，主要面向虚拟地址宽度受限的模拟器回放场景。`elfgen` 会建立 48-bit 到
39-bit 的地址映射（按高位截断投影），并据此重写 Trace 中的 PC、SP、内存访问地址及可识别的
指针来源；常规 48-bit 产物照常生成在 `48bit/`，39-bit 产物额外写入 `39bit/`（变体与 map
规则和 48-bit 相同），配合 `--merge-elf` 时另有 `39bit-merge/`。对无法完全证明的残差，
默认插入兜底桩保证正确性。

可在 Trace 成功后一次性生成 48-bit 和 39-bit 两套 ELF：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf \
  --gen39bit
```

也可针对已有 `.bpraw` 单独执行 `elfgen`：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit elfgen \
  ./out/binprofiler-trace-12345-12345.bpraw \
  --gen39bit
```

该功能仍属于实验特性。若出现回放差异，优先使用同次生成的普通 48-bit ELF 确认
Trace 本身能否正常生成和运行，再排查 39-bit 投影引入的差异；`host.log` 中 39-bit 统计的
`address` / `cond` / `indirect` 补偿量可作参考。

</details>

<details>
<summary>Elfgen 输出文件</summary>

`elfgen` 把切片 ELF 及其对应的 map 文件按生成路径写入输出目录的子目录。
当 Trace 使用 `--emit-elf` 自动调用 Elfgen 时，产物也遵循相同规则：

```text
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-orr.elf
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-brk.elf
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-nofinal.elf
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-orr_map.txt
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-brk_map.txt
<output-dir>/48bit/binprofiler-trace-<pid>-<tid>-nofinal_map.txt
<output-dir>/host.log
```

- 默认生成 48-bit 切片，位于 `48bit/`；开启 `--merge-elf` 后额外生成合并 PT_LOAD
  变体，位于 `48bit-merge/`，文件名相同。
- 39-bit 生成序列（`--gen39bit`）的产物位于 `39bit/`，配合 `--merge-elf` 时另有
  `39bit-merge/`，见下方 39-bit VA 切片功能。
- `-orr.elf`：面向模拟器使用的切片 ELF。结束位置使用 `orr x4, x4, x4` 作为
  finish marker，便于模拟器识别切片执行结束。
- `-brk.elf`：面向真机或普通 Linux 环境运行的切片 ELF。结束位置使用 `brk #0`
  触发 `SIGTRAP`，方便本地 runner 或调试器判断执行完成。
- `-nofinal.elf`：不插入 final PC 结束点补偿代码的版本；没有内置结束信号，
  只能在模拟器上运行，由模拟器按总指令数截断执行。
- `_map.txt`：对应 ELF 的地址映射说明，按区域标注原始业务代码、补偿桩、初始化
  stub、runtime 数据等。调试 replay 差异、统计补偿代码比例或在模拟器中标注不同
  代码区域时，优先参考这个文件。
- `host.log`：本次 `elfgen` 的主机侧日志，包含参数、补偿统计、输出文件路径和错误信息；
  Trace 自动调用 Elfgen 时，同一日志还会记录 Trace 的注入与采集信息。

</details>

## 指令数统计功能说明

`trace --count-only` 不抓取 `.bpraw` trace，只统计指定 trigger PC 到 end PC
之间的动态 AArch64 指令数。Gum 只发布 basic block 的起始 PC 和指令数，
trigger/end 判断和计数在 consumer 线程完成。

count-only 要求同时指定 `--trigger-pc` 与 `--end-pc`（缺一会直接报错）；`--trigger-hit` 与 `--end-hit` 可省略，默认 `1`。窗口参数如下：

- `--trigger-pc <PC>`：计数起点 PC。可以使用运行时地址
  `0x...`，也可以使用静态 ELF 地址 `MODULE:0xADDR`。
- `--trigger-hit <N>`：第 `N` 次动态命中 trigger PC 后开始计数，默认 `1`。
- `--end-pc <PC>`：计数终点 PC，地址格式与 `--trigger-pc` 相同。
- `--end-hit <M>`：计数开始后第 `M` 次命中 end PC 时结束，默认 `1`。

trigger 和 end 指令都计入结果；两个 PC 必须是 4-byte 对齐的 AArch64
地址。未指定 `-p/--tid` 时统计目标主线程（Linux 上通常为 `tid == pid`）。

count-only 的其他行为和限制：

- 不使用 warmup，也不使用 instruction limit；显式传入
  `--warmup-ms` 或 `--instruction-limit` 会报冲突，不会静默忽略。
- `--end-hit` 只能和 `--count-only` 一起使用；普通 trace 传入会报冲突。
- `--trace-jit` 可以保留，用于 Python、PyTorch 等 JIT 或自修改代码目标。
- `--trigger-timeout-s` 仍只保护等待 trigger 的阶段，默认 `10` 秒，`0` 表示禁用。
  第一版没有 end PC timeout；trigger 已命中但 end 长时间未命中时，需要由外部结束目标
  进程和被注入的 agent 进程。
- count-only 不产生 `.bpraw`，不能使用 `--emit-elf` 及其 ELF/context 选项。

成功完成后，host 会在终端和 `<output-dir>/host.log` 输出类似以下结果：

```text
count-only finished: actual trigger pc=0xffff... hit=1 actual end pc=0xffff... hit=1 instructions=8926
```

一个常见用法是统计目标线程执行一段 PyTorch/原生代码窗口。下面的 PC 仅为示例，
实际地址应从目标进程的符号、ELF 或 `/proc/<pid>/maps` 中取得：

```sh
./binprofiler trace <pid> \
  --count-only \
  --trigger-pc 0xffff12345000 \
  --trigger-hit 1 \
  --end-pc 0xffff12346000 \
  --end-hit 1
```

如果目标线程不是主线程，补充 `--tid <tid>`；如果使用静态模块地址，可将 PC 写成
`libfoo.so:0x12340`。命令完成后只读取 terminal/`host.log` 中的
`instructions=<N>`，不会在输出目录留下 trace 文件。

## 常见问题与排查

**附加目标进程失败或无权限**

通常是 `ptrace_scope` 限制导致，配置方法见[运行原理](#运行原理)；共享机器上建议采集结束后恢复原值。

**Trigger / End PC 不命中**

裸 `0xADDR` 会被当作动态运行时 PC；目标程序启用 PIE/ASLR 时，静态地址与运行时地址不一致也会导致不命中。优先使用 `MODULE:0xADDR` 写法，详见 Trace 功能说明中的 Trigger PC 与 End PC 设置。

**切片 ELF 运行失败，或行为与采集时不一致**

先比对回放环境与采集环境的差异：SVE Vector Length、`vm.mmap_min_addr`、ASLR、ptrace 限制、OS/glibc 版本。回放环境的 SVE Vector Length 需与采集环境一致，否则切片可能无法正确回放。

**怀疑两台机器上的 BinProfiler 不是同一份二进制**

先校验下载的 tar.gz sha256 与发布页校验文件是否一致，确认来自同一次发布；再用 `readelf -n binprofiler` 比对 Build ID。`binprofiler -V` 只能证明版本字符串一致，不能证明二进制来源一致。
