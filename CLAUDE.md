# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目简介

FongMi/TV：基于 CatVod 体系的 Android 影音应用（纯 Java，无 Kotlin），通过外部 JSON 配置扩展点播/直播内容，同一套业务逻辑编译出电视版与手机版两个应用。

## 常用命令

```bash
# 调试构建（flavor × buildType 组合）
./gradlew assembleLeanbackDebug
./gradlew assembleMobileDebug

# 发布构建（ABI 拆分为 arm64-v8a / armeabi-v7a，无 universal 包）
./gradlew assembleLeanbackRelease
./gradlew assembleMobileRelease

./gradlew clean
./gradlew :app:lintLeanbackDebug   # lint（已禁用 UnsafeOptInUsageError / ExpiredTargetSdkVersion）
```

- **无单元测试**：仓库没有任何 test source set，验证改动靠编译 + lint + 真机/模拟器运行。
- **构建前置条件**：`app/build.gradle` 配置阶段无条件读取根目录 `local.properties` 的签名信息（`storeFile` / `keyAlias` / `storePassword` / `keyPassword`），文件缺失或字段为空会导致**任何**构建（含 debug）在配置期失败。本机不存在该文件时需先创建。
- 版本号在 `app/build.gradle` 的 `defaultConfig`（versionCode / versionName）。

## 模块与构建拓扑

`settings.gradle` 只纳入四个模块：

| 模块 | 作用 |
|---|---|
| `app` | 主应用，`com.fongmi.android.tv` |
| `catvod` | 爬虫抽象层：`com.github.catvod.crawler.Spider` 基类、共享 OkHttp 栈（DoH/代理/Hosts/拦截器）、通用工具 |
| `quickjs` | JS 爬虫引擎（QuickJS wrapper） |
| `chaquo` | Python 爬虫引擎（Chaquopy，Python 3.10，依赖 `chaquo/requirements.txt`） |

根目录的 `hook/`、`tvbus/`、`forcetech/`、`jianpian/`、`thunder/`、`zlive/` 是**独立库工程，不参与主构建**：它们单独编译成 AAR 后放入 `app/libs/`（`forcetech-release.aar` 等）。`hook` 提供 WebView 内核钩子（`App.setHook` 注入）；tvbus/forcetech/jianpian/thunder 是直播特殊流引擎，由 `app/src/main/java/com/fongmi/android/tv/player/extractor/` 下同名 Extractor 接入。`zlive` 目前无调用方。

## 架构核心

### Flavor 双实现模式（改共享代码时最易踩坑）

`app/src/main/` 存放两版共用业务逻辑；`app/src/leanback/`（电视，遥控器焦点导航）与 `app/src/mobile/`（手机，手势触控）各自实现 UI。两套 flavor 源码集存在**同名类、不同签名**的平行实现（如 `Product.java`：leanback 版无 Context 参数、mobile 版有；`ui/base/BaseActivity`、`ui/activity/HomeActivity`、`VideoActivity` 等）。`main` 通过这些类抹平设备差异——给 flavor 同名类加方法时必须两边同步，签名要保持一致，否则 main 编译不过。

### 爬虫加载分派

入口 `api/loader/BaseLoader.java`：按配置中 `site.api` 字段分派——`.py` → `PyLoader`（Chaquopy）、`.js` → `JsLoader`（QuickJS）、`csp_XXX` → `JarLoader`（DexClassLoader 加载外部 JAR）。三引擎最终都产出 `catvod` 模块的 `Spider` 实例。Spider 方法规格与返回 JSON 结构见 `docs/SPIDER.md`。

### 播放器引擎抽象

`player/engine/PlayerEngine.java` 是播放器接口（`Type.EXO` / `Type.MPV`），实现位于 `player/exo/ExoPlayerEngine` 与 `player/mpv/MpvPlayerEngine`，由 `PlayerEngineFactory` 按解码设置创建。错误处理走 `ErrorAction`（`RECOVERED` / `DECODE` / `FATAL`）驱动硬解↔软解自动降级与换源重试。`PlayerManager` 编排引擎生命周期与播放状态。

### 配置系统（应用的主入口）

`api/config/` 下 `VodConfig`（sites/parses/网络规则）、`LiveConfig`、`RuleConfig`、`WallConfig` 均继承 `BaseConfig`，对应一份远程或本地 JSON 配置。配置加载/变更通过 EventBus（`event/ConfigEvent` 等）广播。字段完整定义见 `docs/CONFIG.md`，直播源格式见 `docs/LIVE.md`。

### 本地 HTTP 服务

`server/Server.java` 启动 NanoHTTPD（`server/Nano`），端口从 9978 向上探测到 9998。它同时承担：远程控制 API（`server/process/`，端点文档 `docs/LOCAL.md`）、爬虫本地代理（`catvod` 的 `Proxy`，`/proxy` 路径）、字幕/弹幕推送。播放器/爬虫生成的 URL 常以本地服务地址为壳。

### 数据与事件

- Room 数据库（`db/AppDatabase`，schema 导出到 `app/schemas/`，迁移在 `db/Migrations.java`）。
- EventBus greenrobot，annotation processor 生成 `EventIndex`；跨组件通信基本都走 `event/` 下的事件类。

### 其他横切能力

- DLNA：mobile 作 DMC 投放端、leanback 作 DMR 接收端（JUPnP），代码在 `dlna/`。
- Android Auto：`service/PlaybackService` 实现 MediaLibraryService（leanback）。
- 网络栈统一走 `catvod` 的 OkHttp（`net/OkHttp`），被 `resolutionStrategy.force` 锁版本；DoH/代理/Hosts/CORS 注入/广告拦截都挂在这一层。

## 发布构建的特殊流程

`buildSrc/src/main/groovy/com/fongmi/gradle/AbiApkPackaging.groovy` 在 release 变体上：按 ABI 拆包并重命名为 `{device}-{abi}.apk` → 用 build-tools 的 zipalign + apksigner 执行 `finalizeXxxApks` 任务（替代 AGP 默认签名流程，读取 `local.properties` 签名）→ 把成品复制到根目录 `Release/apk/`。改动打包/签名行为时动这里，而不是 `app/build.gradle`。

## 约定

- Java 21 + core library desugaring；ViewBinding 已启用；注释用中文，标识符用英文。
- 依赖统一在 `gradle/libs.versions.toml` 版本目录管理。
- 仓库文档（README、docs/、代码注释）为繁体中文，保持原风格。
