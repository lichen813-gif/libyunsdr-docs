# 兼容矩阵

SDK 基线为 `libyunsdr-26-01-00.1.zip`（包内 `26.00.00.1`）。一行对应一组机型、镜像、驱动、主机与传输配置。联机记录见下表及机型接入页。[C 库构建与安装](c-api.md#2-下载构建与安装) · [SWIG 构建 Python 绑定](c-api.md#25-使用-swig-构建-python-绑定) · [使用预编译绑定](python-guide.md#2-使用预编译-python-绑定)

| 机型 | 硬件修订 | SDK | 固件或 FPGA 镜像 | 主机与驱动 | 传输 | 功能与验证状态 |
| --- | --- | --- | --- | --- | --- | --- |
| 无板卡构建 | — | `26.00.00.1` | — | Windows x64，VS2019 16.11 / MSVC 19.29，libusb 1.0.29 | PCIEX、PCIES、USB3，RFIC 配置库开启 | Release 构建与安装通过，速率例程帮助可运行。RFIC 配置待联机验证。[MSVC 默认构建](c-api.md#232-配置编译和安装) |
| 无板卡构建 | — | `26.00.00.1` | — | Windows x64，VS2019，libusb 1.0.29 | 仅 USB3，RFIC 配置库关闭 | Release 构建与安装通过 |
| 无板卡 C 库构建 | — | `26.00.00.1` | — | Windows x64，MinGW-w64（TDM-GCC 9.2.0）、CMake 4.2.1、`gendef`。USB3 使用 libusb 1.0.29 的 MinGW x64 DLL 和 `dlltool` 生成的导入库 | PCIEX、USB3、PCIEX + USB3，RFIC 配置库关闭 | 三组 Release 构建、安装、C 程序动态链接和离线 tick 换算通过。PCIES 链接失败，设备联机待验证。[MinGW-w64 构建步骤](c-api.md#234-使用-mingw-w64-构建-c-库) |
| 无板卡通信实验构建 | — | `26.00.00.1` | — | Windows x64，VS2019，PortAudio 19.7.0、FFTW 3.3.5、libusb 1.0.29 | PCIEX、PCIES；另验证三种后端同时开启 | FM 音频、视频和相位校正目标的 Release 构建、安装及帮助输出通过 |
| 无板卡 Python 绑定 | — | `26.00.00.1` | — | Windows x64，VS2019、CPython 3.11.9、SWIG 4.2.1；USB3 构建使用 libusb 1.0.29 | PCIEX、PCIES；另验证三种后端同时开启 | Release 构建、安装、导入及 tick 换算通过。Spyder 图形调试和设备联机待验证。[MSVC 构建绑定](c-api.md#252-windows-x64-msvc) |
| 无板卡 Python 绑定 | — | `26.00.00.1` | — | Windows x64，独立 VS2022 Build Tools，MSVC `v143` 或可选 `v142`，CPython 3.11.9、SWIG 4.2.1 | PCIEX、PCIES、USB3，RFIC 配置库开启 | 独立 Build Tools 构建及无开发工具主机的预编译包导入待实测。[MSVC 构建绑定](c-api.md#252-windows-x64-msvc) |
| 无板卡 Python 绑定 | — | `26.00.00.1` | — | Windows x64，MinGW-w64（TDM-GCC 9.2.0）、CMake 4.2.1、SWIG 4.2.1、CPython 3.11.9。USB3 使用 libusb 1.0.29 的 MinGW x64 DLL 和 `dlltool` 生成的导入库 | PCIEX、USB3、PCIEX + USB3，RFIC 配置库关闭 | 三组 Release 构建、安装、导入和离线 tick 换算通过。PCIES 受 C 库依赖限制，使用 MSVC 构建。设备联机待验证。[MinGW-w64 构建绑定](c-api.md#253-windows-x64-mingw-w64) |
| C 库及 Python 绑定 | — | `26.00.00.1` | — | Linux x86-64 / AArch64，配套架构的 `libpcies.a`、`libfirmware.a` 与系统 libusb | PCIEX、PCIES、USB3，RFIC 配置库开启 | 已核对交付包的配置和依赖。原生构建、预编译库导入、Spyder 和联机待验证。[Linux C 库构建](c-api.md#242-配置编译和安装) · [Linux 绑定构建](c-api.md#254-linux-x86-64-与-aarch64) |
| [IQX8400](devices/iqx8400.md) | Rev1.1 | `26.00.00.1` 待联机 | 待记录 | Windows x64，配套 PCIe 驱动版本待记录 | `pciex:` / `pcies:` 取决于交付组合 | 设备打开、版本查询、采集、发射、双工及同步待验证 |
| [IQX8400](devices/iqx8400.md) | Rev1.1 | `26.00.00.1` 待联机 | 待记录 | Linux x86-64，发行版、内核和驱动版本待记录 | `pciex:` / `pcies:` 取决于交付组合 | 待构建、安装及联机验证 |
| 待确认 | 待记录 | `26.00.00.1` | 待记录 | Linux AArch64，发行版、内核和驱动版本待记录 | 由产品交付配置决定 | 已检查包内 AArch64 静态依赖的架构，待构建及联机验证 |
| USB3 产品按交付配置 | 待记录 | `26.00.00.1` | 待记录 | 主机、驱动版本与 libusb 版本待记录 | `usb3:` | USB3 设备打开、基础 I/Q 收发及通信实验待联机验证 |

完成联机后，记录设备型号和版本日志、驱动及固件标识、连接串、采样率、中心频率、通道掩码、请求与实际返回点数、设备时间戳、事件计数和 I/Q 文件大小。分别登记单路、多路、定时和持续收发结果。
