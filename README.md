# OneLine

**Practice English, one sentence at a time.**\
每次只练一句英语。

OneLine 是一个安卓口语练习工具。把平时想说的一句话说出来，手机给出一句符合语境的日常美语，自动读给你听。练习材料来自自己当下想表达的事。

## 已确定

- **项目与安卓应用名称**：OneLine。
- **后端名称**：OneLineServer。
- **开发平台**：Android。
- **核心体验**：一次一句，中文和英文共用同一个麦克风按钮。
- **设备形态**：先做普通安卓 App，跑通后让同一个 APK 也能作为默认桌面（Launcher），解锁后直接看到练习入口。

## 手机上的一次练习

1. **按住说，松开发送。** 说中文或英文，无需手动切换模式。
2. **看转写、听表达。** 屏幕显示「听到的原句」和「推荐说法」，手机自动播报推荐的英文。转写听错时，可以改原句并重新获取反馈。
3. **重听并自行跟读。** 点小喇叭重播同一段音频。用户可以口头跟读，App 不采集或处理跟读音频。
4. **练下一句。** 再次按住麦克风按钮，开始练习新的句子。收藏和回顾留作后续功能，先验证单句练习的体验。

做成 Launcher 后，按 Home 会回到 OneLine。是否需要禁止退出到其他应用，等实际用过再决定。

## 表达反馈规则

| 输入 | 输出 |
| --- | --- |
| 中文 | 理解原意，给一句符合场景的自然美语，并读出来。 |
| 英文，表达已自然 | 告知「这句可以直接用」，保留并读出原句。 |
| 英文，需要调整 | 给一句更合适的说法，读出来，再用一句中文解释最值得改的地方。 |

默认只给一个版本。跟朋友聊天、在工作中沟通等场景会影响措辞。只有语气差异会影响意思时，才补充另一个版本。

**中文示例**：「我刚才说到哪儿了？」→ **Where was I?**

**英文示例**：*I very like this book.* → **I really like this book.** 这里修饰动词 *like*，可以用 *really*。

## 设计草图（待确认）

- [一次练习的完整流程](design/practice-flow.svg)
- [安卓界面的六个操作状态](design/android-screens.svg)

## 技术方案与待验证事项

