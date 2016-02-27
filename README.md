Android studio 下robotium框架搭建（MAC版）
## 前言
本文档介绍MAC系统下Android studio 的基于robotium搭建的自动化测试框架，比起eclipse, 使用android studio更加便捷快速。

## 环境要求：
* Java 运行组件环境 (JRE) 6
* Java 开发工具包 (JDK) 7
* Android studio [最新版](http://developer.android.com/intl/zh-cn/sdk/index.html#Requirements)
* Android SDK
* Gradle 最新版（方便shell进行gradle命令行打包）

## 重签名被测试应用

### 重签名步骤

robotium要求测试app和被测试app需要有相同的签名，才能保证测试脚本的运行。所以我们要先对被测试app进行重签名。

对于没有任何签名信息的apk，这里可以默认使用.Android下的debug.keystore来重新签名apk。步骤如下：

* 查看apk的签名信息。使用java的jarsigner来查看apk是否签名。

 在终端输入：`jarsigner -verify -verbose –certs  /Users/***/test.apk` 

 下面的结果是已经签名了：

 ![screenshot1](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot24.png)
 
* 针对上面已签名的apk，删除apk的签名信息：将test.apk改名为test.zip包后，打开压缩包，把META-INF目录下的所有文件删除后，重新压缩文件。把test.zip文件改名成test.apk。再次查看test.apk的签名信息，会发现apk未签名。

* 对apk重新签名。在终端输入命令：

 `jarsigner  -verbose –sigalg SHA1withRSA -digestalg SHA1 -keystore debug.keystore -storepass "android"  -keypass “android” -signedjar 签名后.apk 源.apk androiddebugkey`

 注意在JDK1.7环境下必须加上 `–sigalg SHA1withRSA -digestalg SHA1`
 
 其中debug.keystore为系统.Android下的默认keystore，该密钥为android app默认使用的debug签名工具。其alias 是androiddebugkey，-storepass为android ，-keypass为 android

* 使用zipalign工具修正刚签名的apk包，使apk文件中未压缩的数据在4个字节边界上对齐（4个字节是一个性能很好的值）。工具位置为android sdk的tools目录中。进入该目录后执行命令：
  
  `./zipalign -v 4 源.apk 重命名.apk`

除了以上方法也可以使用被测试app原有的签名文件，对测试app进行签名。

以下是robotium官方文档关于签名的说明:

> Important Steps:

> 1.If you know the certificate signature then use the same signature in test project

> 2.If you do not know the certificate signature then delete the certificate signature and use the same androiddebug key signature in both the application and the test project

> 3.If the application is unsigned then sign the application apk with the android debug key

### 脚本实现自动化重签名
可以通过一段shell脚本实现app的重签名,而不需要手动一步步的去操作,将本地的debug.keystore(默认是在~/.android/debug.keystore路径下)与脚本放在同一个目录下，执行shell 脚本后生成apk包。（shell脚本请联系我索取）

## 创建Android studio 测试工程
### 创建测试工程
* 打开Android studio ，新建一个new project：输入application name和包名，选择存放路径后，下一步。
 **注意**: 包名必须和被测试apk的包名一致，如被测试包名是com.calculator， TestProject的名字最好为com.calculator，因为android studio build的时候会生成一个测试包com.calculator.test，这个包是用来识别为测试程序的

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot1.png)

* 设置安卓的最小SDK版本，点击下一步：

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot2.png)

* 因为这里是新建一个test工程，可以不用需要activity和界面。所以选择no activity后点击完成。一个基本的android studio项目已经完成，项目主要是通过gradle来构建apk的：

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot3.png)

### 修改build.gradle文件

* 进入app module的build.gradle文件

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot4.png)
 
* build.gradle文件中，dependencies增加robotium依赖包：

 ```gradle
     dependencies {
         androidTestCompile 'com.jayway.android.robotium:robotium-solo:5.5.4'
     }
 ```
 或者将robotium 的jar包复制到工程的libs目录下，右键点击jar包选择add as library

* 添加一个gradle任务,在测试工程debug build时候复制被测试的apk到项目的 build/outputs/apk/目录下,并且加入到assembleDebug任务的依赖：
 
 ```gradle
     task copyTask(type: Copy) {
         from '/path/to/com.netease.qa.testdemo.apk'
         into 'build/outputs/apk/'
         rename {
             'com.netease.qa.testdemo.apk'
         }
     }
     assembleDebug.dependsOn copyTask
 ```

