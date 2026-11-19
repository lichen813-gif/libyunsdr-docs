# libyunsdr 二次开发文档

第一版以实际交付的 `libyunsdr-26-01-00.1.zip` 为基线，包内 SDK 版本为 `26.00.00.1`。

| 页面 | 内容 |
| --- | --- |
| [通用 C API 使用说明](docs/c-api.md) | PCIEX、PCIES、USB3 的下载、构建、安装、首帧接收、测试程序参数、函数参考、架构及 I/Q 数据 |
| [Windows x64 MinGW-w64 构建](docs/c-api.md#234-使用-mingw-w64-构建-c-库) | C 库的 PCIEX、USB3 及两种后端同时开启的构建、安装和离线验证 |
| [基础测试命令](docs/c-api.md#41-从哪里运行) | Windows 和 Linux 的测试目录、程序启动和参数 |
| [Python 开发指南](docs/python-guide.md) | SWIG 绑定构建、Windows 和 Linux 虚拟环境与 Spyder、首帧接收、循环发射、波形分析和测试截图 |
| [FM 音频与视频传输实验](docs/c-api.md#44-fm-音频接收与视频传输实验) | PortAudio、FFTW、通信实验构建，FM 播放，VLC/QPSK 视频传输和星座图 |
| [兼容矩阵](docs/compatibility.md) | 按 SDK、驱动、固件、主机和传输方式记录验证结果 |
| [IQX8400 Rev1.1 接入](docs/devices/iqx8400.md) | 硬件手册、驱动配套、PCIe 模式和首轮联机验证 |

文档按交付版本维护。新增机型时沿用通用手册，在兼容矩阵增加已验证组合，并通过机型接入页记录连接方式、通道映射和时钟差异。
