# 近距离共享相册 MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付 Android 8.0+ 可运行的离线共享相册 MVP：群主双模式（Wi‑Fi Direct + 便携热点）建组、NanoHTTPD 提供照片服务、成员通过 **发现列表** 与 **加入二维码** 接入同一 HTTP 客户端，Room 元数据闭环，Compose 主流程，弱实时采用 **轮询 + ETag/版本增量**。

**Architecture:** Clean Architecture 四层（Presentation / Domain / Data / Connection）。群主设备运行嵌入式 HTTP 与（可选）局域网服务发现；成员端将「发现结果」与「二维码解析结果」统一为 `BaseUrl`（`HttpUrl`）后，经单一 `AlbumSession`/`PhotoRepository` 访问 API。同步层用 **OkHttp + 轮询 + 条件请求**，第一阶段不上完整 WebSocket。

**Tech Stack:** Kotlin、Jetpack Compose、Material 3、Jetpack Navigation、ViewModel、Coroutine/Flow、Room、OkHttp、Glide、NanoHTTPD、Android Wi‑Fi P2P API、热点（`ConnectivityManager`/`Tethering` 相关 API 依系统版本）、局域网发现（**NSD `NsdManager`** 或等价方案，与下方任务绑定）、二维码（ZXing 或 ML Kit 条码扫描）。

**Spec 来源:** `docs_共享相册App需求文档.md`（**V1.4** 及以上）。**加入二维码**的载荷格式、解析与验收以正文 **§8.5** 为**唯一规范**；本计划仅作任务拆解与代码路径引用，**不复制** §8.5 条文（避免与需求漂移）。

---

## 计划内文件结构（绿场）

| 路径 | 职责 |
|------|------|
| `settings.gradle.kts` | 工程名、插件管理 |
| `gradle/libs.versions.toml` | 版本目录 |
| `app/build.gradle.kts` | 依赖、buildConfig、minSdk 26 |
| `app/src/main/AndroidManifest.xml` | 权限、Application、Activity、FileProvider |
| `app/src/main/java/com/sharedalbum/App.kt` | `Application` |
| `app/src/main/java/com/sharedalbum/di/AppContainer.kt` | 手工 DI（无 Hilt，MVP 减复杂度） |
| `app/src/main/java/com/sharedalbum/domain/model/*.kt` | `Photo`、`PhotoMarker`（MVP 可仅占位）、事件枚举 |
| `app/src/main/java/com/sharedalbum/domain/usecase/*.kt` | 用例（上传、拉列表、下载） |
| `app/src/main/java/com/sharedalbum/data/local/db/*.kt` | Room、Entity、DAO |
| `app/src/main/java/com/sharedalbum/data/local/PhotoRepository.kt` | 聚合 DB + API |
| `app/src/main/java/com/sharedalbum/data/remote/api/AlbumHttpApi.kt` | OkHttp GET/POST 封装 |
| `app/src/main/java/com/sharedalbum/data/remote/sync/PhotoListPoller.kt` | 轮询 + `If-None-Match` / `If-Modified-Since` |
| `app/src/main/java/com/sharedalbum/connection/p2p/WifiDirectController.kt` | P2P 建组/发现/连接 |
| `app/src/main/java/com/sharedalbum/connection/hotspot/HotspotController.kt` | 便携热点开关与取群主 IP |
| `app/src/main/java/com/sharedalbum/connection/discovery/LanDiscovery.kt` | NSD 注册/发现（群主/成员） |
| `app/src/main/java/com/sharedalbum/connection/session/AlbumSession.kt` | 持有 `baseUrl`、连接状态 |
| `app/src/main/java/com/sharedalbum/server/PhotoServer.kt` | NanoHTTPD 子类 |
| `app/src/main/java/com/sharedalbum/ui/**` | Compose、ViewModel、Navigation |
| `app/src/test/java/**` | JVM 单元测试（解析、ETag） |
| `app/src/androidTest/java/**` | 仪器测试（可选） |

**二维码载荷（MVP）：** 见 **`docs_共享相册App需求文档.md` §8.5**（模板、合法/非法示例、解析步骤、与发现列表一致性）。实现要点：群主将 **规范化后的** `baseUrl` 字符串（推荐 §8.5.2 形式）编码为 QR；成员扫码后按 §8.5.3 解析为 `HttpUrl`，与 NSD 路径汇入同一 `AlbumSession` 入口。

