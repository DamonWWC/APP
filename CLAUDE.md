# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**局域网临时共享相册** — Android 原生应用，支持热点/Wi-Fi Direct 下的临时照片共享，以预览为主、协作编辑（删/排/星/注/入选）、原图按需传输。

## 核心架构

- **星型拓扑**：创建者设备运行 Session Hub（Ktor CIO），全员以 WebSocket 客户端连接
- **协议**：`pv`（协议版本）+ `v`（房间状态版本）+ LWW 冲突解决
- **二维码载荷**：`pv`/`roomId`/`token`/`host`/`port`/`path`，token 从开房到结束不变
- **原图传输**：经 Hub 分块转发，Hub 不落盘原图

## 技术栈

| 层级 | 选型 |
|------|------|
| UI | Kotlin + Jetpack Compose + Navigation |
| 异步 | Coroutines / Flow |
| Hub | Ktor Server CIO + WebSocket |
| 序列化 | kotlinx.serialization |
| 图片 | Coil（展示）、Bitmap 降采样（预览生成） |
| 相册 | ContentResolver 读、MediaStore 写 |
| HTTP 服务 | NanoHTTPD（照片服务） |
| P2P | Wi-Fi Direct API |

## 关键目录

```
app/src/main/java/com/sharedalbum/
├── connection/
│   ├── p2p/WifiDirectController.kt      # Wi-Fi Direct 建组/发现
│   ├── hotspot/HotspotController.kt     # 便携热点
│   ├── discovery/LanDiscovery.kt        # NSD 局域网发现
│   └── session/AlbumSession.kt          # 统一会话入口
├── server/
│   ├── PhotoServer.kt                   # NanoHTTPD 照片 API
│   └── PhotoStorage.kt
├── data/
│   ├── local/db/                       # Room 实体/DAO
│   ├── remote/api/AlbumHttpApi.kt       # OkHttp API 封装
│   └── remote/sync/PhotoListPoller.kt   # ETag 轮询同步
├── domain/model/                        # Photo 等领域模型
├── ui/                                  # Compose 页面
└── di/AppContainer.kt                   # 手工 DI 容器
```

## 构建命令

```bash
./gradlew :app:assembleDebug       # Debug 构建
./gradlew :app:compileDebugKotlin # Kotlin 编译检查
./gradlew :app:testDebugUnitTest  # 单元测试
```

## 需求规范

- 详细需求：`详细需求文档.md`
- 技术设计：`技术设计文档.md`
- 实现计划：`docs_共享相册App_MVP_implementation_plan.md`
- 二维码载荷格式：见需求文档 §8.5（唯一规范来源）

## 注意事项

- `joinToken` 必须使用高熵随机数（`SecureRandom`）
- Hub 绑定 `0.0.0.0` 而非 `127.0.0.1`，确保热点网段可达
- 前台服务使用 `foregroundServiceType: dataSync` 或 `connectedDevice`
- 成员杀进程不断房；创建者杀进程/结束共享即房间失效
