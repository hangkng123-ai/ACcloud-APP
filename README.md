# AC 云验证 · v2.2.0（Android 客户端源码）

> AC Cloud · 专业卡密验证与云控平台 — 原生 Android 控制台

本仓库是 **AC 云验证 v2.2.0** 的 Android 客户端（控制台）源代码。它本身不是服务端，而是一套把「卡密验证 SaaS」全部能力搬到手机上的原生 App：账号体系、验证实例、卡密发放、云控变量、热更新、服务器/静态网站部署、无限邮箱、在线用户、工单、福利、对接文档，全部在一个 APK 内完成，**零 WebView、全原生渲染**。

- 包名：`com.ac.cloud`（debug 包为 `com.ac.cloud.debug`）
- 版本：`versionCode 220` / `versionName 2.2.0`
- 语言：Java + C++（JNI）
- 规模：57 个 Java 文件、51 个布局、116 个 drawable、10 个 RecyclerView 适配器，Java/C++/Gradle 约 1.08 万行

---

## 一、产品定位与核心概念

| 概念 | 说明 |
|---|---|
| **实例（Instance）** | 一个需要做授权验证的"应用"，拥有独立的 `instance_id`、AES 密钥、签名密钥和运行状态（running / stopped） |
| **卡密（Card）** | 发放给终端用户的授权凭证。分 `normal`（时长卡）与 `device`（设备卡，绑定设备数、可设激活方式） |
| **云控变量（Variable）** | 服务端下发的开关型配置，客户端 App 通过轮询接口实时拉取，用于远程控制功能开关 |
| **三件套密钥** | `aes_key`（Base64 的 32 字节）+ `secret_key`（签名密钥）+ `api_key`（账号级 API 对接码） |
| **签名（sign）** | `md5(aes_key + secret_key + api_key + timestamp)`，秒级时间戳 |
| **加密** | AES-256-CBC + PKCS7，IV 每次随机 16 字节，`data` 与 `iv` 均为 Base64 |
| **节点** | 欧洲节点 / 主节点两套服务端，应用内可切换；切换后强制重新登录，避免会话串号 |

---

## 二、功能全景

### 1. 账号与登录
- 账号密码登录（`/login` + `/api/user/info` 双步校验），支持"记住账号"。
- 邮箱验证码注册（`/api/app/register`）：AC ID 限 4–20 位字母数字下划线，密码≥6 位，60 秒重发倒计时。
- 邮箱验证码找回密码（`/reset_password`，解析服务端 HTML 的 `msg-success` / `msg-error` 提示）。
- **网页登录回跳**：拉起浏览器访问 `/app/login`，授权完成后通过自定义 Scheme `accloud://auth?token=xxx` 回到 App，用一次性 token 换会话（`/api/app/token_login`）。
- 登录页实时探活：OkHttp 请求首页测 HTTP 延迟 + C++ `tcpProbe()` 测纯 TCP 握手毫秒数，双指标展示"服务运行中 · xx ms · C++ xx ms"。
- Google 账号绑定（`/api/app/google_url` 先换取一次性绑定令牌，再跳浏览器授权，确保绑定到现有账号而非新建）。

### 2. 仪表盘（Dashboard）
- 问候区 + 认证徽标（昵称、UID、云端同步状态）。
- Hero 数据卡：实例总数 / 运行中 / 卡密总数。
- 统计卡：实例数、卡密数、API 调用总量（汇总各实例 `api_stats.total`）、在线人数（逐个实例调 `/api/online/status` 累加）、部署项目数、无限邮箱数。
- 延迟徽标（`ping` 接口耗时）。
- 全站公告（`/api/notice`）+ 实例公告（实例 `config.announcement_zh`），点击查看详情。
- 快捷操作宫格：新建应用、卡密、云控、在线用户、服务器部署、无限邮箱、云更新。
- "我的实例"前 3 条预览，带运行状态圆点。