* 修改applicationId为被测试app的包名（如果新建的测试包名不和被测试包名一致，则执行这一步）：

 ```
    defaultConfig {
        applicationId "com.netease.qa.testdemo"
        minSdkVersion 23
        targetSdkVersion 23
        versionName "1.0"
    }
 ```

 如果包名不一致，会出现找不instrumentation的目标包。
 
 修改.gradle文件后需要同步：

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot9.png)
 
* 运行robotium的测试用例

 新建的project会有主源代码和测试代码。它们分别在：

 `src/main/`

 `src/androidTest/`

 对androidTest下的ApplicationTest类，右键 Create Test

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot10.png)
 
 选择module/package和test方式
 
 instrumentation选择android.test.ActivityInstrumentationTestCase2

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot11.png)

### 测试工程签名
* 为保证被测试app和测试app有相同的签名（robotium要求测试包与被测试包需同样签名才能进行测试），可以在studio里面进行debug版本和release版本的签名配置。对测试的module右键，打开module配置后输入自己的key，密码等信息：

  ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot12.png)
  
  ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot13.png)
  
* 在Build Types中，debug选择刚刚配置的key后保存。

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot14.png)
 
 这样能够保证测试app与被测试app有一致的签名。
 
* 连接好真机或模拟器后，这时候直接run测试app，看看环境是否搭建成功。
 
 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot15.png)

### 编写测试用例
在test类下编辑测试用例，下面为一元夺宝简单的登陆退出操作。具体使用可参照robotium API

```java

package com.netease.mail.testoneduobaohydrid; 
import android.test.ActivityInstrumentationTestCase2; 
import com.robotium.solo.Solo; 
 
/** 
 * <a href="http://d.android.com/tools/testing/testing_android.html">Testing Fundamentals</a> 
 *@author Dawn  
 */ 
public class ApplicationTest extends ActivityInstrumentationTestCase2 { 
    private Solo solo; 
 
    private static final String LAUNCHER_ACTIVITY_FULL_CLASSNAME = "com.netease.mail.oneduobaohydrid.activity.LaunchActivity"; 
 
    private static Class<?> launcherActivityClass; 
    static{ 
        try { 
            launcherActivityClass = Class.forName(LAUNCHER_ACTIVITY_FULL_CLASSNAME); 
        } catch (ClassNotFoundException e) { 
            throw new RuntimeException(e); 
        } 
    } 
 
    @SuppressWarnings("unchecked") 
    public ApplicationTest() throws ClassNotFoundException { 
        super(launcherActivityClass); 
    } 
 
    public void setUp() throws Exception { 
        super.setUp(); 
        solo = new Solo(getInstrumentation()); 
        getActivity(); 
    } 
 
    @Override 
    public void tearDown() throws Exception { 
        solo.finishOpenedActivities(); 
        super.tearDown(); 
    } 
 
    public void testRun() { 
        //Wait for activity: 'com.netease.mail.oneduobaohydrid.activity.LaunchActivity' 
        solo.waitForActivity("LaunchActivity"， 2000); 
        if(solo.waitForActivity("IntroActivity")) { 
            assertTrue("IntroActivity is not found!"， solo.waitForActivity("IntroActivity")); 
            //Sleep for 167251 milliseconds 
            solo.sleep(2000); 
            //Click on 继续 
            solo.clickOnView(solo.getView("intro_btn1")); 
            //Sleep for 563 milliseconds 
            solo.sleep(563); 
            //Click on 开始1元夺宝 
            solo.clickOnView(solo.getView("intro_btn2"， 1)); 
        } 
        solo.sleep(2000); 
        //Wait for activity: 'com.netease.mail.oneduobaohydrid.activity.MainActivity' 
        assertTrue("MainActivity is not found!"， solo.waitForActivity("MainActivity")); 
        //Sleep for 2112 milliseconds 
        solo.sleep(1112); 
        //Click on 我的 
        solo.clickOnView(solo.getView("tab_wrapper4")); 
        //Sleep for 758 milliseconds 
        solo.sleep(758); 
        //Click on Empty Text View 
        solo.clickOnView(solo.getView("email")); 
        //Sleep for 5049 milliseconds 
        solo.sleep(2049); 
        //Enter the text: 邮箱账号 
        solo.clearEditText((android.widget.EditText) solo.getView("email")); 
        solo.enterText((android.widget.EditText) solo.getView("email")， "***"); 
        solo.sleep(1831); 
        //Enter the text:密码 
        solo.clearEditText((android.widget.EditText) solo.getView("password")); 
        solo.enterText((android.widget.EditText) solo.getView("password")， "****"); 
        //Click on 登录 
        solo.clickOnView(solo.getView("login")); 
        //Sleep for 4016 milliseconds 
        solo.sleep(4016); 
        //Click on 夺宝 
        solo.clickOnView(solo.getView("tab_wrapper0")); 
        //Sleep for 2766 milliseconds 
        solo.sleep(2766); 
        //Click on 我的 
        solo.clickOnView(solo.getView("tab_wrapper4")); 
        solo.sleep(2000); 
        solo.clickOnView(solo.getView("user_layout_setting")); 
        //Wait for activity: 'com.netease.mail.oneduobaohydrid.activity.SettingsActivity' 
        assertTrue("SettingsActivity is not found!"， solo.waitForActivity("SettingsActivity")); 
        //Sleep for 1185 milliseconds 
        solo.sleep(1185); 
        //Click on 退出登录 
        solo.clickOnView(solo.getView("logout")); 
        //Sleep for 2236 milliseconds 
        solo.sleep(2236); 
        //Click on 夺宝 
        solo.clickOnView(solo.getView("tab_wrapper0")); 
    } 
}
```

