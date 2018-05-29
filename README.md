# SerialPortHelper
Android 串口调试助手

# 前言

物联网开发开发是时下热门的行业。Android系统自然也能进行物联网开发。除开Android本身自带的模块还有一类通过外部链接的设备需要通过串口来进行通信。本人在做完两个相关的抓娃娃和寄存柜项目之后觉得需要总结一点东西给大家。

# 一些预备知识

## 关于串口

串口通信指[串口](https://baike.baidu.com/item/%E4%B8%B2%E5%8F%A3)按位（bit）发送和接收字节。尽管比按[字节](https://baike.baidu.com/item/%E5%AD%97%E8%8A%82)（byte）的[并行通信](https://baike.baidu.com/item/%E5%B9%B6%E8%A1%8C%E9%80%9A%E4%BF%A1)慢，但是串口可以在使用一根线发送数据的同时用另一根线接收数据。

在串口通信中，常用的协议包括RS-232、RS-422和RS-485。

当然具体是那种协议和你选择的硬件有关，将你的硬件插到对应协议的串口口即可。

## 开发前的准备

1.检查你的开发板设备，包括开发板信息，开发板上面包含的模块信息。是否有Wifi模块 蓝牙模块 指定接口等。还有一方面就是关于开发板系统的信息，开发板的系统版本。如果需要特别定制，可以和厂商商量。

>关于系统定制
某些特殊的板块需要隐藏状态栏不能被下拉，否则会被退出应用。还有一方面就是可以定制取消掉下导航栏。

2.检查你的硬件装备
正确连接你的设备，向你的硬件提供商索要开发资料。基本的资料包括硬件的通讯命令格式。当然更好的是如果能要到开发程序资料。比如android程序或者源码那就更好了。

3.正确的连接，测试你的硬件与系统
[Android串口助手](http://zhushou.360.cn/detail/index/soft_id/162355?recrefer=SE_D_%E4%B8%B2%E5%8F%A3%E5%8A%A9%E6%89%8B)
下载一个串口调试助手，按照资料输入命令。测试是否能够成功的启动设备。并且收到对应的返回数据。

# 开发阶段

>需要一点点的JNI知识和一点点Android多线程开发经验

整体的开发流程如下:打开指定串口-->开启接收数据线程(readthead)-->发送串口数据-->接收数据处理返回信息-->关闭接收数据线程(readthead)-->关闭串口。

## 导入so库

[谷歌开源serialPort api项目 ](https://github.com/cepr/android-serialport-api)

里面封装了c层代码调用底层代码的通信方式，如果你们喜欢改东西的话。可以自己改着玩，不过我觉得没有必要，因为这些代码已经封装的很好了。直接使用即可。

至于通过c代码如何生成相应的so文件，以及如何java层调用c层代码都是很基础的东西啦。
我不想在这里展开大篇幅的讲JNI，因为串口通信其实用的JNI知识不多。


首先把JNI相关代码导入到自己的工程里面：

先看下目录结构吧:

### jni目录
![TIM截图20180423162725.png](https://upload-images.jianshu.io/upload_images/7149395-d72f58063f410f80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### java 目录

![image.png](https://upload-images.jianshu.io/upload_images/7149395-f8bbfc3d08924e56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### SerialPort.java

了解JNI的同学都知道的，这个`SerialPort.h`对应的就是`SerialPort.java`层的native 方法。

这里用两个方法
```
private native static FileDescriptor open(String path, int baudrate, int flags);
public native void close();
```
很显然一个是打开串口 一个是 关闭串口 方法

打开串口之前，程序需要获得最高权限，`SerialPort.java`的构造函数里面需要获得设备的超级root权限，也是通过输入su命令完成。
```
if (!device.canRead() || !device.canWrite()) {
			try {
				/* Missing read/write permission, trying to chmod the file */
				Process su;
				su = Runtime.getRuntime().exec("/system/bin/su");
				String cmd = "chmod 666 " + device.getAbsolutePath() + "\n"
						+ "exit\n";
				su.getOutputStream().write(cmd.getBytes());
				if ((su.waitFor() != 0) || !device.canRead()
						|| !device.canWrite()) {
					throw new SecurityException();
				}
			} catch (Exception e) {
				e.printStackTrace();
				throw new SecurityException();
			}
		}
```

最后记得调用生成的.so文件

```
static {
		System.loadLibrary("serial_port");
	}
```

### SerialPortFinder

这个类很简单，能用于获取设备的串口信息。通常一个开发板会有几个到十几个的串口。
两个public方法:
- `public String[] getAllDevices()` 获取所有串口名称
- `public String[] getAllDevicesPath()` 获取所有串口地址


## 开始通信

![image.png](https://upload-images.jianshu.io/upload_images/7149395-7a137e0321109798.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

整个信息发送接收步骤如下:

1.初始化`SerialPort` 获得权限打开指定串口
2.打开`ReadThread`监听数据返回
3.使用`SendThread`发送数据 
4.继续发送或者关闭

为此我们需要写一个`SerialHelper`来简化代码，以下是核心代码:

**构造函数**
```
 public SerialHelper(String sPort, int iBaudRate) {
        this.sPort = "/dev/ttyS3";
        this.iBaudRate = 9600;
        this._isOpen = false;
        this._bLoopData = new byte[]{48};
        this.iDelay = 500;
        this.sPort = sPort;
        this.iBaudRate = iBaudRate;
    }
```
**打开 关闭 串口**
```
//打开时打开监听线程
 public void open() throws SecurityException, IOException, InvalidParameterException {
        this.mSerialPort = new SerialPort(new File(this.sPort), this.iBaudRate, 0);
        this.mOutputStream = this.mSerialPort.getOutputStream();
        this.mInputStream = this.mSerialPort.getInputStream();
        this.mReadThread = new SerialHelper.ReadThread();
        this.mReadThread.start();
        this.mSendThread = new SerialHelper.SendThread();
        this.mSendThread.setSuspendFlag();
        this.mSendThread.start();
        this._isOpen = true;
    }

   // 关闭线程 释放函数
    public void close() {
        if (this.mReadThread != null) {
            this.mReadThread.interrupt();
        }

        if (this.mSerialPort != null) {
            this.mSerialPort.close();
            this.mSerialPort = null;
        }

        this._isOpen = false;
    }
```
两个线程 发送线程:
```
private class SendThread extends Thread {
        public boolean suspendFlag;

        private SendThread() {
            this.suspendFlag = true;
        }

        public void run() {
            super.run();

            while(!this.isInterrupted()) {
                synchronized(this) {
                    while(this.suspendFlag) {
                        try {
                            this.wait();
                        } catch (InterruptedException var5) {
                            var5.printStackTrace();
                        }
                    }
                }

                SerialHelper.this.send(SerialHelper.this.getbLoopData());

                try {
                    Thread.sleep((long)SerialHelper.this.iDelay);
                } catch (InterruptedException var4) {
                    var4.printStackTrace();
                }
            }

        }

        public void setSuspendFlag() {
            this.suspendFlag = true;
        }

        public synchronized void setResume() {
            this.suspendFlag = false;
            this.notify();
        }
    }
```
读取线程
```
    private class ReadThread extends Thread {
        private ReadThread() {
        }

        public void run() {
            super.run();

            while(!this.isInterrupted()) {
                try {
                    if (SerialHelper.this.mInputStream == null) {
                        return;
                    }

                    byte[] buffer = new byte[512];
                    int size = SerialHelper.this.mInputStream.read(buffer);
                    if (size > 0) {
                        ComBean ComRecData = new ComBean(SerialHelper.this.sPort, buffer, size);
                        SerialHelper.this.onDataReceived(ComRecData);
                    }
                } catch (Throwable var4) {
                    Log.e("error", var4.getMessage());
                    return;
                }
            }

        }
    }
```
其他函数见代码，包括数值和文本发送 波特率的设置等等。

## 实战一个串口数据调试助手

下面使用封装的`SerialHelper`来完成整个数据发送接收:

界面随便搞一下:

![image.png](https://upload-images.jianshu.io/upload_images/7149395-a3be4920dd02af1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后开始逻辑代码:

首先实例化`SerialPortFinder` 实现数据接收 写入列表
```
serialPortFinder = new SerialPortFinder();
        serialHelper = new SerialHelper() {
            @Override
            protected void onDataReceived(final ComBean comBean) {
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(getBaseContext(), FuncUtil.ByteArrToHex(comBean.bRec), Toast.LENGTH_SHORT).show();
                        logListAdapter.addData(comBean.sRecTime+":   "+FuncUtil.ByteArrToHex(comBean.bRec));
                        recy.smoothScrollToPosition(logListAdapter.getData().size());
                    }
                });
            }
        };
```
然后利用`SerialPortFinder`找到所有的串口号，列出来所有的波特率 ，都传给`spinner` 
```
final String[] ports = serialPortFinder.getAllDevicesPath();
final String[] botes = new String[]{"0", "50", "75", "110", "134", "150", "200", "300", "600", "1200", "1800", "2400", "4800", "9600", "19200", "38400", "57600", "115200", "230400", "460800", "500000", "576000", "921600", "1000000", "1152000", "1500000", "2000000", "2500000", "3000000", "3500000", "4000000"};

SpAdapter spAdapter = new SpAdapter(this);
        spAdapter.setDatas(ports);
        spSerial.setAdapter(spAdapter);

SpAdapter spAdapter2 = new SpAdapter(this);
        spAdapter2.setDatas(botes);
        spBote.setAdapter(spAdapter2)
```
打开串口:
```
btOpen.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                try {
                    serialHelper.open();
                    btOpen.setEnabled(false);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
```
数据发送(两种数据发送格式):

文本类型
```
 if (serialHelper.isOpen()) {
                            serialHelper.sendTxt(edInput.getText().toString());
                        } else {
                            Toast.makeText(getBaseContext(), "搞毛啊,串口都没打开", Toast.LENGTH_SHORT).show();
                        }
```
Hex类型
```
if (serialHelper.isOpen()) {
                            serialHelper.sendHex(edInput.getText().toString());
                        } else {
                            Toast.makeText(getBaseContext(), "搞毛啊,串口都没打开", Toast.LENGTH_SHORT).show();
                        }
```

最后记得关闭一下串口咯:
```
 @Override
    protected void onDestroy() {
        super.onDestroy();
        serialHelper.close();
    }
```
好的 ，完事了。 测试一下

---------------------------------

连线开机:

![image.png](https://upload-images.jianshu.io/upload_images/7149395-7350309d1f19a6f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发串口信息:

![image.png](https://upload-images.jianshu.io/upload_images/7149395-dca15c362d568d96.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


同时设备也滴塌滴塌的响了，完美。

代码我也放上去把[Android串口助手]()

# 一些要说的

虽然整个JNI移植过程非常简单，但是问题出现了。如果大家使用的3.0版本的AS、会发现默认的JNI使用Cmake而不是.mk文件配置的。

所以又增加了一个难度，为了方便大家。我把所有关于串口的资源打包成aar 文件，大家直接使用即可。

[谷歌android串口开发 aar文件
](https://download.csdn.net/download/lw_zhaoritian/9961190)

使用过程:
aar文件导入lib文件夹
gradle文件
```
 repositories {
        flatDir {
            dirs 'libs'
        }
    }

dependencies {
    implementation(name: 'serialport-1.0.1', ext: 'aar')
}
```

完成。


# 总结一下

基本的串口通信到此结束。到了实际生产，更多的要解决多线程上的逻辑问题。设备的各种状态以及突发状况的处理等等。所以串口通信成功只是一个小小的开始，更多的问题还在后面。

