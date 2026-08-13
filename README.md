# STM32-TinyML-Motor-Anomaly-Detection
# 🚀 TinyML Motor Anomaly Detection using STM32

[![Hardware](https://img.shields.io/badge/Hardware-STM32L476RG-blue.svg)](https://www.st.com/)
[![Framework](https://img.shields.io/badge/Framework-STM32Cube_HAL-green.svg)]()
[![AI](https://img.shields.io/badge/AI-NanoEdge_AI_Studio-orange.svg)]()
[![Python](https://img.shields.io/badge/Dashboard-Python_3.x-yellow.svg)]()

A complete, industrial-grade Edge AI (TinyML) project for Predictive Maintenance and Anomaly Detection in electric motors. This project leverages the ARM Cortex-M4 FPU, hardware DMA, and NanoEdge AI Studio to detect motor bearing failures and harmonic anomalies in real-time, with an AI inference time of just **~0.5 milliseconds** and less than 3KB of RAM footprint.

---

## 📖 Project Overview
Implementing Deep Learning on resource-constrained microcontrollers usually requires deep expertise in Data Science. This project demonstrates a streamlined, highly optimized approach using **NanoEdge AI Studio** (an AutoML tool) combined with advanced Embedded C architectures (Event-Driven, Non-Blocking, Object-Oriented C concepts). 

The system detects injected high-frequency harmonics (representing physical motor faults) in a simulated motor current signal, adapting to hardware domain shifts dynamically.

---
<img width="1436" height="699" alt="image" src="https://github.com/user-attachments/assets/0ab2f032-60a9-4096-b42d-15fbc4474eb5" />

<img width="1427" height="711" alt="image" src="https://github.com/user-attachments/assets/cd228cb9-cc80-4873-ab1b-7216dc9adafe" />


## 🛠️ Phase 1: Hardware Simulation & Data Logging

### 1. Motor Signal Simulation (DAC)
To safely and consistently train the AI without physically damaging an industrial motor, the motor's phase current is simulated using the MCU's **DAC (Digital-to-Analog Converter)** triggered by TIM6:
*   **Normal State:** A pure 50Hz sine wave.
*   **Anomaly State:** A 50Hz sine wave combined with a 350Hz 7th harmonic, simulating bearing faults or mechanical vibrations.
*   A physical push-button (with external interrupt) toggles between these two states.

### 2. Data Logging Phase (Preparation for AI Studio)
NanoEdge AI Studio requires raw data (CSV files) to benchmark and find the best algorithm. 
During the initial project phase, the ADC reads the simulated signal, and the MCU sends this data to a PC via UART (using PuTTY). 
*   **The Jitter Factor:** This phase uses a software-based, **blocking** approach. Consequently, the sampling time is not perfectly precise, introducing a software Jitter into the logged dataset. This imperfection is sufficient for initial AI training but creates a "Domain Shift" challenge later.

---

## 🧠 Phase 2: NanoEdge AI Studio & Model Generation
The logged CSV files were imported into NanoEdge AI Studio. The software benchmarked thousands of DSP and statistical combinations.
*   **Selected Algorithm:** A model from the **ZSM (Zero-Shot Modeling)** family was chosen.
*   **Accuracy:** 100% accuracy on the provided dataset.
*   **Output:** A highly optimized static library (libneai.a) ready to be integrated into STM32CubeIDE.

---

## 🧩 Integrating the NanoEdge AI Studio ZIP File
Once NanoEdge AI Studio compiles your model, it provides a ZIP file containing the necessary C headers and pre-compiled libraries. Follow these steps to link it in STM32CubeIDE:

**Step 1: Extract and Copy Files**
*   Extract the generated ZIP file.
*   Copy `NanoEdgeAI.h` into your project's `Core/Inc` folder.
*   Copy the appropriate static library for your ARM core (e.g., `libneai.a` for Cortex-M4) into a new folder named `Core/Lib` in your project.

**Step 2: Link the Library in STM32CubeIDE**
*   Right-click your project in the Project Explorer and select **Properties**.
*   Navigate to **C/C++ Build** -> **Settings** -> **Tool Settings** tab.
*   Go to **MCU GCC Linker** -> **Libraries**.
*   In the **Libraries (-l)** section, click the Add icon and type: `neai` *(Note: strictly drop the 'lib' prefix and '.a' extension)*.
*   In the **Library search path (-L)** section, click the Add icon and type the path to your Lib folder: `"${workspace_loc:/${ProjName}/Core/Lib}"`

**Step 3: Enable the FPU**
*   Since the AI library uses float operations, ensure the Hardware FPU is enabled in STM32CubeMX (System Core -> CORTEX_M4 -> Floating Point Unit: Single Precision).

---

## ⚡ Phase 3: Industrial Inference & Advanced Architecture

Once the AI library is imported, the project transitions to the Inference phase. The architecture is completely revamped to meet industrial standards: no blocking delays, no global variable chaos, and strict hardware-level timing.

### 1. High-Speed Non-Blocking Sampling (ADC + DMA)
*   **Precise Triggering:** TIM3 is configured to generate a strict **2 kHz** TRGO event.
*   **Background Transfer:** ADC1 performs the conversion, and the **DMA Controller** directly transfers the data into a 256-sample ping-pong buffer in RAM without waking up the CPU.
*   **FPU Optimization:** Once the DMA triggers a Half-Transfer or Full-Transfer interrupt, the CPU converts the 12-bit raw integers into 32-bit floats using the Cortex-M4 Floating-Point Unit (FPU) before feeding them to the AI model.

### 2. The Domain Shift Challenge (Static vs. Continual Learning)
A major engineering challenge arises in the Inference phase: the AI model (trained on jittery software data) gets confused by the extremely precise, jitter-free DMA signal and the new ADC hardware calibration offset. This is known as **Domain Shift**. 
To demonstrate and solve this, the project features a macro switch in app_config.h:

    #define CURRENT_AI_LEARNING_MODE    AI_LEARNING_MODE_CONTINUAL

*   **Static Mode:** Uses the raw pre-trained weights. The similarity score drops to ~64% even when the motor is healthy, perfectly demonstrating the Domain Shift issue.
*   **Continual Learning:** Solves the issue using On-Device Fine-Tuning. During the first 30 inference cycles after boot, the MCU calls the learning function to adapt the neural network to the exact physical conditions of the DMA and ADC offsets. After 30 cycles, it switches to detect mode, restoring the similarity score to a rock-solid 100%.

---

## 📦 Custom Library Usage (ai_core)
To keep the codebase modular, thread-safe, and scalable, all AI states, buffers, and peripheral pointers are encapsulated in a custom handle structure (AICore_Handle_t).

**Step 1: Initialization**
Include the header and initialize the AI Core with a 90% accuracy threshold:

    #include "ai_core.h"
    static AICore_Handle_t hAI;
    AICore_Init(&hAI, &hadc1, &htim7, 90);

**Step 2: Start Hardware Background Tasks**
Start the ADC in DMA Circular mode and start the 2kHz trigger timer:

    AICore_Start_DMA(&hAI);
    HAL_TIM_Base_Start(&htim3);

**Step 3: Non-Blocking Processing in Main Loop**
Continuously poll the processing function inside your while(1) loop:

    while (1) {
        if (AICore_Process(&hAI) == 1) {
            if (hAI.health_status == AICORE_MOTOR_ANOMALY) {
                HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_SET); // Trigger Alarm
            } else {
                HAL_GPIO_WritePin(LD2_GPIO_Port, LD2_Pin, GPIO_PIN_RESET);
            }
        }
    }

---

## 💻 Python SCADA Dashboard (Asynchronous Telemetry)
To visualize the AI's performance, the MCU packages a 1034-byte binary payload containing the health status, similarity score, execution time, and raw float data. This packet is transmitted via **UART2 with DMA**, ensuring the CPU is never blocked during transmission.

The repository includes a Python-based real-time dashboard (dashboard.py) built with matplotlib and pyserial.
*   **Time-Domain Signal:** Live view of the ADC DMA buffer.
*   **Frequency Domain (FFT):** Live spectrum analysis showing the 350Hz anomaly peak.
*   **Dynamic Similarity Gauge:** Visual alerts (Green = Normal, Red = Anomaly).
*   **Latency Monitor:** Live display of the AI inference execution time (~0.5ms).

### Running the Dashboard:
Install the required libraries and run the script:

    pip install pyserial matplotlib numpy
    python dashboard.py

*(Note: Update the SERIAL_PORT variable inside dashboard.py to match your STM32 COM port).*

---

## 👨‍💻 Author & Contact
**Mohammad Hosseini**
* Embedded Systems & Power Electronics Engineer
* Instructor at FaraDars & Founder of Esfahan Drive

For any inquiries, collaborations, or questions regarding this project:
* 📧 **Email:** about.mim.hosseini@gmail.com
* 🌐 **Website:** [esfdrive.ir](https://esfdrive.ir)

---
*If you found this project helpful, please consider giving it a ⭐!*