robotium的自动化测试可以巧用junit进行用例编写，这里不详细介绍了。

## monitor控件识别
编写用例时需要使用控件的id值，可使用控件识别工具Monitor或者Hieriarcgy Viewer查找(均在Android sdk安装路径tools目录下，建议使用Monitor，更快更稳定，如果打不开则考虑后者) 
如下图是打开Monitor的界面，左边栏显示的时已连接设备的信息， 右侧显示截屏后定位到的元素信息：
com.netease.mail.oneduobaohydrid:id/tab_wrapper4  可知“我的”控件的Id值为tab_wrapper4

 ![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot16.png)


## 运行测试用例（studio 模式及adb shell模式）
android studio 下直接对AndroidTest进行run,前面已经介绍过了
### adb shell模式下进行robotium自动化测试[3]：
* **先查看连接的安卓机**：

 ```bash
 DawndeMacBook-Por:~ Dawn$ adb devices
 List of devices attached
 HC4AXMZ01269	device
 ```

* **安装测试app**：

  ```bash
  DawndeMacBook-Por:~ Dawn$ adb install /Users/Dawn/TestDemo/app/build/outputs/apk/app-debug-androidTest-unaligned.apk
  1230 KB/s (67244 bytes in 0.053s)
  		pkg: /data/local/tmp/app-debug-androidTest-unaligned.apk
  Success
  ```

* **安装被测试app**:

 ```bash
 DawndeMacBook-Por:~ Dawn$ adb install /Users/Dawn/TestDemo/app/build/outputs/apk/app-debug.apk
 2307 KB/s (1148386 bytes in 0.485s)
 		pkg: /data/local/tmp/app-debug.apk
 Success
 ```
 
* **使用下面adb明天查看手机project包名对应的instrumentation后，找到我们测试的project包名**：
   `adb shell pm list instrumentation`
   
 ```bash
 DawndeMacBook-Por:~ Dawn$ adb shell pm list instrumentation
 instrumentation:com.futuredial/android.test.InstrumentationTestRunner(target=com.futuredial)
 instrumentation:com.netease.mail.onduobaohydrid.test/android.test.InstrumentationTestRunner (target=com.netease.mail.oneduobaodrid)
 ```
 
* **然后运行命令**：
`adb shell am instrument -w com.netease.mail.oneduobaohydrid.test/android.test.InstrumentationTestRunner`
然后就可以在真机看到运行的过程及结果：

 ```bash
 DawndeMacBook-Por:~ Dawn$ adb shell am instrument -w com.netease.mail.oneduobaohydrid.test/android.test.InstrumentationTestRunner
 
 come.netease.qa.testdemo.ApplicationTest:
 Test results for InstrumentationTestRunner=
 Timer:42.91
 
 OK (1 test)
 ```

## 结合阿里云测平台进行脚本测试
上面调试运行通过后，可以上传到阿里云测平台进行部分机器的自动化测试了哦：
http://mqc.aliyun.com
部分运行截图：

![screenshot](https://raw.githubusercontent.com/hcnode/robotium-android-studio-handbook/master/screenshot/screenshot23.png)

## 参考资料
* 为方便Robotium自动测试需要对apk用本地安卓sdk中的debug.keystore进行重新签名： http://stephen830.iteye.com/blog/2079101 
* Robotium with Android studio :
http://stackoverflow.com/questions/23275602/robotium-with-android-studio/23295849  
* TestAndroidCalculatorAPK-BlackBoxTesting-V2_0.pdf :
https://github.com/RobotiumTech/robotium/wiki/Robotium-Tutorials

