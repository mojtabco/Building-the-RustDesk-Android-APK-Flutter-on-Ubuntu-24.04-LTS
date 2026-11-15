# Building the RustDesk Android APK (Flutter) on Ubuntu 24.04 LTS

A step-by-step guide, from `git clone` to a **debug** APK and then a signed **release** APK (with a Keystore), for the `arm64-v8a`, `armeabi-v7a`, `x86_64` and `x86` architectures.

> By default nothing is customized: the name, icon, server and UI stay exactly as in upstream RustDesk. If you want your own name, icon or private server, do the optional [Step 1b](#step-1b-optional-customize-before-building) right after cloning.

---

## Assumed environment

| Item | Value |
|---|---|
| OS | Ubuntu 24.04 LTS (fresh install, not updated) |
| RAM | 16 GB |
| Disk | 128 GB SSD |
| Internet | Open access. If Google services are blocked or filtered for you, also follow [Step 10b](#step-10b-optional-restricted-networks--mirrors) |
| User | Regular user (**no `sudo`** for the build scripts) |

## Versions used

| Tool | Version |
|---|---|
| Flutter | `3.24.5` (Dart 3.5.4) |
| Android NDK | `27.0.12077973` |
| Android cmdline-tools | `11076708` |
| Android SDK Platform | `android-34` and `android-35` |
| Android Build-tools | `34.0.0` and `35.0.0` |
| Android Gradle Plugin / Gradle / Kotlin | `8.10.1` / `8.11.1` / `2.1.21` (as pinned by the project at the time of writing) |
| JDK | OpenJDK 17 |
| flutter_rust_bridge_codegen | `1.80.1` (with the `uuid` feature) |
| Rust | stable (rustup) |
| vcpkg | latest `master` (full clone) |
| RustDesk | `master` (version `1.5.0` at the time of writing) |

> **Why both 34 and 35?** The Android Gradle Plugin used by the project asks for Build-tools `35.0.0` by default, and some plugins compile against `android-35`. If it is missing, the build fails with `Failed to find Build Tools revision 35.0.0`. Installing both avoids that.

## Directory layout (everything under `~`)

```
~/rustdesk          RustDesk source
~/vcpkg             native dependency manager
~/flutter           Flutter SDK
~/Android/Sdk       Android SDK and NDK
~/.cargo            Rust and cargo-ndk
```

## ABI map

| ABI | Rust target | vcpkg (deps argument) | Script | Flutter `--target-platform` | Use |
|---|---|---|---|---|---|
| `arm64-v8a` | `aarch64-linux-android` | `arm64-v8a` | `ndk_arm64.sh` | `android-arm64` | modern phones |
| `armeabi-v7a` | `armv7-linux-androideabi` | `armeabi-v7a` | `ndk_arm.sh` | `android-arm` | older 32-bit phones |
| `x86_64` | `x86_64-linux-android` | `x86_64` | `ndk_x64.sh` | `android-x64` | emulators and Intel devices |
| `x86` | `i686-linux-android` | `x86` | `ndk_x86.sh` | `android-x86` | 32-bit emulators (debug only) |

> `arm64-v8a` and `x86_64` were tested in practice. `armeabi-v7a` and `x86` use the same procedure but are untested. If you only need a modern phone, `arm64-v8a` alone is enough and you can skip the others.

## Approximate time and disk space

- vcpkg, per ABI: **1 to several hours**
- Rust build, per ABI: **20 to 60 minutes**
- Disk space for all 4 ABIs: about **40 to 60 GB** (estimate)

---

## Step 1. Clone RustDesk

```bash
cd ~
sudo apt update && sudo apt upgrade -y
sudo apt install -y git
git clone --recurse-submodules https://github.com/rustdesk/rustdesk
```

Check that the submodules are not empty, and note the commit for reproducibility:

```bash
ls ~/rustdesk/libs/hbb_common | head
git -C ~/rustdesk rev-parse HEAD
```

If `libs/hbb_common` is empty:

```bash
cd ~/rustdesk && git submodule update --init --recursive
```

## Step 1b. (Optional) Customize before building

Skip this step if you want the stock RustDesk app. Otherwise make these changes **right after cloning**, before Step 9, because anything under `src/` or `libs/` is compiled into `librustdesk.so`.

- Changes in `libs/hbb_common/src/config.rs` or `src/common.rs` (Rust): rebuild every ABI with Step 9, then Step 11.
- Changes to icons, strings, manifest or Dart files: no Rust rebuild, only Step 11 (or Step 14).

### 1b.1 Change the app icon

Launcher icons:

```bash
cd ~/rustdesk/flutter/android/app/src/main/res
ls -d mipmap-*
file mipmap-*/ic_launcher*.png     # shows the pixel size of each file
```

Replace every `ic_launcher*.png` in each `mipmap-*` folder with your own image, keeping the **same file name and the same pixel size** as the file you replace. The background color of the adaptive icon is set in:

```
flutter/android/app/src/main/res/values/ic_launcher_background.xml
```

Icons and logos used inside the app UI:

```bash
cd ~/rustdesk/flutter/assets
ls
```

Replace the files you want with your own, keeping the same names.

### 1b.2 Change the app name

The name shown under the launcher icon comes from the Android manifest and strings, not from the Rust code:

- `flutter/android/app/src/main/AndroidManifest.xml`: `android:label` (app name) and the label of the Accessibility service
- `flutter/android/app/src/main/res/values/strings.xml`: display strings (including the Accessibility service name)

```bash
cd ~/rustdesk/flutter/android/app/src/main
grep -n "RustDesk" AndroidManifest.xml res/values/strings.xml
```

Replace `RustDesk` in the matches with your name. Do not change the package name `com.carriez.flutter_hbb` unless you know what you are doing; it is used in several places.

### 1b.3 Connect to your own server (hardcode the address and key)

Edit `libs/hbb_common/src/config.rs`:

```bash
cd ~/rustdesk/libs/hbb_common/src
nano config.rs
```

Set your server address (domain or IP):

```rust
pub const RENDEZVOUS_SERVERS: &'static [&'static str] = &["myserver.com"]; // or an IP address
```

Set your server's public key (the content of `id_ed25519.pub` from your hbbs server's data directory) in `RS_PUB_KEY`:

```rust
pub const RS_PUB_KEY: &'static str = "your_public_key_here";
```

Also update `src/common.rs` (around line 517; the exact line number can differ between versions, so search for it):

```bash
cd ~/rustdesk
grep -n "RENDEZVOUS_PORT" src/common.rs
```

```rust
let rendezvous_server = socket_client::get_target_addr(&format!("myserver.com:{}", config::RENDEZVOUS_PORT))?;
```

Replace `myserver.com` with your own address in all three places. Because these are Rust files, rebuild every ABI with Step 9 afterwards.

### 1b.4 Important files for UI changes

| File | What it controls |
|---|---|
| `flutter/lib/mobile/pages/home_page.dart` | bottom tabs (`initPages()`) |
| `flutter/lib/mobile/pages/settings_page.dart` | settings page |
| `flutter/lib/mobile/pages/server_page.dart` | Share screen and `showScamWarning` |
| `flutter/lib/models/server_model.dart` | `isStart`, `stopService()`, `toggleService()` |
| `flutter/lib/common.dart` | theme (`lightTheme` / `darkTheme`, `appBarTheme`) |
| `flutter/android/app/src/main/AndroidManifest.xml` | `android:label` (app name), Accessibility service label |
| `flutter/android/app/src/main/res/values/strings.xml` | display strings |
| `flutter/android/app/src/main/res/values/ic_launcher_background.xml` | launcher icon background |
| `flutter/android/app/src/main/res/mipmap-*/ic_launcher*.png` | launcher icons |

> If you distribute your customized APK, the AGPL-3.0 terms (see the License section) apply to your modified source as well.

## Step 2. System packages

```bash
sudo apt install -y \
  git curl wget ca-certificates zip unzip tar xz-utils \
  build-essential cmake ninja-build pkg-config nasm yasm \
  clang libclang-dev llvm-dev \
  openjdk-17-jdk python3 python3-venv \
  libssl-dev autoconf automake libtool autoconf-archive libltdl-dev bison flex gperf

sudo apt install -y libgtk-3-dev
```

**Optional but recommended:** add swap so the process is not `Killed` if ffmpeg or Rust needs a lot of memory:

```bash
sudo fallocate -l 8G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
free -h
```

## Step 3. Rust, targets and cargo-ndk

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

rustup target add \
  aarch64-linux-android \
  armv7-linux-androideabi \
  x86_64-linux-android \
  i686-linux-android

cargo install cargo-ndk
```

## Step 4. Android SDK and NDK

```bash
mkdir -p ~/Android/Sdk/cmdline-tools
cd ~/Android/Sdk/cmdline-tools
wget https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-11076708_latest.zip
mv cmdline-tools latest
rm commandlinetools-linux-11076708_latest.zip
```

> The path must be exactly `~/Android/Sdk/cmdline-tools/latest/bin`.

Environment variables (run this block **once**, otherwise the lines are appended to `~/.bashrc` again):

```bash
cat >> ~/.bashrc << 'EOF'
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export ANDROID_HOME=$HOME/Android/Sdk
export ANDROID_SDK_ROOT=$HOME/Android/Sdk
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/27.0.12077973
export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools
EOF
source ~/.bashrc
```

If your JDK directory has a different name, check it with `ls /usr/lib/jvm/` and adjust `JAVA_HOME`.

Install the packages:

```bash
export ANDROID_HOME=~/Android/Sdk
SDKM=$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager

yes | $SDKM --licenses
$SDKM "platform-tools" "platforms;android-34" "platforms;android-35" \
      "build-tools;34.0.0" "build-tools;35.0.0" "ndk;27.0.12077973"
$SDKM --list_installed
ls $ANDROID_HOME/ndk
```

> `ANDROID_NDK_HOME` points to a directory that only exists after the NDK is installed, so `ls $ANDROID_NDK_HOME` fails before this step.
>
> During the build, Gradle may download additional Android platforms on its own. That is normal.
>
> If `dl.google.com` is blocked for you, `sdkmanager` will hang or time out. See [Step 10b](#step-10b-optional-restricted-networks--mirrors).

## Step 5. Flutter 3.24.5

```bash
cd ~
wget https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_3.24.5-stable.tar.xz
tar xf flutter_linux_3.24.5-stable.tar.xz
rm flutter_linux_3.24.5-stable.tar.xz

echo 'export PATH=$PATH:$HOME/flutter/bin' >> ~/.bashrc
source ~/.bashrc

flutter config --android-sdk $HOME/Android/Sdk
yes | flutter doctor --android-licenses
flutter doctor -v
```

In the `flutter doctor` output, **Android toolchain** must show ✓. (Linux desktop / Chrome warnings do not matter.)

> Stay on **3.24.5**. Newer versions are incompatible with the current RustDesk code.

## Step 6. vcpkg

```bash
cd ~
git clone https://github.com/microsoft/vcpkg
cd ~/vcpkg
./bootstrap-vcpkg.sh

echo 'export VCPKG_ROOT=$HOME/vcpkg' >> ~/.bashrc
source ~/.bashrc
echo $VCPKG_ROOT
```

> Clone **without** `--depth 1` and do not check out an old commit. The project needs recent package versions, and with an old commit you get `no version database entry` errors.

## Step 7. Flutter dependencies and bridge files

`bridge_generated.rs` and `generated_bridge.dart` are not in git and must be generated. Run `pub get` first (it creates `pubspec.lock`):

```bash
cd ~/rustdesk/flutter
flutter pub get
ls -la pubspec.lock
```

Then run the codegen:

```bash
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked

cd ~/rustdesk
~/.cargo/bin/flutter_rust_bridge_codegen \
  --rust-input ./src/flutter_ffi.rs \
  --dart-output ./flutter/lib/generated_bridge.dart \
  --c-output ./flutter/macos/Runner/bridge_generated.h

ls -la src/bridge_generated*.rs flutter/lib/generated_bridge.dart
```

Three files should be created: `src/bridge_generated.rs`, `src/bridge_generated.io.rs` and `flutter/lib/generated_bridge.dart`.

## Step 8. Native dependencies (vcpkg) per ABI

Run **without `sudo`**. Each command takes hours. Only run the ABIs you need:

```bash
cd ~/rustdesk/flutter
chmod +x *.sh

./build_android_deps.sh arm64-v8a   2>&1 | tee ~/deps_arm64.log
./build_android_deps.sh armeabi-v7a 2>&1 | tee ~/deps_arm.log
./build_android_deps.sh x86_64      2>&1 | tee ~/deps_x64.log
./build_android_deps.sh x86         2>&1 | tee ~/deps_x86.log
```

Verify:

```bash
ls ~/vcpkg/installed/
```

You should see `arm64-android`, `arm-neon-android` (and the `arm-android` link), `x64-android` and `x86-android`.

If you run low on disk space, you can delete vcpkg's temporary directories when it finishes:

```bash
rm -rf ~/vcpkg/buildtrees ~/vcpkg/downloads ~/vcpkg/packages
```

## Step 9. Build `librustdesk.so` per ABI and copy it to the right place

Rust outputs `liblibrustdesk.so`, but Android expects `librustdesk.so`, so the file is renamed while copying. `libc++_shared.so` is needed alongside it. One function does all of this:

```bash
source ~/.cargo/env
export VCPKG_ROOT=$HOME/vcpkg
export ANDROID_NDK_HOME=$HOME/Android/Sdk/ndk/27.0.12077973
NDK_BIN=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64

build_abi() {
  local script=$1 triple=$2 abi=$3 libdir=$4
  local dst=$HOME/rustdesk/flutter/android/app/src/main/jniLibs/$abi
  cd ~/rustdesk && ./flutter/$script || return 1
  mkdir -p "$dst"
  cp target/$triple/release/liblibrustdesk.so "$dst/librustdesk.so"
  cp $NDK_BIN/sysroot/usr/lib/$libdir/libc++_shared.so "$dst/"
  $NDK_BIN/bin/llvm-strip "$dst/librustdesk.so"
  ls -la "$dst"
}
```

Then build only the ABIs you want (20 to 60 minutes each):

```bash
build_abi ndk_arm64.sh aarch64-linux-android    arm64-v8a   aarch64-linux-android
build_abi ndk_arm.sh   armv7-linux-androideabi  armeabi-v7a arm-linux-androideabi
build_abi ndk_x64.sh   x86_64-linux-android     x86_64      x86_64-linux-android
build_abi ndk_x86.sh   i686-linux-android       x86         i686-linux-android
```

Final file layout:

```
~/rustdesk/flutter/android/app/src/main/jniLibs/
├── arm64-v8a/    librustdesk.so  libc++_shared.so
├── armeabi-v7a/  librustdesk.so  libc++_shared.so
├── x86_64/       librustdesk.so  libc++_shared.so
└── x86/          librustdesk.so  libc++_shared.so
```

> If `ndk_*.sh` fails with a `Cargo.lock` or `--locked` error: **do not edit `Cargo.toml`** (especially `name` and `default-run`).

## Step 10. Gradle memory settings

The default memory is too low for this project and the build fails with `Java heap space` (in `compressDebugAssets` and `JetifyTransform`). With 16 GB of RAM:

```bash
cd ~/rustdesk/flutter/android
sed -i '/^org.gradle.jvmargs/d' gradle.properties
echo 'org.gradle.jvmargs=-Xmx6g -XX:MaxMetaspaceSize=1g -Dfile.encoding=UTF-8' >> gradle.properties
echo 'kotlin.daemon.jvmargs=-Xmx1g' >> gradle.properties
grep -n "jvmargs" gradle.properties
```

## Step 10b. (Optional) Restricted networks / mirrors

**Skip this step if you have normal access to Google services.** Do it if Gradle fails with errors such as:

- `Plugin [id: 'com.android.application', version: '8.10.1'] was not found`
- `Could not find com.android.tools.build:gradle:7.4.2` (required by a plugin such as `desktop_drop`)
- builds that take 5 to 12 minutes and then fail, or `sdkmanager` that hangs

These happen when `dl.google.com` (Google Maven) is filtered while Maven Central and the Gradle Plugin Portal still work. Quick check:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://dl.google.com/dl/android/maven2/androidx/core/core/maven-metadata.xml
curl -s -o /dev/null -w "%{http_code}\n" https://maven.aliyun.com/repository/google/com/android/tools/build/gradle/7.4.2/gradle-7.4.2.pom
```

If the first is not `200` and the second is `200`, use the Aliyun mirrors below.

### 10b.1 `flutter/android/settings.gradle`

Put the mirrors first in `pluginManagement { repositories { ... } }`:

```gradle
pluginManagement {
    // ... existing flutterSdkPath / includeBuild lines stay as they are ...
    repositories {
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

### 10b.2 `flutter/android/build.gradle`

Replace the `allprojects { repositories { ... } }` block. (`jcenter()` is deprecated and read-only, so `mavenCentral()` replaces it.)

```gradle
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        maven { url 'https://maven.aliyun.com/repository/public' }
        google()
        mavenCentral()
        maven { url 'https://jitpack.io' }
    }
}
```

### 10b.3 Global Gradle init script (for plugins with their own `buildscript`)

Some Flutter plugins (for example `desktop_drop`) declare their own old `buildscript { repositories { google() ... } }`, which the blocks above do not reach. A global init script injects the mirror there:

```bash
mkdir -p ~/.gradle/init.d
cat > ~/.gradle/init.d/mirrors.gradle << 'EOF'
settingsEvaluated { settings ->
    settings.pluginManagement {
        repositories {
            maven { url 'https://maven.aliyun.com/repository/google' }
            maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
            maven { url 'https://maven.aliyun.com/repository/public' }
        }
    }
}

