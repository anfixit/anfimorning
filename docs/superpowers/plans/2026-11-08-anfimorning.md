# anfimorning — план реализации

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Android-приложение «Утро в движении»: зарядка с голосом, видео и таймером, календарём, напоминанием и отметкой самочувствия.

**Architecture:** Одна Activity на Jetpack Compose. Логика без Android (программа, движок занятия, серия дней, расчёт напоминания, календарь) — чистый Kotlin с JVM-тестами. Android-слой: Room (история), DataStore (настройки), AlarmManager (напоминание), TextToSpeech и ToneGenerator (голос и сигналы), Media3 (видео). Зависимости собираются вручную в `AppContainer`.

**Tech Stack:** Kotlin 2.1.0, AGP 8.7.3, Gradle 8.11.1, JDK 17, Compose BOM 2024.12.01 (Material 3), Navigation Compose 2.8.5, Media3 1.5.1, Room 2.6.1 + KSP 2.1.0-1.0.29, DataStore 1.1.1, kotlinx.serialization 1.7.3, JUnit 4.13.2.

Спецификация: `docs/superpowers/specs/2026-11-08-anfimorning-design.md`.

## Global Constraints

- Пакет и applicationId: `com.anfixit.anfimorning`; minSdk 26, compileSdk/targetSdk 35; JVM target 17.
- Название приложения на телефоне: «Утро в движении».
- Только вертикальная ориентация, только тёмная тема с палитрой: фон `#0D1A1E`, панель `#15272B`, панель-2 `#1B3135`, текст `#F2F7F4`, приглушённый `#B2C4C1`, акцент `#A9E9C8`, мятный `#7BC9A6`, голубой `#88B6ED`, сиреневый `#C0ADEE`, жёлтый `#EED39A`, коралловый `#FFB4A2`.
- Все тексты интерфейса и голоса — по-русски, на «ты».
- Подготовка: 10 с; 20 с при смене уровня (стоя `standing`/`chair`/`wall` ↔ пол `quadruped`/`prone`/`sidelying`/`supine`/`kneeling`); у первого упражнения 10 с; «+10 с» продлевает подготовку.
- Полная программа: 18 упражнений, 1200 с работы, 1400 с всего. Короткая: 10 упражнений, 490 с работы, 610 с всего.
- Репозиторий публичный: никаких личных данных о здоровье, фото, имён; `keystore.properties`, `*.jks`, `local.properties` — не в git.
- Без аккаунтов, сети, аналитики. Разрешения: только `POST_NOTIFICATIONS`, `SCHEDULE_EXACT_ALARM`, `RECEIVE_BOOT_COMPLETED`.
- Коммиты заканчиваются строкой `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; `git push` (remote `origin` пушит в GitHub и GitLab) — только когда автор скажет.
- Команды Gradle запускаются из корня репозитория `~/dev/anfimorning`.

## Структура файлов

```
anfimorning/
├── .gitignore, README.md, settings.gradle.kts, build.gradle.kts, gradle.properties
├── gradle/libs.versions.toml, gradle/wrapper/*, gradlew, gradlew.bat
├── CREDITS.md                                   (задача 13)
└── app/
    ├── build.gradle.kts
    └── src/
        ├── main/
        │   ├── AndroidManifest.xml
        │   ├── assets/program.json               упражнения и программы
        │   ├── res/values/{strings,colors,themes}.xml
        │   ├── res/drawable/{ic_launcher_foreground,ic_notification}.xml
        │   ├── res/mipmap-anydpi-v26/ic_launcher.xml
        │   ├── res/raw/*.mp4                     видео (задача 13)
        │   └── java/com/anfixit/anfimorning/
        │       ├── AnfimorningApp.kt, AppContainer.kt, MainActivity.kt
        │       ├── program/   Model.kt, Plan.kt, ProgramParser.kt
        │       ├── session/   SessionEngine.kt
        │       ├── history/   SessionRecord.kt, SessionDao.kt, AppDatabase.kt,
        │       │              HistoryRepository.kt, StreakCalculator.kt
        │       ├── settings/  AppSettings.kt, SettingsRepository.kt, NextReminderCalculator.kt
        │       ├── guide/     GuidePhrases.kt, VoiceGuide.kt, Beeper.kt
        │       ├── reminder/  ReminderScheduler.kt, ReminderReceiver.kt,
        │       │              RescheduleReceiver.kt, Notifications.kt
        │       └── ui/        Theme.kt, Format.kt, AppNav.kt, Effects.kt,
        │                      home/, session/, done/, calendar/, settings/
        └── test/java/com/anfixit/anfimorning/   JVM-тесты по пакетам
```

---

### Task 1: Инструменты и каркас проекта

**Files:**
- Create: `.gitignore`, `README.md`, `settings.gradle.kts`, `build.gradle.kts`, `gradle.properties`, `gradle/libs.versions.toml`, `gradlew`, `gradlew.bat`, `gradle/wrapper/*`
- Create: `app/build.gradle.kts`, `app/src/main/AndroidManifest.xml`, `app/src/main/java/com/anfixit/anfimorning/MainActivity.kt`
- Create: `app/src/main/res/values/strings.xml`, `colors.xml`, `themes.xml`, `app/src/main/res/drawable/ic_launcher_foreground.xml`, `app/src/main/res/drawable/ic_notification.xml`, `app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`
- Test: `app/src/test/java/com/anfixit/anfimorning/SmokeTest.kt`

**Interfaces:**
- Produces: собираемый проект, `./gradlew testDebugUnitTest` и `./gradlew assembleDebug` работают; все зависимости из Tech Stack подключены.

- [ ] **Step 1: Поставить JDK 17, Gradle и Android command-line tools**

```bash
brew install openjdk@17 gradle
brew install --cask android-commandlinetools
```

Затем указать Gradle путь к JDK (файл пользователя Gradle, не shell-конфиг):

```bash
mkdir -p ~/.gradle
grep -q org.gradle.java.home ~/.gradle/gradle.properties 2>/dev/null || \
  echo "org.gradle.java.home=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home" >> ~/.gradle/gradle.properties
```

- [ ] **Step 2: СТОП — спросить автора про лицензии Android SDK**

Установка пакетов SDK требует принять лицензии Google. Спросить в чате: «Для сборки нужно принять лицензионные соглашения Android SDK (Google). Принимаю?» — и ждать явного «да». Только после него:

```bash
export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools
yes | sdkmanager --sdk_root=$ANDROID_HOME --licenses
sdkmanager --sdk_root=$ANDROID_HOME "platform-tools" "platforms;android-35" "build-tools;35.0.0"
```

Expected: `platforms/android-35` и `build-tools/35.0.0` появились в `$ANDROID_HOME`.

- [ ] **Step 3: Сгенерировать Gradle wrapper 8.11.1 в пустой папке и скопировать**

```bash
W=$(mktemp -d) && (cd "$W" && gradle wrapper --gradle-version 8.11.1 --distribution-type bin) \
  && cp -R "$W/gradlew" "$W/gradlew.bat" "$W/gradle" ~/dev/anfimorning/ && rm -rf "$W"
```

- [ ] **Step 4: Написать файлы сборки**

`.gitignore`:
```gitignore
.gradle/
.kotlin/
build/
local.properties
.idea/
*.iml
.DS_Store
captures/
keystore.properties
*.jks
*.keystore
```

`settings.gradle.kts`:
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
rootProject.name = "anfimorning"
include(":app")
```

`gradle.properties`:
```properties
org.gradle.jvmargs=-Xmx4g -Dfile.encoding=UTF-8
android.useAndroidX=true
android.nonTransitiveRClass=true
kotlin.code.style=official
```

`gradle/libs.versions.toml`:
```toml
[versions]
agp = "8.7.3"
kotlin = "2.1.0"
ksp = "2.1.0-1.0.29"
coreKtx = "1.15.0"
lifecycle = "2.8.7"
activityCompose = "1.9.3"
composeBom = "2024.12.01"
navigation = "2.8.5"
media3 = "1.5.1"
room = "2.6.1"
datastore = "1.1.1"
serialization = "1.7.3"
coroutines = "1.9.0"
junit = "4.13.2"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycle" }
androidx-lifecycle-runtime-compose = { group = "androidx.lifecycle", name = "lifecycle-runtime-compose", version.ref = "lifecycle" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-graphics = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigation" }
androidx-media3-exoplayer = { group = "androidx.media3", name = "media3-exoplayer", version.ref = "media3" }
androidx-media3-ui = { group = "androidx.media3", name = "media3-ui", version.ref = "media3" }
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serialization" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
junit = { group = "junit", name = "junit", version.ref = "junit" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

`build.gradle.kts` (корень):
```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.ksp) apply false
}
```

`app/build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.kotlin.serialization)
    alias(libs.plugins.ksp)
}

android {
    namespace = "com.anfixit.anfimorning"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.anfixit.anfimorning"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
    buildFeatures {
        compose = true
    }
    testOptions {
        unitTests.isReturnDefaultValues = true
    }
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.lifecycle.runtime.compose)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.activity.compose)
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.graphics)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.compose.material3)
    implementation(libs.androidx.navigation.compose)
    implementation(libs.androidx.media3.exoplayer)
    implementation(libs.androidx.media3.ui)
    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)
    implementation(libs.androidx.datastore.preferences)
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.kotlinx.coroutines.android)
    debugImplementation(libs.androidx.compose.ui.tooling)

    testImplementation(libs.junit)
}
```

`local.properties` (в git не идёт):
```properties
sdk.dir=/opt/homebrew/share/android-commandlinetools
```

- [ ] **Step 5: Манифест, ресурсы, пустая Activity**

`app/src/main/AndroidManifest.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/Theme.Anfimorning">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:screenOrientation="portrait">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

`app/src/main/res/values/strings.xml`:
```xml
<resources>
    <string name="app_name">Утро в движении</string>
</resources>
```

`app/src/main/res/values/colors.xml`:
```xml
<resources>
    <color name="launcher_bg">#0D1A1E</color>
</resources>
```

`app/src/main/res/values/themes.xml`:
```xml
<resources>
    <style name="Theme.Anfimorning" parent="android:Theme.Material.NoActionBar">
        <item name="android:windowBackground">@color/launcher_bg</item>
    </style>
</resources>
```

`app/src/main/res/drawable/ic_launcher_foreground.xml`:
```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp" android:height="108dp"
    android:viewportWidth="108" android:viewportHeight="108">
    <path android:fillColor="#A9E9C8"
        android:pathData="M54,30 a18,18 0 1,1 0,36 a18,18 0 1,1 0,-36z" />
    <path android:fillColor="#7BC9A6" android:pathData="M30,74h48v6h-48z" />
</vector>
```

`app/src/main/res/drawable/ic_notification.xml`:
```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp" android:height="24dp"
    android:viewportWidth="24" android:viewportHeight="24">
    <path android:fillColor="#FFFFFFFF"
        android:pathData="M12,4 a6,6 0 1,1 0,12 a6,6 0 1,1 0,-12z M4,19h16v2h-16z" />
</vector>
```

`app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@color/launcher_bg" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
```

`app/src/main/java/com/anfixit/anfimorning/MainActivity.kt`:
```kotlin
package com.anfixit.anfimorning

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.Text

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent { Text("Утро в движении") }
    }
}
```

`app/src/test/java/com/anfixit/anfimorning/SmokeTest.kt`:
```kotlin
package com.anfixit.anfimorning

import org.junit.Assert.assertEquals
import org.junit.Test

class SmokeTest {
    @Test
    fun `unit tests run`() {
        assertEquals(4, 2 + 2)
    }
}
```

`README.md`:
```markdown
# Утро в движении (anfimorning)

Android-приложение утренней зарядки: голос ведёт по упражнениям, на экране —
короткое видео движения и таймер. Календарь занятий, серия дней подряд,
утреннее напоминание, отметка самочувствия. Работает без интернета и аккаунтов.

Комплекс мягкий и симметричный. Двигайся без боли; при заболеваниях
позвоночника согласуй упражнения с врачом ЛФК.

## Сборка

Нужны JDK 17 и Android SDK (platform 35). Путь к SDK — в `local.properties`
(`sdk.dir=...`).

    ./gradlew testDebugUnitTest
    ./gradlew assembleDebug

Упражнения и программы — в `app/src/main/assets/program.json`.
Дизайн — `docs/superpowers/specs/`, план — `docs/superpowers/plans/`.
```

- [ ] **Step 6: Прогнать тест и сборку**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`, файл `app/build/outputs/apk/debug/app-debug.apk` существует.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "build: Android project skeleton with Compose, Room, Media3

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Модель программы и program.json

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/program/Model.kt`
- Create: `app/src/main/java/com/anfixit/anfimorning/program/Plan.kt`
- Create: `app/src/main/java/com/anfixit/anfimorning/program/ProgramParser.kt`
- Create: `app/src/main/assets/program.json`
- Test: `app/src/test/java/com/anfixit/anfimorning/program/PrepRuleTest.kt`, `ProgramParserTest.kt`, `ProgramFileTest.kt`

**Interfaces:**
- Produces:
  - `enum class Pose(val label: String) { STANDING, CHAIR, WALL, QUADRUPED, PRONE, SIDELYING, SUPINE, KNEELING; val isFloor: Boolean }`
  - `enum class Stage(val label: String) { WARMUP, CORE, STRETCH, BREATH }`
  - `data class Cue(val at: Int, val text: String)`, `data class VideoSource(val url: String, val author: String, val license: String)`
  - `data class Exercise(id, name, pose: Pose, stage: Stage, sides: Boolean = false, short: List<String>, steps: List<String>, avoid: String, easier: String, prepVoice: String, cues: List<Cue> = emptyList(), video: String? = null, videoSource: VideoSource? = null)`
  - `data class ProgramItem(val exercise: String, val work: Int)`, `data class Program(val id: String, val kind: String = "timed", val title: String, val items: List<ProgramItem>)`, `data class ProgramFile(val exercises: List<Exercise>, val programs: List<Program>)`
  - `data class PlannedStep(val exercise: Exercise, val workSec: Int, val prepSec: Int)`
  - `object PrepRule { const val SHORT = 10; const val LONG = 20; fun prepSeconds(previous: Pose?, current: Pose): Int }`
  - `fun ProgramFile.plan(programId: String): List<PlannedStep>`, `fun List<PlannedStep>.totalSeconds(): Int`
  - `object ProgramParser { fun parse(text: String): ProgramFile; fun validate(file: ProgramFile, videoExists: (String) -> Boolean): List<String> }`

- [ ] **Step 1: Написать падающие тесты**

`app/src/test/java/com/anfixit/anfimorning/program/PrepRuleTest.kt`:
```kotlin
package com.anfixit.anfimorning.program

import org.junit.Assert.assertEquals
import org.junit.Test

class PrepRuleTest {
    @Test
    fun `first exercise gets short prep`() {
        assertEquals(10, PrepRule.prepSeconds(null, Pose.QUADRUPED))
    }

    @Test
    fun `same level gets short prep`() {
        assertEquals(10, PrepRule.prepSeconds(Pose.STANDING, Pose.WALL))
        assertEquals(10, PrepRule.prepSeconds(Pose.QUADRUPED, Pose.SUPINE))
        assertEquals(10, PrepRule.prepSeconds(Pose.SUPINE, Pose.KNEELING))
    }

    @Test
    fun `standing to floor and back gets long prep`() {
        assertEquals(20, PrepRule.prepSeconds(Pose.WALL, Pose.QUADRUPED))
        assertEquals(20, PrepRule.prepSeconds(Pose.KNEELING, Pose.STANDING))
    }
}
```

`app/src/test/java/com/anfixit/anfimorning/program/ProgramParserTest.kt`:
```kotlin
package com.anfixit.anfimorning.program

import org.junit.Assert.assertEquals
import org.junit.Assert.assertTrue
import org.junit.Test

class ProgramParserTest {
    private fun exercise(
        id: String,
        cues: List<Cue> = emptyList(),
        video: String? = null,
        source: VideoSource? = null,
    ) = Exercise(
        id = id, name = id, pose = Pose.STANDING, stage = Stage.WARMUP,
        short = listOf("s"), steps = listOf("s"), avoid = "a", easier = "e",
        prepVoice = "p", cues = cues, video = video, videoSource = source,
    )

    private fun file(vararg items: ProgramItem, exercises: List<Exercise> = listOf(exercise("a"))) =
        ProgramFile(exercises, listOf(Program(id = "p", title = "P", items = items.toList())))

    @Test
    fun `parses minimal json`() {
        val text = """
            {"exercises":[{"id":"a","name":"A","pose":"standing","stage":"warmup",
              "short":["s"],"steps":["s"],"avoid":"a","easier":"e","prepVoice":"p"}],
             "programs":[{"id":"p","title":"P","items":[{"exercise":"a","work":30}]}]}
        """.trimIndent()
        val parsed = ProgramParser.parse(text)
        assertEquals(Pose.STANDING, parsed.exercises.single().pose)
        assertEquals("timed", parsed.programs.single().kind)
        assertEquals(30, parsed.programs.single().items.single().work)
    }

    @Test
    fun `valid file has no errors`() {
        assertEquals(emptyList<String>(), ProgramParser.validate(file(ProgramItem("a", 30))) { true })
    }

    @Test
    fun `unknown exercise and zero work are errors`() {
        val errors = ProgramParser.validate(file(ProgramItem("zzz", 30), ProgramItem("a", 0))) { true }
        assertEquals(2, errors.size)
    }

    @Test
    fun `unsorted cues are an error`() {
        val ex = exercise("a", cues = listOf(Cue(10, "x"), Cue(5, "y")))
        val errors = ProgramParser.validate(file(ProgramItem("a", 30), exercises = listOf(ex))) { true }
        assertTrue(errors.single().contains("подсказки"))
    }

    @Test
    fun `missing video file and missing source are errors`() {
        val ex = exercise("a", video = "a_clip")
        val errors = ProgramParser.validate(file(ProgramItem("a", 30), exercises = listOf(ex))) { false }
        assertEquals(2, errors.size)
    }
}
```

`app/src/test/java/com/anfixit/anfimorning/program/ProgramFileTest.kt`:
```kotlin
package com.anfixit.anfimorning.program

import org.junit.Assert.assertEquals
import org.junit.Test
import java.io.File

class ProgramFileTest {
    private val file = ProgramParser.parse(File("src/main/assets/program.json").readText())

    private fun videoExists(name: String) = File("src/main/res/raw/$name.mp4").exists()

    @Test
    fun `program json is valid`() {
        assertEquals(emptyList<String>(), ProgramParser.validate(file, ::videoExists))
    }

    @Test
    fun `full program timing`() {
        val plan = file.plan("full")
        assertEquals(18, plan.size)
        assertEquals(1200, plan.sumOf { it.workSec })
        assertEquals(1400, plan.totalSeconds())
        assertEquals(
            listOf(10, 10, 10, 10, 10, 10, 10, 10, 10, 20, 10, 10, 10, 10, 10, 10, 10, 20),
            plan.map { it.prepSec },
        )
    }

    @Test
    fun `short program timing`() {
        val plan = file.plan("short")
        assertEquals(10, plan.size)
        assertEquals(490, plan.sumOf { it.workSec })
        assertEquals(610, plan.totalSeconds())
    }
}
```

- [ ] **Step 2: Убедиться, что тесты падают**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.program.*'`
Expected: FAIL — `Unresolved reference: PrepRule` / `Pose`.

- [ ] **Step 3: Написать модель, план и парсер**

`app/src/main/java/com/anfixit/anfimorning/program/Model.kt`:
```kotlin
package com.anfixit.anfimorning.program

import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
enum class Pose(val label: String) {
    @SerialName("standing") STANDING("Стоя"),
    @SerialName("chair") CHAIR("У стула"),
    @SerialName("wall") WALL("У стены"),
    @SerialName("quadruped") QUADRUPED("На четвереньках"),
    @SerialName("prone") PRONE("Лёжа на животе"),
    @SerialName("sidelying") SIDELYING("Лёжа на боку"),
    @SerialName("supine") SUPINE("Лёжа на спине"),
    @SerialName("kneeling") KNEELING("Полуколено");

    val isFloor: Boolean
        get() = this == QUADRUPED || this == PRONE || this == SIDELYING || this == SUPINE || this == KNEELING
}

@Serializable
enum class Stage(val label: String) {
    @SerialName("warmup") WARMUP("Разминка"),
    @SerialName("core") CORE("Укрепление"),
    @SerialName("stretch") STRETCH("Растяжка"),
    @SerialName("breath") BREATH("Дыхание"),
}

@Serializable
data class Cue(val at: Int, val text: String)

@Serializable
data class VideoSource(val url: String, val author: String, val license: String)

@Serializable
data class Exercise(
    val id: String,
    val name: String,
    val pose: Pose,
    val stage: Stage,
    val sides: Boolean = false,
    val short: List<String>,
    val steps: List<String>,
    val avoid: String,
    val easier: String,
    val prepVoice: String,
    val cues: List<Cue> = emptyList(),
    val video: String? = null,
    val videoSource: VideoSource? = null,
)

@Serializable
data class ProgramItem(val exercise: String, val work: Int)

@Serializable
data class Program(
    val id: String,
    val kind: String = "timed",
    val title: String,
    val items: List<ProgramItem>,
)

@Serializable
data class ProgramFile(val exercises: List<Exercise>, val programs: List<Program>)
```

`app/src/main/java/com/anfixit/anfimorning/program/Plan.kt`:
```kotlin
package com.anfixit.anfimorning.program

data class PlannedStep(val exercise: Exercise, val workSec: Int, val prepSec: Int)

object PrepRule {
    const val SHORT = 10
    const val LONG = 20

    fun prepSeconds(previous: Pose?, current: Pose): Int =
        if (previous != null && previous.isFloor != current.isFloor) LONG else SHORT
}

fun ProgramFile.plan(programId: String): List<PlannedStep> {
    val program = programs.first { it.id == programId }
    val byId = exercises.associateBy { it.id }
    var previous: Pose? = null
    return program.items.map { item ->
        val exercise = byId.getValue(item.exercise)
        PlannedStep(exercise, item.work, PrepRule.prepSeconds(previous, exercise.pose))
            .also { previous = exercise.pose }
    }
}

fun List<PlannedStep>.totalSeconds(): Int = sumOf { it.prepSec + it.workSec }
```

`app/src/main/java/com/anfixit/anfimorning/program/ProgramParser.kt`:
```kotlin
package com.anfixit.anfimorning.program

import kotlinx.serialization.json.Json

object ProgramParser {
    private val json = Json { ignoreUnknownKeys = false }

    fun parse(text: String): ProgramFile = json.decodeFromString(ProgramFile.serializer(), text)

    fun validate(file: ProgramFile, videoExists: (String) -> Boolean): List<String> {
        val errors = mutableListOf<String>()
        val ids = file.exercises.map { it.id }
        ids.groupingBy { it }.eachCount().filterValues { it > 1 }.keys
            .forEach { errors += "Повторяется id упражнения: $it" }
        val known = ids.toSet()

        file.exercises.forEach { ex ->
            if (ex.short.isEmpty()) errors += "${ex.id}: пустой short"
            if (ex.steps.isEmpty()) errors += "${ex.id}: пустой steps"
            val ats = ex.cues.map { it.at }
            if (ats.any { it < 0 }) errors += "${ex.id}: отрицательное время подсказки"
            if (ats.zipWithNext().any { (a, b) -> b <= a }) errors += "${ex.id}: подсказки не по возрастанию"
            ex.video?.let { if (!videoExists(it)) errors += "${ex.id}: нет файла видео $it" }
            if (ex.video != null && ex.videoSource == null) errors += "${ex.id}: у видео нет источника"
        }

        if (file.programs.isEmpty()) errors += "Нет ни одной программы"
        file.programs.forEach { p ->
            if (p.kind != "timed") errors += "${p.id}: неизвестный kind ${p.kind}"
            if (p.items.isEmpty()) errors += "${p.id}: пустая программа"
            p.items.forEach { item ->
                if (item.exercise !in known) errors += "${p.id}: нет упражнения ${item.exercise}"
                if (item.work <= 0) errors += "${p.id}: work у ${item.exercise} должен быть больше 0"
            }
        }
        return errors
    }
}
```

- [ ] **Step 4: Написать program.json**

`app/src/main/assets/program.json`:
```json
{
  "exercises": [
    {
      "id": "neck", "name": "Шея: повороты и кивки", "pose": "standing", "stage": "warmup",
      "short": ["Плечи расслаблены, взгляд вперёд", "Повороты головы вправо и влево", "Небольшие кивки вниз и обратно"],
      "steps": [
        "Стопы на ширине таза, взгляд перед собой, плечи расслаблены.",
        "Плавно поверни голову вправо и верни в центр, затем влево. 3–4 раза в каждую сторону.",
        "Затем 3–4 раза чуть опусти подбородок к груди и вернись в нейтральное положение."
      ],
      "avoid": "Не делай полных кругов головой и резких запрокидываний.",
      "easier": "Уменьши амплитуду; можно выполнять сидя.",
      "prepVoice": "Шея. Встань прямо, стопы на ширине таза, плечи расслаблены.",
      "cues": [
        {"at": 0, "text": "Плавно поворачивай голову вправо и влево."},
        {"at": 15, "text": "Теперь небольшие кивки: подбородок чуть вниз и обратно."}
      ]
    },
    {
      "id": "shoulders", "name": "Плечи: круги и подъёмы рук", "pose": "standing", "stage": "warmup",
      "short": ["Круги плечами назад и вперёд", "Прямые руки вверх до комфорта", "Шея свободна"],
      "steps": [
        "Опусти руки вдоль тела.",
        "Сделай 5 плавных кругов плечами назад и 5 вперёд.",
        "Подними обе руки перед собой до комфортной высоты и опусти 4–5 раз."
      ],
      "avoid": "Не поднимай руки через боль и не зажимай шею.",
      "easier": "Поднимай руки не выше уровня груди.",
      "prepVoice": "Плечи. Опусти руки вдоль тела.",
      "cues": [
        {"at": 0, "text": "Круги плечами назад."},
        {"at": 10, "text": "Теперь вперёд."},
        {"at": 20, "text": "Поднимай прямые руки перед собой до комфортной высоты и опускай."}
      ]
    },
    {
      "id": "arms", "name": "Локти, кисти и пальцы", "pose": "standing", "stage": "warmup",
      "short": ["Сгибай и разгибай локти", "Круги кистями", "Сжимай и раскрывай пальцы"],
      "steps": [
        "По 5 раз плавно согни и разогни локти.",
        "Сделай по 4 небольших круга кистями в каждую сторону.",
        "5 раз сожми пальцы в кулак и полностью раскрой ладони."
      ],
      "avoid": "Не выкручивай суставы до предельного положения.",
      "easier": "Двигай одной рукой за раз.",
      "prepVoice": "Локти, кисти и пальцы. Можно стоя или сидя.",
      "cues": [
        {"at": 0, "text": "Сгибай и разгибай локти."},
        {"at": 10, "text": "Круги кистями в обе стороны."},
        {"at": 20, "text": "Сжимай пальцы в кулак и полностью раскрывай ладони."}
      ]
    },
    {
      "id": "pelvis", "name": "Таз: наклоны вперёд-назад", "pose": "standing", "stage": "warmup",
      "short": ["Руки на тазу, колени мягкие", "Выдох — подкрути таз", "Вдох — вернись в нейтраль"],
      "steps": [
        "Поставь руки на таз, колени чуть мягкие.",
        "На выдохе слегка подкрути таз под себя; на вдохе вернись к нейтральному положению.",
        "Повтори 7–9 раз, без большой амплитуды."
      ],
      "avoid": "Не делай большой прогиб в пояснице.",
      "easier": "Держись одной рукой за спинку стула.",
      "prepVoice": "Наклоны таза. Поставь руки на таз, колени чуть мягкие.",
      "cues": [
        {"at": 0, "text": "На выдохе подкрути таз под себя, на вдохе верни."}
      ]
    },
    {
      "id": "squats_toes", "name": "Мини-приседания и подъёмы на носки", "pose": "chair", "stage": "warmup",
      "short": ["Держись за спинку стула", "Небольшие приседания", "Затем подъёмы на носки"],
      "steps": [
        "Встань лицом к спинке стула, стопы на ширине таза, держись руками.",
        "Немного согни колени, как будто начинаешь садиться, и выпрямись. 6–8 раз.",
        "Затем 6–8 раз медленно поднимись на носки и опустись."
      ],
      "avoid": "Не проваливай колени внутрь, не приседай глубоко и не заваливай голеностопы наружу.",
      "easier": "Сгибай колени всего на несколько сантиметров; на носки поднимайся невысоко.",
      "prepVoice": "Мини-приседания. Встань лицом к стулу и возьмись за спинку.",
      "cues": [
        {"at": 0, "text": "Небольшие приседания, колени смотрят туда же, куда носки."},
        {"at": 20, "text": "Теперь медленно поднимайся на носки и опускайся."}
      ]
    },
    {
      "id": "single_leg", "name": "Стойка на одной ноге", "pose": "chair", "stage": "warmup", "sides": true,
      "short": ["Лёгкая опора рукой о стул", "Стой на одной ноге, таз ровный", "По команде — другая нога"],
      "steps": [
        "Встань боком к стулу, положи на спинку руку для лёгкой опоры.",
        "Перенеси вес на правую ногу и чуть оторви левую стопу от пола.",
        "Колено опорной ноги мягкое, таз ровный, стой спокойно и дыши.",
        "По команде поменяй ногу."
      ],
      "avoid": "Не заваливай таз в сторону и не выпрямляй опорное колено в замок.",
      "easier": "Оставь носок поднятой ноги на полу или держись двумя руками.",
      "prepVoice": "Стойка на одной ноге. Встань боком к стулу и положи руку на спинку.",
      "cues": [
        {"at": 0, "text": "Стой на правой ноге, левую стопу чуть оторви от пола."}
      ]
    },
    {
      "id": "wall_slides", "name": "Скольжение рук по стене", "pose": "wall", "stage": "core",
      "short": ["Лицом к стене, ладони на стене", "Скользи ладонями вверх до комфорта", "Плечи опущены"],
      "steps": [
        "Встань лицом к стене на расстоянии небольшой длины предплечья.",
        "Положи ладони на стену и медленно скользи ими вверх до комфортной высоты.",
        "Опусти руки. Сделай 6–8 повторений, шея расслаблена."
      ],
      "avoid": "Не прогибай поясницу и не поднимай плечи к ушам.",
      "easier": "Поднимай ладони лишь до уровня глаз.",
      "prepVoice": "Скольжение по стене. Встань лицом к стене и положи на неё ладони.",
      "cues": [
        {"at": 0, "text": "Медленно скользи ладонями вверх и опускай."}
      ]
    },
    {
      "id": "chest_stretch", "name": "Растяжка груди у стены", "pose": "wall", "stage": "stretch", "sides": true,
      "short": ["Предплечье на стене, локоть на уровне плеча", "Маленький шаг вперёд", "Мягкое растяжение в груди"],
      "steps": [
        "Встань правым боком к стене. Положи правое предплечье на стену, локоть примерно на уровне плеча.",
        "Сделай маленький шаг вперёд правой ногой, пока не почувствуешь мягкое растяжение в правой половине груди.",
        "Удерживай и дыши спокойно. По команде встань другим боком и повтори левой рукой."
      ],
      "avoid": "Не поднимай плечо к уху и не тянись до боли или покалывания в руке.",
      "easier": "Опусти локоть ниже уровня плеча и сделай шаг совсем маленьким.",
      "prepVoice": "Растяжка груди. Встань правым боком к стене и положи на неё правое предплечье.",
      "cues": [
        {"at": 0, "text": "Маленький шаг вперёд. Удерживай, дыши спокойно."}
      ]
    },
    {
      "id": "calves", "name": "Икроножные мышцы", "pose": "wall", "stage": "stretch", "sides": true,
      "short": ["Руки на стене, одна нога сзади", "Пятка задней ноги на полу", "Корпус чуть к стене"],
      "steps": [
        "Встань лицом к стене, руки на стене.",
        "Отставь правую ногу назад, левую чуть согни впереди. Обе стопы смотрят вперёд.",
        "Правую пятку держи на полу, слегка подай корпус к стене и удерживай. По команде поменяй ноги."
      ],
      "avoid": "Не отрывай пятку и не продавливай колено.",
      "easier": "Сделай шаг назад короче.",
      "prepVoice": "Икроножные мышцы. Встань лицом к стене, правую ногу отставь назад.",
      "cues": [
        {"at": 0, "text": "Пятка на полу, корпус чуть к стене. Удерживай."}
      ]
    },
    {
      "id": "cat_cow", "name": "Кошка-корова", "pose": "quadruped", "stage": "core",
      "short": ["Кисти под плечами, колени под тазом", "Выдох — округли спину", "Вдох — вернись к нейтрали"],
      "steps": [
        "Встань на четвереньки: кисти под плечами, колени под тазом.",
        "На выдохе плавно округли спину, как будто отталкиваешь коврик от себя.",
        "На вдохе вернись к нейтрали и допускай лишь небольшой прогиб. 6 раз."
      ],
      "avoid": "Не пытайся прогнуться как можно сильнее.",
      "easier": "Работай в очень небольшой амплитуде.",
      "prepVoice": "Кошка-корова. Встань на четвереньки: кисти под плечами, колени под тазом.",
      "cues": [
        {"at": 0, "text": "Выдох — округли спину."},
        {"at": 5, "text": "Вдох — вернись."},
        {"at": 10, "text": "Выдох — округли."},
        {"at": 15, "text": "Вдох — вернись."},
        {"at": 20, "text": "Выдох."},
        {"at": 25, "text": "Вдох."},
        {"at": 30, "text": "Выдох."},
        {"at": 35, "text": "Вдох."},
        {"at": 40, "text": "Выдох."},
        {"at": 45, "text": "Вдох."},
        {"at": 50, "text": "Выдох."},
        {"at": 55, "text": "Вдох, вернись к нейтрали."}
      ]
    },
    {
      "id": "bird_dog", "name": "Птица-собака облегчённая", "pose": "quadruped", "stage": "core",
      "short": ["Живот слегка напряжён", "Нога назад, таз не разворачивай", "Удержи 3 секунды, смени ногу"],
      "steps": [
        "В положении на четвереньках слегка напряги живот.",
        "Не разворачивая таз, медленно отведи правую ногу назад — носок можно оставить на полу. Удержи 3 секунды и верни.",
        "Чередуй ноги, по 4–5 раз на каждую. Если легко и устойчиво, добавь противоположную руку."
      ],
      "avoid": "Не поднимай ногу выше таза и не разворачивай корпус.",
      "easier": "Оставляй пальцы вытянутой ноги на полу.",
      "prepVoice": "Птица-собака. Оставайся на четвереньках, слегка напряги живот.",
      "cues": [
        {"at": 0, "text": "Правая нога назад, удерживай."},
        {"at": 8, "text": "Верни. Левая нога назад."},
        {"at": 16, "text": "Правая."},
        {"at": 24, "text": "Левая."},
        {"at": 32, "text": "Правая."},
        {"at": 40, "text": "Левая."},
        {"at": 48, "text": "Правая."},
        {"at": 56, "text": "Левая."},
        {"at": 64, "text": "Если устойчиво, добавь противоположную руку. Правая нога."},
        {"at": 74, "text": "Левая нога."},
        {"at": 84, "text": "Верни и отдохни."}
      ]
    },
    {
      "id": "prone_yt", "name": "Подъём рук на животе (Y и T)", "pose": "prone", "stage": "core",
      "short": ["Лоб на сложенном полотенце", "Сведи лопатки, приподними руки на 2–3 см", "Сначала буквой Y, потом T"],
      "steps": [
        "Ляг на живот, лоб на сложенное полотенце. Вытяни руки вперёд под углом, как буква Y, большие пальцы вверх.",
        "На выдохе мягко сведи лопатки и приподними руки на 2–3 сантиметра. Удержи 3 секунды и опусти. 5 раз.",
        "Разведи руки в стороны, как буква T, и повтори 5 раз.",
        "Шея длинная, взгляд в пол."
      ],
      "avoid": "Не запрокидывай голову и не отрывай грудь от пола: работают лопатки и спина, не поясница.",
      "easier": "Поднимай по одной руке или просто своди лопатки, не отрывая рук.",
      "prepVoice": "Подъём рук на животе. Ляг на живот, лоб на сложенное полотенце.",
      "cues": [
        {"at": 0, "text": "Руки буквой Y. Сведи лопатки, приподними руки, удержи и опусти."},
        {"at": 45, "text": "Теперь руки в стороны, буквой T."}
      ]
    },
    {
      "id": "side_leg", "name": "Подъём ноги лёжа на боку", "pose": "sidelying", "stage": "core", "sides": true,
      "short": ["Нижняя нога согнута", "Верхняя прямая, носок вперёд", "Медленно вверх и вниз"],
      "steps": [
        "Ляг на правый бок, нижнюю ногу согни, голову положи на руку или полотенце.",
        "Верхняя нога прямая, носок смотрит вперёд, а не вверх.",
        "Медленно подними ногу на 20–30 сантиметров, задержи на секунду и опусти. 8–10 раз.",
        "По команде повернись на другой бок."
      ],
      "avoid": "Не заваливай таз назад и не уводи ногу вперёд.",
      "easier": "Поднимай ногу ниже или согни её в колене.",
      "prepVoice": "Подъём ноги на боку. Ляг на правый бок, нижнюю ногу согни.",
      "cues": [
        {"at": 0, "text": "Поднимай верхнюю прямую ногу и медленно опускай."}
      ]
    },
    {
      "id": "dead_bug", "name": "«Мёртвый жук» облегчённый", "pose": "supine", "stage": "core",
      "short": ["Колени над тазом, руки вверх", "Поясница спокойно на коврике", "По очереди пятку к полу"],
      "steps": [
        "Ляг на спину, подними согнутые ноги: колени над тазом, голени параллельны полу. Руки вытяни вверх над плечами.",
        "Слегка напряги живот, поясница спокойно лежит на коврике.",
        "На выдохе медленно опусти правую стопу и коснись пяткой пола, верни. Затем левой. 6–8 раз каждой.",
        "Если легко, одновременно опускай за голову противоположную руку."
      ],
      "avoid": "Не давай пояснице выгибаться и не задерживай дыхание.",
      "easier": "Держи стопы на полу и по очереди приподнимай одну ногу на пару сантиметров.",
      "prepVoice": "Мёртвый жук. Ляг на спину, подними согнутые ноги, руки вверх.",
      "cues": [
        {"at": 0, "text": "На выдохе опускай пятку к полу, по очереди."},
        {"at": 45, "text": "Продолжай. Поясница спокойно лежит на коврике."}
      ]
    },
    {
      "id": "bridge", "name": "Ягодичный мостик", "pose": "supine", "stage": "core",
      "short": ["Стопы на полу, колени согнуты", "Выдох — таз вверх", "Удержи 2 секунды, опусти"],
      "steps": [
        "Стопы стоят на полу, колени согнуты, руки вдоль тела.",
        "На выдохе напряги ягодицы и подними таз до удобной высоты.",
        "Удержи 2 секунды, медленно опусти. 8–10 повторений с паузами."
      ],
      "avoid": "Не поднимай таз слишком высоко и не переноси вес на шею.",
      "easier": "Поднимай таз на 3–5 сантиметров.",
      "prepVoice": "Ягодичный мостик. Ляг на спину, колени согни, стопы на полу.",
      "cues": [
        {"at": 0, "text": "На выдохе подними таз, удержи, медленно опусти."}
      ]
    },
    {
      "id": "hamstring", "name": "Задняя поверхность бедра", "pose": "supine", "stage": "stretch", "sides": true,
      "short": ["Полотенце на стопе", "Подними ногу до умеренного натяжения", "Колено можно согнуть"],
      "steps": [
        "Ляг на спину, левую ногу согни, стопа на полу.",
        "Накинь полотенце на правую стопу, подними правую ногу и немного приблизь к себе. Колено можно согнуть.",
        "Удерживай, дыши спокойно. По команде поменяй ногу."
      ],
      "avoid": "Не тяни до боли или покалывания, не округляй спину и не дёргай ногу.",
      "easier": "Сильнее согни колено или подними ногу ниже.",
      "prepVoice": "Задняя поверхность бедра. Ляг на спину, накинь полотенце на правую стопу.",
      "cues": [
        {"at": 0, "text": "Подними ногу с полотенцем до умеренного натяжения и удерживай."}
      ]
    },
    {
      "id": "hip_flexor", "name": "Сгибатели бедра", "pose": "kneeling", "stage": "stretch", "sides": true,
      "short": ["Колено на полотенце, другая стопа впереди", "Подкрути таз, напряги ягодицу", "Таз чуть вперёд"],
      "steps": [
        "Опустись на правое колено, под колено подложи полотенце, левая стопа впереди.",
        "Слегка подкрути таз и напряги правую ягодицу.",
        "Плавно перенеси таз вперёд на пару сантиметров и удерживай. По команде поменяй ноги."
      ],
      "avoid": "Не уходи в большой выпад и не прогибай поясницу; не допускай боли в колене.",
      "easier": "Подложи под колено опору мягче; если колену неприятно, пропусти упражнение.",
      "prepVoice": "Сгибатели бедра. Опустись на правое колено, под колено полотенце, левая стопа впереди.",
      "cues": [
        {"at": 0, "text": "Подкрути таз, напряги ягодицу, чуть подай таз вперёд и удерживай."}
      ]
    },
    {
      "id": "breath", "name": "Дыхание и завершение", "pose": "standing", "stage": "breath",
      "short": ["Ладони на нижних рёбрах", "Спокойный вдох — рёбра в стороны", "Выдох — плечи расслабляются"],
      "steps": [
        "Встань устойчиво на обе стопы или сядь на стул.",
        "Положи ладони на нижние рёбра, сделай 5–6 естественных вдохов, позволяя рёбрам расширяться в стороны.",
        "На выдохе расслабляй плечи. В конце отметь, как ощущается тело."
      ],
      "avoid": "Не задерживай дыхание и не дыши глубоко через силу.",
      "easier": "Дыши как обычно, без управления глубиной.",
      "prepVoice": "Дыхание. Встань устойчиво или сядь на стул.",
      "cues": [
        {"at": 0, "text": "Ладони на нижние рёбра. Спокойный вдох, рёбра расширяются в стороны."},
        {"at": 30, "text": "На выдохе расслабляй плечи."}
      ]
    }
  ],
  "programs": [
    {
      "id": "full", "kind": "timed", "title": "Полная",
      "items": [
        {"exercise": "neck", "work": 30},
        {"exercise": "shoulders", "work": 40},
        {"exercise": "arms", "work": 30},
        {"exercise": "pelvis", "work": 40},
        {"exercise": "squats_toes", "work": 40},
        {"exercise": "single_leg", "work": 60},
        {"exercise": "wall_slides", "work": 60},
        {"exercise": "chest_stretch", "work": 60},
        {"exercise": "calves", "work": 90},
        {"exercise": "cat_cow", "work": 60},
        {"exercise": "bird_dog", "work": 90},
        {"exercise": "prone_yt", "work": 90},
        {"exercise": "side_leg", "work": 90},
        {"exercise": "dead_bug", "work": 90},
        {"exercise": "bridge", "work": 90},
        {"exercise": "hamstring", "work": 90},
        {"exercise": "hip_flexor", "work": 90},
        {"exercise": "breath", "work": 60}
      ]
    },
    {
      "id": "short", "kind": "timed", "title": "Короткая",
      "items": [
        {"exercise": "neck", "work": 30},
        {"exercise": "shoulders", "work": 40},
        {"exercise": "pelvis", "work": 40},
        {"exercise": "squats_toes", "work": 40},
        {"exercise": "cat_cow", "work": 60},
        {"exercise": "bird_dog", "work": 60},
        {"exercise": "prone_yt", "work": 60},
        {"exercise": "bridge", "work": 60},
        {"exercise": "hamstring", "work": 60},
        {"exercise": "breath", "work": 40}
      ]
    }
  ]
}
```

- [ ] **Step 5: Убедиться, что тесты проходят**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.program.*'`
Expected: PASS, 11 тестов.

- [ ] **Step 6: СТОП — показать автору тексты новых упражнений**

Показать в чате тексты (`short`, `steps`, `avoid`, `easier`, `prepVoice`, `cues`) упражнений `single_leg`, `chest_stretch`, `prone_yt`, `side_leg`, `dead_bug`, `hamstring`, `hip_flexor` и спросить, всё ли понятно и нет ли правок. Правки внести в `program.json`, снова прогнать Step 5.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/com/anfixit/anfimorning/program app/src/main/assets/program.json app/src/test/java/com/anfixit/anfimorning/program
git commit -m "feat: exercise program model, validation and program.json

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Движок занятия SessionEngine

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/session/SessionEngine.kt`
- Test: `app/src/test/java/com/anfixit/anfimorning/session/SessionEngineTest.kt`

**Interfaces:**
- Consumes: `PlannedStep`, `Exercise`, `Cue` (Task 2).
- Produces:
  - `enum class Phase { PREP, WORK, FINISHED }`
  - `data class SessionState(val index: Int, val phase: Phase, val phaseTotalMs: Long, val remainingMs: Long, val paused: Boolean, val totalRemainingMs: Long, val activeMs: Long)`
  - `sealed interface SessionEvent`: `PrepStarted(index: Int, step: PlannedStep)`, `WorkStarted(index: Int, step: PlannedStep)`, `Cue(text: String)`, `Halfway`, `SwitchSide`, `CountdownBeep(secondsLeft: Int)`, `WorkCompleted(index: Int)`, `Finished`
  - `class SessionEngine(steps: List<PlannedStep>)`: `val state: SessionState`, `fun start(nowMs: Long, fromIndex: Int = 0): List<SessionEvent>`, `fun tick(nowMs: Long): List<SessionEvent>`, `fun pause()`, `fun resume(nowMs: Long)`, `fun next(nowMs: Long): List<SessionEvent>`, `fun prev(nowMs: Long): List<SessionEvent>`, `fun extendPrep()`

Правила событий работы (время от начала работы):
- подсказки `cues` с `at == 0` — сразу после `WorkStarted`; с `0 < at < work` — в свою секунду;
- `sides == true` → `SwitchSide` на половине; иначе `Halfway` на половине, если `work ≥ 40` и на эту миллисекунду нет подсказки;
- при `work ≥ 10` — `CountdownBeep(3)`, `(2)`, `(1)` за 3, 2, 1 с до конца;
- конец работы → `WorkCompleted(index)`, затем `PrepStarted` следующего или `Finished`.

- [ ] **Step 1: Написать падающие тесты**

`app/src/test/java/com/anfixit/anfimorning/session/SessionEngineTest.kt`:
```kotlin
package com.anfixit.anfimorning.session

import com.anfixit.anfimorning.program.Cue
import com.anfixit.anfimorning.program.Exercise
import com.anfixit.anfimorning.program.PlannedStep
import com.anfixit.anfimorning.program.Pose
import com.anfixit.anfimorning.program.Stage
import org.junit.Assert.assertEquals
import org.junit.Assert.assertFalse
import org.junit.Assert.assertTrue
import org.junit.Test

class SessionEngineTest {
    private fun ex(id: String, sides: Boolean = false, cues: List<Cue> = emptyList()) = Exercise(
        id = id, name = id, pose = Pose.STANDING, stage = Stage.WARMUP, sides = sides,
        short = listOf("s"), steps = listOf("s"), avoid = "a", easier = "e",
        prepVoice = "prep $id", cues = cues,
    )

    private fun step(ex: Exercise, work: Int, prep: Int = 10) = PlannedStep(ex, work, prep)

    @Test
    fun `start enters prep of first step`() {
        val s = step(ex("a"), 40)
        val engine = SessionEngine(listOf(s))
        assertEquals(listOf(SessionEvent.PrepStarted(0, s)), engine.start(0))
        assertEquals(Phase.PREP, engine.state.phase)
        assertEquals(10_000, engine.state.remainingMs)
        assertFalse(engine.state.paused)
    }

    @Test
    fun `start can begin from a later step`() {
        val a = step(ex("a"), 40)
        val b = step(ex("b"), 40)
        val engine = SessionEngine(listOf(a, b))
        assertEquals(listOf(SessionEvent.PrepStarted(1, b)), engine.start(0, fromIndex = 1))
    }

    @Test
    fun `prep ends with WorkStarted and cue at zero`() {
        val s = step(ex("a", cues = listOf(Cue(0, "go"))), 40)
        val engine = SessionEngine(listOf(s))
        engine.start(0)
        assertEquals(listOf(SessionEvent.WorkStarted(0, s), SessionEvent.Cue("go")), engine.tick(10_000))
        assertEquals(Phase.WORK, engine.state.phase)
        assertEquals(40_000, engine.state.remainingMs)
    }

    @Test
    fun `halfway for one-sided exercise`() {
        val engine = SessionEngine(listOf(step(ex("a"), 40)))
        engine.start(0)
        engine.tick(10_000)
        assertEquals(listOf(SessionEvent.Halfway), engine.tick(30_000))
    }

    @Test
    fun `switch side instead of halfway for two-sided exercise`() {
        val engine = SessionEngine(listOf(step(ex("a", sides = true), 60)))
        engine.start(0)
        engine.tick(10_000)
        assertEquals(listOf(SessionEvent.SwitchSide), engine.tick(40_000))
    }

    @Test
    fun `no halfway when a cue is at the half`() {
        val engine = SessionEngine(listOf(step(ex("a", cues = listOf(Cue(20, "mid"))), 40)))
        engine.start(0)
        engine.tick(10_000)
        assertEquals(listOf(SessionEvent.Cue("mid")), engine.tick(30_000))
    }

    @Test
    fun `no halfway for short exercise`() {
        val engine = SessionEngine(listOf(step(ex("a"), 30)))
        engine.start(0)
        engine.tick(10_000)
        assertEquals(emptyList<SessionEvent>(), engine.tick(25_000))
    }

    @Test
    fun `cues beyond work are ignored`() {
        val engine = SessionEngine(listOf(step(ex("a", cues = listOf(Cue(50, "late"))), 30)))
        engine.start(0)
        engine.tick(10_000)
        val events = engine.tick(40_000)
        assertFalse(events.contains(SessionEvent.Cue("late")))
    }

    @Test
    fun `countdown then next prep`() {
        val a = step(ex("a"), 30)
        val b = step(ex("b"), 30)
        val engine = SessionEngine(listOf(a, b))
        engine.start(0)
        engine.tick(10_000)
        assertEquals(
            listOf(
                SessionEvent.CountdownBeep(3),
                SessionEvent.CountdownBeep(2),
                SessionEvent.CountdownBeep(1),
                SessionEvent.WorkCompleted(0),
                SessionEvent.PrepStarted(1, b),
            ),
            engine.tick(40_000),
        )
    }

    @Test
    fun `late tick crosses several phases and finishes`() {
        val a = step(ex("a"), 10)
        val b = step(ex("b"), 10)
        val engine = SessionEngine(listOf(a, b))
        engine.start(0)
        val events = engine.tick(45_000)
        assertEquals(
            listOf(
                SessionEvent.WorkStarted(0, a),
                SessionEvent.CountdownBeep(3), SessionEvent.CountdownBeep(2), SessionEvent.CountdownBeep(1),
                SessionEvent.WorkCompleted(0),
                SessionEvent.PrepStarted(1, b),
                SessionEvent.WorkStarted(1, b),
                SessionEvent.CountdownBeep(3), SessionEvent.CountdownBeep(2), SessionEvent.CountdownBeep(1),
                SessionEvent.WorkCompleted(1),
                SessionEvent.Finished,
            ),
            events,
        )
        assertEquals(Phase.FINISHED, engine.state.phase)
        assertEquals(40_000, engine.state.activeMs)
        assertEquals(emptyList<SessionEvent>(), engine.tick(50_000))
    }

    @Test
    fun `pause stops time and resume continues`() {
        val s = step(ex("a"), 30)
        val engine = SessionEngine(listOf(s))
        engine.start(0)
        engine.tick(3_000)
        engine.pause()
        assertTrue(engine.state.paused)
        assertEquals(emptyList<SessionEvent>(), engine.tick(100_000))
        assertEquals(7_000, engine.state.remainingMs)
        engine.resume(100_000)
        assertEquals(listOf(SessionEvent.WorkStarted(0, s)), engine.tick(107_000))
    }

    @Test
    fun `next and prev go to prep`() {
        val a = step(ex("a"), 30)
        val b = step(ex("b"), 30)
        val engine = SessionEngine(listOf(a, b))
        engine.start(0)
        assertEquals(listOf(SessionEvent.PrepStarted(1, b)), engine.next(1_000))
        assertEquals(listOf(SessionEvent.PrepStarted(0, a)), engine.prev(2_000))
        assertEquals(listOf(SessionEvent.PrepStarted(0, a)), engine.prev(3_000))
        engine.next(4_000)
        assertEquals(listOf(SessionEvent.Finished), engine.next(5_000))
    }

    @Test
    fun `extend prep adds ten seconds only in prep`() {
        val engine = SessionEngine(listOf(step(ex("a"), 30)))
        engine.start(0)
        engine.extendPrep()
        assertEquals(20_000, engine.state.remainingMs)
        assertEquals(20_000, engine.state.phaseTotalMs)
        engine.tick(20_000)
        engine.extendPrep()
        assertEquals(30_000, engine.state.remainingMs)
    }

    @Test
    fun `total remaining counts current and later steps`() {
        val engine = SessionEngine(listOf(step(ex("a"), 40), step(ex("b"), 20)))
        engine.start(0)
        assertEquals(80_000, engine.state.totalRemainingMs)
        engine.tick(15_000)
        assertEquals(65_000, engine.state.totalRemainingMs)
    }
}
```

- [ ] **Step 2: Убедиться, что тесты падают**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.session.*'`
Expected: FAIL — `Unresolved reference: SessionEngine`.

- [ ] **Step 3: Реализовать движок**

`app/src/main/java/com/anfixit/anfimorning/session/SessionEngine.kt`:
```kotlin
package com.anfixit.anfimorning.session

import com.anfixit.anfimorning.program.PlannedStep

enum class Phase { PREP, WORK, FINISHED }

data class SessionState(
    val index: Int,
    val phase: Phase,
    val phaseTotalMs: Long,
    val remainingMs: Long,
    val paused: Boolean,
    val totalRemainingMs: Long,
    val activeMs: Long,
)

sealed interface SessionEvent {
    data class PrepStarted(val index: Int, val step: PlannedStep) : SessionEvent
    data class WorkStarted(val index: Int, val step: PlannedStep) : SessionEvent
    data class Cue(val text: String) : SessionEvent
    data object Halfway : SessionEvent
    data object SwitchSide : SessionEvent
    data class CountdownBeep(val secondsLeft: Int) : SessionEvent
    data class WorkCompleted(val index: Int) : SessionEvent
    data object Finished : SessionEvent
}

class SessionEngine(private val steps: List<PlannedStep>) {
    init {
        require(steps.isNotEmpty()) { "Пустая программа" }
    }

    private var index = 0
    private var phase = Phase.PREP
    private var phaseTotalMs = 0L
    private var remainingMs = 0L
    private var paused = true
    private var started = false
    private var lastNowMs: Long? = null
    private var activeMs = 0L

    val state: SessionState
        get() = SessionState(index, phase, phaseTotalMs, remainingMs, paused, totalRemaining(), activeMs)

    fun start(nowMs: Long, fromIndex: Int = 0): List<SessionEvent> {
        check(!started) { "Занятие уже начато" }
        started = true
        paused = false
        lastNowMs = nowMs
        return enterPrep(fromIndex.coerceIn(steps.indices))
    }

    fun tick(nowMs: Long): List<SessionEvent> {
        if (!started || paused || phase == Phase.FINISHED) return emptyList()
        val last = lastNowMs
        lastNowMs = nowMs
        if (last == null) return emptyList()
        var delta = (nowMs - last).coerceAtLeast(0)
        activeMs += delta
        val events = mutableListOf<SessionEvent>()
        while (delta > 0 && phase != Phase.FINISHED) {
            if (delta < remainingMs) {
                if (phase == Phase.WORK) events += workEventsBetween(elapsedWork(), elapsedWork() + delta)
                remainingMs -= delta
                delta = 0
            } else {
                delta -= remainingMs
                if (phase == Phase.WORK) {
                    events += workEventsBetween(elapsedWork(), phaseTotalMs)
                    remainingMs = 0
                    events += SessionEvent.WorkCompleted(index)
                    events += if (index + 1 < steps.size) enterPrep(index + 1) else finish()
                } else {
                    remainingMs = 0
                    events += enterWork()
                }
            }
        }
        activeMs -= delta
        return events
    }

    fun pause() {
        if (!started || paused || phase == Phase.FINISHED) return
        paused = true
        lastNowMs = null
    }

    fun resume(nowMs: Long) {
        if (!started || !paused || phase == Phase.FINISHED) return
        paused = false
        lastNowMs = nowMs
    }

    fun next(nowMs: Long): List<SessionEvent> {
        if (!started || phase == Phase.FINISHED) return emptyList()
        if (!paused) lastNowMs = nowMs
        return if (index + 1 < steps.size) enterPrep(index + 1) else finish()
    }

    fun prev(nowMs: Long): List<SessionEvent> {
        if (!started || phase == Phase.FINISHED) return emptyList()
        if (!paused) lastNowMs = nowMs
        return enterPrep((index - 1).coerceAtLeast(0))
    }

    fun extendPrep() {
        if (phase != Phase.PREP) return
        remainingMs += EXTEND_MS
        phaseTotalMs += EXTEND_MS
    }

    private fun enterPrep(i: Int): List<SessionEvent> {
        index = i
        phase = Phase.PREP
        phaseTotalMs = steps[i].prepSec * 1000L
        remainingMs = phaseTotalMs
        return listOf(SessionEvent.PrepStarted(i, steps[i]))
    }

    private fun enterWork(): List<SessionEvent> {
        val step = steps[index]
        phase = Phase.WORK
        phaseTotalMs = step.workSec * 1000L
        remainingMs = phaseTotalMs
        return listOf(SessionEvent.WorkStarted(index, step)) +
            step.exercise.cues.filter { it.at == 0 }.map { SessionEvent.Cue(it.text) }
    }

    private fun finish(): List<SessionEvent> {
        phase = Phase.FINISHED
        phaseTotalMs = 0
        remainingMs = 0
        return listOf(SessionEvent.Finished)
    }

    private fun elapsedWork(): Long = phaseTotalMs - remainingMs

    /** События работы в интервале (fromMs; toMs]. */
    private fun workEventsBetween(fromMs: Long, toMs: Long): List<SessionEvent> {
        val step = steps[index]
        val workMs = step.workSec * 1000L
        val timed = mutableListOf<Pair<Long, SessionEvent>>()
        step.exercise.cues
            .filter { it.at > 0 && it.at < step.workSec }
            .forEach { timed += it.at * 1000L to SessionEvent.Cue(it.text) }
        val halfMs = workMs / 2
        if (step.exercise.sides) {
            timed += halfMs to SessionEvent.SwitchSide
        } else if (step.workSec >= HALFWAY_MIN_SEC && timed.none { it.first == halfMs }) {
            timed += halfMs to SessionEvent.Halfway
        }
        if (step.workSec >= COUNTDOWN_MIN_SEC) {
            for (s in 3 downTo 1) timed += (workMs - s * 1000L) to SessionEvent.CountdownBeep(s)
        }
        return timed
            .filter { it.first > fromMs && it.first <= toMs }
            .sortedBy { it.first }
            .map { it.second }
    }

    private fun totalRemaining(): Long {
        if (phase == Phase.FINISHED) return 0
        val current = remainingMs + if (phase == Phase.PREP) steps[index].workSec * 1000L else 0
        val later = steps.drop(index + 1).sumOf { (it.prepSec + it.workSec) * 1000L }
        return current + later
    }

    private companion object {
        const val EXTEND_MS = 10_000L
        const val HALFWAY_MIN_SEC = 40
        const val COUNTDOWN_MIN_SEC = 10
    }
}
```

- [ ] **Step 4: Убедиться, что тесты проходят**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.session.*'`
Expected: PASS, 14 тестов.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/anfixit/anfimorning/session app/src/test/java/com/anfixit/anfimorning/session
git commit -m "feat: session engine with prep, cues, countdown and pause

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: История занятий и серия дней

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/history/SessionRecord.kt`, `SessionDao.kt`, `AppDatabase.kt`, `HistoryRepository.kt`, `StreakCalculator.kt`
- Test: `app/src/test/java/com/anfixit/anfimorning/history/StreakCalculatorTest.kt`

**Interfaces:**
- Produces:
  - `enum class Feeling { GOOD, OK, DISCOMFORT }`
  - `@Entity data class SessionRecord(id: Long = 0, startedAt: Long, programId: String, plannedSec: Int, actualSec: Int, exercisesDone: Int, exercisesTotal: Int, completed: Boolean, feeling: Feeling? = null, note: String? = null)`
  - `fun SessionRecord.localDate(zone: ZoneId): LocalDate`
  - `class HistoryRepository(dao: SessionDao)`: `fun observeAll(): Flow<List<SessionRecord>>`, `suspend fun get(id: Long): SessionRecord?`, `suspend fun save(record: SessionRecord): Long` (insert при `id == 0`, иначе update; возвращает id), `suspend fun setFeeling(id: Long, feeling: Feeling?, note: String?)`, `suspend fun completedOn(date: LocalDate, zone: ZoneId = ZoneId.systemDefault()): Boolean`
  - `object StreakCalculator { fun streak(completedDays: Set<LocalDate>, today: LocalDate): Int; fun completedDays(records: List<SessionRecord>, zone: ZoneId): Set<LocalDate> }`
  - `abstract class AppDatabase { fun sessionDao(): SessionDao; companion object { fun create(context: Context): AppDatabase } }`

- [ ] **Step 1: Написать падающий тест**

`app/src/test/java/com/anfixit/anfimorning/history/StreakCalculatorTest.kt`:
```kotlin
package com.anfixit.anfimorning.history