- **安卓端**：Kotlin + Jetpack Compose 做一屏，ViewModel + StateFlow 管理录音、等待、结果和播放状态。用 MediaRecorder 录单句，Media3 播放并重播返回的音频。对话流程跑通后，再给首页加上 Android Home 入口。
- **后端 OneLineServer**：Mac 上用 Python + FastAPI 直接接收局域网请求，完成语音处理。安卓端的服务地址可配置。
- **设备**：先用手边的安卓真机验证。ivmi 17 mini 是曾讨论的 4 英寸候选机型，尚未选定。购买前需确认具体 Android 版本、能否安装第三方 APK、能否设为默认桌面，以及重启后设置是否保留。
- **参考**：[Android 自定义桌面](https://developer.android.com/work/dpc/dedicated-devices/cookbook#custom-home) · [Compose 界面架构](https://developer.android.com/develop/ui/compose/architecture) · [FastAPI 文件上传](https://fastapi.tiangolo.com/tutorial/request-forms-and-files/)

## 手机如何连到 Mac

当前只验证局域网直连。手机用 Wi-Fi，Mac 可接同一 Wi-Fi 或接到同一局域网的有线网络。

```text
安卓 OneLine  ←→  Mac 上的 OneLineServer  ←→  OpenAI Realtime
```

1. OneLineServer 在 Mac 的局域网地址上提供服务。手机在浏览器打开 `http://<Mac 局域网 IP>:<端口>/healthz`，看到成功响应后，再在调试版中填同一个服务地址。
2. 手机松手后通过本次请求上传录音。Mac 使用已验证的连接调用 OpenAI Realtime，通过同一个请求返回原句转写、推荐表达和音频。OpenAI API 密钥留在 Mac。
3. 手机收到结果后自动播放，并缓存这一句供重听。Mac 离线或处理超时时，手机显示相应提示，可以重试。

## AI 后端如何处理一句话

Mac 将录音转成 24 kHz 单声道 PCM16，每句建立一次 Realtime 会话。关闭自动语音检测，用松手作为结束信号，依次追加录音、提交并请求回复。原句转写与回复独立完成，按输入项 ID 收集原句，同时收集推荐句的音频片段和对应文字。等两边完成并确认回复成功后，将音频片段拼成 WAV，连同文字返回。重听直接播放手机已收到的音频。 [OpenAI Docs · Realtime 会话](https://developers.openai.com/api/docs/guides/realtime-conversations) · [服务端事件](https://developers.openai.com/api/reference/resources/realtime/server-events)

## 实施方案

### 代码和部署位置

| 目录 | 运行位置 | 交付内容 |
| --- | --- | --- |
| `android-app/` | 安卓手机 | OneLine 单屏界面，长按录音、松手上传、显示结果、自动播放和重听。 |
| `server/worker/` | Mac | OneLineServer 的局域网 FastAPI 入口和语音处理逻辑。用 FFmpeg 转换音频并调用 Realtime。 |

### 一句练习的接口

| 接口 | 约定 |
| --- | --- |
| `GET /healthz` | 检查手机能否访问 Mac 上的服务。 |
| `POST /v1/practice` | 安卓发送 `multipart/form-data`，含 `request_id` 和 `audio`（AAC/M4A 录音）。成功返回 JSON，含同一个 `request_id`、`original_text`、`recommended_text` 和 `audio_wav_base64`。 |

原型一次处理一句。录音上限暂定 20 秒，上传上限暂定 2 MiB，处理超时暂定 30 秒，真机联调后再调。安卓保留失败的录音供用户点「重试」，正常完成或开始新一句时清理。

局域网 HTTP 仅用于安卓调试版，服务地址手工填写。Mac 从已经配置的私有文件读取 OpenAI 密钥。服务处理完释放本轮录音和结果，日志记录请求编号、错误类型与各段耗时。 [Android 网络安全配置](https://developer.android.com/privacy-and-security/security-config)

### 开发与验收顺序

1. **准备 Mac 后端。** 在本地实现上传和 WAV 返回，接入已验证的 Realtime，准备三类中英文样本。
2. **准备安卓调试版。** 做出可填 Mac 地址、按住录音、松手上传、显示结果并播放的调试 APK。
3. **两端做好后一起进行 Wi-Fi 真机联调。** 用 USB 给现有华为 Android 10 手机安装调试 APK，语音请求通过 Wi-Fi。先在手机上打开 Mac 的 `/healthz`，然后用调试版测试「我刚才说到哪儿了？」、`I very like this book.`、`Could you give me a minute?`，核对原句、推荐句和实际发音。重听播放同一份本地音频，口头跟读期间麦克风关闭。模拟一次连接失败并恢复，记录从松手到出声的耗时。
4. **完善第一版。** 练习流程稳定后加入转写编辑、简短中文纠错说明和 Launcher 入口。用重启后解锁及按 Home 两个场景验收默认桌面。

### 真机联调时确认

- **样机。** 用现有华为 Android 10，开启 USB 调试并授权 Mac 安装调试 APK。USB 用于安装，语音请求通过 Wi-Fi。
- **构建环境。** 复用 Brain Voice 本机已有的 Android SDK（API 35）。当前终端的 `JAVA_HOME` 指向 Java 8，安卓构建时为 OneLine 显式指定已安装的 JDK 17。
- **局域网。** 真机联调时让 Mac 保持唤醒，使用当前局域网 IP。手机浏览器访问 `/healthz` 成功后，继续测 App 的录音和播放。若访问失败，依次排查 IP 与端口、Mac 监听地址与防火墙、路由器设备隔离。

## 第一版验收

- 说一句中文，能看到并听到自然的美语表达。
- 说一句需要改的英文，能看到并听到改法，还能读到简短原因。原句自然时，保留原句。
- 能修正语音转写，重播推荐表达，再用麦克风开始新一句。重播和口头跟读期间，App 不开启麦克风。
- 用 Wi-Fi 直连 Mac 完成练习，记录松手到手机出声的时间。验证 Mac 断线时的提示及恢复后能否继续练习。
- 加入 Launcher 入口后，能设为默认桌面，重启解锁后显示 OneLine，按 Home 能返回。

当前状态：Mac 已用中英文音频验证 Realtime 的转写与语音回复。现有 JDK 和 Android SDK 可用于构建。安卓端和 Mac 服务待开发，Wi-Fi 连通性随原型集中验收。

## 远程连接备选

腾讯云中转暂缓，等明确有移动数据使用需求时再评估。可复用 Brain Voice 的腾讯云主机，为 OneLine 配独立 HTTPS 入口。Mac 主动建立 WSS 长连接，云端用请求编号转发录音和结果，安卓端继续使用同一接口。部署时单独验证蜂窝网络、Mac 断线重连及端到端耗时。
