---
title: Integrate a Flutter module into your Android project
title: 将 Flutter module 集成到 Android 项目
short-title: Integrate Flutter
short-title: 集成 Flutter
description: Learn how to integrate a Flutter module into your existing Android project.
description: 了解如何将 Flutter module 集成到你现有的 Android 项目中。
---

Flutter can be embedded into your existing Android application piecemeal, as a source code Gradle subproject or as AARs.

The integration flow can be done using the Android Studio IDE with the
[Flutter plugin](https://plugins.jetbrains.com/plugin/9212-flutter) or manually.

{{site.alert.warning}}
Your existing Android app may support architectures such as `mips`
or `x86`/`x86_64`. Flutter currently [only supports](/docs/resources/faq#what-devices-and-os-versions-does-flutter-run-on)
building ahead-of-time (AOT) compiled libraries for `armeabi-v7a` and `arm64-v8a`.

Consider using the [`abiFilters`](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.NdkOptions.html)
Android Gradle Plugin API to limit the supported architectures in your APK. Doing this
avoids a missing `libflutter.so` runtime crash, for example:

<?code-excerpt "MyApp/app/build.gradle" title?>
```gradle
android {
  //...
  defaultConfig {
    ndk {
      // Filter for architectures supported by Flutter.
      abiFilters 'armeabi-v7a', 'arm64-v8a'
    }
  }
}
```

The Flutter engine has an `x86` and `x86_64` version. When using an emulator
in debug Just-In-Time (JIT) mode, the Flutter module still runs correctly.
{{site.alert.end}}

## Using Android Studio

The Android Studio IDE is a convenient way of integrating your Flutter
module automatically. With Android Studio, you can co-edit both your Android
code and your Flutter code in the same project. You can also continue to use
your normal IntelliJ Flutter plugin functionalities such as Dart code completion,
hot reload, and widget inspector.

Add-to-app flows with Android Studio are only supported on Android Studio 3.6,
and only supports integrating using a source code Gradle subproject, rather than
using AARs. See below for more details on the distinction.

Using the `File > New > New Module...` menu in Android Studio in your existing
Android project, you can either create a new Flutter module to integrate, or
select an existing Flutter module that was created previously.

{% include app-figure.md image="development/add-to-app/android/project-setup/ide-new-module.png" %}

If you create a new module, you can use a wizard to
select the module name, location, and so on.

{% include app-figure.md image="development/add-to-app/android/project-setup/ide-wizard.png" %}

The Android Studio plugin automatically configures your Android project
to add your Flutter module as a dependency, and your app is ready to build.

{{site.alert.note}}
To see the changes that were automatically made to your Android project by the
IDE plugin, consider using source control for your Android project before performing
any steps. A local diff shows the changes.
{{site.alert.end}}

{{site.alert.tip}}
By default, your project's Project pane is probably showing the 'Android' view. If
you can't see your new Flutter files in the Project pane, ensure that
your Project pane is set to display 'Project Files', which shows all files
without filtering.
{{site.alert.end}}

Your app now includes the Flutter module as a dependency. You can jump to the
[API usage documentations](/docs/development/add-to-app/android/add-flutter-screen)
to follow the next steps.

## Manual integration

To integrate a Flutter module with an existing Android app manually, without
using Flutter's Android Studio plugin, follow these steps:

### Create a Flutter module

Let's assume that you have an existing Android app at `some/path/MyApp`, and that you
want your Flutter project as a sibling:
```terminal
$ cd some/path/
$ flutter create -t module --org com.example my_flutter
```
This creates a `some/path/my_flutter/` Flutter module project with some Dart
code to get you started and a `.android/` hidden subfolder that wraps up the
module project in an Android library.

### Java 8 requirement

The Flutter Android engine uses Java 8 features.

Before attempting to connect your Flutter module project to your host Android app,
ensure that your host Android app declares the following source
compatibility within your app's `build.gradle` file, under the `android { }`
block, such as:

<?code-excerpt "MyApp/app/build.gradle" title?>
```gradle
android {
  //...
  compileOptions {
    sourceCompatibility 1.8
    targetCompatibility 1.8
  }
}
```

### Add the Flutter module as a dependency

Next, add the Flutter module as a dependency of your existing app in Gradle.
There are two ways to achieve this. The AAR mechanism creates generic Android
AARs as intermediaries that packages your Flutter module. This is good when your
downstream app builders don't want to have the Flutter SDK installed. But,
it adds one more build step if you build frequently.

The source code subproject mechanism is a convenient one-click build
process, but requires the Flutter SDK. This is the mechanism used by the
Android Studio IDE plugin.

#### Option A - Depend on the Android Archive (AAR)

This option packages your Flutter library as a generic local Maven repository composed
of AARs and POMs artifacts. This option allows your team to build the host app without
installing the Flutter SDK. You can then distribute the artifacts from
a local or remote repository.

Let's assume you built a Flutter module at `some/path/my_flutter`, and then run:
```terminal
$ cd some/path/my_flutter
$ flutter build aar
```

Then, follow the on-screen instructions to integrate.

{% include app-figure.md image="development/add-to-app/android/project-setup/build-aar-instructions.png" %}

More specifically, this command creates (by default all debug/profile/release
modes) a [local repository](https://docs.gradle.org/current/userguide/repository_types.html#sub:maven_local), with the following files:

```text
build/host/outputs/repo
└── com
    └── example
        └── my_flutter
            ├── flutter_release
            │   ├── 1.0
            │   │   ├── flutter_release-1.0.aar
            │   │   ├── flutter_release-1.0.aar.md5
            │   │   ├── flutter_release-1.0.aar.sha1
            │   │   ├── flutter_release-1.0.pom
            │   │   ├── flutter_release-1.0.pom.md5
            │   │   └── flutter_release-1.0.pom.sha1
            │   ├── maven-metadata.xml
            │   ├── maven-metadata.xml.md5
            │   └── maven-metadata.xml.sha1
            ├── flutter_profile
            │   ├── ...
            └── flutter_debug
                └── ...
```

To depend on the AAR, the host app must be able to find these files.

To do that, edit `app/build.gradle` in your host app such as it includes
the local repository and the dependency:

<?code-excerpt "MyApp/app/build.gradle" title?>
```gradle
android {
  // ...
}

repositories {
  maven {
    url 'some/path/my_flutter/build/host/outputs/repo'
    // This is relative to the location of the build.gradle file
    // if using a relative path.
  }
  maven {
    url 'http://download.flutter.io'
  }
}

dependencies {
  // ...
  debugImplementation 'com.example.flutter_module:flutter_debug:1.0
  profileImplementation 'com.example.flutter_module:flutter_profile:1.0
  releaseImplementation 'com.example.flutter_module:flutter_release:1.0
}
```

{{site.alert.tip}}
You can also build an AAR for your Flutter module in Android Studio using
the `Build > Flutter > Build AAR` menu.

{% include app-figure.md image="development/add-to-app/android/project-setup/ide-build-aar.png" %}
{{site.alert.end}}

Your app now includes the Flutter module as a dependency. You can follow the
next steps in the [API usage documentations](/docs/development/add-to-app/android/add-flutter-screen).

#### Option B - Depend on the module's source code

This option enables a one-step build for both your Android project and Flutter project.
This option is convenient when you work on both parts simultaneously and rapidly
iterate, but your team must install the Flutter SDK to build the host app.

Include the Flutter module as a subproject in the host app's `settings.gradle`:

<?code-excerpt "MyApp/settings.gradle" title?>
```groovy
include ':app'                                     // assumed existing content
setBinding(new Binding([gradle: this]))                                 // new
evaluate(new File(                                                      // new
  settingsDir.parentFile,                                               // new
  'my_flutter/.android/include_flutter.groovy'                          // new
))                                                                      // new
```

Assuming `my_flutter` is a sibling to `MyApp`.

The binding and script evaluation allows the Flutter module to `include` itself
(as `:flutter`) and any Flutter plugins used by the module (as `:package_info`,
`:video_player`, etc) in the evaluation context of your `settings.gradle`.

Introduce an `implementation` dependency on the Flutter module from your app:
<?code-excerpt "MyApp/app/build.gradle" title?>
```groovy
dependencies {
  implementation project(':flutter')
}
```

Your app now includes the Flutter module as a dependency. You can follow the
next steps in the [API usage documentations](/docs/development/add-to-app/android/add-flutter-screen).
