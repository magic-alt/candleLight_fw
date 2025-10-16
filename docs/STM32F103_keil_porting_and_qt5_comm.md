# STM32F103 candleLight 工程移植与 Windows Qt5 上位机通信指南

## 1. 项目概述
candleLight 固件工程主要面向 STM32F0/F0x2/F0x72 与 STM32G0 系列 MCU，固件采用 CMake 构建，并在 `src/`、`include/` 及 `libs/` 目录下提供了 USB、CAN 和协议栈实现，是 Linux `gs_usb` 协议兼容固件的参考实现。【F:README.md†L1-L43】自研开发板如采用 STM32F103，需要在硬件资源、驱动适配和构建系统方面进行额外处理，才能在 Keil MDK 环境下成功构建与调试。

本文档分为两大部分：

1. 详细说明如何将 candleLight 固件移植到以 STM32F103 为核心的 Keil MDK 项目中，包括项目初始化、底层驱动适配、编译链接配置及调试验证。
2. 介绍基于 Windows 平台的 Qt 5 上位机如何与 candleLight 下位机通讯，涵盖驱动准备、数据链路初始化以及典型收发流程。

## 2. 环境准备与项目结构

### 2.1 开发环境与工具
- **硬件**：自研 STM32F103（建议使用 STM32F103CB/CC 系列，拥有 USB FS 与 bxCAN 外设）、高速 CAN 收发器、USB 全速接口。
- **软件**：
  - Keil MDK-ARM 5.38 及以上版本，安装 STM32F1 系列设备包。
  - ST-LINK/V2 调试器驱动与工具链。
  - Git 工具，用于获取 candleLight 源码。
  - 可选：STM32CubeMX（生成系统时钟与 USB/CAN 初始化代码，辅助对比参考）。

### 2.2 candleLight 工程结构总览
- `include/`：公共头文件及协议定义，例如 `config.h` 中定义的 USB VID/PID。【F:include/config.h†L45-L49】
- `src/`：固件主要源文件，包含 USB、CAN、协议栈、调度逻辑等。
- `libs/STM32_HAL/`、`libs/STM32_USB_Device_Library/`：基于 STM32 HAL 与 USB 中间件的依赖库。
- `ldscripts/`：GNU Arm 工具链使用的链接脚本，可作为 Keil Scatter 文件配置参考。
- `cmake/`：原工程使用的 CMake 工具链与板级配置。

了解上述目录有助于在 Keil 环境中正确组织文件和宏定义。

## 3. Keil MDK 移植流程

### 3.1 建立 Keil 工程骨架
1. 在 Keil 中选择 **Project → New uVision Project**，新建工程，目标器件选择对应的 STM32F103 型号（例如 `STM32F103CBTx`）。
2. 在 **Manage Run-Time Environment** 中勾选以下组件：
   - `CMSIS → Core`
   - `Device → Startup`
   - 若计划使用 HAL：`STM32Cube HAL → CAN`, `GPIO`, `RCC`, `USB Device`
3. 生成默认的启动文件 `startup_stm32f10x_xx.s` 和系统时钟文件 `system_stm32f10x.c`。

### 3.2 导入 candleLight 源码
1. 在工程目录中创建与源码仓库一致的子目录（如 `Src`, `Inc`, `Middlewares`），将 `include/` 下的头文件复制到 `Inc/`，将 `src/` 需要的 C 源文件复制到 `Src/`。
2. `libs/STM32_HAL` 中与 F0/F7 相关的 HAL 文件需替换为 F1 对应版本，可从 STM32CubeF1 包中提取 `stm32f1xx_hal_*.c/h`，保持 API 接口一致。
3. 将 `libs/STM32_USB_Device_Library` 中与 USB Device Class 相关的文件加入 `Middlewares`，并补充 ST 官方 F1 USB FS 库（Keil 包内包含 `STM32_USB-FS-Device_Lib`）。
4. 在 Keil 中通过 **Project → Add Existing Files to Group** 将上述文件加入对应分组，确保头文件路径在 **Options for Target → C/C++** 中配置为 `../Inc`, `../Middlewares/...` 等。

### 3.3 预处理宏与编译选项
- 在 **Options for Target → C/C++** 下添加以下宏：
  - `USE_HAL_DRIVER`
  - `STM32F103xB`（或具体芯片宏）
  - `CANARD_ENABLE_USB`（若需要兼容原项目宏，可自定义）
