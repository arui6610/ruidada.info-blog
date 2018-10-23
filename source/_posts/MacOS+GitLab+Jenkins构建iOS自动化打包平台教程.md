---
layout: post
title: "MacOS+GitLab+Jenkins构建iOS自动化打包平台教程"
date: 2017-12-03 23:45
categories: 
- 运动
- [运动, 球类运动]
tags: 
- 教程
- Jenkins
comments: true
toc: true
---
## 一、Jenkins

> Jenkins是一个Java项目，依赖于Java环境。在命令行中输入java -version，如果安装过就会出现对应的Java版本。没有则进入[JavaSDK官网](https://link.jianshu.com/?t=http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)下载安装。

### 1. 安装

> Jenkins安装方式有好几种，建议使用homebrew方式来进行安装Jenkins。因为用dmg方式安装的Jenkins默认会生成一个Shared的jenkins用户，且安装路径为/Users/xx/Shared。

<!-- more -->

#### 1.1 brew安装

> Homebrew是一款Mac OS平台下的软件包管理工具，如果没有安装过[homebrew](https://brew.sh/)，在命令行中输入如下命令安装：
>
> ```
> /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)
> ```
>
> 

- 安装jenkins

  ```
  brew install jenkins
  ```

- 卸载jenkins

  ```
  brew uninstall jenkins
  ```

#### 1.2 修改安装路径

- 安装完成自后，在命令行中cd到/Library/LaunchDaemons文件夹中

  ```
  cd /Library/LaunchDaemons
  ```

- 如果不存在org.jenkins-ci.plist文件，就手动创建一个

  ```
  sudo touch org.jenkins-ci.plist  
  ```

- 修改plist内容，JENKINS_HOME的值修改为你想要放置的安装路径即可

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
    <dict>
      <key>StandardOutPath</key>
      <string>/var/log/jenkins/jenkins.log</string>
      <key>StandardErrorPath</key>
      <string>/var/log/jenkins/jenkins.log</string>
      <key>EnvironmentVariables</key>
      <dict>
        <key>JENKINS_HOME</key>
        <string>/Users/ci/.jenkins</string>
      </dict>
      <key>GroupName</key>
      <string>daemon</string>
      <key>KeepAlive</key>
      <true/>
      <key>Label</key>
      <string>org.jenkins-ci</string>
      <key>ProgramArguments</key>
      <array>
        <string>/bin/bash</string>
        <string>/Library/Application Support/Jenkins/jenkins-runner.sh</string>
      </array>
      <key>RunAtLoad</key>
      <true/>
      <key>UserName</key>
      <string>jenkins</string>
      <key>SessionCreate</key>
      <true/>
    </dict>
  </plist>
  ```

- 开启服务并设置为开机启动

  ```
  sudo launchctl load -w /Library/LaunchDaemons/org.jenkins-ci.plist
  ```

#### 1.3 设置开机启动

- 将/usr/local/opt/jenkins/homebrew.mxcl.jenkins.plist文件拷贝到~/Library/LaunchAgents目录下

  ```
  cp -p /usr/local/opt/jenkins/homebrew.mxcl.jenkins.plist ~/Library/LaunchAgents  
  ```

- 设置开机启动

  ```
  sudo launchctl load -w /Library/LaunchAgents/homebrew.mxcl.jenkins.plist
  PS:若出现"/Library/LaunchAgents/homebrew.mxcl.jenkins.plist: Path had bad ownership/permissions"提示，则执行：
  sudo chown root /Library/LaunchAgents/homebrew.mxcl.jenkins.plist
  之后再次执行上面命令即可
  ```

- 取消开机启动

  ```
  sudo launchctl unload -w /Library/LaunchAgents/homebrew.mxcl.jenkins.plist
  ```

***PS:Mac OS中，使用launchctl指令可以开启一些启动服务，其中/Library/LaunchAgents中存放的是一些用户登陆后启动的服务。而/Library/LaunchDaemons中存放的就是用户未登录前就可以启动的服务。常用的命令有：launchctl load、launchctl unload、launchctl remove 、launchctl list***



### 2.初始化

> 安装完成之后，重启电脑，Jenkins默认会开启一个端口号为8080的本地的web服务。
>
> 在浏览器中输入localhost:8080访问Jenkins页面，进行初始化。

#### 2.1 Unlock Jenkins

按照提示打开文件，获取密码，复制密码到输入框中，继续。

#### 2.2 Customize Jenkins

### 3. 环境配置

#### 3.1 插件下载

> 系统管理-管理插件-可选插件-过滤搜索-直接安装
>
> 如果无法获取插件，可以在系统管理-管理插件-高级-升级站点，将默认的https网址修改为http的。

必选插件：

- Git client plugin 、Git plugin、Git server Plugin
- GitHub API Plugin、GitHub Branch Source Plugin、GitHub Organization Folder Plugin、GitHub plugin
- GitLab Authentication plugin、Gitlab Hook Plugin、GitLab logo Plugin、GitLab Plugin
- Xcode integration

#### 3.2 gitlab配置

在Credentials中添加凭证。我选择的是SSH授权。Username是你的gitlab用户名。查看本机.ssh/is_rsa中保存的私钥，将你gitlab账号中上传的SSH Key对应的私钥复制粘贴到Private Key中即可。Passphrase填写你当时生成ssh key的时候的密码即可。

### 4. 项目配置

## 二、打包脚本

### 1. 相关知识整理

#### 1.1 PlistBuddy

> plist文件是一种xml类型的在Mac OS以及iOS中最常见的一种文件格式，通过键值对的方式进行配置。PlistBuddy则是Mac自带的一个专门用来解析plist文件的工具。PlistBuddy支持的字段类型有：string、array、dict、bool、real、integer、date、data。具体使用可以在命令行中输入/usr/libexec/PlistBuddy --help来查看使用帮助

**基本使用介绍：**

- 添加

  - 添加普通字段

    ```
    /usr/libexec/PlistBuddy -c 'Add :Version string 1.0' test.plist
    ```

    ​

  - 添加数组字段

    ```
    # 先添加key值
    /usr/libexec/PlistBuddy -c 'Add :Application array' test.plist
    # 添加value值
    /usr/libexec/PlistBuddy -c 'Add :Application: string app1' test.plist
    /usr/libexec/PlistBuddy -c 'Add :Application: string app2' test.plist
    /usr/libexec/PlistBuddy -c 'Add :Application: string app3' test.plist
    ```

    ​

  - 添加字典字段

    ```
    # 先添加key值
    /usr/libexec/PlistBuddy -c 'Add :Person dict' test.plist
    # 添加value值,
    /usr/libexec/PlistBuddy -c 'Add :Person:Name string zs' test.plist
    /usr/libexec/PlistBuddy -c 'Add :Person:Age integer 3' test.plist
    ```

    ​

- 删除

  - 删除普通字段

    ```
    /usr/libexec/PlistBuddy -c 'Delete :Version' test.plist
    ```

    ​

  - 删除字典字段

    ```
    /usr/libexec/PlistBuddy -c 'Delete :Person:Age' test.plist
    ```

    ​

  - 删除数组字段

    ```
    /usr/libexec/PlistBuddy -c 'Delete :Application:2' test.plist
    ```

    ​

- 修改

  - 修改数组字段

    ```
    /usr/libexec/PlistBuddy -c 'Set :Application:1 "APP2"' test.plist
    ```

    ​

  - 修改字典字段

    ```
    /usr/libexec/PlistBuddy -c 'Set :Person:Name "ls"' test.plist
    ```

    ​


- 查找

  - 查找打印字段对应的值

    ```
    /usr/libexec/PlistBuddy -c 'Print :Person' test.plist
    ```

  - 查找数组中对应元素的值

    ```
    /usr/libexec/PlistBuddy -c 'Print :Application:2' test.plist
    ```

  ​

#### 1.2 xcodebuild

> 

**xcodebuild工作细节：**

- Check dependencies（检查项目配置，如Code Sign） -> Preprocessor -> Compile -> Link -> Copy Resource、Compile Xib、CompileStoryboard、CompileAssetCatalog -> Generate DSYM File -> ProcessProductPackaging -> Code Signing（需要访问钥匙串信息） -> Validate -> Result（.app和.dSYM）
- Code Signing（Code Signing Identity，Provisioning Profile）

**常用命令:**

- clean

  ```
  xcodebuild clean -workspace ${Workspace_Name} -scheme ${Scheme_Name}
  参数说明：
  * -workspace ${Workspace_Name} 
    指定工作空间文件XXX.xcworkspace
  * -scheme ${Scheme_Name} 
    指定构建工程名称
  ```

- archive

  ```
  xcodebuild archive
  -archivePath "${Archive_Path}" 
  -workspace ${Workspace_Name} 
  -scheme ${Scheme_Name} 
  -configuration ${Build_Config} 
  CODE_SIGN_IDENTITY="${Cert_Identity}" 
  DEVELOPMENT_TEAM="${Team_ID}"
  PROVISIONING_PROFILE_SPECIFIER="${Mobile_Provision_UUID}"  
  参数说明：
  *  -archivePath "${Archive_Path}" 
    Archive后文件导出的路径
  * -workspace ${Workspace_Name} 
    工作空间文件名称（XXX.xcworkspace）
  * -scheme ${Scheme_Name} 
    项目名称
  * -configuration ${Build_Config} 
    构建配置（Debug/Release）
  * CODE_SIGN_IDENTITY="${Cert_Identity}" 
    打包证书名称
  * DEVELOPMENT_TEAM="${Team_ID}"
    证书teamID
  * PROVISIONING_PROFILE_SPECIFIER="${Mobile_Provision_UUID}"  
    描述文件的UUID
  ```


- export

  ```
  xcodebuild
  -exportArchive 
  -archivePath "${Archive_Path}" 
  -exportPath ${Ipa_Name} 
  -exportFormat ipa 
  -exportWithOriginalSigningIdentity 
  参数说明：
  * -exportArchive 
  * -archivePath "${Archive_Path}"
  * -exportPath ${Ipa_Name} 
  * -exportFormat ipa 
  * -exportWithOriginalSigningIdentity 
  ```

  ​

#### 1.3 OCLint

> OCLint是一个静态代码分析工具，在Jenkins打包脚本中引入 OCLint 的目的，一方面是为了CodeReview。另一方面在编译失败的情况下输出分析日志，方便我们定位问题。
>
> OCLint的工作流程：
>
> 编译-->xcodebuild(生成xcodebuild.log)-->xcpretty(分析xcodebuild.log生成compile_commands.json文件)-->oclint-json-compilation-database(获取编译信息)-->oclint(输出分析报告)

- brew安装

  ```
  brew tap oclint/formulae
  brew install oclint
  ```

- 相关命令介绍

  - 1.oclint：基础指令。通过这个指令可以指定加载验证规则、编译代码、分析代码和生成报告。
  - 2.oclint-json-compilation-database：高级指令。通过这个指令可以从 compile_commands.json 文件中读取配置信息并执行 oclint。
  - 3.oclint-xcodebuild：主要用于生成compile_commands.json文件，已经不进行维护了，用xcpretty替代来生成json文件。

- 使用

  - 输出编译报告

    xodebuild生成分析xcodebuild.log文件，xcpretty分析xcodebuild.log生成compile_commands.json文件。

    tee命令获取输出结果。xcpretty命令将编译结果输出为json-compilation-database类型的编译报告。

    ```
    xcodebuild archive -archivePath "${Archive_Path}" -workspace ${Workspace_Name} -scheme ${Scheme_Name} -configuration ${Build_Config} CODE_SIGN_IDENTITY="${Cert_Identity}" DEVELOPMENT_TEAM="${Team_ID}" PROVISIONING_PROFILE_SPECIFIER="${Mobile_Provision_UUID}" | tee xcodebuild.log | xcpretty --report json-compilation-database
    ```

  - 输出分析报告

    根据上一步xcpretty生成的json-compilation-database类型的编译报告，然后就需要使用oclint-json-compilation-database来解析。

    ```
    oclint-json-compilation-database -e Pods -v -- -report-type html -o "${Report_HTML}" -max-priority-1=99999 -max-priority-2=99999 -max-priority-3=99999 1>/dev/null 2>&1
    ```

    ​

### 2. 脚本文件结构

### 3. 脚本描述

- main.sh

  ```
  main中主要做了以下几件事情：
  	1.配置路径信息(例如：脚本本地目录，默认配置目录，描述文件目录，keychain目录等等)
  	2.配置默认证书相关信息(例如：默认证书名称，teamID,描述文件，描述文件UUID)
  	3.Jenkins参数设置
  	4.执行mainUpgrade.sh获取Jenkins项目信息，更新脚本等
  	5.执行diag.sh输出诊断信息
  	6.执行mainRun.sh开始启动打包脚本
  ```

  - mainUpgrade.sh

    ```
    mainUpgrade中主要用来获取Jenkins对应jobs的项目名称，版本号，git分支，连接等相关信息
    ```

  - diag.sh

    ```
    diag主要输出相关使用命令的诊断信息
    ```

  - mainRun.sh

    ```
    mainRun中开始真正的打包脚本执行，依次执行了如下脚本
    1.comment.sh
    2.info.sh
    3.config.sh
    4.entitlements.sh
    5.pod.sh
    6.version.sh
    7.build.sh
    8.plist.sh
    9.html.sh
    10.smail.sh
    11.plistUpload.sh
    12.dSYMUpload
    13.appstore.sh
    ```

    - comment.sh

      ```
      用来生成本次打包的备注文案

      ```

    - diskClean.sh

      ```
      清理磁盘文件，清理DeriverdData,iOS DeviceSupport相关文件

      ```

    - info.sh

      ```
      读取项目工程信息

      ```

      - infoGit.sh

        ```
        主要做了2件事情：
        1.读取项目git信息，包括项目名称，项目当前分支名称
        2.读取打包配置文件(一个plist类型的文件，里面配置了对应项目的打包证书enterprise，ad-hoc,appstore证书信息，添加邮件抄送，debug/release等等一些打包配置信息)

        ```

      - infoProject.sh

        ```
        用来获取项目工程中的信息，包括workspace路径，bundleIdentigier,infoPlist路径，build号，version号等等

        ```

      - infoPath.sh

        ```
        用于配置和生成打包输出的相关文件路径，包括Archive文件路径，ipa文件路径，dSYM文件路径，Html文件路径，OCLint文件路径等等。

        ```

    - config.sh

      ```
      读取证书配置，配置证书相关信息。配置bugly信息。设置编译类型。设置输出包类型等等。

      ```

      - configCert.sh

        ```
        根据打包配置文件以及Jenkins配置信息，读取对应method(输出包类型)相对应的打包证书，描述文件，teamID，描述文件UUID等相关信息。

        ```

    - entitlements.sh

      ```
      读取相应的描述文件信息，利用plsitbuddy分别修改了.pbxproj以及.entitlements文件中的相关权限配置。

      ```

    - pod.sh

      ```
      pod repo update 以及 pod update

      ```

    - version.sh

      ```
      更新版本号，读取本地的一个用于记录版本号的文件。更新方式有2种方式，一种为版本号自增，一种为使用自定义的版本号。设置完更新文件即可。

      ```

    - build.sh

      ```
      1.修复keychain，导入 login.keychain
      2.archive 和 oclint
      3.

      ```

      - fixKeychain.sh

      - buildArchiveOrOCLint.sh

      - dSYM.sh

        ```
        将打包成功之后生成的.dSYM分别复制到项目路径以及dSYM导出路径下，并且将.dSYM文件压缩成zip包。
        ps：切记项目 buildSetting 搜索『dwraf』找到 『Debug Information Format』,应设置为『DWRAF with dSYM File』, 否则编译不会生成 dSYM 文件

        ```

      - reSign.sh

    - plist.sh

      ```
      这个脚本是用来生成一个plist文件，用来提供给itms-services://协议使用的，其中itms-services://协议中最主要的也就是这个plist文件。在之后会生成一个html文件，会将这个itms-services://协议添加进"下载"标签的href中。这样在safari浏览器中打开这个网页，点击下载，就可以很方便的下载安装包。

      脚本参考代码如下：
      #!/bin/sh

      Install_Plist_Name=${Ipa_Name}_install.plist
      Install_Plist_Path="${Install_Path}/${Install_Plist_Name}"
      Install_Ipa_URL="${Download_Ipa_URL}/${Ipa_Name}.ipa"

      cat << EOF > $Install_Plist_Path
      <?xml version="1.0" encoding="UTF-8"?>
      <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
      <plist version="1.0">
      <dict>
      	<key>items</key>
      	<array>
          	<dict>
              	<key>assets</key>
              	<array>
      	            <dict>
      	                <key>kind</key>
      	                <string>software-package</string>
      	                <key>url</key>
      	               	<string>$Install_Ipa_URL</string>
      	            </dict>
                  </array>
                  <key>metadata</key>
                  <dict>
                      <key>bundle-identifier</key>
                      <string>$Bundle_Identifier</string>
                      <key>bundle-version</key>
                     	<string>$New_BuildVersion</string>
                      <key>kind</key>
                     	<string>software</string>
                      <key>title</key>
                     	<string>$Bundle_App_Name</string>
                  </dict>
              </dict>
      	</array>
      </dict>
      </plist>
      EOF

      ```

    - html.sh

      ```
      生成一个静态页面用于展示打包信息以及绑定生成好的itms-services://协议，然后使用一个写好的js二维码生成库，生成一个网页链接二维码。
      网页包含内容：
      1.项目名称
      2.证书名称
      3.版本号
      4.编译号
      5.备注信息
      6.二维码

      ```

    - smail.sh

      - emails.sh

        ```
        用来生成邮件信息

        ```

      - sendmail.py

        ```
        发送邮件的脚本

        ```

    - plistUpload.sh

      ```
      提交之前生成好的plist文件到一个https站点（iOS7之后不支持http）, 这里就使用了coding.net的仓库。

      ```

    - dSYMUpload

      ```
      因为自家接入的是bugly，每次app上线后，手动上传dSYM文件很麻烦。所以这里弄了个脚本，根据Jenkins配置信息是否是否上传bugly。将之前生成好的.dSYM.zip自动上传到bugly。

      上传参考代码：
      #!/bin/sh

      URLEncodeFunc() {
      	local length="${#1}"
      	for (( i = 0; i < length; i++ )); do
      		local c="${1:i:1}"
      		case $c in
      			[a-zA-Z0-9.~_-]) printf "$c" ;;
      			*) printf "$c" | xxd -p -c1 | while read x;do printf "%%%s" "$x";done
      		esac
      	done
      }

      if [[ $Jenkins_dSYM_Upload = true ]]; then
      	echo "\n上传 dSYM 到 Bugly..."

      	# Bugly 的 TS 说他们要的是这样的版本号格式
      	Bugly_Version="${Bundle_Version}(${New_BuildVersion})"
      	
      	# pid默认为2
      	dSYM_Pid="2"
      	dSYM_AppID=${Bugly_AppId}
      	dSYM_AppKey=${Bugly_AppKey}
      	dSYM_BundleId=${Bundle_Identifier}
      	dSYM_Version=`URLEncodeFunc ${Bugly_Version}`
      	dSYM_ZipFileName=`URLEncodeFunc ${dSYM_Path}`

      	java -jar ${Bugly_JAR_Path} -i ${dSYM_Path} -u -id ${dSYM_AppID} -key ${dSYM_AppKey} -package ${dSYM_BundleId} -version ${dSYM_Version} 1>/dev/null

      else
      	echo "\n不上传 dSYM 到 bugly\n"
      fi

      rm -rf ${dSYM_Path}


      ```

    - appstore.sh