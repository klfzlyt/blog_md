---
title: react-native踩坑之旅
date: 2017-07-02 21:20:52
tags: react-native
---

2017-07-20更新:
在本地化的时候遇到点问题，如果没有本地话过的话，get key内部throw的错无法catch到

2017-07-09更新：
做了个TODO的小demo,用的内部的组内的redux业务框架，如下：
![](http://p0.meituan.net/dpgroup/a40c889447c718495475548592821db3926229.jpg)
总结下：
调试利器：[RN debuger](https://github.com/jhen0409/react-native-debugger/tree/master/npm-package)
在mac上开ios模拟器，拥有跟页面调试基本一致的体验。
初步发现的一些三方包：
1. <https://github.com/crazycodeboy/react-native-check-box> 
2. <https://github.com/sunnylqm/react-native-storage>
3. <https://github.com/sconxu/react-native-checkbox>
4. <https://github.com/FaridSafi/react-native-gifted-listview>
5. <https://github.com/jaysoo/react-native-parallax-scroll-view>
6. <https://github.com/ldn0x7dc/react-native-media-kit>
7. <https://github.com/tlenclos/react-native-audio-streaming>
8. <https://github.com/zmxv/react-native-sound>
9. <https://github.com/oblador/react-native-vector-icons>
10. <https://github.com/react-native-material-design/react-native-material-design>
11. <https://github.com/react-native-material-design/react-native-material-design>
12. <https://github.com/halilb/react-native-textinput-effects>


## 关于mac android指南
### 准备工作：(http://reactnative.cn/docs/0.45/getting-started.html#content)：
 1. 去https://developer.android.com/studio/install.html
 下载最新的android studio

android studio有去下载android sdk的能力，只要翻墙就可以用android studio去下载sdk

android studio的6.0必装，对应的是sdk 23
> 然后在Android 6.0 > (Marshmallow)中勾选Google > APIs、Android SDK Platform 23

装好后要设置ANDROID_HOME环境变量
再把工具路径
```js
export PATH=$PATH:$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools
```
加入环境变量

### 如何调试(真机)
我的调试机型 vivo android 5.1.1

用exop mac客户端可以调试
（这个客户端似乎不需要xcode 或者android sdk）用这个客户端建立的项目只有一个.expo文件夹，没有我们的ios,android文件夹

exop像是一个壳客户端,

--- 
在android 5.0以上，要执行
```js
adb reverse tcp:8081 tcp:8081
```
不然localhost认不出来
之后执行`react-native run-android`即可调试，
调试具有热启动的能力，远程debugger能力（不得不感叹很强大），下面是一些截图：

![](http://p1.meituan.net/dpgroup/9a2bf7836454fd7a16891061c83e079568682.jpg)


![](http://p1.meituan.net/dpgroup/2e46d2407fe6853a82a2187d05fd8d8d23932.jpg)
还有类似于看dom的功能
![](http://p1.meituan.net/dpgroup/81345c9a5627d294b390f4d21cc3df5d41824.jpg)

ps:
Genymotion自己装的有 但没有试过

### 之后是打包APK的过程
```js
$ keytool -genkey -v -keystore my-release-key.keystore -alias my-key-alias -keyalg RSA -keysize 2048 -validity 10000
```
用上面的命令生成一个keystore文件，
这个文件使用来签名用的，好像是个二进制文件，这个文件好像是java jdk的能力,引用：
> 这条命令会要求你输入密钥库（keystore）和对应密钥的密码，然后设置一些发行相关的信息。最后它会生成一个叫做my-release-key.keystore的密钥库文件。

> 在运行上面这条语句之后，密钥库里应该已经生成了一个单独的密钥，有效期为10000天。--alias参数后面的别名是你将来为应用签名时所需要用到的，所以记得记录这个别名。

还要生成一个Profile文件：
```js
编辑~/.gradle/gradle.properties（没有这个文件你就创建一个），添加如下的代码（注意把其中的****替换为相应密码）
注意：~表示用户目录，比如windows上可能是C:\Users\用户名，而mac上可能是/Users/用户名。

MYAPP_RELEASE_STORE_FILE=my-release-key.keystore
MYAPP_RELEASE_KEY_ALIAS=my-key-alias
MYAPP_RELEASE_STORE_PASSWORD=*****
MYAPP_RELEASE_KEY_PASSWORD=*****
```

之后把生成的文件拷贝到android/app(是app!)
下，不要拷贝到anroid文件夹下，
另外有2个build.gradle文件，要注意，复制的这段代码要去掉省略号，要复制到子build.gradle中，根build.gradle不用管

```js
...
android {
    ...
    defaultConfig { ... }
    signingConfigs {
        release {
            storeFile file(MYAPP_RELEASE_STORE_FILE)
            storePassword MYAPP_RELEASE_STORE_PASSWORD
            keyAlias MYAPP_RELEASE_KEY_ALIAS
            keyPassword MYAPP_RELEASE_KEY_PASSWORD
        }
    }
    buildTypes {
        release {
            ...
            signingConfig signingConfigs.release
        }
    }
}
...
```
之后在android文件中执行
```js
./gradlew assembleRelease
```
即可获得打包好的apk了


ignite:

https://github.com/infinitered/ignite

> ps:有些库要求不同版本的android build tool

android-building-from-source:

https://facebook.github.io/react-native/docs/android-building-from-source.html


如何run 0.38的官方android例子 ：(看样子是要build)
```js
Running the examples on Android

Note that you'll need the Android NDK installed, see prerequisites.

./gradlew :Examples:Movies:android:app:installDebug
# Start the packager in a separate shell (make sure you ran npm install):
./packager/packager.sh
# Open the Movies app in your emulator
```

### 下面是一些问题记录，有些没解决有些解决了
官方文档：(http://reactnative.cn/docs/0.45/upgrading.html)

mac上
1.最新的官方demo，要求xcode8.3.3
系统要求10.12及以上

<!--more-->
官方用的expo，这个工具我理解是提供了一个供应用运行的app容器。


那些示例项目android ios该怎么运行？`react-native run-ios/android`?

expo的应用怎么打包？目前都是运行在特定的expo那个应用上的

跑expo的应用在android上需要下个google play store，再搜索expo下，翻墙用的蓝灯，试了好多root软件都是，一键root

expo 的应用不能装apk么，都是内嵌到那个expo应用里的？

expo可以在ios模拟器跑，但是好像好慢的样子？
通过create-react-native-app AwesomeProject
 那个cli创建的项目，啥都没有ios android文件夹都没有,执行了yarn start之后 也产生了.expo文件
那么如何部署？
通过create-react-native-app 
出来的项目，执行`yarn start`得到一个二维码，用expo能扫开，很方便.
通过`yarn run android`连上真机的话，即为进行了真机debug,如果不连真机的话，得到一个这样的错
```js
Error running adb: No Android device found. Please connect a device and follow the instructions here to enable USB debugging:
https://developer.android.com/studio/run/device.html#developer-device-options. If you are using Genymotion go to Settings -> ADB, select "Use custom Android SDK tools", and point it at your Android SDK directory.
```
进行yarn run ios：会打开模拟器
```
$ react-native-scripts ios
00:32:00: Starting packager...
00:32:11: Starting simulator...
Found Xcode 7.3.0, which is older than the recommended Xcode 8.2.0.
```
我已经装了xcode 8.3.3为什么有这个提示？
而且文件结构永远没有ios,android文件，用的是expo
<!--more-->
***

如果这样初始项目
```js
react-native init AwesomeProject
```
会有ios android文件夹，npm 还会自动install?
执行`react-native run-ios`
得到build failed 结果，
**但是**通过xcode打开项目，进行run，却能正常打开模拟器，打开项目，通过命令就报错：
```js
** BUILD FAILED **


The following build commands failed:

	CompileC /Users/liyangtao/testProject/AwesomeProject/ios/build/Build/Intermediates/React.build/Debug-iphonesimulator/React.build/Objects-normal/x86_64/RCTTabBarItem.o Views/RCTTabBarItem.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler
(1 failure)

Installing build/Build/Products/Debug-iphonesimulator/AwesomeProject.app
An error was encountered processing the command (domain=NSPOSIXErrorDomain, code=2):
Failed to install the requested application
An application bundle was not found at the provided path.
Provide a valid path to the desired application bundle.
Print: Entry, ":CFBundleIdentifier", Does Not Exist

Command failed: /usr/libexec/PlistBuddy -c Print:CFBundleIdentifier build/Build/Products/Debug-iphonesimulator/AwesomeProject.app/Info.plist
Print: Entry, ":CFBundleIdentifier", Does Not Exist
```
```js
lsof -n -i4TCP:8081
kill - 9 xxxx
```
(issuse:https://github.com/facebook/react-native/issues/7308#issuecomment-219597774)
有可能跟react-native的版本有关
>从0.24版本开始，react-native还需要额外安装react模块，且对react的版本有严格要求，高于或低于某个范围都不可以。本文无法在这里列出所有react native和对应的react模块版本要求，只能提醒读者先尝试执行npm install，然后注意观察安装过程中的报错信息，例如require react@某.某.某版本, but none was installed，然后根据这样的提示，执行npm install react@某.某.某版本 --save。


还有貌似我在任何文件件都能执行`react-native`
如果`react-native run-android`那么
```js
Downloading https://services.gradle.org/distributions/gradle-2.14.1-all.zip
```
都会去下载一次，好慢，
```
> java.lang.UnsupportedClassVersionError: com/android/build/gradle/AppPlugin : Unsupported major.minor version 52.0
```
而且会有这个错误
解决办法是：
我的jdk是1.7的，升级为1.8就好了
(https://stackoverflow.com/questions/42874971/react-native-build-error-android-java-lang-unsupportedclassversionerror-com-a)
```js
up vote
6
down vote
accepted
At last figured out the problem

check $JAVA_HOME

Need JDK 1.8 to work

Install Java JDK 1.8 and change the JAVA_HOME

edit ~/.bashrc and add JDK 1.8 path as JAVA_HOME

export JAVA_HOME=/usr/lib/jvm/java-8-oracle/jre/

and source ~/.bashrc close the current terminal window/tab and run

react-native run-android
```
问题：
react-native-cli是啥？
local-cli是啥？
react-native-cli为什么要全局装？
react-native为什么要局部装？
装的版本有什么要求么
（遇到一个坑，在很老的项目中，react-native包含了react,之后分离了）
eject是个啥？
`react-native upgrade
`能干啥？
答： 
> 新版本的npm包通常还会包含一些动态生成的文件，这些文件是在运行react-native init创建新项目时生成的，比如iOS和Android的项目文件。为了使老项目的项目文件也能得到更新（不重新init），你需要在命令行中运行：

$ react-native upgrade
这一命令会检查最新的项目模板，然后进行如下操作：

如果是新添加的文件，则直接创建。
如果文件和当前版本的文件相同，则跳过。
如果文件和当前版本的文件不同，则会提示你一些选项：查看两者的不同，选择保留你的版本或是用新的模板覆盖。你可以按下h键来查看所有可以使用的命令。
译注：如果你有修改原生代码，那么在使用upgrade升级前，先备份，再覆盖。覆盖完成后，使用比对工具找出差异，将你之前修改的代码逐步搬运到新文件中。
```js
react-native init AwesomeProject
cd AwesomeProject
react-native run-ios
```
这样得到的应用是有ios文件么？

ps aux | grep react-native

不要随意关app，
调试可以删掉app重来

android无法naitive debug??

cp ./gradle.properties /Users/liyangtao/work/dux-rn/duxRnTest/android/app