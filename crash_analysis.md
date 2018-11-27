# 前言
当app crash时，会在iPhone的var/mobile/Library/Logs/CrashReporter自动生成一个崩溃日志。那如何让测试人员快速获得并初步分析log，以便帮助开发快速定位问题呢？下面就跟小白一起学习一下吧。  
通过下面的学习，将学习到常见的崩溃日志案例，及如何获取崩溃日志，如何符号化，从日志追踪到代码。

# 1 获取crash日志
* 通过app接入的第三方平台fabric查找本次操作的日志
* 进入iOS设置->隐私->分析->分析数据
* 通过Xcode导出日志并符号化，即Xcode->window->Devices->view Device Logs。下面是符号化的步骤
     1. 导出crash log
     2. 导出.dSYM文件，解压缩
     3. 打开命令行窗口 
     4. 输入``` /usr/bin/xcode-select-print-path```修复Xcode路径
     5. 查找symbolicatecrash命令所在路径：```find /Applications/Xcode.app -name symbolicatecrash -type f ```
     6. 输入上步骤看到的``` /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/Library/PrivateFrameworks/DVTFoundation.framework/symbolicatecrash```
     7. 上述命令后需跟俩参数，一是崩溃日志，二是dSYM文件。
     
     **具体见[ios crash log符号化](http://www.cocoachina.com/industry/20140514/8418.html)**
     

* 手机连接电脑通过命令行导出. 
``` 1.如果是Mac：～／Library/Logs/CrashReporter/MobileDevices/<device-name> ```. 
``` 2.如果是wind C://Users/<username>/AppDataRoamingApple/ComputerLogsCrashReporterMobileDevice/<device-name>```  


# 2 发生crash的场景  
发生crash主要存在两大类行为：1.应用违反操作系统规则；2.应用中有bug
## 2.1 违反操作系统规则的行为
* **启动，恢复，挂起，退出时watchdog超时**  

  1. 从iOS 4.x开始，退出应用时，应用不会立即终止，而是退到后台。但是，如果你的应用响应不够快，操作系统有可能会终止你的应用，并产生一个崩溃日志。  
  2. 如果你没有把需要花费时间比较长的操作(如网络访问）放在后台线程上就很容易发生这种情况。

* **用户强制退出**  
  1. iOS 4.x开始支持多任务。如果应用阻塞界面并停止响应， 用户可以通过在主屏幕上双击Home按钮来终止应用。此时，操作应用将生成一个崩溃日志。  
  2. 双击Home按钮后，你将看到运行过的所有应用。那些应用不一定是正在运行，也不一定是被挂起
* **低内存终止**  
  1. 当内存使用达到一定程度时，操作系统将发出UIApplicationDidReceiveMemoryWarningNotification 通知。同时,调用 didReceiveMemoryWarning 方法。
 此时，为了让应用继续正常运行，操作系统开始终止在后台的其他应用以释放一些内存。所有后台应用被终止后，如果你的应用还需要更多内存，操作系统会将你的应用也终止掉，并产生一个崩溃日志。而在这种情况下被终止的后台应用，不会产生崩溃日志。
 
## 2.2 应用中的bug


# 3 crash文件结构

首先，获得设备和crash的概况信息  
再次，分析异常信息，定位crash发生在哪个线程上  
最后，深入分析线程信息  
## 3.1 Process & Basic Information
![crash log 头部信息] (/Users/huxiaoyan/Downloads/crash信息头部.png)
每行信息介绍：  
1.crash ID；  2.crash设备ID ； 3.手机型号 ； 4.App名称 ； 5.App位置  ；6.bundleID;   7.版本号 ； 8 codetype：当前app的CPU架构   
## 3.2 Exception
``` Exception Type:  EXC_BAD_ACCESS (SIGBUS)```
```Exception Subtype: KERN_PROTECTION_FAILURE ```  
### 常见的exception Type
 1. EXC_BAD_ACCESS：访问了不该访问的内存导致
    * 补充信息SIGSEGV：由于重复释放对象导致
    * 补充信息SIGABRT：收到Abort退出导致（如插入nil到数组）
    * 补充信息SEGV(Segmentation Violation) ：无效的内存地址（如空指针，栈溢出，未初始化）
    * 补充信息SIGBUS：总线访问异常
    * 补充信息SIGILL：执行非法指令异常（可能是不被识别或没有权限）
 2. EXC_BAD_INSTRUCTION：线程执行非法指令导致
 3. EXC_ARITHMETIC：除零导致

### 常见的exception subtype
* 0xbaaaaaad：非真正的Crash，仅是包含整个系统某一时刻的运行状态
* 0xbad22222：VOIP程序在后台太过频繁的激活时，系统可能会终止此类程序
* 0x8badf00d：程序启动或者恢复时间过长被watch dog终止(eat bad dog)
* 0xc00010ff：执行大量耗费CPU／GPU运算，导致设备过热，触发过热保护被系统终止
* 0xdead10cc：程序退到后台时还占用系统资源，如通讯录被系统终止(dead lock)
* 0xdeadfa11：程序无响应用户强制关闭(dead fail)

## 3.3 Thread
### Thread Backtrace（线程回溯）
发生Crash的线程的Crash调用栈，从上到下分别代表调用顺序，最上面的一个表示抛出异常的位置，依次往下可以看到API的调用顺序。它包含闪退发生时调用函数的清单
### Thread State（线程状态）
Crash时发生时刻，线程的状态，通常我们根据Crash栈即可获取到相关信息，这部分一般不用关心。因为回溯部分的信息已经足够让你找出问题所在 
``` Thread 17 crashed with ARM Thread State (64-bit):
    x0: 0x00000001d488a960   x1: 0x00000001d4db98b8   x2: 0xfffffffffffffff8   x3: 0x00000001d4db9938rm
    x4: 0x00000001d4db9900   x5: 0x00000000000052b6   x6: 0x00000001d4db9d00   x7: 0x00000000000052b7
    x8: 0x00000001056e385c   x9: 0x0000000000000003  x10: 0x0000a80101120000  x11: 0x0000000000000001
   x12: 0x0000010000000203  x13: 0x0000020000000200  x14: 0x0000010000000203  x15: 0x00001d0000001d00
```
### Binary Images（二进制映像）
Crash时刻App加载的所有的库，其中第一行是Crash发生时我们App可执行文件的信息 

感谢[ios崩溃日志分析](http://www.cocoachina.com/industry/20130725/6677.html)提供的详细解答