# BinProfiler

BinProfiler 是一个基于二进制插桩工具（Frida）的性能分析和切片制作工具。当前beta版本提供了面向
Linux AArch64 场景下的trace抓取和切片制作功能。

## 当前形态

当前发布包包含两个核心组件：

- `binprofiler`：host 可执行程序，负责解析命令行参数、attach 目标进程、
  注入 agent，并下发采集配置。
- `libbinprofiler-agent.so`：被注入到目标进程内运行的 agent 动态库，负责执行
  trace 等核心能力。

## 当前支持范围

| CPU ISA \\ OS | `Ubuntu 22.04+`  |`OpenEuler 22.03+`  |
| --- | --- | --- |
| AArch64 | `trace` | `trace` |
| x86_64 | - | - | 

## 获取二进制

从本仓的 GitHub Releases 页面下载对应平台的二进制包：

下载后解压，保证 `binprofiler` 和 `libbinprofiler-agent.so` 位于同一目录，或在
运行时通过 `--agent` 显式指定 agent 路径。

## 运行原理

`binprofiler` 负责 attach 目标进程、注入 agent，并通过控制通道下发启动参数和
trace 请求；agent 常驻在目标进程内，执行具体采集动作并返回结果。

## Trace 功能说明

当前 `trace` 子命令支持把 agent 注入到指定进程，并在指定线程上抓取 basic
block trace。最基本的调用形式：

```sh
./binprofiler [--output-dir <dir>] trace <pid> [trace-options]
```

默认情况下：

- 输出目录是 `./out`
- 未指定线程时，默认抓取主线程，也就是 Linux 上通常 `tid == pid`
- host 默认从自身所在目录加载 `libbinprofiler-agent.so`

如需显式指定 agent：

```sh
./binprofiler --agent /path/to/libbinprofiler-agent.so trace <pid>
```

### 指令支持情况

当前 trace / elfgen 对 AArch64 内存相关指令的支持情况如下：

| 范围 | 代表类别 | Trace probe | Elfgen | 当前状态 |
| --- | --- | --- | --- | --- |
| 标量普通 ld/st | `ScalarRegMem` | 支持 | 支持 | 稳定 
| 标量 pair | `PairRegMem + Scalar` | 支持 | 支持 | 稳定 
| 标量 literal load | `LoadLiteral` | 支持 | 支持 | 稳定 
| 标量 ordered / RCPC | `LDAR/STLR/LDLAR/STLLR/LDAPR/LDAPUR/STLUR` | 支持 | 支持 | 稳定 
| 标量 exclusive | `Exclusive` | 支持 | 支持 | 稳定 
| 标量 LSE atomic | `AtomicRmw` | 支持 | 支持 | 稳定 
| NEON/FP 单寄存器 | `VectorRegMem` | 支持 | 支持 | 稳定 
| NEON/FP pair | `PairRegMem + Vector` | 支持 | 支持 | 稳定 
| NEON/FP literal load | `LoadLiteral + Vector` | 支持 | 支持 | 稳定 
| NEON full-register structure | `SimdStructure` | 支持 | 支持 | 稳定 
| pair ordered / pair RCPC | `STILP/LDIAPP/STLP/LDAP/LDAPP` | 不支持 | 不支持 | 无支持计划 
| NEON/FP RCPC | `LDAP1`、SIMD/FP `LDAPUR/STLUR` | 不支持 | 不支持 | 无支持计划 
| SVE memory | `ldr/str z`、`ldr/str p`、predicated contiguous `ld1/st1/ldnt1/stnt1`、replicate `ld1r* / ld1rq*`、contiguous tuple `ld1..ld4/st1..st4`（含 `q` tuple）、`ld1/st1 mz`、indexed-address gather/scatter、`ldff1* / ldnf1*` | 支持 | 支持 | 未充分验证 
| SVE 未支持 | `ldnt1b/h/w/d mz`、`stnt1b/h/w/d mz`、`ld1sb/sh/sw`、`ld1rob/roh/row/rod`、`ld1rsb/rsh/rsw`、其余尚未接入的 `ai_* / ar_* / bz_*` 地址形态 | 不支持 | 不支持 | 按需支持 
| PRFM | `Prefetch` | 不生成 payload | 不适用 | 按设计跳过
| MOPS | 专用 memory 流水指令 | 不支持 | 不支持 | 无支持计划

### 权限与运行限制

BinProfiler 通过 Frida attach 并注入目标进程，在 Linux 上依赖 `ptrace` 权限。
如果本机启用了较严格的 `ptrace_scope` 限制，即使是同用户进程，也可能无法
attach 成功。

