---
layout: post
title:  "Gradle 打包 Android APP"
date:   2015-01-31 23:27:10
tagline: "Supporting tagline"
tags : Android,gradle
published: true
categories: 技术分享
---
# Gradle 打包 Android APP
---
1.配置Gradle、android sdk path的问题略过，比较简单，网上很多。     
2.Gradle 具体的打包，之前的资料还很少，具体的问题自己也遇到不少，做下总结。
Gradle 的版本跟随着 Android 一直在升级，现在的这个编译已经是修改过几次了，目前代码对应的版本为 ：2.1。      
工程有2个代码文件：      
1.  build.gradle，编译的具体代码，签名信息、代码信息、混淆文件等   
2.  settings.gradle，工程路径说明，本地有自己或者第三方的 lib 时候，需要在这里进行说明。

### build.gradle 的代码

```
import com.android.build.gradle.AppPlugin
import com.android.build.gradle.LibraryPlugin

buildscript   
{  
    repositories {  
        mavenCentral()  
    }  
    dependencies {  
        classpath 'com.android.tools.build:gradle:0.14.+'  
    }  
}

apply plugin: 'android'

android {
    compileSdkVersion 19
    buildToolsVersion "20.0.0"

    defaultConfig {
        applicationId "com.xx.xx"
        minSdkVersion 8
        targetSdkVersion 20
    }

    //签名 
    signingConfigs {
        release {
            storeFile file('xxx.keystore')
            storePassword 'xxxx'
            keyAlias 'xxx.keystore'
            keyPassword 'xx'
        }
    }

    // 代码目录说明，如果是用 Android studio 开发，则不用加入这个，如果还是 Eclipse的版本则还是加入这个
    sourceSets {  
        main {  
            manifest.srcFile 'AndroidManifest.xml'  
            java.srcDirs = ['src']  
            resources.srcDirs = ['src']  
            aidl.srcDirs = ['src']  
            renderscript.srcDirs = ['src']  
            res.srcDirs = ['res']  
            assets.srcDirs = ['assets']             
       }  
     }
       
    packagingOptions {
        exclude 'META-INF/DEPENDENCIES'
        exclude 'META-INF/NOTICE'
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/LICENSE.txt'
        exclude 'META-INF/NOTICE.txt'
        exclude 'META-INF/ASL2.0'
    }

    lintOptions {
        abortOnError false
        ignoreWarnings true
    }

    //多渠道，友盟添加多渠道版本需要 
    productFlavors {
        _360market {}
//        _91market {}
    }

	// 不会一次性编译通过，如果编译或者运行有问题，注意查看错误代码，有可能是混淆代码有问题
    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-project.txt'
        }
    }
}

// 第三方的lib导入
dependencies {
 	compile fileTree(dir: 'libs', include: '*.jar')
    compile project(':volley-libray')
    compile files('libs/eventbus.jar')
    compile files('libs/gson-2.3.jar')
    compile files('libs/greendao-1.3.7.jar')
    compile files('libs/nineoldandroids-library-2.4.0.jar')
    compile files('libs/umeng-analytics-v5.2.4.jar')
    compile files('libs/universal-image-loader-1.9.3.jar')
}

// 友盟多渠道的替代，如果不使用，可以删除这部分代码，这个之前是参考别人的代码，但是参考的版本的 Gradle 已经不适用
android.applicationVariants.all{ variant ->
	variant.outputs.each { output ->
	    output.processManifest.doLast{
	    		def manifestFile = output.processManifest.manifestOutputFile	
	    		def updatedContent = manifestFile.getText('UTF-8').replaceAll("UMENG_CHANNEL_VALUE", "${variant.productFlavors[0].name}")
    			manifestFile.write(updatedContent, 'UTF-8')
	    }
	}
}
```

### settings.gradle 代码

```
include ':volley-libray'

project(':volley-libray').projectDir = new File('/Users/**/**/volley-libray')
```

### 遇到的部分问题
##### 1.Configuration with name 'default' not found.

一般是子模块中出问题，要确定子模块都有 build.gradle 文件，并且都能编译通过。    
出处：http://stackoverflow.com/questions/18178267/configuration-with-name-default-not-found-while-building-android-project-on-gr

##### 2.Getting error “Gradle version 1.10 is required. Current version is 1.12.” when executing “gradle wrapper”?
gradle 的版本问题，修改为高版本

```
buildscript { repositories { mavenCentral() } dependencies { classpath 'com.android.tools.build:gradle:0.14.+' } }
```
出处：http://stackoverflow.com/questions/23464780/getting-error-gradle-version-1-10-is-required-current-version-is-1-12-when-e


