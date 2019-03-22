---
title: TWA踩坑记-从零到一让你的博客变成app
date: 2019-03-21 17:58:10
categories: hexo静态博客搭建教程
tags:
  - PWA
  - TWA
  - Trusted-Web-Activities
  - CircleCI
  - Google-Developer
  - Chrome
translate_title: twa-you-blog-can-become-an-app
img: https://www.kobil.com/wp-content/uploads/2018/09/Trusted-app.png
---

## 前言

在上一篇文章 [PWA踩坑记-从零到一让你的博客也能离线访问](https://tellyouwhat.cn/p/pwa-your-blog-can-also-be-accessed-offline/) 中，我介绍了如何将您的博客升级为`PWA` (Progressive-Web-App) 应用。

**在这篇文章里，我将向您一步一步展示如何使您现有的`PWA`转化为`TWA`**

您将学到：

- 什么是TWA？
- 什么是activity？
- TWA特性
- 安卓开发基础环境搭建
- Gradle的基本概念
- TWA于网站的双向验证方法
- Android软件签名
- 如何自动化把隐藏的静态资源复制到public文件夹？
- 安卓软件的一般发布到应用市场流程

## 什么是TWA？

`TWA`全称为 Trusted Web Activities（被信任的网络应用）。简单来说，它的作用就是**使您现有的`PWA`可以在任意设备上运行，不管有没有安装Chrome浏览器。**

在将您的网站改造为TWA之前，您必须先按照上一篇文章 [PWA踩坑记-从零到一让你的博客也能离线访问](https://tellyouwhat.cn/p/pwa-your-blog-can-also-be-accessed-offline/) 将网站改造成PWA，然后才能进行接下来内容的学习。

> 这篇文站同样适用SPA（单页应用），和其它动态网站

在阅读下文前，请牢记，**TWA就是一个activity**

### 什么是 activity

activity ，既为安卓里面的一种“窗口”或者叫“页面”，每打开一个 app ，就是打开了一个 activity ，一般 app 内包含若干个 activity ，他们之间可以相互转跳，当你看到转跳动画的时候，很可能就是发生了 activity 的转跳。

![activity 跳转](https://ws1.sinaimg.cn/large/bd61005ely1g1bkazjd36j20sg0g0jrw.jpg)

一个 app 含有0至多个 activity，一般情况下不会出现0个 activity 的 app，因为0个 activity 实际上意味着此 app 无用户交互页面。但是实际上，很大一部分的 app 是不需要与用户交互的，您可以翻看一下自己手机的应用列表里面的系统应用，您会发现，安装的应用数量相当庞大，但是自己主屏幕的图标数量却没那么多，就是因为很多 app 都是0个 activity 的原因。

> 以上说法可能不太准确，而且页面还可能是fragment，而不是activity，但是您只需要简单了解至此即可。

### 什么是 web activity

web activity 是相对于 webview 来说的，webview 传统情况下是app承载浏览器功能的一种实现。比如说在QQ里面点击了一个链接，就会打开一个类似于内置浏览器的页面来访问您所点击的链接。当然类似的 webview app 很多，比如：淘宝，京东，QQ，微信，小黑鱼，百度地图，网易云音乐等等。

![浏览器、Custom-Tab、WebView 对比](https://ws1.sinaimg.cn/large/bd61005ely1g1bkvbu4cpg20hs0a0apj.gif)

传统 webview 的好处自然是其丰富的可定制化的特性，比如管理用户 cookie ，防止用户访问不安全的网站，强制启用安全访问等。

但是随之而来的问题就是：

1. webview 的更新不是由用户来决定的，当 app 的 webview 过期时，对于用户的信息安全是非常危险的；

2. app要集成一个webview进去，会造成应用体积暴增，导致用户出现抵触安装的情绪；

3. 开发复杂，移动端工程师必须花费心思去维护webview模块，以保证其功能性。

所以 web activity 的概念是**就是一个页面**，他会替你在内部加载网页，模仿得让用户不太感觉的出来是一个网页。

## TWA的特性

1. **内容安全可信任。**用户打开后访问的内容一定是你认可的网站，你的网站不会被其他app所冒充。（后文将介绍如何让你的TWA和网站相互认证）

2. **你的网页是什么样，用户打开TWA就是什么样**，就像在普通的浏览器里面一样，只不过，它是**全屏运行**的。当然，TWA的先决条件就是你的网站首先能适应移动端的屏幕大小，或专门为移动端开发。

3. **TWA默认还是会使用chrome浏览器作为承载**。但是不像『[PWA踩坑记-从零到一让你的博客也能离线访问](https://tellyouwhat.cn/p/pwa-your-blog-can-also-be-accessed-offline/)』那样，需要强制用户下载安装chrome浏览器才能正常使用。其他实现了TWA的浏览器同样可以作为您app里web内容的承载。

![TWA](https://ws1.sinaimg.cn/large/bd61005ely1g1bloqiddyj216i23kaq3.jpg)

> 截止这篇文章发布的时候2019年3月23日，如果用户没有安装chrome或其他实现了TWA接口的浏览器，TWA会fallback到Custom Tab或用户手机的默认浏览器。

4. 宿主app对您TWA内容没有直接的访问权限，它仅作为一个承载的作用。比如您不能通过app直接操作cookie或`localStorage`。如果您需要根据TWA定制化您的网页内容或显示形式，您可以通过url参数、自定义的请求/响应头（HTTP header）或[intent URIs](https://developer.chrome.com/multidevice/android/intents)，比如您需要实现国际化等操作。

## 开发TWA

正如前文所说到，一个app至少包含0个activity，所以app是您开发任何activity的前置条件。

接下来我将向您展示如何创建一个安卓app项目来开发您的TWA。（您不需要擅长Java或Kotlin的知识）

![以本地APP安装的TWA](https://ws1.sinaimg.cn/large/bd61005ely1g1blpe9lrhj216i23kwly.jpg)

### 环境搭建

#### Android Studio

首先，您需要安装安卓集成开发环境（IDE）：[Android Studio](https://developer.android.com/studio/)。

1. 点击上面链接，下载对应版本的Android Studio，并安装，下文以PC为例：

![点击 Download Android Studio](https://ws1.sinaimg.cn/large/bd61005ely1g1blsr6adgj21ea0q6ajb.jpg)

2. 勾选同意条款，然后开始下载安装。

3. 安装完毕之后，第一次打开会要求您安装SDK

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bm0vvxtnj211i0ml0tu.jpg)

   选择最小化的SDK安装即可，推荐您使用真机进行调试，即不需要安装Simulation之类的虚拟机

### 开发过程

#### 打开Android Studio，新建一个项目

Android Studio将会提示您选择一种预设activity，请在这里选择No activity，因为稍后引入的库会包含一个activity。

![选择No activity](https://ws1.sinaimg.cn/large/bd61005ely1g1bm4oidlej20vr0plq3y.jpg)

#### 点击下一步，填写app基本信息

![填写app信息](https://ws1.sinaimg.cn/large/bd61005ely1g1bmgjmbg6j20vr0plq3n.jpg)

- Name是您应用的名字
- Package Name 一般以倒写的网址开头，是Java编码的即来习惯
- Save Location 项目在磁盘中存放的位置
- Minimum API Level 项请选择API Level 16 或以上，这是TWA支持的最低标准
- 其他选项如图默认即可
  
#### 点击Finish

#### 等待Gradle脚本构建完成

![Gradle脚本正在同步中](https://ws1.sinaimg.cn/large/bd61005ely1g1bm8rk987j20e307rwel.jpg)

> 建议将文件显示方式更改为project，这样会以真实的文件系统存放方式显示您的文件夹。
> ![](https://ws1.sinaimg.cn/large/bd61005ely1g1bmchypc5j20g30d90tf.jpg)


在继续之前，您需要知晓，**gradle是一种构建工具**，就像npm+webpack（nodejs），maven（Java web）等其他包管理构建工具。

**gradle项目可以做到模块化**，在默认的安卓项目里，一个叫做app的模块就是您默认的模块，所以您的项目里应该有两个gradle脚本名字都是：`build.gradle`，在最外层的`build.gradle`是项目级别的配置文件，在app目录下的`build.gradle`是当前模块级别配置文件。

#### 修改gradle文件

1. 打开项目级别的`build.gradle`文件，添加一个源`maven { url "https://jitpack.io" }`

```groovy
allprojects {
   repositories {
       google()
       jcenter()
       maven { url "https://jitpack.io" }
   }
}
```

因为您改动了这些脚本，Android Studio 将会提示您 Sync Now，请点击同步。

2. 在您app模块级别的`build.gradle`文件中添加一条依赖

```groovy
implementation 'com.github.GoogleChrome.custom-tabs-client:customtabs:76d8d07'
```

关于custom-tabs-client的版本问题，请您前往<https://github.com/GoogleChrome/custom-tabs-client>查看最新Commit ID，如：

![图中橘黄色圈起来的即为版本](https://ws1.sinaimg.cn/large/bd61005ely1g1bmnuu09wj20z40nntbd.jpg)

如有必要请您自行替换版本号。

#### 再次选择立即同步（Sync Now）

#### 添加TWA Activity

导航到您App的`app/src/main/AndroidManifest.xml`文件

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bmysjub0j20fl0inwf1.jpg)

将如下`activity`tag添加到`application`tag里面

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="cn.tellyouwhat.articles">

    <application
        android:allowBackup="false"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="false"
        android:theme="@style/AppTheme">

        <activity android:name="android.support.customtabs.trusted.LauncherActivity">

            <!-- Edit android:value to change the url opened by the TWA -->
            <meta-data
                android:name="android.support.customtabs.trusted.DEFAULT_URL"
                android:value="https://tellyouwhat.cn" />

            <!-- This intent-filter adds the TWA to the Android Launcher -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

            <!--
              This intent-filter allows the TWA to handle Intents to open
              tellyouwhat.cn.
            -->
            <intent-filter android:autoVerify="true">
                <action android:name="android.intent.action.VIEW" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <!-- Edit android:host to handle links to the target URL-->
                <data
                    android:host="tellyouwhat.cn"
                    android:scheme="https" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

1. 该`meta-data`标记告诉TWA哪个URL应该打开。`android:value`使用要打开的PWA的URL 更改属性。
2. 在**第二个** `intent-filter`标签允许TWA被其他应用如浏览器打开。

![从浏览器点击了tellyouwhat.cn域下资源跳转到TWA应用](https://ws1.sinaimg.cn/large/bd61005ely1g1bn7y3essg20ex0qd4qq.gif)

#### 全屏运行？

##### 建立从app到网页的认证

在`app > res > values > strings.xml`文件中添加以下声明：

```xml
<resources>
    <string name="app_name">告你什么文集</string>
    <string name="asset_statements">
        [{
            \"relation\": [\"delegate_permission/common.handle_all_urls\"],
            \"target\": {
                \"namespace\": \"web\",
                \"site\": \"https://tellyouwhat.cn\"}
        }]
    </string>
</resources>
```

将其中的<https://tellyouwhat.cn>更改为您自己的网站。

返回到`AndroidManifest.xml`文件中，在activity tag之前，添加如下代码：

```xml
<meta-data
    android:name="asset_statements"
    android:resource="@string/asset_statements" />
```

##### 建立从网页到app的认证

> 本文假设您使用的是hexo静态博客生成器

1. 生成app的签名的证书文件

选择菜单栏上build菜单，选则generate signed bundle/apk，在弹出的窗口中选择apk

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bnidl7e7j20je0fgdg6.jpg)

点击下一步

![选择Create new](https://ws1.sinaimg.cn/large/bd61005ely1g1bnk91jsnj20je0fgmxe.jpg)

点击新建一个key

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bntgwrnuj20ia0jwglz.jpg)

这里面的内容可以随便填一天，但是要注意这么几个细节：

- 一个jks文件可以被多个app使用
- 但是每个app都只能用唯一的别名（alias）
- key的密码和alias的密码是分开的
- 签名时这两个密码都需要，所以您应该记住自己设置的密码

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bnw4oq4gj20je0fg3yt.jpg)

> 继续点击next
>
> ![](https://ws1.sinaimg.cn/large/bd61005ely1g1bnxlbis8j20je0fg0sx.jpg)
>
> 选择V1签名，即可生成正式版app（右上角点击运行出现在手机上的app属于debug版本，非正式签名，是无法发不到应用商店的）

这样我们就得到了签名的证书文件

2. 生成app links

在IDE的右侧边栏寻找Assistant

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bnzvfrsej202z064dfn.jpg)

如果没有找到的话，请双击shift，在打开的面板里输入assistant

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bo1h8s4kj20eu0cy757.jpg)

然后就能打开了。

点击第3条：open digital asset links file Generateor

确认域名和包名的正确性，然后选择keystore文件，点击生成一段json文本

![](https://ws1.sinaimg.cn/large/bd61005ely1g1bo42z621j20yq0o4dha.jpg)

3. 将这段生成的文本复制下来
4. 到您的hexo博客根目录下

还记得我在上篇文章中说到的您自己新建的`static_files`文件夹吗？

在这里它将继续发挥作用。

> 根据TWA的要求，您必须把刚才复制的那段json文本保存成`/.well-known/assetlinks.json`文件放在您的域名下，但是他是一个`.`开头的文件夹，hexo在生成博客的时候，会忽略掉`source`文件夹下以`.`开头的文件(夹)

在您以前在`scripts`文件夹下创建的`event.js`中添加如下代码，他的功能是把某文件夹下所有文件复制到最终生成的`public`文件夹下:

```javascript
hexo.on('generateAfter', function () {
  if (!fs.existsSync('./public')) {
    // 如果缓存文件夹不存在就创建一个
    fs.mkdirSync('./public')
  }
  copyDir('./static_files/copytopublic', './public', function (err) {
    if (err) {
      console.error(err);
    }
  })

  fs.writeFile('./public/sw.js',
    fs.readFileSync('./static_files/sw.js').toString().replace('{uniqueIdentifier}', new Date().toISOString()),
    function (err) {
      if (err) {
        console.error(err)
      } else {
        console.log('service worker created')
      }
    })
})

/*
 * 复制目录、子目录，及其中的文件
 * @param src {String} 要复制的目录
 * @param dist {String} 复制到目标目录
 */
function copyDir(src, dist, callback) {
  fs.access(dist, function (err) {
    if (err) {
      // 目录不存在时创建目录
      fs.mkdirSync(dist);
    }
    _copy(null, src, dist);
  });

  function _copy(err, src, dist) {
    if (err) {
      callback(err);
    } else {
      fs.readdir(src, function (err, paths) {
        if (err) {
          callback(err)
        } else {
          paths.forEach(function (path) {
            var _src = src + '/' + path;
            var _dist = dist + '/' + path;
            fs.stat(_src, function (err, stat) {
              if (err) {
                callback(err);
              } else {
                // 判断是文件还是目录
                if (stat.isFile()) {
                  fs.writeFileSync(_dist, fs.readFileSync(_src));
                  console.log('copying file', _src)
                } else if (stat.isDirectory()) {
                  // 当是目录是，递归复制
                  copyDir(_src, _dist, callback)
                }
              }
            })
          })
        }
      })
    }
  }
}
```

该代码将会把您`./static_files/copytopublic`文件夹下的所有文件复制到最终的`public`文件夹中。

下一步我们在`./static_files/copytopublic`文件夹下创建`/.well-known/assetlinks.json`文件，并把Android Studio Assistant生成的json文本填充进去。

这时候，执行`hexo g`的时候，就会在`public`文件夹看到`/.well-known/assetlinks.json`文件。

但是！！！`hexo g`的时候依然不会把文件最终发布到目的地。

接下来，请打开hexo博客的`_config.yml`文件，在您的`deploy`项中添加`ignore_hidden`属性：

```yaml
deploy:
  - type: git
    repo: git@somegit.com:yourgit.git
    ignore_hidden: false
```

完美解决hexo博客的问题。

##### 测试

在您成功部署之后（如有CDN，您可能需要刷新一下缓存），返回到Android Studio 的 Assistant，往下翻，下方有一个验证的工具，点击之后，如果通过了，就会有两个绿色亮起。

![测试通过](https://ws1.sinaimg.cn/large/bd61005ely1g1bol6raalj20mz07dq31.jpg)

### 更换应用图标

在您项目的`app\src\main\res\`文件夹下，有好几个mipmap开头的文件夹，他们是分别针对不同屏幕尺寸的图标，简单的，您只需将其中的png文件替换即可。

![图标文件](https://ws1.sinaimg.cn/large/bd61005ely1g1bp02q1bcj20es0jj3z1.jpg)

为此您可能需要裁切很多分辨率的图标文件：

![不同尺寸的图标文件](https://ws1.sinaimg.cn/large/bd61005ely1g1bp37zn4aj20xp0kuq5x.jpg)

### 完整代码

您可以在这里找到实例的完整代码：<https://github.com/HarborZeng/tellyouwhat-app>

## 发布应用

这篇文章将会以发布到酷安为例，为您介绍如何将app发布到应用市场。

### 生成release版本app

正如前文所提到过的，在build菜单项，选择generate signed apk，下一步，选择V1签名，即可生成正式版app。

![正式版app存放路径](https://ws1.sinaimg.cn/large/bd61005ely1g1booqjrofj20lp09k3yt.jpg)

### 使用酷安应用市场

1. 首先您需要注册酷安账号，这里省略。

2. 然后前往<https://developer.coolapk.com/>认证为开发者，这里省略。
3. 点击添加新应用。
4. 输入您应用的信息![输入您应用的信息](https://ws1.sinaimg.cn/large/bd61005ely1g1boskmoykj20nn0cqdge.jpg)

5. 上传图标，输入简介，添加截图等
6. 上传apk文件，通过后即可发布![上传apk文件](https://ws1.sinaimg.cn/large/bd61005ely1g1bowyu8n3j219r0plac2.jpg)

### 发布完成

![](https://ws1.sinaimg.cn/large/bd61005ely1g1boyqzhzoj20st0q5djr.jpg)

## 参考资料

[^1]: Using Trusted Web Activities https://developers.google.com/web/updates/2019/02/using-twa
[^2]: custom-tabs-client https://github.com/GoogleChrome/custom-tabs-client
[^3]: 酷安开发者 https://developer.coolapk.com
[^4]: Chrome Custom Tabs最佳实践 https://qq157755587.github.io/2016/08/12/custom-tabs-best-practices/

## 原文参见：<https://tellyouwhat.cn/p/twa-you-blog-can-become-an-app/>
