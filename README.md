# 🚀 TinyML Motor Anomaly Detection using STM32

[![Hardware](https://img.shields.io/badge/Hardware-STM32L476RG-blue.svg)](https://www.st.com/en/microcontrollers-microprocessors/stm32l476rg.html)
![Framework](https://img.shields.io/badge/Framework-STM32Cube_HAL-green.svg)
![AI](https://img.shields.io/badge/AI-NanoEdge_AI_Studio-orange.svg)
![Dashboard](https://img.shields.io/badge/Dashboard-Python_3.x-yellow.svg)

An industrial-grade Edge AI (TinyML) reference implementation for Predictive Maintenance and Anomaly Detection in electric motors on **STM32L476RG** (80 MHz ARM Cortex-M4 with FPU). Using hardware-timed **ADC + Ping-Pong DMA**, **NanoEdge AI Studio**, and **asynchronous UART DMA telemetry**, the firmware detects injected high-frequency harmonics and broadband noise in real time with an on-device inference latency of **~0.5 ms** and under **3 KB** of RAM footprint.

---

## 📖 Project Overview

Deploying machine learning on resource-constrained microcontrollers requires more than a trained model—it demands deterministic timing, race-free DMA handling, and awareness of hardware-induced **Domain Shift**. This project pairs **NanoEdge AI Studio** with modular, object-oriented Embedded C (`Handle Typedef` encapsulation, zero blocking loops during inference, and atomic state transitions).

The firmware is organized into two compile-time selectable phases controlled in `app_config.h`:
1. **Data Logging Phase (`PROJECT_PHASE_DATALOGGING`):** Software-polled data acquisition (`ADC2` + blocking UART `printf`) used to generate baseline CSV datasets—intentionally exposing sampling jitter and acquisition blind spots.
2. **Inference Phase (`PROJECT_PHASE_INFERENCE`):** Hardware-triggered, non-blocking DMA acquisition (`TIM3` + calibrated `ADC1` + Ping-Pong DMA) with real-time anomaly detection, execution-time profiling (`TIM7`), and binary SCADA telemetry (`USART2` DMA).

---

<img width="1436" height="699" alt="Real-time Python SCADA dashboard showing healthy 50 Hz motor current signal and high similarity score" src="https://github.com/user-attachments/assets/0ab2f032-60a9-4096-b42d-15fbc4474eb5" />

<img width="1427" height="711" alt="Real-time Python SCADA dashboard showing injected 350 Hz harmonic anomaly, FFT peak, and triggered fault alert" src="https://github.com/user-attachments/assets/cd228cb9-cc80-4873-ab1b-7216dc9adafe" />

---

## 🔌 Hardware Setup & Peripheral Mapping (STM32L476RG Nucleo-64)

Because the internal on-chip DAC-to-ADC connection is disabled (`DAC_CHIPCONNECT_DISABLE`), a simple external jumper wire (analog loopback) routes the simulated motor signal to the active ADC channel:

| Function | Peripheral | Pin | Configuration Details |
| :--- | :--- | :--- | :--- |
| **Motor Signal Output** | `DAC1` (CH1) + DMA | `PA4` (`MOTOR_SIGNAL_Pin`) | Driven by `TIM6` TRGO at **5 kHz** (100 samples/cycle = **50 Hz**) |
| **Inference Input** | `ADC1` (CH1) + DMA | `PC0` (`SAMPLING_CH_Pin`) | Triggered by `TIM3` TRGO at **2 kHz**; hardware calibrated |
| **Data Logging Input** | `ADC2` (CH2) | `PC1` (`DATALOG_CH_Pin`) | Software-triggered polling (`~2 kHz` via `TIM7` delay) |
| **Fault Toggle Button** | `EXTI15_10` | `PC13` (`B1_Pin`) | Falling-edge interrupt toggles Normal ↔ Anomaly state |
| **Fault Alarm LED** | `GPIO` Output | `PA5` (`LD2_Pin`) | `HIGH` on `AICORE_MOTOR_ANOMALY`, `LOW` on `AICORE_MOTOR_NORMAL` |
| **Telemetry / Logging** | `USART2` (TX DMA) | `PA2` / `PA3` | `115200` baud, 8N1 (ST-Link Virtual COM Port) |
| **Microsecond Timer** | `TIM7` | Internal | `Prescaler = 80 - 1` (**1 µs/tick** at 80 MHz); delay in Phase 1, latency profiler in Phase 3 |

> **Wiring Note:** Connect `PA4` to `PC1` during the Data Logging phase, and connect `PA4` to `PC0` during the Inference phase (or jumper `PA4` to both `PC0` and `PC1` simultaneously).

---

## 🛠️ Phase 1: Hardware Simulation & Data Logging

### 1. Deterministic Motor Signal Simulation (`motor_sim.c`)
To safely and reproducibly test fault scenarios without damaging a physical motor, phase current is synthesized via `DAC1` in circular DMA mode (`12-bit right-aligned`, `100` samples/cycle updated at `5 kHz` by `TIM6`):
* **Normal State (`MOTORSIM_STATE_NORMAL`):** Pure **50 Hz** sine wave centered at DC offset `2048` with amplitude `1500`.
* **Anomaly State (`MOTORSIM_STATE_ANOMALY`):** The 50 Hz fundamental superimposed with a **350 Hz 7th harmonic** (amplitude `300`) plus **broadband random jitter** (`[-100, +99]` LSB), clamped to `[0, 4095]`, emulating bearing defects and mechanical vibration.
* Pressing the onboard user button (`B1_Pin`, `PC13`) triggers `HAL_GPIO_EXTI_Callback`, safely stopping and restarting DAC DMA to switch waveform buffers in real time.

### 2. Blocking Data Logging (`data_logger_blocking.c`)
Set `#define CURRENT_PROJECT_PHASE PROJECT_PHASE_DATALOGGING` in `app_config.h` to collect raw 256-sample windows over UART (e.g., via PuTTY) for NanoEdge AI Studio:
* Uses software-polled `ADC2` (without hardware calibration) paced by a `TIM7` busy-wait loop (`500 µs` target period minus `2 µs` overhead) and blocking `printf` transmission.
* **The Domain Shift Setup:** Software polling jitter, UART transmission blind spots, and the absence of hardware ADC calibration introduce subtle statistical differences compared to the DMA-based inference pipeline.

---

## 🧠 Phase 2: NanoEdge AI Studio & Model Generation

The logged Normal and Anomaly CSV files were imported into **NanoEdge AI Studio**, which benchmarked candidate DSP preprocessing pipelines and anomaly-detection models:
* **Selected Algorithm:** A model from the **ZSM** family.
* **Validation Result:** `100%` separation accuracy on the DAC-simulated 50 Hz vs. 350 Hz + broadband noise benchmark dataset.
* **Output:** A pre-compiled static library (`libneai.a`) and header (`NanoEdgeAI.h`) integrated into STM32CubeIDE.

---

## 🧩 Integrating the NanoEdge AI Studio ZIP File

1. **Extract and Copy Files:**
   * Copy `NanoEdgeAI.h` into `Core/Inc`.
   * Copy the Cortex-M4 static library (`libneai.a`) into a new folder `Core/Lib`.
2. **Link the Library in STM32CubeIDE:**
   * Open **Project Properties** -> **C/C++ Build** -> **Settings** -> **Tool Settings** -> **MCU GCC Linker** -> **Libraries**.
   * Under **Libraries (-l)**, add: `neai` *(omit the `lib` prefix and `.a` extension)*.
   * Under **Library search path (-L)**, add: `"${workspace_loc:/${ProjName}/Core/Lib}"`
3. **Enable Hardware FPU:**
   * Verify Single-Precision FPU is enabled in STM32CubeMX (**System Core -> CORTEX_M4**) and linker/compiler flags (`-mfpu=fpv4-sp-d16 -mfloat-abi=hard`).

---

## ⚡ Phase 3: Non-Blocking Inference & Domain Shift Mitigation

Switch `#define CURRENT_PROJECT_PHASE PROJECT_PHASE_INFERENCE` in `app_config.h` to build the production inference pipeline:

### 1. Hardware-Timed Ping-Pong DMA (`ai_core.c`)
* **Deterministic Sampling:** `TIM3` (`Prescaler = 80 - 1`, `Period = 500 - 1`) generates an exact **2 kHz** `TRGO` trigger for hardware-calibrated `ADC1`.
* **Double Buffering:** DMA transfers 12-bit samples continuously into a **512-sample Ping-Pong buffer** (`AICORE_DMA_BUFFER_SIZE = 2 × 256`). Each half-buffer holds **256 samples** (**128 ms** per inference window).
* **Race-Free State Machine & Overrun Detection:** `HAL_ADC_ConvHalfCpltCallback` and `HAL_ADC_ConvCpltCallback` route events to `AICore_HalfTransfer_ISR` / `AICore_FullTransfer_ISR`. State transitions inside `AICore_Process` are guarded by atomic IRQ-disabled sections, and any missed processing deadline is flagged as `AICORE_DMA_OVERRUN` (`overrun_errors++`).
* **Execution Profiling:** `TIM7` (1 µs resolution) measures exact `neai_anomalydetection_learn` / `neai_anomalydetection_detect` execution latency (`inference_time_us`).

### 2. Solving Domain Shift (Static vs. Continual Learning)
Moving from uncalibrated, software-polled `ADC2` (Phase 1) to calibrated, jitter-free `TIM3 + ADC1 + DMA` (Phase 3) causes a measurable **Domain Shift**. You can toggle between two modes in `app_config.h`:

```c
/* Default is AI_LEARNING_MODE_STATIC to demonstrate Domain Shift;
 * switch to AI_LEARNING_MODE_CONTINUAL for on-device adaptation. */
#define CURRENT_AI_LEARNING_MODE    AI_LEARNING_MODE_CONTINUAL
```

* **`AI_LEARNING_MODE_STATIC` (Default in `app_config.h`):** Runs inference directly on the pre-trained Studio model. Similarity drops to **~64%** even on a healthy motor, clearly exposing the hardware calibration and timing shift.
* **`AI_LEARNING_MODE_CONTINUAL`:** Executes on-device adaptation (`neai_anomalydetection_learn`) during the first **30 inference windows** after boot (~3.84 seconds at 2 kHz / 256 samples) to adapt the anomaly-detection model to the live DMA/ADC1 baseline, then switches to `neai_anomalydetection_detect`, restoring the healthy similarity score to **~100%**.

---

## 🛡️ Operating Constraints & Safety Assumptions

In real industrial deployments—and when feeding edge diagnostics into higher-level maintenance agents—several operating constraints of this architecture must be explicitly managed:
* **Healthy-Start Assumption in Continual Learning:** During the initial 30-window learning phase (`learning_count < 30`), `AICore_Process` intentionally suppresses fault detection and reports `AICORE_MOTOR_NORMAL` (`similarity = 100%`). If a motor starts up with a pre-existing fault, the model will learn that fault as normal behavior; production deployments require human verification or an external health interlock before enabling on-device learning.
* **Telemetry Backpressure Without Blocking Inference:** `main.c` initiates `HAL_UART_Transmit_DMA` only when `huart2.gState == HAL_UART_STATE_READY`, ensuring slow or disconnected serial links never block inference or corrupt the active TX buffer.
* **Overrun Visibility:** If CPU load exceeds the 128 ms window budget, frames are safely dropped and counted in `hAI.overrun_errors` rather than silently corrupting the active half-buffer.

---

## 📦 Custom Library Usage (`ai_core`)

All AI states, DMA buffers, profiling timers, and counters are encapsulated in `AICore_Handle_t` (`ai_core.h`), supporting multi-instance usage without global state pollution.

**Step 1: Calibrate ADC, Start Profiler Timer, and Initialize AI Core (90% Similarity Threshold)**

```c
#include "ai_core.h"

static AICore_Handle_t hAI;

HAL_TIM_Base_Start(&htim7);
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);
AICore_Init(&hAI, &hadc1, &htim7, 90);
```

**Step 2: Start Background DMA Acquisition & 2 kHz Trigger**

```c
AICore_Start_DMA(&hAI);
HAL_TIM_Base_Start(&htim3);
```

**Step 3: Non-Blocking Processing in the Main Loop**

```c
while (1) {
    if (AICore_Process(&hAI) == 1) {
        if (hAI.health_status == AICORE_MOTOR_ANOMALY) {
            HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET);   /* Trigger Alarm */
        } else {
            HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET); /* Normal State */
        }
    }
}
```

---

## 💻 Python SCADA Dashboard (Asynchronous Binary Telemetry)

After each 256-sample window is processed, the MCU transmits a packed **1034-byte** binary frame (`TelemetryPacket_t`, `#pragma pack(push, 1)`) via **USART2 DMA** at `115200` baud (~90 ms wire time per 128 ms window):

| Offset | Field | Type | Size (Bytes) | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0` | `header[2]` | `uint8_t[2]` | `2` | Sync bytes: `0xAA, 0x55` |
| `2` | `health_status` | `uint8_t` | `1` | `0`: Normal, `1`: Anomaly |
| `3` | `similarity_score` | `uint8_t` | `1` | `0–100%` similarity score |
| `4` | `inference_time_us` | `uint32_t` | `4` | Measured execution time (µs via `TIM7`) |
| `8` | `adc_data[256]` | `float[256]` | `1024` | 256-sample floating-point ADC window |
| `1032` | `footer[2]` | `uint8_t[2]` | `2` | Sync bytes: `0x55, 0xAA` |

The included `dashboard.py` script (`matplotlib` + `pyserial`) renders:
* **Time-Domain Signal:** Live 256-sample ADC DMA waveform.
* **Frequency-Domain (FFT):** Real-time spectrum highlighting the 350 Hz 7th-harmonic peak.
* **Similarity Gauge & Status Alert:** Live normal/anomaly indication.
* **Latency Monitor:** Microsecond inference profiling readout (~0.5 ms).

### Running the Dashboard

```bash
pip install pyserial matplotlib numpy
python dashboard.py
```

*(Update the `SERIAL_PORT` variable inside `dashboard.py` to match your STM32 ST-Link COM port.)*

---

## 👨‍💻 Author & Contact

**Mohammad Hosseini**
* Embedded Systems & Power Electronics Engineer
* Instructor at FaraDars & Founder of Esfahan Drive

For inquiries, collaborations, or questions regarding this project:
* 📧 **Email:** about.mim.hosseini@gmail.com
* 🌐 **Website:** [esfdrive.ir](https://esfdrive.ir)

---

*If you found this project helpful, please consider giving it a ⭐!*
