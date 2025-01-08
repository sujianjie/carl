## Framework

1、view层级方法 WindowManagerPolicy.java   getWindowLayerFromTypeLw      																		    		frameworks/base/services/core/java/com/android/server/policy/WindowManagerPolicy.java

2、每一个Activity上面都有一个Window，可以通过getWindow获取  

​	Activity >  PhoneWindow > DecorView > TitleView + ContentView

​	(1)Window   ViewManager接口定义了addView / updateViewLayout / removeView方法

​	(2)WindowManager 接口继承ViewManager，应用通过wm来管理Window，实现类为WindowManagerImpl

​	(3)WindowManagerImpl 内部实现由WindowManagerGlobal完成     WindowManagerGlobal单例

​	(4)ViewRootImpl 一个window对应一个ViewRootImpl 和View

​	activity在onResume的时候显示，获取window decorView，wm add decorView 

​	https://blog.csdn.net/yimelancholy/article/details/130339779

​	wms窗口流程

​	当Activity.onResume()被调用之后，客户端会与WMS进行通信将我们的布局显示在屏幕上。主要过程：
​	1、客户端通知WMS创建一个窗口，并添加到WindowToken。即addToDisplayAsUser阶段。
​	2、客户端通知WMS创建Surface，并计算窗口尺寸大小。即relayoutWindow阶段。
​	3、客户端获取到WMS计算的窗口大小后，进一步测量该窗口下View的宽度和高度。即performMeasure阶段。
​	4、客户端确定该窗口下View的尺寸和位置。即performLayout阶段。
​	5、确定好View的尺寸大小位置之后，便对View进行绘制。即performDraw阶段。
​	6、通知WMS，客户端已经完成绘制。WMS进行系统窗口的状态刷新以及动画处理，并最终将Surface显示出来。即	reportDrawFinished阶段。