使用前建议确认：

- 当前用户对目标进程具备 attach 权限
- 目标架构为 AArch64
- 运行环境允许注入动态库

### SVE的Vector Length说明

在抓取Trace时，可以允许每一个ld st指令都有不同的Vector Length，但在制作elf时，要求整个elf只有一个固定的Vector Length。这主要是因为制作出来的elf不会包含syscall，是纯用户态指令流，所以无法更改vector length。

### 常用选项：

- `--tid`：指定目标线程（默认是主线程）
- `--instruction-limit`：限制采集指令数（默认5000万指令），实际采集的指令数会略多于指定数量
- `--warmup-ms`：初始阶段需要将原始二进制翻译成带插桩代码的二进制，该翻译过程对目标程序性能有影响，利用该选项来避免在翻译阶段抓trace，降低失真（默认1ms）
- `--trigger-pc` / `--trigger-hit`：命中指定 PC 的第 N 次后开始录制（**推荐使用该选项精确控制trace的抓取行为**）
- `--trigger-timeout-s`: 等待trigger pc命中的时间上限，默认10s
- `--end-pc`：为 `elfgen` 切片非正常程序，结束依赖补偿代码，如果结束PC在热点循环内，那会产生大量补偿代码（每次执行到该PC都需要检查是否应该结束执行），导致生成的切片失真，利用该选项标记优选结束位置，比如一个大函数的结束PC。（**强烈推荐使用该选项**）
- `--trace-context-sve`: 记录sve寄存器的初始值，默认关闭，在目标程序有sve指令的情况下需开启
- `--trace-context-neon`: 记录NEON/FP寄存器的初始值，默认开启，只有在确定目标程序只有标量指令的情况下才可以关闭
- `--trace-jit`：用于存在self-modifying行为的目标程序，比如Python程序（未严格测试）
- `--emit-trace`：生成二进制trace文件，格式见BPTF说明
- `--emit-elf`：在trace抓取成功后自动生成 ELF
- `--merge-elf`: 生成的elf可能会有超限的segment，利用该选项降低segment数量，使elf可在真机上运行
- `--to39bit`: 利用48bit VA地址下抓取的trace生成运行在39bit VA环境的切片，用于在模拟器上运行（**开发中的实验特性**）

### 常见用法：

```sh
# 1. 抓主线程，输出到默认 ./out
./binprofiler trace 12345

# 2. 指定线程并限制采集窗口
./binprofiler --output-dir /tmp/binprofiler-out trace 12345 --tid 12360 \
  --instruction-limit 2000000

# 3. 等待 trigger PC，第 3 次命中开始录制，等待trigger命中的上限为1s
./binprofiler trace 12345 \
  --trigger-pc 0x400800 --trigger-hit 3 \
  --trigger-timeout-s 1

# 4. 保留 NEON/FP context，并额外开启 SVE context
./binprofiler --output-dir /tmp/binprofiler-sve trace 12345 \
  --trace-context-sve

# 5. trace 成功后只导出展开后的 BPTF
./binprofiler --output-dir /tmp/binprofiler-bptf trace 12345 \
  --emit-trace \
  --trigger-pc 0x401000

# 6. trace 成功后自动生成 ELF，在trace内最后一个end-pc处计结束ELF的执行，额外导出 trace
./binprofiler --output-dir /tmp/binprofiler-replay trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf \
  --emit-trace

# 7. 对真实 Linux loader 限制较严的场景，压低 PT_LOAD segment 数
./binprofiler --output-dir /tmp/binprofiler-merged trace 12345 \
  --emit-elf \
  --merge-elf
```

## Elfgen 功能说明

如果已经有 `.bpraw` trace，也可以单独运行 `elfgen` 生成切片 ELF。最基本的
调用形式：

```sh
./binprofiler [--output-dir <dir>] elfgen <trace.bpraw> [elfgen-options]
```

默认情况下：

- 输出目录是 `./out`
- 输入文件是已有的 `.bpraw`
- 生成的 ELF 会根据 trace 内容自动命名并写入输出目录

### 常用选项：

- `--emit-trace`：在生成 ELF 的同时导出展开后的 `.bptf`
- `--ignore-end-pc`：忽略 trace 里的 `end-pc` 标记，只按默认回退策略选择结束位置（**强烈不建议使用**）
- `--merge-elf`：降低最终 ELF 的 segment 数量，使其更容易在真实环境加载运行
- `--to39bit`：同时生成普通 48-bit VA 切片 ELF 和面向 39-bit VA 环境的切片 ELF（**开发中的实验特性**）

### 常见用法：