import org.junit.Assert.assertEquals
import org.junit.Test
import java.time.LocalDate
import java.time.ZoneId
import java.time.ZonedDateTime

class StreakCalculatorTest {
    private val today = LocalDate.of(2026, 11, 8)
    private fun days(vararg offsets: Long) = offsets.map { today.minusDays(it) }.toSet()

    @Test
    fun `no sessions gives zero`() = assertEquals(0, StreakCalculator.streak(emptySet(), today))

    @Test
    fun `only today gives one`() = assertEquals(1, StreakCalculator.streak(days(0), today))

    @Test
    fun `streak counts from yesterday when today is empty`() =
        assertEquals(2, StreakCalculator.streak(days(1, 2), today))

    @Test
    fun `gap stops the streak`() = assertEquals(2, StreakCalculator.streak(days(0, 1, 3, 4), today))

    @Test
    fun `old sessions only give zero`() = assertEquals(0, StreakCalculator.streak(days(2, 3), today))

    @Test
    fun `completed days skip incomplete sessions and respect zone`() {
        val samara = ZoneId.of("Europe/Samara")
        fun record(iso: String, completed: Boolean) = SessionRecord(
            startedAt = ZonedDateTime.parse(iso).toInstant().toEpochMilli(),
            programId = "full", plannedSec = 1400, actualSec = 1400,
            exercisesDone = 18, exercisesTotal = 18, completed = completed,
        )
        val result = StreakCalculator.completedDays(
            listOf(
                record("2026-11-08T22:30:00Z", completed = true),
                record("2026-11-07T06:00:00Z", completed = false),
            ),
            samara,
        )
        assertEquals(setOf(LocalDate.of(2026, 11, 9)), result)
    }
}
```

- [ ] **Step 2: Убедиться, что тест падает**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.history.*'`
Expected: FAIL — `Unresolved reference: StreakCalculator`.