### 3. 实例管理
- 列表：名称、ID、运行状态、API 统计；下拉刷新；空态引导。
- **创建实例**（底部弹层）：应用名称、`instance_id`，可选自定义 `secret_key` / `custom_api_key`；创建成功后弹窗展示自动生成的 AES 与签名密钥，一键复制。
- **实例详情**（弹层）：
  - 状态徽标 + 启动/停止（`/api/instances/{id}/toggle`）
  - 三件套密钥脱敏展示（`mask()`：保留首尾若干位，中间替换 `••••••`），逐项复制
  - 重置密钥（`/api/instances/{id}/regenerate_keys`，二次确认，旧密钥立即失效）
  - 编辑并保存应用公告（`/announcement`）
  - 云更新版本列表（`/api/update/files/{id}`），标出当前版本，可直接下载（`DownloadUtil` 走系统 DownloadManager）
  - 删除实例（`DELETE /api/instances/{id}`，`keep_keys=true` 保留密钥 30 天）

### 4. 卡密管理
- 卡密列表 + 三段筛选（全部 / 未使用 / 已使用 / 设备卡），下拉刷新。
- **生成卡密**（`/api/cards/create`）：选实例、卡类型（普通 / 设备）、有效天数、数量、前缀、自定义卡密、设备卡的最大设备数与"激活方式"开关。
- 卡密详情：续期（`/api/cards/extend`）、修改最大设备数（`/api/cards/update_max_devices`）、清空绑定设备（`/api/cards/unbind_device`）、删除（`/api/cards/delete`）。
- **绑定设备页**（`CardDevicesActivity`）：展示 `bind_devices` / `activated_devices` 两套列表、`已用 x / max` 统计、单台解绑、全部解绑，与网页控制台对齐。

### 5. 云控（Cloud Control）
- 按实例下拉切换（记住上次选择 `selected_instance`）。
- 变量列表：开关型变量，点击即改（`/api/cloud_control/variable/update`），长按删除。
- 新建变量：`var_id`、名称、默认值（`/api/cloud_control/variable/create`）。
- 对应服务端轮询接口 `/api/cloud_control/poll`，返回 `variables` + `server_time`，供客户端 SDK 实时拉取。

### 6. 云更新（热更新包）
- 选定实例后上传更新包（`/api/updates/upload`）：版本号、显示名、更新说明、文件（≤200MB），上传进度条基于 `ApiService.uploadProgress` 回调逐块刷新。
- 版本历史：版本号、文件名、大小、说明、"当前版本"标记。
- 操作：设为当前版本（`/api/update/set_current`）、下载（先取 `/api/update/download_url`）、删除版本（`/api/update/delete_version`）。

### 7. 部署中心
两个 Tab，全原生管理，能力与网页端基本对齐。

**服务器部署（Python 项目）** `/api/user/deploy/projects`
- 列表 + 新建项目（名称、域名等，走 `DeploySupport` 表单构建器）
- 项目内：文件树浏览、上传（`/api/deploy/upload`，支持多文件）、新建文件（在线编辑保存 `/api/deploy/save_file`）、新建文件夹、设为主程序、删除文件
- 依赖管理：已安装依赖列表（`/api/deploy/deps/list`）、输入包名安装（`/api/deploy/deps/install`）
- 运行日志（`/api/deploy/logs?project_id=`）、清空日志、重启进程（`/api/deploy/restart`）
- HTTPS 开关、敏感内容守卫开关（`toggle_https` / `toggle_sensitive_guard`）

**网站部署（静态站点）** `/api/static/projects`
- 新建静态项目、文件/文件夹管理（上传 `/api/static/upload`、删除、创建目录）
- 设为主入口页（`/api/static/set_main`）、重命名（`/api/static/rename`）
- HTTPS 开关（`/api/user/static/toggle_https`）、点击域名直接浏览器预览

### 8. 在线用户 / 心跳
- 选实例后调 `/api/online/status`（带 `instance_id` + `api_key`），展示在线数、离线数、总用户数及用户列表（设备、最后心跳等）。
- 客户端侧心跳接口 `/api/online/heartbeat`。

