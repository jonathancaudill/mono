# AGENTS.md

## Cursor Cloud specific instructions

This repo is a single Android product (Spotify client for the Light Phone III):
a Kotlin/Jetpack Compose app (`app/`) with an embedded Rust/`librespot` backend
(`rust/spotify-core/`) exposed via UniFFI. There is no server or database.

### Pre-installed toolchain (persisted in the VM snapshot)

- JDK 17 at `/usr/lib/jvm/java-17-openjdk-amd64`, Android SDK at `/opt/android-sdk`
  (`ANDROID_HOME`), NDK `27.0.12077973`, Rust stable with the `aarch64-linux-android`
  / `x86_64-linux-android` targets, and `cargo-ndk`.
- These plus `JAVA_HOME`, `ANDROID_HOME`, `ANDROID_NDK_HOME`, and `PATH` are exported
  in `~/.bashrc`, so a login shell (`bash -l` / `source ~/.bashrc`) has everything.
- The host UniFFI binding build pulls in `alsa-sys` on Linux (via `rodio-backend`),
  so `libasound2-dev` + `pkg-config` are installed. Without them `scripts/build-rust.sh`
  fails at the host `cargo build` step with "Package alsa was not found".

### Build / lint (commands documented in `README.md`)

- Build: `bash scripts/build-rust.sh` then `./gradlew :app:assembleDebug`. Gradle's
  `preBuild` auto-runs the Rust build via the `cargoBuild` task, so `assembleDebug`
  alone also works. Run gradle with JDK 17 (`JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64`);
  the box's default `java` is 21.
- The NDK version must match AGP's default (`27.0.12077973`) or every native task
  prints a `CXX1104` mismatch warning; `local.properties` and `ANDROID_NDK_HOME`
  already point at it. `local.properties` is gitignored — recreate it with
  `sdk.dir=/opt/android-sdk` + `ndk.dir=/opt/android-sdk/ndk/27.0.12077973` if absent.
- Lint: `./gradlew :app:lintDebug`. It currently FAILS on pre-existing `NewApi`
  errors inside the generated bindings (`ffi/spotify_core.kt` uses `java.lang.ref.Cleaner`,
  API 33, while `minSdk` is 26). These are generated code, not environment problems.
- No unit or instrumentation tests exist; `cargo test` has no tests in `spotify-core`.
- Generated artifacts (`app/src/main/jniLibs/`, `app/src/main/java/com/lightphone/spotify/ffi/`)
  are gitignored. If they go missing, regenerate with `bash scripts/build-rust.sh`.

### Running the app (emulator caveats)

- Running requires an Android emulator. The cloud VM has **no `/dev/kvm`**, so the
  emulator must run in software mode (QEMU TCG): `emulator -avd lp3light -no-audio
  -no-boot-anim -no-snapshot -gpu swiftshader_indirect -accel off -qemu -m 3072`.
  This is CPU-bound and slow; RAM tuning does not fix it (the bottleneck is software
  CPU translation, not memory).
- **Other emulators do not work here** (don't waste time): Genymotion Desktop, MEmu,
  and LDPlayer are VirtualBox/hypervisor-based and need VT-x → blocked by the missing
  `/dev/kvm` (and MEmu/LDPlayer are Windows-only). **Waydroid** needs the kernel
  `binder` driver, which this kernel lacks (`mount -t binder` → "unknown filesystem
  type"), with no `/lib/modules`, no `modprobe`, and zero process capabilities, so it
  cannot run either. The Google emulator in software mode is the only viable option.
- The `google_apis` system image overloads and crashes `system_server` under
  software emulation (`pm`/`activity` services die). Use the lighter
  `system-images;android-30;default;x86_64` image (AVD `lp3light` is already created).
- **Light Phone III screen profile:** `lp3light`'s `config.ini` is set to the LP3
  panel — `hw.lcd.width=1080`, `hw.lcd.height=1240`, `hw.lcd.density=560`
  (≈2.94″ diagonal), `showDeviceFrame=no`. Verify at runtime with `adb shell wm size`
  / `wm density`.
- **Showing it on the desktop:** launch with `DISPLAY=:1` and WITHOUT `-no-window`
  to get a window in the XFCE/TigerVNC Desktop pane. The `swiftshader_indirect` GL
  surface can display a **stale/frozen frame** in the VNC X server; nudge the window
  (`xdotool search --name "Android Emulator" windowmove 60 60`) to force a repaint.
  `adb exec-out screencap -p` always reflects the true live screen.
- Even so, `SystemUI` may show "System UI isn't responding" while the CPU is
  saturated — tap "Wait" / dismiss and relaunch `com.lightphone.spotify/.MainActivity`.
- Full functionality (playback) requires logging in with a **Spotify Premium**
  account — a protocol-level `librespot` requirement that cannot be worked around.
  Without credentials you can still build, install, launch the app, and reach the
  live Spotify OAuth login WebView (the app's entry flow). Note the login WebView
  shows a "Could not connect to the reCAPTCHA service" notice under the flaky
  software-emulated network; the Spotify accounts page itself still loads.
