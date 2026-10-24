# RSIS 6.2 Native Android: complete source, one page (build 5)

Status: never compiled or run. Verify `PairingContract` (in `sync/PairingClient.kt`) against the real `nativePair` function before the first pairing test. Bluetooth company ID `0xFFFF` is a test value.

How to use: in Android Studio create the folders shown in each heading, create each file, and paste its block. Paths are relative to the project root `rsis-native-android/`. The file under `website/` is for your website, not the Android project.

## File index

1. `.gitignore`
2. `CONSENT_COPY.md`
3. `README.md`
4. `UPGRADE-build5.md`
5. `build.gradle.kts`
6. `gradle.properties`
7. `settings.gradle.kts`
8. `protocol/rsis-protocol.json`
9. `.github/workflows/build-apk.yml`
10. `app/build.gradle.kts`
11. `app/src/main/AndroidManifest.xml`
12. `app/src/main/java/com/rsis/nativeapp/Constants.kt`
13. `app/src/main/java/com/rsis/nativeapp/MainActivity.kt`
14. `app/src/main/java/com/rsis/nativeapp/Prefs.kt`
15. `app/src/main/java/com/rsis/nativeapp/ble/BleAdvertiser.kt`
16. `app/src/main/java/com/rsis/nativeapp/ble/BleScanner.kt`
17. `app/src/main/java/com/rsis/nativeapp/bridge/NativeBridge.kt`
18. `app/src/main/java/com/rsis/nativeapp/engine/DistanceEstimator.kt`
19. `app/src/main/java/com/rsis/nativeapp/engine/SafetyEngine.kt`
20. `app/src/main/java/com/rsis/nativeapp/service/RsisService.kt`
21. `app/src/main/java/com/rsis/nativeapp/spb/Spb.kt`
22. `app/src/main/java/com/rsis/nativeapp/store/EventStore.kt`
23. `app/src/main/java/com/rsis/nativeapp/sync/Http.kt`
24. `app/src/main/java/com/rsis/nativeapp/sync/PairingClient.kt`
25. `app/src/main/java/com/rsis/nativeapp/sync/SyncClient.kt`
26. `app/src/main/java/com/rsis/nativeapp/voice/VoiceAlerts.kt`
27. `app/src/test/java/com/rsis/nativeapp/EngineTest.kt`
28. `app/src/test/java/com/rsis/nativeapp/SharedSpecTest.kt`
29. `app/src/test/java/com/rsis/nativeapp/SpbTest.kt`
30. `website/rsis-android-gate.html`


---

## `.gitignore`

````
*.jks
*.keystore
local.properties
build/
.gradle/
.idea/
````

---

## `CONSENT_COPY.md`

````markdown
# RSIS Offline Safety Mode — consent / acknowledgement copy (DRAFT for your review and legal check)

## Offline Safety Mode (Bluetooth Mesh) — Android app only
Offline Safety Mode warns you by voice about other nearby RSIS users when you have no internet. It works by your
phone listening for and broadcasting short-range Bluetooth signals directly to other RSIS phones.

**Android only.** This needs Bluetooth hardware access that only the separate RSIS Android app can provide. It is
**not available** in a web browser or on iPhone, and you will not send or receive offline Bluetooth alerts there.
The RSIS Android app is the Bluetooth build. An app made by simply wrapping the RSIS website does not include Offline Safety Mode.

## How to use it
1. Install the RSIS Android app (steps below if you are not using Google Play).
2. Open RSIS, choose your role, and tap **Turn on Bluetooth Mesh**.
3. Allow Nearby devices (Bluetooth), Location, and Notifications when Android asks. Allow Bluetooth to be turned on if prompted.
4. Keep Bluetooth on. A permanent "RSIS Bluetooth Mesh is active" notification shows while it runs; your screen can be locked and the phone in a pocket.
5. If Bluetooth is turned off, you are not protected offline. RSIS shows an orange bar; tap it to turn Bluetooth back on.
6. After your phone restarts, open RSIS once to turn Offline Safety Mode back on.

## Both phones need Bluetooth on
You will only get an offline alert if the other person is also running RSIS with Bluetooth on. If nobody nearby is using it, no offline alert will occur.

## What is shared
Only your role (Cyclist, Pedestrian, Vehicle, or Passenger) and a random code that changes every 3 seconds. Nothing identifies you, and no Bluetooth identifier is stored.

## Battery and background use
Bluetooth and location run continuously while Offline Safety Mode is on, with a permanent notification. This uses significantly more battery.

## Installing outside Google Play (manual install)
1. Settings → Apps → Special app access → Install unknown apps → allow your browser or file manager.
2. Download the RSIS Android app file only from the official RSIS download page [URL].
3. Tap the file and choose Install. Android and Google Play Protect may warn you the app is from outside the Play Store; this is expected.
4. Manually installed apps do not update automatically. You must download and install updates yourself.
5. Compare the file's checksum with the one published on the download page: [CHECKSUM].

## Use at your own risk
Offline Safety Mode is a supplementary warning tool. It does not replace your attention, does not guarantee you will be
alerted to every hazard, and is not a substitute for emergency services. Bluetooth range is reduced by buildings, terrain,
vehicles, your body, and your phone's hardware. Alerts can be missed or delayed. By continuing you accept that you use it at your own risk.
````

---

## `README.md`

````markdown
# RSIS 6.2 Native (Android / Kotlin)

Native BLE layer for RSIS 6.2: SPB v4.1 advertise + scan, offline FAST-PHYSICS + SAFETY_ALERT engine,
female-voice alerts, durable event queue, sync client, and a WebView host that injects `window.RSISNative`.

**Status: written but NOT compiled or run.** Open in Android Studio (it creates the Gradle wrapper), build, run
the unit tests, then do the §10 acceptance tests on two physical phones.

## Layout
- `spb/Spb.kt` packet encode/decode + CRC16
- `ble/` advertiser (3 s token rotation, Coded PHY if supported) and scanner (CRC-validated, SPB-only filter)
- `engine/` distance estimator + safety engine (tiers, cooldown, escalation, 250 ft fallback, dense-traffic cluster)
- `voice/` TTS, offline female voice selection
- `store/` SQLite event queue + §6a payload builder
- `sync/` flushes oldest-first, one at a time, leaves on failure
- `service/RsisService.kt` foreground service (connectedDevice|location), survives screen lock
- `bridge/NativeBridge.kt` + `MainActivity.kt` WebView host, role/permission onboarding, honest banners

## Decisions I had to make (the brief didn't specify) — confirm each
1. **CRC16** = CCITT-FALSE (poly 0x1021, init 0xFFFF) over bytes 0–12, big-endian. `risk_scaled` big-endian.
2. **role_flags**: bits 0–1 role (Cyclist 0, Pedestrian 1, Vehicle 2, Passenger 3); bits 2–7 reserved = 0.
3. **BLE company ID** is `0xFFFF` (Bluetooth SIG test value). Needs a real one before release.
4. **Same-user linking across token rotation**: the spec requires one 3.5-min cooldown per road user but tokens
   change every 3 s. I link a new token to a recent same-role track by distance continuity — in memory only,
   never persisted. Two same-role users at similar distance can merge (fewer alerts, not more). Needs your call.
5. **Orange/Yellow thresholds & collision probability** (TTC horizon 8 s, Orange at p ≥ 0.6, "approaching" = closing
   > 1 mph) are my physics heuristics; the brief gives no formula. They need field calibration.
6. **Direction** comes from RSSI-distance trend (SPB carries no heading). RSSI is noisy; expect tuning.
7. **Fallback alert** is implemented literally (500 ft alert → 5 s → still < 250 ft and no new beacon → speak again).
   It repeats the same event ID and writes no second record. Because beacons arrive ~10×/s it will rarely trigger.
8. **Dense traffic**: ≥3 nodes ≤ 98.4 ft, each |closing| < 3.1 mph, own GPS speed < 3.1 mph. Re-evaluated every 210 s once active.
9. **Event `role`** = the local user's role. Confirm the server expects that, not the detected user's role.
10. **Lat/long** are omitted when no GPS fix exists (never faked). Confirm the server accepts that.
11. **`connectivity_state`**: stored as `offline_at_detection`, sent as `reconnected` when flushed later.
12. **Female voice**: Android has no gender API. I pick an offline voice whose name contains "female"; if none is
    installed the alert still plays (safety first) and a banner tells the user. Verify on your test devices.
13. **Bridge hand-off**: `getQueuedEvents()` returns and clears the queue (web app syncs them). Native only syncs what remains.
    The bridge answers only while the page is the RSIS origin.
14. Native alerts are recorded only when an alert was actually raised (or attempted). Suppressed/receding/cluster
    detections create no events.

## Changes in build 2
- One welcome screen (what it does, Android only, role, one button) replaces the two popups; then permissions; then Android's own "turn on Bluetooth?" dialog.
- Bluetooth on/off is watched live: off -> status leaves `running`, notification and orange bar say Mesh offline; on -> radios restart.
- 250 ft fallback: 5 s after an alert, if the user is still heard, still inside 250 ft and not receding, the alert is spoken again on the SAME event (no new record).
- Token association never guesses: if two tracks could match, it is treated as a new user (extra alert, never a missed one).
- `protocol/rsis-protocol.json` is the shared spec; `SharedSpecTest` fails the build if Kotlin constants drift. Give the same file to the web side.
- `CONSENT_COPY.md`: draft acknowledgement text.
- Bridge still has exactly three members. Role is NOT shared with the web app (that would need a fourth member).

## Changes in build 3
- Bridge now has a 4th, additive member `getRole()` (the role picked on the native welcome screen) so the web app can stop asking twice. The website must be changed to read it; the three required members are unchanged.
- Optional upload key: set `rsisApiKey` (and `rsisApiHeader`, default `X-RSIS-Key`) in `~/.gradle/gradle.properties`. A key inside an APK can be extracted, so this deters casual abuse only; the server must also validate and rate-limit. The header name is a guess until Base44 confirms its format.
- `.github/workflows/build-apk.yml`: builds a debug APK and prints its SHA-256 in GitHub. Untested.
- `rsis-android-gate.html` (delivered beside this zip): the Android-only gate, acknowledgements, checksum and install steps for the website.

## Changes in build 4
- `getBleStatus()` returns `running` only when the service is up AND the system confirmed advertising started AND scanning started. Scan/advertise failures or Bluetooth-off clear it. (Android gives no success callback for scanning, so "scanning" means the scan was started without a reported failure.)
- `getQueuedEvents()` now always returns `[]`. The native layer uploads its own queue, because it holds the upload key and the web SDK does not (the web re-upload would 401 forever). The web app should keep calling it; it just gets nothing.
- Wire field for time is `timestamp` (already so in `EventPayloads`).

## Changes in build 5 (Claude merge)
See `UPGRADE-build5.md`. Pairing client added against an UNVERIFIED contract; build-time key removed; no device fingerprint.

## Not built / out of scope
- **iOS.** iOS will not advertise custom manufacturer data in the background, so the 15-byte SPB as specified can't
  run there. That needs a packet/advertising redesign decision first.
- Boot auto-start (Android 14 blocks starting these foreground service types from boot).
- WorkManager retry while the app is dead (currently flushes on app open, on network available, and every 60 s while running).
- The 15% add-on (not in the brief).
````

---

## `UPGRADE-build5.md`

````markdown
# Build 5 (Claude merge): pairing, anonymous by design

Built from the review decisions, WITHOUT sight of Base44's own build-5 files or the real `nativePair` function.
Never compiled or run.

## What changed
- `sync/PairingClient.kt` (new): obtains an upload key at runtime. `install_id` = 16 random bytes, not derived from any device value.
- `sync/Http.kt` (new): shared POST helper.
- `sync/SyncClient.kt`: pairs lazily (only when events are queued), sends the key header, handles 401 (re-pair), 403 (revoked), 409 (stale build); events stay queued on every failure.
- `Prefs.kt`: pairing state. `Constants.kt`: `NativeProtocol`, `Timing.REPAIR_AFTER_MS` (7 days), `RsisRuntime.syncIssue/uploadsBlocked`.
- `app/build.gradle.kts`: build-time key removed entirely; versionCode 5.
- `MainActivity.kt`: banner "Uploads paused: ..." (detection and voice alerts still work).
- `SharedSpecTest.kt`: now also checks protocol name and packet version. All earlier drift checks kept.

## Decisions applied
1. No `device_fp`. Nothing hashed from the WebView or hardware is sent.
2. The key and `install_id` are discarded and re-issued every 7 days, so uploads cannot be linked over long periods.
3. Pairing gates uploads only. Detection, alerts and queuing never wait for it.
4. Build-4 fixes preserved (truthful `running`, `getQueuedEvents()` returns `[]`, `getRole()`).