- [ ] **Step 3: Реализовать историю**

`app/src/main/java/com/anfixit/anfimorning/history/SessionRecord.kt`:
```kotlin
package com.anfixit.anfimorning.history

import androidx.room.Entity
import androidx.room.PrimaryKey
import androidx.room.TypeConverter
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneId

enum class Feeling { GOOD, OK, DISCOMFORT }

@Entity(tableName = "session")
data class SessionRecord(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val startedAt: Long,
    val programId: String,
    val plannedSec: Int,
    val actualSec: Int,
    val exercisesDone: Int,
    val exercisesTotal: Int,
    val completed: Boolean,
    val feeling: Feeling? = null,
    val note: String? = null,
)

fun SessionRecord.localDate(zone: ZoneId): LocalDate =
    Instant.ofEpochMilli(startedAt).atZone(zone).toLocalDate()

class Converters {
    @TypeConverter
    fun feelingToString(value: Feeling?): String? = value?.name

    @TypeConverter
    fun stringToFeeling(value: String?): Feeling? = value?.let(Feeling::valueOf)
}
```

`app/src/main/java/com/anfixit/anfimorning/history/SessionDao.kt`:
```kotlin
package com.anfixit.anfimorning.history

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Update
import kotlinx.coroutines.flow.Flow

@Dao
interface SessionDao {
    @Insert
    suspend fun insert(record: SessionRecord): Long

    @Update
    suspend fun update(record: SessionRecord)

    @Query("SELECT * FROM session WHERE id = :id")
    suspend fun get(id: Long): SessionRecord?

    @Query("SELECT * FROM session ORDER BY startedAt DESC")
    fun observeAll(): Flow<List<SessionRecord>>

    @Query("SELECT COUNT(*) FROM session WHERE completed = 1 AND startedAt >= :fromMs AND startedAt < :toMs")
    suspend fun countCompletedBetween(fromMs: Long, toMs: Long): Int
}
```