- 开启 `--c99` 以兼容项目中使用的 C99 特性。
- 若使用 `printf` 调试，可启用微库 `--use_microLIB`，并在 `syscalls.c` 中重定向 `_write`。

### 3.4 链接脚本（Scatter File）配置
参考原仓库的 `ldscripts/flash_stm32f072xb.ld` 结构，将 Flash 与 RAM 区域改写为 STM32F103 的地址：
```text
LR_IROM1 0x08000000 0x00020000  {   ; 128 KB Flash
  ER_IROM1 0x08000000 0x00020000  {
    *.o (RESET, +First)
    *(InRoot$$Sections)
    .ANY (+RO)
  }
  RW_IRAM1 0x20000000 0x00005000  { ; 20 KB SRAM
    .ANY (+RW +ZI)
  }
}
```
将该 Scatter 文件保存为 `candleLight_f103.sct` 并在 **Options for Target → Linker** 中勾选 `Use Memory Layout from Target Dialog` 取消，改为自定义 Scatter 文件。

### 3.5 时钟与外设初始化
1. 在 `system_stm32f10x.c` 中配置系统时钟至 72 MHz，并启用 USB FS 所需的 48 MHz 时钟（通常通过 PLL3/USB prescaler 达成）。
2. 在 `hal_init.c`（自建）或主函数中调用：
   ```c
   HAL_Init();
   SystemClock_Config();
   MX_GPIO_Init();
   MX_CAN_Init();
   MX_USB_DEVICE_Init();
   ```
   其中 `MX_USB_DEVICE_Init` 需结合 ST USB FS 库创建 `USBD_HandleTypeDef` 并注册自定义的 `gs_usb` 类。
3. 将原 `src/usb_*` 中使用的底层寄存器访问替换为 HAL 或寄存器级实现，确保 USB 中断向量表与 Keil 启动文件一致。

### 3.6 gs_usb 协议层适配
1. `include/gs_usb.h` 定义了 USB 端点与控制请求号，确保在 F1 平台下端点号保持 `0x81/0x02`。【F:include/gs_usb.h†L29-L83】
2. 将原针对 F0/G0 的 USB 回调移植到 F1 HAL：
   - 在 `usbd_gsusb_if.c`（新建）实现 `USBD_GSUSB_Init/DeInit/Setup/DataIn/DataOut`。
   - 在 `usbd_conf.c` 中完成低层驱动（LPM、SOF、中断）配置，参考 Keil 提供的 USB 设备模板。
3. CAN 层可继续沿用 `src/can.c` 的调度逻辑，但需将寄存器地址修改为 bxCAN，对应的 HAL 调用为 `HAL_CAN_Start`、`HAL_CAN_AddTxMessage` 等。

### 3.7 调试与验证
1. 在 Keil 中选择 **ST-LINK** 调试器，配置下载算法为 `STM32F10x Flash`。
2. 通过 **Debug → Start/Stop Debug Session** 下载固件，实时观察 USB 与 CAN 中断触发情况。
3. 连接 PC，打开设备管理器确认枚举为 WinUSB 设备（VID:0x1D50, PID:0x606F）。【F:include/config.h†L45-L49】
4. 使用 CAN 总线分析仪验证帧收发，确保 `GS_CAN_MODE_NORMAL` 等模式能够通过控制请求设置。【F:include/gs_usb.h†L33-L58】

## 4. STM32F103 特定注意事项
1. **USB 与 CAN 同时使用的限制**：STM32F103 共享 USB SRAM，需要合理划分端点缓冲区，推荐使用 ST 官方 USB 库的双缓冲模式。
2. **电源与振荡器**：USB FS 需要 48 MHz，确保外部晶振为 8 MHz，并配置正确的 PLL 分频。
3. **引脚复用**：检查板级原理图，确认 CAN_TX/CAN_RX、USB_DP/DM 未冲突，并正确配置 `GPIO_AF_CAN` 与 `GPIO_AF_USB`。
4. **中断优先级**：USB 和 CAN 中断需设置合适的抢占与响应优先级，避免发送队列阻塞。
5. **固件升级**：若需 DFU 功能，可结合 ST 的 USB DFU 类或使用 Keil 提供的 IAP 模板。

## 5. Windows Qt5 上位机通信方案