## Must be verified against the real server before the first test
`PairingContract` in `PairingClient.kt` holds every guessed name: path `/nativePair`, request fields
(`install_id`, `role`, `protocol`, `packet_version`), response field `key`, and key header `X-RSIS-Key`.
Open the real `nativePair` and `syncNativeEvent` validators and correct that one object. Also confirm the server
tolerates a new `install_id` appearing every 7 days.

## Merging with Base44's build 5
Do not `cp -r` over this tree. Diff first. Keep the build-4 behaviour above and keep `device_fp` out.
If Base44's files differ in contract details, change `PairingContract`, not the engine.

## Verify
1. `gradle :app:testDebugUnitTest` (or run the GitHub Actions workflow).
2. One phone: pair, then confirm the server sees `protocol` = SPB v4.1 and `packet_version` = 65.
3. Two phones: the §10 acceptance tests, including airplane-mode queue then reconnect (exactly one record per event).
4. Failure paths: revoke the install (banner "unpaired by support", uploads stop, detection continues); wrong packet version (banner "out of date").
````

---

## `build.gradle.kts`

````kotlin
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
````

---

## `gradle.properties`

````properties
org.gradle.jvmargs=-Xmx2g
android.useAndroidX=true
kotlin.code.style=official
````

---

## `settings.gradle.kts`

````kotlin
pluginManagement {
    repositories { google(); mavenCentral(); gradlePluginPortal() }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "RSIS"
include(":app")
````

---

## `protocol/rsis-protocol.json`

````json
{
  "protocol": "SPB v4.1",
  "packet": { "version": 65, "size": 15, "token_bytes": 8, "token_rotation_ms": 3000 },
  "alert_perimeters": { "gps_relay_miles": 1.0, "ble_mesh_ft": 500, "ble_fallback_ft": 250, "immediate_ft": 0 },
  "bands_ft": { "critical_max": 16, "high_max": 49, "medium_max": 98, "low_max": 164 },
  "timing_ms": { "cooldown": 210000, "fallback_wait": 5000, "cluster_recompute": 210000 },
  "cluster": { "radius_ft": 98.4, "min_nodes": 3, "max_mph": 3.1 },
  "sensing_ft_never_alert_radii": {
    "open_road": [98, 262], "urban_obstructed": [49, 131], "warning_zone": [49, 164], "multi_hop_relay": [200, 300]
  }
}
````

---

## `.github/workflows/build-apk.yml`

````yaml
# Cloud build: runs the unit tests and produces a debug APK + its SHA-256. Untested workflow; expect to tweak it.
name: build-apk
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
        with: { gradle-version: 8.7 }
      - name: Unit tests
        run: gradle :app:testDebugUnitTest
      - name: Build debug APK
        run: gradle :app:assembleDebug
      - name: Checksum
        run: |
          cd app/build/outputs/apk/debug
          sha256sum app-debug.apk | tee "$GITHUB_STEP_SUMMARY"
      - uses: actions/upload-artifact@v4
        with:
          name: rsis-debug-apk
          path: app/build/outputs/apk/debug/app-debug.apk
````

---

## `app/build.gradle.kts`

````kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.rsis.nativeapp"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.rsis.nativeapp"
        minSdk = 26
        targetSdk = 34
        versionCode = 5
        versionName = "6.2.0-native.5"
    }
    buildTypes { release { isMinifyEnabled = false } }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions { jvmTarget = "17" }
    testOptions { unitTests.isReturnDefaultValues = true }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("androidx.webkit:webkit:1.11.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.json:json:20240303")
}
````

---

## `app/src/main/AndroidManifest.xml`

````xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-feature android:name="android.hardware.bluetooth_le" android:required="true" />

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <!-- Android 12+: no location needed to scan; we declare we don't derive position from BLE -->
    <uses-permission android:name="android.permission.BLUETOOTH_SCAN"
        android:usesPermissionFlags="neverForLocation" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADVERTISE" />
    <uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
    <!-- Android 11 and below -->
    <uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />

    <!-- GPS position for event records and own-speed (cluster detection); also required for BLE scan on Android 11- -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_CONNECTED_DEVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />

    <application
        android:allowBackup="false"
        android:label="RSIS"
        android:usesCleartextTraffic="false"
        android:theme="@style/Theme.AppCompat.DayNight.NoActionBar"
        tools:ignore="DataExtractionRules">

        <activity android:name=".MainActivity" android:exported="true"
            android:configChanges="orientation|screenSize|keyboardHidden">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <service android:name=".service.RsisService" android:exported="false"
            android:foregroundServiceType="connectedDevice|location" />
    </application>
</manifest>
````

---

## `app/src/main/java/com/rsis/nativeapp/Constants.kt`

````kotlin
package com.rsis.nativeapp

/** Online alert perimeters. FROZEN (spec §2.4). Never derived from signal strength. */
object AlertPerimeters {
    const val GPS_RELAY_MILES = 1.0
    const val BLE_MESH_ALERT_FT = 500.0
    const val BLE_FALLBACK_ALERT_FT = 250.0
    const val IMMEDIATE_FT = 0.0
}

/**
 * What the BLE radio can physically hear (spec §4). NEVER used as an alert radius.
 * Kept in a separate object on purpose (spec §2.3). Nothing in the engine reads this.
 */
object RadioSensingRanges {
    val OPEN_ROAD_FT = 98.0..262.0
    val URBAN_OBSTRUCTED_FT = 49.0..131.0
    val WARNING_ZONE_FT = 49.0..164.0
    val MULTI_HOP_RELAY_FT = 200.0..300.0
}

/** Must match protocol/rsis-protocol.json (SharedSpecTest enforces it). A stale build is refused pairing (409). */
object NativeProtocol {
    const val PROTOCOL = "SPB v4.1"
    const val PACKET_VERSION = 65
}

object Timing {
    /** Upload credential + install_id are discarded and re-issued this often, so uploads can't be linked over long periods. */
    const val REPAIR_AFTER_MS = 7L * 24 * 60 * 60 * 1000
    const val COOLDOWN_MS = 210_000L          // 3.5 min per road user
    const val FALLBACK_WAIT_MS = 5_000L       // 500 ft alert -> wait -> 250 ft fallback
    const val TOKEN_ROTATION_MS = 3_000L
    const val CLUSTER_RECOMPUTE_MS = 210_000L // synced to cooldown
}

object ClusterRule {
    const val RADIUS_FT = 98.4
    const val MIN_NODES = 3
    const val MAX_MPH = 3.1
}

enum class Role(val code: Int, val label: String, val spoken: String) {
    CYCLIST(0, "Cyclist", "cyclist"),
    PEDESTRIAN(1, "Pedestrian", "pedestrian"),
    VEHICLE(2, "Vehicle", "vehicle"),
    PASSENGER(3, "Passenger", "passenger");

    companion object {
        fun fromCode(c: Int): Role = values().first { it.code == (c and 0x03) }
    }
}

enum class Tier { WHITE, YELLOW, ORANGE }

/** BLE proximity bands, feet (spec §4). */
enum class Band {
    CRITICAL, HIGH, MEDIUM, LOW, OUT_OF_BAND;

    companion object {
        fun of(ft: Int): Band = when {
            ft < 16 -> CRITICAL
            ft < 49 -> HIGH
            ft < 98 -> MEDIUM
            ft <= 164 -> LOW
            else -> OUT_OF_BAND
        }
    }
}

/** Process-wide, in-memory-only state read by the bridge and UI. Nothing here is persisted. */
object RsisRuntime {
    @Volatile var serviceUp: Boolean = false
    @Volatile var advertising: Boolean = false   // set only when the system confirms advertising started
    @Volatile var scanning: Boolean = false      // set when scan started; cleared on any scan failure
    /** What the bridge reports. Never true unless both radios are genuinely active. */
    val running: Boolean get() = serviceUp && advertising && scanning
    @Volatile var issue: String? = null
    /** null = not determined yet */
    @Volatile var femaleVoice: Boolean? = null
    /** Upload-side problem only. Never affects detection or alerts. */
    @Volatile var syncIssue: String? = null
    @Volatile var uploadsBlocked: Boolean = false
}
````

---

## `app/src/main/java/com/rsis/nativeapp/MainActivity.kt`

````kotlin
package com.rsis.nativeapp

import android.Manifest
import android.annotation.SuppressLint
import android.bluetooth.BluetoothAdapter
import android.bluetooth.BluetoothManager
import android.content.Intent
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.graphics.Typeface
import android.net.Uri
import android.os.Build
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.provider.Settings
import android.view.View
import android.view.ViewGroup
import android.webkit.GeolocationPermissions
import android.webkit.WebChromeClient
import android.webkit.WebResourceRequest
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.*
import androidx.activity.OnBackPressedCallback
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import androidx.webkit.WebViewCompat
import androidx.webkit.WebViewFeature
import com.rsis.nativeapp.bridge.NativeBridge
import com.rsis.nativeapp.bridge.WEB_HOST
import com.rsis.nativeapp.bridge.WEB_ORIGIN
import com.rsis.nativeapp.service.RsisService
import com.rsis.nativeapp.store.EventStore
import com.rsis.nativeapp.sync.SyncClient
import kotlinx.coroutines.*

/**
 * Front door: the native app. One welcome screen (what it does + role + one button), then the RSIS web app
 * loads inside it with window.RSISNative already injected.
 */
class MainActivity : AppCompatActivity() {
    private lateinit var web: WebView
    private lateinit var banner: TextView
    private lateinit var onboarding: View
    private lateinit var roleGroup: RadioGroup
    private lateinit var bridge: NativeBridge
    private var deniedOnce = false
    private val handler = Handler(Looper.getMainLooper())