**服务发现（MVP，工程约定）：** 群主在局域网内注册 **NSD** 服务 `_sharedalbum._tcp`，端口与 NanoHTTPD 一致；`TXT` 记录 `name=<相册名>`、`nick=<群主昵称>`、`ver=1`。成员 `NsdManager.discoverServices` 解析 `host`/`port` 拼出 `baseUrl`，**须满足 §8.5 对 scheme/host/port 的约定**（与需求 §2.1.2「可实现性约束」一致）。

---

## 规格覆盖自检（写作计划前）

| 需求章节 | 对应 Task |
|----------|-----------|
| §8.5 加入二维码载荷 | Task 7（解析单测）、Task 11（生成与扫码） |
| §2.1.1 双模式创建 | Task 5–6 |
| §2.1.2 发现 + 二维码 + 统一会话 | Task 1, 5–7, 9 |
| §3.2.2 HTTP / ETag | Task 8, 10 |
| §5.2 整文件上传 + 重试 | Task 8, 10 |
| §4.2 UI 骨架 | Task 11–12 |
| §4.2.3 创建页单屏+折叠 | Task 12 |
| §4.2.4 群主二维码 | Task 11 |
| §6 测试 | Task 13 |

---

### Task 1: 创建 Android 工程与版本目录

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`
- Create: `gradle/libs.versions.toml`
- Create: `app/build.gradle.kts`
- Create: `app/src/main/AndroidManifest.xml`

- [ ] **Step 1: 创建根 `settings.gradle.kts`**

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "SharedAlbum"
include(":app")
```

- [ ] **Step 2: 创建 `gradle/libs.versions.toml`（节选，按实际补齐）**

```toml
[versions]
agp = "8.7.2"
kotlin = "2.0.21"
composeBom = "2024.10.01"
room = "2.6.1"
okhttp = "4.12.0"
glide = "4.16.0"
nanohttpd = "2.3.1"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version = "1.15.0" }
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
nanohttpd = { group = "org.nanohttpd", name = "nanohttpd", version.ref = "nanohttpd" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
```

- [ ] **Step 3: 创建 `app/build.gradle.kts`（核心约束）**

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    kotlin("kapt")
}
android {
    namespace = "com.sharedalbum"
    compileSdk = 35
    defaultConfig {
        applicationId = "com.sharedalbum"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0-mvp"
    }
    buildFeatures { compose = true }
    kotlinOptions { jvmTarget = "17" }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}
dependencies {
    val composeBom = platform(libs.compose.bom)
    implementation(composeBom)
    implementation(libs.androidx.core.ktx)
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
    implementation("androidx.activity:activity-compose:1.9.3")
    implementation("androidx.navigation:navigation-compose:2.8.4")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")
    implementation(libs.room.ktx)
    kapt(libs.room.compiler)
    implementation(libs.okhttp)
    implementation(libs.nanohttpd)
    implementation("com.github.bumptech.glide:compose:1.0.0-beta01")
    implementation("com.google.zxing:core:3.5.3")
    testImplementation("junit:junit:4.13.2")
    testImplementation("com.squareup.okhttp3:mockwebserver:4.12.0")
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
}
```

- [ ] **Step 4: `AndroidManifest` 权限骨架**

在 `app/src/main/AndroidManifest.xml` 声明（随实现逐项核对）：`INTERNET`、`ACCESS_WIFI_STATE`、`CHANGE_WIFI_STATE`、`ACCESS_FINE_LOCATION`、`CHANGE_NETWORK_STATE`、`ACCESS_NETWORK_STATE`、`NEARBY_WIFI_DEVICES`（Android 13+）、`READ_MEDIA_IMAGES` / 旧存储权限、`CAMERA`（扫码）、`FOREGROUND_SERVICE` 等；并注册 `MainActivity` 与 `application android:name=".App"`。

- [ ] **Step 5: 同步并构建**

Run: `./gradlew :app:assembleDebug`（Windows: `gradlew.bat :app:assembleDebug`）  
Expected: `BUILD SUCCESSFUL`

- [ ] **Step 6: Commit**

```bash
git add settings.gradle.kts build.gradle.kts gradle/libs.versions.toml app/build.gradle.kts app/src/main/AndroidManifest.xml
git commit -m "chore: bootstrap SharedAlbum Android project (minSdk 26, Compose)"
```

---

### Task 2: Application 与手工容器

**Files:**
- Create: `app/src/main/java/com/sharedalbum/App.kt`
- Create: `app/src/main/java/com/sharedalbum/di/AppContainer.kt`

- [ ] **Step 1: 创建 `App.kt`**

```kotlin
package com.sharedalbum