`app/src/main/java/com/anfixit/anfimorning/history/AppDatabase.kt`:
```kotlin
package com.anfixit.anfimorning.history

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import androidx.room.TypeConverters

@Database(entities = [SessionRecord::class], version = 1, exportSchema = false)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun sessionDao(): SessionDao

    companion object {
        fun create(context: Context): AppDatabase =
            Room.databaseBuilder(context, AppDatabase::class.java, "anfimorning.db").build()
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/history/HistoryRepository.kt`:
```kotlin
package com.anfixit.anfimorning.history

import kotlinx.coroutines.flow.Flow
import java.time.LocalDate
import java.time.ZoneId

class HistoryRepository(private val dao: SessionDao) {
    fun observeAll(): Flow<List<SessionRecord>> = dao.observeAll()

    suspend fun get(id: Long): SessionRecord? = dao.get(id)

    suspend fun save(record: SessionRecord): Long =
        if (record.id == 0L) dao.insert(record) else record.id.also { dao.update(record) }

    suspend fun setFeeling(id: Long, feeling: Feeling?, note: String?) {
        val record = dao.get(id) ?: return
        dao.update(record.copy(feeling = feeling, note = note?.trim()?.ifEmpty { null }))
    }

    suspend fun completedOn(date: LocalDate, zone: ZoneId = ZoneId.systemDefault()): Boolean {
        val from = date.atStartOfDay(zone).toInstant().toEpochMilli()
        val to = date.plusDays(1).atStartOfDay(zone).toInstant().toEpochMilli()
        return dao.countCompletedBetween(from, to) > 0
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/history/StreakCalculator.kt`:
```kotlin
package com.anfixit.anfimorning.history

import java.time.LocalDate
import java.time.ZoneId

object StreakCalculator {
    fun streak(completedDays: Set<LocalDate>, today: LocalDate): Int {
        var day = if (today in completedDays) today else today.minusDays(1)
        var count = 0
        while (day in completedDays) {
            count++
            day = day.minusDays(1)
        }
        return count
    }

    fun completedDays(records: List<SessionRecord>, zone: ZoneId): Set<LocalDate> =
        records.filter { it.completed }.map { it.localDate(zone) }.toSet()
}
```

- [ ] **Step 4: Тесты и сборка (проверка, что Room-процессор отработал)**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.history.*' assembleDebug`
Expected: PASS, 6 тестов; `BUILD SUCCESSFUL`.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/anfixit/anfimorning/history app/src/test/java/com/anfixit/anfimorning/history
git commit -m "feat: session history in Room and streak calculation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Настройки, точка продолжения и расчёт напоминания

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/settings/AppSettings.kt`, `SettingsRepository.kt`, `NextReminderCalculator.kt`
- Test: `app/src/test/java/com/anfixit/anfimorning/settings/NextReminderCalculatorTest.kt`, `ResumePointTest.kt`

**Interfaces:**
- Produces:
  - `data class ReminderSettings(val enabled: Boolean = false, val hour: Int = 7, val minute: Int = 30, val days: Set<DayOfWeek> = DEFAULT_REMINDER_DAYS)`; `val DEFAULT_REMINDER_DAYS: Set<DayOfWeek>` (пн–пт)
  - `data class AppSettings(val voice: Boolean = true, val beeps: Boolean = true, val speechRate: Float = 1.0f, val reminder: ReminderSettings = ReminderSettings())`
  - `data class ResumePoint(val programId: String, val index: Int, val savedAtMs: Long) { fun isFresh(nowMs: Long): Boolean }` — свежая 3 часа
  - `val Context.settingsDataStore: DataStore<Preferences>`
  - `class SettingsRepository(store: DataStore<Preferences>)`: `val settings: Flow<AppSettings>`, `suspend fun setVoice(Boolean)`, `suspend fun setBeeps(Boolean)`, `suspend fun setSpeechRate(Float)`, `suspend fun setReminder(ReminderSettings)`, `val resumePoint: Flow<ResumePoint?>`, `suspend fun saveResume(ResumePoint)`, `suspend fun clearResume()`
  - `object NextReminderCalculator { fun next(now: ZonedDateTime, hour: Int, minute: Int, days: Set<DayOfWeek>): ZonedDateTime? }`

- [ ] **Step 1: Написать падающие тесты**

`app/src/test/java/com/anfixit/anfimorning/settings/NextReminderCalculatorTest.kt`:
```kotlin
package com.anfixit.anfimorning.settings

import org.junit.Assert.assertEquals
import org.junit.Assert.assertNull
import org.junit.Test
import java.time.DayOfWeek
import java.time.ZonedDateTime

class NextReminderCalculatorTest {
    private val weekdays = DEFAULT_REMINDER_DAYS

    private fun at(iso: String) = ZonedDateTime.parse(iso)

    @Test
    fun `later today`() {
        // 2026-11-09 — понедельник
        val next = NextReminderCalculator.next(at("2026-11-09T06:00+04:00[Europe/Samara]"), 7, 30, weekdays)
        assertEquals(at("2026-11-09T07:30+04:00[Europe/Samara]"), next)
    }

    @Test
    fun `already passed today goes to next allowed day`() {
        val next = NextReminderCalculator.next(at("2026-11-09T07:30+04:00[Europe/Samara]"), 7, 30, weekdays)
        assertEquals(at("2026-11-10T07:30+04:00[Europe/Samara]"), next)
    }

    @Test
    fun `friday evening skips weekend`() {
        val next = NextReminderCalculator.next(at("2026-11-13T20:00+04:00[Europe/Samara]"), 7, 30, weekdays)
        assertEquals(at("2026-11-16T07:30+04:00[Europe/Samara]"), next)
    }

    @Test
    fun `single day wraps to next week`() {
        val next = NextReminderCalculator.next(
            at("2026-11-09T08:00+04:00[Europe/Samara]"), 7, 30, setOf(DayOfWeek.MONDAY),
        )
        assertEquals(at("2026-11-16T07:30+04:00[Europe/Samara]"), next)
    }

    @Test
    fun `no days means no reminder`() {
        assertNull(NextReminderCalculator.next(at("2026-11-09T06:00+04:00[Europe/Samara]"), 7, 30, emptySet()))
    }

    @Test
    fun `time in DST gap moves forward`() {
        // 2026-03-29 02:00 → 03:00 в Берлине; 2026-03-29 — воскресенье
        val next = NextReminderCalculator.next(
            at("2026-03-29T00:00+01:00[Europe/Berlin]"), 2, 30, setOf(DayOfWeek.SUNDAY),
        )
        assertEquals(at("2026-03-29T03:30+02:00[Europe/Berlin]"), next)
    }
}
```

`app/src/test/java/com/anfixit/anfimorning/settings/ResumePointTest.kt`:
```kotlin
package com.anfixit.anfimorning.settings

import org.junit.Assert.assertFalse
import org.junit.Assert.assertTrue
import org.junit.Test

class ResumePointTest {
    private val point = ResumePoint("full", 5, savedAtMs = 1_000_000)

    @Test
    fun `fresh within three hours`() = assertTrue(point.isFresh(1_000_000 + 3 * 3_600_000L))

    @Test
    fun `stale after three hours`() = assertFalse(point.isFresh(1_000_000 + 3 * 3_600_000L + 1))

    @Test
    fun `saved in the future is stale`() = assertFalse(point.isFresh(999_999))
}
```

- [ ] **Step 2: Убедиться, что тесты падают**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.settings.*'`
Expected: FAIL — `Unresolved reference: NextReminderCalculator`.

- [ ] **Step 3: Реализовать**

`app/src/main/java/com/anfixit/anfimorning/settings/AppSettings.kt`:
```kotlin
package com.anfixit.anfimorning.settings

import java.time.DayOfWeek

val DEFAULT_REMINDER_DAYS: Set<DayOfWeek> = setOf(
    DayOfWeek.MONDAY, DayOfWeek.TUESDAY, DayOfWeek.WEDNESDAY, DayOfWeek.THURSDAY, DayOfWeek.FRIDAY,
)

data class ReminderSettings(
    val enabled: Boolean = false,
    val hour: Int = 7,
    val minute: Int = 30,
    val days: Set<DayOfWeek> = DEFAULT_REMINDER_DAYS,
)

data class AppSettings(
    val voice: Boolean = true,
    val beeps: Boolean = true,
    val speechRate: Float = 1.0f,
    val reminder: ReminderSettings = ReminderSettings(),
)

data class ResumePoint(val programId: String, val index: Int, val savedAtMs: Long) {
    fun isFresh(nowMs: Long): Boolean = nowMs - savedAtMs in 0..MAX_AGE_MS

    companion object {
        const val MAX_AGE_MS = 3 * 60 * 60 * 1000L
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/settings/NextReminderCalculator.kt`:
```kotlin
package com.anfixit.anfimorning.settings

import java.time.DayOfWeek
import java.time.LocalTime
import java.time.ZonedDateTime

object NextReminderCalculator {
    fun next(now: ZonedDateTime, hour: Int, minute: Int, days: Set<DayOfWeek>): ZonedDateTime? {
        if (days.isEmpty()) return null
        val time = LocalTime.of(hour, minute)
        for (offset in 0L..7L) {
            val date = now.toLocalDate().plusDays(offset)
            if (date.dayOfWeek !in days) continue
            val candidate = ZonedDateTime.of(date, time, now.zone)
            if (candidate.isAfter(now)) return candidate
        }
        return null
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/settings/SettingsRepository.kt`:
```kotlin
package com.anfixit.anfimorning.settings

import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.floatPreferencesKey
import androidx.datastore.preferences.core.intPreferencesKey
import androidx.datastore.preferences.core.longPreferencesKey
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.core.stringSetPreferencesKey
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import java.time.DayOfWeek

val Context.settingsDataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

class SettingsRepository(private val store: DataStore<Preferences>) {
    private object Keys {
        val voice = booleanPreferencesKey("voice")
        val beeps = booleanPreferencesKey("beeps")
        val rate = floatPreferencesKey("speech_rate")
        val reminderOn = booleanPreferencesKey("reminder_enabled")
        val reminderHour = intPreferencesKey("reminder_hour")
        val reminderMinute = intPreferencesKey("reminder_minute")
        val reminderDays = stringSetPreferencesKey("reminder_days")
        val resumeProgram = stringPreferencesKey("resume_program")
        val resumeIndex = intPreferencesKey("resume_index")
        val resumeAt = longPreferencesKey("resume_saved_at")
    }

    val settings: Flow<AppSettings> = store.data.map { p ->
        AppSettings(
            voice = p[Keys.voice] ?: true,
            beeps = p[Keys.beeps] ?: true,
            speechRate = p[Keys.rate] ?: 1.0f,
            reminder = ReminderSettings(
                enabled = p[Keys.reminderOn] ?: false,
                hour = p[Keys.reminderHour] ?: 7,
                minute = p[Keys.reminderMinute] ?: 30,
                days = p[Keys.reminderDays]?.map(DayOfWeek::valueOf)?.toSet() ?: DEFAULT_REMINDER_DAYS,
            ),
        )
    }

    val resumePoint: Flow<ResumePoint?> = store.data.map { p ->
        val program = p[Keys.resumeProgram] ?: return@map null
        ResumePoint(program, p[Keys.resumeIndex] ?: 0, p[Keys.resumeAt] ?: 0L)
    }

    suspend fun setVoice(value: Boolean) {
        store.edit { it[Keys.voice] = value }
    }

    suspend fun setBeeps(value: Boolean) {
        store.edit { it[Keys.beeps] = value }
    }

    suspend fun setSpeechRate(value: Float) {
        store.edit { it[Keys.rate] = value }
    }

    suspend fun setReminder(value: ReminderSettings) {
        store.edit {
            it[Keys.reminderOn] = value.enabled
            it[Keys.reminderHour] = value.hour
            it[Keys.reminderMinute] = value.minute
            it[Keys.reminderDays] = value.days.map(DayOfWeek::name).toSet()
        }
    }

    suspend fun saveResume(point: ResumePoint) {
        store.edit {
            it[Keys.resumeProgram] = point.programId
            it[Keys.resumeIndex] = point.index
            it[Keys.resumeAt] = point.savedAtMs
        }
    }

    suspend fun clearResume() {
        store.edit {
            it.remove(Keys.resumeProgram)
            it.remove(Keys.resumeIndex)
            it.remove(Keys.resumeAt)
        }
    }
}
```

