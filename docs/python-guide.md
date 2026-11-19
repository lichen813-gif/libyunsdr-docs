# libyunsdr Python 开发指南

本指南以 `libyunsdr-26-01-00.1.zip` 为基线，包内 SDK 版本为 **`26.00.00.1`**。使用 SWIG 绑定 `yunsdr_ss_api` 配置设备和传输 IQ16 数据。安装设备驱动、核对固件及选择 `pciex:`、`pcies:` 或 `usb3:` 时，参照 [C API 手册的驱动与构建章节](c-api.md#2-下载构建与安装)。

[构建与导入](#2-构建与导入-python-绑定) · [首帧接收](#4-接收首帧-iq) · [射频配置](#5-配置射频与时间戳) · [循环发射](#6-生成波形与循环发射) · [数据分析和校准](#7-分析数据与相位校准) · [排查问题](#8-常见问题)

## 1. 获取 SDK 和例程

| 资料 | 内容与用途 |
| --- | --- |
| `libyunsdr-26-01-00.1.zip` | 交付源码、C 库构建工程、SWIG 接口文件 `swig/yunsdr_ss_api.i` 和 `swig/python/yunsdr_ss/example/` 下的基础脚本。ZIP 哈希见 [交付版本核对](c-api.md#1-交付版本核对) |
| `python_20260213.7z` | 另配的 Python 参考例程包，配套说明列有循环收发、单独发送、OFDM 和相位校准脚本。通过产品资料下载时，在 **Python API基础和基础程序** 中选择此包并按提示填写产品编号。该参考包未随上述 ZIP 交付，文件内容及与新版绑定的匹配关系需根据实际配套资料核对 |

解压 SDK 后，在包含 `CMakeLists.txt` 的根目录执行构建。按当前解释器、CPU 架构和同版源码重新生成绑定。SDK ZIP 中 `swig/python/yunsdr_ss/old/` 内的二进制和脚本属于历史例程，下面使用新生成的产物。

## 2. 构建与导入 Python 绑定

测试基线为 Windows x64、Visual Studio 2019、CPython 3.11.9 和 SWIG 4.2.1。Python 绑定由 `ENABLE_PYTHON=ON` 和 `BUILD_PYTHON=ON` 启用。仅使用 Python 接口时，`ENABLE_EXAMPLE` 和 `ENABLE_COMLABS` 可以保持关闭。应使用构建时选中的 Python 解释器导入生成的扩展。

### 2.1 Windows x64

1. 安装 x64 的 [Python 3.11.9](https://www.python.org/downloads/release/python-3119/)、Visual Studio 2019 C++ 桌面开发工具及 CMake 3.16 或更新版本。解压 [SWIG 4.2.1](https://github.com/swig/swig/releases/tag/v4.2.1) 的 Windows 包，确保 PowerShell 能找到 `swig.exe`。在 PowerShell 检查 `py -3.11 --version` 和 `swig -version`。
2. 从 SDK 源码根目录创建虚拟环境。下面用 `.venv-sdk` 运行脚本并生成配套扩展。

   ```powershell
   py -3.11 -m venv .venv-sdk
   $py = (Resolve-Path .\.venv-sdk\Scripts\python.exe).Path
   & $py -m pip install numpy
   & $py -c "import struct,sys; print(sys.version); print(struct.calcsize('P') * 8)"
   ```

3. 配置并编译 Release。提前创建 `sdk-win64-python\bin`，因为本包的构建规则会将 Python 文件复制到该目录。检查 CMake 输出中选择的 Python 解释器、开发组件和 SWIG 路径。

   ```powershell
   $swig = (Get-Command swig.exe).Source
   $prefix = Join-Path (Get-Location).Path "sdk-win64-python"
   New-Item -ItemType Directory -Force -Path "$prefix\bin" | Out-Null
   cmake -S . -B build-win64-python -G "Visual Studio 16 2019" -A x64 `
     -DCMAKE_BUILD_TYPE=Release "-DCMAKE_INSTALL_PREFIX=$prefix" `
     "-DPython_EXECUTABLE=$py" "-DSWIG_EXECUTABLE=$swig" `
     -DENABLE_PYTHON=ON -DBUILD_PYTHON=ON -DENABLE_EXAMPLE=OFF -DENABLE_COMLABS=OFF `
     -DENABLE_PCIEX=ON -DENABLE_PCIES=ON -DENABLE_PCIE=OFF -DENABLE_SFP=OFF -DENABLE_USB3=OFF
   cmake --build build-win64-python --config Release --parallel 4
   cmake --install build-win64-python --config Release
   Get-ChildItem -LiteralPath "$prefix\bin"
   $env:PYTHONPATH = "$prefix\bin"
   & $py -c "import yunsdr_ss_api as api, _yunsdr_ss_api as ext; print(api.__file__); print(ext.__file__)"
   ```

4. 检查 `bin` 下的 `yunsdr_ss_api.py`、`_yunsdr_ss_api.pyd` 和 `libyunsdr_ss.dll`。导入结果打印的两个路径均应指向本次安装目录。新开终端时，从源码根目录重新设置上述 `$py`、`$prefix` 和 `PYTHONPATH`。

**加入 USB3。** 使用独立的 `build-win64-python-usb` 和 `sdk-win64-python-usb`。将上面的 `ENABLE_USB3` 改为 `ON`，按 [Windows USB3 构建](c-api.md#233-加入-usb3-支持) 指定 `LIBUSB_HEADER_FILE` 和 `usb_LIBRARY`。安装后将 `lib\libusb-1.0.dll` 复制到 `bin`，与 `libyunsdr_ss.dll` 和 `_yunsdr_ss_api.pyd` 放在一起。更新 `PYTHONPATH`，重做导入检查。

   ```powershell
   $prefix = Join-Path (Get-Location).Path "sdk-win64-python-usb"
   Copy-Item -LiteralPath "$prefix\lib\libusb-1.0.dll" -Destination "$prefix\bin"
   $env:PYTHONPATH = "$prefix\bin"
   & $py -c "import yunsdr_ss_api as api; print(api.__file__)"
   ```

### 2.2 Linux x86-64 与 AArch64

以下以 Debian/Ubuntu 为例。在 SDK 源码根目录运行，用系统 Python 或已安装的 CPython 3.11 创建虚拟环境。使用另一 Python 版本时，开发头文件和库也必须与所选解释器匹配。AArch64 按主机架构选择配套驱动，本包 CMake 会在该架构优先查找 `lib/ARM64/` 的静态依赖。

```bash
sudo apt install build-essential cmake swig python3 python3-dev python3-venv
python3 -m venv .venv-sdk
py="$PWD/.venv-sdk/bin/python"
"$py" -m pip install numpy
cmake -S . -B build-linux-python -DCMAKE_BUILD_TYPE=Release \
  -DPython_EXECUTABLE="$py" -DENABLE_PYTHON=ON -DBUILD_PYTHON=ON \
  -DENABLE_EXAMPLE=OFF -DENABLE_COMLABS=OFF -DENABLE_USB3=OFF \
  -DENABLE_PCIEX=ON -DENABLE_PCIES=ON -DENABLE_PCIE=OFF -DENABLE_SFP=OFF
cmake --build build-linux-python --parallel 4
export PYTHONPATH="$PWD/build-linux-python/lib"
export LD_LIBRARY_PATH="$PWD/build-linux-python/lib"
ldd build-linux-python/lib/_yunsdr_ss_api.so
"$py" -c 'import yunsdr_ss_api as api, _yunsdr_ss_api as ext; print(api.__file__); print(ext.__file__)'
```

本包将 `yunsdr_ss_api.py` 和 `_yunsdr_ss_api.so` 复制到 `build-linux-python/lib/`。应用可以直接使用此目录的 Python 绑定和 `libyunsdr_ss.so`，`ldd` 应解析到这一构建的 C 库及所需的系统依赖。USB3 构建先安装 `libusb-1.0-0-dev` 与 `pkg-config`，把 `ENABLE_USB3` 设为 `ON`，使用独立构建目录，并同步替换三个路径。Linux 编译和联机状态见 [兼容矩阵](compatibility.md)。

### 2.3 Spyder 与依赖

命令行即可运行本指南。使用 Spyder 时，在「首选项 → Python 解释器」选择 `.venv-sdk` 的 Python。按 Spyder 启动控制台时提示的版本范围，用同一解释器安装 `spyder-kernels`，再重启 IPython 内核。需要交互式图窗时，在 IPython 控制台的绘图设置里选择可用的 Qt 后端。Windows 可从设置好 `PYTHONPATH` 的终端启动 Spyder，Linux 同时保留 `LD_LIBRARY_PATH`，再在控制台运行 `import yunsdr_ss_api` 确认导入路径。[Spyder 环境配置说明](https://docs.spyder-ide.org/current/faq.html)

本指南的接收和波形生成示例只需 NumPy。另配参考包的波形分析与绘图脚本按用途安装 `matplotlib scipy PyQt5 ModulationPy`，例如 Windows 运行 `& $py -m pip install matplotlib scipy PyQt5 ModulationPy`，Linux 运行 `"$py" -m pip install matplotlib scipy PyQt5 ModulationPy`。

## 3. Python 调用约定

| 调用 | 行为 |
| --- | --- |
| `yunsdr_open_device(url)` / `yunsdr_close_device(dev)` | 打开成功返回设备句柄，失败返回 `None`。应用在 `finally` 中关闭有效句柄 |
| `yunsdr_get_model_version(dev)`、`yunsdr_get_firmware_version(dev)` | 解包为 `rc, value`。固件值的低 16 位是 FPGA，高 16 位是 ARM |
| `yunsdr_read_timestamp(dev, dev_id)` | 解包为 `rc, tick`。时间基准由当前设备配置决定 |
| `yunsdr_read_samples_multiport_Python(dev, buf, count, mask, 0)` | 解包为 `nread, tick`，I/Q 写入 `buf`。最后一个参数是时间戳的输入初值，常规接收用 `0` |
| `yunsdr_write_samples_multiport_Python(dev, buf, count, mask, tick, flags)` | 返回写入点数或负值，普通示例使用 `flags=0`。使用定时发射时，`tick` 取设备域的调度时间 |

查询接口有独立的 SWIG 签名，例如该版的 `yunsdr_get_device_configuration(dev, OUTPUT)` 仍需要 `DEV_CFG` 指针参数。以本次生成的 `yunsdr_ss_api.py` 的声明为准。频率、采样率、掩码和样点数传普通 Python `int`，例如 `int(30.72e6)`。

普通 IQ16 缓冲区使用 C 连续的 `np.int16` 数组，形状为 `(K, N, 2)`，其中 `K` 是掩码置位数，`N` 是传给 API 的每路复数样点数。最后一维依次是 I、Q。通道按掩码位升序紧密排列，例如 `0x5` 对应本地第 1、3 路，数组的第 0、1 行分别对应这两路。每路需要 `4N` 字节。RX 数组必须可写。用户需检查缓冲区长度、连续性和格式，当前包装层将 Python 缓冲区的地址直接传给 C。

本版 buffer typemap 取得 `Py_buffer` 后未调用 `PyBuffer_Release`。下面的首帧及有限次收发用于上手验证。需要长期连续采集时，先在对应 SDK 版本处理包装层的缓冲区生命周期问题并验证内存占用。

## 4. 接收首帧 IQ

先用 `-h` 或导入检查验证主机环境，再连接设备。下面的程序读取型号和固件，沿用设备已经配置好的射频参数，通过普通多路 API 采集本地第 1 路。将代码保存为 `rx_once.py`。初次运行用已知输入信号，`--count` 采用设备支持的每路块长。

```python
import argparse
from pathlib import Path

import numpy as np
import yunsdr_ss_api as api


def check(rc, name):
    if rc < 0:
        raise RuntimeError(f"{name} failed: {rc}")


def main():
    parser = argparse.ArgumentParser(description="Read one IQ16 block from RX1.")
    parser.add_argument("url", help="e.g. pciex:0")
    parser.add_argument("--count", type=int, default=30720)
    parser.add_argument("--output", type=Path, default=Path("rx_ch1.iq16"))
    args = parser.parse_args()
    backend, separator, address = args.url.partition(":")
    if (
        not separator
        or not address
        or backend not in {"pciex", "pcies", "usb3"}
        or len(args.url.encode("utf-8")) >= 128
    ):
        parser.error("use one valid SDK device URL")
    if not 0 < args.count <= 0x7FFFFFFF:
        parser.error("--count must be a positive 32-bit signed integer")

    dev = api.yunsdr_open_device(args.url)
    if dev is None:
        raise RuntimeError(f"cannot open {args.url}")
    try:
        rc, model = api.yunsdr_get_model_version(dev)
        check(rc, "read model")
        rc, version = api.yunsdr_get_firmware_version(dev)
        check(rc, "read firmware")
        print(f"model=0x{model:08x}, firmware=0x{version:08x}")

        iq = np.zeros((1, args.count, 2), dtype=np.int16)
        nread, tick = api.yunsdr_read_samples_multiport_Python(
            dev, iq, args.count, 0x1, 0
        )
        if nread != args.count:
            raise RuntimeError(f"RX returned {nread}, expected {args.count}")
        with args.output.open("xb") as output:
            iq.astype("<i2", copy=False).tofile(output)
        print(f"samples={nread}, tick={tick}, file={args.output}")
    finally:
        api.yunsdr_close_device(dev)


if __name__ == "__main__":
    main()
```

在步骤 2 设置好路径的终端运行。按机型改用 `pcies:0.0` 或 `usb3:0` 时，确认加载的是已启用该后端的构建。若输出文件已存在，改用另一个文件名或先移走旧文件。

```powershell
& $py .\rx_once.py 'pciex:0' --count 30720 --output rx_ch1.iq16
(Get-Item -LiteralPath .\rx_ch1.iq16).Length
```

```bash
"$py" ./rx_once.py 'pciex:0' --count 30720 --output rx_ch1.iq16
wc -c < rx_ch1.iq16
```

成功时文件为 `30720 × 4 = 122880` 字节。用下面的方式重建复数样点并核对已知输入。文件为小端的 `I0 Q0 I1 Q1 …`，采样率、频点和 tick 需随测试记录保存。

```python
import numpy as np

raw = np.fromfile("rx_ch1.iq16", dtype="<i2").reshape(-1, 2)
signal = raw[:, 0].astype(np.float32) + 1j * raw[:, 1].astype(np.float32)
```

## 5. 配置射频与时间戳

在接收代码的 `try` 内、申请缓冲区之前，可加上如下配置。`rf_id=0` 是例程的首组 RF 控制目标，机型的 RF 组、组内通道和掩码对应关系见配套接口说明。频率、采样率和增益为示例值，运行前替换为设备支持的配置。

```python
rf_id = 0
sample_rate = 30_720_000
center_hz = 2_400_000_000
check(api.yunsdr_set_rx_sampling_freq(dev, rf_id, sample_rate), "set RX sample rate")
check(api.yunsdr_set_rx_lo_freq(dev, rf_id, center_hz), "set RX LO")
check(api.yunsdr_set_rx1_gain_control_mode(dev, rf_id, api.RF_GAIN_MGC), "set RX gain mode")
check(api.yunsdr_set_rx1_rf_gain(dev, rf_id, 5), "set RX1 gain")
rc, actual_hz = api.yunsdr_get_rx_lo_freq(dev, rf_id)
check(rc, "read RX LO")
print(f"RX LO: {actual_hz} Hz")
```

- 发射路径用 `yunsdr_set_tx_sampling_freq`、`yunsdr_set_tx_lo_freq` 和 `yunsdr_set_tx1_attenuation` 对应配置。衰减单位为 mdB，例如 `30000` 等于 30 dB。使用共享 LO、共用采样率或可调带宽的机型时，按实际 RF 控制组写入并读取设备支持的查询项。
- 如果固件及后端支持用连接串调整接收帧长，PCIEX 示例可写成 `pciex:0,nsamples_recv_frame:7680`。此时 `--count 30720` 是每路四帧的数据量。以实际后端允许的帧长及对齐要求为准。
- 使用定时收发时，按设备配置参考时钟和 PPS，再调用 `yunsdr_enable_timestamp`，用 `yunsdr_read_timestamp` 获取当前 tick。转换函数接受计数器速率（tick/s）。例如计数器速率为 30.72 MHz 时，`yunsdr_timeNsToTicks(500000, 30720000.0)` 返回 `15360`。多通道或多设备需实测计数器是否同起点，并通过回环确认触发与链路延迟。
- RFSoC 的 ADC 校准、ADRV90xx 快锁及设备内数字回环属于机型扩展。参考脚本中的扩展命令编号、参数、RF 缓存大小和最大块长，须核对对应固件后再使用。

## 6. 生成波形与循环发射

参考例程的工作顺序是准备逐通道波形、向支持循环发送的设备写入一个块、启动重复发射、用 RX 掩码取样，再停止发射并释放设备。初次联机用单通道和带衰减器的有线回环，按设备允许的功率及输入范围配置。设备须支持所选的循环模式。

### 6.1 准备 IQ16 波形

下例为相位连续重复的单音，一个块内含整数个周期。数组按 `(通道, 样点, I/Q)` 排列。自研 OFDM 输入或文件输入也按此布局准备，各通道的 `count` 相同，短通道在构建缓冲区时补零。

```python
count = 30720
period = 40
phase = 2.0 * np.pi * np.arange(count) / period
tx = np.empty((1, count, 2), dtype=np.int16)
tx[0, :, 0] = np.rint(12000 * np.cos(phase)).astype(np.int16)
tx[0, :, 1] = np.rint(12000 * np.sin(phase)).astype(np.int16)
```

归一化复数输入应先按所需幅度缩放、取整并裁剪到 `[-32768, 32767]`，再转换成 `int16`。文件输入使用 `np.fromfile(path, dtype="<i2")`，检查元素数可被 2 整除且每路点数符合配置。

### 6.2 启动周期发送并读取

在有效句柄和射频已配置、输入和输出均准备好的情况下使用下面的调用。本包随附的 `get_dataset.py` 采用 `0 → 1 → 写入 → 3` 的循环模式顺序。对应固件要求的预读、时间戳复位方式或缓存配置按实际设备流程执行。PCIES 与 PCIEX 的处理可以不同。检查实际写入点数后再启动。

```python
dev_id = 0
tx_mask = 0x1
rx_mask = 0x1
rx = np.zeros((1, count, 2), dtype=np.int16)
check(api.yunsdr_tx_cyclic_enable(dev, dev_id, 0), "stop existing loop")
try:
    check(api.yunsdr_tx_cyclic_enable(dev, dev_id, 1), "prepare loop")
    nwrite = api.yunsdr_write_samples_multiport_Python(
        dev, tx, count, tx_mask, 0, 0
    )
    if nwrite != count:
        raise RuntimeError(f"TX returned {nwrite}, expected {count}")
    check(api.yunsdr_tx_cyclic_enable(dev, dev_id, 3), "start loop")
    nread, tick = api.yunsdr_read_samples_multiport_Python(
        dev, rx, count, rx_mask, 0
    )
    if nread != count:
        raise RuntimeError(f"RX returned {nread}, expected {count}")
    print(f"received {nread} IQ samples at tick {tick}")
finally:
    stop_rc = api.yunsdr_tx_cyclic_enable(dev, dev_id, 0)
    if stop_rc < 0:
        print(f"stop loop failed: {stop_rc}")
```

结合上一节的外层 `finally` 关闭设备。普通接收返回的 tick 用于记录块起点。即使计数器已同步，滤波、转换和射频链路也可能带来固定时延，需用已知波形做回环测量。

## 7. 分析数据与相位校准

| 任务 | 操作 |
| --- | --- |
| 单音检查 | 从 IQ16 得到复数数组，逐路检查时域、散点及频谱。FFT 频轴由实际采样率生成，转换为绝对频率时再加 RX 中心频率。记录主峰位置和相对幅度 |
| OFDM 环境验证 | 另配参考包中进入含 `test_cofdm.py`、`cofdm_tx.py`、`cofdm_rx.py` 的目录，安装其依赖，再用当前 Python 运行 `test_cofdm.py`。验证调制、同步及解调输出，检查频偏与 EVM 是否为有限值。该离线检查无需打开设备 |
| 宽带收发 | 在参考包的顶层脚本选择宽带波形，设置相同的 FFT 长度、子载波数、调制阶数及采样率，再核对捕获过渡段、频率估计和 EVM。信号生成与解调参数随参考包而定 |
| 相位校准 | 将同一个已知单音经功分器送入待校 RX 通道。在目标频点测量各路相对参考通道的相位差，按该机型的系数约定生成校准文件。写入后重新采集同源信号验证相位差。每个使用频点分别校准，断电后的保留方式随固件确认 |

### 7.1 检查单音与通道相位

下例用 6.2 中的 `rx` 和设定的接收采样率 `sample_rate` 估计基带主峰。在各通道同步接收同一个离散单音时，取参考通道的主峰所在 FFT 格点，比较各路的幅度和相位。若信号很弱或存在多个强分量，先核实所取 FFT 格点。

```python
samples = rx[:, :, 0].astype(np.float64) + 1j * rx[:, :, 1].astype(np.float64)
window = np.hanning(samples.shape[1])
spectrum = np.fft.fft(samples * window, axis=1)
peak_bin = int(np.argmax(np.abs(spectrum[0])))
offset_hz = np.fft.fftfreq(samples.shape[1], d=1.0 / sample_rate)[peak_bin]
peak = spectrum[:, peak_bin]
scale = abs(peak) / (window.sum() * 32768)
amplitude_dbfs = 20 * np.log10(np.maximum(scale, np.finfo(float).tiny))
phase_deg = np.angle(peak * np.conj(peak[0]), deg=True)
print(offset_hz, amplitude_dbfs, phase_deg)
```

### 7.2 运行配套例程

另配包的主要入口是 `yunsdr_tx_rxcyclic.py`（循环收发）、`yunsdr_tx.py`（独立发送）、`test_cofdm.py`（离线 OFDM）、`coeff_calc.py` / `config_coeff.py`（生成和应用相位补偿）、`disable_coeff.py`（关闭或清除补偿）及 `calib_from_gps.py`（设备支持时的参考校准）。先用对应参考包的模块和脚本组合运行。在需要使用本指南编译的 SWIG 绑定时，确认脚本目录中没有会覆盖导入路径的同名旧 `.py`、`.pyd` 或 `.so`，用 `api.__file__` 验证。

解压例程包并切换到包含 `test_cofdm.py` 的目录，保留第 2 节设定的 Python 环境及绝对路径变量。在 Windows 运行 `& $py .\test_cofdm.py`，在 Linux 运行 `"$py" ./test_cofdm.py`。离线验证通过后，再在顶层脚本中设置设备地址、射频类型、目标频点、采样率、每路块长和通道掩码，按同样方式运行所需的入口脚本。

使用配套脚本校准接收相位时，在循环收发例程中选择单音并将相位差保存到 `data/`。在 `coeff_calc.py` 设定同一频点，生成相应的补偿文件。随后在 `config_coeff.py` 选择该频点的两份系数文件并运行。重新收取同源单音，验证与参考通道的相位差，再把已验证的参数纳入应用启动流程。

交付 SDK 的 `get_txbuf.py`、`gen_tone_data.py` 和 `disp_rx_data.py` 可用来阅读基础波形与绘图流程。该 ZIP 内的 `get_rxbuf.py` 会删除当前目录的 `.dat`，对 `yunsdr_read_samples_multiport_Matlab` 的调用还少了本版生成接口要求的时间戳实参。使用本指南的首帧代码作为联机起点。

## 8. 常见问题

| 现象 | 排查方法 |
| --- | --- |
| `No module named 'yunsdr_ss_api'` | 检查 `PYTHONPATH` 和解释器。包装文件 `yunsdr_ss_api.py` 应与生成的扩展一同存在 |
| Windows `DLL load failed` 或 Linux `cannot open shared object file` | 检查 Python 版本和位数、生成扩展的路径与同版 `libyunsdr_ss`。USB3 检查 libusb 动态库。Linux 在测试终端用 `ldd` 核对 |
| `None` 句柄或设备打开失败 | 核对地址前缀、CMake 实际启用的后端、主机驱动和访问权限。先用 [基础 C 测试](c-api.md#43-基础测试程序) 验证同一设备 |
| 样点数异常、通道错位 | 检查掩码、数组的 `int16` 类型与连续性，以及每路点数。`0x5` 需要两个连续的通道块，按掩码置位顺序存放 |
| 频谱频点或相位不符 | 确认 RX 实际采样率、I/Q 顺序、中心频率、参考时钟和同源回环。检查信号是否过载或输入太弱 |
