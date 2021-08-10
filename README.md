# 编译jitsi-meet  ios sdk-- 发布公有库-- 集成flutter插件-- 使用
## 环境配置
> node 环境

 node >12.0,npm > 6.0

使用nvm管理node版本


首先 ```nvm ls```检查本机node版本，

如果本机node版本不合适，使用```nvm use v2.12.0```将node环境切换到12以上。如果本地没有，使用```nvm ls-remote```

选择12.0以上版本，执行```nvm install v15.0.0```就可以安装v15.0.0这个版本了。

> 安装 ```install-local```

执行```npm i -g install-local```

## 运行工程
1、```cd ~/.../lib-jitsi-meet```目录下，执行```npm update```,然后执行```npm install```将依赖安装到本地

2、修改jitsi-meet的package.json文件，将lib-jitsi-meet的依赖改为本地依赖```"lib-jitsi-meet": "file:../lib-jitsi-meet",```

3、删除jitsi-meet/node_moudles/lib-jitsi-meet文件

4、cd 到 jitsi-meet目录下执行```npm install```安装所需依赖

4、在jitsi-meet目录下执行：```install-local ../lib-jitsi-meet```，安装本地依赖

5、运行：```react-native start```启动react-native

6、运行 ```react-native run```运行项目，或者使用Xcode运行项目

## 编译SDK
当项目运行通过后，如果可以正常使用，则可以通过下面脚本来构建iOS sdk
```
mkdir -p ios/sdk/out
xcodebuild clean \
    -workspace ios/jitsi-meet.xcworkspace \
    -scheme JitsiMeetSDK
xcodebuild archive \
    -workspace ios/jitsi-meet.xcworkspace \
    -scheme JitsiMeetSDK  \
    -configuration Release \
    -sdk iphonesimulator \
    -destination='generic/platform=iOS Simulator' \
    -archivePath ios/sdk/out/ios-simulator \
    VALID_ARCHS=x86_64 \
    ENABLE_BITCODE=NO \
    SKIP_INSTALL=NO \
    BUILD_LIBRARY_FOR_DISTRIBUTION=YES
xcodebuild archive \
    -workspace ios/jitsi-meet.xcworkspace \
    -scheme JitsiMeetSDK  \
    -configuration Release \
    -sdk iphoneos \
    -destination='generic/platform=iOS' \
    -archivePath ios/sdk/out/ios-device \
    VALID_ARCHS=arm64 \
    ENABLE_BITCODE=NO \
    SKIP_INSTALL=NO \
    BUILD_LIBRARY_FOR_DISTRIBUTION=YES
xcodebuild -create-xcframework \
    -framework ios/sdk/out/ios-device.xcarchive/Products/Library/Frameworks/JitsiMeetSDK.framework \
    -framework ios/sdk/out/ios-simulator.xcarchive/Products/Library/Frameworks/JitsiMeetSDK.framework \
    -output ios/sdk/out/JitsiMeetSDK.xcframework
cp -a node_modules/react-native-webrtc/apple/WebRTC.xcframework ios/sdk/out
```


## 发布自有CocoaPods 库
> 注册Cocoapods用户
参考文档：https://guides.cocoapods.org/making/getting-setup-with-trunk.html

1、查看是否注册了Cocoapods

```pod trunk me```

2、注册

```pod trunk register emailAddress 'name'```

修改原有sdk 的 podspec文件

将原来的podspec文件名：JitsiMeetSDK修改为自己的文件名 HyJitsiMeetSDK，

修改HyJitsiMeetSDK 文件的依赖关系 s.version 显示的是我的库的版本号

修改s.source 为我自己sdk的仓库，对应的tag与远端tag要相符合

```
Pod::Spec.new do |s|
  s.name             = 'HyJitsiMeetSDK'
  s.version          = '1.6.0'
  s.summary          = 'Jitsi Meet iOS SDK'
  s.description      = 'Jitsi Meet is a WebRTC compatible, free and Open Source video conferencing system that provides browsers and mobile applications with Real Time Communications capabilities.'
  s.homepage         = 'https://github.com/lylcsaa/jitsi-meet-sdk.git'
  s.license          = 'Apache 2'
  s.authors          = 'The Jitsi Meet project authors'
  s.source           = { :git => 'https://github.com/lylcsaa/jitsi-meet-sdk.git', :tag => s.version }

  s.platform         = :ios, '11.0'

  s.vendored_frameworks = 'Frameworks/JitsiMeetSDK.xcframework', 'Frameworks/WebRTC.xcframework'
  
  s.pod_target_xcconfig = { 'EXCLUDED_ARCHS[sdk=iphonesimulator*]' => 'arm64' }
  s.user_target_xcconfig = { 'EXCLUDED_ARCHS[sdk=iphonesimulator*]' => 'arm64' }
end
```

4、替换我自己编译的sdk

在```jitsi-meet/ios/sdk/out```目录下，将```JitsiMeetSDK.xcframework```、```WebRTC.xcframework```两个文件copy到我的sdk目录下，即：sdk/Frameworks/

5、上传我的库

将我的库上传到远端仓库，并打好tag，注意：tag要与 podspec文件中的tag一致。

6、发布我的公有库

```pod trunk push HyJitsiMeetSDK.podspec --allow-warnings```

发布成功后，我们注册的邮箱会推送一条消息，告诉我们已经发布了，但是此时如果我们执行```pod search HyJitsiMeetSDK```是搜不到的，这个时候，我们采取以下方案：

首先执行：```pod repo remove trunk ```移除trunk源

随便找一个iOS工程，依赖我发布的库，执行```pod install``` 会从新添加trunk源，但是可能仍然找不到，我们再执行```pod search HyJitsiMeetSDK``试试。

可以参考：【Cocoapod公有库踩坑】https://www.jianshu.com/p/99788e10a4b3

## 修改flutter 插件对sdk 的依赖

我们将插件远端仓库copy一份到我们自己的远程仓库，进入plugin项目中的/jitsi-meet/ios,编辑jitsi_meet.podspec文件。将s.dependency 'HyJitsiMeetSDK', '1.6.0'中的依赖修改为我们发布的cocoapod公有库即可。

## flutter 端集成

进入工程的 ```pubspec.yarm```文件，添加依赖

jitsi_meet:
    git:
      url: https://github.com/lylcsaa/jitsi_meet.git
      path: jitsi_meet
      ref: master 
      
## 运行flutter 工程

终端执行```flutter clean```

然后执行```flutter pub get```

最后在iOS目录下执行```pod install```

最后，在iOS设备上运行项目

至此，手动编译jitsi-meet项目sdk 并添加到flutter 依赖就成功了