- [ ] **Step 4: Убедиться, что тесты проходят**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.settings.*'`
Expected: PASS, 9 тестов.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/com/anfixit/anfimorning/settings app/src/test/java/com/anfixit/anfimorning/settings
git commit -m "feat: settings, resume point and next reminder calculation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Контейнер, тема, навигация и главный экран

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/AnfimorningApp.kt`, `AppContainer.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/MainActivity.kt` (полностью заменить)
- Modify: `app/src/main/AndroidManifest.xml` (атрибут `android:name` у `<application>`)
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/Theme.kt`, `Format.kt`, `AppNav.kt`, `Effects.kt`
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/home/HomeViewModel.kt`, `HomeScreen.kt`
- Create (временные заглушки, заменяются в задачах 8–10, 12): `ui/session/SessionScreen.kt`, `ui/done/DoneScreen.kt`, `ui/calendar/CalendarScreen.kt`, `ui/settings/SettingsScreen.kt`
- Test: `app/src/test/java/com/anfixit/anfimorning/ui/FormatTest.kt`

**Interfaces:**
- Consumes: всё из задач 2–5.
- Produces:
  - `class AppContainer(context: Context)`: `val program: Result<ProgramFile>`, `val history: HistoryRepository`, `val settings: SettingsRepository`
  - `class AnfimorningApp : Application { lateinit var container: AppContainer }`
  - `object Palette` (цвета из Global Constraints: `Bg, Panel, Panel2, Ink, Muted, Accent, Mint, Blue, Lilac, Yellow, Coral`), `fun Stage.color(): Color`, `@Composable fun AnfiTheme(content)`
  - `fun formatMs(ms: Long): String` («мм:сс», округление вверх), `fun daysWord(n: Int): String`
  - `object Routes { HOME, CALENDAR, SETTINGS, SESSION, DONE; fun session(programId: String, from: Int = 0); fun done(recordId: Long) }`
  - `@Composable fun KeepScreenOn()`, `@Composable fun OnLifecycleEvent(event: Lifecycle.Event, action: () -> Unit)`
  - Сигнатуры экранов: `HomeScreen(container, onStart: (programId: String, from: Int) -> Unit)`, `SessionScreen(container, programId: String, from: Int, onFinished: (Long) -> Unit, onStopped: () -> Unit)`, `DoneScreen(container, recordId: Long, onClose: () -> Unit)`, `CalendarScreen(container)`, `SettingsScreen(container)`

- [ ] **Step 1: Написать падающий тест форматирования**

`app/src/test/java/com/anfixit/anfimorning/ui/FormatTest.kt`:
```kotlin
package com.anfixit.anfimorning.ui

import org.junit.Assert.assertEquals
import org.junit.Test

class FormatTest {
    @Test
    fun `formats milliseconds rounding up`() {
        assertEquals("00:00", formatMs(0))
        assertEquals("00:01", formatMs(1))
        assertEquals("01:01", formatMs(61_000))
        assertEquals("23:20", formatMs(1_400_000))
        assertEquals("00:00", formatMs(-500))
    }

    @Test
    fun `russian plural for days`() {
        assertEquals("день", daysWord(1))
        assertEquals("дня", daysWord(3))
        assertEquals("дней", daysWord(5))
        assertEquals("дней", daysWord(11))
        assertEquals("день", daysWord(21))
        assertEquals("дня", daysWord(22))
        assertEquals("дней", daysWord(112))
    }
}
```

- [ ] **Step 2: Убедиться, что тест падает**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.ui.*'`
Expected: FAIL — `Unresolved reference: formatMs`.

- [ ] **Step 3: Format и тема**

`app/src/main/java/com/anfixit/anfimorning/ui/Format.kt`:
```kotlin
package com.anfixit.anfimorning.ui

fun formatMs(ms: Long): String {
    val seconds = ((ms.coerceAtLeast(0) + 999) / 1000)
    return "%02d:%02d".format(seconds / 60, seconds % 60)
}

fun daysWord(n: Int): String {
    val mod100 = n % 100
    val mod10 = n % 10
    return when {
        mod100 in 11..14 -> "дней"
        mod10 == 1 -> "день"
        mod10 in 2..4 -> "дня"
        else -> "дней"
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/Theme.kt`:
```kotlin
package com.anfixit.anfimorning.ui

import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color
import com.anfixit.anfimorning.history.Feeling
import com.anfixit.anfimorning.program.Stage

object Palette {
    val Bg = Color(0xFF0D1A1E)
    val Panel = Color(0xFF15272B)
    val Panel2 = Color(0xFF1B3135)
    val Ink = Color(0xFFF2F7F4)
    val Muted = Color(0xFFB2C4C1)
    val Accent = Color(0xFFA9E9C8)
    val Mint = Color(0xFF7BC9A6)
    val Blue = Color(0xFF88B6ED)
    val Lilac = Color(0xFFC0ADEE)
    val Yellow = Color(0xFFEED39A)
    val Coral = Color(0xFFFFB4A2)
    val OnAccent = Color(0xFF102E28)
}

fun Stage.color(): Color = when (this) {
    Stage.WARMUP -> Palette.Mint
    Stage.CORE -> Palette.Blue
    Stage.STRETCH -> Palette.Lilac
    Stage.BREATH -> Palette.Yellow
}

fun Feeling?.color(): Color = when (this) {
    Feeling.GOOD -> Palette.Mint
    Feeling.OK -> Palette.Yellow
    Feeling.DISCOMFORT -> Palette.Coral
    null -> Palette.Muted
}

fun Feeling.label(): String = when (this) {
    Feeling.GOOD -> "😊 хорошо"
    Feeling.OK -> "😐 нормально"
    Feeling.DISCOMFORT -> "😣 был дискомфорт"
}

@Composable
fun AnfiTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = darkColorScheme(
            primary = Palette.Accent,
            onPrimary = Palette.OnAccent,
            secondary = Palette.Blue,
            background = Palette.Bg,
            onBackground = Palette.Ink,
            surface = Palette.Panel,
            onSurface = Palette.Ink,
            surfaceVariant = Palette.Panel2,
            onSurfaceVariant = Palette.Muted,
        ),
        content = content,
    )
}
```

- [ ] **Step 4: Тест проходит**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.ui.*'`
Expected: PASS, 2 теста.

- [ ] **Step 5: Контейнер, Application, эффекты**

`app/src/main/java/com/anfixit/anfimorning/AppContainer.kt`:
```kotlin
package com.anfixit.anfimorning

import android.content.Context
import com.anfixit.anfimorning.history.AppDatabase
import com.anfixit.anfimorning.history.HistoryRepository
import com.anfixit.anfimorning.program.ProgramFile
import com.anfixit.anfimorning.program.ProgramParser
import com.anfixit.anfimorning.settings.SettingsRepository
import com.anfixit.anfimorning.settings.settingsDataStore

class AppContainer(context: Context) {
    private val app = context.applicationContext

    val program: Result<ProgramFile> by lazy {
        runCatching {
            ProgramParser.parse(app.assets.open("program.json").bufferedReader().use { it.readText() })
        }
    }
    private val database by lazy { AppDatabase.create(app) }
    val history by lazy { HistoryRepository(database.sessionDao()) }
    val settings by lazy { SettingsRepository(app.settingsDataStore) }
}
```

`app/src/main/java/com/anfixit/anfimorning/AnfimorningApp.kt`:
```kotlin
package com.anfixit.anfimorning

import android.app.Application

class AnfimorningApp : Application() {
    lateinit var container: AppContainer
        private set

    override fun onCreate() {
        super.onCreate()
        container = AppContainer(this)
    }
}
```

В `AndroidManifest.xml` у `<application` добавить первым атрибутом:
```xml
        android:name=".AnfimorningApp"
```

`app/src/main/java/com/anfixit/anfimorning/ui/Effects.kt`:
```kotlin
package com.anfixit.anfimorning.ui

import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.rememberUpdatedState
import androidx.compose.ui.platform.LocalView
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver
import androidx.lifecycle.compose.LocalLifecycleOwner

@Composable
fun KeepScreenOn() {
    val view = LocalView.current
    DisposableEffect(view) {
        view.keepScreenOn = true
        onDispose { view.keepScreenOn = false }
    }
}

@Composable
fun OnLifecycleEvent(event: Lifecycle.Event, action: () -> Unit) {
    val owner = LocalLifecycleOwner.current
    val current by rememberUpdatedState(action)
    DisposableEffect(owner, event) {
        val observer = LifecycleEventObserver { _, e -> if (e == event) current() }
        owner.lifecycle.addObserver(observer)
        onDispose { owner.lifecycle.removeObserver(observer) }
    }
}
```

- [ ] **Step 6: Навигация и Activity**

`app/src/main/java/com/anfixit/anfimorning/ui/AppNav.kt`:
```kotlin
package com.anfixit.anfimorning.ui

import androidx.compose.foundation.layout.padding
import androidx.compose.material3.NavigationBar
import androidx.compose.material3.NavigationBarItem
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.navigation.NavType
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.currentBackStackEntryAsState
import androidx.navigation.compose.rememberNavController
import androidx.navigation.navArgument
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.ui.calendar.CalendarScreen
import com.anfixit.anfimorning.ui.done.DoneScreen
import com.anfixit.anfimorning.ui.home.HomeScreen
import com.anfixit.anfimorning.ui.session.SessionScreen
import com.anfixit.anfimorning.ui.settings.SettingsScreen

object Routes {
    const val HOME = "home"
    const val CALENDAR = "calendar"
    const val SETTINGS = "settings"
    const val SESSION = "session/{programId}?from={from}"
    const val DONE = "done/{recordId}"
    fun session(programId: String, from: Int = 0) = "session/$programId?from=$from"
    fun done(recordId: Long) = "done/$recordId"
}

private data class Tab(val route: String, val icon: String, val label: String)

private val tabs = listOf(
    Tab(Routes.HOME, "🏠", "Главная"),
    Tab(Routes.CALENDAR, "📅", "Календарь"),
    Tab(Routes.SETTINGS, "⚙️", "Настройки"),
)

@Composable
fun AppNav(container: AppContainer) {
    val nav = rememberNavController()
    val entry by nav.currentBackStackEntryAsState()
    val route = entry?.destination?.route
    Scaffold(
        containerColor = Palette.Bg,
        bottomBar = {
            if (tabs.any { it.route == route }) {
                NavigationBar(containerColor = Palette.Panel) {
                    tabs.forEach { tab ->
                        NavigationBarItem(
                            selected = route == tab.route,
                            onClick = {
                                nav.navigate(tab.route) {
                                    popUpTo(Routes.HOME) { saveState = true }
                                    launchSingleTop = true
                                    restoreState = true
                                }
                            },
                            icon = { Text(tab.icon) },
                            label = { Text(tab.label) },
                        )
                    }
                }
            }
        },
    ) { padding ->
        NavHost(nav, startDestination = Routes.HOME, modifier = Modifier.padding(padding)) {
            composable(Routes.HOME) {
                HomeScreen(container, onStart = { id, from -> nav.navigate(Routes.session(id, from)) })
            }
            composable(Routes.CALENDAR) { CalendarScreen(container) }
            composable(Routes.SETTINGS) { SettingsScreen(container) }
            composable(
                Routes.SESSION,
                arguments = listOf(
                    navArgument("programId") { type = NavType.StringType },
                    navArgument("from") { type = NavType.IntType; defaultValue = 0 },
                ),
            ) { e ->
                SessionScreen(
                    container = container,
                    programId = e.arguments?.getString("programId").orEmpty(),
                    from = e.arguments?.getInt("from") ?: 0,
                    onFinished = { id -> nav.navigate(Routes.done(id)) { popUpTo(Routes.HOME) } },
                    onStopped = { nav.popBackStack(Routes.HOME, inclusive = false) },
                )
            }
            composable(Routes.DONE, arguments = listOf(navArgument("recordId") { type = NavType.LongType })) { e ->
                DoneScreen(
                    container = container,
                    recordId = e.arguments?.getLong("recordId") ?: 0L,
                    onClose = { nav.popBackStack(Routes.HOME, inclusive = false) },
                )
            }
        }
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/MainActivity.kt` (заменить целиком):
```kotlin
package com.anfixit.anfimorning

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import com.anfixit.anfimorning.ui.AnfiTheme
import com.anfixit.anfimorning.ui.AppNav

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        val container = (application as AnfimorningApp).container
        setContent { AnfiTheme { AppNav(container) } }
    }
}
```

- [ ] **Step 7: Временные заглушки остальных экранов**

`app/src/main/java/com/anfixit/anfimorning/ui/session/SessionScreen.kt`:
```kotlin
package com.anfixit.anfimorning.ui.session

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import com.anfixit.anfimorning.AppContainer

@Composable
fun SessionScreen(
    container: AppContainer,
    programId: String,
    from: Int,
    onFinished: (Long) -> Unit,
    onStopped: () -> Unit,
) {
    Text("Занятие $programId с упражнения ${from + 1}")
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/done/DoneScreen.kt`:
```kotlin
package com.anfixit.anfimorning.ui.done

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import com.anfixit.anfimorning.AppContainer

@Composable
fun DoneScreen(container: AppContainer, recordId: Long, onClose: () -> Unit) {
    Text("Готово")
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarScreen.kt`:
```kotlin
package com.anfixit.anfimorning.ui.calendar

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import com.anfixit.anfimorning.AppContainer

@Composable
fun CalendarScreen(container: AppContainer) {
    Text("Календарь")
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/settings/SettingsScreen.kt`:
```kotlin
package com.anfixit.anfimorning.ui.settings

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import com.anfixit.anfimorning.AppContainer

@Composable
fun SettingsScreen(container: AppContainer) {
    Text("Настройки")
}
```

- [ ] **Step 8: Главный экран**

`app/src/main/java/com/anfixit/anfimorning/ui/home/HomeViewModel.kt`:
```kotlin
package com.anfixit.anfimorning.ui.home

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.history.StreakCalculator
import com.anfixit.anfimorning.program.plan
import com.anfixit.anfimorning.program.totalSeconds
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.stateIn
import java.time.LocalDate
import java.time.ZoneId

data class ProgramCard(val id: String, val title: String, val minutes: Int, val steps: Int)

data class ResumeCard(val programId: String, val index: Int, val title: String, val exerciseName: String)

data class HomeUi(
    val programs: List<ProgramCard> = emptyList(),
    val streak: Int = 0,
    val resume: ResumeCard? = null,
    val loadError: String? = null,
)

class HomeViewModel(
    container: AppContainer,
    private val nowMs: () -> Long = System::currentTimeMillis,
) : ViewModel() {
    val ui: StateFlow<HomeUi> = build(container)

    private fun build(container: AppContainer): StateFlow<HomeUi> {
        val file = container.program.getOrElse { error ->
            return MutableStateFlow(HomeUi(loadError = error.message ?: error.javaClass.simpleName))
        }
        val cards = file.programs.map { p ->
            val plan = file.plan(p.id)
            ProgramCard(p.id, p.title, (plan.totalSeconds() + 30) / 60, plan.size)
        }
        return combine(container.history.observeAll(), container.settings.resumePoint) { records, resume ->
            val zone = ZoneId.systemDefault()
            val resumeCard = resume?.takeIf { it.isFresh(nowMs()) }?.let { point ->
                val program = file.programs.firstOrNull { it.id == point.programId } ?: return@let null
                val plan = file.plan(program.id)
                val i = point.index.coerceIn(plan.indices)
                ResumeCard(program.id, i, program.title, plan[i].exercise.name)
            }
            HomeUi(
                programs = cards,
                streak = StreakCalculator.streak(StreakCalculator.completedDays(records, zone), LocalDate.now(zone)),
                resume = resumeCard,
            )
        }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), HomeUi(programs = cards))
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/home/HomeScreen.kt`:
```kotlin
package com.anfixit.anfimorning.ui.home

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.ui.Palette
import com.anfixit.anfimorning.ui.daysWord

@Composable
fun HomeScreen(container: AppContainer, onStart: (programId: String, from: Int) -> Unit) {
    val vm: HomeViewModel = viewModel(factory = viewModelFactory { initializer { HomeViewModel(container) } })
    val ui by vm.ui.collectAsStateWithLifecycle()

    Column(
        Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
    ) {
        Text("Утро в движении", style = MaterialTheme.typography.headlineMedium, fontWeight = FontWeight.Bold)
        Text(
            if (ui.streak > 0) "🔥 ${ui.streak} ${daysWord(ui.streak)} подряд" else "Начни серию сегодня",
            style = MaterialTheme.typography.titleMedium,
            color = Palette.Accent,
        )
        ui.loadError?.let { Text("Комплекс не загрузился: $it", color = Palette.Coral) }
        ui.resume?.let { r ->
            OutlinedButton(onClick = { onStart(r.programId, r.index) }, modifier = Modifier.fillMaxWidth()) {
                Text("↻ Продолжить: ${r.title}, упражнение ${r.index + 1} — ${r.exerciseName}")
            }
        }
        ui.programs.forEach { p ->
            Button(
                onClick = { onStart(p.id, 0) },
                modifier = Modifier.fillMaxWidth().height(72.dp),
                shape = RoundedCornerShape(18.dp),
            ) {
                Text("${p.title} · ${p.minutes} мин", style = MaterialTheme.typography.titleLarge)
            }
        }
        Card(colors = CardDefaults.cardColors(containerColor = Palette.Panel)) {
            Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(6.dp)) {
                Text("Перед началом", fontWeight = FontWeight.Bold)
                listOf("Коврик или нескользкий пол", "Полотенце", "Устойчивый стул", "Свободная стена")
                    .forEach { Text("✓ $it", color = Palette.Muted) }
            }
        }
        Text(
            "Двигайся без боли. При заболеваниях позвоночника согласуй комплекс с врачом ЛФК.",
            style = MaterialTheme.typography.bodySmall,
            color = Palette.Yellow,
        )
    }
}
```

- [ ] **Step 9: Сборка и все тесты**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 10: Commit**

```bash
git add -A
git commit -m "feat: app container, theme, navigation and home screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Голос и сигналы

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/guide/GuidePhrases.kt`, `VoiceGuide.kt`, `Beeper.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/AppContainer.kt` (добавить `voice`, `beeper`)
- Test: `app/src/test/java/com/anfixit/anfimorning/guide/GuidePhrasesTest.kt`

**Interfaces:**
- Consumes: `SessionEvent` (Task 3).
- Produces:
  - `object GuidePhrases { fun speech(event: SessionEvent): String?; fun isBeep(event: SessionEvent): Boolean }`
  - `class VoiceGuide(context: Context)`: `val ready: StateFlow<Boolean?>` (null — инициализация, false — нет русского голоса), `fun say(texts: List<String>, rate: Float)`, `fun stop()`
  - `class Beeper`: `fun beep()`
  - `AppContainer.voice: VoiceGuide`, `AppContainer.beeper: Beeper`

- [ ] **Step 1: Падающий тест**

`app/src/test/java/com/anfixit/anfimorning/guide/GuidePhrasesTest.kt`:
```kotlin
package com.anfixit.anfimorning.guide

import com.anfixit.anfimorning.program.Exercise
import com.anfixit.anfimorning.program.PlannedStep
import com.anfixit.anfimorning.program.Pose
import com.anfixit.anfimorning.program.Stage
import com.anfixit.anfimorning.session.SessionEvent
import org.junit.Assert.assertEquals
import org.junit.Assert.assertFalse
import org.junit.Assert.assertNull
import org.junit.Assert.assertTrue
import org.junit.Test

class GuidePhrasesTest {
    private val step = PlannedStep(
        Exercise(
            id = "a", name = "A", pose = Pose.STANDING, stage = Stage.WARMUP,
            short = listOf("s"), steps = listOf("s"), avoid = "a", easier = "e", prepVoice = "Встань прямо.",
        ),
        workSec = 30, prepSec = 10,
    )

    @Test
    fun `phrases for events`() {
        assertEquals("Встань прямо.", GuidePhrases.speech(SessionEvent.PrepStarted(0, step)))
        assertEquals("Начали.", GuidePhrases.speech(SessionEvent.WorkStarted(0, step)))
        assertEquals("тянись", GuidePhrases.speech(SessionEvent.Cue("тянись")))
        assertEquals("Половина.", GuidePhrases.speech(SessionEvent.Halfway))
        assertEquals("Смени сторону.", GuidePhrases.speech(SessionEvent.SwitchSide))
        assertEquals("Готово. Отличная работа!", GuidePhrases.speech(SessionEvent.Finished))
        assertNull(GuidePhrases.speech(SessionEvent.CountdownBeep(3)))
        assertNull(GuidePhrases.speech(SessionEvent.WorkCompleted(0)))
    }

    @Test
    fun `only countdown is a beep`() {
        assertTrue(GuidePhrases.isBeep(SessionEvent.CountdownBeep(1)))
        assertFalse(GuidePhrases.isBeep(SessionEvent.Halfway))
    }
}
```

