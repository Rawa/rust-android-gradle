# Rust Android Gradle Plugin

Cross compiles rust cargo projects for Android

# Usage

To begin you must first install the rust toolchains for your target platforms.

```
rustup target add armv7-linux-androideabi   # for arm
rustup target add i686-linux-android        # for x86
rustup target add aarch64-linux-android     # for arm64
rustup target add x86_64-linux-android      # for x86_64
...
```

Next add the `cargo` configuration to android project. Point to your cargo project using `module`
and add targets.  Currently supported targets are `arm`, `arm64`, `x86`, `x86_64`, and `default`.
`default` is special: it's whatever Cargo builds when no `--target` parameter is supplied -- usually
the host architecture.

```
cargo {
    module = "../rust"
    targets = ["arm", "x86"]
}
```

Run the `cargoBuild` task to cross compile

```
./gradlew cargoBuild
```

## Configuration

The `cargo` Gradle configuration accepts many options.

### Linking Java code to native libraries

Generated libraries will be added to the Android `jniLibs` source-sets, when correctly referenced in
the `cargo` configuration through the `libname` and/or `targetIncludes` options.  The latter
defaults to `["lib${libname}.so", "lib${libname}.dylib", "lib{$libname}.dll"]`, so the following configuration will
include all `libbackend` libraries generated in the Rust project in `../rust`:

```
cargo {
    module = "../rust"
    libname = "backend"
}
```

Now, Java code can reference the native library using, e.g.,

```java
static {
    System.loadLibrary("backend");
}
```

### Native `apiLevel`

