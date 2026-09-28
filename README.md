# PCILeech FPGA Firmware — Genesys Logic GL9755 SD Host Controller Emulation

PCILeech FPGA firmware that presents the FPGA as a **Genesys Logic GL9755 PCIe SD
Host Controller** — VID `0x17a0` / DID `0x9755`, class code `0x080501` (SD Host
Controller), 4 KB BAR0 with a modelled register set, interrupt registers and a
PCIe / MSI / PM capability chain.

---
Discord: @JinJinovo

# English

## Overview

This is a PCILeech FPGA project scaffold built on
[pcileech-fpga](https://github.com/ufrisk/pcileech-fpga) by Ulf Frisk (tops from
version 4.9 up to 4.15). On the target system the device enumerates as a
**Genesys Logic GL9755 SD Host Controller** instead of the Xilinx default ID:

- `lspci` reports `17a0:9755`, class code `0x080501`, revision `0x01`,
  subsystem `17a0:9755`, BAR0 = 4 KB memory BAR.
- The configuration space carries a full capability chain:
  PCI Express (`0x80`) → **MSI** (`0xE0`) → Power Management (`0xF8`).
  The MSI capability is programmed with Message Control `0x0081`
  (MSI enable + 64-bit capable) and Message Address `0xfee00004`.
- BAR0 is implemented by `pcileech_bar_impl_GL9755` in
  `src/pcileech_tlps128_bar_controller.sv` — it answers the register accesses a
  GL9755 would answer.

The FPGA DMA path is untouched: PCILeech on the host still talks to the TLP DMA
engine; the emulated SD controller registers are independent of it.

## Features

- **GL9755 PCIe identity** — VID/DID `17a0:9755`, class code `0x080501`, revision
  `0x01`, subsystem `17a0:9755`, Command `0x0407`, Status `0x0010` (capability
  list present), BAR0 4 KB memory, BAR1–BAR5 unimplemented, BAR6 option ROM.
- **PCIE / MSI / PM capability chain** in the preloaded configuration space
  (`ip/pcileech_cfgspace.coe`), MSI capable of 64-bit message addresses.
- **GL9755 BAR0 register model** — a read table of preloaded values covering the
  control / status / card-interface register window (`0x0004` … `0x050c`), plus:
  - `0x0040` interrupt status (read / write-1-clear) and `0x0044` interrupt mask;
  - `0x0050`…`0x005c` MSI control / address low / address high / data registers
    (enable, 64-bit capable and multi-message bits are decoded).
- **Periodic interrupt source** — the model raises `int_status[0]` roughly every
  1,000,000 clock cycles and, if `int_mask[0]` is set with MSI enabled, queues an
  MSI request.
- **Board scaffold for 8 Artix-7 projects** — 35T (PCIeScreamer / PCIeSquirrel /
  Screamer M.2), 75T (Enigma X1 / Captain / Immortal) and the 100T TB x4 design.
- **Two preloaded configuration spaces are shipped**:
  - `ip/pcileech_cfgspace.coe` — Genesys Logic GL9755 SD Host Controller
    (`17a0:9755`, class `0x080501`), used by every project except the 100T one;
  - `ip/100T/pcileech_cfgspace.coe` — Realtek RTL8168 Gigabit Ethernet
    (`10ec:8168`, class `0x020000`), used by the 100T project only.
- **Upstream PCILeech building blocks** — TLP DMA engine, BAR PIO read/write
  engines, FIFO / COM control network, FT601 USB3 interface and shadow
  configuration space handling.

## Supported boards

