![language](https://img.shields.io/badge/language-verilog_(IEEE1364_2001)-9A90FD.svg) ![RDMA](https://img.shields.io/badge/RDMA-RoCE_v2_(ERNIC)-orange.svg) ![RF](https://img.shields.io/badge/RF-2_/_4_GSPS_I/Q_(MTS)-purple.svg) ![deploy](https://img.shields.io/badge/deploy-vivado_2023.2-FF1010.svg) ![board](https://img.shields.io/badge/board-RFSoC_4x2_(ZCU208_next)-blue.svg) ![host](https://img.shields.io/badge/host-Linux_rdma--core-green.svg)

[English](#en) | [中文](#cn)

　

<span id="en">OFDM modem in the FPGA at 2 and 4 GSPS, over 100G RDMA</span>
===========================

The OFDM transmitter and receiver of [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm) (where the host CPU is the modem) moved into the FPGA, bit-exact with their C models. The host only sends and checks payloads over RoCE v2 (AMD ERNIC, 100G). Now on the RFSoC 4x2 (XCZU48DR), loopback DAC_A → ADC_B (I), DAC_B → ADC_D (Q), multi-tile synchronised; the ZCU208 (the same XCZU48DR) comes next.

* **2 GSPS, on the board**: `ofdm_tx` (8 × 1024-point IFFT) and `ofdm_rx` (8 × 1024-point FFT, widely linear equaliser, pilot phase) run together. FPGA to FPGA, 16-QAM (5.29 Gb/s of payload) for 30 s: BER 2.0 × 10⁻¹⁰; 64-QAM 2.3 × 10⁻⁷, 256-QAM 2.1 × 10⁻⁴, the same as the CPU modem. BRAM 73 %, DSP 20 %, timing met.
* **4 GSPS, in progress**: transmitter, receiver and rf_stream take 16 samples per cycle (16 lanes each); see [4 GSPS](#4-gsps) for where it stands.

The host CPU modem, the video link and the streaming results of the sample path are in [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm), which this work continues. **This repository documents the design and the measurements; the source of the FPGA modem (RTL, bit-exact models, testbenches) is not published.**

　

## How it works

```
host: payloads -> RDMA WRITE -> rf_stream TX ring (32 KB slots) -> ofdm_tx (S lanes x IFFT) -> DAC_A / DAC_B
host: checker  <- RDMA WRITE WITH IMMEDIATE <- rf_stream RX ring (32 KB slots) <- ofdm_rx (S lanes x FFT) <- ADC_B / ADC_D
```

* **rf_stream** is the streaming core of [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm): TX and RX rings in ERNIC's on-chip window, SQ doorbells from the FPGA, a status word read with RDMA READ. Control bit 3 puts the modulator in the TX path, bit 4 the demodulator in the RX path; otherwise the rings carry raw samples, which the host uses to lock to the frame grid and to compute the channel coefficients.
* **S** is the number of samples per 250 MHz cycle: 8 at 2 GSPS, 16 at 4 GSPS. At 4 GSPS the RF data converter runs its streams at 500 MHz with 8 samples, as in [rfsoc4x2_mts](https://github.com/uceeyuf/rfsoc4x2_mts), and `rf_gear` turns two of those beats into one 250 MHz beat of 16 samples.
* **Frame**: N = 1024, CP = 128, 890 active sub-carriers (±16 … ±460) with 56 pilots; 2 training symbols, 26 data symbols and 512 zeros = 32768 samples.

　

## The FPGA transmitter

The transmitter (`ofdm_tx`) is the host's streaming modulator in hardware, 8 samples per 250 MHz cycle (2 GSPS). It is switched on by control bit 3 of rf_stream; the TX ring then holds 32 KB payload slots.

* **Schedule.** A frame is 4096 cycles: symbol *n* starts at cycle *n* × 144. The symbols go in turn to 8 lanes, each with a 1024-point IFFT (xfft v9.1, pipelined, 16-bit input and twiddles, unscaled). A lane takes 1024 cycles per symbol, so 8 lanes carry the 144-cycle symbol rate.
* **Lane.** Each lane feeds its IFFT bin by bin:
  * the bin table gives empty, pilot or data and the data index;
  * the data bits come from the payload buffer (UltraRAM, 72-bit words, so a group of m ≤ 8 bits never spans two words);
  * each axis is Gray-decoded and mapped to a level already scaled by K = gain × 64.
  * A small FIFO in front of the IFFT absorbs the cycles in which the core drops `tready` as a frame starts.
* **Output.** The 27-bit IFFT output becomes sat16(⌊(y + 32) / 64⌋), the host modulator's level and clipping. It goes into ping-pong banks, and is read out 8 wide with the cyclic prefix, 3600 cycles after the symbol's input started (the IFFT latency is 2175 cycles).
* **Keeping time.** A payload not written in time is modulated as zeros and counted; the frame grid stays.
* **Verification.**
  * A C model of the transmitter is bit-exact, with the same tables and AMD's bit-accurate xfft C model. It is within 1 LSB of the floating-point modulator, and the host receiver decodes it without errors.
  * The RTL is simulated (xsim, with the IP) and compared with the model sample by sample: identical for QPSK, 16, 64 and 256-QAM.
* **Resources.** The transmitter takes 34 BRAM36, 8 UltraRAM and 321 DSP (each IFFT: 4 BRAM36, 40 DSP). The design's BRAM goes mostly to rf_stream's sample rings, used when the host modulates or demodulates.

| ![fpga tx](./docs/img/ofdm_16qam_fpga_tx.png) |
| :-------------------------------------------: |
| **Figure1** : 16-QAM modulated in the FPGA: the same spectrum and EVM (−30.6 dB) as from the CPU |

　

## The FPGA receiver

The receiver (`ofdm_rx`) demodulates 8 samples per 250 MHz cycle with a static channel. Control bit 4 of rf_stream switches it on, also during a run: from the next chunk boundary the RX ring holds one 32 KB payload slot per frame instead of samples, and status word +0x1C gives that chunk.

* **Host-assisted start.** `rf_ofdm --fpga-rx 1` first locks to the frame grid in sample mode as usual. It then:
  * sums the FFTs of the training symbols of 4 frames and computes the coefficients (A / B, the back-off phase ramp removed, a 7-tap smoother, the widely linear 2 × 2 inverse per pair (k, −k), int18 with one exponent);
  * computes the rotation gain from the known amplitude (not from the pilots: their coherent sum shrinks with the residual phase across the band);
  * writes the 445 coefficient words (`+0x28_8000`), the gain and the window position in RX time (status +0x24 gives the time of the run's first sample; the time counter never stops, so the grid carries over), and switches.
* **Lanes.** Data symbols 2 … 27 go in turn to 8 lanes. A lane stores the 1024 samples in 128 cycles and feeds its FFT (xfft v9.1, forward, unscaled); Y = sat18(round(Y27 / 16)) goes into one half of a ping-pong buffer.
* **Pair engine.** Per pair (k, −k): x = M y with int18 coefficients (>> 16).
  * The 28 pilot pairs come first. Their sum goes to one shared float32 unit (multiply, add, square root, divide IP, one after another), which returns the unit phase × gain as int18.
  * Then the 417 data pairs: v = x · rot (>> 14), sliced on the grid of odd integers × 2¹⁰, Gray, m bits into the lane's bit buffers by payload index.
* **Packer.** The symbols in order, 8 sub-carriers per cycle, into 512-bit words; each frame padded to 512 words, at most one word every other cycle (the ring's clock crossing drains 200 M words/s).
* **Verification.**
  * A C model follows every step bit-exactly (AMD's xfft C model, float32 in the RTL's order). On board dumps its decisions match the floating-point receiver: identical at 16 and 64-QAM, 3 × 10⁻⁵ of the bits differ at 256-QAM; EVM −30.6 / −30.5 / −30.3 dB.
  * The RTL is simulated on the samples of board dumps and compared with the model, payload bit by payload bit: identical at 16, 64 and 256-QAM.
* **Resources.** The receiver takes 68.5 BRAM36, 516 DSP and 32 k LUT; no UltraRAM.

　

## Results

Host: Core Ultra 7 265K, Mellanox ConnectX-4, Ubuntu 24.04. Board: RFSoC 4x2, SMA loopback DAC_A → ADC_B, DAC_B → ADC_D. 2 GSPS.

| Test | Result |
| :--- | :----- |
| FPGA transmitter and receiver (`--fpga-tx 1 --fpga-rx 1`), 16-QAM 720p, 30 s | 1.83 M frames demodulated in the FPGA, BER 2.0 × 10⁻¹⁰ (32 bits in 1.6 × 10¹¹, at most 7.6 × 10⁻¹⁰ in any second), 14310 of 14338 video frames byte-exact, 0 TX underflows, 0 RX overflows ([log](./docs/results/ofdm_fpga_rx_m4_30s_log.txt)) |
| FPGA transmitter and receiver, 64-QAM 1080p / 256-QAM, 20 s | BER 2.3 × 10⁻⁷ (per-second median 2.1 × 10⁻⁷) / 2.1 × 10⁻⁴, as with the CPU receiver ([64](./docs/results/ofdm_fpga_rx_m6_20s_log.txt), [256](./docs/results/ofdm_fpga_rx_m8_20s_log.txt)) |
| FPGA transmitter, CPU receiver (`--fpga-tx 1`), 16-QAM 720p, 30 s | 29 / 29 s without errors, BER 5.9 × 10⁻¹⁰, 14259 of 14338 video frames byte-exact, 0 TX underflows ([log](./docs/results/ofdm_fpga_tx_m4_30s_log.txt)) |
| FPGA transmitter, 64-QAM 1080p / 256-QAM, 20 s | BER 2.3 × 10⁻⁷ / 1.9 × 10⁻⁴, as with the CPU modulator ([64](./docs/results/ofdm_fpga_tx_m6_20s_log.txt), [256](./docs/results/ofdm_fpga_tx_m8_20s_log.txt)) |
| Timing, resources | all constraints met (WNS +0.049 ns); BRAM 73 %, UltraRAM 85 %, DSP 20 %, LUT 43 % ([report](./docs/results/rdma_ofdm_timing_summary.rpt), [utilisation](./docs/results/rdma_ofdm_utilization.rpt), [by instance](./docs/results/rdma_ofdm_utilization_hierarchical.rpt)) |

　

## 4 GSPS

### The FPGA modem at 4 GSPS (in progress)

* [x] `ofdm_tx`, `ofdm_rx` and `rf_stream` take S = 16 (16 lanes each; two pilot rotation units; the packer takes 16 sub-carriers per cycle). The payload fetch of the transmitter is timed per symbol region, since a frame (2048 cycles) is now shorter than a lane's symbol.
* [x] The RX ring takes two words per clock crossing (two UltraRAM banks): at S = 16 the samples arrive at one word per 250 MHz cycle, more than the 200 MHz side of a one-word crossing drains.
* [x] One-shot sample capture (control bit 6): the link cannot carry 4 GSPS of samples (128 Gb/s), so the capture stops at its first overflow, on a chunk boundary, and the host locks and trains on those frames.
* [x] `rf_gear`: the 500 MHz × 8 streams of the block design to 250 MHz × 16 and back.
* [x] Simulation: `ofdm_tx` and `ofdm_rx` at S = 16 bit-exact with the models (16-QAM, 3 frames); `rf_stream` at S = 8 and 16.
* [ ] Build at 4 GSPS, host (`rf_ofdm` for S = 16, one-shot acquisition), board.
* [ ] ZCU208.

　

## Citation

If this work helps your research, please cite it:

```bibtex
@misc{yu2026zcu208_4gsps_ofdm,
    author = {Yijie Yu},
    title = {{OFDM modem in the FPGA at 2 and 4 GSPS, over 100G RDMA}},
    year = {2026},
    howpublished = {\url{https://github.com/uceeyuf/zcu208_4gsps_ofdm}},
    note = {GitHub repository},
}
```

GitHub also offers the citation under **Cite this repository** (from [CITATION.cff](CITATION.cff)).

　

## Credits

* Streaming core, host modem and link: [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm).
* RDMA side: [rfsoc4x2_ernic](https://github.com/uceeyuf/rfsoc4x2_ernic).
* RF side: [rfsoc4x2_mts](https://github.com/uceeyuf/rfsoc4x2_mts).
* URAM player / capture blocks and the MTS design approach: [Xilinx/RFSoC-MTS](https://github.com/Xilinx/RFSoC-MTS) (MIT).
* Clock register values: PYNQ RFSoC4x2 `LMK04828_500.0` / `LMX2594_500.0`.
* SPI driver: from [RFSoC4x2_clock_LMK_LMX](https://github.com/uceeyuf/RFSoC4x2_clock_LMK_LMX).
* RF data converter driver: Xilinx `rfdc`.
* Ethernet / AXI stream modules: Alex Forencich's [verilog-ethernet](https://github.com/alexforencich/verilog-ethernet) (MIT, submodule pinned at `274831c`).
* Host FFT: [FFTW](https://www.fftw.org) (Frigo and Johnson).

　

## License

* The source of the FPGA modem is not published.
* The streaming core, the host software and the MTS design it builds on are open source (BSD 3-Clause) in [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm).
* Text, figures and logs of this repository: Copyright (c) 2026, Yijie Yu.

　

<span id="cn">FPGA 内的 OFDM 调制解调：2 与 4 GSPS，100G RDMA</span>
===========================

[rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm)（主机 CPU 做调制解调）的 OFDM 发射机和接收机搬进了 FPGA，与各自的 C 模型逐位一致。主机只经 RoCE v2（AMD ERNIC，100G）发送和校验净荷。目前在 RFSoC 4x2（XCZU48DR）上，环回 DAC_A → ADC_B（I）、DAC_B → ADC_D（Q），多 tile 同步；下一步是 ZCU208（同为 XCZU48DR）。

* **2 GSPS，已上板**：`ofdm_tx`（8 个 1024 点 IFFT）和 `ofdm_rx`（8 个 1024 点 FFT、宽线性均衡、导频相位）同时运行。FPGA 到 FPGA，16-QAM（净荷 5.29 Gb/s）30 s BER 2.0 × 10⁻¹⁰；64-QAM 2.3 × 10⁻⁷，256-QAM 2.1 × 10⁻⁴，与 CPU 调制解调相同。BRAM 73 %，DSP 20 %，时序满足。
* **4 GSPS，进行中**：发射机、接收机和 rf_stream 已支持每周期 16 个样本（各 16 条 lane），进度见 [4 GSPS](#4-gsps-1)。

主机 CPU 调制解调、视频链路和样本流的结果在 [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm)，本工作在它的基础上继续。**本仓库只介绍设计和测量结果，FPGA 调制解调的源码（RTL、位精确模型、testbench）不公开。**

　

## 原理

* **rf_stream** 是 [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm) 的流式核心：TX / RX 环位于 ERNIC 片上窗口，FPGA 自己敲 SQ 门铃，状态字用 RDMA READ 读取。控制字 bit 3 把调制器接入 TX 通路，bit 4 把解调器接入 RX 通路；否则环里传原始样本，主机用它们锁定帧网格、计算信道系数。
* **S** 是每个 250 MHz 周期的样本数：2 GSPS 为 8，4 GSPS 为 16。4 GSPS 时 RF 数据转换器的流按 [rfsoc4x2_mts](https://github.com/uceeyuf/rfsoc4x2_mts) 的方式跑在 500 MHz、每拍 8 个样本，由 `rf_gear` 把两拍合成一个 250 MHz、16 个样本的节拍。
* **帧**：N = 1024、CP = 128，890 个有效子载波（±16 … ±460），其中 56 个导频；2 个训练符号、26 个数据符号、512 个零 = 32768 个样本。

　

## FPGA 发射端

发射机（`ofdm_tx`）是主机流式调制器的硬件实现，每个 250 MHz 周期输出 8 个样本（2 GSPS），由 rf_stream 控制字 bit 3 打开，此时 TX 环中放的是 32 KB 的净荷槽。
* **调度**：每帧 4096 个周期，符号 *n* 在第 *n* × 144 个周期开始，轮流交给 8 条 lane。每条 lane 有一个 1024 点 IFFT（xfft v9.1，流水线，16 位输入和旋转因子，不缩放），一个符号用 1024 个周期。
* **lane**：
  * 查 bin 表区分空、导频或数据；
  * 数据 bit 从净荷缓存（UltraRAM，72 位字，m ≤ 8 位的一组不会跨字）取出，每个轴做 Gray 解码后查电平表，电平已乘以 K = 增益 × 64；
  * IFFT 前的小 FIFO 用来吸收内核在帧开始时拉低 `tready` 的那几个周期。
* **输出**：27 位 IFFT 输出取 sat16(⌊(y + 32) / 64⌋)，与主机调制器的电平和削顶一致；写入乒乓缓存后每周期读出 8 个样本并加循环前缀，读出时间比该符号开始输入晚 3600 个周期（IFFT 延迟 2175 个周期）。
* **按时间走**：没按时写到的净荷按全零调制并计数，帧网格不变。
* **验证**：
  * 发射机的 C 模型用相同的表和 AMD 的 xfft 位精确模型逐位建模，与浮点调制器相差不超过 1 LSB，主机接收机解调无误码；
  * RTL 仿真（xsim，带 IP）与模型逐样本比较，QPSK / 16 / 64 / 256-QAM 全部相同。
* **资源**：发射端占 34 个 BRAM36、8 块 UltraRAM、321 个 DSP（每个 IFFT 占 4 个 BRAM36、40 个 DSP）。全设计的 BRAM 主要用于 rf_stream 的样本环（主机调制 / 解调时使用）。

　

## FPGA 接收端

接收机（`ofdm_rx`）按静态信道解调，每个 250 MHz 周期处理 8 个样本。由 rf_stream 控制字 bit 4 打开，运行中也可以切换：从下一个 chunk 边界起，RX 环里放的不再是样本，而是每帧一个 32 KB 的净荷槽；状态字 +0x1C 给出从哪个 chunk 开始。
* **主机辅助启动**：`rf_ofdm --fpga-rx 1` 先照常在样本模式下锁定帧网格，然后：
  * 把 4 帧训练符号的 FFT 相加，计算系数（A / B、去掉回退相位斜坡、7 点平滑、每对 (k, −k) 的宽线性 2 × 2 求逆，int18 共用一个指数）；
  * 由已知幅度算出旋转增益（不用导频幅度：导频相干求和会因频带内残余相位而变小）；
  * 写入 445 个系数字（`+0x28_8000`）、增益和 RX 时间下的窗口位置（状态字 +0x24 给出本次运行第一个样本的时间，时间计数器从不清零，网格可以沿用），然后切换。
* **lane**：数据符号 2 … 27 轮流交给 8 条 lane。每条 lane 用 128 个周期存下 1024 个样本，送入自己的 FFT（xfft v9.1，正变换，不缩放），Y = sat18(round(Y27 / 16)) 写入乒乓缓存的一半。
* **成对处理**：每对 (k, −k) 计算 x = M y（int18 系数，>> 16）。
  * 先处理 28 对导频，求和后交给共用的 float32 单元（乘、加、开方、除 IP 依次使用），得到单位相位 × 增益的 int18；
  * 再处理 417 对数据：v = x · rot（>> 14），在奇数 × 2¹⁰ 的网格上判决、Gray 映射，m 个 bit 按净荷序号写入该 lane 的 bit 缓存。
* **打包**：按符号顺序、每周期 8 个子载波拼成 512 位字，每帧补齐到 512 个字，最多隔一拍输出一个字（RX 环的跨时钟 FIFO 每秒排出 2 亿个字）。
* **验证**：
  * C 模型逐步位精确建模（AMD 的 xfft C 模型，float32 按 RTL 的运算顺序）。在板上采集的数据上，判决与浮点接收机一致：16 / 64-QAM 完全相同，256-QAM 有 3 × 10⁻⁵ 的 bit 不同；EVM −30.6 / −30.5 / −30.3 dB。
  * 用板上采集的样本仿真 RTL，与模型逐 bit 比较净荷：16 / 64 / 256-QAM 全部相同。
* **资源**：接收端占 68.5 个 BRAM36、516 个 DSP、3.2 万个 LUT，不用 UltraRAM。

　

## 结果

2 GSPS，RFSoC 4x2，SMA 环回。

| 测试 | 结果 |
| :--- | :--- |
| FPGA 发射 + FPGA 接收（`--fpga-tx 1 --fpga-rx 1`），16-QAM 720p，30 s | FPGA 解调 183 万帧，BER 2.0 × 10⁻¹⁰（1.6 × 10¹¹ bit 中 32 个错，任一秒不超过 7.6 × 10⁻¹⁰），14338 帧视频中 14310 帧逐字节正确，0 TX underflow，0 RX overflow |
| FPGA 发射 + 接收，64-QAM 1080p / 256-QAM，20 s | BER 2.3 × 10⁻⁷（每秒中位数 2.1 × 10⁻⁷）/ 2.1 × 10⁻⁴，与 CPU 接收相同 |
| FPGA 发射、CPU 接收（`--fpga-tx 1`），16-QAM 720p，30 s | 29/29 秒无误码，BER 5.9 × 10⁻¹⁰，14338 帧中 14259 帧逐字节正确，0 TX underflow |
| FPGA 发射，64-QAM 1080p / 256-QAM，20 s | BER 2.3 × 10⁻⁷ / 1.9 × 10⁻⁴，与 CPU 调制相同 |
| 时序、资源 | 全部满足（WNS +0.049 ns）；BRAM 73 %，UltraRAM 85 %，DSP 20 %，LUT 43 % |

　

## 4 GSPS

### 4 GSPS 的 FPGA 调制解调（进行中）

* [x] `ofdm_tx`、`ofdm_rx`、`rf_stream` 支持 S = 16（各 16 条 lane；两个导频旋转单元；打包每周期 16 个子载波）。帧（2048 个周期）比一条 lane 处理一个符号的时间还短，所以发射端取净荷按符号区域安排时间。
* [x] RX 环每次跨时钟传两个字（两个 UltraRAM bank）：S = 16 时每个 250 MHz 周期来一个字，单字跨时钟的 200 MHz 一侧排不完。
* [x] 一次性样本抓取（控制字 bit 6）：链路传不了 4 GSPS 的样本（128 Gb/s），抓取在第一次溢出时停下（正好在 chunk 边界），主机用抓到的帧锁定和训练。
* [x] `rf_gear`：block design 的 500 MHz × 8 与 250 MHz × 16 之间互转。
* [x] 仿真：S = 16 的 `ofdm_tx` 和 `ofdm_rx` 与模型逐位一致（16-QAM，3 帧）；`rf_stream` 在 S = 8 和 16 下通过。
* [ ] 4 GSPS 编译，主机端（`rf_ofdm` 支持 S = 16、一次性抓取锁定），上板。
* [ ] ZCU208。

　

## 引用

如果本工作对你的研究有帮助，请引用：

```bibtex
@misc{yu2026zcu208_4gsps_ofdm,
    author = {Yijie Yu},
    title = {{OFDM modem in the FPGA at 2 and 4 GSPS, over 100G RDMA}},
    year = {2026},
    howpublished = {\url{https://github.com/uceeyuf/zcu208_4gsps_ofdm}},
    note = {GitHub repository},
}
```

GitHub 仓库页面的 **Cite this repository** 也提供引用（来自 [CITATION.cff](CITATION.cff)）。

　

## 致谢

* 流式核心、主机调制解调和链路：[rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm)。
* RDMA 部分：[rfsoc4x2_ernic](https://github.com/uceeyuf/rfsoc4x2_ernic)。
* 射频部分：[rfsoc4x2_mts](https://github.com/uceeyuf/rfsoc4x2_mts)。
* URAM 播放 / 采集模块和 MTS 设计思路：[Xilinx/RFSoC-MTS](https://github.com/Xilinx/RFSoC-MTS)（MIT）。
* 时钟寄存器值：PYNQ RFSoC4x2 `LMK04828_500.0` / `LMX2594_500.0`。
* SPI 驱动：来自 [RFSoC4x2_clock_LMK_LMX](https://github.com/uceeyuf/RFSoC4x2_clock_LMK_LMX)。
* RF 数据转换器驱动：Xilinx `rfdc`。
* 以太网 / AXI stream 模块：Alex Forencich 的 [verilog-ethernet](https://github.com/alexforencich/verilog-ethernet)（MIT，子模块 `274831c`）。
* 主机端 FFT：[FFTW](https://www.fftw.org)（Frigo 与 Johnson）。

　

## 许可证

* FPGA 调制解调的源码不公开。
* 它所基于的流式核心、主机软件和 MTS 设计在 [rfsoc4x2_rdma_ofdm](https://github.com/uceeyuf/rfsoc4x2_rdma_ofdm) 开源（BSD 3-Clause）。
* 本仓库的文字、图和日志：版权所有 (c) 2026 Yijie Yu。
