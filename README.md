# STM32N6 Face Recognition System

Real-time face detection and recognition on the **STM32N6570-DK** development board using the on-chip NPU and STEdgeAI.

> **Based on** [STM32N6-FaceRecognition](https://github.com/PeleAB/STM32N6-FaceRecognition) by [PeleAB](https://github.com/PeleAB) — STMicroelectronics.

---

## Features

- **Real-time Face Detection** — CenterFace model (128×128, INT8, ~9 ms)
- **Face Recognition** — MobileFaceNet embeddings + cosine similarity (~120 ms)
- **Multi-face Tracking** — embedding bank with similarity voting
- **NPU Acceleration** — STM32N6 Neural Processing Unit
- **Live Camera Input** — IMX335, VD55G1, or VD66GY supported
- **LCD Output** — 800×480 display with bounding boxes and identity labels

---

## Hardware Requirements

| Component | Details |
|-----------|---------|
| Board | **STM32N6570-DK** |
| Camera | IMX335, VD55G1, or VD66GY module |
| Display | LCD 800×480 (included on DK board) |
| PC connection | USB (ST-Link, for flashing and debug) |

---

## Software Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| **Git Bash** | any | Run `.sh` scripts on Windows |
| **STM32CubeIDE** | 1.18.1+ | Contains ARM GCC 13.3 toolchain |
| **STM32CubeProgrammer** | 2.18+ | Flashing tool (`STM32_Programmer_CLI`) |
| **STEdgeAI** | 3.0 | AI model conversion (optional — models already included) |
| **Python** | 3.x | Required by build scripts (must be in PATH as `python3`) |

> **Windows note:** All `.sh` scripts must be run from **Git Bash**, not PowerShell or CMD.

---

## 📁 Project Structure

```
STM32N6570-FaceRecognition-FaceDetection/
├── stm32_tools_config.json       # Tool paths configuration — edit this first
├── input_models/                 # Source AI models (.onnx, .tflite)
├── converted_models/             # STEdgeAI output (not needed unless regenerating models)
├── scripts/                      # Build and deployment scripts
│   ├── compile_model.sh          # Regenerate AI model C code via STEdgeAI
│   ├── sign_binary.sh            # Sign a compiled .bin for STM32N6 boot
│   └── flash_firmware.sh         # Flash all components to the board
├── embedded/                     # Main STM32 firmware project
│   ├── Makefile                  # ← Primary build system (use this)
│   ├── Models/                   # AI model C files (pre-generated, ready to compile)
│   ├── Binary/                   # Pre-built hex files ready for flashing
│   │   ├── ai_fsbl.hex           # First Stage Boot Loader (pre-built, do not regenerate)
│   │   └── STM32N6_FaceRecognition_signed.hex  # Signed app (generated after build)
│   ├── Src/                      # Application source code
│   ├── Inc/                      # Header files (app_config.h for tuning)
│   ├── Middlewares/              # AI runtime (LL_ATON) + Camera middleware
│   └── STM32Cube_FW_N6/         # STM32N6 HAL drivers and BSP
└── Doc/                          # Additional documentation
```

---

## 🚀 Quick Start (Windows / Git Bash)

### Step 1 — Configure tool paths

Edit `stm32_tools_config.json` and verify that all paths match your installation:

```json
{
  "tools": {
    "stm32cubeide":      { "path": "C:/ST/STM32CubeIDE_1.18.1/STM32CubeIDE" },
    "stm32edgeai":       { "path": "C:/ST/STEdgeAI/3.0/Utilities/windows/stedgeai.exe" },
    "stm32programmer":   { "path": "C:/Program Files/.../STM32_Programmer_CLI.exe" },
    "stm32signingtool":  { "path": "C:/Program Files/.../STM32_SigningTool_CLI.exe" },
    "arm_gcc_toolchain": { "path": "C:/ST/STM32CubeIDE_1.18.1/.../tools/bin" },
    "external_loader":   { "path": "C:/Program Files/.../MX66UW1G45G_STM32N6570-DK.stldr" }
  }
}
```

> **The AI models in `embedded/Models/` are already pre-compiled.** You do **not** need to run `compile_model.sh` unless you want to regenerate them from scratch.

---

### Step 2 — Build the firmware

Open **Git Bash** and run:

```bash
export GCC_PATH="C:/ST/STM32CubeIDE_1.18.1/STM32CubeIDE/plugins/com.st.stm32cube.ide.mcu.externaltools.gnu-tools-for-stm32.13.3.rel1.win32_1.0.0.202411081344/tools/bin"

cd embedded
make clean
make -j4 GCC_PATH="$GCC_PATH"
```

Expected output on success:
```
Memory region         Used Size  Region Size  %age Used
      AXISRAM1_S:      672520 B      1023 KB     64.20%
           PSRAM:     2533120 B        16 MB     15.10%
   text    data     bss
 587352    1024 2617256
```

---

### Step 3 — Sign the binary

```bash
export PATH="C:/Program Files/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin:$PATH"

# Sign
STM32_SigningTool_CLI -s -bin build/Project.bin -nk -t ssbl -hv 2.3 -o build/Project_sign.bin

# Convert to Intel HEX at application base address
"$GCC_PATH/arm-none-eabi-objcopy" -I binary -O ihex \
  --change-address 0x70100000 \
  build/Project_sign.bin \
  Binary/STM32N6_FaceRecognition_signed.hex
```

---

### Step 4 — Flash the board

> ⚠️ Set the **BOOT1 switch to the RIGHT position** (development mode) before flashing.

```bash
PROG="C:/Program Files/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin/STM32_Programmer_CLI.exe"
EL="C:/Program Files/STMicroelectronics/STM32Cube/STM32CubeProgrammer/bin/ExternalLoader/MX66UW1G45G_STM32N6570-DK.stldr"
BINDIR="./Binary"

# 1. Flash FSBL (First Stage Boot Loader) at 0x70000000
"$PROG" -c port=SWD mode=HOTPLUG -el "$EL" -hardRst -w "$BINDIR/ai_fsbl.hex"

# 2. Flash signed application at 0x70100000
"$PROG" -c port=SWD mode=HOTPLUG -el "$EL" -hardRst -w "$BINDIR/STM32N6_FaceRecognition_signed.hex"
```

> The AI models are **embedded in the firmware binary** — there are no separate model `.hex` files to flash.

---

### Step 5 — Boot and run

1. Set **BOOT1 switch to the LEFT position** (boot from external flash)
2. Power cycle the board (unplug and reconnect USB power)
3. The system starts automatically — face detection is active immediately

---

## 🎮 Using the System

### Display feedback

| Color | Meaning |
|-------|---------|
| 🟥 Red bounding box | Face detected — **not recognized** |
| 🟩 Green bounding box | Face detected — **recognized / authorized** |

### Enrolling a face (USER1 button)

| Action | Duration | Effect |
|--------|----------|--------|
| **Short press** | < 1 second | ➕ Enrolls the currently detected face into the recognition bank |
| **Long press** | ≥ 1 second | 🗑️ Resets the entire face bank (removes all enrolled faces) |

**Procedure:**
1. Place your face in front of the camera — wait for a detection box to appear
2. **Short press USER1** to enroll
3. Your face box turns **green** on the next frame

> **Note:** Enrolled faces are stored in RAM and are **lost on power cycle**. There is no persistent storage in the current version.

---

## ⚙️ Configuration

Key parameters in `embedded/Inc/app_config.h`:

```c
// Camera orientation
#define CAMERA_FLIP CMW_MIRRORFLIP_NONE   // or CMW_MIRRORFLIP_FLIP / _MIRROR / _FLIP_MIRROR

// Detection sensitivity
#define AI_PD_MODEL_PP_CONF_THRESHOLD  (0.5f)   // Increase to reduce false detections
#define AI_PD_MODEL_PP_IOU_THRESHOLD   (0.3f)
#define AI_PD_MODEL_PP_MAX_BOXES_LIMIT (10)     // Max simultaneous face detections

// Input source
#define INPUT_SRC_MODE INPUT_SRC_CAMERA          // or INPUT_SRC_PC

// Debug: replace camera with a static test image
//#define DUMMY_INPUT_BUFFER
```

---

## 🤖 AI Models

| Model | Architecture | Input | Output | Inference |
|-------|-------------|-------|--------|-----------|
| Face Detection | CenterFace (INT8) | 128×128 RGB | Bounding boxes + keypoints | ~9 ms |
| Face Recognition | MobileFaceNet (INT8) | 112×112 RGB | 128-dim embedding | ~120 ms |

**Overall performance:** ~17 FPS (detection) / ~5 FPS (detection + recognition)

The models are pre-compiled as C source files in `embedded/Models/` and use the **LL_ATON** API (compatible with the runtime in `Middlewares/AI_Runtime/`).

> ⚠️ **STEdgeAI version warning:** If you regenerate the models with STEdgeAI 3.0+, the generated code uses a new `STAI` API that is **not compatible** with the LL_ATON runtime included in this project. Always use the pre-generated files in `embedded/Models/` unless you also update the middleware.

---

## System Architecture

```
Camera → ISP → Face Detection (NPU) → Face Cropping → Face Recognition (NPU) → LCD
                       ↓                                        ↓
               Bounding boxes                           128-dim embeddings
                                                               ↓
                                                    Cosine similarity vs. bank
                                                               ↓
                                                    Green/Red box on display
```

### Memory Layout (STM32N6 External Flash)

| Address | Content |
|---------|---------|
| `0x70000000` | FSBL — First Stage Boot Loader |
| `0x70100000` | Signed application firmware (includes AI models) |

---

## Documentation

| File | Content |
|------|---------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Detailed software architecture |
| [TECHNICAL_DOCUMENTATION.md](TECHNICAL_DOCUMENTATION.md) | Technical deep-dive |
| [BUILD_SETUP.md](BUILD_SETUP.md) | Additional build notes after cloning |
| [Doc/Boot-Overview.md](Doc/Boot-Overview.md) | STM32N6 boot process |
| [Doc/Build-Options.md](Doc/Build-Options.md) | Makefile build options |
| [Doc/Deploy-your-tflite-Model.md](Doc/Deploy-your-tflite-Model.md) | How to add/replace AI models |
| [CODING_STANDARDS.md](CODING_STANDARDS.md) | Embedded C coding guidelines |

---

## Contributing

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

1. Fork the repository
2. Create a feature branch
3. Install git hooks: `./install-git-hooks.sh`
4. Follow coding standards and run `make check`
5. Submit a pull request

---

## License

- Application code: [LICENSE](LICENSE)
- STM32 components: ST proprietary license
- Third-party libraries: see individual components

## Acknowledgments

- STMicroelectronics for STM32N6 platform and STEdgeAI
- Original model authors (CenterFace, MobileFaceNet)