| Board | Generate script | Tcl script | Device | Top module |
|---|---|---|---|---|
| PCIeScreamer / Immortal 35T | `generate – immortal - 35T.bat` | `vivado_generate_project_immortal_35T.tcl` | `xc7a35tfgg484-2` | `pcileech_pciescreamer_top` |
| PCIeSquirrel 35T | `generate - squirrel.bat`, `生成35T文件夹.bat` | `vivado_generate_project_squirrel.tcl` | `xc7a35tfgg484-2` | `pcileech_squirrel_top` |
| Screamer M.2 35T | `generate - m2.bat` | `vivado_generate_project_m2.tcl` | `xc7a35tcsg325-2` | `pcileech_screamer_m2_top` |
| Enigma X1 75T | `generate – enigma - x1.bat` | `vivado_generate_project_enigma_x1.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Captain 75T | `generate – captain - 75T.bat` | `vivado_generate_project_captain_75T.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Immortal 75T | `generate – immortal - 75T.bat` | `vivado_generate_project_immortal_75T.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Immortal 75Ts | `generate - immmortal - 75Ts .bat` | `vivado_generate_project_immortal_75Ts.tcl` | `xc7a75tfgg484-2` | `pcileech_squirrel_top` |
| TB x4 100T | `generate – 100T.bat` | `vivado_generate_project_100t.tcl` | `xc7a100tfgg484-2` | `pcileech_tbx4_100t_top` |

Top module versions: PCIeScreamer 4.9, Squirrel / Screamer M.2 / Enigma X1 4.13,
TB x4 4.15.

## Repository layout

```
src/                            firmware RTL (board tops, TLP engines, GL9755 BAR0 model, FT601)
ip/                             Vivado IP cores + GL9755 configuration space (BRAM / DROM / writemask)
ip/100T/                        IP set + Realtek RTL8168 configuration space used by the 100T project
pcie_7x/                        Xilinx PCIe 7-series core wrappers (pcie_7x_0*)
pcie_7x/zdma/                   PCIe core wrapper variant used by the 100T project
vivado_generate_project_*.tcl   project generation scripts (8 boards)
*.bat                           convenience launchers (Vivado path is hard-coded, edit it)
```

## Build (Vivado 2023.2 / 2024.2)

1. Install Xilinx Vivado WebPACK 2023.2 or later.
2. Edit the `vivado` path inside the `.bat` launcher you want to use (they point at
   `C:\Xilinx\Vivado\...`, `D:\Vivado\...` or `E:\Xilinx\Vivado\...` by default),
   or call Vivado directly.
3. Generate the project, e.g.:
   `vivado -source vivado_generate_project_squirrel.tcl -notrace -nolog -nojournal`
4. Open the generated project and run *Generate Bitstream*. A full synthesis +
   implementation takes roughly one hour.
5. Flash the produced `.bin` to the FPGA board's SPI flash.

Notes:

- Keep the checkout in a short path (e.g. `C:\Temp`) — long paths make Vivado
  builds fail. Forward slashes are required inside the Tcl scripts.
- The 100T script references `ip/100T/...` while the directory is named `ip/100t`.
  Windows is case-insensitive so it works there, but on Linux the path has to be
  fixed (or the directory renamed) before building.

## Verification

- Target system reports a Genesys Logic GL9755 SD Host Controller
  (`17a0:9755`, class `0x080501`); `lspci -d 17a0:9755 -xxxx` shows the full
  configuration space including the PCIe / MSI / PM capability chain.
- Reads from BAR0 return the preloaded GL9755 register values; writes to the
  interrupt status/mask registers and the MSI control registers are latched.
- PCILeech host-side connection works as usual — FPGA DMA is handled by the TLP DMA
  engine and is independent of the emulated SD controller.

## Credits

Special thanks to:

- **ufrisk** — [PCILeech](https://github.com/ufrisk/pcileech) / [pcileech-fpga](https://github.com/ufrisk/pcileech-fpga), the upstream project this firmware is built on.
- **Dmytro Oleksiuk (@d_olex)** — original FIFO/control design ideas in `pcileech_fifo.sv`.

## Disclaimer

This project is provided for learning, research and authorized security testing only.
Use it only on hardware and systems you own or are explicitly authorized to test. Do
not use it for illegal activity, unauthorized access, or any kind of harm. You are
responsible for complying with all applicable laws and terms of service. Use at your
own risk.

---

# PCILeech FPGA 固件 —— 模拟 Genesys Logic GL9755 SD 卡控制器

本固件基于 PCILeech FPGA，把 FPGA 在目标系统上模拟成一块 **Genesys Logic GL9755
PCIe SD 卡控制器** —— VID `0x17a0` / DID `0x9755`，类码 `0x080501`（SD Host
Controller），BAR0 4KB 寄存器模型，带中断寄存器与 PCIe / MSI / PM capability 链。

---
Discord: @JinJinovo

# 中文说明

## 概述

这是一个基于 Ulf Frisk 的 [pcileech-fpga](https://github.com/ufrisk/pcileech-fpga)
（顶层版本 4.9 ~ 4.15）搭建的 PCILeech FPGA 工程脚手架。目标机上设备不再使用
Xilinx 默认 ID，而是枚举为一块 **Genesys Logic GL9755 SD 卡控制器**：

- `lspci` 报告 `17a0:9755`，类码 `0x080501`，修订号 `0x01`，子系统 `17a0:9755`，
  BAR0 为 4KB memory BAR。
- 配置空间带完整的 capability 链：PCI Express（`0x80`）→ **MSI**（`0xE0`）→
  Power Management（`0xF8`）。MSI capability 的 Message Control 为 `0x0081`
  （MSI enable + 64-bit capable），Message Address 为 `0xfee00004`。
- BAR0 由 `src/pcileech_tlps128_bar_controller.sv` 中的
  `pcileech_bar_impl_GL9755` 实现，按 GL9755 的行为应答寄存器访问。

FPGA 的 DMA 通路保持不变：主机侧 PCILeech 仍然与 TLP DMA 引擎通信，模拟的 SD
控制器寄存器与其相互独立。

## 特性

- **GL9755 PCIe 身份** —— VID/DID `17a0:9755`、类码 `0x080501`、修订号 `0x01`、
  子系统 `17a0:9755`、Command `0x0407`、Status `0x0010`（存在 capability 链）、
  BAR0 4KB memory、BAR1–BAR5 未实现、BAR6 为 option ROM。
- **PCIE / MSI / PM capability 链** 预录在配置空间中
  （`ip/pcileech_cfgspace.coe`），MSI 支持 64-bit 消息地址。
- **GL9755 BAR0 寄存器模型** —— 覆盖控制 / 状态 / 卡接口寄存器窗口
  （`0x0004` … `0x050c`）的预录只读值表，另外还包括：
  - `0x0040` 中断状态（读 / 写 1 清零）与 `0x0044` 中断掩码；
  - `0x0050`…`0x005c` MSI 控制 / 地址低 / 地址高 / 数据寄存器
    （解析 enable、64-bit capable、multi-message 等位）。
- **周期性中断源** —— 模型大约每 1,000,000 个时钟周期置位一次 `int_status[0]`，
  若 `int_mask[0]` 已使能且 MSI 已开启，则产生一次 MSI 请求。
- **8 个 Artix-7 板卡工程脚手架** —— 35T（PCIeScreamer / PCIeSquirrel /
  Screamer M.2）、75T（Enigma X1 / Captain / Immortal）以及 100T TB x4 设计。
- **随附两套预录配置空间**：
  - `ip/pcileech_cfgspace.coe` —— Genesys Logic GL9755 SD 卡控制器
    （`17a0:9755`，类码 `0x080501`），除 100T 外的所有工程使用；
  - `ip/100T/pcileech_cfgspace.coe` —— Realtek RTL8168 千兆网卡
    （`10ec:8168`，类码 `0x020000`），仅 100T 工程使用。
- **上游 PCILeech 基础模块** —— TLP DMA 引擎、BAR PIO 读/写引擎、FIFO / COM
  控制网络、FT601 USB3 接口以及配置空间 shadow 处理。

## 支持的板卡

| 板卡 | 生成脚本 | Tcl 脚本 | 器件 | 顶层模块 |
|---|---|---|---|---|
| PCIeScreamer / Immortal 35T | `generate – immortal - 35T.bat` | `vivado_generate_project_immortal_35T.tcl` | `xc7a35tfgg484-2` | `pcileech_pciescreamer_top` |
| PCIeSquirrel 35T | `generate - squirrel.bat`、`生成35T文件夹.bat` | `vivado_generate_project_squirrel.tcl` | `xc7a35tfgg484-2` | `pcileech_squirrel_top` |
| Screamer M.2 35T | `generate - m2.bat` | `vivado_generate_project_m2.tcl` | `xc7a35tcsg325-2` | `pcileech_screamer_m2_top` |
| Enigma X1 75T | `generate – enigma - x1.bat` | `vivado_generate_project_enigma_x1.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Captain 75T | `generate – captain - 75T.bat` | `vivado_generate_project_captain_75T.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Immortal 75T | `generate – immortal - 75T.bat` | `vivado_generate_project_immortal_75T.tcl` | `xc7a75tfgg484-2` | `pcileech_enigma_x1_top` |
| Immortal 75Ts | `generate - immmortal - 75Ts .bat` | `vivado_generate_project_immortal_75Ts.tcl` | `xc7a75tfgg484-2` | `pcileech_squirrel_top` |
| TB x4 100T | `generate – 100T.bat` | `vivado_generate_project_100t.tcl` | `xc7a100tfgg484-2` | `pcileech_tbx4_100t_top` |

顶层版本：PCIeScreamer 4.9，Squirrel / Screamer M.2 / Enigma X1 4.13，TB x4 4.15。

## 目录结构

```
src/                            固件 RTL（板卡顶层、TLP 引擎、GL9755 BAR0 模型、FT601）
ip/                             Vivado IP 核 + GL9755 配置空间（BRAM / DROM / 写掩码）
ip/100T/                        100T 工程使用的 IP 集合 + Realtek RTL8168 配置空间
pcie_7x/                        Xilinx PCIe 7 系列核包装（pcie_7x_0*）
pcie_7x/zdma/                   100T 工程使用的 PCIe 核包装变体
vivado_generate_project_*.tcl   工程生成脚本（8 块板卡）
*.bat                           便捷启动脚本（Vivado 路径为硬编码，需要自行修改）
```

## 构建（Vivado 2023.2 / 2024.2）

1. 安装 Xilinx Vivado WebPACK 2023.2 或更高版本。
2. 修改要使用的 `.bat` 脚本里的 `vivado` 路径（默认指向 `C:\Xilinx\Vivado\...`、
   `D:\Vivado\...` 或 `E:\Xilinx\Vivado\...`），或直接调用 Vivado。
3. 生成工程，例如：
   `vivado -source vivado_generate_project_squirrel.tcl -notrace -nolog -nojournal`
4. 打开生成的工程点击 *Generate Bitstream*。完整综合 + 实现大约需要一小时。
5. 将生成的 `.bin` 烧录到 FPGA 板卡的 SPI flash。

注意事项：

- 请把代码放在较短的路径下（例如 `C:\Temp`），路径过长会导致 Vivado 构建失败；
  Tcl 脚本内的路径必须使用正斜杠。
- 100T 脚本里引用的是 `ip/100T/...`，而目录名是 `ip/100t`。Windows 大小写不敏感
  所以能正常构建；在 Linux 上需要先修正路径（或重命名目录）。

## 验证

- 目标机报告 Genesys Logic GL9755 SD Host Controller（`17a0:9755`，类码
  `0x080501`）；`lspci -d 17a0:9755 -xxxx` 可查看完整配置空间，包括 PCIe / MSI /
  PM capability 链。
- 读取 BAR0 会返回预录的 GL9755 寄存器值；对中断状态/掩码寄存器与 MSI 控制寄存器
  的写入会被锁存。
- PCILeech 主机侧按原流程连接即可 —— FPGA 的 DMA 由 TLP DMA 引擎完成，与模拟的 SD
  控制器无关。

## 致谢

特别感谢以下项目和贡献者：

- **ufrisk** —— [PCILeech](https://github.com/ufrisk/pcileech) / [pcileech-fpga](https://github.com/ufrisk/pcileech-fpga)，本固件所基于的上游项目。
- **Dmytro Oleksiuk (@d_olex)** —— `pcileech_fifo.sv` 中 FIFO / 控制设计的原始思路。

## 免责声明

本项目仅供学习、研究与获得授权的安全测试使用。请仅在你自己拥有或获得明确授权测试的
硬件与系统上使用。不得用于违法活动、未授权访问或任何形式的损害。你需要自行遵守所有
适用法律与服务条款，使用风险自负。
