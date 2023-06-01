---
title: "Deconstructing Unity Android Apps"
date: 2023-02-09T00:00:00-00:00
lastmod: 2023-02-09T00:00:00-00:00
tags: ['unity', 'android']
description: "Analyzing a deconstructed Unity Android application."
draft: false
# cover:
#     image: "<image path/url>" # image path/url
#     alt: "<alt text>" # alt text
#     caption: "<text>" # display caption under cover
#     relative: false # when using page bundles set this to true
#     hidden: true # only hide on current single page
---

Recently, I was interested in comparing two Android games built with Unity. I was particularly interested in analyzing 
how much font data each game included in the package submitted to the Google Play Store. A basic analysis of included 
files and their sizes can be accomplished with the following tools.

- adb
  - `adb shell pm list` - Find the app ID of the target by listing all installed apps
  - `adb shell pm path` - Find the unique install path of the provided app ID
  - `adb pull` - Transfer the app files to your computer for further analysis
- zipinfo - Inspect the content and compressed file sizes of a `.zip` archive
- apktool - Unpack an Android application package (`.akp`) for further analysis
- AssetRipper - A tool for extracting assets from serialized files (`CAB-*`, `*.assets`, `*.sharedAssets`, etc.) and 
  assets bundles (*.unity3d, *.bundle, etc.) and converting them into the native Unity engine format.

For this post, we are going to take a look at Marvel Snap. The Snap download from the Play Store is under 150mb; 
however, it does appear to download a small amount of data when launched and more data as you progress. Developers may 
desire their submitted app package to be under 150mb because it can be downloaded on a mobile data plan without 
prompting the user about the download size. The Apple App Store has a similar limit of 200mb.

