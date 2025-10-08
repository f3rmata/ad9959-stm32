# AD9959_STM32

基于 STM32 HAL 的 AD9959（4 通道 DDS）最小驱动，支持单音输出、相位/幅度设置与线性扫频（频率/幅度/相位）。本文档提供硬件连接说明、CubeMX 配置要点、API 讲解与示例代码，帮助你快速集成与使用。

参考实现与思路部分借鉴自 https://github.com/cjheath/AD9959

---

## 目录
- 项目简介
- 硬件连接
- STM32/CubeMX 配置
- 快速开始
- API 速览与说明
- 频率校准方法
- 设计注意事项与限制
- 常见问题
- 许可证

---

## 项目简介

文件结构:
- AD9959.h / AD9959.c：核心驱动
- README.md：使用说明（本文档）
- LICENSE：开源许可证

支持功能:
- 初始化与硬复位
- 配置内部 PLL 倍频与系统时钟
- 设置通道频率（使用 32 位 CFTW）
- 设置相位（14 位）
- 设置幅度（10 位，支持幅度乘法器）
- 频率/相位/幅度扫（线性扫），含步进速率配置

SPI 侧关键特性（驱动假设）:
- nSS 由软件控制（CS 引脚手动拉低/拉高）
- MSB first
- CPOL = 0，CPHA = 1（第 1 个边沿采样）
- 数据位宽 8 bit

注：更高带宽可用 QSPI，但当前实现基于标准 SPI。

---

## 硬件连接

AD9959 | 连接
---|---
SCK | STM32-SCK
CS | GPIO-CS（片选）
UPDT | GPIO-UPDT（I/O UPDATE）
MRST | GPIO-MRST（主复位）
PDC | 必须接地
SD0 | STM32-MOSI
SD1 | 悬空
SD2 | 悬空
SD3 | 必须接地
P0 | 用到扫频/调制时接 GPIO/定时器
P1 | 用到扫频/调制时接 GPIO/定时器
P2 | 用到扫频/调制时接 GPIO/定时器
P3 | 用到扫频/调制时接 GPIO/定时器

其它硬件要点:
- 参考晶振/时钟输入请按板卡设计配置至 AD9959。驱动通过 AD9959_SetClock 配置内部 PLL 倍频。
- SYNC_CLK 默认被禁用（降低干扰），如需同步跨片时钟，请按需修改 FR1 配置。

---

## STM32/CubeMX 配置

- SPI:
  - 数据位宽：8 bits
  - CPOL: Low, CPHA: 1 Edge
  - NSS：硬件管理关闭（由软件控制 CS）
  - 波特率：尽可能高（受硬件与布线限制），AD9959 SCLK 可高达 200 MHz，实际 MCU 端通常远低于该值
- GPIO:
  - CS、UPDT、MRST：推挽输出
  - Profile 引脚（P0~P3）：如需触发扫频/调制，可接 GPIO 或定时器 PWM
- HAL 工程：确保已生成对应的 SPI 句柄（例如 hspi1）

Tips:
- 在 CubeMX 中将 SPI 数据大小设为 8 bits（必须）。

---

## 快速开始

下面示例展示最小使用流程：初始化、配置时钟、设置频率。

```c
#include "AD9959.h"

AD9959_Handler u_ad9959;

// 假设你已经通过 CubeMX 生成了 hspi1，并将下列 GPIO 定义为输出
void AD9959_UserInit(void)
{
    // 参数说明：
    // crystal_clock: 外部参考时钟频率（Hz），如 25MHz
    // calibration: 频率校准（ppb），初始用 0，后续可用测量结果更新
    AD9959_Init(&u_ad9959,
                &hspi1,
                ADI_CS_GPIO_Port, ADI_CS_Pin,
                ADI_RST_GPIO_Port, ADI_RST_Pin,
                ADI_UPDT_GPIO_Port, ADI_UPDT_Pin,
                25000000,  // 外部参考 25 MHz
                0);        // 暂不校准

    // 配置内部 PLL 倍频（4..20），例如 16 倍 => 系统时钟约 400 MHz（取决于外部参考与校准）
    AD9959_SetClock(&u_ad9959, 16, 0);

    // 设置所有通道输出 10 kHz（注意：芯片限制，建议 >= 100 kHz）
    AD9959_SetChannelFrequency(&u_ad9959, Channel_All, 100000);
}
```

扫频（示例）：
```c
// 设定目标频率为 50 MHz，follow=0 表示 No-Dwell（一次跳转/步进不驻留）
AD9959_SweepFrequency(&u_ad9959, Channel_All, 50000000, 0);

// 设置上升/下降步进量与速率（越大越慢；单位由系统时钟决定）
AD9959_SweepRates(&u_ad9959, Channel_All,
                  5000000, 125,    // 上升：每步 ΔCFTW=5e6，步进计数=125
                  5000000, 125);   // 下降：同上
```

---

## API 速览与说明

以下函数在 AD9959.h 中声明，在 AD9959.c 中实现。

- AD9959_Init(AD9959_Handler* device, SPI_HandleTypeDef* hspi, ...,
  uint32_t crystal_clock, uint32_t calibration)
  - 作用：绑定 SPI 与控制引脚，保存外部参考时钟与校准值，并进行一次芯片复位与默认配置。
  - 注意：根据 SPI Direction 自动选择 2/3 线模式。初始化后将进行寄存器重置与一次 I/O UPDATE。

- AD9959_Reset(AD9959_Handler* device, AD9959_CFR_Bits device_cfr)
  - 作用：硬件复位 + 关键寄存器初始化。
  - device_cfr=0 时使用默认组合：DACFullScale | MatchPipeDelay | OutputSineWave | AutoclearPhase | AutoclearSweep。