### 9. 无限邮箱
- 依赖 Google 绑定（`/api/google/status`）；未绑定则展示引导卡片和一键绑按钮。
- 可选域名池（`/api/unlimited_email/domains`）、创建邮箱（前缀 + 域名 + 转发地址，`/api/unlimited_email/create`），支持"需验证"提示。
- 邮箱列表：复制地址、修改转发邮箱、启停、删除；一键同步转发规则（`/api/unlimited_email/sync`）。

### 10. 福利中心
- `/api/welfare` 拉取福利列表（标题、描述、按钮态），`/api/welfare/claim` 领取，已领取置灰。

### 11. 帮助工单
- 工单列表 + 状态、创建/更新时间、附件名、管理员回复。
- 新建工单：标题、描述、附件（≤15MB，先落盘到私有目录再 multipart 上传 `/api/help/tickets/upload`，然后 `/api/help/tickets/create`）。

### 12. 对接下载（SDK 资源）
- `/api/documents` 文档列表，支持系统 DownloadManager 下载（落 `Environment.DIRECTORY_DOWNLOADS`，文件名非法字符清洗）、复制直链。
- 侧边栏/通知红点：`DocsBadge` 记录已读状态，发现新文档时显示红点。

### 13. 同行识别
- UID 名单维护（`/api/peer/add`、`/api/peer/delete`、`/api/peer/list`），用于检测同行/异常 UID。

### 14. 卡密验证测试工具（原生验证沙箱）
本机即可完成一次完整验证闭环，**加解密与签名全部在 C++ JNI 层执行**：
1. 输入卡密 + 设备 ID（设备 ID 由 `NativeCore.fingerprint(ANDROID_ID)` 生成，取 `md5("ac-cloud|androidId|native-v2")` 前 16 位）
2. `aesEncrypt()` 生成 `data` / `iv`，`sign()` 生成签名
3. POST `/api/verify`（`_instance_id`、`_uid`、`api_key`、`timestamp`、`sign`、`data`、`iv`）
4. 响应中的 `data` / `iv` 再用 `aesDecrypt()` 解出明文 `code` / `message` / `remaining_days` / `device_count`
5. 面板展示签名前缀、时间戳、解密 JSON 和总耗时（ms）

### 15. Shizuku 系统增强（可选）
- 依赖 `dev.rikka.shizuku:api/provider 13.1.5`，Manifest 中注册 `ShizukuProvider` 并声明 `queries`。
- 未安装/未授权时全部能力安全降级，不影响主流程。
- 授权后可通过 binder 创建 `IRemoteProcess` 执行 shell：
  - `ping` 当前节点
  - 网络优化（关闭 WiFi 常扫、关 captive portal 检测、关 Private DNS、调 `tcp_tw_reuse` / `tcp_fin_timeout` / `tcp_keepalive_time`、解除后台网络限制）
  - 刷新 DNS 缓存、查看 `wlan0` 地址与 `net.dns1`
  - `/dev/tcp` 直连目标服务器做 TCP 稳定化

### 16. 内置文件管理器
- 自研 `FileManagerActivity`，不依赖系统文档选择器：目录浏览、多选、文件夹模式、返回相对路径列表（`RESULT_PATHS` / `RESULT_RELATIVES`），供上传更新包、部署文件、工单附件等所有上传入口复用。
- Android 11+ 引导授予 `MANAGE_EXTERNAL_STORAGE`。

### 17. 设置与外观
- 深色/浅色/跟随系统（`AppCompatDelegate`，持久化 `night_mode`）。
- 节点切换（欧洲 / 主服务器），首次启动强制选择节点。
- 修改昵称（`/api/user/update_name`）、修改密码（`/api/user/change_password`）、查看/复制/重置 API 对接码（`/api/user/regenerate_api_key`）。
- 退出登录（`/logout` + 清理本地会话）。

---

## 三、技术架构

