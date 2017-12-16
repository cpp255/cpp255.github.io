---
layout: post
title:  "Android-Universal-Image-Loader disk 缓存图片更改"
keywords : Android,image
published: true
categories: 技术分享
---
# Android-Universal-Image-Loader缓存图片更改
---
之前在论坛上看到过介绍 [bither-android-lib](https://github.com/bither/bither-android-lib) 这个工程，主要是说 Android 现有的图片缓存机制比较老，效果不好，然后替换成这个工程，作者说有测试过，效果会比较好。我也拿来做测试了，没看出来。想着可以把平时用的图片的加载库替换掉 [Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader)。


### 1. 修改依赖库中的保存代码 

1.把 use-libjpeg-trubo-adnroid 工程下 net.bither.util 包下的 NativeUtil.java 拷贝到 Android-Universal-Image-Loader 工程下 library 的工程com.nostra13.universalimageloader.utilsb 包中；   

2.找到 com.nostra13.universalimageloader.cache.disc.impl 包下的 BaseDiskCache.java 类，重写其中的两个保存方法(源代码中注释掉的为之前版本库的代码)：


```
@Override
	public boolean save(String imageUri, InputStream imageStream, IoUtils.CopyListener listener) throws IOException {
		File imageFile = getFile(imageUri);
		File tmpFile = new File(imageFile.getAbsolutePath() + TEMP_IMAGE_POSTFIX);
		boolean loaded = false;
		try {
			OutputStream os = new BufferedOutputStream(new FileOutputStream(tmpFile), bufferSize);
			try {
//				loaded = IoUtils.copyStream(imageStream, os, listener, bufferSize);
				// 同样的方式保存，未返回结果标志
				Bitmap bitmap = BitmapFactory.decodeStream(imageStream);
				save(imageUri, bitmap);
			} finally {
				IoUtils.closeSilently(os);
			}
		} finally {
			if (loaded && !tmpFile.renameTo(imageFile)) {
				loaded = false;
			}
			if (!loaded) {
				tmpFile.delete();
			}
		}
		return loaded;
	}

	@Override
	public boolean save(String imageUri, Bitmap bitmap) throws IOException {
		File imageFile = getFile(imageUri);
		File tmpFile = new File(imageFile.getAbsolutePath() + TEMP_IMAGE_POSTFIX);
		OutputStream os = new BufferedOutputStream(new FileOutputStream(tmpFile), bufferSize);
		boolean savedSuccessfully = false;
		try {
//			savedSuccessfully = bitmap.compress(compressFormat, compressQuality, os);
			// 重写图片保存
			NativeUtil.compressBitmap(bitmap, compressQuality,
					imageFile.getAbsolutePath(), true);
		} finally {
			IoUtils.closeSilently(os);
			if (savedSuccessfully && !tmpFile.renameTo(imageFile)) {
				savedSuccessfully = false;
			}
			if (!savedSuccessfully) {
				tmpFile.delete();
			}
		}
		bitmap.recycle();
		return savedSuccessfully;
	}
```



### maven .so 打包

需要把 armeabi/ 下的.so 打包成第三方 .jar 包。

##### 1. mvn install .so文件
先把 armeabi/ 文件夹连同下面的 .so 文件拷到当前项目组的 libs/ 下，然后用 install 的方式把环境配置到本地的 maven 环境。我当前有2个 .so 需要配置，命令如下：

**libbitherjni.so**

```
$ mvn install:install-file -DgroupId=net.bither -DartifactId=libbitherjni -Dversion=v1 -Dfile=/Users/cpp255/git/Android-Universal-Image-Loader/library/libs/armeabi/libbitherjni.so -Dpackaging=so -DgeneratePom=true -Dclassifier=armeabi  
```

**libjpegbither.so**

```
$ mvn install:install-file -DgroupId=net.bither -DartifactId=libjpegbither -Dversion=v1 -Dfile=/Users/mac-600672/git/Android-Universal-Image-Loader/library/libs/armeabi/libjpegbither.so -Dpackaging=so -DgeneratePom=true -Dclassifier=armeabi  

```

执行上述命令后会提示 success.

##### 2. 配置 pom.xml
按照步骤1的完成配置后，则可以在 pom.xml 文件中配置这两个包的信息了

**libbitherjni.so**

```
<dependencies>
	<dependency>
          <groupId>net.bither</groupId>
           <artifactId>libbitherjni</artifactId>
           <version>v1</version>
           <classifier>armeabi</classifier>
            <scope>runtime</scope>
            <type>so</type>
  	/dependency>
</dependencies>
```

**libjpegbither.so**

```
<dependencies>
	<dependency>
          <groupId>net.bither</groupId>
           <artifactId>libjpegbither</artifactId>
           <version>v1</version>
           <classifier>armeabi</classifier>
            <scope>runtime</scope>
            <type>so</type>
  	</dependency>
</dependencies>
```

然后执行打包命令，整个过程就结束了：

```
$ mvn clean package
```

修改的项目工程为: [cpp255/Android-Universal-Image-Loader](https://github.com/cpp255/Android-Universal-Image-Loader)


参考：
   
1.[How To Include Custom Library Into Maven Local Repository?](http://www.mkyong.com/maven/how-to-include-library-manully-into-maven-local-repository/)   

2.[使用maven构建android项目](http://keepcleargas.bitbucket.org/android/2014/04/01/using-maven-to-package-android.html)