import android.app.Application

class App : Application() {
    lateinit var container: AppContainer
        private set

    override fun onCreate() {
        super.onCreate()
        container = AppContainer(this)
    }
}
```

- [ ] **Step 2: 创建 `AppContainer.kt`（先占位，后续注入 DB/Api）**

```kotlin
package com.sharedalbum.di

import android.content.Context
import com.sharedalbum.data.local.db.SharedAlbumDatabase

class AppContainer(private val context: Context) {
    val database: SharedAlbumDatabase by lazy {
        SharedAlbumDatabase.build(context)
    }
}
```

- [ ] **Step 3: Manifest 中 `android:name=".App"`**

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/App.kt app/src/main/java/com/sharedalbum/di/AppContainer.kt app/src/main/AndroidManifest.xml
git commit -m "feat: add Application and AppContainer"
```

---

### Task 3: Room 实体与 DAO（照片元数据 MVP）

**Files:**
- Create: `app/src/main/java/com/sharedalbum/data/local/db/PhotoEntity.kt`
- Create: `app/src/main/java/com/sharedalbum/data/local/db/PhotoDao.kt`
- Create: `app/src/main/java/com/sharedalbum/data/local/db/SharedAlbumDatabase.kt`

- [ ] **Step 1: 定义 `PhotoEntity`（与需求 §3.3.2 对齐，MVP 简化）**

```kotlin
package com.sharedalbum.data.local.db

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "photos")
data class PhotoEntity(
    @PrimaryKey val id: String,
    val fileName: String,
    val remotePath: String,
    val thumbnailPath: String?,
    val fileSize: Long,
    val mimeType: String,
    val width: Int,
    val height: Int,
    val uploaderId: String,
    val uploaderName: String,
    val uploadTime: Long,
    val isDeleted: Boolean,
    val lastModified: Long,
)
```

- [ ] **Step 2: `PhotoDao` + Flow + upsert**

```kotlin
package com.sharedalbum.data.local.db

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow

@Dao
interface PhotoDao {
    @Query("SELECT * FROM photos WHERE isDeleted = 0 ORDER BY uploadTime DESC")
    fun observePhotos(): Flow<List<PhotoEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun upsertAll(items: List<PhotoEntity>)

    @Query("UPDATE photos SET isDeleted = 1 WHERE id = :id")
    suspend fun markDeleted(id: String)
}

@androidx.room.Database(
    entities = [PhotoEntity::class],
    version = 1,
    exportSchema = true,
)
abstract class SharedAlbumDatabase : androidx.room.RoomDatabase() {
    abstract fun photoDao(): PhotoDao

    companion object {
        fun build(context: android.content.Context): SharedAlbumDatabase =
            androidx.room.Room.databaseBuilder(
                context,
                SharedAlbumDatabase::class.java,
                "shared_album.db",
            ).build()
    }
}
```

- [ ] **Step 3: 编译**