```
┌──────────────────────────────────────────────┐
│ UI 层  Activity / Fragment / BottomSheet      │  纯原生 View + ViewBinding
│ 5 个主 Tab：仪表盘 / 实例 / 卡密 / 云控 / 我的   │  Material3 + Inter 字体
└───────────────┬──────────────────────────────┘
                │ Calls（异步门面，回调切回主线程）
┌───────────────▼──────────────────────────────┐
│ 进程内 AIDL：IApiService                      │  UI 永不直接碰 HTTP
│ ApiService（bindService + linkToDeath 重连）   │
└───────────────┬──────────────────────────────┘
                │ OkHttp 3.12.13
┌───────────────▼──────────────────────────────┐
│ 网络：HTTP/2 多路复用 + TLS1.3 + 连接池(8/15min) │
│ 请求队列 64 / 每 Host 16，重定向关闭            │
└───────────────┬──────────────────────────────┘
                │ JNI
┌───────────────▼──────────────────────────────┐
│ libaccloud.so（C++17，NDK r29）                │
│ 自实现 Base64 / AES-256-CBC / MD5 / 设备指纹    │
│ DNS + TCP 握手预热（tcpProbe / tcpWarm）        │
└──────────────────────────────────────────────┘
```

**关键设计点**

1. **网络全下沉到 Service**：`IApiService.aidl` 定义 17 个方法（login / register / sendEmailCode / get / post / delete / postForm / isLoggedIn / getSessionCookie / getUserJson / getApiKey / ping / warmup / appTokenLogin / appGoogleUrl / upload / logout）。UI 通过 `Api.ready()` 排队，服务未就绪时请求自动挂起，连上后 `drain()` 一次性执行；binder 死亡自动重绑。
2. **启动即预热**：`warmup()` 先让 C++ 做 DNS + TCP 握手，再由 OkHttp 建 TLS/HTTP2，首次请求少付握手成本；`DataCache.preload()` 并发拉 5 个核心接口（用户、实例、卡密、公告、文档），8 秒兜底，全部 Tab 从同一份快照渲染，切页不重复请求（2 分钟内视为新鲜）。
3. **会话管理**：`Set-Cookie` 中的 `session=...` 被手工提取存 Prefs，后续请求以 `Cookie` 头注入；`followRedirects(false)` 避免跨域跳转丢会话。
4. **原生加密**：`native-lib.cpp` 自己实现 S 盒查表版 AES-256（14 轮、CBC、PKCS7）、Base64、MD5，`-O2 -fvisibility=hidden`，不链接 OpenSSL，减小体积也避免符号暴露。
5. **安全网络配置**：release 禁明文（`cleartextTrafficPermitted=false`），信任锚 = 系统证书 + 内置 ISRG Root X1/X2 + 用户证书；debug 变体单独放行明文以便本地联调。
6. **视觉体系**：Material3 DayNight、Inter 五字重、玻璃拟态卡片（`card_glass` / `card_stroke`）、`AuroraView` 自绘极光背景（18 秒循环三团径向渐变光斑）、Android 12+ 用 `RenderEffect` / `setBackgroundBlurRadius` 做真实高斯模糊，低版本自动降级为半透明玻璃。

---

## 四、构建环境

### 工具链要求

| 项目 | 值 |
|---|---|
| 构建工具 | Gradle + AGP `8.7.3`（需 **JDK 17**、**Gradle 8.9+**） |
| compileSdk / targetSdk | 35 |
| buildToolsVersion | 35.0.0 |
| minSdk | 24（Android 7.0） |
| NDK | `29.0.13599879`（CMake 不用，走 **ndkBuild** + `Android.mk`） |
| ABI | `arm64-v8a`、`armeabi-v7a`、`x86_64` |
| Java 兼容性 | source/target 1.8 |
| C++ 标准 | C++17，`APP_STL := c++_static`，`APP_PLATFORM := android-24` |
| 构建特性 | ViewBinding、AIDL、BuildConfig 均开启 |

### 依赖清单

```gradle
kotlin-stdlib:2.1.20
androidx.core:1.10.1            androidx.annotation:1.10.0
appcompat:1.7.1                 recyclerview:1.4.0
dynamicanimation:1.1.0          swiperefreshlayout:1.2.0
vectordrawable:1.2.0            collection:1.6.0
lifecycle-runtime/common/process:2.5.1
cardview:1.0.0                  palette:1.0.0
material:1.10.0
okhttp:3.12.13
shizuku:api / provider 13.1.5
```

