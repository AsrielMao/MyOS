# MyOS 开发环境准备记录说明

> 项目：手写 x86 操作系统内核（学习用途，非商业产品）
> 目标平台：x86 / 32 位（i386），运行于 QEMU 模拟机
> 开发宿主：Windows 11 + WSL2（Ubuntu），不使用虚拟机、不使用双系统
> 记录日期：2026-10-21
> 文档定位：本文是环境搭建过程的汇总归档，替代早期草稿 `~/myos/docs/environment.md`；后续换机、重装 WSL 时照本文复现即可。

---

## 1. 环境总览

| 项目  | 取值  | 说明  |
| --- | --- | --- |
| 宿主系统 | Windows 11 | 只作为桌面与 WSL2 的承载，不直接参与编译 |
| 虚拟化层 | **WSL2**（Windows Subsystem for Linux 2） | 真正跑 Linux 内核的轻量虚拟机 |
| Linux 发行版 | **Ubuntu**（codename `resolute`） | 26.04 开发版分支 |
| WSL 版本 | **VERSION 2** | `wsl -l -v` 中 VERSION 列为 `2`，且为默认发行版（`*`） |
| 工作目录 | `/home/asriel/myos` | **全部开发在此目录内进行**，禁止放 `/mnt/c` |
| 图形输出 | WSLg，`DISPLAY=:0` | QEMU 窗口直接弹在 Windows 桌面，无需 X Server |
| 目标架构 | x86（32 位，i386） | 先 32 位保护模式，暂不碰 64 位 |
| 调试方式 | QEMU GDB Stub + GDB | `-s -S` + `target remote :1234` |
| 磁盘可用空间 | 约 1024 GB（WSL 虚拟盘） | 本次安装占用约 675 MB |

### 1.1 版本清单（实测，非推测）

| 工具  | 实测版本 |
| --- | --- |
| NASM | `NASM version 3.01` |
| GCC | `gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0` |
| GNU ld（binutils） | `GNU ld (GNU Binutils for Ubuntu) 2.46` |
| QEMU | `QEMU emulator version 10.2.1 (Debian 1:10.2.1+ds-1ubuntu3.2)` |
| GDB | `GNU gdb (Ubuntu 17.1-2ubuntu1) 17.1` |
| Make | 随 `build-essential` 安装（4.4.1） |

> 说明：Ubuntu `resolute` 属于开发版分支，工具链版本比常见教程（GCC 9/11、QEMU 6/8）新很多。实测编译、链接、启动均正常，**无需降级**，也不必照抄旧教程的版本号。

---

## 2. 关键决策与理由

搭建前先定下四条原则，后续所有操作都围绕它们展开。

### 2.1 用 WSL2，不用虚拟机 / 双系统

- **快速开始，减少预备时间**：不需要 VirtualBox / VMware，也不需要分区装双系统；WSL2 启动只要几秒，随开随用。
- **工具链齐全**：NASM、GCC、binutils、QEMU、GDB 在 Ubuntu 仓库里都是一条 `apt install` 的事。
- **图形可用**：Windows 11 自带 **WSLg**，QEMU 这类依赖 X11 的程序直接弹窗，**不必**再装 VcXsrv / Xming。
- **文件系统原生**：ext4 保留权限位、大小写敏感、LF 换行，避免链接脚本和 Makefile 在 `/mnt/c` 上出怪问题。

> **必须是 WSL2，不能是 WSL1。** WSL1 只是 syscall 翻译层，不支持完整内核特性，也无法承载 QEMU 的虚拟化/模拟运行。确认命令：PowerShell 中 `wsl -l -v`。

### 2.2 暂不安装交叉编译器 `x86_64-elf-gcc`

- 宿主 Ubuntu 的 `gcc -m32` 配合 `-ffreestanding -nostdlib`，足以编译出运行在 QEMU 里的 32 位裸机内核。
- 纯交叉工具链的收益是「不依赖宿主 libc」，但安装步骤多，容易在第一步就消耗热情。
- 策略：**先用 `gcc -m32` 跑通「引导 → 打印字符」闭环**；等真的遇到宿主环境导致的链接怪问题，再回来配交叉工具链。