- [ ] **Step 2: Убедиться, что падает**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.guide.*'`
Expected: FAIL — `Unresolved reference: GuidePhrases`.

- [ ] **Step 3: Реализовать**

`app/src/main/java/com/anfixit/anfimorning/guide/GuidePhrases.kt`:
```kotlin
package com.anfixit.anfimorning.guide

import com.anfixit.anfimorning.session.SessionEvent

object GuidePhrases {
    fun speech(event: SessionEvent): String? = when (event) {
        is SessionEvent.PrepStarted -> event.step.exercise.prepVoice
        is SessionEvent.WorkStarted -> "Начали."
        is SessionEvent.Cue -> event.text
        SessionEvent.Halfway -> "Половина."
        SessionEvent.SwitchSide -> "Смени сторону."
        SessionEvent.Finished -> "Готово. Отличная работа!"
        is SessionEvent.CountdownBeep, is SessionEvent.WorkCompleted -> null
    }

    fun isBeep(event: SessionEvent): Boolean = event is SessionEvent.CountdownBeep
}
```

`app/src/main/java/com/anfixit/anfimorning/guide/VoiceGuide.kt`:
```kotlin
package com.anfixit.anfimorning.guide

import android.content.Context
import android.media.AudioAttributes
import android.media.AudioFocusRequest
import android.media.AudioManager
import android.speech.tts.TextToSpeech
import android.speech.tts.UtteranceProgressListener
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import java.util.Locale

class VoiceGuide(context: Context) : TextToSpeech.OnInitListener {
    private val audioManager = context.getSystemService(AudioManager::class.java)
    private val attributes = AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_ASSISTANCE_NAVIGATION_GUIDANCE)
        .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH)
        .build()
    private val focusRequest = AudioFocusRequest.Builder(AudioManager.AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK)
        .setAudioAttributes(attributes)
        .build()
    private val tts = TextToSpeech(context.applicationContext, this)
    private val _ready = MutableStateFlow<Boolean?>(null)
    private var counter = 0L

    @Volatile
    private var lastUtterance: String? = null

    /** null — синтезатор ещё запускается, false — русского голоса нет, true — можно говорить. */
    val ready: StateFlow<Boolean?> = _ready

    override fun onInit(status: Int) {
        if (status != TextToSpeech.SUCCESS) {
            _ready.value = false
            return
        }
        val result = tts.setLanguage(Locale.forLanguageTag("ru-RU"))
        tts.setAudioAttributes(attributes)
        tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
            override fun onStart(utteranceId: String?) = Unit
            override fun onDone(utteranceId: String?) {
                if (utteranceId == lastUtterance) audioManager.abandonAudioFocusRequest(focusRequest)
            }

            @Deprecated("Deprecated in Java")
            override fun onError(utteranceId: String?) {
                audioManager.abandonAudioFocusRequest(focusRequest)
            }

            override fun onStop(utteranceId: String?, interrupted: Boolean) {
                audioManager.abandonAudioFocusRequest(focusRequest)
            }
        })
        _ready.value = result != TextToSpeech.LANG_MISSING_DATA && result != TextToSpeech.LANG_NOT_SUPPORTED
    }

    fun say(texts: List<String>, rate: Float) {
        if (_ready.value != true || texts.isEmpty()) return
        audioManager.requestAudioFocus(focusRequest)
        tts.setSpeechRate(rate)
        texts.forEachIndexed { i, text ->
            val id = "u${counter++}"
            if (i == texts.lastIndex) lastUtterance = id
            tts.speak(text, if (i == 0) TextToSpeech.QUEUE_FLUSH else TextToSpeech.QUEUE_ADD, null, id)
        }
    }

    fun stop() {
        tts.stop()
        audioManager.abandonAudioFocusRequest(focusRequest)
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/guide/Beeper.kt`:
```kotlin
package com.anfixit.anfimorning.guide

import android.media.AudioManager
import android.media.ToneGenerator

class Beeper {
    private val tone: ToneGenerator? = runCatching { ToneGenerator(AudioManager.STREAM_MUSIC, 70) }.getOrNull()

    fun beep() {
        tone?.startTone(ToneGenerator.TONE_PROP_BEEP, 150)
    }
}
```

В `AppContainer.kt` добавить импорты `com.anfixit.anfimorning.guide.Beeper`, `com.anfixit.anfimorning.guide.VoiceGuide` и в конец класса:
```kotlin
    val voice by lazy { VoiceGuide(app) }
    val beeper by lazy { Beeper() }
```

- [ ] **Step 4: Тесты и сборка**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`, тесты `guide.*` проходят.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: russian voice guide and countdown beeps

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Экран занятия

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/session/SessionViewModel.kt`, `ExerciseVideo.kt`, `TimerDial.kt`, `DetailsSheet.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/ui/session/SessionScreen.kt` (заменить заглушку)

**Interfaces:**
- Consumes: `SessionEngine`, `SessionEvent`, `Phase` (Task 3); `plan`, `totalSeconds` (Task 2); `HistoryRepository.save`, `SessionRecord` (Task 4); `SettingsRepository.settings/saveResume/clearResume`, `ResumePoint`, `AppSettings` (Task 5); `VoiceGuide`, `Beeper`, `GuidePhrases` (Task 7); `KeepScreenOn`, `OnLifecycleEvent`, `formatMs`, `Palette`, `color()` (Task 6).
- Produces: `SessionScreen(container, programId, from, onFinished: (recordId: Long) -> Unit, onStopped: () -> Unit)`; запись в истории после первого законченного упражнения; `onFinished` вызывается с id записи после `Finished`.

- [ ] **Step 1: ViewModel**

`app/src/main/java/com/anfixit/anfimorning/ui/session/SessionViewModel.kt`:
```kotlin
package com.anfixit.anfimorning.ui.session

import android.os.SystemClock
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.guide.GuidePhrases
import com.anfixit.anfimorning.history.SessionRecord
import com.anfixit.anfimorning.program.Exercise
import com.anfixit.anfimorning.program.plan
import com.anfixit.anfimorning.program.totalSeconds
import com.anfixit.anfimorning.session.Phase
import com.anfixit.anfimorning.session.SessionEngine
import com.anfixit.anfimorning.session.SessionEvent
import com.anfixit.anfimorning.settings.AppSettings
import com.anfixit.anfimorning.settings.ResumePoint
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharedFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.isActive
import kotlinx.coroutines.launch

data class SessionUi(
    val stepNumber: Int = 0,
    val stepCount: Int = 0,
    val exercise: Exercise? = null,
    val next: Exercise? = null,
    val phase: Phase = Phase.PREP,
    val paused: Boolean = false,
    val remainingMs: Long = 0,
    val phaseTotalMs: Long = 1,
    val totalRemainingMs: Long = 0,
    val totalMs: Long = 1,
    val voiceMissing: Boolean = false,
)

class SessionViewModel(
    private val container: AppContainer,
    private val programId: String,
    private val fromIndex: Int,
) : ViewModel() {
    private val plan = container.program.getOrThrow().plan(programId)
    private val engine = SessionEngine(plan)
    private val startedAtMs = System.currentTimeMillis()
    private var recordId = 0L
    private var exercisesDone = 0
    private var current = AppSettings()

    private val _ui = MutableStateFlow(SessionUi())
    val ui: StateFlow<SessionUi> = _ui

    private val _finished = MutableSharedFlow<Long>(replay = 1)
    val finished: SharedFlow<Long> = _finished

    init {
        viewModelScope.launch {
            container.settings.settings.collect {
                current = it
                refreshVoiceBanner()
            }
        }
        viewModelScope.launch { container.voice.ready.collect { refreshVoiceBanner() } }
        viewModelScope.launch {
            current = container.settings.settings.first()
            handle(engine.start(SystemClock.elapsedRealtime(), fromIndex))
            while (isActive) {
                delay(TICK_MS)
                handle(engine.tick(SystemClock.elapsedRealtime()))
            }
        }
    }

    fun togglePause() {
        if (engine.state.paused) engine.resume(SystemClock.elapsedRealtime()) else pause()
        publish()
    }

    fun pause() {
        engine.pause()
        container.voice.stop()
        publish()
    }

    fun next() {
        viewModelScope.launch { handle(engine.next(SystemClock.elapsedRealtime())) }
    }

    fun prev() {
        viewModelScope.launch { handle(engine.prev(SystemClock.elapsedRealtime())) }
    }

    fun extendPrep() {
        engine.extendPrep()
        publish()
    }

    fun stop(onDone: () -> Unit) {
        viewModelScope.launch {
            engine.pause()
            container.voice.stop()
            saveRecord(completed = false)
            container.settings.clearResume()
            onDone()
        }
    }

    private suspend fun handle(events: List<SessionEvent>) {
        if (events.isNotEmpty()) {
            if (current.voice) container.voice.say(events.mapNotNull(GuidePhrases::speech), current.speechRate)
            if (current.beeps && events.any(GuidePhrases::isBeep)) container.beeper.beep()
            for (event in events) {
                when (event) {
                    is SessionEvent.PrepStarted -> container.settings.saveResume(
                        ResumePoint(programId, event.index, System.currentTimeMillis()),
                    )
                    is SessionEvent.WorkCompleted -> {
                        exercisesDone++
                        saveRecord(completed = false)
                    }
                    SessionEvent.Finished -> {
                        saveRecord(completed = true)
                        container.settings.clearResume()
                        _finished.emit(recordId)
                    }
                    else -> Unit
                }
            }
        }
        publish()
    }

    private suspend fun saveRecord(completed: Boolean) {
        if (exercisesDone == 0) return
        recordId = container.history.save(
            SessionRecord(
                id = recordId,
                startedAt = startedAtMs,
                programId = programId,
                plannedSec = plan.totalSeconds(),
                actualSec = (engine.state.activeMs / 1000).toInt(),
                exercisesDone = exercisesDone,
                exercisesTotal = plan.size,
                completed = completed,
            ),
        )
    }

    private fun refreshVoiceBanner() {
        _ui.update { it.copy(voiceMissing = current.voice && container.voice.ready.value == false) }
    }

    private fun publish() {
        val s = engine.state
        _ui.update {
            it.copy(
                stepNumber = s.index + 1,
                stepCount = plan.size,
                exercise = plan[s.index].exercise,
                next = plan.getOrNull(s.index + 1)?.exercise,
                phase = s.phase,
                paused = s.paused,
                remainingMs = s.remainingMs,
                phaseTotalMs = s.phaseTotalMs.coerceAtLeast(1),
                totalRemainingMs = s.totalRemainingMs,
                totalMs = plan.totalSeconds() * 1000L,
            )
        }
    }

    private companion object {
        const val TICK_MS = 100L
    }
}
```

- [ ] **Step 2: Видео, круг таймера, шторка**

`app/src/main/java/com/anfixit/anfimorning/ui/session/ExerciseVideo.kt`:
```kotlin
package com.anfixit.anfimorning.ui.session

import android.annotation.SuppressLint
import androidx.annotation.OptIn
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.Box
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.remember
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.viewinterop.AndroidView
import androidx.media3.common.MediaItem
import androidx.media3.common.Player
import androidx.media3.common.util.UnstableApi
import androidx.media3.datasource.RawResourceDataSource
import androidx.media3.exoplayer.ExoPlayer
import androidx.media3.ui.AspectRatioFrameLayout
import androidx.media3.ui.PlayerView
import com.anfixit.anfimorning.program.Exercise
import com.anfixit.anfimorning.ui.Palette

@SuppressLint("DiscouragedApi")
@Composable
fun ExerciseVideo(exercise: Exercise, modifier: Modifier = Modifier) {
    val context = LocalContext.current
    val resId = remember(exercise.video) {
        exercise.video
            ?.let { context.resources.getIdentifier(it, "raw", context.packageName) }
            ?.takeIf { it != 0 }
    }
    if (resId == null) {
        Box(modifier.background(Palette.Panel2), contentAlignment = Alignment.Center) {
            Text(exercise.pose.label, style = MaterialTheme.typography.headlineSmall, color = Palette.Muted)
        }
    } else {
        LoopingVideo(resId, modifier)
    }
}

@OptIn(UnstableApi::class)
@Composable
private fun LoopingVideo(resId: Int, modifier: Modifier) {
    val context = LocalContext.current
    val player = remember {
        ExoPlayer.Builder(context).build().apply {
            repeatMode = Player.REPEAT_MODE_ONE
            volume = 0f
            playWhenReady = true
        }
    }
    DisposableEffect(player) { onDispose { player.release() } }
    LaunchedEffect(resId) {
        player.setMediaItem(MediaItem.fromUri(RawResourceDataSource.buildRawResourceUri(resId)))
        player.prepare()
    }
    AndroidView(
        factory = {
            PlayerView(it).apply {
                useController = false
                resizeMode = AspectRatioFrameLayout.RESIZE_MODE_ZOOM
                this.player = player
            }
        },
        modifier = modifier,
    )
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/session/TimerDial.kt`:
```kotlin
package com.anfixit.anfimorning.ui.session

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.anfixit.anfimorning.session.Phase
import com.anfixit.anfimorning.ui.Palette
import com.anfixit.anfimorning.ui.formatMs

@Composable
fun TimerDial(ui: SessionUi, color: Color, modifier: Modifier = Modifier) {
    val progress = 1f - ui.remainingMs.toFloat() / ui.phaseTotalMs
    val label = when {
        ui.paused -> "Пауза"
        ui.phase == Phase.PREP -> "Подготовка"
        else -> "Работа"
    }
    Box(modifier, contentAlignment = Alignment.Center) {
        Canvas(Modifier.fillMaxSize()) {
            val stroke = Stroke(width = 10.dp.toPx())
            drawArc(Palette.Panel2, 0f, 360f, useCenter = false, style = stroke)
            drawArc(
                if (ui.phase == Phase.PREP) Palette.Muted else color,
                -90f, 360f * progress.coerceIn(0f, 1f), useCenter = false, style = stroke,
            )
        }
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(formatMs(ui.remainingMs), style = MaterialTheme.typography.headlineMedium, fontWeight = FontWeight.Bold)
            Text(label, style = MaterialTheme.typography.labelMedium, color = Palette.Muted)
        }
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/session/DetailsSheet.kt`:
```kotlin
package com.anfixit.anfimorning.ui.session

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.ModalBottomSheet
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.anfixit.anfimorning.program.Exercise
import com.anfixit.anfimorning.ui.Palette

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DetailsSheet(exercise: Exercise, next: Exercise?, onDismiss: () -> Unit) {
    ModalBottomSheet(onDismissRequest = onDismiss, containerColor = Palette.Panel) {
        Column(
            Modifier.verticalScroll(rememberScrollState()).padding(horizontal = 20.dp, vertical = 8.dp),
            verticalArrangement = Arrangement.spacedBy(12.dp),
        ) {
            Text(exercise.name, style = MaterialTheme.typography.titleLarge, fontWeight = FontWeight.Bold)
            Text("Как делать", fontWeight = FontWeight.Bold)
            exercise.steps.forEachIndexed { i, s -> Text("${i + 1}. $s") }
            Text("Чего избегать", fontWeight = FontWeight.Bold)
            Text(exercise.avoid, color = Palette.Coral)
            Text("Если сложно", fontWeight = FontWeight.Bold)
            Text(exercise.easier, color = Palette.Muted)
            next?.let { Text("Дальше: ${it.name}", color = Palette.Muted) }
            Text("Занятие на паузе. Закрой шторку и нажми ▶.", color = Palette.Yellow)
            Text("")
        }
    }
}
```

- [ ] **Step 3: Экран занятия**

`app/src/main/java/com/anfixit/anfimorning/ui/session/SessionScreen.kt` (заменить целиком):
```kotlin
package com.anfixit.anfimorning.ui.session

import androidx.activity.compose.BackHandler
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.LinearProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberUpdatedState
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.session.Phase
import com.anfixit.anfimorning.ui.KeepScreenOn
import com.anfixit.anfimorning.ui.OnLifecycleEvent
import com.anfixit.anfimorning.ui.Palette
import com.anfixit.anfimorning.ui.color
import com.anfixit.anfimorning.ui.formatMs

@Composable
fun SessionScreen(
    container: AppContainer,
    programId: String,
    from: Int,
    onFinished: (Long) -> Unit,
    onStopped: () -> Unit,
) {
    val vm: SessionViewModel = viewModel(
        key = "session-$programId-$from",
        factory = viewModelFactory { initializer { SessionViewModel(container, programId, from) } },
    )
    val ui by vm.ui.collectAsStateWithLifecycle()
    val finishedCallback by rememberUpdatedState(onFinished)
    var showDetails by remember { mutableStateOf(false) }
    var confirmStop by remember { mutableStateOf(false) }

    KeepScreenOn()
    OnLifecycleEvent(Lifecycle.Event.ON_STOP) { vm.pause() }
    LaunchedEffect(vm) { vm.finished.collect { finishedCallback(it) } }
    BackHandler { confirmStop = true }

    val exercise = ui.exercise ?: return
    val stageColor = exercise.stage.color()

    Column(
        Modifier.fillMaxSize().padding(horizontal = 16.dp, vertical = 8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
    ) {
        LinearProgressIndicator(
            progress = { (1f - ui.totalRemainingMs.toFloat() / ui.totalMs).coerceIn(0f, 1f) },
            modifier = Modifier.fillMaxWidth(),
            color = stageColor,
            trackColor = Palette.Panel2,
        )
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text(
                "${ui.stepNumber} из ${ui.stepCount} · осталось ${formatMs(ui.totalRemainingMs)}",
                color = Palette.Muted,
                modifier = Modifier.weight(1f),
            )
            TextButton(onClick = { confirmStop = true }) { Text("Завершить") }
        }
        if (ui.voiceMissing) {
            Text(
                "Русский голос не найден — подсказки только на экране. Поставь его в «Настройках».",
                color = Palette.Yellow,
                style = MaterialTheme.typography.bodySmall,
            )
        }
        ExerciseVideo(
            exercise,
            Modifier
                .fillMaxWidth()
                .weight(1f)
                .clip(RoundedCornerShape(20.dp))
                .clickable(enabled = ui.phase == Phase.PREP) { vm.extendPrep() },
        )
        Text(exercise.name, style = MaterialTheme.typography.headlineSmall, fontWeight = FontWeight.Bold)
        Text("${exercise.pose.label} · ${exercise.stage.label}", color = stageColor)
        Row(verticalAlignment = Alignment.CenterVertically) {
            TimerDial(ui, stageColor, Modifier.size(120.dp))
            Column(Modifier.padding(start = 16.dp).weight(1f), verticalArrangement = Arrangement.spacedBy(4.dp)) {
                exercise.short.forEach { Text("• $it", style = MaterialTheme.typography.bodyMedium) }
                TextButton(onClick = { vm.pause(); showDetails = true }) { Text("Подробнее") }
            }
        }
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp), verticalAlignment = Alignment.CenterVertically) {
            OutlinedButton(onClick = vm::prev, modifier = Modifier.height(56.dp)) { Text("←") }
            Button(onClick = vm::togglePause, modifier = Modifier.weight(1f).height(56.dp)) {
                Text(if (ui.paused) "▶ Продолжить" else "Ⅱ Пауза", style = MaterialTheme.typography.titleMedium)
            }
            if (ui.phase == Phase.PREP) {
                OutlinedButton(onClick = vm::extendPrep, modifier = Modifier.height(56.dp)) { Text("+10") }
            }
            OutlinedButton(onClick = vm::next, modifier = Modifier.height(56.dp)) { Text("→") }
        }
        Spacer(Modifier.height(4.dp))
    }

    if (showDetails) DetailsSheet(exercise, ui.next, onDismiss = { showDetails = false })
    if (confirmStop) {
        AlertDialog(
            onDismissRequest = { confirmStop = false },
            title = { Text("Завершить занятие?") },
            text = { Text("Сделанные упражнения сохранятся в календаре.") },
            confirmButton = { TextButton(onClick = { confirmStop = false; vm.stop(onStopped) }) { Text("Завершить") } },
            dismissButton = { TextButton(onClick = { confirmStop = false }) { Text("Продолжить") } },
        )
    }
}
```