Gradle 侧对 Kotlin BOM、`kotlin-stdlib-jdk7/jdk8`、`kotlinx`、`profileinstaller` 做了统一排除，避免 collection/annotation 拉高版本导致的运行期 `NoSuchMethodError`。

### 仓库镜像
`settings.gradle` 优先 Huawei Cloud 镜像（`repo.huaweicloud.com`），再回落到 google() / mavenCentral()，国内拉取更快；`RepositoriesMode.FAIL_ON_PROJECT_REPOS` 禁止模块内再声明仓库。

### ⚠️ 构建前必须处理的 4 件事

1. **代理**：`gradle.properties` 写死了 `systemProp.http.proxyHost=127.0.0.1` / `proxyPort=18080`。没有本地代理的环境（CI、他人机器）必须先删除或改掉这四行，否则所有依赖下载都会失败。
2. **签名文件缺失**：`app/build.gradle` 的 release 签名指向 `../keystore/accloud.jks`，压缩包内**不含**该文件，且密码明文写死（`accloud2026` / alias `accloud`）。请自建 keystore 并替换，切勿把生产签名提交进仓库。
3. **Gradle Wrapper 缺失**：仓库里没有 `gradlew`、`gradle/wrapper/*`。用本机已安装的 Gradle 8.9+ 执行，或先 `gradle wrapper --gradle-version 8.9` 生成。
4. **`proguard-rules.pro` 缺失**：`release` 引用了它但文件不在包内；当前 `minifyEnabled false`、`shrinkResources false`，所以不影响编译，但需补一个空文件或自行开启混淆。

### 构建命令

```bash
# 默认服务端地址（构建时注入 BuildConfig.API_BASE）
./gradlew assembleRelease

# 自定义服务端（本地联调 / 自建后端）
./gradlew assembleRelease -PapiBase=http://10.0.2.2:8080

./gradlew assembleDebug      # 包名后缀 .debug，放行明文流量
```

`-PapiBase` 默认值为 `https://xn--dkwo7b.xn--ac-c72cw32v8pp.eu.org`。若传入 `http://`、`10.0.2.2` 或 `192.168.*`，`Servers.customBuild()` 判定为自建环境，**绕过双节点选择**直接使用该地址。

产物：`app/build/outputs/apk/release/app-release.apk`

---

## 五、目录结构

```
ACCloud/
├─ build.gradle                  # AGP 8.7.3 声明
├─ settings.gradle               # 镜像与模块
├─ gradle.properties             # 内存/并行/缓存 + 代理（需改）
└─ app/
   ├─ build.gradle               # SDK/NDK/ABI/签名/BuildConfig
   └─ src/
      ├─ debug/res/xml/network_security_config.xml   # 仅调试包放行明文
      └─ main/
         ├─ AndroidManifest.xml                  # 19 个 Activity + ShizukuProvider + Service
         ├─ aidl/com/ac/cloud/IApiService.aidl   # UI ↔ 网络服务契约
         ├─ java/com/ac/cloud/
         │  ├─ App.java              # 崩溃日志、Prefs、夜间模式、Api 初始化
         │  ├─ core/                 # Json / Prefs / Servers(双节点) / DataCache / NativeCore / ShizukuManager
         │  ├─ net/                  # Api(AIDL 代理) / Calls(异步门面) / Io(线程池)
         │  ├─ service/ApiService.java   # OkHttp 后端（登录/上传/cookie 捕获/预热）
         │  ├─ ui/                   # 19 个 Activity + 7 个 Fragment + 4 个 Sheet + 10 个 Adapter
         │  ├─ util/                 # UiUtils / DownloadUtil / DocsBadge
         │  └─ view/AuroraView.java  # 极光动画背景
         ├─ jni/
         │  ├─ Android.mk / Application.mk
         │  └─ native-lib.cpp        # AES-256/MD5/Base64/指纹/TCP 预热（464 行）
         └─ res/
            ├─ layout/ (51)  drawable/ (116)  values/ + values-night/
            ├─ font/ Inter 5 字重
            └─ raw/ isrg_root_x1.pem, isrg_root_x2.pem
```