### 2.3 代码一律放在 WSL 的 ext4 文件系统（`~`）下

`/mnt/c` 是 Windows 文件系统经 9P 协议挂载的，有三个坑：

1. **性能**：文件 I/O 极慢，编译大型项目明显卡顿；
2. **权限**：不保留 Linux 权限位与可执行位，`make`、链接脚本易出怪问题；
3. **换行符**：Windows 默认 CRLF，链接器脚本（`linker.ld`）会对 CRLF 报错。

因此：从 `/mnt/c/Users/Administrator` 立即 `cd ~`，所有工作在 `/home/asriel/myos` 内完成。

### 2.4 不折腾 KVM 加速

- WSL2 内 KVM 通常不可用，QEMU 默认回退到 **TCG（纯软件模拟）**。
- 对 512 字节引导扇区和教学内核，TCG 性能绰绰有余。
- **不要**为了启用 KVM 去改 `bcdedit` 或调整 Hyper-V 设置，那可能破坏 WSL2 自身的运行基础。

---

## 3. 工具链清单与职责

### 3.1 一次性安装命令

```bash
sudo apt update
sudo apt install -y build-essential nasm gcc-multilib qemu-system-x86 gdb
```

本次安装规模：升级 5 个包、新装 199 个包、下载约 195 MB、占用磁盘约 675 MB。

### 3.2 每个包为什么需要

| 包名  | 是什么 | 在本项目中的用途 | 本项目中的具体出场时机 |
| --- | --- | --- | --- |
| `nasm` | Netwide Assembler，汇编器 | 把 `.asm` 翻译成裸机器码 `-f bin` | 编译引导扇区 `boot.asm` → `boot.bin`；后续所有实模式/保护模式入口代码 |
| `build-essential` | 元包：`gcc`、`g++`、`make`、`binutils`、`libc6-dev` | 提供 C 编译器、链接器 `ld`、构建工具 `make` | 编译 `kernel.c` → `kernel.o`；`ld` 按链接脚本生成 `kernel.elf`；`make` 驱动整个构建 |
| `gcc-multilib` | 32 位编译支持（`-m32`、`lib32*`） | 产出 32 位目标代码 | 内核早期运行在 32 位保护模式，所有 C 代码都需 `-m32` |
| `qemu-system-x86` | QEMU 系统模拟器（含 `qemu-system-i386`、SeaBIOS、`qemu-img`） | 模拟一台完整 x86 机器：CPU、内存、磁盘、BIOS | 从 `disk.img`「开机」，验证每一步成果；崩溃只影响虚拟机 |
| `gdb` | GNU 调试器 | 通过 QEMU 内建 GDB Stub 做源码级调试 | `qemu-system-i386 -s -S` + `target remote :1234`，单步、看寄存器、定位 Triple Fault |

> `binutils` 里的 `objcopy` 也是链条上的关键一环（把 ELF 剥成裸二进制），它随 `build-essential` 一起装好，无需单独安装。
> `xxd`（十六进制查看）随 `vim-common` 提供；若缺失可 `sudo apt install xxd`，或改用 `od -A x -t x1z`。

### 3.3 工具在「源码 → 磁盘映像 → QEMU」中的位置（速查）

```
boot.asm ──NASM(-f bin)──▶ boot.bin (512B, 末尾 55 AA)
                                   │
kernel.c ──GCC(-m32 -ffreestanding -c)──▶ kernel.o
                                   │
                              ld(-T linker.ld, -m elf_i386)──▶ kernel.elf
                                   │
                              objcopy(-O binary)──▶ kernel.bin
                                   │
              cat boot.bin kernel.bin > disk.img
                                   │
                    qemu-system-i386 -drive format=raw,file=disk.img
                                   ▼
        BIOS 读第 0 扇区 → 0x7C00 → 我们的引导代码 → 读磁盘 → 跳内核
```