allprojects {
    buildscript {
        repositories {
            maven { url 'https://maven.aliyun.com/repository/google' }
            maven { url 'https://maven.aliyun.com/repository/public' }
        }
    }
}
EOF
```

> Do **not** add a project-level `repositories { ... }` block to this init script. The project is configured to prefer settings repositories, and Gradle fails with `Build was configured to prefer settings repositories over project repositories but repository 'maven' was added by settings file`.

### 10b.4 Flutter engine and pub mirrors

```bash
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```

(Step 11 already sets these around the build.)

### 10b.5 Reset Gradle after changes

```bash
cd ~/rustdesk/flutter/android && ./gradlew --stop && cd ..
```

## Step 11. Build the debug APK

Only build for the ABIs you built in Step 9. If the `.so` file is missing for an ABI, the app crashes immediately on that architecture.

```bash
# Only needed on restricted networks (see Step 10b)
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn

cd ~/rustdesk/flutter
flutter pub get

# Modern phones only (arm64)
flutter build apk --debug --target-platform android-arm64

# Emulator / Intel only (x86_64)
# flutter build apk --debug --target-platform android-x64

# 32-bit ARM only
# flutter build apk --debug --target-platform android-arm

# 32-bit x86 only (debug only)
# flutter build apk --debug --target-platform android-x86

