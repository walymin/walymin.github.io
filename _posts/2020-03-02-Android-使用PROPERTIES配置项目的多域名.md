---
layout:     post
title:      Android 使用PROPERTIES配置项目的多域名
subtitle:   Android 告别SharedPreferences,使用Properties读、写多域名
date:       2020-03-02
author:     Admin
# header-img: assets/post-bg-android-2.jpg
header-img: https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-post-bg-img-2020/post-bg-android-2.jpg?x-oss-process=style/q_80
header-mask: 0.2
catalog: true
tags:
    - Android
---

#### 需求
 一般我们在开发app的时候，会在app上面增加一个开发者选项页面，用于给测试或者运营人员切换环境使用，同时提供环境域名的修改，跟保存。
 

#### 需求分析
 一个app的域名可能会有很多个，这个可配置可修改的域名也相应同步。
 例如一般会增加一个开发者选项页面，类似如图：
 
![49f0a1acafad326dbcd0487c99c90899](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355582699.png?x-oss-process=style/watermark-yumi)

使用者可以根据需要切换到不同的环境，类似如图：
![f0d66e203dde822b74ce2ca63390eb1e](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355583699.png?x-oss-process=style/watermark-yumi)

这样就不需要每次测试找你打某个环境的包了，节省了很多不必要的麻烦。


#### 需求实现-常用方法

下面两种方法是我在经历不同的项目中看到的写法：
* 方式一：通过定义全局的静态常量数组+ SharedPreferences。
    * 针对每一个地址定义String[]静态常量数组，存放该地址的不同的环境。
    * 通过SharedPreferences保存开发者选项里面的环境切换，跟环境修改。
    * 在application 生命周期方法里，先读取SharedPreferences保存的地址，没有就去String数组里面获取。

* 方式二： 通过gradle.properties配置 + build.gradle(app)读取 + SharedPreferences保存修改
    * 使用项目根目录下的gradle.properties配置域名：![d227c73131d12daab6a8d2289ca3c5f5](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355584699.png?x-oss-process=style/watermark-yumi)
    * 在app下的build.gradle 的defaultConfig 里面配置：![ebb7d4704e39821e83d5789945325aa7](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355585699.png?x-oss-process=style/watermark-yumi)
    * 通过getString(R.string.text_one_url)，这样的方式读取。
    * 开发者选项里面的修改，也是通过SharedPreferences保存，读取的时候先读SharedPreferences，为空则通过getString(R.string.xx)读取。
    * 这里可能你要在defaultConfig里配置所以环境的域名，因为开发者选项页面切换环境，要显示不同环境的地址。我看到的是有开发者直接在这个页面也维护了一份string[]数组。


#### 使用PROPERTIES来配置读写域名

在app/src/main/assets目录下新建各环境的properties文件，如图：![5b2885f598835107e4cbde80f2edf27a](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355586699.png?x-oss-process=style/watermark-yumi)

**1. 读取assets目录下的properties文件**

```java
    
    val props = Properties()                   
    props?.load(context.assets.open("DE.url.properties"))
    
    //这里还有一种方法,使用这种方法读取的不是assets目录下的，下面会讲到
    props?.load(context.openFileInput("DE.url.properties"));
    
```
  
* 因为Properties 继承Hashtable<Object,Object>，获取到properties后，可以通过key值获取values的值（域名）。


**2. 写入域名到properties：**

* 这里读取assets目录下的，根本无法修改，网上有很多说法通过 FileOutputStream去读取，然后修改，也是有点问题。
* 先理解一下获取Properties的值的两种方式：