The analysis conducted for this post was performed on Windows with WSL2, Ubuntu, and ZSH. You may already have an 
Android development environment setup with `adb` available in your PATH variable. If not, here are some [general 
instructions](https://www.xda-developers.com/install-adb-windows-macos-linux/) for installing `adb` on Ubuntu. The 
following sections go into more detail about setting up `adb` for use with WSL2 and connecting an Android device.

## Enable USB Debugging on Android device

First, we enable developer mode on our Android device. The steps for doing this can vary by version of Android or by the 
manufacturer. Check out the [Android developer docs](https://developer.android.com/studio/debug/dev-options) for 
instructions on enabling developer mode and USB debugging.

## Setting Up ADB on WSL2
To perform any analysis, it is helpful to copy the app files to your computer. This is done with the Android Debug 
Bridge (`adb`). Getting `adb` to connect to an Android device on WSL2 requires some extra steps. If we were to run `adb 
devices` without additional WSL2 setup, we should see the following output and notice that no devices are listed.

~~~
❯ adb devices
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
~~~

We need to install ADB on both Windows and WSL2. Here is how we install ADB on Windows:

1. Download the [Android SDK Platform Tools ZIP for 
   Windows](https://dl.google.com/android/repository/platform-tools-latest-windows.zip) 
1. Extract this to a location such as `%USERPROFILE%\AppData\Local\Android\platform-tools`
1. Add the above path to the `Path` environment variable
1. With our Android device connected and USB debugging enabled, ADB may not yet recognize your device. If we run `adb 
   devices` in Powershell and it does not list any devices, try running the following before checking for the device 
   again: 

   ~~~
   adb kill-server
   adb start-server
   adb devices
   ~~~

   If we disconnect the Android device from USB and reconnect it we should be prompted on the device to allow the
   Windows machine to connect. Further troubleshooting will be required if `adb devices` does not list the device as 
   seen below.

   ~~~
   PS C:\Users\heath> adb devices
   List of devices attached
   1A021FDEE0088H  device
   ~~~

1. In Powershell run `adb tcpip 5555`
1. In WSL2 run `adb connect <IP>:5555` where `<IP>` is the local network IP address of your Android device
1. The output might contain something like "failed to authenticate to <IP>:5555" but there may have another prompt to 
   accept on the Android device. Run the `adb connect` command again and it should output something like "already 
   connected to <IP>:5555"
1. In WSL2 confirm that `adb devices` now lists the Android device. 

   ~~~
   ❯ adb devices
   List of devices attached
   <IP>:5555      device
   ~~~

## Transfer the App files

First, we need to know the app identifier for the target app. In my case, the target app is Marvel Snap. We use `adb 
shell pm list packages` to list the app identifiers of all installed apps on the device. Hopefully, your target app has 
an identifier you can recognize. The output of the command will look like this:

~~~
❯ adb shell pm list packages
package:com.google.audio.hearing.visualization.accessibility.scribe
package:com.android.systemui.auto_generated_rro_vendor__
package:com.google.android.providers.media.module
package:com.google.android.overlay.permissioncontroller
package:com.google.android.overlay.googlewebview
package:com.android.calllogbackup
<Truncated...>
~~~

Marvel Snap will appear in the list as `package:com.nvsgames.snap`. Next, we need to discover the path on the device to 
the installed app. The path is lightly obfuscated and will change each time you install the app. Now we use `adb shell 
pm path com.nvsgames.snap` to discover the path we need.

~~~
❯ adb shell pm path com.nvsgames.snap
package:/data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/base.apk
package:/data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/split_config.arm64_v8a.apk
~~~

We can see above that the path we need for the next step is 
`/data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/`. Finally, we can copy the files to 
the computer where the device is connected. Change to the directory where we want to copy files to and run `adb pull 
<path>` where `<path>` is the app path we found above. The command output will update as it copies files but after it 
has completed it will look similar to below:

~~~
❯ adb pull /data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/
adb: warning: skipping special file '/data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/oat' (mode = 0o0)
/data/app/~~3ptHVzClzNmwSowM3hAu0A==/com.nvsgames.snap-kVFALO2K7yXXBtqhMI35dw==/: 35 files pulled, 1 skipped. 18.8 MB/s (256557359 bytes in 13.032s)
~~~

## Analyzing the App files

Great! The files are on our computer and ready for further analysis. This post is by no means an extensive analysis 
guide but next, I'll point out a few things related to the sizes of different sections of the app. We will also take a 
look at deconstructing parts of the Unity app to see its contents.

Before we get too far, let's take a quick look at the contents of the files we transferred. For Snap, it should look 
like the following:

~~~
❯ tree
.
├── base.apk
├── base.dm
├── lib
│   └── arm64
│       ├── libalog.so
│       ├── libbacktrace-native.so
│       ├── libbreakpad_client.so
│       ├── lib_burst_generated.so
│       ├── libbytehook.so
│       ├── libcpuinfo.so
│       ├── libcrashpad_handler.so
│       ├── libc++_shared.so
│       ├── libEncryptor.so
│       ├── libgp_base.so
│       ├── libgpm.so
│       ├── libgp.so
│       ├── libgsdk_base.so
│       ├── libil2cpp.so
│       ├── libkeva.so
│       ├── libmain.so
│       ├── libmonitorcollector-lib.so
│       ├── libnative-lib.so
│       ├── libnpth_dl.so
│       ├── libnpth_dumper.so
│       ├── libnpth_heap_tracker.so
│       ├── libnpth_logcat.so
│       ├── libnpth.so
│       ├── libnpth_tools.so
│       ├── libnpth_wrapper.so
│       ├── libnpth_xasan.so
│       ├── libping.so
│       ├── libsscronet.so
│       ├── libttboringssl.so
│       ├── libttcrypto.so
│       ├── libttgame_tps_client.so
│       └── libunity.so
└── split_config.arm64_v8a.apk
~~~

Snap is a relatively simple package. Everything fits in the base APK and no Android Asset Packs are used to provide 
additional data. There is a limit to how large the base APK can be before asset packs must be introduced. Asset packs 
themselves have size and quantity limits. To learn more about asset packs, check out the [Play Asset Delivery 
documentation](https://developer.android.com/guide/playcore/asset-delivery). An additional point of interest is that if 
an App Store download is under 150mb, it can be downloaded on a mobile data plan without prompting the user. It's 
assumed that more users will download your app if they are not concerned about it using a significant portion of a 
user's monthly data cap.

When an app is submitted to the Play Store, it will be processed by Google. The base APK will have binaries split into 
additional APKs for each supported target architecture. We can see in the output above that the binary APK is called 
`split_config.arm64_v8a.apk`.

Additionally, the binary APK contains compressed binaries that need to be extracted before the app can run. During app 
installation, the binaries are extracted to the `lib/` directory as seen in the output above.

Analyzing the binaries may give insight into how the app is structured or what third-party libraries the app is using. 
Snap is a Unity app so it's no surprise that we see `libunity.so` and `libil2cpp.so`. As an arbitrary example, we can 
see that Snap is using [Backtrace](https://backtrace.io/) for crash and error reporting based on the presence of 
`libbacktrace-native.so` and `libcrashpad_handler.so`.

### Analyzing Binary Sizes

We can easily look at the binaries in `lib/` to understand the extracted size of the files; however, I am more 
interested in the compressed size of binaries as it relates to App Store size limits and the 150mb threshold. We need to 
have a look inside the binary APK and to do this we will use the `zipinfo` command. If you don't have `zipinfo` 
installed, you can install it on WSL2 with `sudo adb install zip unzip`. We could simply look at the size of the binary 
APK to understand the size of all binaries but I'm interested in looking at the size of individual binaries.

Let's take a look by running `zipinfo -lt split_config.arm64_v8a.apk`

~~~
❯ zipinfo -lt split_config.arm64_v8a.apk
Archive:  split_config.arm64_v8a.apk
Zip file size: 40147658 bytes, number of entries: 37
-rw-r--r--  0.0 unx      952 b-      419 defN 81-Jan-01 01:01 AndroidManifest.xml
-rw-r--r--  0.0 unx    75744 b-    25809 defN 81-Jan-01 01:01 lib/arm64-v8a/libEncryptor.so
-rw-r--r--  0.0 unx   361000 b-   158184 defN 81-Jan-01 01:01 lib/arm64-v8a/lib_burst_generated.so
-rw-r--r--  0.0 unx   121208 b-    66067 defN 81-Jan-01 01:01 lib/arm64-v8a/libalog.so
-rw-r--r--  0.0 unx  1485240 b-   464514 defN 81-Jan-01 01:01 lib/arm64-v8a/libbacktrace-native.so
-rw-r--r--  0.0 unx   182328 b-    69734 defN 81-Jan-01 01:01 lib/arm64-v8a/libbreakpad_client.so
-rw-r--r--  0.0 unx    43536 b-    20925 defN 81-Jan-01 01:01 lib/arm64-v8a/libbytehook.so
-rw-r--r--  0.0 unx   911696 b-   272821 defN 81-Jan-01 01:01 lib/arm64-v8a/libc++_shared.so
-rw-r--r--  0.0 unx    27456 b-    10914 defN 81-Jan-01 01:01 lib/arm64-v8a/libcpuinfo.so
-rw-r--r--  0.0 unx  2917440 b-  1344878 defN 81-Jan-01 01:01 lib/arm64-v8a/libcrashpad_handler.so
-rw-r--r--  0.0 unx  3395632 b-  1241680 defN 81-Jan-01 01:01 lib/arm64-v8a/libgp.so
-rw-r--r--  0.0 unx   519264 b-   207392 defN 81-Jan-01 01:01 lib/arm64-v8a/libgp_base.so
-rw-r--r--  0.0 unx   977104 b-   310883 defN 81-Jan-01 01:01 lib/arm64-v8a/libgpm.so
-rw-r--r--  0.0 unx    10312 b-     3595 defN 81-Jan-01 01:01 lib/arm64-v8a/libgsdk_base.so
-rw-r--r--  0.0 unx 85567976 b- 24113775 defN 81-Jan-01 01:01 lib/arm64-v8a/libil2cpp.so
-rw-r--r--  0.0 unx   203368 b-    67908 defN 81-Jan-01 01:01 lib/arm64-v8a/libkeva.so
-rw-r--r--  0.0 unx    10416 b-     2629 defN 81-Jan-01 01:01 lib/arm64-v8a/libmain.so
-rw-r--r--  0.0 unx   243992 b-    87711 defN 81-Jan-01 01:01 lib/arm64-v8a/libmonitorcollector-lib.so
-rw-r--r--  0.0 unx   223288 b-    76642 defN 81-Jan-01 01:01 lib/arm64-v8a/libnative-lib.so
-rw-r--r--  0.0 unx    76616 b-    37492 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth.so
-rw-r--r--  0.0 unx    26560 b-    11422 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_dl.so
-rw-r--r--  0.0 unx   121704 b-    63883 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_dumper.so
-rw-r--r--  0.0 unx    59344 b-    27079 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_heap_tracker.so
-rw-r--r--  0.0 unx    27000 b-    11889 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_logcat.so
-rw-r--r--  0.0 unx    83896 b-    35318 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_tools.so
-rw-r--r--  0.0 unx     6048 b-     1425 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_wrapper.so
-rw-r--r--  0.0 unx    42920 b-    17910 defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_xasan.so
-rw-r--r--  0.0 unx    14144 b-     4765 defN 81-Jan-01 01:01 lib/arm64-v8a/libping.so
-rw-r--r--  0.0 unx  4428584 b-  2029259 defN 81-Jan-01 01:01 lib/arm64-v8a/libsscronet.so
-rw-r--r--  0.0 unx   341976 b-   142671 defN 81-Jan-01 01:01 lib/arm64-v8a/libttboringssl.so
-rw-r--r--  0.0 unx  1170688 b-   466898 defN 81-Jan-01 01:01 lib/arm64-v8a/libttcrypto.so
-rw-r--r--  0.0 unx   530640 b-   248325 defN 81-Jan-01 01:01 lib/arm64-v8a/libttgame_tps_client.so
-rw-r--r--  0.0 unx 20348216 b-  8486117 defN 81-Jan-01 01:01 lib/arm64-v8a/libunity.so
-rw----     2.0 fat       32 b-       35 defN 81-Jan-01 01:01 stamp-cert-sha256
-rw----     2.0 fat     2912 b-     1200 defN 81-Jan-01 01:01 META-INF/BNDLTOOL.SF
-rw----     2.0 fat     1405 b-     1067 defN 81-Jan-01 01:01 META-INF/BNDLTOOL.RSA
-rw----     2.0 fat     2804 b-     1125 defN 81-Jan-01 01:01 META-INF/MANIFEST.MF
37 files, 124563441 bytes uncompressed, 40134360 bytes compressed:  67.8%
~~~

We can see in the last line the total uncompressed and compressed size of all the files. We also see a compression 
ratio of 67.8%. For each file, we can see the compressed and uncompressed size in bytes of each file. If you would 
rather see the compression ratio as a percent instead of size in bytes, you can change the arguments we used and run 
`zipinfo -mt split_config.arm64_v8a.apk`

~~~
❯ zipinfo -mt split_config.arm64_v8a.apk
Archive:  split_config.arm64_v8a.apk
Zip file size: 40147658 bytes, number of entries: 37
-rw-r--r--  0.0 unx      952 b- 56% defN 81-Jan-01 01:01 AndroidManifest.xml
-rw-r--r--  0.0 unx    75744 b- 66% defN 81-Jan-01 01:01 lib/arm64-v8a/libEncryptor.so
-rw-r--r--  0.0 unx   361000 b- 56% defN 81-Jan-01 01:01 lib/arm64-v8a/lib_burst_generated.so
-rw-r--r--  0.0 unx   121208 b- 46% defN 81-Jan-01 01:01 lib/arm64-v8a/libalog.so
-rw-r--r--  0.0 unx  1485240 b- 69% defN 81-Jan-01 01:01 lib/arm64-v8a/libbacktrace-native.so
-rw-r--r--  0.0 unx   182328 b- 62% defN 81-Jan-01 01:01 lib/arm64-v8a/libbreakpad_client.so
-rw-r--r--  0.0 unx    43536 b- 52% defN 81-Jan-01 01:01 lib/arm64-v8a/libbytehook.so
-rw-r--r--  0.0 unx   911696 b- 70% defN 81-Jan-01 01:01 lib/arm64-v8a/libc++_shared.so
-rw-r--r--  0.0 unx    27456 b- 60% defN 81-Jan-01 01:01 lib/arm64-v8a/libcpuinfo.so
-rw-r--r--  0.0 unx  2917440 b- 54% defN 81-Jan-01 01:01 lib/arm64-v8a/libcrashpad_handler.so
-rw-r--r--  0.0 unx  3395632 b- 63% defN 81-Jan-01 01:01 lib/arm64-v8a/libgp.so
-rw-r--r--  0.0 unx   519264 b- 60% defN 81-Jan-01 01:01 lib/arm64-v8a/libgp_base.so
-rw-r--r--  0.0 unx   977104 b- 68% defN 81-Jan-01 01:01 lib/arm64-v8a/libgpm.so
-rw-r--r--  0.0 unx    10312 b- 65% defN 81-Jan-01 01:01 lib/arm64-v8a/libgsdk_base.so
-rw-r--r--  0.0 unx 85567976 b- 72% defN 81-Jan-01 01:01 lib/arm64-v8a/libil2cpp.so
-rw-r--r--  0.0 unx   203368 b- 67% defN 81-Jan-01 01:01 lib/arm64-v8a/libkeva.so
-rw-r--r--  0.0 unx    10416 b- 75% defN 81-Jan-01 01:01 lib/arm64-v8a/libmain.so
-rw-r--r--  0.0 unx   243992 b- 64% defN 81-Jan-01 01:01 lib/arm64-v8a/libmonitorcollector-lib.so
-rw-r--r--  0.0 unx   223288 b- 66% defN 81-Jan-01 01:01 lib/arm64-v8a/libnative-lib.so
-rw-r--r--  0.0 unx    76616 b- 51% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth.so
-rw-r--r--  0.0 unx    26560 b- 57% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_dl.so
-rw-r--r--  0.0 unx   121704 b- 48% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_dumper.so
-rw-r--r--  0.0 unx    59344 b- 54% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_heap_tracker.so
-rw-r--r--  0.0 unx    27000 b- 56% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_logcat.so
-rw-r--r--  0.0 unx    83896 b- 58% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_tools.so
-rw-r--r--  0.0 unx     6048 b- 76% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_wrapper.so
-rw-r--r--  0.0 unx    42920 b- 58% defN 81-Jan-01 01:01 lib/arm64-v8a/libnpth_xasan.so
-rw-r--r--  0.0 unx    14144 b- 66% defN 81-Jan-01 01:01 lib/arm64-v8a/libping.so
-rw-r--r--  0.0 unx  4428584 b- 54% defN 81-Jan-01 01:01 lib/arm64-v8a/libsscronet.so
-rw-r--r--  0.0 unx   341976 b- 58% defN 81-Jan-01 01:01 lib/arm64-v8a/libttboringssl.so
-rw-r--r--  0.0 unx  1170688 b- 60% defN 81-Jan-01 01:01 lib/arm64-v8a/libttcrypto.so
-rw-r--r--  0.0 unx   530640 b- 53% defN 81-Jan-01 01:01 lib/arm64-v8a/libttgame_tps_client.so
-rw-r--r--  0.0 unx 20348216 b- 58% defN 81-Jan-01 01:01 lib/arm64-v8a/libunity.so
-rw----     2.0 fat       32 b- -8% defN 81-Jan-01 01:01 stamp-cert-sha256
-rw----     2.0 fat     2912 b- 59% defN 81-Jan-01 01:01 META-INF/BNDLTOOL.SF
-rw----     2.0 fat     1405 b- 24% defN 81-Jan-01 01:01 META-INF/BNDLTOOL.RSA
-rw----     2.0 fat     2804 b- 60% defN 81-Jan-01 01:01 META-INF/MANIFEST.MF
37 files, 124563441 bytes uncompressed, 40134360 bytes compressed:  67.8%
~~~

A quick observation is that the Unity app binaries are large. I also checked apps like Clash Royale and 
Brawl Stars. Their binary sizes were significantly smaller. The Unity engine alone can be as large as the code base for 
entire games.

### Analyzing Embedded Asset Bundles

Using [apktool](https://ibotpeaches.github.io/Apktool/) we can unpack `base.apk`. `apktool` can be installed with `sudo 
apt install apktool`. It's worth noting that an APK is essentially a zip archive. You can extract it like a regular zip 
archive. Hopefully, this was apparent when we used `zipinfo` to look inside the binary APK. However, simply unzipping 
the APK will leave parts like the `AndroidManifest.xml` unreadable.

Unpack the base APK with `apktool d base.apk`.

~~~
❯ apktool d base.apk
I: Using Apktool 2.5.0-dirty on base.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/hfarrow/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Baksmaling classes2.dex...
I: Baksmaling classes3.dex...
I: Baksmaling classes4.dex...
I: Baksmaling classes5.dex...
I: Baksmaling classes6.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
I: Copying META-INF/services directory
~~~

We should now see a directory named `base`. Here is a look at the top-level directory listing:

~~~
❯ tree -L 1
.
├── AndroidManifest.xml
├── apktool.yml
├── assets
├── kotlin
├── META-INF
├── original
├── res
├── smali
├── smali_classes2
├── smali_classes3
├── smali_classes4
├── smali_classes5
├── smali_classes6
└── unknown
~~~

Notice that if you open `AndroidManifest.xml` it will be readable. We aren't interested in the manifest for the purpose 
of this post but it's useful to know about. Even when troubleshooting your own Unity app, it can be useful to view the 
manifest in your built APK to verify the final manifest is what you expected.

We are interested in app size so let's take a high-level look at the distribution of large files in the APK. Another 
handy tool I used in my terminal is [broot](https://github.com/Canop/broot). It has "whale" mode that will sort and 
display files by size. Install `broot` with `sudo apt install broot` and then run `broot -m` in the unpacked APK 
directory.

![](broot-base-whale.png)

For our purposes, we are interested in the `assets` directory where we will find the contents of the Unity project's 
`Assets/StreamingAssets/`, and all the assets that were embedded as part of scenes and `Resources/` directories. Let's 
move `broot` into the `assets` directory and take another look.

![](broot-assets-whale.png)

From inspecting the contents of `aa` we can determine that Snap uses the Addressable system and that the Android 
directory contains asset bundles embedded into the app.

![](broot-bundles-whale.png)

When you embed asset bundles into a Unity APK, you have the decision to make. Do you compress the asset bundle in the 
APK archive or do you leave them uncompressed? Compressing the bundles is in addition to the compression setting used to 
build the individual bundles. That is to say, you can double-compress asset bundles. The contents of a bundle itself can 
be compressed and the section of the APK archive containing the bundle can also be compressed. 

Compressing bundles in the APK will help reduce the size of your APK but it comes with a significant trade-off. At 
runtime, loading an asset bundle out of the APK will be significantly slower because Unity will have to extract the file 
out of the APK to read it into memory. As an example, in a game I worked on where we pre-loaded many bundles, 
compressing the bundles increased app startup time by at least a multiple of 4 on decent hardware. You can improve this 
experience by extracting the asset bundles and caching them outside of the APK. Unity has a built-in asset bundle cache 
and if you load bundles through a 
[UnityWebRequest](https://docs.unity3d.com/ScriptReference/Networking.UnityWebRequest.html), I believe it will 
automatically perform this step for you. There is still a trade-off that the first time you load a bundle, you need to 
extract it. The app should be tested to determine how this affects the user experience and if the trade-off is 
acceptable. Having longer load times on a player's first session might impact their impression negatively. No one likes 
long loading phases.

So, are Snap bundles compressed in the APK or not? We can go back to `zipinfo` to analyze the `base.apk`. `zipinfo` can 
take a trailing argument that is a file filter for the output. Try running `zipinfo -m base.apk assets/aa/Android/\*` to 
only output info about the asset bundles. Below we can see the first few lines of output but all the bundles are using 
the same compression strategy.

~~~
❯ zipinfo -m base.apk assets/aa/Android/\*
-rw-r--r--  0.0 unx     3566 b- 19% defN 81-Jan-01 01:01 assets/aa/Android/0ea51b79af36d03ba15ac4b5ea3be170.bundle
-rw-r--r--  0.0 unx     3547 b- 19% defN 81-Jan-01 01:01 assets/aa/Android/4ed2cdfb0e9393601d4ffe68e478f867.bundle
-rw-r--r--  0.0 unx     4850 b- 23% defN 81-Jan-01 01:01 assets/aa/Android/515070c0151e19d075d62b181d0d1c29.bundle
-rw-r--r--  0.0 unx     9698 b- 20% defN 81-Jan-01 01:01 assets/aa/Android/5e4bad4874a8ef52921df1b8d1188b7c.bundle
-rw-r--r--  0.0 unx    20358 b- 18% defN 81-Jan-01 01:01 assets/aa/Android/6c14c35b58746c8605fc410e435b38b5.bundle
-rw-r--r--  0.0 unx    70554 b- 40% defN 81-Jan-01 01:01 assets/aa/Android/6e507c8d892466c20bdc6e65eeff6f19.bundle
-rw-r--r--  0.0 unx     4716 b- 20% defN 81-Jan-01 01:01 assets/aa/Android/87b3b19c363616f26590d1191c153135.bundle
-rw-r--r--  0.0 unx     8920 b- 18% defN 81-Jan-01 01:01 assets/aa/Android/8965793a594f49ff7b8283b12bdab12b.bundle
-rw-r--r--  0.0 unx    14902 b- 18% defN 81-Jan-01 01:01 assets/aa/Android/93d9c5e9586500293f5fe79095052b54.bundle
-rw-r--r--  0.0 unx   110994 b- 17% defN 81-Jan-01 01:01 assets/aa/Android/b0d5d1c80db4e7ada5b102f136b05c06.bundle
-rw-r--r--  0.0 unx  1844795 b-  0% defN 81-Jan-01 01:01 assets/aa/Android/baked-smalllogos_assets_all_a01de1e78f3277d29d3895934ef45348.bundle
-rw-r--r--  0.0 unx  2239676 b- 15% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-card_assets_all_72d770ac4c8ab54515d4be59bf32c3eb.bundle
-rw-r--r--  0.0 unx     8582 b- 22% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_applesdf_3f50ebe88e074a9767830cc1fb5c5f24.bundle
-rw-r--r--  0.0 unx     8103 b- 21% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_contentsdf_0e9b61d8eb8d950cf81900554acab2a1.bundle
-rw-r--r--  0.0 unx     8596 b- 22% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_googlesdf_2263bf4501d15f601a859e5d86fafa67.bundle
-rw-r--r--  0.0 unx    22919 b- 21% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_header_891ee8005509ee8884e463deb3b810f2.bundle
-rw-r--r--  0.0 unx     8158 b- 21% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_headersdf_cc807b1a807e72e2f7bb8d9ef97eb82a.bundle
-rw-r--r--  0.0 unx   353188 b- 11% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_helveticaneueltstd-mdcnsdf_a49787f3b1b5550c636345c5aaa010e9.bundle
-rw-r--r--  0.0 unx    29498 b- 29% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_tmp_sdf-mobile_590e52215c1985ca7d514bcd1eee9c5b.bundle
-rw-r--r--  0.0 unx    31015 b- 29% defN 81-Jan-01 01:01 assets/aa/Android/bundledinapp-font_assets_tmp_sdf-mobilemasking_f14fca07d81ca13d20379d31d61ecd97.bundle
<Truncated...>
~~~

At a glance, we can confirm the asset bundles are compressed. The `defN` value in the 7th column confirms this but so 
does the fact that we see compression ratios in the 6th column. Interestingly and assuming this was a deliberate choice, 
this may have helped the developers keep their APK under the Play Store 150mb limit I mentioned previously. I won't go 
so far for this post but one could play the app and inspect the app data to see if the embedded bundles are being cached 
outside of the APK.

The compression ratio varies widely among the bundles. We can see values ranging from single digits up to 
59%. If you run `zipinfo` again with the `-t` flag added, the final line of output will include the total compression 
ratio of all files. We see that the overall compression ratio is 10.5%.

~~~
❯ zipinfo -mt base.apk assets/aa/Android/\*
<Truncated...>
235 files, 65842186 bytes uncompressed, 58926405 bytes compressed:  10.5%
~~~

### Analyzing Asset Bundles Contents

In the previous steps, we already extracted all asset bundles when we unpacked the `base.apk` and they can be found in 
`./base/assets/aa/Android/`. We can create an empty Unity project and install the [Asset Bundle 
Browser](https://docs.unity3d.com/Manual/AssetBundles-Browser.html) tool package. The link contains installation 
instructions and a link to the tool's documentation. With this tool, we can add the `assets/aa/Android/` directory and 
inspect all the bundles together. We can collect a lot of information about the bundles doing this but I've chosen to 
skip doing so for this post.

![](asset-bundle-browser.png)

### Analyzing Embedded Scenes and Resources

We already looked at asset bundles but what about all the assets referenced by embedded scenes and assets found inside 
`Resources/` directories in the project's `Assets/` directory? All of that data can be found in 
`base/assets/bin/Data/data.unity3d`. 

I may come back in the future and expand this section further but for now, I'll just point out that a tool called [Asset 
Ripper](https://assetripper.github.io/AssetRipper/) that can both inspect the contents of this file and export a Unity 
project containing much of the contents. For a Standalone player using the mono backend you can decompile scripts but 
for an Android or iOS game using the IL2CPP backend that will not be possible. Asset Ripper is an insightful tool for 
looking at the contents and structure of the project. For my purposes, I was interested in looking at how much font 
data was embedded in the APK and I accomplished that goal with this tool.

Asset Ripper can also be used to inspect and unpack individual assets bundles; although, I didn't go that far for my 
purposes as the Asset Bundle Browser was sufficient and more convenient to use.

There is a lot more to be said about deconstructing, reverse engineering, and analyzing an Android app but I think this 
serves as a good introduction to the topic.
