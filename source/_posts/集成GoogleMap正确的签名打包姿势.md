---
title: 集成GoogleMap正确的签名打包姿势
date: 2017-12-22 20:42:39
tags:
---

在集成了Google Map之后，在debug模式下测试一切正常，地图正常显示，都是签名打包后，运行app发现地图不显示了
<!--more-->

![debug模式正常显示](http://img.blog.csdn.net/20171222205018389?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

![签名打包后release模式地图不显示](http://img.blog.csdn.net/20171222205039360?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

stackoverflow一番后，[这是解决方式](https://stackoverflow.com/questions/30298223/google-maps-signed-apk-android)

原因就是，申请的google map API_KEY是放在了debug文件夹下，没有对应的release的 API_KEY
![这里写图片描述](http://img.blog.csdn.net/20171222205946398?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvR2FvX3l1eXU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


解决方式

1、build.gradle下添加manifestPlaceholders字段值
```Java
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            manifestPlaceholders = [ map_key:"AIzaSyAFhhAGX_9cW87jrdz06uDo96iweddwBl4" ]
        }
        debug {
            manifestPlaceholders = [ map_key:"AIzaSyAFhhAGX_9cW87jrdz06uDo96iweddwBl4" ]
        }
    }
```
2、修改注册文件
```Java
        <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="${map_key}"/>
```
再次签名打包运行，就OK了

仔细看Google Map的集成过程，其实都有Google官方的爱心指导，只是我当初没仔细看而已！
```xml
google_maps_api.xml

<resources>
    <!--
    TODO: Before you run your application, you need a Google Maps API key.

    To get one, follow this link, follow the directions and press "Create" at the end:

    https://console.developers.google.com/flows/enableapi?apiid=maps_android_backend&keyType=CLIENT_SIDE_ANDROID&r=9B:03:08:37:67:8F:08:E8:D6:F5:ED:66:42:43:2E:EF:58:48:E0:C4%3Bcom.gaoyy.delivery4res

    You can also add your credentials to an existing key, using this line:
    9B:03:08:37:67:8F:08:E8:D6:F5:ED:66:42:43:2E:EF:58:48:E0:C4;com.***.d******s

    Alternatively, follow the directions here:
    https://developers.google.com/maps/documentation/android/start#get-key

    Once you have your key (it starts with "AIza"), replace the "google_maps_key"
    string in this file.
    -->
    <string name="google_maps_key" templateMergeStrategy="preserve" translatable="false">AIzaSyAFhhAGX_9cW87jrdz06uDo96iweddwBl4</string>
</resources>

AndroidManifest.xml
<!--
     The API key for Google Maps-based APIs is defined as a string resource.
     (See the file "res/values/google_maps_api.xml").
     Note that the API key is linked to the encryption key used to sign the APK.
     You need a different API key for each encryption key, including the release key that is used to
     sign the APK for publishing.
     You can define the keys for the debug and release targets in src/debug/ and src/release/. 
-->
<meta-data
    android:name="com.google.android.geo.API_KEY"
    android:value="${map_key}"/>
```

解决方式3：还是按照google官方的建议来，生成2个key，一个用于debug，一个用于release，签名打包。






