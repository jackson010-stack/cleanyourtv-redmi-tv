# CleanYourTV 红米电视适配（ASTP0 / mulan）

这是 [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) 在**小米 / 红米电视 ASTP0
（代号 mulan，国内版 MIUI TV 固件）**上的适配记录：一份按本机实测裁剪的禁用清单、一份底栏主题配置，
以及把这套方案真正跑通所踩到的坑。

上游仓库解决的是「电视桌面和预装广告」这个通用问题，但它按机型/地区给的清单大多不是为国内版
ASTP0 写的；这个仓库只保留**这台机器上真正有效**的部分。

- 设备：红米电视 ASTP0（mulan），Android 9 / SDK 28，固件 `26.3.17.OS3.0.101.0.PSTAATV`
- 内容：`debloat/`（禁用清单）、`themes-redmi/`（底栏主题）、`tools/`（上游 CLI）、`docs/`（踩坑记录）
- 授权：MIT

---

## 这套方案做了什么

1. **备份** —— 把机型、包列表、`dumpsys` 结果留档，作为回滚依据
2. **裁剪** —— 用 `pm disable-user --user 0` 禁掉 16 个广告 / 埋点 / 管家 / 旧桌面相关的包
3. **换桌面** —— 装第三方桌面 `com.gathin.tvhome` 并把 HOME 键让给它
4. **配主题** —— 底栏四个图标、隐藏相册、显示使用统计、10 分钟息屏进时钟屏保
5. **装影视客户端** —— 见 [librettv-tv](https://github.com/jackson010-stack/librettv-tv)

> 第 1、5 步的具体内容不在本仓库；本仓库聚焦第 2~4 步。

## 快速开始

### 0. 准备 ADB

```bash
python3 -m venv tools/venv
tools/venv/bin/pip install -r tools/requirements.txt
```

电视上要走「设置 → 账号与安全 → ADB调试」，**必须用遥控器点开关**（鼠标点击不会弹出授权框）。
打开后：

```bash
export TV_HOST=<电视IP>
tools/venv/bin/python tools/tv.py --probe
```

### 1. 备份

```bash
tools/venv/bin/python tools/check.py --backup    # 机型 / 包列表 / 关键 dumpsys
```

### 2. 裁剪

```bash
TV_HOST=$TV_HOST tools/disable.sh debloat/astp0-cn.txt
```

全部是 **disable-user**（禁用，不是卸载），随时可以一条命令还原：

```bash
TV_HOST=$TV_HOST tools/restore.sh debloat/astp0-cn.txt com.mitv.tvhome
```

### 3. 换桌面

桌面包 `com.gathin.tvhome` 的源码在上游仓库的 `launcher/` 目录（内置 158 个机型的主题与壁纸）：

```bash
git clone https://github.com/Gathin23/cleanyourtv.git
cd cleanyourtv
launcher/build-and-install.sh          # 构建 → 推送 → 安装 → 校验 HOME
```

> **本机实测：** `set-home-activity` 在这版固件上**静默失败**
> （`dumpsys package preferred-xml` 一直是空的），所以除了设置 HOME，
> 还必须 `pm disable-user --user 0 com.mitv.tvhome` 把旧桌面禁用掉，Home 键才会真正落到新桌面。
> 这两步 `build-and-install.sh` 都会做。

### 4. 配主题

```bash
TV_HOST=$TV_HOST tools/theme.sh themes-redmi
```

`themes-redmi/config.json` 的含义：

| 键 | 值 | 说明 |
| --- | --- | --- |
| `dock_apps` | 云视听极光 / 酷喵 / 云视听小电视 / 网易云音乐 | 底栏四个图标（都是 LEANBACK 应用） |
| `fill_dock` | `false` | 底栏不放满，空位留白 |
| `hidden_apps` | `com.mitv.gallery`、`com.xiaomi.tv.gallery` | 从应用库里隐藏系统相册 |
| `rows_below_dock` | `1` | 底栏上面留一行 |
| `app_library` / `library_button` | `true` | 保留应用库和它的入口按钮 |
| `show_stats` | `true` | 桌面显示使用统计 |
| `home_idle_clock_minutes` | `10` | 息屏 10 分钟后切到时钟屏保 |

---

## 裁剪清单说明

`debloat/astp0-cn.txt` 共 16 个包，分四类：

| 类别 | 包 |
| --- | --- |
| 广告 / 购物 | `com.mitv.shoplugin`、`com.xm.webcontent`、`com.mitv.overseasservice` |
| 埋点 / 推送 / 云控 | `com.miui.tv.analytics`、`com.mitv.cloudcontrol`、`com.xiaomi.mitv.tvpush.tvpushservice` |
| 后台常驻 / 管家 | `com.xiaomi.mitv.tvmanager`、`com.xiaomi.mitv.healthdiagnosis`、`com.xiaomi.mitv.smartpanel`、`com.xiaomi.mitv.karaoke.service`、`com.xiaomi.gamecenter.sdk.service.mibox`、`com.xiaomi.wakeupservice`、`com.xiaomi.mitv.hyper.screensaver` |
| 系统更新 / 旧链路 / 旧桌面 | `com.xiaomi.mitv.upgrade`、`com.xiaomi.miplay`、`com.xiaomi.mi_connect_service`、`com.mitv.tvhome` |

禁用系统更新（`com.xiaomi.mitv.upgrade`）是有意为之：OTA 一旦跑起来，会把预装内容原样装回来，
前面禁掉的广告也就白禁了。

### 绝对不能动的包

| 包 | 禁掉的后果 |
| --- | --- |
| `com.xiaomi.setupwizard` | 开机向导持有已保存的 WiFi，禁掉后电视直接掉线，ADB 和手机遥控一起没了 |
| `com.droidlogic` / `com.droidlogic.tvinput` | HDMI / AV / 调谐器输入层，禁了黑屏无信号 |
| `com.sohu.inputmethod.sogou.tv` | TV 上唯一的输入法 |
| `com.xiaomi.mitv.settings` / `com.xiaomi.account` / `com.xiaomi.account.auth` | 系统设置、账号（商店和支付依赖） |
| `com.xiaomi.tvqs` / `com.xiaomi.mitv.systemui` / `com.xiaomi.mitv.appstore` | 搜索、系统 UI、应用商店 |
| `mitv.service` | 核心服务 |

---

## 踩坑记录

完整版见 [docs/redmi-tv-notes.md](docs/redmi-tv-notes.md)，这里是最要命的几条：

- **`set-home-activity` 在这版固件上不写入 `preferred-activities`。** 必须额外
  `pm disable-user --user 0 com.mitv.tvhome`，Home 键才会落到新桌面。
- **`com.xiaomi.wakeupservice` 每次开机会被系统重新启用。** 重启后要再禁一次；
  它的存在会吃掉约 30% CPU（遥控器语音键由 `com.xiaomi.voicecontrol` 处理，不受影响）。
- **电视每次重启都会关掉 5555 端口。** `settings get global adb_enabled` 仍然是 `1`，
  说明是开机时有组件杀了 adbd 的 TCP 监听，不是设置项的问题——在 Mac 侧改这个值没用，
  必须拿遥控器去设置里重新打开。
- **桌面图标只认 LEANBACK 应用。** `Intent.CATEGORY_LEANBACK_LAUNCHER` 是唯一来源，
  普通 `LAUNCHER` 应用（小爱同学之类）只能从应用库按钮进，别想放进底栏。
- **Homebrew 的 `adb` 和仓库自带的 Python 客户端会抢 5555。** 电视的 adbd 一次只服务一个 TCP 客户端，
  用 `tools/tv.py` 前先 `adb kill-server`。

---

## 目录结构

```
debloat/astp0-cn.txt      按本机实测裁剪的 16 个禁用包
themes-redmi/config.json  底栏 / 应用库 / 屏保主题配置
tools/                    上游的 CLI：disable.sh、restore.sh、theme.sh、tv.py、check.py、recovery.py
docs/redmi-tv-notes.md    完整的踩坑与验证记录
```

## 致谢

- [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) —— 电视端部署与裁剪方案、CLI 工具、
  第三方桌面 `com.gathin.tvhome`
- [jackson010-stack/librettv-tv](https://github.com/jackson010-stack/librettv-tv) —— 配套的原生影视客户端

## 许可证

MIT，见 [LICENSE](LICENSE)。`tools/` 目录来自上游 cleanyourtv，同样是 MIT 许可，
版权归 Gathin 所有。
