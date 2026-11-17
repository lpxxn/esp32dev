# ESP32-S3 hello_world 疑问与解答

整理自 hello_world 工程在 VS Code / ESP-IDF 扩展里的配置和调试过程。

适用环境：

- 工程：`E:\esp32dev\hello_world`
- 芯片：ESP32-S3（Xtensa）
- ESP-IDF：v6.1，路径 `C:\esp\v6.1\esp-idf`
- 工具链：`xtensa-esp32s3-elf-gcc`（GCC 15.2.0）
- ESP-IDF 扩展：2.3.0
- 板子上有两个 Type-C 口：一个标 **COM**（CH343 转串口），一个标 **USB**（芯片片内 USB JTAG）

---

## 1. Build 成功，但 `#include` 很多是红的

**现象**

`freertos/FreeRTOS.h`、`esp_system.h` 等头文件下面有红线，提示 `cannot open source file`，编辑器顶部还有 “Please update your includePath”。用 ESP-IDF 的 Build Project 却能编过。

**原因**

红线来自微软 C/C++ 扩展的 IntelliSense，不是编译错误。编译器用的是 `build/compile_commands.json` 里的头文件路径；IntelliSense 只看 `includePath`。当时工程里没有 `.vscode/c_cpp_properties.json`，它只在工程目录里找，找不到 `C:\esp\v6.1\esp-idf\components\...` 下的头文件。

**处理**

已添加 `.vscode/c_cpp_properties.json`。配置名是 **ESP-IDF**，编译器指向 `xtensa-esp32s3-elf-gcc`，并用 `compileCommands` 指向 `build/compile_commands.json`。右下角 C/C++ 配置选 **ESP-IDF**，然后执行 **Developer: Reload Window**。

---

## 2. “Select a Kit” 里该选哪个？

**可选条目**

- Clang / GCC `riscv32-esp-elf`
- GCC 15.2.0 `xtensa-esp-elf`
- Visual Studio 生成工具 2022（amd64、x86 等）

**选择**

选 **GCC 15.2.0 xtensa-esp-elf**。

ESP32-S3 是 Xtensa 内核，对应 `xtensa-esp32s3-elf-gcc`。`riscv32-esp-elf` 给 ESP32-C3、C6 这类 RISC-V 芯片用。Visual Studio 那几项是本机 x86/x64 编译器，不能用来编这个工程。

这个对话框来自 CMake Tools。ESP-IDF 工程日常编译用扩展自己的 **Build Project**，不要依赖这里选中的 Kit 去重新配置工程。见下一节。

---

## 3. 为什么 Build Project 能过，状态栏 “Build the selected target: [all]” 会失败？

**现象**

```text
CMake Error at CMakeLists.txt:5 (include):
  include could not find requested file:

    /tools/cmake/project.cmake

CMake Error at CMakeLists.txt:7 (idf_build_set_property):
  Unknown CMake command "idf_build_set_property".
```

CMake Tools 实际执行的是 Visual Studio 自带的 cmake，只传入了 `xtensa-esp-elf-gcc`，没有 ESP-IDF 环境。

**原因**

`CMakeLists.txt` 里是：

```cmake
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
```

这条命令运行时 `IDF_PATH` 是空的，路径被拼成 `/tools/cmake/project.cmake`，`idf_build_set_property` 也就不存在。

ESP-IDF 的 **Build Project** 走 `idf.py build`。扩展会先设好 `IDF_PATH`（`C:\esp\v6.1\esp-idf`）、`IDF_TARGET=esp32s3` 和 Xtensa 工具链，再调用 cmake。

**处理**

编译这个工程用 ESP-IDF 的 **Build Project**。状态栏那个 **Build** 是 CMake Tools 的按钮，点它会按没有 IDF 环境的方式重新配置 `build` 目录。

---

## 4. 烧录时 JTAG、UART、DFU 选哪个？

Type-C 已经插上时，选 **UART**。

插上后电脑会出现一个 COM 口，UART 用这个串口烧录。选完后在端口列表里选刚出现的那个口。当时识别到的是 CH343 的 **COM6**（插在标 **COM** 的口上）。

烧录方式记在 `idf.flashType`，之后可以在 `.vscode/settings.json` 里改。打断点仍然走 OpenOCD（`board/esp32s3-builtin.cfg`），和这次选的烧录方式分开。

### 三个选项分别是什么

| 选项 | 全称 | 做什么 |
| --- | --- | --- |
| UART | Universal Asynchronous Receiver-Transmitter，通用异步收发器 | 把板子当成 COM 口，用 `esptool.py` 跟芯片里的串口下载程序通信。日常烧录、看日志用这个。 |
| JTAG | Joint Test Action Group，联合测试行动组 | 通过调试口烧录，由 OpenOCD 写入。同一根插在 **USB** 口上的线也能走这条路。主要给下断点用，比较慢，还要先让 OpenOCD 和 USB 驱动正常。 |
| DFU | Device Firmware Upgrade，设备固件升级 | USB 标准里的升级协议。芯片在下载模式下变成 USB DFU 设备，电脑用 `dfu-util` 写固件，不经过 COM 口。一般只在没有串口、专门用 USB 升级时才用。 |

三种方式写的是同一份程序，差别只是电脑用哪条通道跟芯片说话。

---

## 5. 怎么下断点？

ESP-IDF 扩展 2.3 的命令面板里没有 “ESP-IDF: Debug your project”。搜 `debug` 时出现的 **ESP-IDF: Launch QEMU Debug Session** 是在电脑里模拟芯片，不会连真实的板子。