| 阶段  | 产物  | 谁产生 | 谁消费 |
| --- | --- | --- | --- |
| 汇编  | `boot.bin` | NASM | BIOS（读第 0 扇区到 `0x7C00`） |
| 编译  | `kernel.o` | GCC | `ld` |
| 链接  | `kernel.elf` | `ld` + `linker.ld` | `objcopy` |
| 剥壳  | `kernel.bin` | `objcopy` | 我们自己写的引导程序（`int 0x13` 读盘） |
| 打包  | `disk.img` | `cat` / `dd` | QEMU（当作裸硬盘） |
| 运行  | 内存中的指令 | QEMU 模拟的 CPU | 屏幕 / GDB |

与普通应用开发最大的差别：**普通程序由操作系统读 ELF/PE 头并加载；裸机内核没有这层，加载器要自己写。** BIOS 只帮最原始的一步——把第一个 512 字节扇区搬到 `0x7C00` 并跳过去。

---

## 4. 安装与验证实录

### 4.1 确认 WSL 版本

在 **Windows PowerShell**（不是 WSL 内部）执行：

```powershell
wsl -l -v
```

实测输出：

```
  NAME      STATE           VERSION
* Ubuntu    Stopped         2
```

`VERSION` 为 `2` 且带 `*`（默认发行版）→ 满足要求。

> 常见误区：`wsl2` 不是命令，正确命令就是 `wsl`。若 `wsl -l -v` 报错，先 `wsl --update`，再 `wsl --shutdown` 后重进。
> 若某发行版显示 VERSION 1，管理员 PowerShell 执行 `wsl --set-version Ubuntu 2`。

### 4.2 安装工具链

在 WSL 终端内执行（`sudo` 需要输入 WSL 用户密码）：

```bash
sudo apt update
sudo apt install -y build-essential nasm gcc-multilib qemu-system-x86 gdb
```

安装过程正常结束，无依赖冲突、无 held back 包。

### 4.3 建工作目录

```bash
cd ~
mkdir -p ~/myos
cd ~/myos
pwd          # 预期 /home/asriel/myos
```

### 4.4 四项验证（全部通过）

**① 工具版本**

```bash
nasm --version
gcc --version | head -1
ld --version  | head -1
qemu-system-i386 --version | head -1
gdb --version | head -1
```

实测结果见「1.1 版本清单」，五个命令均有正常输出。

**② 32 位编译能力（关键）**

```bash
echo 'int main(){return 0;}' > /tmp/t.c
gcc -m32 /tmp/t.c -o /tmp/t && echo "32-bit compile OK"
```

实测输出：`32-bit compile OK` → `gcc-multilib` 生效，后续 `-m32` 编译内核不会因缺 32 位头文件/库而失败。

**③ WSLg 图形通路**

```bash
echo "DISPLAY=$DISPLAY"
```

实测输出：`DISPLAY=:0` → WSLg 就绪，QEMU 可弹窗。
若为空：PowerShell 执行 `wsl --update` 后 `wsl --shutdown`，再重进 WSL。

**④ QEMU 冒烟测试**

```bash
qemu-system-i386 -display gtk
```

实测现象：Windows 桌面弹出 QEMU 窗口，显示 SeaBIOS 启动信息，最终停在：

```
No bootable device
```

这一行恰恰证明环境是**正确**的：

- QEMU 能启动虚拟 CPU ✅
- SeaBIOS 正常工作 ✅
- WSLg 图形通路正常 ✅
- 我们已经有了一台「什么都没装的裸机」 ✅

退出：`Ctrl+Alt+Q` 或直接关窗。
若报 `gtk initialization failed`，先试 `-display sdl`；仍不行再回到 `wsl --update`。

---

## 5. 工程目录规范

```
~/myos/
├── docs/       # 文档（环境说明、阶段设计记录）
├── boot/       # 引导扇区 / 引导加载程序（汇编）
├── kernel/     # 内核 C 代码与底层汇编
└── build/      # 编译产物：.o / .bin / .elf / .img（保持源码目录干净）
```