```sh
# 1. 从已有 trace 生成 ELF
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw

# 2. 生成 ELF，并同时导出展开后的 BPTF
./binprofiler --output-dir /tmp/binprofiler-elf elfgen \
  ./out/binprofiler-trace-12345-12345.bpraw \
  --emit-trace

# 3. 忽略 trace 里的 end-pc 标记
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw \
  --ignore-end-pc

# 4. 压低 segment 数量，使生成的 ELF 更容易在真机运行
./binprofiler elfgen ./out/binprofiler-trace-12345-12345.bpraw \
  --merge-elf
```

### 39-bit VA 切片功能

`--to39bit` 用于把在 48-bit VA Linux 环境下抓取的 `.bpraw` trace 投影成可在
39-bit VA 环境中运行的切片 ELF，主要用于需要限制虚拟地址宽度的模拟器回放场景。
开启后，`elfgen` 会对 trace 中的 PC、SP、内存访问地址以及可识别的指针来源做
39-bit 地址投影；同时会保留一份普通 48-bit VA baseline ELF，并生成 39-bit
版本的 ELF。

可以在 trace 后自动生成 48-bit 和 39-bit 两份 ELF：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit trace 12345 \
  --trigger-pc 0x401000 \
  --end-pc 0x4019f0 \
  --emit-elf \
  --to39bit
```

也可以对已有 `.bpraw` 单独执行 `elfgen`：

```sh
./binprofiler --output-dir /tmp/binprofiler-39bit elfgen \
  ./out/binprofiler-trace-12345-12345.bpraw \
  --to39bit
```

未开启 `--to39bit` 时，默认生成普通 48-bit VA 切片；开启后输出文件名会带有
`-48bit.elf` 和 `-39bit.elf` 后缀，分别对应普通 baseline 和 39-bit 版本。
该功能仍属于实验特性；遇到回放差异时，优先用同次生成的普通 48-bit ELF 确认
trace 本身是否可正常生成和运行。39-bit 生成过程中可能会额外留下用于验证的
`-39bit-probe_map.txt`、`-39bit-tagprobe-*_map.txt` 等中间 map 文件。

## 输出文件

当前自动生成的产物文件名规则如下：

```text
<output-dir>/binprofiler-trace-<pid>-<tid>.bpraw
<output-dir>/binprofiler-trace-<pid>-<tid>.bptf
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-39bit.elf
<output-dir>/binprofiler-trace-<pid>-<tid>-48bit_map.txt
<output-dir>/host.log
```

是否生成 `.bptf` 和 `.elf` 取决于是否启用了 `--emit-trace` 和 `--emit-elf`。
未开启 `--to39bit` 时只生成 `-48bit.elf`；开启 `--to39bit` 时会同时生成
`-48bit.elf` 和 `-39bit.elf`。普通 48-bit ELF 会生成对应的 `_map.txt` 文件。

## BPTF Trace 格式

`.bptf` 是从 `.bpraw` 展开的二进制指令流导出格式，用于后续 diff、回放校验和
离线分析。它不是原始捕获格式；完整元数据仍然以 `.bpraw` 为准。

当前实现使用小端紧凑二进制布局，文件按顺序由三部分组成：

```text
RawBptfHeader
RawBptfInstRecord[inst_count]
RawBptfMemRecord[mem_count]
```

可以这样理解：

- 每条 `RawBptfInstRecord` 对应一条动态执行的指令
- 如果该指令有关联的内存访问，则通过 `mem_index + mem_count` 指向连续的
  `RawBptfMemRecord[]`
- 当前版本对NEON和SVE的内存指令的支持有限
- 无效索引统一使用 `0xffffffff`

当前版本读取时，建议先校验下面这些固定条件；对当前实现来说，这些条件必须满足：

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

如果这些条件有任意一条不满足，最合理的处理就是把它当成“不是当前版本的
`.bptf`”直接拒绝。

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

如果只是自己写一个 reader，最小读取流程是：

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

当前语义上还需要注意：

- `block_id + op_index` 适合用作对齐不同 trace 的主键，`pc` 主要用于定位和可读性
- 一条指令可以关联多条内存记录，比如 pair、split、unaligned、atomic 等场景
- `mem_count == 0` 时，`mem_index` 必为 `0xffffffff`
- `mem_count != 0` 时，`mem_index` 表示 `RawBptfMemRecord[]` 的起始下标，不是字节 offset
- `DC ZVA` 导出到 BPTF 时会展开成多条普通 `Store` 记录
- 寄存器快照、context snapshot 和字符串元数据当前不单独落到 `.bptf`，这些信息仍保留在 `.bpraw`