The [Android NDK](https://developer.android.com/ndk/guides/stable_apis) also fixes an API level,
which can be specified using the `apiLevel` option.  This option defaults to the minimum SDK API
level.  As of API level 21, 64-bit builds are possible; and conversely, the `arm64` and `x86_64`
targets require `apiLevel >= 21`.

### Cargo release profile

The `profile` option selects between the `--debug` and `--release` profiles in `cargo`.  *Defaults
to `debug`!*

### Extension reference

### module

The path to the Rust library to build with Cargo; required.  `module` is interpreted as a relative
path to the Gradle `projectDir`.

```groovy
cargo {
    module = '../rust'
}
```

### libname

The library name produced by Cargo; required.

`libname` is used to determine which native libraries to include in the produced AARs and/or APKs.
See also [`targetIncludes`](#targetincludes).

`libname` is also used to determine the ELF SONAME to declare in the Android libraries produced by
Cargo.  Different versions of the Android system linker
[depend on the ELF SONAME](https://android-developers.googleblog.com/2016/06/android-changes-for-ndk-developers.html).

In `Cargo.toml`:

```toml
[lib]
name = "test"
```

In `build.gradle`:

```groovy
cargo {
    libname = 'test'
}
```

### targets

A list of Android targets to build with Cargo; required.

Valid targets are `'arm'`, `'arm64'`, `'x86'`, `'x86_64'`, and `'default'`.  `'default'` is special:
it's whatever Cargo builds when no `--target ...` parameter is supplied -- usually the host
architecture.  `'default'` is useful for testing native code in Android unit tests that run on the
host, not on the target device.  Better support for this feature is
[planned](https://github.com/ncalexan/rust-android-gradle/issues/13).

```groovy
cargo {
    targets = ['arm', 'x86', 'default']
}
```

### verbose

When set, execute `cargo build` with or without the `--verbose` flag.  When unset, respect the
Gradle log level: execute `cargo build` with or without the `--verbose` flag according to whether
the log level is at least `INFO`.  In practice, this makes `./gradlew ... --info` (and `./gradlew
... --debug`) execute `cargo build --verbose ...`.

Defaults to `null`.

```groovy
cargo {
    verbose = true
}
```

### profile

The Cargo [release profile](https://doc.rust-lang.org/book/second-edition/ch14-01-release-profiles.html#customizing-builds-with-release-profiles) to build.

Defaults to `"debug"`.

```groovy
cargo {
    profile = 'release'
}
```

### features

Set the Cargo [features](https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section).

Defaults to passing no flags to `cargo`.

To pass `--all-features`, use
```groovy
cargo {
    features {
        all()
    }
}
```

To pass an optional list of `--features`, use
```groovy
cargo {
    features {
        defaultAnd("x")
        defaultAnd("x", "y")
    }
}
```

To pass `--no-default-features`, and an optional list of replacement `--features`, use
```groovy
cargo {
    features {
        noDefaultFeatures()
        noDefaultFeatures("x")
        noDefaultFeatures "x", "y"
    }
}
```

### targetDirectory

The target directory into which Cargo writes built outputs.

Defaults to `${module}/target`.  `targetDirectory` is interpreted as a relative path to the Gradle
`projectDir`.

```groovy
cargo {
    targetDirectory = 'release'
}
```

### targetIncludes

Which Cargo outputs to consider JNI libraries.

Defaults to `["lib${libname}.so", "lib${libname}.dylib", "lib{$libname}.dll"]`.

```groovy
cargo {
    targetIncludes = ['libnotlibname.so']
}
```

### apiLevel

The Android NDK API level to target.  NDK API levels are not the same as SDK API versions; they are
updated less frequently.  For example, SDK API versions 18, 19, and 20 all target NDK API level 18.

Defaults to the minimum SDK version of the Android project's default configuration.

```groovy
cargo {
    apiLevel = 21
}
```

### defaultToolchainBuildPrefixDir

Android toolchains know where to put their outputs; it's a well-known value like `armeabi-v7a` or
`x86`.  The default toolchains don't know where to put their output; use this to say where.  For use
with [JNA](https://github.com/java-native-access/jna), this should depend on the host platform.

Defaults to `""`.


```groovy
cargo {
    // This puts the output of `cargo build` (the "default" toolchain) into the correct directory
    // for JNA to find it.
    defaultToolchainBuildPrefixDir = com.sun.jna.Platform.RESOURCE_PREFIX
}
```

## Specifying NDK toolchains

The plugin looks for (and will generate) per-target architecture standalone NDK toolchains as
generated by `make_standalone_toolchain.py`.

The toolchains are rooted in a single Android NDK toolchain directory.  In order of preference, the
toolchain root directory is determined by:

1. `rust.androidNdkToolchainDir` in the per-(multi-)project `${rootDir}/local.properties`
1. the environment variable `ANDROID_NDK_TOOLCHAIN_DIR`
1. `${System.getProperty(java.io.tmpdir)}/rust-android-ndk-toolchains`

Note that the Java system property `java.io.tmpdir` is not necessarily `/tmp`, including on macOS hosts.

Each target architecture toolchain is named like `$arch-$apiLevel`: for example, `arm-16` or `arm64-21`.

## Specifying local targets

When developing a project that consumes `rust-android-gradle` locally, it's often convenient to
temporarily change the set of Rust target architectures.  In order of preference, the plugin
determines the per-project targets by:

1. `rust.targets.${project.Name}` for each project in `${rootDir}/local.properties`
1. `rust.targets` in `${rootDir}/local.properties`
1. the `cargo { targets ... }` block in the per-project `build.gradle`

The targets are split on `','`.  For example:

```
rust.targets.library=default
rust.targets=arm,default
```

## Specifying paths to sub-commands (Python and Cargo)

The plugin invokes Python and Cargo.  In order of preference, the plugin determines what command to invoke for Python by:

1. `rust.pythonCommand` in `${rootDir}/local.properties`
1. the environment variable `RUST_ANDROID_GRADLE_PYTHON_COMMAND`
1. the default, `python`

In order of preference, the plugin determines what command to invoke for Cargo by:

1. `rust.cargoCommand` in `${rootDir}/local.properties`
1. the environment variable `RUST_ANDROID_GRADLE_CARGO_COMMAND`
1. the default, `cargo`

Paths must be host operating system specific.  For example, on Windows:

```properties
rust.pythonCommand=c:\Python27\bin\python
```

On Linux,
```shell
env RUST_ANDROID_GRADLE_CARGO_COMMAND=$HOME/.cargo/bin/cargo ./gradlew ...
```

# Development

At top-level, the `publish` Gradle task updates the Maven repository
under `samples`:

```
$ ./gradlew publish
...
$ ls -al samples/maven-repo/org/mozilla/rust-android-gradle/org.mozilla.rust-android-gradle.gradle.plugin/0.4.0/org.mozilla.rust-android-gradle.gradle.plugin-0.4.0.pom
-rw-r--r--  1 nalexander  staff  670 18 Sep 10:09
samples/maven-repo/org/mozilla/rust-android-gradle/org.mozilla.rust-android-gradle.gradle.plugin/0.4.0/org.mozilla.rust-android-gradle.gradle.plugin-0.4.0.pom
```

## Sample projects

To run the sample projects:

```
$ ./gradlew -p samples/library :assembleDebug
...
$ ls -al samples/library/build//outputs/aar/library-debug.aar
-rw-r--r--  1 nalexander  staff  8926315 18 Sep 10:22 samples/library/build//outputs/aar/library-debug.aar
```

## Real projects

To test in a real project, use the local Maven repository in your `build.gradle`, like:

```
buildscript {
    repositories {
        maven {
            url "file:///Users/nalexander/Mozilla/rust-android-gradle/samples/maven-repo"
        }
    }

    dependencies {
        classpath 'org.mozilla.rust-android-gradle:plugin:0.3.0'
    }
}
```