约定：

- 构建产物**只**写入 `build/`，源码目录不接受生成文件；
- 每个里程碑在 `docs/` 留一份简短设计记录（如 `01-boot-sector.md`），积累成开发日志；
- `build/` 可随时清空重建，应可被 `.gitignore` 忽略；
- 若用 Git，仓库建在 `~/myos`（ext4 内），不要在 `/mnt/c` 上初始化。

---

## 6. 踩坑记录与注意事项

| 现象  | 原因  | 处理  |
| --- | --- | --- |
| `'wsl2' 不是内部或外部命令` | `wsl2` 不是命令，只是版本代号 | 用 `wsl` 启动；用 `wsl -l -v` 查版本 |
| 初始落在 `/mnt/c/Users/Administrator` | WSL 默认继承 Windows 当前目录 | `cd ~` 回 home，项目建在 `~/myos` |
| 担心 WSL 里 QEMU 没窗口 | 误以为要装 X Server | Win11 自带 WSLg，`DISPLAY=:0` 即可弹窗 |
| 看到「WSL 中 KVM 受限」的讨论 | WSL2 内 KVM 通常不可用 | 用默认 TCG，学习足够，别改 Hyper-V/bcdedit |
| 后续若链接报错、符号找不到 | 默认 GCC 会引入宿主 libc | 加 `-ffreestanding -nostdlib`；仍不行再考虑交叉工具链 |
| 若链接脚本报语法错 | 文件被存成 CRLF | 确认文件在 ext4 内、编辑器用 LF |

**通用原则**：每一步都要在 QEMU 里有可见反馈（一个字符、一个颜色、一个计数），不要连续写几百行再一起调试。内核崩溃往往没有任何提示，`-s -S` + GDB 是定位 Triple Fault 的唯一有效手段。

---

## 7. 环境复现速查（换机 / 重装 WSL）

```bash
# 0) Windows PowerShell：确认 WSL2 与发行版
wsl -l -v

# 1) 进入 WSL，安装工具链
sudo apt update
sudo apt install -y build-essential nasm gcc-multilib qemu-system-x86 gdb

# 2) 建工作区
mkdir -p ~/myos/{docs,boot,kernel,build}
cd ~/myos

# 3) 三项验证
gcc -m32 -x c -o /tmp/t - <<<'int main(){return 0;}' && echo "32-bit OK"
echo "DISPLAY=$DISPLAY"          # 期望 :0
qemu-system-i386 -display gtk    # 期望弹出窗口并停在 No bootable device
```

三步全过，环境即复现。

---

## 8. 验收清单

- [x] `wsl -l -v` 显示发行版为 Ubuntu、VERSION 为 2
- [x] `build-essential / nasm / gcc-multilib / qemu-system-x86 / gdb` 安装成功
- [x] `nasm`、`gcc`、`ld`、`qemu-system-i386`、`gdb` 五个命令均有版本输出
- [x] `gcc -m32` 输出 `32-bit compile OK`
- [x] `DISPLAY=:0`，WSLg 正常
- [x] `qemu-system-i386 -display gtk` 弹出窗口并显示 `No bootable device`
- [x] 工作目录为 `~/myos`，已建立 `docs/ boot/ kernel/ build/`
- [x] 明确不在 `/mnt/c` 下开发
- [x] 明确暂不安装交叉编译器、暂不折腾 KVM

---

## 9. 下一步

环境已具备「写代码 → 编译 → 运行 → 调试」的完整闭环，下一步进入第一个里程碑：

1. 写 512 字节引导扇区 `boot/boot.asm`，用 BIOS `int 0x10` 打印 `Hello, my OS!`；
2. `nasm -f bin` 生成 `boot.bin`，用 `xxd` 确认恰好 512 字节且末尾为 `55 AA`；
3. `qemu-system-i386 -drive format=raw,file=build/boot.bin` 看到字符输出；
4. 再写 stage 2：用 `int 0x13` 把 `kernel.bin` 读进内存并跳转，进入 C 语言内核。
