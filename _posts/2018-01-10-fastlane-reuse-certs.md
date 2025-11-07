---
layout: post
title: 关于fastlane已存在的证书复用问题
tags: fastlane
categories: iOS
date: 2018-01-10
---
### 前言
iOS开发在团队项目协作中，面临着许许多的挑战，除了被大家诟病的nib文件和故事板以外，还有就是今天要说的证书管理问题，相信做过iOS开发的用户对fastlane已经不陌生了，它提供了很多有用的功能来帮助开发者从繁琐的重复性劳动中解脱出来，这里列举出一些：
![](https://ws3.sinaimg.cn/large/006tNc79ly1fnbcmq6spkj30to0sbtag.jpg)
* deliver: 上传截图, 元数据, app应用程序到App Store
* supply: 上传Android app应用程序和元数据到Google Play
* snapshot: 自动捕获iOS app应用程序本地截图
* screengrab: 自动捕获Android app应用程序本地截图
* frameit: 快速截屏并将截屏放入设备中
* pem: 自动生成和更新推送通知配置文件
* sigh: 开发证书和描述文件下载
* produce: 使用命令行在iTunes Connect上创建新的app和开发入口
* cert: 自动创建和配置iOS代码签名证书
* spaceship: Ruby 库访问 Apple开发者中心和 iTunes Connect
* pilot: 最好的方式管理你的TestFlight 测试人员和从终端构建
* boarding: 最简单的方式邀请你的TestFlight beta测试人员
* gym: iOS app打包签名自动化工具
* match: 使用Git同步你的团队证书和配置文件
* scan: 最简单方式测试你的 iOS 和 Mac apps

今天说的其实是match，我们知道，苹果公司在个人开发者账号上面对于证书的生成是有严格的数量限制的，`development` 和 `distribution`证书类型只能生成2个，所以如果按照fastlane每次build不同的target或者不同的Bundle ID的话，它都会重新去生成一个新的证书并以此生成对应的描述文件，这样以来，我们也只能最多同时用该开发者账号签名两个App安装在真机上，想用第三个就必须revoke掉以前生成的证书，当然了，一旦把证书revoke掉了，这也就意味着我们用该证书签名的App也不能在真机上面使用了。所以就得考虑一下，该如果复用现有证书。
#### 1. 拿到你想要复用证书的ID
关于这个证书ID，从钥匙串和openssl工具库中没有找到方法来取到，但是可以通过spaceship这个库来实现，下面是相关脚本：
```
require 'spaceship'
Spaceship.login('your@apple.id')
Spaceship.select_team
Spaceship.certificate.all.each do |cert| 
  cert_type = Spaceship::Portal::Certificate::CERTIFICATE_TYPE_IDS[cert.type_display_id].to_s.split("::")[-1]
  puts "Cert id: #{cert.id}, name: #{cert.name}, expires: #{cert.expires.strftime("%Y-%m-%d")}, type: #{cert_type}"
end
```
执行上面代码，会输出所有证书的相应信息，你可以从中找到你想复用的那个证书的ID。
#### 2. 创建远程仓库来保存证书。
建立一个远程仓库，并在该目录下创建`certs/distribution`和 `certs/development`目录，分别存放生产和开发环境下的相关证书文件。
#### 3. 通过钥匙串导出你想要复用的那个证书
导出对应的cer文件和p12文件。
#### 4. 执行下面命令，导出私钥文件
```
openssl pkcs12 -nocerts -nodes -out key.pem -in certificate.p12
```
#### 5. 生成最后需要的证书
```
openssl aes-256-cbc -k <your_password> -in key.pem -out <cert_id>.p12 -a
openssl aes-256-cbc -k <your_password> -in certificate.cer -out <cert_id>.cer -a
```
这里的cert_id是上面我们保存的证书id，其中执行完上述步骤后，就生成了fastlane match想要的证书，当执行fastlane match development/adhoc/appstore命令后，match就不会在Apple Development Center重新生成证书了，而是用现有的。
将证书分别放到对应的git仓库目录中，提交并推送到远程仓库。
#### 6. 在开发者网站上面生成App ID
```
fastlane produce -u <your@apple.id> -a <your_app_bundle_id> --skip_itc
```
如果你的App需要在ITC（iTunes Connect）中创建，则移除`--skip_itc`选项。
#### 7. 生成证书对应的描述文件
```
fastlane match <type> 
```
其中type有四种：development/adhoc/distribution/appstore
如果执行过程中，出现输入Git Repo密码后，密码错误导致的不能解密repo，可以尝试着用`fastlane match change_password`来重置密码。如果修改密码后，发现还是不行的话，可以在与distribution同级目录下创建一个txt文件："match_version.txt"，内容为fastlane版本号即可，再重新执行。

> [22:57:23]: Cloning remote git repo...
> [22:57:28]: Migrating to new match...
> [22:57:28]: Enter the passphrase that should be used to encrypt/decrypt your certificates
> [22:57:28]: This passphrase is specific per repository and will be stored in your local keychain
> [22:57:28]: Make sure to remember the password, as you'll need it when you run match on a different machine
> Passphrase for Git Repo: ******
> Type passphrase again: ******
> [22:57:34]: 🔒 Successfully encrypted certificates repo
> [22:57:34]: Cloning remote git repo...
> [22:57:39]: Couldn't decrypt the repo, please make sure you enter the right password!
> version: 256
> class: "inet"
> ：：：：
> ：：：：

### 关于注册新设备

在这之前我们都是通过开发者中心来添加和管理更新设备以及描述文件，有了fastlane提供的match命令则可以帮助我们做这些事情。
**注册新设备**
**我们可以**通过添加action的方式**更新Fastfile文件：**

**直接添加设备**
```
register_devices(
  devices: {
    "Luka iPhone 6" => "1234567890123456789012345678901234567890",
    "Felix iPad Air 2" => "abcdefghijklmnopqrstvuwxyzabcdefghijklmn"
  }
) # Simply provide a list of devices as a Hash
```

**通过文件添加设备**
```
register_devices(
  devices_file: "./devices.txt"
) # Alternatively provide a standard UDID export .txt file, see the Apple Sample (https://devimages.apple.com/downloads/devices/Multiple-Upload-Samples.zip)
```

文件格式参考demo：[https://devimages.apple.com/downloads/devices/Multiple-Upload-Samples.zip](https://link.zhihu.com/?target=http%3A//devimages.apple.com/downloads/devices/Multiple-Upload-Samples.zip)

你也可以添加参数：
```
register_devices(
  devices_file: "./devices.txt", # You must pass in either `devices_file` or `devices`.
  team_id: "XXXXXXXXXX",         # Optional, if you"re a member of multiple teams, then you need to pass the team ID here.
  username: "luka@goonbee.com"   # Optional, lets you override the Apple Member Center username.
)
```
**更新描述文件**
```
match(type: "adhoc", force_for_new_devices: true)
```
注意这里的type，对应我们前面提到的几种类型，除此之外，我们也可以通过命令行的方式来更新描述文件：
```
fastlane match adhoc --force_for_new_devices
```
这样以来，fastlane会重新更新描述文件并提交到我们的证书仓库。
后面我们需要做的就是，只需重新打包，然后将包通过Airport安装到新的设备上就可以了，经测试以前用该描述文件打的包也可以安装到新设备上面去。

### 关于Apple ID开启双重验证

如果开启双重验证，默认苹果会在新设备登录时，需要手动输入验证码，这时候如果是在CI上面构建，就会带来问题，此时我们可以通过以下方式解决：
1. 访问[Apple ID网站](https://appleid.apple.com/),找到 安全 - App 专用密码，生成一个专用密码
2. 然后在构建服务器上面配置环境变量: vim ~/.bash_profile
```
export FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD=<YOUR_PASSWORD>
```
3. 执行 fastlane spaceauth -u <YOUR_APPLE_ID> 按提示获取session信息
4. 复制session信息（很长一大段） 配置环境变量: vim ~/.bash_profile
```
export FASTLANE_SESSION=‘YOUR SESSION’
```


### 参考
1. [Simplify your life with fastlane match](https://macoscope.com/blog/simplify-your-life-with-fastlane-match/)
2.  [register_devices - fastlane docs](https://docs.fastlane.tools/actions/register_devices/)
3.  [match - fastlane docs](https://docs.fastlane.tools/actions/match/)