### 5.1 驱动与接口
- candleLight 固件包含 WCID 描述符，可在 Windows 10+ 上无需额外驱动直接作为 WinUSB 设备识别。【F:README.md†L77-L83】
- Qt 5 本身不直接提供 WinUSB 封装，可采用以下方案之一：
  1. 使用 `libusb-1.0` Windows 版本，通过 Qt 的跨平台编译系统链接静态/动态库。
  2. 使用 `QCanBus` + 第三方 gs_usb 插件（若已有驱动支持）。对 Windows 环境，推荐自行封装基于 libusb 的接口。

### 5.2 通信初始化流程（以 libusb 封装为例）
1. 在 Qt 工程的 `.pro` 文件中添加：
   ```pro
   INCLUDEPATH += $$PWD/3rdparty/libusb/include
   LIBS += -L$$PWD/3rdparty/libusb/lib -lusb-1.0
   DEFINES += GSUSB_VID=0x1D50 GSUSB_PID=0x606F
   ```
2. 在运行时：
   ```cpp
   libusb_context *ctx = nullptr;
   libusb_init(&ctx);
   libusb_device_handle *handle = libusb_open_device_with_vid_pid(ctx, GSUSB_VID, GSUSB_PID);
   libusb_claim_interface(handle, 0);
   ```
3. 发送控制请求设置波特率/模式：
   ```cpp
   gs_host_config config;
   config.byte_order = 0;
   libusb_control_transfer(handle,
       LIBUSB_REQUEST_TYPE_VENDOR | LIBUSB_RECIPIENT_INTERFACE | LIBUSB_ENDPOINT_OUT,
       GS_USB_BREQ_HOST_FORMAT,
       0, 0,
       reinterpret_cast<unsigned char*>(&config),
       sizeof(config),
       1000);
   ```
4. 配置位时序并启动 CAN：
   - 构造 `gs_device_bittiming` 并通过 `GS_USB_BREQ_BITTIMING` 发送。
   - 发送 `GS_USB_BREQ_MODE`，`wValue`= `GS_CAN_MODE_START`，`wIndex` 设为通道号。
5. 数据收发：
   - **发送**：向 Bulk OUT 端点 `0x02` 写入 `gs_host_frame`（含 CAN ID、DLC、数据），使用 `libusb_bulk_transfer(handle, 0x02, buffer, length, &transferred, timeout)`。【F:include/gs_usb.h†L29-L83】
   - **接收**：从 Bulk IN 端点 `0x81` 轮询读取，解析 `flags` 确认是否为 FD 帧或错误帧。
6. 清理资源：退出时调用 `libusb_release_interface`、`libusb_close`、`libusb_exit`。

### 5.3 Qt 信号槽封装建议
- 创建 `GsUsbDevice` QObject，内部维护独立线程处理 bulk 读写，将接收到的帧通过 `signals` 发射至主线程。
- 使用 `QMutex` 保护发送队列，避免多线程竞争。
- 通过 `QTimer` 定时发送 `GS_USB_BREQ_TIMESTAMP` 请求，校验设备时间戳同步。

### 5.4 错误处理与调试
- 若 `libusb_open_device_with_vid_pid` 返回空指针，检查 Windows 是否自动加载了其他驱动，可使用 Zadig 重新绑定 WinUSB。
- 控制传输返回 `LIBUSB_ERROR_PIPE` 多为请求号错误，核对 `gs_usb.h` 中的定义是否与固件一致。【F:include/gs_usb.h†L85-L117】
- CAN 帧发送超时需检查固件是否启用了回环或监听模式，以及总线终端电阻状态。

## 6. 交付检查清单
- [ ] Keil 工程可成功编译并下载至 STM32F103，USB 枚举正常。
- [ ] CAN 总线可收发标准帧，模式切换有效。
- [ ] Qt5 上位机能识别设备并完成控制请求、数据收发。
- [ ] 所有源码文件添加必要注释，关键配置在文档中有出处说明。

## 7. 参考资料
- candleLight 官方仓库 README 与配置文件。【F:README.md†L1-L83】【F:include/config.h†L45-L49】
- `gs_usb` 协议头文件，控制请求与端点定义。【F:include/gs_usb.h†L29-L117】
- ST 官方 STM32F1 USB FS Device Library 与 HAL 文档。