# Multiple architectures in one APK (only if all .so files are built)
# flutter build apk --debug --target-platform android-arm,android-arm64,android-x64

unset FLUTTER_STORAGE_BASE_URL
```

The first build takes 10 to 30 minutes (Gradle and dependency downloads). If it fails, rerun with `--stacktrace` for details. Output:

```
~/rustdesk/flutter/build/app/outputs/flutter-apk/app-debug.apk
```

Check that the libraries are inside the APK:

```bash
unzip -l build/app/outputs/flutter-apk/app-debug.apk | grep librustdesk
```

You should see one `lib/<abi>/librustdesk.so` line for every ABI you built.

## Step 12. Install and test

Connect the phone via USB (Developer options → USB debugging):

```bash
adb devices
adb install -r ~/rustdesk/flutter/build/app/outputs/flutter-apk/app-debug.apk
```

If the app does not open or crashes, collect logs:

```bash
adb logcat -c
# launch the app, then:
adb logcat -d *:E | grep -E "rustdesk|AndroidRuntime|dlopen|UnsatisfiedLink|FATAL"
```

| Message | Cause |
|---|---|
| `couldn't find "librustdesk.so"` | the `.so` for that device's ABI was not built or copied |
| `INSTALL_FAILED_NO_MATCHING_ABIS` | the APK was not built for the device's architecture |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | the previous version has a different signature; run `adb uninstall com.carriez.flutter_hbb` first |