- AD9959_SetClock(AD9959_Handler* device, uint8_t mult, int32_t calibration)
  - 作用：配置内部 PLL 倍频（mult=4..20，<4 或 >20 则关闭倍频），设置校准值（ppb）。
  - core_clock = crystal_clock × mult × (1 + calibration/1e9)。
  - 根据核心频率自动选择 VCO 增益，并写入 FR1。
  - 提示：初始校准设为 0，后续根据测量结果更新（见“频率校准方法”）。

- AD9959_SetChannels(AD9959_Handler* device, AD9959_Channel chan)
  - 作用：选择后续写入生效的通道（CSR）。若与前次选择不同才写 CSR，减少总线访问。
  - 提示：对特定通道写寄存器会“使能”该通道输出。

- AD9959_GenerateFrequencyDelta(AD9959_Handler* device, uint32_t freq)
  - 作用：将目标频率（Hz）转换为 32 位 CFTW（Channel Frequency Tuning Word）。
  - 可配合 AD9959_SetChannelDelta 使用，减少重复换算开销。

- AD9959_SetChannelFrequency(AD9959_Handler* device, AD9959_Channel chan, uint32_t freq)
  - 作用：直接以 Hz 设置通道频率（内部调用 GenerateFrequencyDelta）。
  - 限制：频率建议 ≥ 100 kHz，否则波形可能失真（芯片限制）。

- AD9959_SetChannelDelta(AD9959_Handler* device, AD9959_Channel chan, uint32_t delta)
  - 作用：以 CFTW 直接配置频率。写 CFTW 后触发 I/O UPDATE。

- AD9959_SetPhase(AD9959_Handler* device, AD9959_Channel chan, uint16_t phase)
  - 作用：设置相位（14 位，0..16383）。写 CPOW 后 I/O UPDATE。

- AD9959_SetAmplitude(AD9959_Handler* device, AD9959_Channel chan, uint16_t amplitude)
  - 作用：设置幅度（10 位，0..1024），小于 1024 时自动启用幅度乘法器（MultiplierEnable）。
  - 写 ACR 后 I/O UPDATE。

- 扫频相关
  - AD9959_SweepFrequency(..., uint32_t freq, uint8_t follow)
  - AD9959_SweepDelta(..., uint32_t delta, uint8_t follow)
  - AD9959_SweepRates(..., uint32_t increment, uint8_t up_rate, uint32_t decrement, uint8_t down_rate)
  - AD9959_SweepAmplitude(..., uint16_t amplitude, uint8_t follow)
  - AD9959_SweepPhase(..., uint16_t phase, uint8_t follow)
  - 说明：
    - SweepFrequency/Delta 会：
      - 选择通道 -> 将 CFR 设置为 FrequencyModulation | SweepEnable | DACFullScale | MatchPipeDelay | (follow ? 0 : SweepNoDwell)
      - 将目标 CFTW 写到 CW1
      - 触发 I/O UPDATE
    - SweepAmplitude/Phase 类似，只是目标寄存器 MSB 对齐写入幅度/相位到 CW1。
    - up_rate/down_rate 为 8 位步进计数器参数，值越大，步进越慢；具体步进时间与系统时钟相关。
    - follow 参数：1 表示允许驻留/跟随（不置 SweepNoDwell），0 表示 No-Dwell。

---

## 频率校准方法

驱动支持以 “十亿分之一(ppb)” 为单位的频率校准：
1. 将 calibration 设为 0，配置某个已知目标频率（例如 10 MHz）。
2. 用频率计数器或示波器测量实际输出 f_meas。
3. 计算误差 ppb = round((f_meas - f_set) / f_set × 1e9)。例如测到 10,000,043.2 Hz，则 ppb ≈ +4320。
4. 调用 AD9959_SetClock(&dev, mult, ppb) 重新配置时钟。
5. 后续频率换算均基于校准后的 core_clock 进行。

---

## 设计注意事项与限制

- SPI 时序：CPOL=0、CPHA=1，8 位数据宽度。
- I/O UPDATE：除选择通道（CSR）外，大多数寄存器写入需要一次 I/O UPDATE 才会生效；驱动会在写后自动触发。
- 最低频率：建议 ≥ 100 kHz，过低频率可能出现波形异常（芯片限制）。
- VCO 设置：当核心时钟较高（>~200 MHz）时会自动选择高 VCO 增益以加快锁定；中间区域以数据手册为准。
- 多通道读写：AD9959 的部分寄存器按通道复用（通过 CSR 选择），写入时会对已选通道同时生效。
- Profile 引脚：在扫频/调制方案中，可利用 P0~P3 作为外部触发/选择；具体时序与电平定义请参考数据手册。
- SYNC/多片同步：当前默认关闭 SYNC_CLK 输出，如需跨片同步或对齐相位，请结合 FR1/FR2 的同步配置使用。
- 电源与地：模拟/数字地与电源去耦按参考设计执行，保证高频稳定。

---

## 常见问题

- 没有波形输出或幅度很小
  - 确认 MRST、UPDT 连接正确且执行了初始化。
  - 确认 PDC 接地、SD3 接地。
  - 确认 CFTW 对应频率 ≥ 100 kHz。
  - 检查 ACR 配置与幅度是否限制在 10 位范围内。

- 频率不准确
  - 先将 calibration 置 0，测量后根据 ppb 进行校准。
  - 确认外部参考频率设置正确，PLL 倍频 mult 合理（4..20 或关闭）。

- 扫频速度与预期不符
  - 调整 up_rate/down_rate（值越大越慢）。
  - 检查系统时钟 core_clock 是否与预期一致（受外部参考与校准影响）。

---

## 许可证

见本仓库 LICENSE 文件。