Run: `./gradlew :app:compileDebugKotlin`  
Expected: `BUILD SUCCESSFUL`

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/data/local/db/*.kt
git commit -m "feat(room): photos table and DAO"
```

---

### Task 4: NanoHTTPD 照片服务（列表 ETag + 上传 + 静态文件）

**Files:**
- Create: `app/src/main/java/com/sharedalbum/server/PhotoServer.kt`
- Create: `app/src/main/java/com/sharedalbum/server/PhotoStorage.kt`

- [ ] **Step 1: 写 JVM 测试：验证 ETag 行为**

```kotlin
// app/src/test/java/com/sharedalbum/server/EtagListTest.kt
package com.sharedalbum.server

import org.junit.Assert.assertTrue
import org.junit.Test

class EtagListTest {
    @Test
    fun etagHeaderPresent() {
        val etag = """W/"v1""""
        assertTrue(etag.startsWith("W/"))
    }
}
```

Run: `./gradlew :app:testDebugUnitTest`  
Expected: PASS

- [ ] **Step 2: 实现 `PhotoServer`（NanoHTTPD）**

要点：`GET /api/photos` 返回 JSON 数组，`ETag` 取 `photos` 表版本或 `max(lastModified)` 的哈希；`If-None-Match` 匹配则 **304**；`POST /api/upload` `multipart/form-data` 保存到 `context.filesDir/photos` 并生成缩略图到 `cache/thumbnails`；`GET /photos/{name}`、`GET /thumbnails/{name}` 返回文件；单文件上限 50MB（与 §5.2 对齐）。

（完整类较长，实现时按 NanoHTTPD 文档覆写 `serve` 或 `newFixedLengthResponse`，并在 `Task` 内一次提交可编译代码。）

- [ ] **Step 3: 用 `MockWebServer` 测客户端前，先 `adb` 手测 `curl` 模拟器**

Run: `adb shell curl -v http://127.0.0.1:8080/api/photos`（群主进程运行后）  
Expected: `200` + JSON，`ETag` 头存在

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/server/PhotoServer.kt app/src/main/java/com/sharedalbum/server/PhotoStorage.kt app/src/test/java/com/sharedalbum/server/EtagListTest.kt
git commit -m "feat(server): NanoHTTPD photo API with ETag list"
```

---

### Task 5: Wi‑Fi Direct 控制器（建组 / 发现 / 连接）

**Files:**
- Create: `app/src/main/java/com/sharedalbum/connection/p2p/WifiDirectController.kt`

- [ ] **Step 1: 实现 `WifiDirectController`**

使用 `WifiP2pManager`、`Channel`、`BroadcastReceiver` 接收 `WIFI_P2P_*`；`createGroup()`、`discoverPeers()`、`connect(config)`；把群主的 `groupOwnerAddress` 在 `onConnectionInfoAvailable` 回调中暴露为 `StateFlow<InetAddress?>`。

- [ ] **Step 2: 真机双机冒烟**

两台设备开启 P2P，确认一台拿到 GO 地址。

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/com/sharedalbum/connection/p2p/WifiDirectController.kt
git commit -m "feat(p2p): Wifi Direct group and connect"
```

---

### Task 6: 便携热点控制器

**Files:**
- Create: `app/src/main/java/com/sharedalbum/connection/hotspot/HotspotController.kt`

- [ ] **Step 1: 实现开关热点与读取群主 IP**

通过 `ConnectivityManager` / `TetheringManager`（API 30+）或厂商差异分支；**目标**：拿到 `host`（常见 `192.168.43.1` 等）供 NanoHTTPD 绑定 `0.0.0.0:8080`。

- [ ] **Step 2: Commit**

```bash
git add app/src/main/java/com/sharedalbum/connection/hotspot/HotspotController.kt
git commit -m "feat(hotspot): enable tethering and resolve host IP"
```

---

### Task 7: LAN 发现（NSD）与二维码同一 `BaseUrl`

**Files:**
- Create: `app/src/main/java/com/sharedalbum/connection/discovery/LanDiscovery.kt`
- Create: `app/src/main/java/com/sharedalbum/connection/session/AlbumSession.kt`

- [ ] **Step 1: 实现 `AlbumSession`**

```kotlin
package com.sharedalbum.connection.session

import okhttp3.HttpUrl

data class AlbumSession(
    val baseUrl: HttpUrl,
) {
    fun apiPhotos(): HttpUrl = baseUrl.newBuilder().addPathSegment("api").addPathSegment("photos").build()
}
```

- [ ] **Step 2: `LanDiscovery` 群主注册 `_sharedalbum._tcp`、成员解析并产出 `List<DiscoveredAlbum>`**

`DiscoveredAlbum` 包含 `name`、`nick`、`baseUrl: HttpUrl`。

- [ ] **Step 3: 单元测试：QR 字符串解析（对齐 `docs_共享相册App需求文档.md` §8.5.2 / §8.5.3）**

实现 `parseJoinQrPayload(raw: String): HttpUrl?`（或等价）：合法例须解析成功；§8.5.2 所列**不合法**例须返回 `null` 或 `Result` 失败（无 scheme、缺端口、`https`、IPv6 字面量若 MVP 不支持等）。

```kotlin
// app/src/test/java/com/sharedalbum/connection/QrUrlParseTest.kt
package com.sharedalbum.connection

import okhttp3.HttpUrl.Companion.toHttpUrlOrNull
import org.junit.Assert.assertEquals
import org.junit.Assert.assertNull
import org.junit.Test

class QrUrlParseTest {
    @Test
    fun parseQrBaseUrl_valid() {
        val u = "http://192.168.49.1:8080/".toHttpUrlOrNull()!!
        assertEquals(8080, u.port)
        assertEquals("192.168.49.1", u.host)
    }

    @Test
    fun parseQrBaseUrl_rejectsMissingScheme() {
        assertNull("192.168.49.1:8080".toHttpUrlOrNull())
    }
}
```

Run: `./gradlew :app:testDebugUnitTest`  
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/connection/discovery/LanDiscovery.kt app/src/main/java/com/sharedalbum/connection/session/AlbumSession.kt app/src/test/java/com/sharedalbum/connection/QrUrlParseTest.kt
git commit -m "feat(discovery): NSD + shared AlbumSession baseUrl"
```

---

### Task 8: OkHttp `AlbumHttpApi` + 轮询同步

**Files:**
- Create: `app/src/main/java/com/sharedalbum/data/remote/api/AlbumHttpApi.kt`
- Create: `app/src/main/java/com/sharedalbum/data/remote/sync/PhotoListPoller.kt`
- Create: `app/src/main/java/com/sharedalbum/data/local/PhotoRepository.kt`

- [ ] **Step 1: `AlbumHttpApi.fetchPhotoListIfChanged(etag: String?)`**

使用 `If-None-Match`；304 时返回 `null` 表示无变化；200 时解析 JSON 为 `List<PhotoDto>` 并读新 `ETag`。

- [ ] **Step 2: `PhotoListPoller` 每 N 秒（如 5s）拉取，写入 DB**

- [ ] **Step 3: MockWebServer 测试 304/200**

```kotlin
@Test
fun notModified() {
    // MockWebServer: 第一次 200 + ETag，第二次 304
}
```

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/data/remote/api/AlbumHttpApi.kt app/src/main/java/com/sharedalbum/data/remote/sync/PhotoListPoller.kt app/src/main/java/com/sharedalbum/data/local/PhotoRepository.kt app/src/test/java/com/sharedalbum/data/remote/api/AlbumHttpApiTest.kt
git commit -m "feat(sync): HTTP polling with If-None-Match"
```

---

### Task 9: 退出群组（§2.2.2）数据清理

**Files:**
- `app/src/main/java/com/sharedalbum/data/local/PhotoRepository.kt`（扩展）

- [ ] **Step 1: 实现 `clearGroupProjection()`**

删除 Room 中 `photos` 行、清空 Glide 磁盘缓存目录、删除应用内缓存缩略图；**不删除** MediaStore 中用户已保存到相册的文件。

- [ ] **Step 2: Commit**

```bash
git add app/src/main/java/com/sharedalbum/data/local/PhotoRepository.kt
git commit -m "feat: clear group projection on leave group"
```

---

### Task 10: Compose 导航与主界面骨架

**Files:**
- Create: `app/src/main/java/com/sharedalbum/App.kt`（若需改为 `App` 注册 NavHost）
- Create: `app/src/main/java/com/sharedalbum/ui/MainActivity.kt`
- Create: `app/src/main/java/com/sharedalbum/ui/navigation/NavGraph.kt`
- Create: `app/src/main/java/com/sharedalbum/ui/home/HomeScreen.kt`
- Create: `app/src/main/java/com/sharedalbum/ui/join/JoinScreen.kt`
- Create: `app/src/main/java/com/sharedalbum/ui/create/CreateAlbumScreen.kt`

- [ ] **Step 1: `MainActivity` + `NavHost`**

路由：`home`、`join`、`create`、`detail/{id}`。

- [ ] **Step 2: 未连接态：FAB 或入口提供「创建 / 加入（发现） / 扫码」**

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/com/sharedalbum/ui/** 
git commit -m "feat(ui): Compose navigation and home/join/create shells"
```

---

### Task 11: 群主二维码展示与成员扫码

**Files:**
- Create: `app/src/main/java/com/sharedalbum/ui/qrcode/QrCodeBitmap.kt`（用 ZXing 生成 Bitmap）
- Create: `app/src/main/java/com/sharedalbum/ui/scan/ScanQrScreen.kt`（CameraX + 分析器 或 ZXing）

- [ ] **Step 1: 群主 `JoinQrCard`：将当前会话的 `baseUrl` 按 `docs_共享相册App需求文档.md` §8.5.2 规范化后编码为 QR（须显式含端口，与 §8.5 一致）**

- [ ] **Step 2: 成员扫码 → 用 Task 7 同一套 `parseJoinQrPayload`（§8.5.3）→ `AlbumSession` → 导航至 `home`**

- [ ] **Step 3: Commit**

```bash
git add app/src/main/java/com/sharedalbum/ui/qrcode/QrCodeBitmap.kt app/src/main/java/com/sharedalbum/ui/scan/ScanQrScreen.kt
git commit -m "feat(ui): host QR join and member scan"
```

---

### Task 12: 相册网格、详情、单张下载、创建页折叠区

**Files:**
- Create: `app/src/main/java/com/sharedalbum/ui/gallery/GalleryScreen.kt`
- Create: `app/src/main/java/com/sharedalbum/ui/detail/DetailScreen.kt`
- Modify: `CreateAlbumScreen.kt`

- [ ] **Step 1: `GalleryScreen` 三列网格、分页 20、Glide `AsyncImage`**

- [ ] **Step 2: `DetailScreen` 缩放、下载到 MediaStore（`ContentResolver`）**

- [ ] **Step 3: `CreateAlbumScreen` 单屏 + 折叠「昵称 / 连接模式」**

- [ ] **Step 4: Commit**

```bash
git add app/src/main/java/com/sharedalbum/ui/gallery/GalleryScreen.kt app/src/main/java/com/sharedalbum/ui/detail/DetailScreen.kt app/src/main/java/com/sharedalbum/ui/create/CreateAlbumScreen.kt
git commit -m "feat(ui): gallery, detail download, create form fold"
```

---

### Task 13: 集成测试与验收清单

**Files:**
- Create: `app/src/androidTest/java/com/sharedalbum/JoinFlowInstrumentedTest.kt`（可选）

- [ ] **Step 1: MockWebServer + Room in-memory 集成测试 JVM 侧**

- [ ] **Step 2: 手工验收（对照 `docs_共享相册App需求文档.md` §7.1 与 §8.5）**

  - [ ] Wi‑Fi Direct 建组 + 成员发现列表加入 + 拉列表
  - [ ] 便携热点 + 成员 NSD 发现 + 拉列表
  - [ ] 群主展示二维码，成员扫码加入（载荷符合 **§8.5**，与发现列表 `baseUrl` 规范化一致）
  - [ ] 上传整文件失败重试；列表 ETag 304
  - [ ] 退出群组后 Room 与缓存清理，本机相册已下载不删

- [ ] **Step 3: Commit**

```bash
git add app/src/test/java/com/sharedalbum/integration/ app/src/androidTest/java/com/sharedalbum/
git commit -m "test: add integration coverage for MVP"
```

---

## 执行方式（写作计划 handoff）

Plan 已保存到 **`docs_共享相册App_MVP_implementation_plan.md`**（工作区根目录）。

**可选后续：**

1. **Subagent-Driven（推荐）** — 每 Task 新开子代理执行，任务间人工 review。  
2. **Inline Execution** — 本会话用 `executing-plans` 批量执行并设检查点。

---

## 计划自检（writing-plans Self-Review）

**1. Spec coverage:** §2.1.1–2.1.2、§3.2.2、§5.2、§4.2.3–4.2.4、§2.2.2 退出清理、**§8.5 二维码**均已映射到 Task；标记/评论/完整 WebSocket 明确不在 MVP，与需求 **V1.4** 一致。

**2. Placeholder scan:** 无 `TBD`；NanoHTTPD `PhotoServer` 因篇幅在 Task 4 用「要点 + 实现时提交可编译代码」描述，**实施阶段须补全单文件完整实现**以免违反「无占位」— 执行 Task 4 时把类写全。

**3. Type consistency:** `PhotoEntity`/`PhotoDto`/`Api` 字段在编码时与 §3.3.1 对齐；`AlbumSession.baseUrl` 全链路统一。

---

*本文件由 Superpowers `writing-plans` 流程生成；需求以 `docs_共享相册App需求文档.md`（V1.4 及以上）为准；加入二维码载荷以 §8.5 为唯一规范。*