板上调试用 `.vscode/launch.json`，再按 F5。工程里已有这份配置：

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "gdbtarget",
      "request": "attach",
      "name": "ESP-IDF",
      "initialBreakpoint": "app_main"
    }
  ]
}
```

步骤：

1. 线插在标 **USB** 的口上（片内 JTAG）。OpenOCD 配置已是 `board/esp32s3-builtin.cfg`。
2. `Ctrl+Shift+P`，运行 **ESP-IDF: Select Port to Use**，选当前这个口对应的 COM 口。
3. 用 ESP-IDF 的 **Build Project** 编译，再 **Flash** 烧进去。改过代码后要重新烧录，否则断点行号和板子里的程序对不上。工程已打开 `CONFIG_COMPILER_OPTIMIZATION_DEBUG`，带调试符号。
4. 在 `main/hello_world_main.c` 里 `printf("Hello world!\n");` 的行号左侧点一个红点。
5. `Ctrl+Shift+P`，运行 **ESP-IDF: OpenOCD Manager**，点 **Start**。Output 里选 **ESP-IDF**，看到 `Listening on port 3333 for gdb connections` 后再继续。
6. 按 **F5**。运行和调试里的配置选 **ESP-IDF**。会话会停在 `app_main`，或停在你下的断点上。
7. `F10` 单步跳过，`F11` 步入，`F5` 继续。

状态栏里 CMake Tools 的虫子图标没有 `IDF_PATH`，也没连 OpenOCD，不要用它调试。

---

## 6. 按 F5 提示 “OpenOCD is not running”

**提示**

```text
OpenOCD is not running. Please start OpenOCD before launching the debug session.
```

对话框上的 **Open 'launch.json'** 只是打开配置文件。先点 **Cancel**。

**处理**

GDB 要通过 OpenOCD 才能连上芯片。先单独启动 OpenOCD：

1. `Ctrl+Shift+P`，运行 **ESP-IDF: OpenOCD Manager**（中文界面是 **ESP-IDF: OpenOCD 管理器**）。
2. 点 **Start**。
3. **View → Output**，下拉选 **ESP-IDF**。出现监听 3333 端口后再按 **F5**。

---

## 7. OpenOCD 报 `esp_usb_jtag: could not find or open device`

**日志**

```text
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Error: esp_usb_jtag: could not find or open device!
OpenOCD Exit with non-zero error code 1
```

**原因**

当时电脑上只有 `USB-Enhanced-SERIAL CH343 (COM6)`，硬件 ID 是 `VID_1A86`。这是 **COM** 口上的 USB 转串口芯片。OpenOCD 使用 `board/esp32s3-builtin.cfg` 时要找的是 ESP32-S3 片内 USB JTAG，硬件 ID 是 `VID_303A`。线插在 COM 口上时，这个设备不会出现。

**处理**

把 Type-C 线换到标着 **USB** 的口。设备管理器里出现 **USB JTAG/serial debug unit**（`VID_303A`）后，再在 OpenOCD Manager 里点 Start。这一步已经成功。

如果换口之后设备已经出现，OpenOCD 仍然打不开，用 [Zadig](https://zadig.akeo.ie/) 把 **USB JTAG/serial debug unit (Interface 2)** 的驱动换成 **WinUSB**。Interface 0 是串口，不要改。

### 需要装 CH340 驱动吗？

这次报错不需要。板上的串口芯片是 **CH343**，COM6 当时已经正常，驱动是好的。

CH340 和 CH343 都是沁恒的 USB 转串口芯片。它们的驱动让 Windows 多出一个 COM 口，ESP-IDF 用这个口做 UART 烧录，并用监视器看 `printf`。片内 JTAG 不经过这颗芯片。

---

## 8. 板上 USB 口和 COM 口有什么区别？

| | COM 口 | USB 口 |
| --- | --- | --- |
| 接到哪里 | 板上的 CH343，再接到芯片 UART0 | ESP32-S3 自己的 USB 引脚（GPIO19、GPIO20） |
| 电脑上看到什么 | `USB-Enhanced-SERIAL CH343`，例如 COM6 | `USB JTAG/serial debug unit`（`VID_303A`），另外还有一个虚拟串口 |
| 适合做什么 | 日常烧录、串口监视器 | OpenOCD 下断点。这个口里同时有虚拟串口和 JTAG |
| OpenOCD | 找不到 `esp_usb_jtag` 设备 | `board/esp32s3-builtin.cfg` 用的就是这个口 |

当前 `sdkconfig`：

- 主控制台是 UART0（**COM** 口），波特率 115200，`CONFIG_ESP_CONSOLE_UART_DEFAULT=y`
- USB Serial/JTAG 是第二路控制台，`CONFIG_ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG=y`

所以 `Hello world` 从两个口都能看到。日常烧录和看日志用 **COM** 口；要下断点时把线换到 **USB** 口，再启动 OpenOCD。

---

## 9. OpenOCD 是什么？

OpenOCD（Open On-Chip Debugger，开源片上调试器）是电脑上的一个程序，负责跟芯片的调试口说话。这块板子的调试口就是 **USB** 口里的 JTAG。

按 F5 时是两段连接：

1. OpenOCD 连着板子，能让 CPU 停住、单步、读内存。
2. VS Code 里的 GDB 连着 OpenOCD 的 **3333** 端口，把编辑器里的红点变成芯片上的断点。

另外两个端口：4444 是 telnet，6666 是 tcl。

OpenOCD 不编译，也不负责平时那种 UART 烧录。烧录走 **COM** 口和 `esptool.py`；OpenOCD 只负责调试。`Listening on port 3333 for gdb connections` 表示它已经在等 GDB。
