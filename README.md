# Deep-Learning-Based sEMG-Controlled Below-Elbow Prosthetic Arm

## 📌 Executive Summary
This repository presents a high-performance, real-time edge computing architecture designed for a surface Electromyography (sEMG)-controlled below-elbow prosthetic arm. Built around a **Raspberry Pi 4**, the system bridges specialized high-speed hardware acquisition (AD7606 ADC) with robust Digital Signal Processing (DSP) and Machine Learning (ML) inference pipelines. 

Engineered for **deterministic execution and ultra-low latency**, the software bypasses standard operating system bottlenecks by implementing strict multi-core isolation (`sched_setaffinity`), real-time scheduling policies (`SCHED_FIFO`), and zero-copy shared memory communication.

---

## ⚙️ Core Architecture & Multiprocessing Strategy
To achieve a stable sampling frequency of 6000 Hz and real-time inference without dropped frames or UI stuttering, the application workload is distributed across isolated CPU cores:

*   **Core 3 (Acquisition Engine):** Dedicated exclusively to high-speed hardware polling via SPI and hardware PWM generation, running under maximum real-time priority (`SCHED_FIFO`, priority 99).
*   **Core 2 (DSP Engine):** Continuously pulls raw data from ring buffers, applies real-time filtering, performs decimation, and feeds processed streams to storage or inference engines.
*   **Core 1 (AI / Inference Engine):** Executes sliding-window feature extraction, machine learning classification, RMS thresholding, and state-machine stabilization independently of UI threads.
*   **Core 0 (Main GUI & Visualization Thread):** Runs the PySide6 application, handling interactive PyQtGraph dashboards, live signal plotting, and user session controls.

---

## 🛠️ Hardware & Communication Layer
*   **ADC:** AD7606 (16-bit, 8-channel simultaneous sampling Analog-to-Digital Converter).
*   **Communication:** SPI bus operating at 10 MHz (`spidev`, mode `0b10`) with hardware-level byte unpacking via `struct`.
*   **Timing & Synchronization:** Utilizes `pigpio` hardware PWM to generate precise conversion start (`CONVST`) pulses at 6000 Hz, synchronized with the AD7606 `BUSY` pin status.
*   **Inter-Process Communication:** Custom lock-free `SharedRingBuffer` built on `multiprocessing.Array` and `ctypes` to ensure zero-copy, tear-free memory sharing between isolated processes.

---

## 📂 Repository Structure & Modules

### 1. `Data_Collector/`
The clinical dataset acquisition suite designed to safely guide subjects through multi-movement, multi-round recording sessions.
*   **`AADataCollector1.py`**: Features a state-machine-driven PySide6 GUI, live signal plotting, automatic data trimming (ignoring transient muscle start-up noise), and asynchronous dataset logging directly into structured **HDF5 (`.h5`)** files containing subject metadata and labeled sEMG streams.

### 2. `Predictor/`
The real-time edge inference engine responsible for translating live muscle contractions into discrete prosthetic movement commands.
*   **`AAPredictor.py`**: Loads serialized machine learning models (`model.pkl`) and scalers (`standard_scaler.pkl`) via `joblib`. It computes time-domain features over sliding windows and outputs classification states.

---

## 🧠 Signal Processing & AI Pipeline

### Digital Signal Processing (DSP)
*   **Bandpass Filtering:** 4th-order Butterworth filter isolating the primary sEMG frequency band ($20\text{ Hz}$ to $500\text{ Hz}$).
*   **Notch Filtering:** Cascaded IIR notch filters targeting power-line interference and harmonics ($50\text{ Hz}, 100\text{ Hz}, 150\text{ Hz}, 200\text{ Hz}$).
*   **Decimation:** Downsampling data streams to optimize feature extraction throughput.

### Feature Extraction
Computes four core time-domain metrics across active channels:
1.  **MAV (Mean Absolute Value)**
2.  **RMS (Root Mean Square)**
3.  **WL (Waveform Length)**
4.  **ZC (Zero Crossings)**

### Stabilization & Post-Processing Engine
To prevent jitter, false triggers, and erratic actuator behavior, the prediction pipeline implements a multi-layered defense mechanism:
*   **RMS Firewall:** Time-gated debouncing that evaluates rolling variance and signal amplitude to ensure muscle activation is intentional before dispatching inference requests.
*   **Voting Window & Kinematic Smoothing:** A history-based voting buffer combined with a temporal cooldown/lock mechanism to guarantee steady state transitions between `REST` and movements.

---

## 🚀 Getting Started

### Prerequisites
*   Raspberry Pi OS (or compatible Linux environment with root/sudo access for real-time scheduling).
*   Python 3.10+
*   Required Python packages:
    ```bash
    pip install numpy scipy scikit-learn joblib PySide6 pyqtgraph h5py spidev pigpio
    ```
*   Ensure the `pigpiod` daemon is active on your system:
    ```bash
    sudo systemctl enable pigpiod
    sudo systemctl start pigpiod
    ```

### Execution
Run the data collection interface with root privileges to secure high-priority scheduling (`SCHED_FIFO`):
```bash
sudo python3 Data_Collector/AADataCollector1.py
