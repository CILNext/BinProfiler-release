# BinProfiler

BinProfiler 是一个基于 Frida 的二进制插桩工具，用于性能分析和切片制作；对应平台的二进制包可从本仓的 GitHub Releases 页面获取。

当前支持范围：

| CPU ISA \\ OS | `Ubuntu 22.04+` | `OpenEuler 22.03+` |
| --- | --- | --- |
| AArch64 | `trace`、`elfgen` | `trace`、`elfgen` |
| x86_64 | - | - |

## 目录

- [运行原理](#运行原理)
- [Trace 功能说明](#trace-功能说明)
- [Elfgen 功能说明](#elfgen-功能说明)
- [指令数统计功能说明](#指令数统计功能说明)
- [BPTF Trace 格式](#bptf-trace-格式)

## 运行原理

`binprofiler` 负责附加到目标进程、注入 `agent`，并通过控制通道下发启动参数和任务请求；`agent` 常驻于目标进程内，执行采集并回传结果。

在 Linux 上`agent`的注入依赖 `ptrace` 权限。如果本机启用了较严格的 `ptrace_scope` 限制，即使是同用户进程，也可能无法附加成功。

使用前建议确认：

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
# Trace 成功后自动生成 ELF 和 BPTF。默认采集 5,000 万条指令，
# 并优先将指定的 End PC 作为切片结束位置。
./binprofiler --output-dir /tmp/binprofiler-replay trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf \
  --emit-trace
```

<details>
<summary>选项说明与 Trace 输出</summary>

### 选项说明

- `--tid`：指定目标线程（默认是主线程）
- `--instruction-limit`：限制采集指令数，默认 5,000 万条；实际数量会略高于该值。
- `--warmup-ms`：设置预热时间。初始翻译会将原始二进制转换为带插桩的代码，可能影响目标程序性能；预热期间不采集 Trace，以减少失真。默认值为 1 ms。
- `--trigger-pc` / `--trigger-hit`：在指定 PC 第 N 次命中后开始录制，推荐用于精确控制 Trace 的采集窗口。
- `--trigger-timeout-s`：等待 Trigger PC 命中的最长时间，默认 10 s。
- `--end-pc`：为 `elfgen` 提供优先的切片结束位置。若 End PC 位于热点循环中，会产生大量结束判断和补偿代码，可能影响切片代表性；建议选择大函数末尾等低频位置。
- `--trace-context-sve`：记录 SVE 寄存器初始值，默认关闭；目标程序包含 SVE 指令时需要开启。
- `--trace-context-neon`：记录 NEON/FP 寄存器初始值，默认开启；仅在确认目标程序只含标量指令时才可关闭。
- `--trace-jit`：用于存在自修改行为的目标程序，例如 Python 程序；尚未充分验证。
- `--emit-trace`：导出二进制 Trace 文件，格式见 BPTF 说明。
- `--emit-elf`：Trace 成功后自动生成 ELF；普通 48-bit 路径会同时生成
  `-48bit-orr.elf` 和 `-48bit-brk.elf` 两个 finish terminator 变体
- `--merge-elf`：合并 segment，降低最终 ELF 的 Program Header 数量，以满足 Linux 7.0
  之前内核对 Program Header Table 大小不超过 4 KiB 的限制；Linux 7.0 及以上无需使用此选项。
  可通过 `uname -r` 查看当前内核版本。
- `--to39bit`：将 48-bit VA 环境采集的 Trace 转换为可在 39-bit VA 环境运行的切片，适用于模拟器；会额外生成
  `-39bit-orr.elf`、`-39bit-brk.elf` 和 `-39bit-nofinal.elf`（**开发中的实验特性**）

### Trace 输出文件

Trace 成功后会生成原始 `.bpraw` 和 `host.log`；开启 `--emit-trace` 后还会导出展开后的 `.bptf`。文件名规则如下：

```text
<output-dir>/binprofiler-trace-<pid>-<tid>.bpraw
<output-dir>/binprofiler-trace-<pid>-<tid>.bptf
<output-dir>/host.log
```

- `.bpraw`：原始 Trace 文件，包含 catalog、动态执行记录、内存 payload 和必要的 context 信息，是后续 `elfgen` 的主要输入。
- `.bptf`：由 `.bpraw` 展开的指令与内存事件流，便于 diff、校验和离线分析；通常明显大于 `.bpraw`，建议仅在排查问题、对比 Trace 或进行离线分析时生成。
- `host.log`：本次 Trace 的主机侧日志，包含参数、`agent` 注入过程、实际解析出的trigger/end PC、输出文件路径和错误信息。

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
  --to39bit

# 3. 降低 Program Header 数量，以满足 Linux 7.0 之前内核对 Program Header Table
#    大小不超过 4 KiB 的限制。
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw \
  --merge-elf
```

<details>
<summary>选项说明</summary>

- `--emit-trace`：在生成 ELF 的同时导出展开后的 `.bptf`
- `--ignore-end-pc`：忽略 trace 里的 `end-pc` 标记，只按默认回退策略选择结束位置（**强烈不建议使用**）
- `--merge-elf`：合并 segment，降低最终 ELF 的 Program Header 数量，以满足 Linux 7.0
  之前内核对 Program Header Table 大小不超过 4 KiB 的限制；Linux 7.0 及以上无需使用此选项。
  可通过 `uname -r` 查看当前内核版本。
- `--to39bit`：同时生成普通 48-bit VA 切片 ELF 和面向 39-bit VA 环境的切片 ELF；
  39-bit 输出包含 `-39bit-orr.elf`、`-39bit-brk.elf` 和 `-39bit-nofinal.elf`（**开发中的实验特性**）

</details>

<details>
<summary>补偿代码</summary>

`elfgen` 生成的 replay ELF 并非简单按顺序重放 Trace 中的原始指令。对于初始寄存器状态、
系统调用、系统寄存器读取、部分内存访问和最终退出点，直接重放要么无法恢复 Trace 时的精确状态，
要么会产生不应出现的外部副作用。补偿代码是 `elfgen` 额外插入的 runtime stub：它会根据
`.bpraw` 中的上下文、内存事件和专用 payload 修正运行时状态，再跳回原始指令流或终止执行。

补偿统计会出现在 `host.log` 里，例如：

```text
compensation instructions by category={initial=... svc=... mrs=... ldst=... final_pc=... exclusive=...}
```

这些字段表示补偿 stub 生成并实际计入的指令量，含义如下：

| 项 | 目的 | 备注 |
| --- | --- | --- |
| `initial` | 在 replay ELF 入口从 trace 的 initial context 恢复初始寄存器状态，包括 PC、SP、通用寄存器、NZCV、FPCR/FPSR，以及可选的 NEON/SVE 上下文。 | 通常每个 replay ELF 有一个，是进入 trace 指令流前的状态基线。 |
| `svc` | 避免重新执行可能产生外部副作用或上下文依赖的 `svc`；改为从 runtime table 选择该动态实例的 context snapshot，恢复后跳到记录的后继 PC。 | `restore_context` 也归入 `svc` 统计，用于部分 SVE first-fault / no-fault load 等需要完整上下文恢复的场景。 |
| `mrs` | 避免重新读取 replay 时可能变化的系统寄存器；改为把 trace 捕获到的读值写回目标寄存器。 | 常见于计数器、状态寄存器等系统寄存器读值。 |
| `ldst` | 修正普通 load/store、atomic/LSE、`DC ZVA` 等内存效果；在原始内存指令前后补齐 trace 中记录的内存值、地址、scratch 寄存器或 zero range。 | 这里对应常规 memory 补偿路径；如果 shadow memory 已能证明直接重放安全，可能不会生成对应补偿。 |
| `exclusive` | 修正 exclusive load/store 序列，保证 store-exclusive 的成功/失败结果和必要的内存副作用与 trace 一致。 | 只有非零时才出现在 summary。 |
| `final_pc` | 标记 replay 应该停止的动态位置。 | 普通 48bit 和 39bit 双 variant 共享同一份 final PC 选择；`-orr.elf` 和 `-brk.elf` 只是使用不同的结束信号。39-bit nofinal 版本不插入这类结束点补偿代码。 |
| `conditional_control` | 在 39bit VA 管线中强制采用 trace 记录的条件分支方向和后继 PC，避免 48bit 到 39bit 地址投影后条件判断漂移。 | 只有非零时才出现在 summary。 |

生成的补偿代码位于 `.bp.runtime.stub.<kind>.*` 等 section，运行时表位于对应的
runtime table section。map 通过这些 section/stub 名称描述 patch site、stub 位置、动态实例数量
和补偿指令数；日志中的 `initial/svc/mrs/ldst/final_pc/exclusive/...` 则提供补偿开销来源的汇总视图。

</details>

<details>
<summary>39-bit VA 切片功能</summary>

`--to39bit` 用于将 48-bit VA Linux 环境采集的 `.bpraw` Trace 转换为可在 39-bit VA
环境运行的切片 ELF，主要面向虚拟地址宽度受限的模拟器回放场景。启用后，`elfgen` 会建立
48-bit 到 39-bit 的地址映射，并据此重写 Trace 中的 PC、SP、内存访问地址及可识别的指针来源；
同时保留普通 48-bit VA baseline ELF，并额外生成 39-bit ELF。当前地址映射采用高位清零的
投影方式：若多个 48-bit 地址清零后落入同一 39-bit 地址，暂时无法自动消解该冲突。

可在 Trace 成功后自动生成 48-bit 和 39-bit 两份 ELF：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf \
  --to39bit
```

也可针对已有 `.bpraw` 单独执行 `elfgen`：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit elfgen \
  ./out/binprofiler-trace-12345-12345.bpraw \
  --to39bit
```

该功能仍属于实验特性。若出现回放差异，优先使用同次生成的普通 48-bit ELF 确认
Trace 本身能否正常生成和运行。39-bit 生成过程中还可能留下用于验证的
`-39bit-probe_map.txt`、`-39bit-tagprobe-*_map.txt` 等中间 map 文件。

</details>

<details>
<summary>Elfgen 输出文件</summary>

`elfgen` 会生成切片 ELF 及其对应的 map 文件；开启 `--emit-trace` 时还会导出 `.bptf`。
当 Trace 使用 `--emit-elf` 自动调用 Elfgen 时，产物也遵循相同规则：

```text
<output-dir>/binprofiler-trace-<pid>-<tid>.bptf
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit-orr.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit-brk.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit-orr_map.txt
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit-brk_map.txt
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-orr.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-brk.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-nofinal.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-orr_map.txt
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-brk_map.txt
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit-nofinal_map.txt
<output-dir>/host.log
```

未开启 `--to39bit` 时只生成 `-48bit-orr.elf` 和 `-48bit-brk.elf`；开启后还会生成
`-39bit-orr.elf`、`-39bit-brk.elf` 和 `-39bit-nofinal.elf`。各 ELF 会生成对应的
`_map.txt` 文件。

`-39bit-orr.elf` 和 `-39bit-brk.elf` 来自同一份 final PC / compensation plan，
只在结束信号上不同。

- `.bptf`：仅在开启 `--emit-trace` 时生成；与 Trace 命令生成的 `.bptf` 格式相同，
  用于 diff、校验和离线分析。
- `-orr.elf`：面向模拟器使用的切片 ELF。结束位置使用 `orr x4, x4, x4` 作为
  finish marker，便于模拟器识别切片执行结束。
- `-brk.elf`：面向真机或普通 Linux 环境运行的切片 ELF。结束位置使用 `brk #0`
  触发 `SIGTRAP`，方便本地 runner 或调试器判断执行完成。
- `-39bit-nofinal.elf`：39-bit VA 管线额外生成的无 final PC 补偿版本，不插入结束点补偿代码；
  只能在模拟器上运行，依赖模拟器靠总指令数结束运行。
- `_map.txt`：对应 ELF 的地址映射说明，用来区分原始业务代码、补偿代码、
  finish stub、runtime 数据等区域。调试 replay 差异、统计补偿代码比例或在模拟器
  中标注不同代码区域时，优先参考这个文件。
- `host.log`：本次 `elfgen` 的主机侧日志，包含参数、输出文件路径和错误信息；Trace 自动
  调用 Elfgen 时，同一日志还会记录 Trace 的注入与采集信息。

</details>

## 指令数统计功能说明

`trace --count-only` 不抓取 `.bpraw` trace，只统计指定 trigger PC 到 end PC
之间的动态 AArch64 指令数。Gum 只发布 basic block 的起始 PC 和指令数，
trigger/end 判断和计数在 consumer 线程完成。

count-only 必须同时提供以下四个窗口参数：

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

## BPTF Trace 格式

<details>
<summary>BPTF 二进制格式规范</summary>

`.bptf` 是由 `.bpraw` 展开的二进制指令流格式，可用于 diff、回放校验和离线分析。
它不是原始捕获格式，完整元数据仍以 `.bpraw` 为准。

当前实现采用小端紧凑二进制布局，文件按以下顺序由三部分组成：

```text
RawBptfHeader
RawBptfInstRecord[inst_count]
RawBptfMemRecord[mem_count]
```

各部分含义如下：

- 每条 `RawBptfInstRecord` 对应一条动态执行的指令
- 如果该指令有关联的内存访问，则通过 `mem_index + mem_count` 指向连续的
  `RawBptfMemRecord[]`
- 当前版本对 NEON 和 SVE 内存指令的支持有限
- 无效索引统一使用 `0xffffffff`

读取当前版本时，建议先校验以下固定条件；这些条件必须全部满足：

```text
magic == "BPTF"
version == 1
header_size == 80
endian == 1            # little endian
arch == 1              # AArch64
inst_record_size == 32
mem_record_size == 32
inst_offset == 80
mem_offset == inst_offset + inst_count * 32
ext_data_offset == mem_offset + mem_count * 32
ext_data_size == 0
string_table_offset == ext_data_offset
string_table_size == 0
file_size == ext_data_offset
```

若任一条件不满足，应将文件视为非当前版本的 `.bptf` 并直接拒绝。

当前落盘结构如下：

```c
struct RawBptfHeader {
    uint32_t magic;               // 'BPTF'
    uint16_t version;             // 当前为 1
    uint16_t header_size;         // 当前为 80
    uint16_t endian;              // 1 = little endian
    uint16_t arch;                // 1 = AArch64
    uint16_t inst_record_size;    // 当前为 32
    uint16_t mem_record_size;     // 当前为 32
    uint64_t inst_offset;         // 当前固定等于 sizeof(RawBptfHeader)
    uint64_t inst_count;
    uint64_t mem_offset;
    uint64_t mem_count;
    uint64_t ext_data_offset;     // 当前等于文件尾
    uint64_t ext_data_size;       // 当前固定为 0
    uint64_t string_table_offset; // 当前等于文件尾
    uint64_t string_table_size;   // 当前固定为 0
};

struct RawBptfInstRecord {
    uint64_t pc;                  // 指令 PC
    uint32_t opcode;              // AArch64 32-bit opcode
    uint32_t block_id;            // catalog block 序号
    uint32_t op_index;            // block 内静态指令序号
    uint32_t mem_index;           // 无内存访问时为 0xffffffff
    uint16_t mem_count;           // 关联的内存记录数
    uint16_t type;                // InstructionType
    uint16_t flags;               // InstructionFlags bitmask
    uint8_t inst_size;            // AArch64 当前通常为 4
    uint8_t reserved;             // 当前保留为 0
};

struct RawBptfMemRecord {
    uint64_t addr;                // 访问地址
    uint64_t data_lo;             // 低 64-bit 数据
    uint64_t data_hi;             // 高 64-bit 数据，<=8B 访问时通常为 0
    uint16_t size;                // 本次访问字节数
    uint8_t kind;                 // MemoryAccessKind
    uint8_t flags;                // MemoryAccessFlags bitmask
    uint32_t ext_offset;          // 当前固定为 0xffffffff
};
```

若自行实现读取器，最小读取流程如下：

```text
1. 读取 RawBptfHeader，并校验固定值和 offset/size 关系
2. 跳到 inst_offset，读取 inst_count 个 RawBptfInstRecord
3. 跳到 mem_offset，读取 mem_count 个 RawBptfMemRecord
4. 对每条 inst：
   - 如果 mem_index != 0xffffffff
   - 则 [mem_index, mem_index + mem_count) 这段连续 mem records
   - 就是这条动态指令的内存行为
```

`type` / `kind` / `flags` 的当前取值约定：

```c
enum class InstructionType : uint16_t {
    Unknown = 0,
    Alu, Load, Store, Atomic, Branch, Call, Return,
    Barrier, System, Runtime, Custom, Exit,
};

enum InstructionFlags : uint16_t {
    InstructionFlagNone        = 0,
    InstructionFlagTaken       = 1 << 0,
    InstructionFlagRuntime     = 1 << 1,
    InstructionFlagFault       = 1 << 2,
    InstructionFlagExclusive   = 1 << 3,
    InstructionFlagAtomic      = 1 << 4,
    InstructionFlagVector      = 1 << 5,
    InstructionFlagCustom      = 1 << 6,
    InstructionFlagBrk0Exit    = 1 << 7,
};

enum class MemoryAccessKind : uint8_t {
    Load = 1,
    Store = 2,
    Atomic = 3,
};

enum MemoryAccessFlags : uint8_t {
    MemoryAccessFlagNone                   = 0,
    MemoryAccessFlagExclusive              = 1 << 0,
    MemoryAccessFlagSplit                  = 1 << 1,
    MemoryAccessFlagUnaligned              = 1 << 2,
    MemoryAccessFlagFailedStoreConditional = 1 << 3,
    MemoryAccessFlagExtData                = 1 << 4,
};
```

还需注意以下语义约定：

- `block_id + op_index` 适合用作对齐不同 Trace 的主键，`pc` 主要用于定位和可读性
- 一条指令可以关联多条内存记录，比如 pair、split、unaligned、atomic 等场景
- `mem_count == 0` 时，`mem_index` 必为 `0xffffffff`
- `mem_count != 0` 时，`mem_index` 表示 `RawBptfMemRecord[]` 的起始下标，不是字节 offset
- `DC ZVA` 导出到 BPTF 时会展开成多条普通 `Store` 记录
- 寄存器快照、context snapshot 和字符串元数据当前不单独落到 `.bptf`，这些信息仍保留在 `.bpraw`

</details>