3、touchevent传递流程 [Android事件分发机制（一）——ViewRootImpl篇_viewrootimpl接受事件吗?-CSDN博客](https://blog.csdn.net/dongxianfei/article/details/83863888?ops_request_misc=&request_id=&biz_id=102&utm_term=viewrootimpl&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-2-83863888.nonecase&spm=1018.2226.3001.4187)

​	InputEventReceiver->ViewRootImpl->DecorView->Activity->PhoneWindow->View

4、DisplayContent 管理显示设备上的内容包括窗口surface动画等。DisplayId用来标识不同的显示设备，唯一

​	  WindowManager.LayoutParams Token ->  WindowToken 1、标识窗口 2、确定窗口层级 3、窗口交互，多窗口模式管理窗口的显示和隐藏

​	   WindowState 是表示窗口状态的类，描述窗口的状态和属性。1、窗口类型type 2、窗口位置和大小 3、窗口状态 是否可见、是否聚焦等 4、窗口属性 窗口的透明度，背景色、动画效果等 5、窗口层级   **控制和管理窗口的状态**  WindowState中的client为IWindow，将wms操作回调给ViewRootImpl

​		DisplayPolicy 管理和控制显示策略 1、屏幕方向  2、屏幕亮度管理  3、屏幕锁定管理 4、虚拟按键管理

```
if (!WindowInspector.getGlobalWindowViews().contains(mHikNavigationBarView)) {
```

5、判断应用是否处于全屏模式或者分屏等

frameworks/base/core/java/android/app/WindowConfiguration.java    

```
InformationWrapperController.getInstance(context).getTaskWindowingMode(topActivityPackageName) == WindowConfiguration.WINDOWING_MODE_FULLSCREEN
```



## SystemUI 与 Launcher3 AIDL通信

**1、**在SystemUI模块调用CommandQueue的showRecentApps方法可以实现最近应用列表的显示。

​		frameworks/base/packages/SystemUI/src/com/[android](https://so.csdn.net/so/search?q=android&spm=1001.2101.3001.7020)/systemui/statusbar/CommandQueue.java

​		showRecentApps方法会调用Handler发送类型为MSG_SHOW_RECENT_APPS的消息。

​		Handler对象在接收到MSG_SHOW_RECENT_APPS消息之后，会调用所有回调对象的showRecentApps方法。

**2、**类型为Recents的SystemUI组件实现了CommandQueue.Callbacks接口，并将自身添加到了CommandQueue的监听回调对象集合中，这样CommandQueue的showRecentApps方法会触发Recents组件的showRecentApps方法。

​		Recents的showRecentApps方法会进一步调用RecentsImplementation的showRecentApps方法。RecentsImplementation是一个		接口。在SystemUI模块的具体实现类是OverviewProxyRecentsImpl。

​		OverviewProxyRecentsImpl的showRecentApps方法会进一步调用代理者IOverviewProxy的onOverviewShown方法，IOverviewProxy是一个aidl。这里调用OverviewProxyService的getProxy方法为overviewProxy赋值。mOverviewProxyService最初是通过Dependency进行赋值的。

​		OverviewProxyService类和getProxy方法 

​		frameworks/base/packages/SystemUI/src/com/android/systemui/recents/OverviewProxyService.java

​		OverviewProxyService在初始化的时候 

​				（1）为最近应用列表组件名称mRecentsComponentName赋值。

​				（2）创建最近应用列表Activity的意图对象mQuickStepIntent

​				（3）创建最近应用列表服务的意图对象launcherServiceIntent，并进行服务绑定，触发mOverviewServiceConnection的回调方法拿到IOverviewProxy对象，为后续跨进程通信做准备。

​				（4）重新我们在前面第5步提到的OverviewProxyRecentsImpl的showRecentApps方法，该方法最终就是调用IOverviewProxy的onOverviewShown方法进行跨进程通信告知Launcher3唤起最近应用列表的。

**3、** Launcher3模块 ，SystemUI模块在创建OverviewProxyService实例对象的时候，会绑定Launcher3模块的一个服务，该服务		完整声明	

```xml
<application android:backupAgent="com.android.launcher3.LauncherBackupAgent">
        <service android:name="com.android.quickstep.TouchInteractionService"
             android:permission="android.permission.STATUS_BAR_SERVICE"
             android:directBootAware="true"
             android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.QUICKSTEP_SERVICE"/>
            </intent-filter>
        </service>
    </application> 
```

​	调用CommandQueue的showRecentApps方法显示最近应用列表，该方法最终调用的其实是IOverviewProxy的	onOverviewShown。
​	这个IOverviewProxy aidl的 void onOverviewShown(boolean triggeredFromAltTab) = 7;

**4、** Launcher3模块TouchInteractionService服务的TISBinder对象实现了这个aidl。	

​		public class TISBinder extends IOverviewProxy.Stub {

​		TISBinder对象的onOverviewShown方法会进一步调用OverviewCommandHelper对象的addCommand方法，传入TYPE_SHOW。

​		

## Binder

1、server ->  Stub  - Parcelable  -  Proxy <- client

2、AIDL 最终目标是跨进程访问，与接口类似，本质都是**Interface**，AIDL文件是IInterface，而Interface是继承自Interface的），而且都只定义了抽象方法，没有具体的实现，需要子类去实现。

与接口不同的是：由AIDL生成的stub类本质上是一个Binder这个类所生成的对象有两种方式可以传递给另外一个进程：

（1）一种是通过bindService的方式，绑定一个服务，而在绑定后，服务将会返回给客户端一个Binder的对象，此时可以把继承自stub的Binder传递给客户端。

（2）另外一种就是把继承自stub的类提升为系统服务，此时，我们通过ServiceManager去得到当前的系统服务，ServiceManager就会把目标Service的Binder对象传递给客户端。

```java
IBinder b = ServiceManager.getService("com.hikvision.media.touchcontrol");
```

客户端拿到服务端的唯一标志IBinder对象，即可通过这个对象的transact方法向服务端发起请求。



服务端要支持远程调用，从Binder类派生一个类。Binder类继承自IBinder，Binder类里有一个onTransact()方法。服务端通过这个方法响应客户端的请求。|   asInterface(IBinder obj)  将服务端的 Binder对象生成客户端所需的AIDL接口类型对象，这种转换	过程是区分进程的，如果位于同一进程，返回的就是Stub 对象本身，否则返回的是系统封装后的Stub.proxy对象。

asBinder()方法返回IBinder对象。   TouchService实现服务

```java
public class TouchService extends ITouchService.Stub
```

3、C++ AIDL   Bn为服务端，Bp为客户端

​	ICameraService.aidl会生成三个.h文件，分别是ICameraService.h、BnCameraService.h、BpCameraService.h。

​	ICameraService.h有一个ICameraService类，该类是一个接口类，其中定义的接口和枚举都是AIDL中定义的内容。class 		     	ICameraService : public ::android::IInterface 接口类继承于IInterface类，IInterface又继承于RefBase

​	BnCameraService继承于BnInterface BnInterface继承两个类，一个是它的模板（即Binder机制中的接口类），另外一个是  	   	BBinder，为Binder通信而存在

​	BpCameraService继承自BpInterface BpInterface又继承自它的模板类（接口类）和BpRefBase。 	Bn端Binder通信的类是		   	BBinder，而Bp端关于Binder通信的类则是BpRefBase。

写一个可执行程序，将它编译成可执行程序文件 main_hikinfraredbox.cpp

​	instantiate()调用BinderService 的instantiate()函数，主要作用就是将TestService注册到ServiceManager里面

## Linux

1、搜索文件 find -name xxx.java

2、搜索关键字 grep -nri xxx ./

3、du -sh <目录>  查看指定目录大小

4、tar -zcvf test.tar.gz ./test/    压缩指定文件夹 test 压缩为 test.tar.gz  

​	tar -xzvf test.tar.gz  解压缩 test.tar.gz压缩文件

5、sudo apt install --reinstall <package_name>  

## Android系统

1、手动启动init.rc中服务  setprop ctl.start touchframeservice

2、调出原生设置  am start com.android.settings/com.android.settings.Settings

3、原生TV设置  am start -n com.android.tv.settings/com.android.tv.settings.MainSettings

4、通过包名启动/activity启动

```java
  Intent LaunchIntent = getPackageManager().getLaunchIntentForPackage("com.package.address");
  startActivity(LaunchIntent);
```

```java
  Intent intent = new Intent(Intent.ACTION_MAIN);
  intent.setComponent(new ComponentName("com.package.address","com.package.address.MainActivity"));
  startActivity(intent);
```
4、adb发广播  am broadcast -a android.intent.action.TOOL_BOX_START --ei toolName 23     -a  action

​	--es string类型extra data   --ez boolean  --ei int 类型extra data

4、打开adb  settings put global adb_enabled 1   |   setprop persist.sys.usbmode 1

5、查看上报input事件  getevent -lt

6、查看Android帧率  setprop debug.sf.fps  1 ,  logcat -c  ,  logcat | grep mFps

7、查看当前Soc声卡状态   cat /proc/asound/cards        ls  /proc/asound/card0  查看输入输出通道  Capture输入，Playback输出

8、cat /proc/asound/pcm 查看pcm设备列表

9、tinypcminfo -D 3 -d 0 查看pcm通道信息

10、**tinycap** mnt/sdcard/test.pcm -D 3 -d 0 -c 1 -r 16000 -b 16 -p 240 -n 10 采集音频

 -D card 声卡/-d device 设备/ -c channels 通道/ -r rate 采样率/ -b bits pcm位宽/ -p period_size 一次中断帧数/ -n n_periods   周期数

11、**tinyplay** storage/0018-F8BD/dukou_48k.wav -D 1 -d 0   播放音频   指定声卡1播放

12、ps -A 查看当前运行的服务  ps -A | grep hmm

13、dumpsys activity a  --->  dumpsys activity xxxactivity   /   dumpsys window a 

14、拉出android系统中已经安装的应用 adb shell pm list package  /  adb shell pm path <包名>  /  adb pull xxx

15、logcat | grep -i (忽略大小写) -E (过滤多个关键词)    查看崩溃打印 logcat -b crash  

16、查看应用版本号 dumpsys package display.interactive.screensaver | grep version

17、查看是否有此应用  pm list package | grep screensaver

18、super分区包含 system vendor product odm 查看打包生成super分区大小

19、v4l2抓取图像  v4l2-ctl -d /dev/video5 --set-fmt-video=width=640,height=360,pixelformat=MJPG --stream-mmap=4  --stream-count=50 --stream-to=/sdcard/test --verbose

查看当前占用video占用节点  lsof  /dev/video*

20、录屏命令screenrecord   screenrecord --size 1280x720 --bit-rate 6000000 --time-limit 30 /sdcard/demo.mp4 

​	--size 指定视频分辨率  --bit-rate 指定视频比特率，默认为4M，该值越小，保存的视频文件越小； 

​    --time-limit 指定录制时长，若设定大于180，命令不会被执行；

21、eventlog中有activity的生命周期打印

22、dumpsys window | grep -i focused 查看焦点持有者

23、adb enable-verity 删除adb remount分区   lpdump

24、logcat --pid=

25、设置沉浸式 

```
getWindow().getDecorView().getWindowInsetsController().hide(WindowInsets.Type.navigationBars());
getWindow().getDecorView().getWindowInsetsController().setSystemBarsBehavior(3);
```

26、svg 转换为xml资源 文件 as 中drawable new vector asset 生成 xml    @drawadble/xxxxx.xml

## Android按键

根据vid和pid来找对应的kl文件  kl文件是对应linux keycode和keyevent.java中的映射关系表

getevent -l  看是什么事件  例如 input1  // dumpsys input

cat proc/bus/input/devices 找到对应input event1的vid 和pid  Bus=0010 Vendor=1b8e Product=0001 对应kl文件

## Hikvision

1、设备序列号生成加密文件网站:  http://10.19.219.105:8000/

2、IE代理  10.1.204.246:8080 

3、代码审核提交工具  git fmt-commit -m "fix:"      export GIT_SSL_NO_VERIFY=1

4、切中控串口hex 20 20 53 45

5、串口升级 开机 u 进入mcu升级模式 1，工具->传输二进制->发送Ymodem

6、adb shell  hcs-t -spls 0

7、代码注释率 https://software.hikvision.com/version/resources/list/detail?type=code-review

8、海外网络路由器ip端口  IP10.42.99.253  掩码 255.255.255.0 网关 10.42.99.254  dns 10.1.204.77

新 IP 10.40.3.40   掩码 255.255.255.0    网关10.40.3.254   dns 10.1.208.77

9、lunch Interactive_White_Board-userdebug
	./build.sh -AUCKu -d rk3588-evb7-lp4-v10-hik-p1009
	source build/envsetup.sh

10、https://psh.hikvision.com.cn/query-tools/psh-shell

## 工具

1、Android源码导入 Android Studio :  https://blog.csdn.net/u012514113/article/details/124211601?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522169141107416800211555545%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=169141107416800211555545&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-124211601-null-null.142^v92^chatgptT3_1&utm_term=%E6%BA%90%E7%A0%81%E5%AF%BC%E5%85%A5android%20studio&spm=1018.2226.3001.4187

2、Ctrl + F12 在当前类快速查找方法属性

2、合入补丁命令：patch -p1 < ../..xxx00001.xxx 

3、dmesg -n 1  串口停止输出

4、frameworks/base/services/Android.bp下的optimize的true都改成false   frameworks/base/packages/SystemUI/Android.bp

​	build\core\proguard.flags混淆配置忽略了android开头的包名   build/make/core/proguard_basic_keeps.flags 

```
-keep public class android.** { *; }  -keep class android.*   -keep class android.**
```

5、android 14 debug进程为空的问题： 1.  setprop persist.debug.dalvik.vm.jdwp.enabled 1    2.reboot

6、拉取线上打包  source ci/build/getOnlineAPK.sh https:.......git

7、CertUtil -hashfile C:\TEMP\MyDataFile.img MD5

## EDLA认证

韩国ESOL的 fingerprint  ESOL/HCB_5186/Interactive_White_Board:13/TQ2A.230305.008.F1/20231114:user/release-keys

ESOL1.5 EDLA认证名称：HCB_5186            eSOL 2.0 EDLA认证名称：HCB_52series

​	        ESOL/HCB_52series/Interactive_White_Board:13/TQ2A.230305.008.F1/20240523:user/release-keys

海康过认证 fingerprint   HIKVISION/DS_D5C65RB_A/DS_D5C65RB_A:13/TQ2A.230305.008.F1/20230908:user/release-keys

罗马尼亚65寸fingerprint HORIZON/ID65HZA3B1_C/Interactive_White_Board:13/TQ2A.230305.008.F1/20240103:user/release-keys

基线edla认证显示状态问题 1、fingerprint是否一致  2、安全补丁日期  3、build_number  4、build.type .... flavor ... description

**锁死fingerprint**：保证fingerprint与认证版本一致，安全补丁等级ro属性一致。build_number一致 gms package_version

**注意** 设备提交信息中SKU需要和 product_name 保持一致          查看当前设备的feature      pm list features

修改ro.build.fingerprint定义在 framework/base/core/java/android/os/Build.java 

​	  ro.bootimage.build.fingerprint &  ro.vendor.build.fingerprint     build/make/core/sysprop.mk

EDLA认证中Apex签名问题修改：使用RK脚本，init_var.sh修改为hik属性
	CN，Zhejiang，Hangzhou，HIKVISION，hikvision@hikvision.com



14 DS_D5C   HIKVISION/DS_D5C65RB_A/DS_D5C65RB_A:14/UQ1A.240205.004.B1/20241016:user/release-keys

[ro.build.description]: [DS_D5C65RB_A-user 14 UQ1A.240205.004.B1 20241016 release-keys]

[ro.build.flavor]: DS_D5C65RB_A-user

[ro.build.display.id]: [UQ1A.240205.004.B1 release-keys]

14罗马尼亚65

HORIZON/ID65HZA3B1_C/Interactive_White_Board:14/UQ1A.240205.004.B1/20241024:user/release-keys

ro.build.description       Interactive_White_Board-user 14 UQ1A.240205.004.B1 20241024 release-keys

ro.build.flavor       Interactive_White_Board-user

ro.build.display.id     UQ1A.240205.004.B1 release-keys



#### 测试方法

1、单项跑测指令： run gts -m 模块名  -t  case -s 设备号 

2、retry命令  run retry --retry session_id        l  r 查看当前测试历史session_id

3、压测命令200次   run cts --include-filter "模块+空格+case" -s 设备号 --retry-strategy RETRY_ANY_FAILURE --max-testcase-run-count 200

```
run cts --include-filter "arm64-v8a CtsOsHostTestCases[instant] android.os.cts.StaticSharedLibsHostTests#testPruneUnusedStaticSharedLibraries_reboot_instantMode"
```

4、跑测崩溃，retry测试未跑完的模块 =

​	run retry -r   + session_id  -s  设备号 --retry-type not_executed

## RK

1、0591 83991906 座机电话

2、获取编译环境变量属性值  get_build_var xxx

3、OTA升级失败 报错 failed verify whole-file signature  删除U盘升级包或者格式化U盘重新拷贝

4、strings libKeyboxUtil.so  | grep lock 查看编译生成物so中是否包含 

5、linux FTP： ftp ftp.rock-chips.com  name  xzj-guest/  password   V3qTwMQ4YV

\## (可选)对于分区空间紧张的设备，可以先执行本条命令删除product分区后再烧写

GSI 

fastboot delete-logical-partition product ## 

对于使用A/B，虚拟A/B的设备，需要同时删除product_a/product_b 

fastboot delete-logical-partition product_a 

fastboot delete-logical-partition product_b 

fastboot flash system system.img ## 

烧misc.img恢复出厂设置(否则机器开机可能卡在加载界面)，misc是固件一起编译出来的 

fastboot flash misc misc.img ## 

VTS测试：GKI固件烧写vendor_boot-debug.img到vendor_boot分区；非GKI固件烧写boot-debug.img到boot分区 

fastboot flash vendor_boot vendor_boot-debug.img ## 

烧写成功后，重启 fastboot reboot



## 其他

teams账号:     hikifpd@teams365.in      HIKhik@2017

1、USB HUB 连接host和device之间扩展usb接口的集线器，将一个usb上行接口扩展为多个下行接口，使一个host同时连接多个device，上行端口面向host，下行端口面向device，下行hub提供了设备接入检测和设备移除检测能力，并给下行端口供电。

2、打印混淆字段：frameworks/base/services/Android.bp下的optimize的true都改成false，再把\build\core\proguard.flags混淆配置忽略了android开头的包名，其实选一种应该就可以了

3、vscode shift+alt 进入选择模式，鼠标左键一键选择删除



## Camera 框架

Camera App：应用层处于整个框架的顶端，承担着于用户直接进行交互的责任，承接来自用户直接或者间接的比如预览/拍照/录像等一系列具体需求，一旦接收到用户相关UI操作，便会通过Camera Api2标准接口将需求发送至Camera Framework部分，并且等待Camera Framework回传处理结果。

Camera2 Api：该层主要位于Camera App与Camera Service之间，以jar包的形式运行在App进程中，它封装了Camera Api2接口的实现细节，暴露接口给App进行调用，进而接收来自App的请求，同时维护着请求在内部流转的业务逻辑，最终通过调用Camera AIDL跨进程接口将请求发送至Camera Service中进行处理，紧接着，等待Camera Service结果的回传，进而将最终结果发送至App。

CaemraService：该层位于Camera Framework与Camera Provider之间，作为一个独立进程存在于Android系统中，在系统启动初期会运行起来，它封装了Camera AIDL跨进程接口，提供给Framework进行调用，进而接收来自Framework的图像请求，同时内部维护着关于请求在该层的处理逻辑，最终通过调用Camera HIDL跨进程接口将请求再次下发到Camera Provider中，并且等待结果的回传，进而将结果上传至Framework中。

CameraHAL（CameraProvider）：该层位于Camera Service与Camera Driver之间，CameraProvider作为一个独立的进程存在于Android系统中，同时在系统启动初期被运行，提供Camera HIDL跨进程接口供Camera Service进行调用，封装了该接口的实现细节，接收来自Service的图像请求，并且内部加载了Camera HAL Module，该Module由OEM/ODM实现，遵循谷歌制定的标准Camera HAL3接口，进而通过该接口控制Camera HAL部分，最后等待Camera HAL的结果回传，紧接着Provider通过Camera HIDL接口将结果发送至Camera Service。



SurfaceTexture java中mSurfaceTexture设置GLConsumer对象地址

SurfaceTexture java中mProducer设置producer对象地址

创建surface成功之后会回调onSurfaceTextureAvailable方法

## 底部栏待解决问题

1、锁屏结束后底部栏显示异常 attachtowindow，但是不显示  （规避方案：延迟两秒显示底部栏）

2、批注通信

3、3.4.0断电重启底部栏消失 attach   、 最近进程切换分屏应用  、  窗口化切换用户偶现锁屏出现底部栏

## JAVA

1、"常量".equals(变量) 避免空指针问题

## SystemUI

单编SystemUI     make SystemUI    

配置文件 framework-res.apk      framework/base/core  --- framework.jar       framework/base/service --- service.jar

home 键的监听应用层一般都是监听Intent.ACTION_CLOSE_SYSTEM_DIALOGS 这个广播来实现的.SystemUI 也是采用.

IStatusBar.aidl   rameworks/base/core/java/com/android/internal/statusbar    是framework和SystemUi通信接口