    private val permLauncher = registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) {
        if (hasCore()) { deniedOnce = false; enableBluetoothThenStart() } else { deniedOnce = true; onboarding.visibility = View.GONE }
        refreshBanner()
    }
    private val btLauncher = registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { startIfReady() }

    @SuppressLint("SetJavaScriptEnabled")
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        bridge = NativeBridge(this) { runOnUiThread { ensureReady() } }

        banner = TextView(this).apply {
            setPadding(32, 24, 32, 24); setTextColor(0xFF000000.toInt()); setBackgroundColor(0xFFFFE0B2.toInt())
            visibility = View.GONE
            setOnClickListener { onBannerTap() }
        }
        web = WebView(this)
        val main = LinearLayout(this).apply {
            orientation = LinearLayout.VERTICAL
            addView(banner, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT))
            addView(web, LinearLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f))
        }
        onboarding = buildOnboarding().apply { visibility = View.GONE }
        setContentView(FrameLayout(this).apply {
            addView(main, FrameLayout.LayoutParams(-1, -1))
            addView(onboarding, FrameLayout.LayoutParams(-1, -1))
        })
        configureWeb()
        web.loadUrl(WEB_ORIGIN)

        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                when {
                    onboarding.visibility == View.VISIBLE -> onboarding.visibility = View.GONE
                    web.canGoBack() -> web.goBack()
                    else -> finish()
                }
            }
        })

        if (Prefs.role(this) == null || !hasCore()) onboarding.visibility = View.VISIBLE else enableBluetoothThenStart()
        CoroutineScope(Dispatchers.Default).launch { SyncClient(this@MainActivity, EventStore(this@MainActivity)).flush() }
    }

    override fun onResume() {
        super.onResume()
        handler.post(poll)
    }
    override fun onPause() { handler.removeCallbacks(poll); super.onPause() }

    private val poll = object : Runnable {
        override fun run() { refreshBanner(); handler.postDelayed(this, 2_000) }
    }

    // ---------- onboarding: one screen ----------
    private fun buildOnboarding(): View {
        val col = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setPadding(56, 72, 56, 56) }
        fun text(s: String, size: Float, bold: Boolean = false) = TextView(this).apply {
            this.text = s; textSize = size; setTextColor(0xFF111111.toInt()); setPadding(0, 0, 0, 28)
            if (bold) setTypeface(typeface, Typeface.BOLD)
        }
        col.addView(text("RSIS Bluetooth Mesh", 26f, true))
        col.addView(text(
            "Warns you by voice about other RSIS users nearby, even with no internet.\n\n" +
                "• Android only. It is not available in a web browser or on iPhone.\n" +
                "• Other riders and drivers must also be running RSIS with Bluetooth on.\n" +
                "• Nothing identifies you: only your role and a code that changes every 3 seconds.\n" +
                "• It runs in the background with a permanent notification and uses more battery. " +
                "After a phone restart, open RSIS once to turn it back on.\n" +
                "• It is a supplementary warning tool, used at your own risk. It does not replace your attention " +
                "or emergency services.", 16f))
        col.addView(text("Your role", 18f, true))
        roleGroup = RadioGroup(this)
        Role.values().forEach { r ->
            roleGroup.addView(RadioButton(this).apply { id = 1000 + r.code; text = r.label; textSize = 17f })
        }
        roleGroup.check(1000 + (Prefs.role(this)?.code ?: Role.CYCLIST.code))
        col.addView(roleGroup)
        col.addView(Button(this).apply {
            text = "Turn on Bluetooth Mesh"
            setOnClickListener { onTurnOn() }
        }, LinearLayout.LayoutParams(-1, -2).apply { topMargin = 36 })
        col.addView(Button(this).apply {
            text = "Not now - GPS relay only"
            setOnClickListener { onboarding.visibility = View.GONE; refreshBanner() }
        })
        return ScrollView(this).apply { setBackgroundColor(0xFFFFFFFF.toInt()); addView(col) }
    }

    private fun onTurnOn() {
        Prefs.setRole(this, Role.fromCode(roleGroup.checkedRadioButtonId - 1000))
        if (hasCore()) { onboarding.visibility = View.GONE; enableBluetoothThenStart(); return }
        val perms = RsisService.requiredCorePermissions(this).toMutableList()
        if (Build.VERSION.SDK_INT >= 33) perms.add(Manifest.permission.POST_NOTIFICATIONS)
        permLauncher.launch(perms.toTypedArray())
        onboarding.visibility = View.GONE
    }

    /** Called by the web app's startOfflineSafety(). */
    private fun ensureReady() {
        if (Prefs.role(this) == null || !hasCore()) onboarding.visibility = View.VISIBLE else enableBluetoothThenStart()
    }

    private fun btOn(): Boolean =
        (getSystemService(BLUETOOTH_SERVICE) as BluetoothManager).adapter?.isEnabled == true

    /** Android 13+ cannot switch Bluetooth on silently: show the system's own "allow?" dialog. */
    @SuppressLint("MissingPermission")
    private fun enableBluetoothThenStart() {
        if (btOn()) startIfReady() else btLauncher.launch(Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE))
    }

    private fun startIfReady() {
        if (Prefs.role(this) != null && hasCore() && btOn()) RsisService.start(this)
        refreshBanner()
    }

    private fun onBannerTap() {
        when {
            Prefs.role(this) == null || !hasCore() -> if (deniedOnce) openAppSettings() else onboarding.visibility = View.VISIBLE
            !btOn() -> enableBluetoothThenStart()
        }
    }

    private fun refreshBanner() {
        val msg = when {
            Prefs.role(this) == null || !hasCore() ->
                if (deniedOnce) "Bluetooth Mesh is off (permission denied). GPS relay still works. Tap to open settings."
                else "Bluetooth Mesh is off. GPS relay still works. Tap to turn it on."
            !btOn() -> "Bluetooth is off - Mesh offline, you are NOT protected offline. GPS relay still works. Tap to turn Bluetooth on."
            RsisRuntime.issue != null -> "Bluetooth Mesh problem: ${RsisRuntime.issue}. GPS relay still works."
            RsisRuntime.syncIssue != null ->
                "Uploads paused: ${RsisRuntime.syncIssue}. Bluetooth detection and voice alerts still work."
            RsisRuntime.femaleVoice == false ->
                "No offline female voice is installed, so alerts use the default voice. Add one in your phone's text-to-speech settings."
            else -> null
        }
        if (!this::banner.isInitialized) return
        banner.text = msg ?: ""
        banner.visibility = if (msg == null) View.GONE else View.VISIBLE
    }

    // ---------- web ----------
    @SuppressLint("SetJavaScriptEnabled")
    private fun configureWeb() {
        web.settings.javaScriptEnabled = true
        web.settings.domStorageEnabled = true
        web.settings.setGeolocationEnabled(true)
        web.addJavascriptInterface(bridge, NativeBridge.JS_NAME)

        val docStart = WebViewFeature.isFeatureSupported(WebViewFeature.DOCUMENT_START_SCRIPT)
        if (docStart) WebViewCompat.addDocumentStartJavaScript(web, NativeBridge.SHIM, setOf(WEB_ORIGIN))

        web.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(v: WebView, r: WebResourceRequest): Boolean {
                val h = r.url.host ?: return true
                if (h == WEB_HOST || h.endsWith("stripe.com")) return false
                startActivity(Intent(Intent.ACTION_VIEW, r.url)); return true
            }
            override fun onPageStarted(v: WebView, url: String, f: Bitmap?) {
                bridge.currentHost = Uri.parse(url).host
                if (!docStart && bridge.currentHost == WEB_HOST) v.evaluateJavascript(NativeBridge.SHIM, null)
            }
        }
        web.webChromeClient = object : WebChromeClient() {
            override fun onGeolocationPermissionsShowPrompt(origin: String, cb: GeolocationPermissions.Callback) {
                cb.invoke(origin, origin == WEB_ORIGIN && hasFineLocation(), false)
            }
        }
    }

    private fun hasFineLocation() =
        ContextCompat.checkSelfPermission(this, Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED
    private fun hasCore() = RsisService.requiredCorePermissions(this).all {
        ContextCompat.checkSelfPermission(this, it) == PackageManager.PERMISSION_GRANTED
    }
    private fun openAppSettings() =
        startActivity(Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, Uri.fromParts("package", packageName, null)))
}
````

---

## `app/src/main/java/com/rsis/nativeapp/Prefs.kt`

````kotlin
package com.rsis.nativeapp

import android.content.Context

/** Only the anonymous role and a dB calibration offset. No identifiers. */
object Prefs {
    private fun sp(c: Context) = c.getSharedPreferences("rsis_prefs", Context.MODE_PRIVATE)
    fun role(c: Context): Role? = sp(c).getInt("role", -1).let { if (it < 0) null else Role.fromCode(it) }
    fun setRole(c: Context, r: Role) = sp(c).edit().putInt("role", r.code).apply()
    // --- pairing (upload credential only; detection never depends on it) ---
    fun installId(c: Context): String? = sp(c).getString("install_id", null)
    fun key(c: Context): String? = sp(c).getString("upload_key", null)
    fun pairedAt(c: Context): Long = sp(c).getLong("paired_at", 0L)
    fun savePairing(c: Context, installId: String, key: String, nowMs: Long) =
        sp(c).edit().putString("install_id", installId).putString("upload_key", key).putLong("paired_at", nowMs).apply()
    fun clearPairing(c: Context) = sp(c).edit().remove("install_id").remove("upload_key").remove("paired_at").apply()
    /** 16 random bytes. Not the Android ID, advertising ID, serial, or derived from any of them. */
    fun newInstallId(): String = ByteArray(16).also { java.security.SecureRandom().nextBytes(it) }.joinToString("") { "%02x".format(it) }

    fun calibrationDb(c: Context): Double = sp(c).getFloat("cal_db", 0f).toDouble()
}
````

---

## `app/src/main/java/com/rsis/nativeapp/ble/BleAdvertiser.kt`

````kotlin
package com.rsis.nativeapp.ble

import android.annotation.SuppressLint
import android.bluetooth.BluetoothManager
import android.bluetooth.le.*
import android.content.Context
import android.os.Handler
import android.os.Looper
import com.rsis.nativeapp.Role
import com.rsis.nativeapp.RsisRuntime
import com.rsis.nativeapp.Timing
import com.rsis.nativeapp.spb.Spb
import java.security.SecureRandom

/**
 * Advertises SPB v4.1. A fresh cryptographically random token every 3 s; never derived from any device ID,
 * never stored. Uses BLE 5 Coded PHY extended advertising where the hardware supports it, else legacy.
 * Caller must already hold BLUETOOTH_ADVERTISE (API 31+) / BLUETOOTH (<=30).
 */
