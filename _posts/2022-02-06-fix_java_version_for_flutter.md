---
title: Fix java version for Flutter
last_modified_at: 2022-02-07 15:17:57 +0100
---

I had some issues with building Android applications in Flutter. With the following as error message:

```txt
FAILURE: Build failed with an exception.

* Where:
Build file 'android/build.gradle'

* What went wrong:
Could not compile build file 'android/build.gradle'.
> startup failed:
  General error during semantic analysis: Unsupported class file major version 61

  java.lang.IllegalArgumentException: Unsupported class file major version 61
```

Install java 1.8

```sh
brew install --cask temurin8
```

The following java versions is now installed:

```console
$ /usr/libexec/java_home -V
Matching Java Virtual Machines (3):
    17.0.1 (arm64) "Eclipse Temurin" - "Eclipse Temurin 17" /Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home
    1.8.0_322 (x86_64) "Eclipse Temurin" - "Eclipse Temurin 8" /Library/Java/JavaVirtualMachines/temurin-8.jdk/Contents/Home
```

Run `flutter build apk` either with inline variable as:

```sh
JAVA_HOME="$(/usr/libexec/java_home -v1.8)" flutter build apk
```

Or (more preferrable) `export` the variable with:

```sh
export JAVA_HOME="$(/usr/libexec/java_home -v1.8)"
# export PATH=$JAVA_HOME/bin:$PATH   # this might be needed
flutter build apk
```

Et volià `✓  Built build/app/outputs/flutter-apk/app-release.apk (46.0MB).`.

References:

* [Upgrade to OpenJDK Temurin using Homebrew](https://www.yippeecode.com/topics/upgrade-to-openjdk-temurin-using-homebrew/)
