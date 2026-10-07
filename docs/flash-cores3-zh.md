# M5Stack CoreS3 编译与烧录指南

本指南适用于 **M5Stack CoreS3（16 MB Flash、Quad PSRAM）**。示例使用 Windows PowerShell、ESP-IDF 5.5.2 和 esptool 5.x；其他系统可使用相同的烧录地址。烧录会覆盖当前固件，建议先备份。

## 1. 准备源码和 ESP-IDF

1. 安装 [ESP-IDF 5.5.2](https://docs.espressif.com/projects/esp-idf/en/v5.5.2/esp32s3/get-started/index.html) 和 Python 3.10 或更新版本。
2. 克隆本仓库，并在已启用 ESP-IDF 环境的终端中进入项目目录。Windows 上建议使用较短的绝对路径，例如 `D:\sc\Stackchan-HtSz`，以避开命令行长度限制。
3. 设置目标芯片：

   ```powershell
   idf.py set-target esp32s3
   ```

项目的 `sdkconfig.defaults` 已选中 `CONFIG_BOARD_TYPE_M5STACK_CORE_S3=y`。构建前还需核对下列两项。

### PSRAM 模式：必须是 Quad

CoreS3 使用 Quad PSRAM。当前仓库的 `sdkconfig.defaults` 选择 Quad，但 `sdkconfig.defaults.esp32s3` 选择 Octal，后者可能覆盖前者，导致启动日志出现 `octal_psram: PSRAM chip is not connected, or wrong PSRAM line mode` 并反复重启、黑屏。

在项目目录执行 `idf.py menuconfig`，搜索 `SPIRAM_MODE`，把 PSRAM 模式改为 **Quad** 并保存。随后检查最终的 `sdkconfig`：

```text
CONFIG_SPIRAM_MODE_QUAD=y
# CONFIG_SPIRAM_MODE_OCT is not set
```

如果重新执行 `set-target` 或删除 `sdkconfig`，请再次确认此设置。不要只检查 `sdkconfig.defaults`。

### OTA 服务地址

仓库当前的 `CONFIG_OTA_URL` 是 `YOUR_SERVER_IP` 占位地址。使用小智官方服务时，构建前在 `sdkconfig.defaults` 中将其设为：

```text
CONFIG_OTA_URL="https://api.tenclass.net/xiaozhi/ota/"
```

如果已有自建服务端，则填入自己的 OTA 地址。修改默认配置后，确认最终 `sdkconfig` 中的 URL 也正确。

## 2. 构建

```powershell
idf.py build
```

构建完成后检查 `build/flasher_args.json`；它是当前构建生成的烧录清单。本仓库 CoreS3 的 16 MB 分区方案需要以下六个镜像：

| 地址 | 构建输出 | 用途 |
| --- | --- | --- |
| `0x0` | `build/bootloader/bootloader.bin` | 启动程序 |
| `0x8000` | `build/partition_table/partition-table.bin` | 分区表 |
| `0xD000` | `build/ota_data_initial.bin` | OTA 初始状态 |
| `0x10000` | `build/srmodels/srmodels.bin` | 语音模型 |
| `0x410000` | `build/xiaozhi.bin` | 主程序 |
| `0xA10000` | `build/generated_assets.bin` | 界面资源 |

若烧录清单中的地址与此表不同，应以**当前构建**生成的 `build/flasher_args.json` 为准。

## 3. 确认设备并备份

用可传输数据的 USB 线连接 CoreS3，关闭其他占用串口的程序。查看串口：

```powershell
[System.IO.Ports.SerialPort]::GetPortNames()
```

下面以 `COM3` 为例；请替换成你的串口。ESP-IDF 环境已包含 esptool，也可另行执行 `python -m pip install esptool`。

```powershell
$Port = 'COM3'
python -m esptool --chip esp32s3 -p $Port flash-id
```

确认识别到 ESP32-S3 和 16 MB Flash。推荐在烧录前读取整片 Flash：

```powershell
python -m esptool --chip esp32s3 -p $Port -b 460800 read-flash 0x0 0x1000000 .\coreS3-original-backup.bin
```

备份应为 16,777,216 字节，可能含有 Wi-Fi 凭据，**不要提交到 GitHub**。

## 4. 烧录

在项目根目录运行：

```powershell
python -m esptool --chip esp32s3 -p $Port -b 460800 --before default-reset --after hard-reset write-flash --flash-mode dio --flash-freq 80m --flash-size 16MB `
  0x0      .\build\bootloader\bootloader.bin `
  0x8000   .\build\partition_table\partition-table.bin `
  0xD000   .\build\ota_data_initial.bin `
  0x10000  .\build\srmodels\srmodels.bin `
  0x410000 .\build\xiaozhi.bin `
  0xA10000 .\build\generated_assets.bin
```

PowerShell 的续行反引号 `` ` `` 后不能有空格。烧录工具应对写入数据报告 `Hash of data verified.`；无需先运行 `erase-flash`。若 `Failed to connect`，关闭串口监视器、核对端口；CoreS3 可按住侧面复位键约 2 秒，看到内部绿色指示灯亮起后松开，进入下载模式再重试。

## 5. 启动检查

烧录完成后短按复位键，或重新插拔 USB。正常情况下屏幕出现 Stack-chan 表情或主界面。如果仍然黑屏，请先核对最终 `sdkconfig` 中的 Quad PSRAM 设置、六个镜像及地址。若串口日志报 `octal_psram`，重新设置 Quad 并完整重编、重烧。

联网后设备会访问所配置的 OTA 服务。使用官方服务时，按屏幕提示配网和激活；唤醒词以当前构建的配置为准。

## 参考

- [Espressif：安装 esptool](https://docs.espressif.com/projects/esptool/en/latest/esp32/installation.html)
- [Espressif：ESP32-S3 烧录说明](https://docs.espressif.com/projects/esptool/en/latest/esp32s3/esptool/flashing-firmware.html)
- [Espressif：读取 Flash](https://docs.espressif.com/projects/esptool/en/latest/esp32/esptool/basic-commands.html)
- [M5Stack：CoreS3 进入下载模式](https://docs.m5stack.com/en/guide/cores3/restore_factory)