- [ ] **Step 4: Сборка**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 5: Проверка на телефоне**

автор включает отладку по USB (Настройки → О телефоне → 7 раз по «Номер сборки» → Для разработчиков → Отладка по USB) и подключает телефон. Затем:

```bash
adb devices
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Проверить вручную: «Короткая» стартует, голос говорит подготовку, через 10 с «Начали», круг идёт, на 3-2-1 сигналы; «+10» и тап по заглушке видео добавляют 10 с; пауза/продолжить; ←/→; «Подробнее» ставит паузу; экран не гаснет; кнопка питания → после включения занятие на паузе; «Завершить» → подтверждение → главная.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: guided session screen with video, timer and controls

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Экран «Готово» и самочувствие

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/done/DoneViewModel.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/ui/done/DoneScreen.kt` (заменить заглушку)

**Interfaces:**
- Consumes: `HistoryRepository.get/setFeeling`, `Feeling`, `SessionRecord` (Task 4); `Feeling.label()`, `Feeling?.color()`, `Palette` (Task 6).
- Produces: `DoneScreen(container, recordId, onClose)`.

- [ ] **Step 1: ViewModel и экран**

`app/src/main/java/com/anfixit/anfimorning/ui/done/DoneViewModel.kt`:
```kotlin
package com.anfixit.anfimorning.ui.done

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.anfixit.anfimorning.history.Feeling
import com.anfixit.anfimorning.history.HistoryRepository
import com.anfixit.anfimorning.history.SessionRecord
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

class DoneViewModel(private val history: HistoryRepository, private val recordId: Long) : ViewModel() {
    private val _record = MutableStateFlow<SessionRecord?>(null)
    val record: StateFlow<SessionRecord?> = _record

    init {
        viewModelScope.launch { _record.value = history.get(recordId) }
    }

    fun save(feeling: Feeling?, note: String, onSaved: () -> Unit) {
        viewModelScope.launch {
            history.setFeeling(recordId, feeling, note)
            onSaved()
        }
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/done/DoneScreen.kt` (заменить целиком):
```kotlin
package com.anfixit.anfimorning.ui.done

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.Button
import androidx.compose.material3.FilterChip
import androidx.compose.material3.FilterChipDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedTextField
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.saveable.rememberSaveable
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.history.Feeling
import com.anfixit.anfimorning.ui.Palette
import com.anfixit.anfimorning.ui.color
import com.anfixit.anfimorning.ui.label

@Composable
fun DoneScreen(container: AppContainer, recordId: Long, onClose: () -> Unit) {
    val vm: DoneViewModel = viewModel(
        key = "done-$recordId",
        factory = viewModelFactory { initializer { DoneViewModel(container.history, recordId) } },
    )
    val record by vm.record.collectAsStateWithLifecycle()
    var feeling by rememberSaveable { mutableStateOf<Feeling?>(null) }
    var note by rememberSaveable { mutableStateOf("") }

    Column(
        Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.spacedBy(16.dp),
    ) {
        Text("✦", style = MaterialTheme.typography.displayMedium, color = Palette.Accent)
        Text("Готово!", style = MaterialTheme.typography.headlineLarge, fontWeight = FontWeight.Bold)
        record?.let { Text("Ты занималась ${(it.actualSec + 30) / 60} мин", color = Palette.Muted) }
        Text("Как самочувствие?", style = MaterialTheme.typography.titleMedium)
        Feeling.entries.forEach { f ->
            FilterChip(
                selected = feeling == f,
                onClick = { feeling = if (feeling == f) null else f },
                label = { Text(f.label(), style = MaterialTheme.typography.titleMedium) },
                modifier = Modifier.fillMaxWidth().height(52.dp),
                colors = FilterChipDefaults.filterChipColors(selectedContainerColor = f.color().copy(alpha = 0.35f)),
            )
        }
        OutlinedTextField(
            value = note,
            onValueChange = { note = it },
            label = { Text("Заметка (например: тянуло в пояснице на мостике)") },
            modifier = Modifier.fillMaxWidth(),
            minLines = 2,
        )
        Button(onClick = { vm.save(feeling, note, onClose) }, modifier = Modifier.fillMaxWidth().height(56.dp)) {
            Text("Сохранить")
        }
        TextButton(onClick = onClose) { Text("Пропустить") }
    }
}
```

- [ ] **Step 2: Сборка**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 3: Проверка на телефоне**

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

Начать «Короткую», дождаться конца первого упражнения (чтобы в истории было хотя бы одно законченное), остальные пролистать «→». После «Готово» выбрать самочувствие, написать заметку, «Сохранить» → главная.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: done screen with feeling and note

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Календарь

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarLogic.kt`, `CalendarViewModel.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarScreen.kt` (заменить заглушку)
- Test: `app/src/test/java/com/anfixit/anfimorning/ui/calendar/CalendarLogicTest.kt`

**Interfaces:**
- Consumes: `SessionRecord`, `Feeling`, `localDate` (Task 4); `Palette`, `color()`, `label()` (Task 6).
- Produces: `data class DayMark(val completed: Boolean, val feeling: Feeling?)`; `object CalendarLogic { fun marks(records: List<SessionRecord>, zone: ZoneId): Map<LocalDate, DayMark>; fun monthGrid(month: YearMonth): List<LocalDate?> }` (сетка с понедельника, `null` — пустые клетки до 1-го числа).

- [ ] **Step 1: Падающий тест**

`app/src/test/java/com/anfixit/anfimorning/ui/calendar/CalendarLogicTest.kt`:
```kotlin
package com.anfixit.anfimorning.ui.calendar

import com.anfixit.anfimorning.history.Feeling
import com.anfixit.anfimorning.history.SessionRecord
import org.junit.Assert.assertEquals
import org.junit.Test
import java.time.LocalDate
import java.time.YearMonth
import java.time.ZoneId
import java.time.ZonedDateTime

class CalendarLogicTest {
    private val zone = ZoneId.of("Europe/Samara")

    private fun record(iso: String, completed: Boolean, feeling: Feeling? = null) = SessionRecord(
        startedAt = ZonedDateTime.parse(iso).toInstant().toEpochMilli(),
        programId = "full", plannedSec = 1400, actualSec = 1400,
        exercisesDone = 18, exercisesTotal = 18, completed = completed, feeling = feeling,
    )

    @Test
    fun `november 2026 starts on sunday`() {
        val grid = CalendarLogic.monthGrid(YearMonth.of(2026, 11))
        assertEquals(36, grid.size)
        assertEquals(List(6) { null }, grid.take(6))
        assertEquals(LocalDate.of(2026, 11, 1), grid[6])
        assertEquals(LocalDate.of(2026, 11, 30), grid.last())
    }

    @Test
    fun `marks use latest completed session and show incomplete days`() {
        val marks = CalendarLogic.marks(
            listOf(
                record("2026-11-08T07:00+04:00[Europe/Samara]", completed = true, feeling = Feeling.OK),
                record("2026-11-08T19:00+04:00[Europe/Samara]", completed = true, feeling = Feeling.GOOD),
                record("2026-11-08T21:00+04:00[Europe/Samara]", completed = false),
                record("2026-11-07T07:00+04:00[Europe/Samara]", completed = false),
            ),
            zone,
        )
        assertEquals(DayMark(completed = true, feeling = Feeling.GOOD), marks[LocalDate.of(2026, 11, 8)])
        assertEquals(DayMark(completed = false, feeling = null), marks[LocalDate.of(2026, 11, 7)])
    }
}
```

- [ ] **Step 2: Убедиться, что падает**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.ui.calendar.*'`
Expected: FAIL — `Unresolved reference: CalendarLogic`.

- [ ] **Step 3: Логика**

`app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarLogic.kt`:
```kotlin
package com.anfixit.anfimorning.ui.calendar

import com.anfixit.anfimorning.history.Feeling
import com.anfixit.anfimorning.history.SessionRecord
import com.anfixit.anfimorning.history.localDate
import java.time.LocalDate
import java.time.YearMonth
import java.time.ZoneId

data class DayMark(val completed: Boolean, val feeling: Feeling?)

object CalendarLogic {
    fun marks(records: List<SessionRecord>, zone: ZoneId): Map<LocalDate, DayMark> =
        records.groupBy { it.localDate(zone) }.mapValues { (_, day) ->
            val latestCompleted = day.filter { it.completed }.maxByOrNull { it.startedAt }
            if (latestCompleted == null) DayMark(false, null) else DayMark(true, latestCompleted.feeling)
        }

    fun monthGrid(month: YearMonth): List<LocalDate?> {
        val lead = month.atDay(1).dayOfWeek.value - 1
        return List(lead) { null } + (1..month.lengthOfMonth()).map { month.atDay(it) }
    }
}
```

- [ ] **Step 4: Тест проходит**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.ui.calendar.*'`
Expected: PASS, 2 теста.

- [ ] **Step 5: ViewModel и экран**

`app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarViewModel.kt`:
```kotlin
package com.anfixit.anfimorning.ui.calendar

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.history.SessionRecord
import com.anfixit.anfimorning.history.localDate
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.flow.stateIn
import java.time.LocalDate
import java.time.YearMonth
import java.time.ZoneId

class CalendarViewModel(container: AppContainer) : ViewModel() {
    private val zone = ZoneId.systemDefault()
    private val records = container.history.observeAll()
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())

    val month = MutableStateFlow(YearMonth.now(zone))
    val selected = MutableStateFlow<LocalDate?>(LocalDate.now(zone))
    val titles: Map<String, String> =
        container.program.getOrNull()?.programs?.associate { it.id to it.title }.orEmpty()

    val marks: StateFlow<Map<LocalDate, DayMark>> = records
        .map { CalendarLogic.marks(it, zone) }
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyMap())

    val selectedRecords: StateFlow<List<SessionRecord>> = combine(records, selected) { list, day ->
        if (day == null) emptyList() else list.filter { it.localDate(zone) == day }.sortedBy { it.startedAt }
    }.stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())

    fun shiftMonth(by: Long) {
        month.value = month.value.plusMonths(by)
        selected.value = null
    }

    fun select(day: LocalDate) {
        selected.value = day
    }

    fun timeOf(record: SessionRecord): String =
        java.time.Instant.ofEpochMilli(record.startedAt).atZone(zone).toLocalTime().toString().take(5)
}
```

`app/src/main/java/com/anfixit/anfimorning/ui/calendar/CalendarScreen.kt` (заменить целиком):
```kotlin
package com.anfixit.anfimorning.ui.calendar

import androidx.compose.foundation.BorderStroke
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.aspectRatio
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.Card
import androidx.compose.material3.CardDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.ui.Palette
import com.anfixit.anfimorning.ui.color
import com.anfixit.anfimorning.ui.label
import java.time.format.TextStyle
import java.util.Locale

private val ru = Locale.forLanguageTag("ru")
private val weekdays = listOf("Пн", "Вт", "Ср", "Чт", "Пт", "Сб", "Вс")

@Composable
fun CalendarScreen(container: AppContainer) {
    val vm: CalendarViewModel = viewModel(factory = viewModelFactory { initializer { CalendarViewModel(container) } })
    val month by vm.month.collectAsStateWithLifecycle()
    val marks by vm.marks.collectAsStateWithLifecycle()
    val selected by vm.selected.collectAsStateWithLifecycle()
    val dayRecords by vm.selectedRecords.collectAsStateWithLifecycle()

    Column(
        Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp),
    ) {
        Row(verticalAlignment = Alignment.CenterVertically) {
            TextButton(onClick = { vm.shiftMonth(-1) }) { Text("◀") }
            Text(
                month.month.getDisplayName(TextStyle.FULL_STANDALONE, ru).replaceFirstChar { it.uppercase() } +
                    " ${month.year}",
                style = MaterialTheme.typography.titleLarge,
                textAlign = TextAlign.Center,
                modifier = Modifier.weight(1f),
            )
            TextButton(onClick = { vm.shiftMonth(1) }) { Text("▶") }
        }
        Row(Modifier.fillMaxWidth()) {
            weekdays.forEach {
                Text(it, color = Palette.Muted, textAlign = TextAlign.Center, modifier = Modifier.weight(1f))
            }
        }
        CalendarLogic.monthGrid(month).chunked(7).forEach { week ->
            Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(4.dp)) {
                (0 until 7).forEach { i ->
                    val day = week.getOrNull(i)
                    Box(Modifier.weight(1f).aspectRatio(1f), contentAlignment = Alignment.Center) {
                        if (day != null) {
                            val mark = marks[day]
                            val fill = if (mark?.completed == true) mark.feeling.color() else Color.Transparent
                            val border = when {
                                day == selected -> BorderStroke(2.dp, Palette.Ink)
                                mark != null && !mark.completed -> BorderStroke(2.dp, Palette.Muted)
                                else -> null
                            }
                            Box(
                                Modifier
                                    .fillMaxSize()
                                    .padding(2.dp)
                                    .background(fill, CircleShape)
                                    .then(if (border != null) Modifier.border(border, CircleShape) else Modifier)
                                    .clickable { vm.select(day) },
                                contentAlignment = Alignment.Center,
                            ) {
                                Text(
                                    day.dayOfMonth.toString(),
                                    color = if (mark?.completed == true) Palette.OnAccent else Palette.Ink,
                                )
                            }
                        }
                    }
                }
            }
        }
        selected?.let { day ->
            Text(
                "${day.dayOfMonth} ${day.month.getDisplayName(TextStyle.FULL, ru)}",
                style = MaterialTheme.typography.titleMedium,
                fontWeight = FontWeight.Bold,
            )
            if (dayRecords.isEmpty()) Text("Занятий нет", color = Palette.Muted)
            dayRecords.forEach { r ->
                Card(colors = CardDefaults.cardColors(containerColor = Palette.Panel)) {
                    Column(Modifier.padding(12.dp), verticalArrangement = Arrangement.spacedBy(4.dp)) {
                        Text(
                            "${vm.titles[r.programId] ?: r.programId} · ${vm.timeOf(r)} · ${(r.actualSec + 30) / 60} мин",
                            fontWeight = FontWeight.Bold,
                        )
                        Text(
                            (if (r.completed) "Завершено" else "Не закончено") +
                                " · ${r.exercisesDone} из ${r.exercisesTotal} упражнений",
                            color = Palette.Muted,
                        )
                        r.feeling?.let { Text(it.label()) }
                        r.note?.let { Text("«$it»", color = Palette.Muted) }
                    }
                }
            }
        }
    }
}
```

- [ ] **Step 6: Сборка и проверка на телефоне**

Run: `./gradlew testDebugUnitTest assembleDebug && adb install -r app/build/outputs/apk/debug/app-debug.apk`
Expected: `BUILD SUCCESSFUL`; во вкладке «Календарь» сегодняшний день закрашен цветом самочувствия из задачи 9, тап показывает карточку занятия; ◀ ▶ листают месяцы.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: calendar with feeling colors and day details

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Напоминание

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/reminder/Notifications.kt`, `ReminderScheduler.kt`, `ReminderReceiver.kt`, `RescheduleReceiver.kt`
- Modify: `app/src/main/AndroidManifest.xml` (разрешения и приёмники)
- Modify: `app/src/main/java/com/anfixit/anfimorning/AppContainer.kt` (добавить `reminderScheduler`)
- Modify: `app/src/main/java/com/anfixit/anfimorning/AnfimorningApp.kt` (канал уведомлений)

**Interfaces:**
- Consumes: `NextReminderCalculator`, `ReminderSettings`, `SettingsRepository.settings` (Task 5); `HistoryRepository.completedOn` (Task 4).
- Produces: `class ReminderScheduler(context: Context)`: `fun canScheduleExact(): Boolean`, `fun apply(settings: ReminderSettings)`; `AppContainer.reminderScheduler`; `object Notifications { fun createChannel(context: Context); fun showReminder(context: Context) }`.

- [ ] **Step 1: Уведомления и планировщик**

`app/src/main/java/com/anfixit/anfimorning/reminder/Notifications.kt`:
```kotlin
package com.anfixit.anfimorning.reminder

import android.annotation.SuppressLint
import android.app.NotificationChannel
import android.app.NotificationManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import androidx.core.app.NotificationCompat
import androidx.core.app.NotificationManagerCompat
import com.anfixit.anfimorning.MainActivity
import com.anfixit.anfimorning.R

object Notifications {
    private const val CHANNEL = "reminders"
    private const val REMINDER_ID = 1

    fun createChannel(context: Context) {
        context.getSystemService(NotificationManager::class.java).createNotificationChannel(
            NotificationChannel(CHANNEL, "Напоминания о зарядке", NotificationManager.IMPORTANCE_DEFAULT),
        )
    }

    @SuppressLint("MissingPermission")
    fun showReminder(context: Context) {
        val manager = NotificationManagerCompat.from(context)
        if (!manager.areNotificationsEnabled()) return
        val open = PendingIntent.getActivity(
            context, 0,
            Intent(context, MainActivity::class.java)
                .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP),
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
        )
        val notification = NotificationCompat.Builder(context, CHANNEL)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle("Пора на зарядку")
            .setContentText("Пара десятков минут для спины и бодрого утра")
            .setContentIntent(open)
            .setAutoCancel(true)
            .build()
        manager.notify(REMINDER_ID, notification)
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/reminder/ReminderScheduler.kt`:
```kotlin
package com.anfixit.anfimorning.reminder

import android.app.AlarmManager
import android.app.PendingIntent
import android.content.Context
import android.content.Intent
import android.os.Build
import com.anfixit.anfimorning.settings.NextReminderCalculator
import com.anfixit.anfimorning.settings.ReminderSettings
import java.time.ZonedDateTime

class ReminderScheduler(private val context: Context) {
    private val alarms = context.getSystemService(AlarmManager::class.java)

    fun canScheduleExact(): Boolean = Build.VERSION.SDK_INT < Build.VERSION_CODES.S || alarms.canScheduleExactAlarms()

    fun apply(settings: ReminderSettings) {
        val pending = pendingIntent()
        alarms.cancel(pending)
        if (!settings.enabled) return
        val next = NextReminderCalculator.next(ZonedDateTime.now(), settings.hour, settings.minute, settings.days)
            ?: return
        val atMs = next.toInstant().toEpochMilli()
        if (canScheduleExact()) {
            alarms.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, atMs, pending)
        } else {
            alarms.setWindow(AlarmManager.RTC_WAKEUP, atMs, WINDOW_MS, pending)
        }
    }

    private fun pendingIntent(): PendingIntent = PendingIntent.getBroadcast(
        context, 1,
        Intent(context, ReminderReceiver::class.java).setAction(ACTION_FIRE),
        PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT,
    )

    companion object {
        const val ACTION_FIRE = "com.anfixit.anfimorning.action.REMINDER"
        private const val WINDOW_MS = 10 * 60_000L
    }
}
```

- [ ] **Step 2: Приёмники**

`app/src/main/java/com/anfixit/anfimorning/reminder/ReminderReceiver.kt`:
```kotlin
package com.anfixit.anfimorning.reminder

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.anfixit.anfimorning.AnfimorningApp
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch
import java.time.LocalDate

class ReminderReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != ReminderScheduler.ACTION_FIRE) return
        val container = (context.applicationContext as AnfimorningApp).container
        val pending = goAsync()
        CoroutineScope(Dispatchers.IO).launch {
            try {
                if (!container.history.completedOn(LocalDate.now())) Notifications.showReminder(context)
                container.reminderScheduler.apply(container.settings.settings.first().reminder)
            } finally {
                pending.finish()
            }
        }
    }
}
```

`app/src/main/java/com/anfixit/anfimorning/reminder/RescheduleReceiver.kt`:
```kotlin
package com.anfixit.anfimorning.reminder