```java
  
//方式一
props?.load(context.assets.open("custom.url.properties")
    
//方式二
props?.load(context.openFileInput("custom.url.properties")
     
```
这两种方法加载的不是同一个custom.url.properties文件，看如下图：
![a698d2884ee411395bfb40c814c619eb](https://github-io-blog.oss-cn-shanghai.aliyuncs.com/github-blog/blog-img-2020/android-1584355587699.png?x-oss-process=style/watermark-yumi)

那么怎么在files目录下生成custom.url.properties文件呢？

通过读取一个properties然后保存到files中

```java

val props = Properties()
props?.load(context.assets.open("custom.url.properties"))
val out = context.openFileOutput("custom.url.properties",Context.MODE_PRIVATE);
props?.store(out, null)
                
```

这样，读跟写的方式都解决了，就是写一些业务逻辑之类的行为了。下面将写的工具类贴一下，每个方法都有注释：

```java

/**
 *  域名工具
 * author : AdminWang
 */
class PropertiesUtils {
    companion object {
        private var URLPROPERTIES: Properties? = null
        private var CUSTOMURLPROPERTIES: Properties? = null
        //各域名的key
        const val TEXT_ONE_URL = "text_one_url"
        const val TEXT_TWO_URL = "text_two_url"
        const val TEXT_THREE_URL = "text_three_url"
        const val TEXT_APPLOG_URL = "text_applog_url"
        const val TEXT_ACTIVITY_URL = "text_activity_url"

        /**
         * URL参数
         */
        var urlProperties: Properties?
            get() = URLPROPERTIES
            set(value) {}

        var customUrlProperties: Properties?
            get() = CUSTOMURLPROPERTIES
            set(value) {}

        /**
         * 在application oncreate中初始化
         */
        fun initProperties(context: Context) {
            initUrlProperties(context)
            initCustomUrlProperties(context)
        }

        /**
         * 判断环境是开发者选项里已经切换保存的，还是第一次打包打的环境
         */
        fun getEnvironment(context: Context): String =
            if(SharedPreferencesUtil.getSPString(context, SharedPreferencesUtil.URL_ENVIRONMENT_SETTING).isNullOrBlank()) {
                context.getString(R.string.env)
            } else {
                SharedPreferencesUtil.getSPString(context, SharedPreferencesUtil.URL_ENVIRONMENT_SETTING)
            }

        fun isReleaseBuild(context: Context): Boolean = context.getString(R.string.env) != "PE"


        /**
         * 保存开发者选项自定义的环境配置
         */
        fun setCustomUrlProperties(context: Context, keyName: String?, keyValue: String?) {
            val properties = Properties()
            try {
                properties?.load(context.openFileInput("custom.url.properties"))
                properties?.setProperty(keyName, keyValue)
                val out = context.openFileOutput("custom.url.properties",Context.MODE_PRIVATE)
                properties?.store(out, null);
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }

        /**
         * 保存开发者选项自定义的环境配置
         */
        fun setCustomUrlProperties(context: Context, map: ArrayMap<String, String>) {
            val properties = Properties()
            try {
                properties?.load(context.openFileInput("custom.url.properties"))
                map.forEach {
                    properties?.setProperty(it.key, it.value)
                }
                val out = context.openFileOutput("custom.url.properties",Context.MODE_PRIVATE);
                properties?.store(out, null);
            } catch (e: Exception) {
                e.printStackTrace()
            }
        }


        fun getUrlProperties(context: Context): Properties? {
            if(CUSTOMURLPROPERTIES.isNullOrEmpty()) {
                return URLPROPERTIES
            }
            return CUSTOMURLPROPERTIES
        }

        /**
         * 开发者选择切换环境，获取某环境下的域名展示
         */
        fun getSettingUrlProperties(context: Context,env: String): Properties? {
            val props = Properties()
            props?.load(context.assets.open("${env}.url.properties"))
            return props
        }

        /**
         * 初始化域名环境
         */
        private fun initUrlProperties(context: Context) {
            URLPROPERTIES = Properties()
            val environment = getEnvironment(context)
            if (environment == "DE") {
                URLPROPERTIES?.load(context.assets.open("DE.url.properties"))
                return
            }
            if (environment == "AE") {
                URLPROPERTIES?.load(context.assets.open("AE.url.properties"))
                return
            }
            if (environment == "ME") {
                URLPROPERTIES?.load(context.assets.open("ME.url.properties"))
                return
            }
            if (environment == "XE") {
                URLPROPERTIES?.load(context.assets.open("XE.url.properties"))
                return
            }
        }

        /**
         * 初始化域名环境
         */
        private fun initCustomUrlProperties(context: Context) {
            CUSTOMURLPROPERTIES = Properties()
            try {
                //第一次读取，因为文件不存在所以直接报错，走catch的生成文件的逻辑。后面都是直接读取files里面的properties文件。
                CUSTOMURLPROPERTIES?.load(context.openFileInput("custom.url.properties"))
            } catch (e: Exception) {
                e.printStackTrace()
                CUSTOMURLPROPERTIES?.load(context.assets.open("custom.url.properties"))
                val out = context.openFileOutput("custom.url.properties",Context.MODE_PRIVATE);
                CUSTOMURLPROPERTIES?.store(out, null)
            }
        }
    }
}


```