@SuppressLint("MissingPermission")
class BleAdvertiser(
    private val ctx: Context,
    private val role: Role,
    private val riskSupplier: () -> Double,
    private val onIssue: (String) -> Unit,
) {
    private val rng = SecureRandom()
    private val handler = Handler(Looper.getMainLooper())
    private var advertiser: BluetoothLeAdvertiser? = null
    private var set: AdvertisingSet? = null
    private var legacyCb: AdvertiseCallback? = null
    private var useCoded = false
    @Volatile private var running = false

    @Volatile var currentTokenHex = ""; private set
    @Volatile var previousTokenHex = ""; private set

    fun start(): Boolean {
        val adapter = (ctx.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager).adapter
        if (adapter == null || !adapter.isEnabled) { onIssue("Bluetooth is off"); return false }
        advertiser = adapter.bluetoothLeAdvertiser
        if (advertiser == null) { onIssue("This phone cannot advertise over Bluetooth LE"); return false }
        useCoded = adapter.isLeCodedPhySupported && adapter.isLeExtendedAdvertisingSupported
        running = true
        if (useCoded) startCoded() else startLegacy()
        handler.postDelayed(rotation, Timing.TOKEN_ROTATION_MS)
        return true
    }

    fun stop() {
        running = false
        RsisRuntime.advertising = false
        handler.removeCallbacksAndMessages(null)
        try { legacyCb?.let { advertiser?.stopAdvertising(it) } } catch (_: Exception) {}
        try { advertiser?.stopAdvertisingSet(setCb) } catch (_: Exception) {}
        legacyCb = null; set = null
    }

    private val rotation = object : Runnable {
        override fun run() {
            if (!running) return
            if (useCoded) set?.setAdvertisingData(nextData(extended = true))
            else { try { legacyCb?.let { advertiser?.stopAdvertising(it) } } catch (_: Exception) {}; startLegacy() }
            handler.postDelayed(this, Timing.TOKEN_ROTATION_MS)
        }
    }

    private fun nextData(extended: Boolean): AdvertiseData {
        val token = ByteArray(Spb.TOKEN_SIZE).also { rng.nextBytes(it) }
        previousTokenHex = currentTokenHex
        currentTokenHex = token.joinToString("") { "%02x".format(it) }
        val pkt = Spb.encode(role, 0, token, riskSupplier())
        return AdvertiseData.Builder()
            .setIncludeDeviceName(false)
            // legacy: put TX power in the payload so scanners get it; extended carries it in the header
            .setIncludeTxPowerLevel(!extended)
            .addManufacturerData(Spb.COMPANY_ID, pkt)
            .build()
    }

    private fun startLegacy() {
        val settings = AdvertiseSettings.Builder()
            .setAdvertiseMode(AdvertiseSettings.ADVERTISE_MODE_LOW_LATENCY)
            .setTxPowerLevel(AdvertiseSettings.ADVERTISE_TX_POWER_HIGH)
            .setConnectable(false).setTimeout(0).build()
        val cb = object : AdvertiseCallback() {
            override fun onStartSuccess(settingsInEffect: AdvertiseSettings?) { RsisRuntime.advertising = true }
            override fun onStartFailure(errorCode: Int) { RsisRuntime.advertising = false; onIssue("Bluetooth advertising failed ($errorCode)") }
        }
        legacyCb = cb
        advertiser?.startAdvertising(settings, nextData(extended = false), cb)
    }

    private fun startCoded() {
        val params = AdvertisingSetParameters.Builder()
            .setLegacyMode(false).setConnectable(false).setScannable(false)
            .setPrimaryPhy(BluetoothDevice_PHY_LE_CODED).setSecondaryPhy(BluetoothDevice_PHY_LE_CODED)
            .setInterval(AdvertisingSetParameters.INTERVAL_LOW)
            .setTxPowerLevel(AdvertisingSetParameters.TX_POWER_HIGH)
            .setIncludeTxPower(true)
            .build()
        advertiser?.startAdvertisingSet(params, nextData(extended = true), null, null, null, setCb)
    }

    private val setCb = object : AdvertisingSetCallback() {
        override fun onAdvertisingSetStarted(advertisingSet: AdvertisingSet?, txPower: Int, status: Int) {
            if (status == ADVERTISE_SUCCESS) { set = advertisingSet; RsisRuntime.advertising = true }
            else { // fall back to standard BLE
                useCoded = false
                if (running) startLegacy()
            }
        }
    }

    private companion object {
        const val BluetoothDevice_PHY_LE_CODED = android.bluetooth.BluetoothDevice.PHY_LE_CODED
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/ble/BleScanner.kt`

````kotlin
package com.rsis.nativeapp.ble

import android.annotation.SuppressLint
import android.bluetooth.BluetoothManager
import android.bluetooth.le.*
import android.content.Context
import android.os.SystemClock
import com.rsis.nativeapp.RsisRuntime
import com.rsis.nativeapp.engine.Sighting
import com.rsis.nativeapp.spb.Spb

/**
 * Continuous scan filtered to SPB v4.1. Anything that isn't a valid 15-byte, CRC-correct v4.1 packet
 * (other apps, the obsolete 10-byte format, corrupted frames) is dropped before the engine sees it.
 * Caller must already hold BLUETOOTH_SCAN.
 */
@SuppressLint("MissingPermission")
class BleScanner(
    private val ctx: Context,
    private val isOwnToken: (String) -> Boolean,
    private val onSighting: (Sighting) -> Unit,
    private val onIssue: (String) -> Unit,
) {
    private var scanner: BluetoothLeScanner? = null

    fun start(): Boolean {
        val adapter = (ctx.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager).adapter
        if (adapter == null || !adapter.isEnabled) { onIssue("Bluetooth is off"); return false }
        scanner = adapter.bluetoothLeScanner ?: run { onIssue("Bluetooth LE scanner unavailable"); return false }
        val filter = ScanFilter.Builder()
            .setManufacturerData(Spb.COMPANY_ID, byteArrayOf(Spb.VERSION.toByte()), byteArrayOf(0xFF.toByte()))
            .build()
        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)
            .setLegacy(false)                       // see extended (Coded PHY) AND legacy adverts
            .setPhy(ScanSettings.PHY_LE_ALL_SUPPORTED)
            .build()
        scanner?.startScan(listOf(filter), settings, cb)
        RsisRuntime.scanning = true
        return true
    }

    fun stop() { RsisRuntime.scanning = false; try { scanner?.stopScan(cb) } catch (_: Exception) {} }

    private val cb = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, r: ScanResult) {
            val raw = r.scanRecord?.getManufacturerSpecificData(Spb.COMPANY_ID)
            val pkt = Spb.decode(raw) ?: return
            if (isOwnToken(pkt.tokenHex)) return
            val tx = r.txPower.takeIf { it != ScanResult.TX_POWER_NOT_PRESENT }
            onSighting(Sighting(pkt, r.rssi, tx, SystemClock.elapsedRealtime()))
        }
        override fun onScanFailed(errorCode: Int) { RsisRuntime.scanning = false; onIssue("Bluetooth scan failed ($errorCode)") }
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/bridge/NativeBridge.kt`

````kotlin
package com.rsis.nativeapp.bridge

import android.content.Context
import android.content.pm.PackageManager
import android.webkit.JavascriptInterface
import androidx.core.content.ContextCompat
import com.rsis.nativeapp.Prefs
import com.rsis.nativeapp.RsisRuntime
import com.rsis.nativeapp.service.RsisService

const val WEB_HOST = "rsis-road-safety-inelligence-system.base44.app"
const val WEB_ORIGIN = "https://$WEB_HOST"

/**
 * Backing object for window.RSISNative (spec §7). The three exposed members keep their exact names via SHIM.
 * Calls are honoured only while the page being shown is the RSIS web app's own origin.
 */
class NativeBridge(
    private val ctx: Context,
    private val requestStart: () -> Unit, // runs on UI thread: role/permission flow, then starts the service
) {
    @Volatile var currentHost: String? = null

    private fun trusted() = currentHost == WEB_HOST

    /** Synchronous, cheap, never a Promise. */
    @JavascriptInterface
    fun getBleStatus(): String = status()

    fun status(): String = when {
        RsisRuntime.running -> "running"
        Prefs.role(ctx) == null || !RsisService.requiredCorePermissions(ctx).all {
            ContextCompat.checkSelfPermission(ctx, it) == PackageManager.PERMISSION_GRANTED
        } -> "permission_required"
        else -> "installed"
    }

    /**
     * Always empty. The native layer uploads its own queue (it holds the upload key; the web SDK does not).
     * Returning events here would make the web app re-upload them without the key and loop on 401.
     */
    @JavascriptInterface
    fun getQueuedEvents(): String = "[]"


    /** Additive 4th member: the role chosen on the native welcome screen, so the web app need not ask again. */
    @JavascriptInterface
    fun getRole(): String = if (!trusted()) "" else (Prefs.role(ctx)?.label ?: "")

    @JavascriptInterface
    fun startOfflineSafety(): String {
        if (!trusted()) return status()
        requestStart()
        return status()
    }

    companion object {
        const val JS_NAME = "__rsisBridge"
        /** Defines window.RSISNative before page scripts run (document-start script). */
        val SHIM = """
            (function(){
              if (window.RSISNative) return;
              var b = window.$JS_NAME;
              window.RSISNative = {
                getBleStatus: function(){ return String(b.getBleStatus()); },
                getRole: function(){ return String(b.getRole()); },
                getQueuedEvents: function(){ return Promise.resolve().then(function(){ return JSON.parse(b.getQueuedEvents()); }); },
                startOfflineSafety: function(){ return Promise.resolve().then(function(){ return b.startOfflineSafety(); }); }
              };
            })();
        """.trimIndent()
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/engine/DistanceEstimator.kt`

````kotlin
package com.rsis.nativeapp.engine

import kotlin.math.pow
import kotlin.math.roundToInt

/**
 * RSSI -> feet. Uses the advertised TX power when the radio reports one, else a default RSSI@1m,
 * plus a per-device dB calibration offset. Output is rounded feet. The estimate is a *measurement*;
 * it is never used to size an alert radius (spec §2.4).
 * Metres exist only inside this function.
 */
class DistanceEstimator(
    private val calibrationDb: Double = 0.0,
    private val pathLossExponent: Double = 2.2,
) {
    fun feet(rssiDbm: Double, advertisedTxPowerDbm: Int?): Int {
        val rssiAt1m = (advertisedTxPowerDbm?.let { it - FREE_SPACE_LOSS_1M_DB } ?: DEFAULT_RSSI_AT_1M) + calibrationDb
        val meters = 10.0.pow((rssiAt1m - rssiDbm) / (10.0 * pathLossExponent))
        return (meters * 3.28084).roundToInt().coerceIn(0, 10_000)
    }

    companion object {
        const val DEFAULT_RSSI_AT_1M = -59.0
        const val FREE_SPACE_LOSS_1M_DB = 41.0 // ~2.4 GHz at 1 m
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/engine/SafetyEngine.kt`

````kotlin
package com.rsis.nativeapp.engine

import com.rsis.nativeapp.*
import com.rsis.nativeapp.spb.SpbPacket
import java.security.SecureRandom
import kotlin.math.abs

class Sighting(val packet: SpbPacket, val rssi: Int, val txPowerDbm: Int?, val atMs: Long)

class AlertDecision(
    val eventId: String,
    val role: Role,          // role of the detected road user
    val tier: Tier,
    val distanceFt: Int,
    val risk: Double,        // collision probability 0.0..1.0
    val message: String,
    val fallback: Boolean,   // true = the 250 ft fallback repeat of an already-recorded event
)

interface AlertListener {
    /** Deliver the alert on-device. Return true only if the user was actually alerted. */
    fun onAlert(d: AlertDecision): Boolean
    fun onDenseTraffic(active: Boolean)
}

/**
 * FAST-PHYSICS + SAFETY_ALERT — fully offline, in-memory only.
 * Fed by BLE sightings; owns tracks, tiering, cooldown/escalation, 250 ft fallback and cluster suppression.
 * Thread-safe via a single lock. Clock is injected for tests (use a monotonic clock in production).
 */
class SafetyEngine(
    private val clock: () -> Long,
    private val estimator: DistanceEstimator,
    private val listener: AlertListener,
) {
    private class Track(val role: Role) {
        var lastSeen = 0L
        var sampleCount = 0
        var smoothedRssi = 0.0
        var txPower: Int? = null
        var advRisk = 0.0
        var feet = 0
        var closingMph: Double? = null
        var lastP = 0.0
        val samples = ArrayDeque<Pair<Long, Int>>()
        var lastAlertAt: Long? = null
        var lastAlertTier: Tier? = null
        var lastEventId: String? = null
        var fallbackDueAt: Long? = null
        var pruned = false
    }

    private val tracks = ArrayList<Track>()
    private val byToken = HashMap<String, Track>()
    private var dense = false
    private var denseRecomputeAt = 0L
    private val rng = SecureRandom()

    @Synchronized
    fun onSighting(s: Sighting) {
        val now = s.atMs
        val key = s.packet.tokenHex
        var t = byToken[key]
        if (t == null) {
            val rawFt = estimator.feet(s.rssi.toDouble(), s.txPowerDbm)
            t = associate(s.packet.role, rawFt, now) ?: Track(s.packet.role).also { tracks.add(it) }
            byToken[key] = t
        }
        update(t, s, now)
        evaluate(t, now)
    }

    /** Call about once per second. */
    @Synchronized
    fun tick(ownSpeedMph: Double?) {
        val now = clock()
        // 250 ft fallback
        for (t in tracks) {
            val due = t.fallbackDueAt ?: continue
            if (now < due) continue
            t.fallbackDueAt = null
            val tier = t.lastAlertTier ?: continue
            // Reminder on the SAME event: still heard, still inside 250 ft, and not moving away.
            val notReceding = t.closingMph?.let { it > -APPROACH_MPH } ?: true
            if (t.feet <= AlertPerimeters.BLE_FALLBACK_ALERT_FT && now - t.lastSeen <= 3_000 && notReceding && !dense) {
                deliver(t, tier, now, fallback = true, eventId = t.lastEventId!!)
            }
        }
        // cluster / dense traffic
        if (!dense || now >= denseRecomputeAt) {
            val near = tracks.count {
                now - it.lastSeen <= 3_000 && it.feet <= ClusterRule.RADIUS_FT &&
                    it.closingMph?.let { c -> abs(c) < ClusterRule.MAX_MPH } == true
            }
            val shouldBeDense = ownSpeedMph != null && ownSpeedMph < ClusterRule.MAX_MPH && near >= ClusterRule.MIN_NODES
            if (shouldBeDense != dense) {
                dense = shouldBeDense
                listener.onDenseTraffic(dense)
            }
            if (dense) denseRecomputeAt = now + Timing.CLUSTER_RECOMPUTE_MS
        }
        // prune after the cooldown window has fully elapsed
        val it = tracks.iterator()
        while (it.hasNext()) {
            val t = it.next()
            if (now - t.lastSeen > Timing.COOLDOWN_MS + 5_000) { t.pruned = true; it.remove() }
        }
        byToken.entries.removeAll { it.value.pruned }
    }

    /** Highest current collision probability among recently-heard nodes; advertised as our own risk. */
    @Synchronized
    fun currentRisk(): Double {
        val now = clock()
        return tracks.filter { now - it.lastSeen <= 3_000 }.maxOfOrNull { it.lastP } ?: 0.0
    }

    // ---- internals ----

    /**
     * Tokens rotate every 3 s and nothing may survive a rotation (spec §2.5), so the only way to keep one
     * per-user cooldown is *local, in-memory* kinematic association: a new token is attached to a recently
     * heard track of the same role whose distance is continuous. Never persisted, never sent anywhere.
     */
    private fun associate(role: Role, rawFt: Int, now: Long): Track? {
        // If more than one track could own this token, do NOT guess: treat it as a new user.
        // Wrong in that direction costs an extra alert, never a missed one.
        var match: Track? = null
        var n = 0
        for (t in tracks) {
            if (t.role != role || now - t.lastSeen > ASSOC_WINDOW_MS) continue
            if (abs(t.feet - rawFt) <= maxOf(25, t.feet / 2)) { match = t; n++ }
        }
        return if (n == 1) match else null
    }

    private fun update(t: Track, s: Sighting, now: Long) {
        t.lastSeen = now
        t.smoothedRssi = if (t.sampleCount == 0) s.rssi.toDouble() else 0.7 * t.smoothedRssi + 0.3 * s.rssi
        t.sampleCount++
        t.txPower = s.txPowerDbm ?: t.txPower
        t.advRisk = s.packet.risk
        t.feet = estimator.feet(t.smoothedRssi, t.txPower)
        if (t.samples.isEmpty() || now - t.samples.last().first >= 250) t.samples.addLast(now to t.feet)
        while (t.samples.isNotEmpty() && now - t.samples.first().first > 8_000) t.samples.removeFirst()
        t.closingMph = closingMph(t.samples)
    }

    private fun closingMph(s: ArrayDeque<Pair<Long, Int>>): Double? {
        if (s.size < 4 || s.last().first - s.first().first < 2_000) return null
        val t0 = s.first().first
        val xs = s.map { (it.first - t0) / 1000.0 }
        val ys = s.map { it.second.toDouble() }
        val mx = xs.average(); val my = ys.average()
        var num = 0.0; var den = 0.0
        for (i in xs.indices) { num += (xs[i] - mx) * (ys[i] - my); den += (xs[i] - mx) * (xs[i] - mx) }
        if (den == 0.0) return null
        return -(num / den) * FPS_TO_MPH // positive = closing
    }

    private fun collisionProbability(feet: Int, closingMph: Double?, advRisk: Double): Double {
        if (closingMph == null || closingMph <= 0.0) return 0.0
        val ttc = feet / (closingMph / FPS_TO_MPH)
        val timeFactor = (1.0 - ttc / TTC_HORIZON_S).coerceIn(0.0, 1.0)
        val prox = (1.0 - feet / 164.0).coerceIn(0.0, 1.0)
        return ((0.7 * timeFactor + 0.3 * prox) * (0.8 + 0.2 * advRisk)).coerceIn(0.0, 1.0)
    }

    private fun evaluate(t: Track, now: Long) {
        val c = t.closingMph
        t.lastP = collisionProbability(t.feet, c, t.advRisk)
        if (dense) return
        if (c != null && c < -APPROACH_MPH) return          // receding: suppressed
        val band = Band.of(t.feet)
        val approaching = c != null && c > APPROACH_MPH
        val tier = when {
            t.feet > AlertPerimeters.BLE_MESH_ALERT_FT -> Tier.WHITE
            t.lastP >= ORANGE_P || (band == Band.CRITICAL && approaching) -> Tier.ORANGE
            approaching || band == Band.CRITICAL || band == Band.HIGH -> Tier.YELLOW
            else -> Tier.WHITE
        }
        if (tier == Tier.WHITE) return
        val last = t.lastAlertAt
        val lastTier = t.lastAlertTier
        if (last != null && now - last < Timing.COOLDOWN_MS && lastTier != null && tier.ordinal <= lastTier.ordinal) return
        deliver(t, tier, now, fallback = false, eventId = newEventId())
    }

    private fun deliver(t: Track, tier: Tier, now: Long, fallback: Boolean, eventId: String) {
        val r = maxOf(5, ((t.feet + 2) / 5) * 5)
        val msg = if (tier == Tier.ORANGE) "Warning. ${t.role.spoken} on a collision path, about $r feet."
        else "Caution. ${t.role.spoken} nearby, about $r feet."
        listener.onAlert(AlertDecision(eventId, t.role, tier, t.feet, t.lastP, msg, fallback))
        if (!fallback) {
            t.lastAlertAt = now
            t.lastAlertTier = tier
            t.lastEventId = eventId
            t.fallbackDueAt = now + Timing.FALLBACK_WAIT_MS
        }
    }

    private fun newEventId(): String {
        val chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
        return "RSIS-LOCAL-" + (1..8).map { chars[rng.nextInt(chars.length)] }.joinToString("")
    }

    companion object {
        const val ASSOC_WINDOW_MS = 4_500L
        const val APPROACH_MPH = 1.0
        const val ORANGE_P = 0.6
        const val TTC_HORIZON_S = 8.0
        const val FPS_TO_MPH = 0.681818
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/service/RsisService.kt`

````kotlin
package com.rsis.nativeapp.service

import android.Manifest
import android.annotation.SuppressLint
import android.app.*
import android.bluetooth.BluetoothAdapter
import android.content.BroadcastReceiver
import android.content.Context
import android.content.IntentFilter
import android.content.Intent
import android.content.pm.PackageManager
import android.content.pm.ServiceInfo
import android.location.Location
import android.location.LocationListener
import android.location.LocationManager
import android.net.ConnectivityManager
import android.net.Network
import android.net.NetworkCapabilities
import android.os.Bundle
import android.os.IBinder
import android.os.SystemClock
import androidx.core.app.NotificationCompat
import androidx.core.app.ServiceCompat
import androidx.core.content.ContextCompat
import com.rsis.nativeapp.*
import com.rsis.nativeapp.ble.BleAdvertiser
import com.rsis.nativeapp.ble.BleScanner
import com.rsis.nativeapp.engine.*
import com.rsis.nativeapp.store.EventPayloads
import com.rsis.nativeapp.store.EventStore
import com.rsis.nativeapp.sync.SyncClient
import com.rsis.nativeapp.voice.VoiceAlerts
import kotlinx.coroutines.*

/** Foreground service hosting advertise + scan + engine + voice + queue + sync. Survives screen lock. */
@SuppressLint("MissingPermission")
class RsisService : Service(), AlertListener, LocationListener {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    private lateinit var voice: VoiceAlerts
    private lateinit var store: EventStore
    private lateinit var sync: SyncClient
    private lateinit var engine: SafetyEngine
    private var adv: BleAdvertiser? = null
    private var scan: BleScanner? = null
    private lateinit var ownRole: Role
    @Volatile private var lastLoc: Location? = null
    private var netCb: ConnectivityManager.NetworkCallback? = null

    override fun onBind(i: Intent?): IBinder? = null

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        if (RsisRuntime.serviceUp) return START_STICKY
        val role = Prefs.role(this)
        if (role == null || !hasPermissions()) { stopSelf(); return START_NOT_STICKY }
        ownRole = role
        startForegroundCompat()

        voice = VoiceAlerts(this)
        store = EventStore(this)
        sync = SyncClient(this, store)
        engine = SafetyEngine({ SystemClock.elapsedRealtime() }, DistanceEstimator(Prefs.calibrationDb(this)), this)

        if (!startRadios()) { shutdown(); stopSelf(); return START_NOT_STICKY }
        registerReceiver(btReceiver, IntentFilter(BluetoothAdapter.ACTION_STATE_CHANGED))
        btReceiverRegistered = true

        startLocation()
        registerNetwork()
        RsisRuntime.issue = null
        RsisRuntime.serviceUp = true
        scope.launch {
            var n = 0
            while (isActive) {
                delay(1_000)
                engine.tick(ownSpeedMph())
                if (++n % 60 == 0 && store.count() > 0 && isOnline()) sync.flush()
            }
        }
        scope.launch { if (isOnline()) sync.flush() }
        return START_STICKY
    }

    // ---- radios + Bluetooth state ----
    private var btReceiverRegistered = false

    private fun startRadios(): Boolean {
        val issue = { m: String -> RsisRuntime.issue = m }
        val a = BleAdvertiser(this, ownRole, { engine.currentRisk() }, issue)
        val sc = BleScanner(
            this,
            isOwnToken = { it == a.currentTokenHex || it == a.previousTokenHex },
            onSighting = { engine.onSighting(it) },
            onIssue = issue,
        )
        adv = a; scan = sc
        val ok = a.start() && sc.start()
        if (!ok) { a.stop(); sc.stop() }
        return ok
    }

    private fun stopRadios() {
        try { adv?.stop() } catch (_: Exception) {}
        try { scan?.stop() } catch (_: Exception) {}
    }

    /** Mid-ride Bluetooth toggles: never keep claiming protection when the radio is off. */
    private val btReceiver = object : BroadcastReceiver() {
        override fun onReceive(c: Context, i: Intent) {
            when (i.getIntExtra(BluetoothAdapter.EXTRA_STATE, -1)) {
                BluetoothAdapter.STATE_TURNING_OFF, BluetoothAdapter.STATE_OFF -> {
                    stopRadios()
                    RsisRuntime.advertising = false
                    RsisRuntime.scanning = false
                    RsisRuntime.issue = "Bluetooth is off"
                    getSystemService(NotificationManager::class.java).notify(NOTIF_ID, notification("Bluetooth is OFF - Mesh offline"))
                }
                BluetoothAdapter.STATE_ON -> {
                    if (startRadios()) {
                        RsisRuntime.issue = null
                        getSystemService(NotificationManager::class.java).notify(NOTIF_ID, notification("RSIS Bluetooth Mesh is active"))
                    }
                }
            }
        }
    }

    // ---- AlertListener ----
    override fun onAlert(d: AlertDecision): Boolean {
        val delivered = voice.speak(d.message)
        if (!d.fallback) { // fallback repeats the same physical event; it never creates a second record
            val online = isOnline()
            val l = lastLoc
            store.add(
                d.eventId,
                EventPayloads.build(d, ownRole, delivered, if (online) "online_at_detection" else "offline_at_detection", l?.latitude, l?.longitude),
            )
            if (online) scope.launch { sync.flush() }
        }
        return delivered
    }

    override fun onDenseTraffic(active: Boolean) {
        getSystemService(NotificationManager::class.java).notify(NOTIF_ID, notification(if (active) "DENSE TRAFFIC" else "RSIS Bluetooth Mesh is active"))
    }

    // ---- location (own speed + event coordinates) ----
    private fun startLocation() {
        val lm = getSystemService(LocationManager::class.java)
        try { lm.requestLocationUpdates(LocationManager.GPS_PROVIDER, 1_000L, 0f, this) } catch (_: Exception) {}
        lastLoc = try { lm.getLastKnownLocation(LocationManager.GPS_PROVIDER) } catch (_: Exception) { null }
    }
    override fun onLocationChanged(location: Location) { lastLoc = location }
    override fun onProviderEnabled(provider: String) {}
    override fun onProviderDisabled(provider: String) {}
    @Deprecated("Deprecated in Java") override fun onStatusChanged(p: String?, s: Int, e: Bundle?) {}

    private fun ownSpeedMph(): Double? {
        val l = lastLoc ?: return null
        val ageMs = (SystemClock.elapsedRealtimeNanos() - l.elapsedRealtimeNanos) / 1_000_000
        return if (l.hasSpeed() && ageMs < 5_000) l.speed * 2.23694 else null
    }

    // ---- connectivity ----
    private fun isOnline(): Boolean {
        val cm = getSystemService(ConnectivityManager::class.java)
        val caps = cm.getNetworkCapabilities(cm.activeNetwork) ?: return false
        return caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)
    }
    private fun registerNetwork() {
        val cm = getSystemService(ConnectivityManager::class.java)
        val cb = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { scope.launch { delay(1_500); sync.flush() } }
        }
        netCb = cb
        cm.registerDefaultNetworkCallback(cb)
    }

    // ---- lifecycle ----
    private fun hasPermissions() = requiredCorePermissions(this).all {
        ContextCompat.checkSelfPermission(this, it) == PackageManager.PERMISSION_GRANTED
    }

    private fun shutdown() {
        RsisRuntime.serviceUp = false
        RsisRuntime.advertising = false
        RsisRuntime.scanning = false
        scope.coroutineContext[Job]?.cancelChildren()
        stopRadios()
        if (btReceiverRegistered) { try { unregisterReceiver(btReceiver) } catch (_: Exception) {}; btReceiverRegistered = false }
        try { getSystemService(LocationManager::class.java).removeUpdates(this) } catch (_: Exception) {}
        try { netCb?.let { getSystemService(ConnectivityManager::class.java).unregisterNetworkCallback(it) } } catch (_: Exception) {}
        if (this::voice.isInitialized) voice.shutdown()
    }

    override fun onDestroy() { shutdown(); scope.cancel(); super.onDestroy() }

    private fun startForegroundCompat() {
        val ch = NotificationChannel(CHANNEL, "RSIS safety", NotificationManager.IMPORTANCE_LOW)
        getSystemService(NotificationManager::class.java).createNotificationChannel(ch)
        ServiceCompat.startForeground(
            this, NOTIF_ID, notification("RSIS Bluetooth Mesh is active"),
            ServiceInfo.FOREGROUND_SERVICE_TYPE_CONNECTED_DEVICE or ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION,
        )
    }

    private fun notification(text: String): Notification {
        val pi = PendingIntent.getActivity(this, 0, Intent(this, MainActivity::class.java), PendingIntent.FLAG_IMMUTABLE)
        return NotificationCompat.Builder(this, CHANNEL)
            .setSmallIcon(android.R.drawable.stat_sys_data_bluetooth)
            .setContentTitle("RSIS").setContentText(text)
            .setOngoing(true).setContentIntent(pi).build()
    }

    companion object {
        private const val CHANNEL = "rsis_safety"
        private const val NOTIF_ID = 62
        fun start(ctx: Context) = ContextCompat.startForegroundService(ctx, Intent(ctx, RsisService::class.java))

        /** Permissions without which Bluetooth Mesh cannot run. */
        fun requiredCorePermissions(ctx: Context): List<String> = buildList {
            if (android.os.Build.VERSION.SDK_INT >= 31) {
                add(Manifest.permission.BLUETOOTH_SCAN); add(Manifest.permission.BLUETOOTH_ADVERTISE); add(Manifest.permission.BLUETOOTH_CONNECT)
            }
            add(Manifest.permission.ACCESS_FINE_LOCATION)
        }
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/spb/Spb.kt`

````kotlin
package com.rsis.nativeapp.spb

import com.rsis.nativeapp.Role

/** Decoded SPB v4.1 packet. */
class SpbPacket(val role: Role, val statusBits: Int, val token: ByteArray, val riskScaled: Int) {
    val risk: Double get() = riskScaled / 10000.0
    val tokenHex: String get() = token.joinToString("") { "%02x".format(it) }
}

/**
 * Safety Presence Beacon v4.1 — 15 bytes.
 *  0     version      0x41
 *  1     role_flags   bits0-1 role (0 Cyclist,1 Pedestrian,2 Vehicle,3 Passenger); bits2-7 status (reserved = 0)
 *  2-9   token        8 bytes, rotates every 3 s
 *  10-12 risk_scaled  risk*10000, big-endian
 *  13-14 crc16        CRC-16/CCITT-FALSE (poly 0x1021, init 0xFFFF) over bytes 0..12, big-endian
 *
 * Byte order, CRC variant, and the status-bit layout are NOT specified in the brief; these are the
 * choices made here and must be confirmed against the web/server side.
 */
object Spb {
    const val VERSION = 0x41
    const val SIZE = 15
    const val TOKEN_SIZE = 8
    const val MAX_RISK_SCALED = 10_000
    /** 0xFFFF is the Bluetooth SIG "reserved for testing" company ID. Replace before shipping. */
    const val COMPANY_ID = 0xFFFF

    fun crc16(data: ByteArray, length: Int = data.size): Int {
        var crc = 0xFFFF
        for (i in 0 until length) {
            crc = crc xor ((data[i].toInt() and 0xFF) shl 8)
            repeat(8) {
                crc = if (crc and 0x8000 != 0) (crc shl 1) xor 0x1021 else crc shl 1
                crc = crc and 0xFFFF
            }
        }
        return crc
    }

    fun encode(role: Role, statusBits: Int, token: ByteArray, risk: Double): ByteArray {
        require(token.size == TOKEN_SIZE) { "token must be 8 bytes" }
        val scaled = Math.round(risk.coerceIn(0.0, 1.0) * 10000.0).toInt()
        val b = ByteArray(SIZE)
        b[0] = VERSION.toByte()
        b[1] = ((role.code and 0x03) or ((statusBits and 0x3F) shl 2)).toByte()
        System.arraycopy(token, 0, b, 2, TOKEN_SIZE)
        b[10] = ((scaled shr 16) and 0xFF).toByte()
        b[11] = ((scaled shr 8) and 0xFF).toByte()
        b[12] = (scaled and 0xFF).toByte()
        val crc = crc16(b, 13)
        b[13] = ((crc shr 8) and 0xFF).toByte()
        b[14] = (crc and 0xFF).toByte()
        return b
    }

    /** Returns null for anything that isn't a valid v4.1 packet, including the obsolete 10-byte format. */
    fun decode(raw: ByteArray?): SpbPacket? {
        if (raw == null || raw.size != SIZE) return null
        if ((raw[0].toInt() and 0xFF) != VERSION) return null
        val crcRx = ((raw[13].toInt() and 0xFF) shl 8) or (raw[14].toInt() and 0xFF)
        if (crc16(raw, 13) != crcRx) return null
        val scaled = ((raw[10].toInt() and 0xFF) shl 16) or ((raw[11].toInt() and 0xFF) shl 8) or (raw[12].toInt() and 0xFF)
        if (scaled > MAX_RISK_SCALED) return null
        val flags = raw[1].toInt() and 0xFF
        return SpbPacket(Role.fromCode(flags and 0x03), (flags shr 2) and 0x3F, raw.copyOfRange(2, 10), scaled)
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/store/EventStore.kt`

````kotlin
package com.rsis.nativeapp.store

import android.content.ContentValues
import android.content.Context
import android.database.sqlite.SQLiteDatabase
import android.database.sqlite.SQLiteOpenHelper
import com.rsis.nativeapp.engine.AlertDecision
import com.rsis.nativeapp.Role
import org.json.JSONObject
import java.time.Instant

/** Durable event queue. Event records only — never notifications (spec §6c). Survives app kill and reboot. */
class EventStore(ctx: Context) : SQLiteOpenHelper(ctx.applicationContext, "rsis_events.db", null, 1) {
    class Item(val eventId: String, val payload: JSONObject)

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL("CREATE TABLE events(seq INTEGER PRIMARY KEY AUTOINCREMENT, event_id TEXT UNIQUE NOT NULL, payload TEXT NOT NULL)")
    }
    override fun onUpgrade(db: SQLiteDatabase, o: Int, n: Int) {}

    @Synchronized fun add(eventId: String, payload: JSONObject) {
        val cv = ContentValues().apply { put("event_id", eventId); put("payload", payload.toString()) }
        writableDatabase.insertWithOnConflict("events", null, cv, SQLiteDatabase.CONFLICT_IGNORE)
    }

    /** Oldest first. */
    @Synchronized fun pending(limit: Int = 50): List<Item> {
        val out = ArrayList<Item>()
        readableDatabase.rawQuery("SELECT event_id,payload FROM events ORDER BY seq ASC LIMIT $limit", null).use { c ->
            while (c.moveToNext()) out.add(Item(c.getString(0), JSONObject(c.getString(1))))
        }
        return out
    }

    @Synchronized fun remove(eventId: String) { writableDatabase.delete("events", "event_id=?", arrayOf(eventId)) }

    @Synchronized fun count(): Int =
        readableDatabase.rawQuery("SELECT COUNT(*) FROM events", null).use { it.moveToFirst(); it.getInt(0) }

    /** Hand everything to the web app (it syncs them itself), then clear. One transaction. */
    @Synchronized fun drain(): List<JSONObject> {
        val db = writableDatabase
        db.beginTransaction()
        try {
            val all = ArrayList<JSONObject>()
            db.rawQuery("SELECT payload FROM events ORDER BY seq ASC", null).use { c ->
                while (c.moveToNext()) all.add(JSONObject(c.getString(0)))
            }
            db.delete("events", null, null)
            db.setTransactionSuccessful()
            return all
        } finally { db.endTransaction() }
    }
}

/** Builds the exact §6a payload. Optional fields that are unknown are omitted — never faked. */
object EventPayloads {
    fun build(
        d: AlertDecision, ownRole: Role, delivered: Boolean, connectivity: String,
        lat: Double?, lon: Double?, nowMs: Long = System.currentTimeMillis(),
    ): JSONObject = JSONObject().apply {
        put("event_id", d.eventId)
        put("source", if (connectivity == "offline_at_detection") "native_offline_ble" else "native_hybrid")
        put("event_type", "SAFETY_ALERT")
        put("role", ownRole.label)
        put("risk", Math.round(d.risk * 10000.0) / 10000.0)
        put("local_distance_ft", d.distanceFt)
        put("local_alert_delivered", delivered)
        put("online_alert_allowed", !delivered)
        put("connectivity_state", connectivity)
        if (lat != null && lon != null) { put("latitude", lat); put("longitude", lon) }
        put("alert_message", d.message)
        put("timestamp", Instant.ofEpochMilli(nowMs).toString())
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/sync/Http.kt`

````kotlin
package com.rsis.nativeapp.sync

import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL

object Http {
    const val BASE = "https://rsis-road-safety-inelligence-system.base44.app/functions"

    fun postJson(url: String, json: JSONObject, headers: Map<String, String> = emptyMap()): Pair<Int, String> {
        val c = URL(url).openConnection() as HttpURLConnection
        try {
            c.requestMethod = "POST"; c.connectTimeout = 10_000; c.readTimeout = 10_000; c.doOutput = true
            c.setRequestProperty("Content-Type", "application/json")
            headers.forEach { (k, v) -> c.setRequestProperty(k, v) }
            c.outputStream.use { it.write(json.toString().toByteArray()) }
            val code = c.responseCode
            val stream = if (code in 200..299) c.inputStream else c.errorStream
            return code to (stream?.bufferedReader()?.use { it.readText() } ?: "")
        } finally { c.disconnect() }
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/sync/PairingClient.kt`

````kotlin
package com.rsis.nativeapp.sync

import android.content.Context
import com.rsis.nativeapp.NativeProtocol
import com.rsis.nativeapp.Prefs
import com.rsis.nativeapp.Timing
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import org.json.JSONObject
import java.io.IOException

sealed class PairResult {
    class Ok(val key: String) : PairResult()
    object Revoked : PairResult()      // support unpaired this install
    object StaleBuild : PairResult()   // server refused this protocol/packet version
    object Unreachable : PairResult()  // network problem: try again later
    class Rejected(val code: Int) : PairResult()
}

/**
 * UNVERIFIED CONTRACT. The request/response shape below is taken from Base44's description of `nativePair`
 * (install_id in, key out, 409 = stale build, 403 = revoked). Field names and the key header must be checked
 * against the real function before the first pairing test. Everything contract-specific lives in this object.
 */
object PairingContract {
    const val PATH = "/nativePair"
    const val KEY_HEADER = "X-RSIS-Key"
    const val REQ_INSTALL_ID = "install_id"
    const val REQ_ROLE = "role"
    const val REQ_PROTOCOL = "protocol"
    const val REQ_PACKET_VERSION = "packet_version"
    const val RES_KEY = "key"
}

/**
 * Pairing only gates UPLOADS. Offline detection, voice alerts and queuing never wait for it.
 * install_id and key are random, device-local, and re-issued every Timing.REPAIR_AFTER_MS.
 */
class PairingClient(private val ctx: Context) {
    suspend fun ensurePaired(): PairResult = withContext(Dispatchers.IO) {
        val now = System.currentTimeMillis()
        val key = Prefs.key(ctx)
        val at = Prefs.pairedAt(ctx)
        if (key != null && at in 1..now && now - at < Timing.REPAIR_AFTER_MS) return@withContext PairResult.Ok(key)

        Prefs.clearPairing(ctx) // expired or missing: start a fresh, unlinkable identity
        val installId = Prefs.newInstallId()
        val body = JSONObject()
            .put(PairingContract.REQ_INSTALL_ID, installId)
            .put(PairingContract.REQ_ROLE, Prefs.role(ctx)?.label ?: "")
            .put(PairingContract.REQ_PROTOCOL, NativeProtocol.PROTOCOL)
            .put(PairingContract.REQ_PACKET_VERSION, NativeProtocol.PACKET_VERSION)
        try {
            val (code, text) = Http.postJson(Http.BASE + PairingContract.PATH, body)
            when {
                code in 200..299 -> {
                    val k = JSONObject(text).optString(PairingContract.RES_KEY, "")
                    if (k.isEmpty()) PairResult.Rejected(code)
                    else { Prefs.savePairing(ctx, installId, k, now); PairResult.Ok(k) }
                }
                code == 409 -> PairResult.StaleBuild
                code == 403 -> PairResult.Revoked
                else -> PairResult.Rejected(code)
            }
        } catch (_: IOException) { PairResult.Unreachable } catch (_: org.json.JSONException) { PairResult.Rejected(-1) }
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/sync/SyncClient.kt`

````kotlin
package com.rsis.nativeapp.sync

import android.content.Context
import com.rsis.nativeapp.Prefs
import com.rsis.nativeapp.RsisRuntime
import com.rsis.nativeapp.store.EventStore
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.withContext
import org.json.JSONObject
import java.io.IOException

/**
 * Flushes queued event records oldest-first, one at a time. Any failure leaves the event queued.
 * Never raises an alert; never blocks detection. Pairing happens here, lazily, only when there is something to send.
 */
class SyncClient(private val ctx: Context, private val store: EventStore) {
    private val mutex = Mutex()
    private val pairing = PairingClient(ctx)

    /** @return number of events accepted (including server-side duplicates). */
    suspend fun flush(): Int = mutex.withLock {
        withContext(Dispatchers.IO) {
            if (RsisRuntime.uploadsBlocked || store.count() == 0) return@withContext 0
            val key: String = when (val r = pairing.ensurePaired()) {
                is PairResult.Ok -> r.key
                PairResult.Revoked -> { block("this device was unpaired by support"); return@withContext 0 }
                PairResult.StaleBuild -> { block("this app build is out of date"); return@withContext 0 }
                PairResult.Unreachable -> return@withContext 0
                is PairResult.Rejected -> { RsisRuntime.syncIssue = "pairing was refused (${r.code})"; return@withContext 0 }
            }
            var sent = 0
            for (item in store.pending()) {
                val body = JSONObject(item.payload.toString())
                if (body.optString("connectivity_state") == "offline_at_detection") body.put("connectivity_state", "reconnected")
                try {
                    val (code, text) = Http.postJson(
                        Http.BASE + "/syncNativeEvent", body, mapOf(PairingContract.KEY_HEADER to key),
                    )
                    when {
                        code in 200..299 -> {
                            val j = JSONObject(text)
                            if (j.optBoolean("ok") || j.optBoolean("duplicate")) {
                                store.remove(item.eventId); sent++; RsisRuntime.syncIssue = null
                            } else return@withContext sent
                        }
                        code == 401 -> { Prefs.clearPairing(ctx); return@withContext sent } // key no longer valid: re-pair next time
                        code == 403 -> { block("this device was unpaired by support"); return@withContext sent }
                        code == 409 -> { block("this app build is out of date"); return@withContext sent }
                        else -> return@withContext sent // leave queued, retry later
                    }
                } catch (_: IOException) { return@withContext sent } catch (_: org.json.JSONException) { return@withContext sent }
            }
            sent
        }
    }

    private fun block(reason: String) { RsisRuntime.uploadsBlocked = true; RsisRuntime.syncIssue = reason }

    /** §6b. Not needed by the native alert path. */
    suspend fun checkDuplicate(eventId: String): JSONObject? = withContext(Dispatchers.IO) {
        try {
            val (code, text) = Http.postJson(Http.BASE + "/checkDuplicate", JSONObject().put("event_id", eventId))
            if (code in 200..299) JSONObject(text) else null
        } catch (_: Exception) { null }
    }
}
````

---

## `app/src/main/java/com/rsis/nativeapp/voice/VoiceAlerts.kt`

````kotlin
package com.rsis.nativeapp.voice

import android.content.Context
import android.media.AudioAttributes
import android.media.AudioFocusRequest
import android.media.AudioManager
import android.speech.tts.TextToSpeech
import android.speech.tts.UtteranceProgressListener
import com.rsis.nativeapp.RsisRuntime
import java.util.Locale
import java.util.UUID

/**
 * Spoken alerts. Android has no official voice-gender API, so a female voice is selected from the
 * installed OFFLINE voices whose name carries a "female" marker (e.g. Google TTS "...#female_1-local").
 * If none exists the alert still plays (safety first), RsisRuntime.femaleVoice=false, and the UI tells the user.
 */
class VoiceAlerts(ctx: Context) {
    private val audio = ctx.getSystemService(Context.AUDIO_SERVICE) as AudioManager
    private var tts: TextToSpeech? = null
    @Volatile private var ready = false
    private var focus: AudioFocusRequest? = null

    init {
        tts = TextToSpeech(ctx.applicationContext) { status ->
            if (status != TextToSpeech.SUCCESS) { RsisRuntime.issue = "Text-to-speech unavailable"; return@TextToSpeech }
            val t = tts ?: return@TextToSpeech
            val locale = Locale.getDefault().takeIf { t.isLanguageAvailable(it) >= TextToSpeech.LANG_AVAILABLE } ?: Locale.US
            t.language = locale
            t.setAudioAttributes(
                AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
                    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build()
            )
            val female = t.voices.orEmpty().firstOrNull {
                it.locale.language == locale.language && !it.isNetworkConnectionRequired &&
                    it.name.contains("female", ignoreCase = true)
            }
            if (female != null) { t.voice = female; RsisRuntime.femaleVoice = true } else RsisRuntime.femaleVoice = false
            t.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
                override fun onStart(id: String?) {}
                override fun onDone(id: String?) { release() }
                @Deprecated("Deprecated in Java") override fun onError(id: String?) { release() }
            })
            ready = true
        }
    }

    /** True only if the utterance was accepted for playback. */
    fun speak(text: String): Boolean {
        val t = tts ?: return false
        if (!ready) return false
        val req = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
            .setAudioAttributes(
                AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
                    .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build()
            ).build()
        focus = req
        audio.requestAudioFocus(req)
        return t.speak(text, TextToSpeech.QUEUE_FLUSH, null, UUID.randomUUID().toString()) == TextToSpeech.SUCCESS
    }

    private fun release() { focus?.let { audio.abandonAudioFocusRequest(it) }; focus = null }

    fun shutdown() { release(); tts?.stop(); tts?.shutdown(); tts = null; ready = false }
}
````

---

## `app/src/test/java/com/rsis/nativeapp/EngineTest.kt`

````kotlin
package com.rsis.nativeapp

import com.rsis.nativeapp.engine.*
import com.rsis.nativeapp.spb.Spb
import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Test
import java.util.Random
import kotlin.math.log10
import kotlin.math.roundToInt

class EngineTest {
    private var now = 0L
    private val alerts = ArrayList<AlertDecision>()
    private val engine = SafetyEngine({ now }, DistanceEstimator(), object : AlertListener {
        override fun onAlert(d: AlertDecision): Boolean { alerts.add(d); return true }
        override fun onDenseTraffic(active: Boolean) {}
    })
    private val rnd = Random(7)

    private fun rssiFor(ft: Double) = (-59.0 - 22.0 * log10(ft / 3.28084)).roundToInt()

    private fun feed(ft: Double, token: ByteArray) {
        val pkt = Spb.decode(Spb.encode(Role.CYCLIST, 0, token, 0.0))!!
        engine.onSighting(Sighting(pkt, rssiFor(ft), null, now))
        engine.tick(null)
    }

    /** Token rotates every 3 s, like the real beacon. */
    private fun run(fromFt: Double, toFt: Double, seconds: Int, tok: () -> ByteArray) {
        val steps = seconds * 4
        repeat(steps) { i ->
            now += 250
            if (now % 3000L == 0L) tok()
            feed(fromFt + (toFt - fromFt) * i / steps, currentToken)
        }
    }
    private var currentToken = ByteArray(8).also { rnd.nextBytes(it) }
    private val rotate = { currentToken = ByteArray(8).also { rnd.nextBytes(it) }; currentToken }

    @Test fun approachAlertsOnceThenOnlyOnEscalation() {
        run(150.0, 60.0, 18, rotate)   // yellow
        run(60.0, 10.0, 10, rotate)    // orange (critical + approaching)
        val fresh = alerts.filter { !it.fallback }
        assertEquals(listOf(Tier.YELLOW, Tier.ORANGE), fresh.map { it.tier })
        assertTrue(alerts.all { it.eventId.matches(Regex("RSIS-LOCAL-[A-Z0-9]{8}")) })
    }

    @Test fun fallbackRepeatsSameEventAndCanFire() {
        run(150.0, 60.0, 18, rotate)
        val fb = alerts.filter { it.fallback }
        assertTrue(fb.isNotEmpty())
        assertTrue(fb.all { f -> alerts.any { !it.fallback && it.eventId == f.eventId } })
    }

    @Test fun recedingUserNeverAlerts() {
        run(100.0, 200.0, 20, rotate)
        assertTrue(alerts.isEmpty())
    }
}
````

---

## `app/src/test/java/com/rsis/nativeapp/SharedSpecTest.kt`

````kotlin
package com.rsis.nativeapp

import com.rsis.nativeapp.spb.Spb
import org.json.JSONObject
import org.junit.Assert.assertEquals
import org.junit.Test
import java.io.File

/** Fails the build if the Kotlin constants drift from protocol/rsis-protocol.json (the single shared spec). */
class SharedSpecTest {
    private val spec = JSONObject(File("../protocol/rsis-protocol.json").readText())

    @Test fun perimeters() {
        val p = spec.getJSONObject("alert_perimeters")
        assertEquals(p.getDouble("gps_relay_miles"), AlertPerimeters.GPS_RELAY_MILES, 0.0)
        assertEquals(p.getDouble("ble_mesh_ft"), AlertPerimeters.BLE_MESH_ALERT_FT, 0.0)
        assertEquals(p.getDouble("ble_fallback_ft"), AlertPerimeters.BLE_FALLBACK_ALERT_FT, 0.0)
        assertEquals(p.getDouble("immediate_ft"), AlertPerimeters.IMMEDIATE_FT, 0.0)
    }

    @Test fun timing() {
        val t = spec.getJSONObject("timing_ms")
        assertEquals(t.getLong("cooldown"), Timing.COOLDOWN_MS)
        assertEquals(t.getLong("fallback_wait"), Timing.FALLBACK_WAIT_MS)
        assertEquals(t.getLong("cluster_recompute"), Timing.CLUSTER_RECOMPUTE_MS)
        assertEquals(spec.getJSONObject("packet").getLong("token_rotation_ms"), Timing.TOKEN_ROTATION_MS)
    }

    @Test fun cluster() {
        val c = spec.getJSONObject("cluster")
        assertEquals(c.getDouble("radius_ft"), ClusterRule.RADIUS_FT, 0.0)
        assertEquals(c.getInt("min_nodes"), ClusterRule.MIN_NODES)
        assertEquals(c.getDouble("max_mph"), ClusterRule.MAX_MPH, 0.0)
    }

    @Test fun protocolIdentityMatchesPairingConstants() {
        assertEquals(spec.getString("protocol"), NativeProtocol.PROTOCOL)
        assertEquals(spec.getJSONObject("packet").getInt("version"), NativeProtocol.PACKET_VERSION)
    }

    @Test fun packetAndBands() {
        val k = spec.getJSONObject("packet")
        assertEquals(k.getInt("version"), Spb.VERSION)
        assertEquals(k.getInt("size"), Spb.SIZE)
        assertEquals(k.getInt("token_bytes"), Spb.TOKEN_SIZE)
        val b = spec.getJSONObject("bands_ft")
        assertEquals(Band.CRITICAL, Band.of(b.getInt("critical_max") - 1))
        assertEquals(Band.HIGH, Band.of(b.getInt("critical_max")))
        assertEquals(Band.MEDIUM, Band.of(b.getInt("high_max")))
        assertEquals(Band.LOW, Band.of(b.getInt("medium_max")))
        assertEquals(Band.LOW, Band.of(b.getInt("low_max")))
        assertEquals(Band.OUT_OF_BAND, Band.of(b.getInt("low_max") + 1))
    }
}
````

---

## `app/src/test/java/com/rsis/nativeapp/SpbTest.kt`

````kotlin
package com.rsis.nativeapp

import com.rsis.nativeapp.spb.Spb
import org.junit.Assert.*
import org.junit.Test

class SpbTest {
    private val token = ByteArray(8) { (it + 1).toByte() }

    @Test fun crcKnownVector() {
        assertEquals(0x29B1, Spb.crc16("123456789".toByteArray()))
    }

    @Test fun roundTrip() {
        val raw = Spb.encode(Role.CYCLIST, 0, token, 0.4321)
        assertEquals(15, raw.size)
        assertEquals(0x41, raw[0].toInt())
        val p = Spb.decode(raw)!!
        assertEquals(Role.CYCLIST, p.role)
        assertEquals(4321, p.riskScaled)
        assertArrayEquals(token, p.token)
    }

    @Test fun rejectsObsolete10Byte() { assertNull(Spb.decode(ByteArray(10))) }

    @Test fun rejectsBadCrc() {
        val raw = Spb.encode(Role.VEHICLE, 0, token, 0.1)
        raw[5] = (raw[5] + 1).toByte()
        assertNull(Spb.decode(raw))
    }

    @Test fun rejectsWrongVersion() {
        val raw = Spb.encode(Role.PEDESTRIAN, 0, token, 0.1)
        raw[0] = 0x40
        assertNull(Spb.decode(raw))
    }
}
````

---

## `website/rsis-android-gate.html`

````html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>RSIS Offline Safety Mode</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Atkinson+Hyperlegible:wght@400;700&display=swap" rel="stylesheet">
<style>
  :root {
    --asphalt: #1d2229;
    --ink: #1d2229;
    --muted: #4d5764;
    --ground: #f3f5f7;
    --panel: #ffffff;
    --rule: #cfd5dc;
    --lane: #f2c230;      /* Yellow tier */
    --lane-ink: #1d2229;
    --warn: #d9480f;      /* Orange tier */
    --ok: #1f7a4d;
    --focus: #0b5fff;
    color-scheme: light;
    box-sizing: border-box;
    padding-top: env(safe-area-inset-top, 0px);
    padding-bottom: env(safe-area-inset-bottom, 0px);
  }
  @media (prefers-color-scheme: dark) {
    :root:not([data-theme="light"]) {
      --ink: #eef1f4; --muted: #aab3bd; --ground: #12161b; --panel: #1b2128; --rule: #333c46;
      --ok: #55c58b; --focus: #7aa7ff; --warn: #ff8a4c; color-scheme: dark;
    }
  }
  :root[data-theme="dark"] {
    --ink: #eef1f4; --muted: #aab3bd; --ground: #12161b; --panel: #1b2128; --rule: #333c46;
    --ok: #55c58b; --focus: #7aa7ff; --warn: #ff8a4c; color-scheme: dark;
  }
  *, *::before, *::after { box-sizing: inherit; }
  html { scroll-padding-top: env(safe-area-inset-top, 0px); }
  body {
    margin: 0; background: var(--ground); color: var(--ink);
    font: 400 1.0625rem/1.55 "Atkinson Hyperlegible", system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  }
  main { max-width: 34rem; margin: 0 auto; padding: 1.5rem 1.25rem 3rem; }
  h1 { font-size: clamp(1.7rem, 7vw, 2.3rem); line-height: 1.15; margin: 0 0 .6rem; font-weight: 700; letter-spacing: -.01em; }
  h2 { font-size: 1.2rem; margin: 2.2rem 0 .6rem; font-weight: 700; }
  p { margin: 0 0 .9rem; max-width: 62ch; }
  .lead { font-size: 1.15rem; color: var(--muted); }

  /* The one memorable element: the alert distances, drawn to scale (0 to 164 ft). */
  .bands { margin: 1.4rem 0 .4rem; }
  .strip { display: flex; height: 2.6rem; border: 2px solid var(--asphalt); border-radius: 4px; overflow: hidden; }
  .strip span { display: flex; align-items: center; justify-content: center; font-size: .8rem; font-weight: 700; color: #1d2229; }
  .b1 { flex: 16;  background: var(--warn); color: #fff !important; }
  .b2 { flex: 33;  background: #f08c3a; }
  .b3 { flex: 49;  background: var(--lane); }
  .b4 { flex: 66;  background: #f7e08a; }
  .ticks { display: flex; font-size: .8rem; color: var(--muted); margin-top: .25rem; }
  .ticks span:nth-child(1) { flex: 16; } .ticks span:nth-child(2) { flex: 33; }
  .ticks span:nth-child(3) { flex: 49; } .ticks span:nth-child(4) { flex: 66; }
  .caption { font-size: .9rem; color: var(--muted); }

  .state { background: var(--panel); border: 1px solid var(--rule); border-left: 6px solid var(--lane); border-radius: 4px; padding: 1rem 1.1rem; margin: 1.4rem 0; }
  .state[data-kind="good"] { border-left-color: var(--ok); }
  .state[data-kind="stop"] { border-left-color: var(--warn); }
  .state p:last-child { margin-bottom: 0; }

  fieldset { border: 0; padding: 0; margin: 0; }
  legend { padding: 0; }
  .ack { display: flex; gap: .8rem; align-items: flex-start; padding: .75rem 0; border-top: 1px solid var(--rule); }
  .ack:last-of-type { border-bottom: 1px solid var(--rule); }
  .ack input { flex: none; width: 1.5rem; height: 1.5rem; margin-top: .1rem; accent-color: var(--asphalt); }
  .ack label { cursor: pointer; }

  .btn {
    display: block; width: 100%; text-align: center; text-decoration: none; cursor: pointer;
    font-family: inherit; font-weight: 700; font-size: 1.1rem; line-height: 1.2;
    padding: 1rem 1.2rem; margin-top: 1.1rem; border-radius: 4px;
    background: var(--lane); color: var(--lane-ink); border: 2px solid var(--asphalt);
  }
  .btn[aria-disabled="true"] { background: transparent; color: var(--muted); border-style: dashed; border-color: var(--muted); cursor: not-allowed; }
  .btn.secondary { background: transparent; color: var(--ink); }
  :focus-visible { outline: 3px solid var(--focus); outline-offset: 3px; }
  .hint { font-size: .95rem; color: var(--muted); margin-top: .6rem; }

  .hash { display: block; margin: .4rem 0; padding: .7rem .8rem; border: 1px solid var(--rule); background: var(--panel); border-radius: 4px;
          font: .85rem/1.4 ui-monospace, "SF Mono", Menlo, Consolas, monospace; word-break: break-all; }
  .linkish { background: none; border: 0; padding: .4rem 0; color: var(--ink); text-decoration: underline; font: inherit; cursor: pointer; }
  ol.steps { padding-left: 1.3rem; } ol.steps li { margin-bottom: .6rem; }
  .risk { margin-top: 2.2rem; padding-top: 1rem; border-top: 2px solid var(--asphalt); font-size: .95rem; color: var(--muted); }
  [hidden] { display: none !important; }
</style>
</head>
<body>
<main>
  <h1 id="headline">Checking this device…</h1>
  <p class="lead" id="subline"></p>

  <section class="state" id="state" aria-live="polite" hidden>
    <p id="stateText"></p>
    <button class="btn" id="startBtn" type="button" hidden>Turn on Bluetooth Mesh</button>
  </section>

  <div class="bands" aria-label="Alert distance bands from 0 to 164 feet">
    <div class="strip" role="img" aria-label="Critical 0 to 16 feet, high 16 to 49 feet, medium 49 to 98 feet, low 98 to 164 feet">
      <span class="b1">0</span><span class="b2">16</span><span class="b3">49</span><span class="b4">98</span>
    </div>
    <div class="ticks" aria-hidden="true"><span>critical</span><span>high</span><span>medium</span><span>low</span></div>
    <p class="caption">Bluetooth alert bands in feet: 0 to 16, 16 to 49, 49 to 98, and 98 to 164.</p>
  </div>

  <h2>What Offline Safety Mode does</h2>
  <p>Your phone listens for other RSIS users nearby over Bluetooth and speaks a warning before your paths cross. It works with no internet. Only your role and a code that changes every 3 seconds are shared.</p>
  <p>It works only in the RSIS Android app. It is not available in a web browser or on iPhone.</p>

  <!-- Install flow: shown only on Android browsers that are not the RSIS app -->
  <section id="install" hidden>
    <h2>Before you install</h2>
    <form id="ackForm" onsubmit="return false">
      <fieldset>
        <legend class="hint">Tick each box to continue.</legend>
        <div class="ack"><input type="checkbox" id="a1"><label for="a1">I understand Offline Safety Mode works only in the RSIS Android app, not in a browser or on iPhone.</label></div>
        <div class="ack"><input type="checkbox" id="a2"><label for="a2">I understand other RSIS users nearby must also have the app running with Bluetooth on, or I will not get an offline alert.</label></div>
        <div class="ack"><input type="checkbox" id="a3"><label for="a3">I understand it runs in the background with a permanent notification, uses more battery, and needs the app opened again after my phone restarts.</label></div>
        <div class="ack"><input type="checkbox" id="a4"><label for="a4">I am installing from outside Google Play. I will install only from this page and update the app myself.</label></div>
        <div class="ack"><input type="checkbox" id="a5"><label for="a5">I use it at my own risk. It supplements my attention, can miss or delay alerts, and does not replace emergency services.</label></div>
      </fieldset>
    </form>

    <a class="btn" id="dl" role="button" aria-disabled="true" href="#">Download the RSIS Android app</a>
    <p class="hint" id="dlHint"></p>

    <h2>Check the file</h2>
    <p>After downloading, compare the file’s SHA-256 checksum with this one. If they differ, delete the file and do not install it.</p>
    <code class="hash" id="hash">Not published yet</code>
    <button class="linkish" id="copyHash" type="button" hidden>Copy checksum</button>

    <h2>Install steps</h2>
    <ol class="steps">
      <li>Open Settings, then Apps, then Special app access, then Install unknown apps. Allow your browser to install apps.</li>
      <li>Download the file with the button above.</li>
      <li>Tap the downloaded file and choose Install. Android or Google Play Protect may warn you it is from outside the Play Store. Check the file first as described above.</li>
      <li>Open RSIS, choose your role, and tap Turn on Bluetooth Mesh. Allow Nearby devices, Location, and Notifications when asked, and allow Bluetooth to be turned on.</li>
      <li>Keep Bluetooth on. A permanent RSIS notification shows while it runs. The screen can be locked and the phone in your pocket.</li>
    </ol>
    <p>If you turn Bluetooth off, you are not protected offline until you turn it back on. After a restart, open RSIS once.</p>
  </section>

  <p class="risk">Offline Safety Mode is a supplementary warning tool used at your own risk. Range is reduced by buildings, terrain, vehicles, and your phone’s hardware. In an emergency, call emergency services directly.</p>
</main>

<script>
  // ---- Fill these in once the signed APK exists. Until then the download button stays disabled. ----
  var CONFIG = {
    apkUrl: "",     // direct HTTPS link that downloads the Bluetooth build (NOT the website-wrapper APK)
    sha256: "",     // 64 hex characters from sha256sum / certutil / shasum
    version: ""     // e.g. "6.2.0-native.1"
  };

  var $ = function (id) { return document.getElementById(id); };
  var native = window.RSISNative;
  var isAndroid = /Android/i.test(navigator.userAgent);

  function show(el, on) { el.hidden = !on; }

  function setState(kind, text) {
    var s = $("state"); s.hidden = false; s.setAttribute("data-kind", kind); $("stateText").textContent = text;
  }

  function renderNative() {
    var status = "";
    try { status = String(native.getBleStatus()); } catch (e) { status = "installed"; }
    $("headline").textContent = status === "running" ? "Bluetooth Mesh is on" : "Bluetooth Mesh is off";
    $("subline").textContent = "You are using the RSIS Android app.";
    var btn = $("startBtn");
    if (status === "running") {
      setState("good", "Running. You will hear alerts about nearby RSIS users, even with no internet.");
      show(btn, false);
    } else if (status === "permission_required") {
      setState("stop", "RSIS needs your permission to use Bluetooth. GPS relay still works.");
      show(btn, true);
    } else {
      setState("stop", "Not running. If you turned Bluetooth off, turn it back on. GPS relay still works.");
      show(btn, true);
    }
  }

  function renderBrowser() {
    if (isAndroid) {
      $("headline").textContent = "Install the RSIS Android app to use Offline Safety Mode";
      $("subline").textContent = "This browser cannot use Bluetooth in the background or with the screen locked.";
      show($("install"), true);
    } else {
      $("headline").textContent = "Offline Safety Mode is not available on this device";
      $("subline").textContent = "It works only in the RSIS Android app. GPS relay works here as usual.";
    }
  }

  // ---- install gate ----
  var boxes = ["a1", "a2", "a3", "a4", "a5"];
  var urlOk = /^https:\/\//i.test(CONFIG.apkUrl);
  var hashOk = /^[0-9a-f]{64}$/i.test(CONFIG.sha256);

  function allTicked() { return boxes.every(function (id) { return $(id).checked; }); }

  function refreshDownload() {
    var dl = $("dl"), hint = $("dlHint");
    var ready = urlOk && hashOk && allTicked();
    dl.setAttribute("aria-disabled", ready ? "false" : "true");
    dl.setAttribute("href", ready ? CONFIG.apkUrl : "#");
    if (!urlOk || !hashOk) hint.textContent = "The download is not available yet.";
    else if (!allTicked()) hint.textContent = "Tick every box above to enable the download.";
    else hint.textContent = CONFIG.version ? "Version " + CONFIG.version + "." : "";
  }

  $("dl").addEventListener("click", function (e) {
    if ($("dl").getAttribute("aria-disabled") === "true") e.preventDefault();
  });
  boxes.forEach(function (id) { $(id).addEventListener("change", refreshDownload); });

  if (hashOk) {
    $("hash").textContent = CONFIG.sha256.toLowerCase();
    show($("copyHash"), !!(navigator.clipboard));
    $("copyHash").addEventListener("click", function () {
      navigator.clipboard.writeText(CONFIG.sha256.toLowerCase()).then(function () {
        $("copyHash").textContent = "Copied";
      }, function () { $("copyHash").textContent = "Copy failed. Select the checksum and copy it."; });
    });
  }

  // ---- start ----
  $("startBtn").addEventListener("click", function () {
    try {
      Promise.resolve(native.startOfflineSafety()).then(function () { setTimeout(renderNative, 800); });
    } catch (e) { setTimeout(renderNative, 800); }
  });

  if (native && typeof native.getBleStatus === "function") {
    renderNative();
    setInterval(renderNative, 3000); // status changes if Bluetooth is switched off or on
  } else {
    renderBrowser();
    refreshDownload();
  }
</script>
</body>
</html>
````