import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import com.anfixit.anfimorning.AnfimorningApp
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.first
import kotlinx.coroutines.launch

/** Перепланирует напоминание после перезагрузки, смены времени и разрешения на точные будильники. */
class RescheduleReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action !in HANDLED) return
        val container = (context.applicationContext as AnfimorningApp).container
        val pending = goAsync()
        CoroutineScope(Dispatchers.IO).launch {
            try {
                container.reminderScheduler.apply(container.settings.settings.first().reminder)
            } finally {
                pending.finish()
            }
        }
    }

    private companion object {
        val HANDLED = setOf(
            Intent.ACTION_BOOT_COMPLETED,
            Intent.ACTION_TIME_CHANGED,
            Intent.ACTION_TIMEZONE_CHANGED,
            "android.app.action.SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED",
        )
    }
}
```

- [ ] **Step 3: Манифест, контейнер, канал**

В `AndroidManifest.xml` перед `<application` добавить:
```xml
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
```

Внутри `<application>` после `</activity>` добавить:
```xml
        <receiver
            android:name=".reminder.ReminderReceiver"
            android:exported="false" />

        <receiver
            android:name=".reminder.RescheduleReceiver"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.TIME_SET" />
                <action android:name="android.intent.action.TIMEZONE_CHANGED" />
                <action android:name="android.app.action.SCHEDULE_EXACT_ALARM_PERMISSION_STATE_CHANGED" />
            </intent-filter>
        </receiver>
```

В `AppContainer.kt` добавить импорт `com.anfixit.anfimorning.reminder.ReminderScheduler` и в конец класса:
```kotlin
    val reminderScheduler by lazy { ReminderScheduler(app) }
```

В `AnfimorningApp.kt` добавить импорт `com.anfixit.anfimorning.reminder.Notifications` и в `onCreate()` после создания контейнера:
```kotlin
        Notifications.createChannel(this)
```

- [ ] **Step 4: Сборка**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: morning reminder with exact alarms and reboot handling

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Ручная проверка напоминания — в задаче 12, когда появится экран настроек.)

---

### Task 12: Экран настроек

**Files:**
- Create: `app/src/main/java/com/anfixit/anfimorning/ui/settings/SettingsViewModel.kt`
- Modify: `app/src/main/java/com/anfixit/anfimorning/ui/settings/SettingsScreen.kt` (заменить заглушку)

**Interfaces:**
- Consumes: `SettingsRepository` (Task 5), `ReminderScheduler` (Task 11), `VoiceGuide.ready` (Task 7), `ProgramFile.exercises[].videoSource` (Task 2), `OnLifecycleEvent`, `Palette` (Task 6).
- Produces: `SettingsScreen(container)`.

- [ ] **Step 1: ViewModel**

`app/src/main/java/com/anfixit/anfimorning/ui/settings/SettingsViewModel.kt`:
```kotlin
package com.anfixit.anfimorning.ui.settings

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.settings.AppSettings
import com.anfixit.anfimorning.settings.ReminderSettings
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class SettingsViewModel(private val container: AppContainer) : ViewModel() {
    val settings: StateFlow<AppSettings> =
        container.settings.settings.stateIn(viewModelScope, SharingStarted.Eagerly, AppSettings())
    val voiceReady: StateFlow<Boolean?> = container.voice.ready
    val credits: List<String> = container.program.getOrNull()?.exercises
        ?.mapNotNull { ex -> ex.videoSource?.let { "${ex.name} — ${it.author}, ${it.license}\n${it.url}" } }
        .orEmpty()

    fun canScheduleExact(): Boolean = container.reminderScheduler.canScheduleExact()

    fun setVoice(value: Boolean) = viewModelScope.launch { container.settings.setVoice(value) }
    fun setBeeps(value: Boolean) = viewModelScope.launch { container.settings.setBeeps(value) }
    fun setSpeechRate(value: Float) = viewModelScope.launch { container.settings.setSpeechRate(value) }

    fun setReminder(value: ReminderSettings) = viewModelScope.launch {
        container.settings.setReminder(value)
        container.reminderScheduler.apply(value)
    }
}
```

- [ ] **Step 2: Экран**

`app/src/main/java/com/anfixit/anfimorning/ui/settings/SettingsScreen.kt` (заменить целиком):
```kotlin
package com.anfixit.anfimorning.ui.settings

import android.Manifest
import android.app.TimePickerDialog
import android.content.Intent
import android.net.Uri
import android.os.Build
import android.provider.Settings
import android.speech.tts.TextToSpeech
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Slider
import androidx.compose.material3.Switch
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableIntStateOf
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.core.app.NotificationManagerCompat
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import androidx.lifecycle.viewmodel.initializer
import androidx.lifecycle.viewmodel.viewModelFactory
import com.anfixit.anfimorning.AppContainer
import com.anfixit.anfimorning.ui.OnLifecycleEvent
import com.anfixit.anfimorning.ui.Palette
import java.time.DayOfWeek
import java.util.Locale

private val dayLabels = listOf(
    DayOfWeek.MONDAY to "Пн", DayOfWeek.TUESDAY to "Вт", DayOfWeek.WEDNESDAY to "Ср",
    DayOfWeek.THURSDAY to "Чт", DayOfWeek.FRIDAY to "Пт", DayOfWeek.SATURDAY to "Сб", DayOfWeek.SUNDAY to "Вс",
)

@Composable
fun SettingsScreen(container: AppContainer) {
    val vm: SettingsViewModel = viewModel(factory = viewModelFactory { initializer { SettingsViewModel(container) } })
    val s by vm.settings.collectAsStateWithLifecycle()
    val voiceReady by vm.voiceReady.collectAsStateWithLifecycle()
    val context = LocalContext.current
    var resumeTick by remember { mutableIntStateOf(0) }
    var showCredits by remember { mutableStateOf(false) }
    OnLifecycleEvent(Lifecycle.Event.ON_RESUME) { resumeTick++ }
    val notificationsOn = remember(resumeTick) { NotificationManagerCompat.from(context).areNotificationsEnabled() }
    val exactOk = remember(resumeTick) { vm.canScheduleExact() }
    val askNotifications = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) {
        resumeTick++
    }
    val r = s.reminder

    Column(
        Modifier.fillMaxSize().verticalScroll(rememberScrollState()).padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp),
    ) {
        Text("Напоминание", style = MaterialTheme.typography.titleLarge, fontWeight = FontWeight.Bold)
        SwitchRow("Напоминать о зарядке", r.enabled) { on ->
            vm.setReminder(r.copy(enabled = on))
            if (on && Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU && !notificationsOn) {
                askNotifications.launch(Manifest.permission.POST_NOTIFICATIONS)
            }
        }
        Row(verticalAlignment = Alignment.CenterVertically) {
            Text("Время", modifier = Modifier.weight(1f))
            TextButton(onClick = {
                TimePickerDialog(context, { _, h, m -> vm.setReminder(r.copy(hour = h, minute = m)) }, r.hour, r.minute, true)
                    .show()
            }) {
                Text(String.format(Locale.ROOT, "%02d:%02d", r.hour, r.minute), style = MaterialTheme.typography.titleLarge)
            }
        }
        Row(Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.SpaceBetween) {
            dayLabels.forEach { (day, label) ->
                val on = day in r.days
                Box(
                    Modifier
                        .size(40.dp)
                        .background(if (on) Palette.Accent else Palette.Panel2, CircleShape)
                        .clickable { vm.setReminder(r.copy(days = if (on) r.days - day else r.days + day)) },
                    contentAlignment = Alignment.Center,
                ) {
                    Text(label, color = if (on) Palette.OnAccent else Palette.Ink)
                }
            }
        }
        if (r.enabled && !notificationsOn) {
            Text("Уведомления выключены в системе — напоминание не появится.", color = Palette.Coral)
            TextButton(onClick = {
                context.startActivity(
                    Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).putExtra(Settings.EXTRA_APP_PACKAGE, context.packageName),
                )
            }) { Text("Открыть настройки уведомлений") }
        }
        if (r.enabled && !exactOk && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            Text("Без разрешения на точные будильники напоминание может прийти с опозданием до 10 минут.", color = Palette.Yellow)
            TextButton(onClick = {
                context.startActivity(
                    Intent(Settings.ACTION_REQUEST_SCHEDULE_EXACT_ALARM, Uri.parse("package:${context.packageName}")),
                )
            }) { Text("Разрешить точное время") }
        }

        HorizontalDivider(color = Palette.Panel2)
        Text("Голос и сигналы", style = MaterialTheme.typography.titleLarge, fontWeight = FontWeight.Bold)
        SwitchRow("Голосовые подсказки", s.voice) { vm.setVoice(it) }
        Text("Скорость речи: ${String.format(Locale.ROOT, "%.1f", s.speechRate)}")
        Slider(
            value = s.speechRate,
            onValueChange = { vm.setSpeechRate((it * 10).toInt() / 10f) },
            valueRange = 0.8f..1.2f,
            steps = 3,
        )
        if (voiceReady == false) {
            Text("Русский голос не найден. Поставь «Синтезатор речи Google» и русский голос.", color = Palette.Yellow)
            TextButton(onClick = {
                runCatching { context.startActivity(Intent(TextToSpeech.Engine.ACTION_INSTALL_TTS_DATA)) }
            }) { Text("Установить голос") }
        }
        SwitchRow("Сигналы за 3 секунды до конца", s.beeps) { vm.setBeeps(it) }

        HorizontalDivider(color = Palette.Panel2)
        TextButton(onClick = { showCredits = true }) { Text("Источники видео") }
    }

    if (showCredits) {
        AlertDialog(
            onDismissRequest = { showCredits = false },
            title = { Text("Источники видео") },
            text = {
                Column(Modifier.verticalScroll(rememberScrollState()), verticalArrangement = Arrangement.spacedBy(8.dp)) {
                    if (vm.credits.isEmpty()) Text("Видео пока не добавлены.")
                    vm.credits.forEach { Text(it, style = MaterialTheme.typography.bodySmall) }
                }
            },
            confirmButton = { TextButton(onClick = { showCredits = false }) { Text("Закрыть") } },
        )
    }
}

@Composable
private fun SwitchRow(label: String, checked: Boolean, onChange: (Boolean) -> Unit) {
    Row(verticalAlignment = Alignment.CenterVertically) {
        Text(label, modifier = Modifier.weight(1f))
        Switch(checked = checked, onCheckedChange = onChange)
    }
}
```

- [ ] **Step 3: Сборка**

Run: `./gradlew testDebugUnitTest assembleDebug`
Expected: `BUILD SUCCESSFUL`.

- [ ] **Step 4: Проверка напоминания на телефоне**

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

1. «Настройки» → включить напоминание → разрешить уведомления. Время поставить на 2 минуты вперёд, сегодняшний день включён.
2. Если появилась строка про точное время — нажать «Разрешить точное время», включить.
3. Проверить, что будильник стоит: `adb shell dumpsys alarm | grep -A3 anfimorning` — есть запись с `REMINDER`.
4. Дождаться уведомления «Пора на зарядку», тап открывает приложение.
5. Перезагрузить телефон (`adb reboot`), после загрузки снова `adb shell dumpsys alarm | grep -A3 anfimorning` — запись на следующий день есть.
6. Скорость речи и выключение голоса/сигналов влияют на занятие.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: settings screen for reminder, voice and beeps

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Видео упражнений

**Files:**
- Create: `app/src/main/res/raw/<exercise_id>.mp4` (по одному на упражнение, имя = `id` из program.json)
- Modify: `app/src/main/assets/program.json` (поля `video`, `videoSource`)
- Create: `CREDITS.md`

**Interfaces:**
- Consumes: `Exercise.video` — имя ресурса без расширения (`cat_cow` → `res/raw/cat_cow.mp4`); проверка `ProgramFileTest` (Task 2).

- [ ] **Step 1: Подобрать кандидатов**

Для каждого из 18 упражнений найти на pexels.com/videos и pixabay.com/videos (встроенный браузер, поиск по-английски: «neck rotation exercise», «cat cow stretch», «bird dog exercise», «glute bridge», «dead bug exercise», «side lying leg raise», «prone Y raise», «wall slide shoulder», «doorway chest stretch», «calf stretch wall», «hamstring stretch towel», «half kneeling hip flexor stretch», «single leg balance», «mini squat chair», «pelvic tilt standing», «shoulder circles», «wrist circles», «breathing exercise standing»). Для каждого записать: ссылку на страницу ролика, автора, лицензию, секунды нужного фрагмента. Лицензию проверять на странице ролика: подходит Pexels License и Pixabay Content License.

- [ ] **Step 2: СТОП — согласовать список с автором**

Показать в чате таблицу «упражнение → ссылка → автор → фрагмент → примерный размер файла» и отдельно список упражнений без подходящего ролика. Скачивать только после её явного «да». Для упражнений без ролика — её решение: похожий ролик, остаться без видео (заглушка с позой) или снять самой.

- [ ] **Step 3: Скачать в отдельную пустую папку**

```bash
RAW_DL="$(mktemp -d)/videos" && mkdir -p "$RAW_DL" && echo "$RAW_DL"
```

Скачать одобренные файлы в `$RAW_DL` (кнопка «Download» на странице ролика в браузере или `curl -L -o "$RAW_DL/<id>.src.mp4" "<прямая ссылка на файл>"`).

- [ ] **Step 4: Перекодировать каждый ролик**

Для каждого упражнения (`ID`, `START` — секунда начала, `DUR` — 5–10 с):
```bash
ffmpeg -y -ss "$START" -t "$DUR" -i "$RAW_DL/$ID.src.mp4" -an \
  -vf "scale='if(gt(iw,ih),-2,480)':'if(gt(iw,ih),480,-2)',fps=30" \
  -c:v libx264 -profile:v main -pix_fmt yuv420p -preset slow -crf 28 -movflags +faststart \
  "app/src/main/res/raw/$ID.mp4"
ls -lh "app/src/main/res/raw/$ID.mp4"
```
Expected: файл 0,3–1,5 МБ. Если больше 1,5 МБ — повторить с `-crf 31`.

- [ ] **Step 5: Прописать видео в program.json и CREDITS.md**

Для каждого упражнения с роликом добавить в его объект:
```json
      "video": "cat_cow",
      "videoSource": {"url": "https://www.pexels.com/video/…", "author": "Имя автора", "license": "Pexels License"},
```
(значения — из одобренной таблицы шага 2).

`CREDITS.md`:
```markdown
# Источники видео

Короткие фрагменты роликов перекодированы (обрезка, 480p, без звука).

| Упражнение | Автор | Лицензия | Источник |
|---|---|---|---|
```
и по строке на каждый ролик из одобренной таблицы.

- [ ] **Step 6: Проверка**

Run: `./gradlew testDebugUnitTest --tests 'com.anfixit.anfimorning.program.*' assembleDebug && ls -lh app/build/outputs/apk/debug/app-debug.apk`
Expected: PASS (`program json is valid` — все файлы на месте, у всех есть источник); APK до ~35 МБ.

Установить на телефон, пройти «Короткую» до конца: видео крутятся по кругу без звука, заполняют блок, не тормозят; «Настройки» → «Источники видео» показывает список.

- [ ] **Step 7: Commit**

```bash
git add app/src/main/res/raw app/src/main/assets/program.json CREDITS.md
git commit -m "feat: looping exercise videos with credits

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Подпись, установка и приёмка

**Files:**
- Modify: `app/build.gradle.kts` (подпись release)
- Modify: `README.md` (раздел «Установка»)
- Modify (вне репозитория, не коммитить): `~/dev/ai-context/projects/anfimorning.md`

**Interfaces:**
- Produces: подписанный `app/build/outputs/apk/release/app-release.apk`, установленный на телефоне.

- [ ] **Step 1: Подпись release в Gradle**

В начало `app/build.gradle.kts` (над `plugins`) добавить:
```kotlin
import java.util.Properties

val keystoreProps: Properties? = rootProject.file("keystore.properties")
    .takeIf { it.exists() }
    ?.let { f -> Properties().apply { f.inputStream().use { load(it) } } }
```

Внутри `android { }` перед `buildTypes` добавить:
```kotlin
    signingConfigs {
        if (keystoreProps != null) {
            create("release") {
                storeFile = file(keystoreProps.getProperty("storeFile"))
                storePassword = keystoreProps.getProperty("storePassword")
                keyAlias = keystoreProps.getProperty("keyAlias")
                keyPassword = keystoreProps.getProperty("keyPassword")
            }
        }
    }
```

И в `buildTypes { release { ... } }` после `isMinifyEnabled = false`:
```kotlin
            signingConfig = signingConfigs.findByName("release")
```

- [ ] **Step 2: СТОП — автор создаёт ключ сама**

Пароли агент не придумывает и не вводит. Попросить автора выполнить в своём терминале (пароль придумывает сама и сразу сохраняет в Bitwarden вместе с файлом ключа: без него обновить приложение поверх установленного не выйдет):

```bash
mkdir -p ~/.android-keys && keytool -genkeypair -v -keystore ~/.android-keys/anfimorning.jks -alias anfimorning -keyalg RSA -keysize 4096 -validity 36500
```

и создать в корне репозитория `keystore.properties` (он в `.gitignore`):
```properties
storeFile=/Users/<you>/.android-keys/anfimorning.jks
storePassword=<её пароль>
keyAlias=anfimorning
keyPassword=<её пароль>
```

Проверить, что файл не попадает в git: `git status --short` не показывает `keystore.properties`.

- [ ] **Step 3: Собрать release**

Run: `./gradlew testDebugUnitTest assembleRelease`
Expected: `BUILD SUCCESSFUL`, есть `app/build/outputs/apk/release/app-release.apk`.

- [ ] **Step 4: Установить**

Отладочная версия подписана другим ключом, поэтому сначала удалить её (история занятий отладочной версии пропадёт — предупредить автора):

```bash
adb uninstall com.anfixit.anfimorning
adb install app/build/outputs/apk/release/app-release.apk
```

- [ ] **Step 5: Приёмка вместе с автором**

Чек-лист (все пункты — «да»):
- [ ] Иконка «Утро в движении» на телефоне, приложение открывается без интернета.
- [ ] Главная: серия, кнопки «Полная · 23 мин» и «Короткая · 10 мин», памятка, строка про безопасность.
- [ ] Полная версия пройдена целиком: голос ведёт, видео крутятся, «смени сторону» звучит у двусторонних, подготовка 20 с перед кошкой-коровой и дыханием, экран не гаснет.
- [ ] Свернуть приложение посреди упражнения → пауза; на главной есть «Продолжить», он открывает то же упражнение.
- [ ] «Готово» → самочувствие и заметка → в календаре день закрашен, карточка показывает заметку.
- [ ] Напоминание приходит в выбранное время; не приходит, если сегодня уже было завершённое занятие; переживает перезагрузку.

- [ ] **Step 6: README и карточка проекта**

В `README.md` в конец добавить:
```markdown
## Установка на телефон

Release подписывается ключом из `keystore.properties` (в git не хранится).

    ./gradlew assembleRelease
    adb install -r app/build/outputs/apk/release/app-release.apk
```

В `~/dev/ai-context/projects/anfimorning.md` дописать строку: дата, версия 0.1.0 установлена на телефон, где ключ подписи (`~/.android-keys/anfimorning.jks`, копия и пароль в Bitwarden), что осталось на следующий этап (вечерний комплекс, силовые). Этот файл не коммитить.

- [ ] **Step 7: Commit**

```bash
git add app/build.gradle.kts README.md
git commit -m "build: release signing and install notes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Затем спросить автора, пушить ли в `origin` (GitHub + GitLab).