---

## 六、服务端接口一览（客户端实际调用）

**账号**
`/login`、`/logout`、`/reset_password`、`/api/app/login`、`/api/app/register`、`/api/app/token_login`、`/api/app/google_url`、`/api/email/send_code`、`/api/user/info`、`/api/user/update_name`、`/api/user/change_password`、`/api/user/regenerate_api_key`、`/api/google/status`

**实例 / 卡密**
`/api/instances`、`/api/instances/create`、`/api/instances/{id}` (DELETE)、`/api/instances/{id}/keys`、`/api/instances/{id}/toggle`、`/api/instances/{id}/regenerate_keys`、`/api/instances/{id}/announcement`、`/api/cards`、`/api/cards/create`、`/api/cards/extend`、`/api/cards/delete`、`/api/cards/update_max_devices`、`/api/cards/unbind_device`、`/api/cards/{card}/devices`

**验证 / 云控 / 在线**
`/api/verify`、`/api/cloud_control/variables`、`/api/cloud_control/variable/create|update|delete`、`/api/cloud_control/poll`、`/api/online/status`、`/api/online/heartbeat`、`/api/notice`

**更新 / 部署 / 静态站**
`/api/updates/upload`、`/api/update/files/{id}`、`/api/update/download_url`、`/api/update/set_current`、`/api/update/delete_version`；
`/api/user/deploy/projects`、`/api/deploy/create`、`/api/deploy/files`、`/api/deploy/upload`、`/api/deploy/create_file`、`/api/deploy/save_file`、`/api/deploy/mkdir`、`/api/deploy/file/delete`、`/api/deploy/file_content`、`/api/deploy/set_main`、`/api/deploy/restart`、`/api/deploy/logs`、`/api/deploy/logs/clear`、`/api/deploy/deps/list`、`/api/deploy/deps/install`、`/api/deploy/project/delete`、`/api/user/deploy/toggle_https`、`/api/user/deploy/toggle_sensitive_guard`；
`/api/static/projects`、`/api/static/create`、`/api/static/files`、`/api/static/upload`、`/api/static/set_main`、`/api/static/rename`、`/api/static/file/delete`、`/api/static/project/delete`、`/api/user/static/toggle_https`

**其它**
`/api/unlimited_email/{domains,list,create,sync,update_forward,toggle,delete}`、`/api/welfare`、`/api/welfare/claim`、`/api/help/tickets`、`/api/help/tickets/create`、`/api/help/tickets/upload`、`/api/documents`、`/api/peer/{list,add,delete}`、`/api/get_announcement`

**验证请求参数**（`/api/verify`、`/api/cloud_control/poll`、公告接口同构）

| 字段 | 类型 | 说明 |
|---|---|---|
| `_instance_id` | string | 实例 ID |
| `_uid` | string | 用户 ID |
| `api_key` | string | API 对接码 |
| `timestamp` | long | 秒级时间戳 |
| `sign` | string | `md5(aes_key + secret_key + api_key + timestamp)` |
| `data` | string | AES-256-CBC 密文（Base64） |
| `iv` | string | 随机 IV（Base64，16 字节） |

明文：`{"card_password":"卡密","device_id":"设备ID"}`；响应为 `{data, iv, timestamp}`，解密后得到 `code / message / remaining_days / device_count`（轮询接口另外返回 `variables`、`server_time`）。

---

## 七、安全与合规说明（务必阅读)

**做得好的地方**
- release 禁止明文 HTTP，重定向关闭，会话 Cookie 手工注入。
- UI 层不直接接触网络，全部经 AIDL 隔离；binder 死亡自动重连。
- 密钥在界面统一脱敏（`UiUtils.mask`），明文只在"复制"动作发生时进入剪贴板。
- `allowBackup="false"`，避免 ADB 备份导出会话；签名/加密在 native 层完成，Java 层无硬编码密钥。
- 崩溃兜底：未捕获异常写入 `getExternalFilesDir("crash")/crash-last.txt`（时间 + 完整堆栈）。