> ✅ **Only continue if the debug build runs correctly.**

---

## Step 13. Create a Keystore for release

The release APK must be signed. Create this file only **once**.

```bash
keytool -genkeypair -v \
  -keystore ~/rustdesk-release.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias rustdesk
```

keytool asks for passwords and details. Notes:

- **The country code must be two letters** (for example `IR`, `US`, `DE`), not a phone code such as `+98`.
- Write the password down. If you just press Enter for the key password, it becomes the same as the keystore password.

Verify:

```bash
keytool -list -keystore ~/rustdesk-release.jks
```

Then create `key.properties` (the project reads it from `flutter/android/key.properties`):

```bash
cat > ~/rustdesk/flutter/android/key.properties << 'EOF'
storePassword=<STORE_PASSWORD>
keyPassword=<KEY_PASSWORD>
keyAlias=rustdesk
storeFile=/home/<USER>/rustdesk-release.jks
EOF
```

Replace `<STORE_PASSWORD>`, `<KEY_PASSWORD>` and `<USER>` with the real values. Do not put spaces or quotes after `=`.

> ⚠️ **Keep a backup** of `~/rustdesk-release.jks` and its password outside this machine. If you lose it, you can no longer ship updates over the installed version. Never commit `key.properties` or the `.jks` to Git:
>
> ```bash
> echo "android/key.properties" >> ~/rustdesk/flutter/.gitignore
> echo "*.jks" >> ~/rustdesk/flutter/.gitignore
> ```

