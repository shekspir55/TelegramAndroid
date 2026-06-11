# Building with Docker

Produces `TMessagesProj_App/build/outputs/apk/afat/debug/app.apk`.

## Command

```bash
SOURCE_DATE_EPOCH="$(git log -1 --format=%ct HEAD)"
docker run --rm \
    -v "/home/ruben/Projects/TelegramAndroid:/project" \
    -w /project \
    -e SOURCE_DATE_EPOCH \
    -e GRADLE_OPTS="-Dorg.gradle.daemon=false -Duser.timezone=UTC" \
    telegramandroid-reproducible:latest \
    bash -c '
        set -e
        git config --global --add safe.directory "*"
        git submodule update --init --recursive

        cd TMessagesProj/jni
        export NDK=/opt/android-sdk/ndk/21.4.7075529
        export NINJA_PATH=/usr/bin/ninja

        # Restore submodule source trees (undo damage from partial previous runs)
        git -C libvpx reset --hard HEAD
        git -C ffmpeg reset --hard HEAD
        git -C boringssl reset --hard HEAD && git -C boringssl clean -fd
        git -C dav1d reset --hard HEAD && git -C dav1d clean -fd

        # Remove stale libvpx output symlinks (ln -s fails if they already exist)
        rm -f libvpx/build/arm64-v8a/include/libvpx \
              libvpx/build/armeabi-v7a/include/libvpx \
              libvpx/build/x86/include/libvpx \
              libvpx/build/x86_64/include/libvpx

        # Build native deps WITHOUT patches — patches break ffmpeg self-compilation
        ./build_libvpx_clang.sh arm arm64
        ./build_ffmpeg_clang.sh arm arm64
        ./build_dav1d_clang.sh arm arm64

        # Apply ffmpeg patches AFTER build, then copy patched headers to output
        # (patches strip get_bits.h for Telegram JNI use, but avc.c needs the originals)
        patch -d ffmpeg -p1 < patches/ffmpeg/0001-compilation-magic.patch
        patch -d ffmpeg -p1 < patches/ffmpeg/0002-compilation-magic-2.patch
        for abi in arm64-v8a armeabi-v7a; do
            install -D ffmpeg/libavformat/dv.h        ffmpeg/build/$abi/include/libavformat/dv.h
            install -D ffmpeg/libavformat/isom.h      ffmpeg/build/$abi/include/libavformat/isom.h
            install -D ffmpeg/libavcodec/bytestream.h ffmpeg/build/$abi/include/libavcodec/bytestream.h
            install -D ffmpeg/libavcodec/get_bits.h   ffmpeg/build/$abi/include/libavcodec/get_bits.h
            install -D ffmpeg/libavcodec/golomb.h     ffmpeg/build/$abi/include/libavcodec/golomb.h
            install -D ffmpeg/libavcodec/vlc.h        ffmpeg/build/$abi/include/libavcodec/vlc.h
            install -D ffmpeg/libavutil/intmath.h     ffmpeg/build/$abi/include/libavutil/intmath.h
        done

        ./patch_boringssl.sh
        ./build_boringssl.sh arm arm64

        cd /project
        # Delete stale CMake/Gradle dirs instead of "gradle clean"
        # ("gradle clean" fails when .cxx has stale ninja state from a previous run)
        rm -rf TMessagesProj/.cxx TMessagesProj_App/build .gradle
        ./gradlew --no-daemon :TMessagesProj_App:assembleAfatDebug
    '
```

## Pitfalls

**ffmpeg patches must be applied AFTER building ffmpeg.**
`0002-compilation-magic-2.patch` removes `get_bitsz` from `get_bits.h`. ffmpeg's own
`libavformat/avc.c` calls `get_bitsz`, so the build fails if patched first. The patches
are only meant for the exported headers consumed by Telegram's JNI C++ code.

**`libvpx/build/` contains source scripts, not just build output.**
`libvpx/build/make/configure.sh` is part of the libvpx source tree. Deleting the whole
`libvpx/build/` directory destroys it. Only delete ABI subdirs (`arm64-v8a`, `armeabi-v7a`).

**`git submodule update` does not restore manually-deleted source files.**
If `build/make/configure.sh` was deleted by a previous bad clean, `git submodule update`
does nothing (the submodule HEAD is already correct). Run `git -C libvpx reset --hard HEAD`
to restore deleted tracked files.

**boringssl needs `git clean -fd` in addition to `reset --hard`.**
`patch_boringssl.sh` creates `crypto/fipsmodule/aes/aes_ige.c` as a new file. It persists
as an untracked file across runs. `reset --hard` won't remove it; `git clean -fd` does.

**libvpx build symlinks persist across runs.**
`build_libvpx_clang.sh` runs `ln -s vpx libvpx` inside the include dir. If the symlink
already exists, `ln` resolves it as a directory and tries to create `libvpx/vpx` inside,
failing with "File exists". Remove the symlinks with `rm -f` before rebuilding.

**Use NDK 21 for native deps.**
`$ANDROID_NDK_HOME` in the Docker image points to NDK 27. The native build scripts need
NDK 21 (`/opt/android-sdk/ndk/21.4.7075529`). Gradle uses NDK 27 automatically — don't
override that.

**Don't use `gradle clean` — delete stale dirs manually instead.**
`gradle clean` triggers `externalNativeBuildCleanDebug` which runs ninja on the `.cxx` dir.
If `.cxx` has stale state from a previous run, ninja crashes and `clean` fails before it
can do anything useful. Instead, delete the dirs directly: `rm -rf TMessagesProj/.cxx TMessagesProj_App/build .gradle`.

**The release keystore must exist and match the passwords in `gradle.properties`.**
`gradle.properties` sets `RELEASE_STORE_PASSWORD=android`, `RELEASE_KEY_ALIAS=androidkey`,
`RELEASE_KEY_PASSWORD=android`. The keystore at `TMessagesProj/config/release.keystore`
must match. If it's missing or corrupted, regenerate it:
```bash
rm -f TMessagesProj/config/release.keystore
keytool -genkeypair -keystore TMessagesProj/config/release.keystore \
  -storepass android -keypass android -alias androidkey \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -dname "CN=Forkgram Debug, O=Debug, C=US"
```

**dav1d must also be built before Gradle.**
`libdav1d.a` is required by the CMake build. Run `./build_dav1d_clang.sh arm arm64`
alongside libvpx, ffmpeg, and boringssl.

**Docker runs as root; set git safe.directory.**
Project files are owned by the host user. Git refuses to operate on them inside Docker.
Fix with `git config --global --add safe.directory "*"` at the start of the script.