**需要你自行加固的点**
1. **会话明文落盘**：`cookie`、`user_json`、`api_key` 直接存 `SharedPreferences`，root/备份场景可读。建议改 EncryptedSharedPreferences 或 Keystore 包裹。
2. **签名口令入库**：`storePassword "accloud2026"`、`keyPassword`、keystore 路径全部明文在 `build.gradle`，应改为 `local.properties` / 环境变量。
3. **信任用户证书**：`network_security_config.xml` 中 `<certificates src="user" />` 会让抓包工具直接 MITM；生产包建议移除，并按需做证书固定（当前只用内置 ISRG 根，**未做 pinning**）。
4. **无混淆**：`minifyEnabled false` + `shrinkResources false`，类名/接口路径完全暴露；上线前建议开启 R8 并保留 AIDL/反射白名单。
5. **`MANAGE_EXTERNAL_STORAGE`**：全盘文件权限，上架 Google Play 会被重点审核；如非必要，改用 SAF / MediaStore。
6. **Shizuku 提权命令**：`optimizeNetwork()` 会修改全局网络设置，属于系统级副作用，建议明确二次确认并提示用户风险。
7. **调试兜底逻辑**：`LoginActivity` 登录失败时会额外请求 `/api/app/login` 取服务端详细错误，等于一次额外凭据提交，生产环境可考虑移除。

> 客户端仅提供"验证/授权管理"能力，**不含服务端代码**。请确保你的使用场景符合当地法律与目标应用的用户协议；卡密/授权系统不得用于绕过他人软件的正版保护。

---

## 八、二次开发指南

**新增一个页面**
1. Activity 注册进 `AndroidManifest.xml`（统一 `screenOrientation="portrait"`）。
2. `UiUtils.edgeToEdge()` + `applyInsets()` 处理全面屏。
3. 数据请求只用 `Calls.get/post/delete/form/upload`，回调天然在主线程。
4. 需要走原生能力的（加密、TCP、指纹）加到 `IApiService.aidl` + `ApiService`，不要在 UI 里直接开线程发 HTTP。

**新增一个接口调用**
```java
Calls.post("/api/xxx", body.toString(), json -> {
    if (!Json.ok(json)) { UiUtils.toast(Json.message(json)); return; }
    // 成功逻辑
});
```

**改服务端地址**
优先用 `-PapiBase=...`；若要保持双节点，改 `Servers.EUROPE` / `Servers.MAIN` 两个常量即可（注意是 punycode IDN 域名）。

**改主题**
`res/values/colors.xml` 与 `res/values-night/colors.xml` 成对维护；圆角/玻璃质感统一在 `Widget.AC.Card` 样式里调（`cardCornerRadius 26dp`、`card_glass #F7FFFFFF`）。

**加 native 能力**
在 `NativeCore` 声明 `native` 方法 → `native-lib.cpp` 加 `JNIEXPORT`，命名遵循 `Java_com_ac_cloud_core_NativeCore_xxx`；`Android.mk` 已开 `-fvisibility=hidden`，导出函数必须显式 `JNIEXPORT`。

---

## 九、版本信息

| 项 | 值 |
|---|---|
| 应用名 | AC 云验证（`ACCloud`） |
| 版本 | 2.2.0（versionCode 220） |
| 原生库标识 | `AC-Native 2.0 · NDK r29 · AES-256/MD5` |
| 服务器节点 | 欧洲节点 `xn--dkwo7b.xn--ac-c72cw32v8pp.eu.org` / 主节点 `xn--ac-c72cw32v8pp.eu.org` |
| 源码规模 | 253 个文件，Java 57 个、布局 51 个、drawable 116 个，Java/C++ 约 1.08 万行 |
| 最低系统 | Android 7.0（API 24） |
| 目标系统 | Android 15（API 35） |

---

*本文档由源码静态分析生成，功能描述均对应代码中的实际实现与请求路径；服务端行为以你自建/采购的后端为准。*

> （注：以上内容由AC云验证提供）