## Step 14. Build the release version

### 14.1 Disable lint

The release build also runs the lint step, which is not needed to produce the APK and can fail because of extra dependency downloads (`desugar_jdk_libs` and others). Append this block to the end of `flutter/android/build.gradle`:

```bash
cat >> ~/rustdesk/flutter/android/build.gradle << 'EOF'

gradle.taskGraph.whenReady { graph ->
    graph.allTasks.each { t ->
        if (t.name.toLowerCase().contains("lint")) {
            t.enabled = false
        }
    }
}
EOF
```

### 14.2 Build

```bash
cd ~/rustdesk/flutter/android && ./gradlew --stop && cd ..

# Modern phones only (arm64)
flutter build apk --release --target-platform android-arm64
```

Output:

```
~/rustdesk/flutter/build/app/outputs/flutter-apk/app-release.apk
```

For multiple architectures (only if all three `.so` files are built), one APK per ABI:

```bash
flutter build apk --release --split-per-abi \
  --target-platform android-arm,android-arm64,android-x64
```

Outputs:

```
app-arm64-v8a-release.apk
app-armeabi-v7a-release.apk
app-x86_64-release.apk
```

> `x86` is not supported in release (debug only).

### 14.3 Verify and install

```bash
cd ~/rustdesk/flutter/build/app/outputs/flutter-apk
unzip -l app-release.apk | grep librustdesk
~/Android/Sdk/build-tools/34.0.0/apksigner verify --print-certs app-release.apk

adb uninstall com.carriez.flutter_hbb      # if the debug build is installed (different signature)
adb install app-release.apk
```

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `no version database entry for ...` | old vcpkg or shallow clone | full clone, then `git -C ~/vcpkg pull && ~/vcpkg/bootstrap-vcpkg.sh` |
| `file not found for module bridge_generated` (E0583) and several `E0277` | bridge files not generated | Step 7 |
| `missing pubspec.lock` in codegen | `flutter pub get` not run or failed | repeat `flutter pub get` until it succeeds |
| `Killed` during build | out of RAM | add swap (Step 2) and close other programs |
| `Java heap space` in Gradle | default memory too low | Step 10 |
| `cannot update the lock file ... --locked` | `name` changed in `Cargo.toml` | `git checkout Cargo.toml Cargo.lock` |
| `SigningConfig "release" is missing required property "storeFile"` | `key.properties` not created | Step 13 |
| `command not found` / `permission denied` on `.sh` | not executable, or run with `sudo` | `chmod +x ~/rustdesk/flutter/*.sh` and **no** `sudo` |
| App name or icon did not change after customizing | the name comes from the manifest/strings, and Rust edits need a rebuild | edit `AndroidManifest.xml` / `strings.xml` (Step 1b.2); rerun Step 9 for Rust changes; `adb uninstall` the old app before installing |
| `Failed to find Build Tools revision 35.0.0` | Build-tools 35 not installed | install `build-tools;35.0.0` and `platforms;android-35` (Step 4). Do not lower `compileSdk` in `app/build.gradle` |
| `Plugin [id: 'com.android.application', version: '8.10.1'] was not found` | Google Maven unreachable | Step 10b |
| `Could not find com.android.tools.build:gradle:7.4.2` (project `:desktop_drop`) | a plugin's own `buildscript` uses `google()` | init script in Step 10b.3 |
| `Build was configured to prefer settings repositories ...` | project-level `repositories` added from an init script | remove it from the init script (Step 10b.3) |
| `Flutter was unable to detect project Gradle, Java, AGP, and KGP versions` | usually a repository resolution problem (same as above) | Step 10b; it is only a warning once the build succeeds |

## License

RustDesk is licensed under **AGPL-3.0**. If you distribute the APK you build to others, you must also make the source (including your changes) available.

## Credits

Thanks to the RustDesk team for their outstanding open-source project. If this guide helped you, give the original repository a star!

---

## 🚀 About Me

[![portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://github.com/mojtabco) [![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/mojtaba-askari-b0522b378/)

