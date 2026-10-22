# 红米电视 ASTP0 适配：踩坑与验证记录

> 本文档记录的是在**这台**电视（ASTP0 / mulan，国内版 MIUI TV，Android 9 / SDK 28）上实测的结论。
> 别的机型固件可能完全不同，请以自己机器上的实际输出为准。

## 一、备份

裁剪之前先留档，后面每一步都拿它对照：

```bash
tools/venv/bin/python tools/check.py --backup
```

留档内容里包含机型信息、完整包列表和几个关键 `dumpsys` 的输出。

> **注意：** 这些 dump 里有设备序列号、MAC、`android_id` 等标识，**不要提交到公开仓库**。
> 上游的 `.gitignore` 也明确把这类 dump 排除在外（`backup/`）。本仓库不包含任何备份文件。

## 二、禁用包

```bash
TV_HOST=<电视IP> bash tools/disable.sh debloat/astp0-cn.txt
```

清单来自「本机实际包列表」，而不是照抄上游 `examples/xiaomi-mitv-astp0/debloat.txt`
——那份是**越南版机器**的定制清单，里面大量包在本机根本不存在，而且按越南用户偏好把
小爱同学和投屏都禁掉了，直接套用会禁错。

### 还原

```bash
TV_HOST=<电视IP> bash tools/restore.sh debloat/astp0-cn.txt com.mitv.tvhome
```

`pm disable-user` 的状态写在 `packages.xml` 里，所以重启不会丢。

### 反复出现的坑：`com.xiaomi.wakeupservice`

**每次重启，这个包都会被系统重新启用。** 重启后要再禁一次：

```bash
TV_HOST=<电视IP> bash tools/disable.sh <(printf 'com.xiaomi.wakeupservice\n')
```

它常驻监听小爱唤醒词，约占 30% CPU。遥控器上的语音键由 `com.xiaomi.voicecontrol` 处理，
禁掉 wakeupservice 不影响遥控器按键。

## 三、换桌面（`com.gathin.tvhome`）

上游 `bash launcher/build-and-install.sh` 一条命令完成：构建 → 推送 → 安装 → 校验 HOME。

```bash
TV_HOST=<电视IP> bash launcher/build-and-install.sh
# → Home now resolves to com.gathin.tvhome/.MainActivity
```

### 坑 1：`set-home-activity` 静默失败

```bash
adb shell cmd package set-home-activity com.gathin.tvhome/.MainActivity
adb shell dumpsys package preferred-xml     # 一直是空的
```

这版固件上这个命令**不写入 `preferred-activities`**。所以必须额外把旧桌面禁用掉：

```bash
adb shell pm disable-user --user 0 com.mitv.tvhome
adb shell cmd package resolve-activity --brief \
  -a android.intent.action.MAIN -c android.intent.category.HOME
# → com.gathin.tvhome/.MainActivity
```

`build-and-install.sh` 在 `STOCK_LAUNCHER=none` 时会自动做这一步。

### 坑 2：桌面图标只认 LEANBACK 应用

底栏和图标来源只有 `Intent.CATEGORY_LEANBACK_LAUNCHER` 一种。
普通 `LAUNCHER` 应用（比如小爱同学）**不会**出现在底栏，只能通过「应用库」按钮找到。
所以底栏只能放本身声明了 LEANBACK 的应用。

### 坑 3：桌面配置文件的位置

新桌面从外部目录读主题配置：

```
/sdcard/Android/data/com.gathin.tvhome/files/config.json
```

重启后配置文件和壁纸都在，说明写在外置存储上是稳的。

## 四、主题

```bash
TV_HOST=<电视IP> bash tools/theme.sh themes-redmi
```

`theme.sh` 推完配置后会 force-stop + 重启 Home，让桌面重新读配置。
配置项的完整说明见上游 `themes/README.md`。

## 五、ADB 的两个坑

### 坑 1：重启后 5555 端口就没了

```bash
adb connect <电视IP>:5555     # Connection refused
settings get global adb_enabled   # 还是 1
```

设置项还在，说明**不是设置被改回去了**，而是开机时有组件杀了 adbd 的 TCP 监听。
在 Mac 侧改 `adb_enabled` 的值**没用**，必须拿**遥控器**去「设置 → 账号与安全 → ADB调试」
重新打开开关。

> 鼠标点击那个开关不会弹出授权对话框，**遥控器是必须的**。

### 坑 2：两个 adb 客户端抢 5555

电视的 adbd 一次只服务一个 TCP 客户端。Homebrew 的 `adb` 和仓库自带的 `tools/tv.py`
（`adb_shell` 实现的 Python 客户端）会互相把对方踢掉。用 Python 客户端之前先：

```bash
adb kill-server
```

## 六、验证清单

改完之后，逐条核对：

| 检查项 | 命令 | 期望 |
| --- | --- | --- |
| HOME 归属 | `cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME` | `com.gathin.tvhome/.MainActivity` |
| 禁用是否生效 | `pm list packages -d` | 清单里的包都在，且 `disabled=true` |
| 禁用是否被系统翻回 | 同上，重启后再看一遍 | `com.xiaomi.wakeupservice` **会被翻回**，需重禁 |
| 网络是否还在 | 电视能连 WiFi、能播视频 | setupwraith 没被禁 |
| 信号是否正常 | 接 HDMI 源有画面 | `com.droidlogic.tvinput` 没被禁 |

## 七、没有 ADB 时怎么往电视上推东西

万一 ADB 关了、但你想在电视上弹点东西（比如一张图），还有一条路：
电视自带的 `hyperDLNA`（UPnP MediaRenderer）在 **TCP 49152** 上接受 UPnP AVTransport 的 SOAP 请求：

```
POST /smartshare/render/_urn:schemas-upnp-org:service:AVTransport_control
SOAPACTION: "urn:schemas-upnp-org:service:AVTransport:1#SetAVTransportURI"
```

InstanceID 0，先 `SetAVTransportURI` 再 `Play`。用 `protocolInfo="http-get:*:image/jpeg:*"`、
`upnp:class` 填 `object.item.imageItem.photo`，电视会自己去 HTTP 服务器取这张 JPEG 并全屏显示。

- 本机要有一个局域网可达的 HTTP 服务器；
- `text/html` 会被取到但**不渲染**；
- 渲染器**不执行 JavaScript**，所以这条路只能「输出」，不能「交互」。

其它端口：8899（心跳，不派发按键）、10003（无响应）、47000（AirTunes / AirPlay 接收）。
