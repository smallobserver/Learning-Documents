# Android进阶



## Android多线程断点续传

### 一、原理

其实断点续传的原理很简单，从字面上理解，所谓断点续传就是从停止的地方重新下载。 断点：线程停止的位置。 续传：从停止的位置重新下载。

用代码解析就是： 断点 ： 当前线程已经下载完成的数据长度。 续传 ： 向服务器请求上次线程停止位置之后的数据。 原理知道了，功能实现起来也简单。每当线程停止时就把已下载的数据长度写入记录文件，当重新下载时，从记录文件读取已经下载了的长度。而这个长度就是所需要的断点。

续传的实现也简单，可以通过设置网络请求参数，请求服务器从指定的位置开始读取数据。 而要实现这两个功能只需要使用到httpURLconnection里面的setRequestProperty方法便可以实现.

```java
public void setRequestProperty(String field, String newValue)
```

如下所示，便是向服务器请求500-1000之间的500个byte：

```Java
conn.setRequestProperty("Range", "bytes=" + 500 + "-" + 1000);
```

以上只是续传的一部分需求，当我们获取到下载数据时，还需要将数据写入文件，而普通发File对象并不提供从指定位置写入数据的功能，这个时候，就需要使用到RandomAccessFile来实现从指定位置给文件写入数据的功能。

```java
public void seek(long offset)
```

如下所示，便是从文件的的第100个byte后开始写入数据。

```java
raFile.seek(100);
```

而开始写入数据时还需要用到RandomAccessFile里面的另外一个方法

```java
public void write(byte[] buffer, int byteOffset, int byteCount)
```

该方法的使用和OutputStream的write的使用一模一样...

以上便是断点续传的原理。

### 二、多线程断点续传

多线程断点续传便是在单线程的断点续传上延伸的。多线程断点续传是把整个文件分割成几个部分，每个部分由一条线程执行下载，而每一条下载线程都要实现断点续传功能。 为了实现文件分割功能，我们需要使用到httpURLconnection的另外一个方法：

```java
public int getContentLength()
```

当请求成功时，可以通过该方法获取到文件的总长度。 `每一条线程下载大小 = fileLength / THREAD_NUM`

如下图所示，描述的便是多线程的下载模型：

![img](http://upload-images.jianshu.io/upload_images/1824042-411c25f0cb1927de?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在多线程断点续传下载中，有一点需要特别注意： 由于文件是分成多个部分是被不同的线程的同时下载的，这就需要，每一条线程都分别需要有一个断点记录，和一个线程完成状态的记录；

![img](http://upload-images.jianshu.io/upload_images/1824042-843517c30becdcd6?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

只有所有线程的下载状态都处于完成状态时，才能表示文件已经下载完成。 实现记录的方法多种多样，我这里采用的是JDK自带的Properties类来记录下载参数。

### 三、断点续传结构

通过原理的了解，便可以很快的设计出断点续传工具类的基本结构图

![img](http://upload-images.jianshu.io/upload_images/1824042-74a1e86dee5bd13c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### IDownloadListener.java

```java
    package com.arialyy.frame.http.inf;
    import java.net.HttpURLConnection;

    /** 
     * 在这里面编写你的业务逻辑
     */
    public interface IDownloadListener {
        /**
         * 取消下载
         */
        public void onCancel();

        /**
         * 下载失败
         */
        public void onFail();

        /**
         * 下载预处理,可通过HttpURLConnection获取文件长度
         */
        public void onPreDownload(HttpURLConnection connection);

        /**
         * 下载监听
         */
        public void onProgress(long currentLocation);

        /**
         * 单一线程的结束位置
         */
        public void onChildComplete(long finishLocation);

        /**
         * 开始
         */
        public void onStart(long startLocation);

        /**
         * 子程恢复下载的位置
         */
        public void onChildResume(long resumeLocation);

        /**
         * 恢复位置
         */
        public void onResume(long resumeLocation);

        /**
         * 停止
         */
        public void onStop(long stopLocation);

        /**
         * 下载完成
         */
        public void onComplete();
    }
```

该类是下载监听接口

#### DownloadListener.java

```java
import java.net.HttpURLConnection;

/**
 * 下载监听
 */
public class DownloadListener implements IDownloadListener {

    @Override
    public void onResume(long resumeLocation) {

    }

    @Override
    public void onCancel() {

    }

    @Override
    public void onFail() {

    }

    @Override
    public void onPreDownload(HttpURLConnection connection) {

    }

    @Override
    public void onProgress(long currentLocation) {

    }

    @Override
    public void onChildComplete(long finishLocation) {

    }

    @Override
    public void onStart(long startLocation) {

    }

    @Override
    public void onChildResume(long resumeLocation) {

    }

    @Override
    public void onStop(long stopLocation) {

    }

    @Override
    public void onComplete() {

    }
}
```

### 下载参数实体

```java
    /**
     * 子线程下载信息类
     */
    private class DownloadEntity {
        //文件总长度
        long fileSize;
        //下载链接
        String downloadUrl;
        //线程Id
        int threadId;
        //起始下载位置
        long startLocation;
        //结束下载的文章
        long endLocation;
        //下载文件
        File tempFile;
        Context context;

        public DownloadEntity(Context context, long fileSize, String downloadUrl, File file, int threadId, long startLocation, long endLocation) {
            this.fileSize = fileSize;
            this.downloadUrl = downloadUrl;
            this.tempFile = file;
            this.threadId = threadId;
            this.startLocation = startLocation;
            this.endLocation = endLocation;
            this.context = context;
        }
    }
```

该类是下载信息配置类，每一条子线程的下载都需要一个下载实体来配置下载信息。

### 下载任务线程

```java
    /**
     * 多线程下载任务类
     */
    private class DownLoadTask implements Runnable {
        private static final String TAG = "DownLoadTask";
        private DownloadEntity dEntity;
        private String configFPath;

        public DownLoadTask(DownloadEntity downloadInfo) {
            this.dEntity = downloadInfo;
            configFPath = dEntity.context.getFilesDir().getPath() + "/temp/" + dEntity.tempFile.getName() + ".properties";
        }

        @Override
        public void run() {
            try {
                L.d(TAG, "线程_" + dEntity.threadId + "_正在下载【" + "开始位置 : " + dEntity.startLocation + "，结束位置：" + dEntity.endLocation + "】");
                URL url = new URL(dEntity.downloadUrl);
                HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                //在头里面请求下载开始位置和结束位置
                conn.setRequestProperty("Range", "bytes=" + dEntity.startLocation + "-" + dEntity.endLocation);
                conn.setRequestMethod("GET");
                conn.setRequestProperty("Charset", "UTF-8");
                conn.setConnectTimeout(TIME_OUT);
                conn.setRequestProperty("User-Agent", "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.2; Trident/4.0; .NET CLR 1.1.4322; .NET CLR 2.0.50727; .NET CLR 3.0.04506.30; .NET CLR 3.0.4506.2152; .NET CLR 3.5.30729)");
                conn.setRequestProperty("Accept", "image/gif, image/jpeg, image/pjpeg, image/pjpeg, application/x-shockwave-flash, application/xaml+xml, application/vnd.ms-xpsdocument, application/x-ms-xbap, application/x-ms-application, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*");
                conn.setReadTimeout(2000);  //设置读取流的等待时间,必须设置该参数
                InputStream is = conn.getInputStream();
                //创建可设置位置的文件
                RandomAccessFile file = new RandomAccessFile(dEntity.tempFile, "rwd");
                //设置每条线程写入文件的位置
                file.seek(dEntity.startLocation);
                byte[] buffer = new byte[1024];
                int len;
                //当前子线程的下载位置
                long currentLocation = dEntity.startLocation;
                while ((len = is.read(buffer)) != -1) {
                    if (isCancel) {
                        L.d(TAG, "++++++++++ thread_" + dEntity.threadId + "_cancel ++++++++++");
                        break;
                    }

                    if (isStop) {
                        break;
                    }

                    //把下载数据数据写入文件
                    file.write(buffer, 0, len);
                    synchronized (DownLoadUtil.this) {
                        mCurrentLocation += len;
                        mListener.onProgress(mCurrentLocation);
                    }
                    currentLocation += len;
                }
                file.close();
                is.close();

                if (isCancel) {
                    synchronized (DownLoadUtil.this) {
                        mCancelNum++;
                        if (mCancelNum == THREAD_NUM) {
                            File configFile = new File(configFPath);
                            if (configFile.exists()) {
                                configFile.delete();
                            }

                            if (dEntity.tempFile.exists()) {
                                dEntity.tempFile.delete();
                            }
                            L.d(TAG, "++++++++++++++++ onCancel +++++++++++++++++");
                            isDownloading = false;
                            mListener.onCancel();
                            System.gc();
                        }
                    }
                    return;
                }

                //停止状态不需要删除记录文件
                if (isStop) {
                    synchronized (DownLoadUtil.this) {
                        mStopNum++;
                        String location = String.valueOf(currentLocation);
                        L.i(TAG, "thread_" + dEntity.threadId + "_stop, stop location ==> " + currentLocation);
                        writeConfig(dEntity.tempFile.getName() + "_record_" + dEntity.threadId, location);
                        if (mStopNum == THREAD_NUM) {
                            L.d(TAG, "++++++++++++++++ onStop +++++++++++++++++");
                            isDownloading = false;
                            mListener.onStop(mCurrentLocation);
                            System.gc();
                        }
                    }
                    return;
                }

                L.i(TAG, "线程【" + dEntity.threadId + "】下载完毕");
                writeConfig(dEntity.tempFile.getName() + "_state_" + dEntity.threadId, 1 + "");
                mListener.onChildComplete(dEntity.endLocation);
                mCompleteThreadNum++;
                if (mCompleteThreadNum == THREAD_NUM) {
                    File configFile = new File(configFPath);
                    if (configFile.exists()) {
                        configFile.delete();
                    }
                    mListener.onComplete();
                    isDownloading = false;
                    System.gc();
                }
            } catch (MalformedURLException e) {
                e.printStackTrace();
                isDownloading = false;
                mListener.onFail();
            } catch (IOException e) {
                FL.e(this, "下载失败【" + dEntity.downloadUrl + "】" + FL.getPrintException(e));
                isDownloading = false;
                mListener.onFail();
            } catch (Exception e) {
                FL.e(this, "获取流失败" + FL.getPrintException(e));
                isDownloading = false;
                mListener.onFail();
            }
        }
```

这个是每条下载子线程的下载任务类，子线程通过下载实体对每一条线程进行下载配置，由于在多断点续传的概念里，停止表示的是暂停状态，而恢复表示的是线程从记录的断点重新进行下载，所以，线程处于停止状态时是不能删除记录文件的。

### 下载入口

```java
    /**
     * 多线程断点续传下载文件，暂停和继续
     *
     * @param context          必须添加该参数，不能使用全局变量的context
     * @param downloadUrl      下载路径
     * @param filePath         保存路径
     * @param downloadListener 下载进度监听 {@link DownloadListener}
     */
    public void download(final Context context, @NonNull final String downloadUrl, @NonNull final String filePath,
                         @NonNull final DownloadListener downloadListener) {
        isDownloading = true;
        mCurrentLocation = 0;
        isStop = false;
        isCancel = false;
        mCancelNum = 0;
        mStopNum = 0;
        final File dFile = new File(filePath);
        //读取已完成的线程数
        final File configFile = new File(context.getFilesDir().getPath() + "/temp/" + dFile.getName() + ".properties");
        try {
            if (!configFile.exists()) { //记录文件被删除，则重新下载
                newTask = true;
                FileUtil.createFile(configFile.getPath());
            } else {
                newTask = false;
            }
        } catch (Exception e) {
            e.printStackTrace();
            mListener.onFail();
            return;
        }
        newTask = !dFile.exists();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    mListener = downloadListener;
                    URL url = new URL(downloadUrl);
                    HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                    conn.setRequestMethod("GET");
                    conn.setRequestProperty("Charset", "UTF-8");
                    conn.setConnectTimeout(TIME_OUT);
                    conn.setRequestProperty("User-Agent", "Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 5.2; Trident/4.0; .NET CLR 1.1.4322; .NET CLR 2.0.50727; .NET CLR 3.0.04506.30; .NET CLR 3.0.4506.2152; .NET CLR 3.5.30729)");
                    conn.setRequestProperty("Accept", "image/gif, image/jpeg, image/pjpeg, image/pjpeg, application/x-shockwave-flash, application/xaml+xml, application/vnd.ms-xpsdocument, application/x-ms-xbap, application/x-ms-application, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*");
                    conn.connect();
                    int len = conn.getContentLength();
                    if (len < 0) {  //网络被劫持时会出现这个问题
                        mListener.onFail();
                        return;
                    }
                    int code = conn.getResponseCode();
                    if (code == 200) {
                        int fileLength = conn.getContentLength();
                        //必须建一个文件
                        FileUtil.createFile(filePath);
                        RandomAccessFile file = new RandomAccessFile(filePath, "rwd");
                        //设置文件长度
                        file.setLength(fileLength);
                        mListener.onPreDownload(conn);
                        //分配每条线程的下载区间
                        Properties pro = null;
                        pro = Util.loadConfig(configFile);
                        int blockSize = fileLength / THREAD_NUM;
                        SparseArray<Thread> tasks = new SparseArray<>();
                        for (int i = 0; i < THREAD_NUM; i++) {
                            long startL = i * blockSize, endL = (i + 1) * blockSize;
                            Object state = pro.getProperty(dFile.getName() + "_state_" + i);
                            if (state != null && Integer.parseInt(state + "") == 1) {  //该线程已经完成
                                mCurrentLocation += endL - startL;
                                L.d(TAG, "++++++++++ 线程_" + i + "_已经下载完成 ++++++++++");
                                mCompleteThreadNum++;
                                if (mCompleteThreadNum == THREAD_NUM) {
                                    if (configFile.exists()) {
                                        configFile.delete();
                                    }
                                    mListener.onComplete();
                                    isDownloading = false;
                                    System.gc();
                                    return;
                                }
                                continue;
                            }
                            //分配下载位置
                            Object record = pro.getProperty(dFile.getName() + "_record_" + i);
                            if (!newTask && record != null && Long.parseLong(record + "") > 0) {       //如果有记录，则恢复下载
                                Long r = Long.parseLong(record + "");
                                mCurrentLocation += r - startL;
                                L.d(TAG, "++++++++++ 线程_" + i + "_恢复下载 ++++++++++");
                                mListener.onChildResume(r);
                                startL = r;
                            }
                            if (i == (THREAD_NUM - 1)) {
                                endL = fileLength;//如果整个文件的大小不为线程个数的整数倍，则最后一个线程的结束位置即为文件的总长度
                            }
                            DownloadEntity entity = new DownloadEntity(context, fileLength, downloadUrl, dFile, i, startL, endL);
                            DownLoadTask task = new DownLoadTask(entity);
                            tasks.put(i, new Thread(task));
                        }
                        if (mCurrentLocation > 0) {
                            mListener.onResume(mCurrentLocation);
                        } else {
                            mListener.onStart(mCurrentLocation);
                        }
                        for (int i = 0, count = tasks.size(); i < count; i++) {
                            Thread task = tasks.get(i);
                            if (task != null) {
                                task.start();
                            }
                        }
                    } else {
                        FL.e(TAG, "下载失败，返回码：" + code);
                        isDownloading = false;
                        System.gc();
                        mListener.onFail();
                    }
                } catch (IOException e) {
                    FL.e(this, "下载失败【downloadUrl:" + downloadUrl + "】\n【filePath:" + filePath + "】" + FL.getPrintException(e));
                    isDownloading = false;
                    mListener.onFail();
                }
            }
        }).start();
    }
```

其实也没啥好说的，注释已经很完整了，需要注意两点 1、恢复下载时：`已下载的文件大小 = 该线程的上一次断点的位置 - 该线程起始下载位置`； 2、为了保证下载文件的完整性，只要记录文件不存在就需要重新进行下载；





## Android全局异常处理

在做android项目开发时，大家都知道如果程序出错了，会弹出来一个强制退出的弹出框，这个本身没什么问题，但是这个UI实在是太丑了，别说用户接受不了，就连我们自己本身可能都接受不了。虽然我们在发布程序时总会经过仔细的测试，但是难免会碰到预料不到的错误。

今天就来自定义一个程序出错时的处理，类似iphone的闪退。(虽然闪退也是用户不愿意看到的，但是在用户体验上明显比那个原生的弹窗好多了)

废话不多说，直接上代码：

### CrashHandler

```Java
/** 
 * 自定义的 异常处理类 , 实现了 UncaughtExceptionHandler接口  
 * 
 */  
public class CrashHandler implements UncaughtExceptionHandler {  
    // 需求是 整个应用程序 只有一个 MyCrash-Handler   
    private static CrashHandler INSTANCE ;  
    private Context context;  

    //1.私有化构造方法  
    private CrashHandler(){  

    }  

    public static synchronized CrashHandler getInstance(){  
        if (INSTANCE == null)  
            INSTANCE = new CrashHandler();  
        return INSTANCE;
    }

    public void init(Context context){  
        this.context = context;
    }  


    public void uncaughtException(Thread arg0, Throwable arg1) {  
        System.out.println("程序挂掉了 ");  
        // 在此可以把用户手机的一些信息以及异常信息捕获并上传,由于UMeng在这方面有很程序的api接口来调用，故没有考虑

        //干掉当前的程序   
        android.os.Process.killProcess(android.os.Process.myPid());  
    }  

}
```

### CrashApplication

```java
/** 
 * 在开发应用时都会和Activity打交道，而Application使用的就相对较少了。 
 * Application是用来管理应用程序的全局状态的，比如载入资源文件。 
 * 在应用程序启动的时候Application会首先创建，然后才会根据情况(Intent)启动相应的Activity或者Service。 
 * 在本文将在Application中注册未捕获异常处理器。 
 */  
public class CrashApplication extends Application {  
    @Override  
    public void onCreate() {  
        super.onCreate();  
        CrashHandler handler = CrashHandler.getInstance();  
        handler.init(getApplicationContext());
        Thread.setDefaultUncaughtExceptionHandler(handler);  
    }  
}
```

### 在AndroidManifest.xml中注册

```xml
<?xml version="1.0" encoding="utf-8"?>  
<manifest xmlns:android="http://schemas.android.com/apk/res/android"  
    package="org.wp.activity" android:versionCode="1" android:versionName="1.0">  
    <application android:icon="@drawable/icon" android:label="@string/app_name"  
        android:name=".CrashApplication" android:debuggable="true">  
        <activity android:name=".MainActivity" android:label="@string/app_name">  
            <intent-filter>  
                <action android:name="android.intent.action.MAIN" />  
                <category android:name="android.intent.category.LAUNCHER" />  
            </intent-filter>  
        </activity>  
    </application>  
    <uses-sdk android:minSdkVersion="8" />  
</manifest>
```

至此，可以测试下在出错的时候程序会直接闪退，并杀死后台进程。当然也可以自定义一些比较友好的出错UI提示，进一步提升用户体验。





## Android Parcelable和Serializable的区别

本文主要**介绍Parcelable和Serializable的作用、效率、区别及选择**。

### **1、作用**

Serializable的作用是**为了保存对象的属性到本地文件、数据库、网络流、rmi以方便数据传输，当然这种传输可以是程序内的也可以是两个程序间的**。而Android的Parcelable的设计初衷是因为Serializable效率过慢，**为了在程序内不同组件间以及不同Android程序间(AIDL)高效**的传输数据而设计，这些数据仅在内存中存在，Parcelable是通过IBinder通信的消息的载体。

从上面的设计上我们就可以看出优劣了。

### **2、效率及选择**

Parcelable的性能比Serializable好，在内存开销方面较小，所以**在内存间数据传输时推荐使用Parcelable**，如activity间传输数据，而Serializable可将数据持久化方便保存，所以**在需要保存或网络传输数据时选择Serializable**，因为android不同版本Parcelable可能不同，所以不推荐使用Parcelable进行数据持久化。

### **3、编程实现**

对于Serializable，类只需要实现Serializable接口，并提供一个序列化版本id(serialVersionUID)即可。而Parcelable则需要实现writeToParcel、describeContents函数以及静态的CREATOR变量，实际上就是**将如何打包和解包的工作自己来定义，而序列化的这些操作完全由底层实现**。

Parcelable的一个实现例子如下

```java
public class MyParcelable implements Parcelable {
     private int mData;
     private String mStr;

     public int describeContents() {
         return 0;
     }

     // 写数据进行保存
     public void writeToParcel(Parcel out, int flags) {
         out.writeInt(mData);
         out.writeString(mStr);
     }

     // 用来创建自定义的Parcelable的对象
     public static final Parcelable.Creator<MyParcelable> CREATOR
             = new Parcelable.Creator<MyParcelable>() {
         public MyParcelable createFromParcel(Parcel in) {
             return new MyParcelable(in);
         }

         public MyParcelable[] newArray(int size) {
             return new MyParcelable[size];
         }
     };

     // 读数据进行恢复
     private MyParcelable(Parcel in) {
         mData = in.readInt();
         mStr = in.readString();
     }
 }
```

从上面我们可以看出Parcel的写入和读出顺序是一致的。如果元素是list读出时需要先new一个ArrayList传入，否则会报空指针异常。如下：

```java
list = new ArrayList<String>();
in.readStringList(list);
```

PS: 在自己使用时，read数据时误将前面int数据当作long读出，结果后面的顺序错乱，报如下异常，当类字段较多时**务必保持写入和读取的类型及顺序一致**。

```
11-21 20:14:10.317: E/AndroidRuntime(21114): Caused by: java.lang.RuntimeException: Parcel android.os.Parcel@4126ed60: Unmarshalling unknown type code 3014773 at offset 164
```

### **4、高级功能上**

Serializable序列化不保存静态变量，可以使用Transient关键字对部分字段不进行序列化，也可以覆盖writeObject、readObject方法以实现序列化过程自定义。





## 一个APP从启动到主页面显示经历了哪些过程？

### 一、流程概述

![img](http://upload-images.jianshu.io/upload_images/3985563-b7edc7b70c9c332f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 启动流程：

①点击桌面App图标，Launcher进程采用Binder IPC向system_server进程发起startActivity请求；

②system_server进程接收到请求后，向zygote进程发送创建进程的请求；

③Zygote进程fork出新的子进程，即App进程；

④App进程，通过Binder IPC向sytem_server进程发起attachApplication请求；

⑤system_server进程在收到请求后，进行一系列准备工作后，再通过binder IPC向App进程发送scheduleLaunchActivity请求；

⑥App进程的binder线程（ApplicationThread）在收到请求后，通过handler向主线程发送LAUNCH_ACTIVITY消息；

⑦主线程在收到Message后，通过发射机制创建目标Activity，并回调Activity.onCreate()等方法。

⑧到此，App便正式启动，开始进入Activity生命周期，执行完onCreate/onStart/onResume方法，UI渲染结束后便可以看到App的主界面。

上面的一些列步骤简单介绍了一个APP启动到主页面显示的过程，可能这些流程中的一些术语看的有些懵，什么是Launcher，什么是zygote，什么是applicationThread.....

下面我们一一介绍。

### 二、理论基础

#### 1.zygote

zygote意为“受精卵“。Android是基于Linux系统的，而在Linux中，所有的进程都是由init进程直接或者是间接fork出来的，zygote进程也不例外。

在Android系统里面，zygote是一个进程的名字。Android是基于Linux System的，当你的手机开机的时候，Linux的内核加载完成之后就会启动一个叫“init“的进程。在Linux System里面，所有的进程都是由init进程fork出来的，我们的zygote进程也不例外。

我们都知道，每一个App其实都是

● 一个单独的dalvik虚拟机

● 一个单独的进程

所以当系统里面的第一个zygote进程运行之后，在这之后再开启App，就相当于开启一个新的进程。而为了实现资源共用和更快的启动速度，Android系统开启新进程的方式，是通过fork第一个zygote进程实现的。所以说，除了第一个zygote进程，其他应用所在的进程都是zygote的子进程，这下你明白为什么这个进程叫“受精卵”了吧？因为就像是一个受精卵一样，它能快速的分裂，并且产生遗传物质一样的细胞！

#### 2.system_server

SystemServer也是一个进程，而且是由zygote进程fork出来的。

知道了SystemServer的本质，我们对它就不算太陌生了，这个进程是Android Framework里面两大非常重要的进程之一——另外一个进程就是上面的zygote进程。

为什么说SystemServer非常重要呢？因为系统里面重要的服务都是在这个进程里面开启的，比如 ActivityManagerService、PackageManagerService、WindowManagerService等等。

#### 3.ActivityManagerService

ActivityManagerService，简称AMS，服务端对象，负责系统中所有Activity的生命周期。

ActivityManagerService进行初始化的时机很明确，就是在SystemServer进程开启的时候，就会初始化ActivityManagerService。

#### 下面介绍下Android系统里面的服务器和客户端的概念。

其实服务器客户端的概念不仅仅存在于Web开发中，在Android的框架设计中，使用的也是这一种模式。服务器端指的就是所有App共用的系统服务，比如我们这里提到的ActivityManagerService，和前面提到的PackageManagerService、WindowManagerService等等，这些基础的系统服务是被所有的App公用的，当某个App想实现某个操作的时候，要告诉这些系统服务，比如你想打开一个App，那么我们知道了包名和MainActivity类名之后就可以打开

```java
Intent intent = new Intent(Intent.ACTION_MAIN);  
intent.addCategory(Intent.CATEGORY_LAUNCHER);              
ComponentName cn = new ComponentName(packageName, className);              
intent.setComponent(cn);  
startActivity(intent);
```

但是，我们的App通过调用startActivity()并不能直接打开另外一个App，这个方法会通过一系列的调用，最后还是告诉AMS说：“我要打开这个App，我知道他的住址和名字，你帮我打开吧！”所以是AMS来通知zygote进程来fork一个新进程，来开启我们的目标App的。这就像是浏览器想要打开一个超链接一样，浏览器把网页地址发送给服务器，然后还是服务器把需要的资源文件发送给客户端的。

知道了Android Framework的客户端服务器架构之后，我们还需要了解一件事情，那就是我们的App和AMS(SystemServer进程)还有zygote进程分属于三个独立的进程，他们之间如何通信呢？

**App与AMS通过Binder进行IPC通信，AMS(SystemServer进程)与zygote通过Socket进行IPC通信。后面具体介绍。**

那么AMS有什么用呢？**在前面我们知道了，如果想打开一个App的话，需要AMS去通知zygote进程，除此之外，其实所有的Activity的开启、暂停、关闭都需要AMS来控制，所以我们说，AMS负责系统中所有Activity的生命周期。**

在Android系统中，**任何一个Activity的启动都是由AMS和应用程序进程（主要是ActivityThread）相互配合来完成的。AMS服务统一调度系统中所有进程的Activity启动，而每个Activity的启动过程则由其所属的进程具体来完成。**

#### 4.Launcher

当我们点击手机桌面上的图标的时候，App就由Launcher开始启动了。但是，你有没有思考过Launcher到底是一个什么东西？

**Launcher本质上也是一个应用程序，和我们的App一样，也是继承自Activity**

packages/apps/Launcher2/src/com/android/launcher2/Launcher.java

```java
public final class Launcher extends Activity
        implements View.OnClickListener, OnLongClickListener, LauncherModel.Callbacks,
                   View.OnTouchListener {
                   }
```

Launcher实现了点击、长按等回调接口，来接收用户的输入。既然是普通的App，那么我们的开发经验在这里就仍然适用，比如，我们点击图标的时候，是怎么开启的应用呢？**捕捉图标点击事件，然后startActivity()发送对应的Intent请求呗！是的，Launcher也是这么做的，就是这么easy！**

#### 5.Instrumentation和ActivityThread

每个Activity都持有Instrumentation对象的一个引用，但是整个进程只会存在一个Instrumentation对象。 Instrumentation这个类里面的方法大多数和Application和Activity有关，**这个类就是完成对Application和Activity初始化和生命周期的工具类。**Instrumentation这个类很重要，对Activity生命周期方法的调用根本就离不开他，他可以说是一个大管家。

ActivityThread，依赖于UI线程。App和AMS是通过Binder传递信息的，那么ActivityThread就是专门与AMS的外交工作的。

#### 6.ApplicationThread

前面我们已经知道了App的启动以及Activity的显示都需要AMS的控制，那么我们便需要和服务端的沟通，而这个沟通是双向的。

**客户端-->服务端**

![img](http://upload-images.jianshu.io/upload_images/3985563-f9b3071c35cdb5de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而且由于继承了同样的公共接口类，ActivityManagerProxy提供了与ActivityManagerService一样的函数原型，使用户感觉不出Server是运行在本地还是远端，从而可以更加方便的调用这些重要的系统服务。

**服务端-->客户端**

还是通过Binder通信，不过是换了另外一对，换成了ApplicationThread和ApplicationThreadProxy。

![img](http://upload-images.jianshu.io/upload_images/3985563-363b74b716570dd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

他们也都实现了相同的接口IApplicationThread

```java
  private class ApplicationThread extends ApplicationThreadNative {}

  public abstract class ApplicationThreadNative extends Binder implements IApplicationThread{}

  class ApplicationThreadProxy implements IApplicationThread {}
```

**好了，前面罗里吧嗦的一大堆，介绍了一堆名词，可能不太清楚，没关系，下面结合流程图介绍。**

### 三、启动流程

#### 1.创建进程

①先从Launcher的startActivity()方法，通过Binder通信，调用ActivityManagerService的startActivity方法。

②一系列折腾，最后调用startProcessLocked()方法来创建新的进程。

③该方法会通过前面讲到的socket通道传递参数给Zygote进程。Zygote孵化自身。调用ZygoteInit.main()方法来实例化ActivityThread对象并最终返回新进程的pid。

④调用ActivityThread.main()方法，ActivityThread随后依次调用Looper.prepareLoop()和Looper.loop()来开启消息循环。

**方法调用流程图如下:**

![img](http://upload-images.jianshu.io/upload_images/3985563-25c23ee6ccb48048.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**更直白的流程解释：**

![img](http://upload-images.jianshu.io/upload_images/3985563-ed91fd7c240e6bd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

①App发起进程：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；

②system_server进程：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；

③zygote进程：在执行ZygoteInit.main()后便进入runSelectLoop()循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；

④新进程：执行handleChildProc方法，最后调用ActivityThread.main()方法。

#### 2.绑定Application

上面创建进程后，执行ActivityThread.main()方法，随后调用attach()方法。

将进程和指定的Application绑定起来。这个是通过上节的ActivityThread对象中调用bindApplication()方法完成的。该方法发送一个BIND_APPLICATION的消息到消息队列中, 最终通过handleBindApplication()方法处理该消息. 然后调用makeApplication()方法来加载App的classes到内存中。

**方法调用流程图如下：**

![img](http://upload-images.jianshu.io/upload_images/3985563-0eb6b9d2b091de3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**更直白的流程解释：**

![img](http://upload-images.jianshu.io/upload_images/3985563-d8def9358f4646e1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（如果看不懂AMS,ATP等名词，后面有解释）

#### 3.显示Activity界面

经过前两个步骤之后, 系统已经拥有了该application的进程。 后面的调用顺序就是普通的从一个已经存在的进程中启动一个新进程的activity了。

实际调用方法是realStartActivity(), 它会调用application线程对象中的scheduleLaunchActivity()发送一个LAUNCH_ACTIVITY消息到消息队列中, 通过 handleLaunchActivity()来处理该消息。在 handleLaunchActivity()通过performLaunchActiivty()方法回调Activity的onCreate()方法和onStart()方法，然后通过handleResumeActivity()方法，回调Activity的onResume()方法，最终显示Activity界面。

![img](http://upload-images.jianshu.io/upload_images/3985563-5222775558226c7d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**更直白的流程解释：**

![img](http://upload-images.jianshu.io/upload_images/3985563-5f711b4bca6bf21b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、Binder通信

![img](http://upload-images.jianshu.io/upload_images/3985563-cb3187996516846a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简称:

**ATP:** ApplicationThreadProxy

**AT:** ApplicationThread

**AMP:** ActivityManagerProxy

**AMS:** ActivityManagerService

图解:

①system_server进程中调用startProcessLocked方法,该方法最终通过socket方式,将需要创建新进程的消息告知Zygote进程,并阻塞等待Socket返回新创建进程的pid;

②Zygote进程接收到system_server发送过来的消息, 则通过fork的方法，将zygote自身进程复制生成新的进程，并将ActivityThread相关的资源加载到新进程app process,这个进程可能是用于承载activity等组件;

③ 在新进程app process向servicemanager查询system_server进程中binder服务端AMS, 获取相对应的Client端,也就是AMP. 有了这一对binder c/s对, 那么app process便可以通过binder向跨进程system_server发送请求,即attachApplication()

④system_server进程接收到相应binder操作后,经过多次调用,利用ATP向app process发送binder请求, 即bindApplication. system_server拥有ATP/AMS, 每一个新创建的进程都会有一个相应的AT/AMP,从而可以跨进程 进行相互通信. 这便是进程创建过程的完整生态链。

以上大概介绍了一个APP从启动到主页面显示经历的流程，主要从宏观角度介绍了其过程，具体可结合源码理解。





## Android性能优化总结

### 一、Android性能优化的方面

针对Android的性能优化，主要有以下几个有效的优化方法：

1.布局优化

2.绘制优化

3.内存泄漏优化

4.响应速度优化

5.ListView/RecycleView及Bitmap优化

6.线程优化

7.其他性能优化的建议

下面我们具体来介绍关于以上这几个方面优化的具体思路及解决方案。

### 二、布局优化

关于布局优化的思想很简单，就是**尽量减少布局文件的层级**。这个道理很浅显，布局中的层级少了，就意味着Android绘制时的工作量少了，那么程序的性能自然就提高了。

#### 如何进行布局优化？

**①删除布局中无用的控件和层次，其次有选择地使用性能比较低的ViewGroup。**

关于有选择地使用性能比较低的ViewGroup,这就需要我们开发就实际灵活选择了。

例如：如果布局中既可以使用LinearLayout也可以使用RelativeLayout，那么就采用LinearLayout，这是因为RelativeLayout的功能比较复杂，它的布局过程需要花费更多的CPU时间。FrameLayout和LinearLayout一样都是一种简单高效的ViewGroup，因此可以考虑使用它们，但是**很多时候单纯通过一个LinearLayout或者FrameLayout无法实现产品效果，需要通过嵌套的方式来完成。这种情况下还是建议采用RelativeLayout,因为ViewGroup的嵌套就相当于增加了布局的层级，同样会降低程序的性能。**

**②采用标签,标签,ViewStub。**

标签主要用于布局重用。

标签一般和配合使用，可以降低减少布局的层级。

ViewStub提供了按需加载的功能，当需要时才会将ViewStub中的布局加载到内存，提高了程序初始化效率。

**③避免多度绘制**

过度绘制（Overdraw）描述的是屏幕上的某个像素在同一帧的时间内被绘制了多次。在多层次重叠的 UI 结构里面，如果不可见的 UI 也在做绘制的操作，会导致某些像素区域被绘制了多次，同时也会浪费大量的 CPU 以及 GPU 资源。

如下所示，有些部分在布局时，会被重复绘制。

![img](http://upload-images.jianshu.io/upload_images/3985563-7f91a67f91bf3317.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于过度绘制产生的一般场景及解决方案，参考：[Android 过度绘制优化](http://jaeger.itscoder.com/android/2016/09/29/android-performance-overdraw.html)

### 三、绘制优化

绘制优化是指**View的onDraw方法要避免执行大量的操作，**这主要体现在两个方面：

**①onDraw中不要创建新的局部对象。**

因为onDraw方法可能会被频繁调用，这样就会在一瞬间产生大量的临时对象，这不仅占用了过多的内存而且还会导致系统更加频繁gc，降低了程序的执行效率。

**②onDraw方法中不要做耗时的任务，也不能执行成千上万次的循环操作，尽管每次循环都很轻量级，但是大量的循环仍然十分抢占CPU的时间片，这会造成View的绘制过程不流畅。**

按照Google官方给出的性能优化典范中的标准，View的绘制频率保证60fps是最佳的，这就要求每帧绘制时间不超过16ms(16ms = 1000/60)，虽然程序很难保证16ms这个时间，但是尽量降低onDraw方法中的复杂度总是切实有效的。

![img](http://upload-images.jianshu.io/upload_images/3985563-81b81fb8ab4d92db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 四、内存泄漏优化

内存泄漏是开发过程中的一个需要重视的问题，但是由于内存泄露问题对开发人员的经验和开发意识有较高的要求，因此也是开发人员最容易犯的错误之一。

内存泄露的优化分为两个方面：

**①在开发过程中避免写出有内存泄漏的代码**

**②通过一些分析工具比如MAT来找出潜在的内存泄露，然后解决。**

对应于两种不同情况，一个是了解内存泄漏的可能场景以及如何规避，二是怎么查找内存泄漏。

#### 1.那么我们就先了解什么是内存泄漏?这样我们才能知道如何避免。

大家都知道，java是有垃圾回收机制的，这使得java程序员比C++程序员轻松了许多，存储申请了，不用心心念念要加一句释放，java虚拟机会派出一些回收线程兢兢业业不定时地回收那些不再被需要的内存空间（注意回收的不是对象本身，而是对象占据的内存空间）。

**Q1：什么叫不再被需要的内存空间？**

**答：**Java没有指针，全凭引用来和对象进行关联，通过引用来操作对象。如果一个对象没有与任何引用关联，那么这个对象也就不太可能被使用到了，回收器便是把这些“无任何引用的对象”作为目标，回收了它们占据的内存空间。

**Q2：如何分辨为对象无引用？**

**答：**2种方法

**引用计数法**直接计数，简单高效，Python便是采用该方法。但是如果出现 两个对象相互引用，即使它们都无法被外界访问到，计数器不为0它们也始终不会被回收。为了解决该问题，java采用的是b方法。

**可达性分析法**这个方法设置了一系列的“GC Roots”对象作为索引起点，如果一个对象 与起点对象之间均无可达路径，那么这个不可达的对象就会成为回收对象。这种方法处理 两个对象相互引用的问题，如果两个对象均没有外部引用，会被判断为不可达对象进而被回收（如下图）。

![img](https://segmentfault.com/img/remote/1460000006884313?w=631&h=235)

**Q3：有了回收机制，放心大胆用不会有内存泄漏？**

**答：**答案当然是No！

虽然垃圾回收器会帮我们干掉大部分无用的内存空间，但是对于还保持着引用，但逻辑上已经不会再用到的对象，垃圾回收器不会回收它们。这些对象积累在内存中，直到程序结束，就是我们所说的“内存泄漏”。 当然了，用户对单次的内存泄漏并没有什么感知，但当泄漏积累到内存都被消耗完，就会导致卡顿，崩溃。

下面这张图可以帮助我们更好地理解对象的状态，以及内存泄漏的情况

![img](http://upload-images.jianshu.io/upload_images/3985563-2cb740a394402ae0.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

左边未引用的对象是会被GC回收的，右边被引用的对象不会被GC回收，但是未使用的对象中除了未引用的对象，还包括已被引用的一部分对象，那么内存泄漏久发生这部分已被引用但未使用的对象。

#### 2.Android一般在什么情况下会出现内存泄漏？

①集合类泄漏 ②单例/静态变量造成的内存泄漏 ③匿名内部类/非静态内部类 ④资源未关闭造成的内存泄漏

大概可以分为以上几类，还有一些经常会听到的Hanlder,AsyncTask引起内存泄漏，都属于上述③中的情况。

那么上述四种情况是怎么造成的内存泄漏，具体是什么原因，以及Android中一些知名的引起内存泄漏的原因，以及解决方法是怎么样的？

#### 3.Android怎么分析内存泄漏？

上面介绍了内存泄漏的场景，对应的有一些解决方案。

那么在内存泄漏已经发生的情况下，我们该如何解决呢？

我们可以通过MAT(Memory Analyzer Tool)，或者 LeakCanary来检测Android中的内存泄漏。

### 五、响应速度优化

响应速度优化的核心思想就是**避免在主线程中做耗时操作**。

如果有耗时操作，可以开启子线程执行，即采用异步的方式来执行耗时操作。

如果在主线程中做太多事情，会导致Activity启动时出现黑屏现象，甚至ANR。

![img](http://upload-images.jianshu.io/upload_images/3985563-a1e005753b8e32a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Android规定，Activity如果5秒钟之内无法响应屏幕触摸事件或者键盘输入事件就会出现ANR，而BroadcastReceiver如果10秒钟之内还未执行完操作也会出现ANR。**

为了避免ANR，可以开启子线程执行耗时操作，但是子线程不能更新UI，所以需要子线程与主线程进行通信来解决子线程执行耗时任务后，通知主线程更新UI的场景。关于这部分，需要掌握Handler消息机制，AsyncTask，IntentService等内容。

然而，在实际开发中，ANR仍然不可避免的发生了，而且很难从代码上发现，这时候就要用到ANR日志分析。当一个进程发生了ANR之后，系统会在/data/anr目录下创建一个文件traces.txt，通过分析这个文件就能定位出ANR的原因。

### 六、ListView/RecycleView及Bitmap优化

![img](http://upload-images.jianshu.io/upload_images/3985563-d51c4fec20e776cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**ListView/RecycleView的优化思想主要从以下几个方面入手：**

①使用ViewHolder模式来提高效率

②异步加载：耗时的操作放在异步线程中

③ListView/RecycleView的滑动时停止加载和分页加载

具体优化建议及详情，参考：[ListView的优化](http://www.jianshu.com/p/f0408a0f0610)

**Bitmap优化**

![img](http://upload-images.jianshu.io/upload_images/3985563-eab380aea4795930.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要是对加载图片进行压缩，避免加载图片多大导致OOM出现。

### 七、线程优化

线程优化的思想就是**采用线程池，避免程序中存在大量的Thread。**线程池可以重用内部的线程，从而避免了线程的创建和销毁锁带来的性能开销，同时线程池还能有效地控制线程池的最大并法术，避免大量的线程因互相抢占系统资源从而导致阻塞现象的发生。因此在实际开发中，尽量采用线程池，而不是每次都要创建一个Thread对象。

![img](http://upload-images.jianshu.io/upload_images/3985563-7dda79e4c0ec6d78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 八、其他性能优化建议

①避免过度的创建对象

②不要过度使用枚举，枚举占用的内存空间要比整型大

③常量请使用static final来修饰

④使用一些Android特有的数据结构，比如SparseArray和Pair等

⑤适当采用软引用和弱引用

⑥采用内存缓存和磁盘缓存

⑦尽量采用静态内部类，这样可以避免潜在的由于内部类而导致的内存泄漏。

以上是关于Android性能优化方面，我们一些入手点。从这些方面，我们可以在平时的开发中注意，避免类似错误，提高Android程序的性能，但是其中一些方面的要求则需要我们不断的学习，以及平时良好的意识与习惯。由于自己开发经验几乎为0，没办法根据实际经验来说明，只能写下这篇文章来提醒自己以后开发的时候需要注意和培养的地方。





## Android 内存泄漏总结

内存管理的目的就是让我们在开发中怎么有效的避免我们的应用出现内存泄漏的问题。内存泄漏大家都不陌生了，简单粗俗的讲，就是该被释放的对象没有释放，一直被某个或某些实例所持有却不再被使用导致 GC 不能回收

我会从 java 内存泄漏的基础知识开始，并通过具体例子来说明 Android 引起内存泄漏的各种原因，以及如何利用工具来分析应用内存泄漏，最后再做总结。

篇幅有些长，大家可以分几节来看！

### Java 内存分配策略

Java 程序运行时的内存分配策略有三种,分别是静态分配,栈式分配,和堆式分配，对应的，三种存储策略使用的内存空间主要分别是静态存储区（也称方法区）、栈区和堆区。

- 静态存储区（方法区）：主要存放静态数据、全局 static 数据和常量。这块内存在程序编译时就已经分配好，并且在程序整个运行期间都存在。
- 栈区 ：当方法被执行时，方法体内的局部变量都在栈上创建，并在方法执行结束时这些局部变量所持有的内存将会自动被释放。因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。
- 堆区 ： 又称动态内存分配，通常就是指在程序运行时直接 new 出来的内存。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收。

#### 栈与堆的区别：

在方法体内定义的（局部变量）一些基本类型的变量和对象的引用变量都是在方法的栈内存中分配的。当在一段方法块中定义一个变量时，Java 就会在栈中为该变量分配内存空间，当超过该变量的作用域后，该变量也就无效了，分配给它的内存空间也将被释放掉，该内存空间可以被重新使用。

堆内存用来存放所有由 new 创建的对象（包括该对象其中的所有成员变量）和数组。在堆中分配的内存，将由 Java 垃圾回收器来自动管理。在堆中产生了一个数组或者对象后，还可以在栈中定义一个特殊的变量，这个变量的取值等于数组或者对象在堆内存中的首地址，这个特殊的变量就是我们上面说的引用变量。我们可以通过这个引用变量来访问堆中的对象或者数组。

举个例子:

```java
public class Sample() {
    int s1 = 0;
    Sample mSample1 = new Sample();

    public void method() {
        int s2 = 1;
        Sample mSample2 = new Sample();
    }
}

Sample mSample3 = new Sample();
```

Sample 类的局部变量 s2 和引用变量 mSample2 都是存在于栈中，但 mSample2 指向的对象是存在于堆上的。 mSample3 指向的对象实体存放在堆上，包括这个对象的所有成员变量 s1 和 mSample1，而它自己存在于栈中。

结论：

局部变量的基本数据类型和引用存储于栈中，引用的对象实体存储于堆中。—— 因为它们属于方法中的变量，生命周期随方法而结束。

成员变量全部存储于堆中（包括基本数据类型，引用和引用的对象实体）—— 因为它们属于类，类对象终究是要被new出来使用的。

了解了 Java 的内存分配之后，我们再来看看 Java 是怎么管理内存的。

### Java是如何管理内存

Java的内存管理就是对象的分配和释放问题。在 Java 中，程序员需要通过关键字 new 为每个对象申请内存空间 (基本类型除外)，所有的对象都在堆 (Heap)中分配空间。另外，对象的释放是由 GC 决定和执行的。在 Java 中，内存的分配是由程序完成的，而内存的释放是由 GC 完成的，这种收支两条线的方法确实简化了程序员的工作。但同时，它也加重了JVM的工作。这也是 Java 程序运行速度较慢的原因之一。因为，GC 为了能够正确释放对象，GC 必须监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等，GC 都需要进行监控。

监视对象状态是为了更加准确地、及时地释放对象，而释放对象的根本原则就是该对象不再被引用。

为了更好理解 GC 的工作原理，我们可以将对象考虑为有向图的顶点，将引用关系考虑为图的有向边，有向边从引用者指向被引对象。另外，每个线程对象可以作为一个图的起始顶点，例如大多程序从 main 进程开始执行，那么该图就是以 main 进程顶点开始的一棵根树。在这个有向图中，根顶点可达的对象都是有效对象，GC将不回收这些对象。如果某个对象 (连通子图)与这个根顶点不可达(注意，该图为有向图)，那么我们认为这个(这些)对象不再被引用，可以被 GC 回收。 以下，我们举一个例子说明如何用有向图表示内存管理。对于程序的每一个时刻，我们都有一个有向图表示JVM的内存分配情况。以下右图，就是左边程序运行到第6行的示意图。

[![img](http://www.ibm.com/developerworks/cn/java/l-JavaMemoryLeak/1.gif)](javascript:;)

Java使用有向图的方式进行内存管理，可以消除引用循环的问题，例如有三个对象，相互引用，只要它们和根进程不可达的，那么GC也是可以回收它们的。这种方式的优点是管理内存的精度很高，但是效率较低。另外一种常用的内存管理技术是使用计数器，例如COM模型采用计数器方式管理构件，它与有向图相比，精度行低(很难处理循环引用的问题)，但执行效率很高。

### 什么是Java中的内存泄露

在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

在C++中，内存泄漏的范围更大一些。有些对象被分配了内存空间，然后却不可达，由于C++中没有GC，这些内存将永远收不回来。在Java中，这些不可达的对象都由GC负责回收，因此程序员不需要考虑这部分的内存泄露。

通过分析，我们得知，对于C++，程序员需要自己管理边和顶点，而对于Java程序员只需要管理边就可以了(不需要管理顶点的释放)。通过这种方式，Java提高了编程的效率。

[![img](http://www.ibm.com/developerworks/cn/java/l-JavaMemoryLeak/2.gif)](javascript:;)

因此，通过以上分析，我们知道在Java中也有内存泄漏，但范围比C++要小一些。因为Java从语言上保证，任何对象都是可达的，所有的不可达对象都由GC管理。

对于程序员来说，GC基本是透明的，不可见的。虽然，我们只有几个函数可以访问GC，例如运行GC的函数System.gc()，但是根据Java语言规范定义， 该函数不保证JVM的垃圾收集器一定会执行。因为，不同的JVM实现者可能使用不同的算法管理GC。通常，GC的线程的优先级别较低。JVM调用GC的策略也有很多种，有的是内存使用到达一定程度时，GC才开始工作，也有定时执行的，有的是平缓执行GC，有的是中断式执行GC。但通常来说，我们不需要关心这些。除非在一些特定的场合，GC的执行影响应用程序的性能，例如对于基于Web的实时系统，如网络游戏等，用户不希望GC突然中断应用程序执行而进行垃圾回收，那么我们需要调整GC的参数，让GC能够通过平缓的方式释放内存，例如将垃圾回收分解为一系列的小步骤执行，Sun提供的HotSpot JVM就支持这一特性。

同样给出一个 Java 内存泄漏的典型例子，

```Java
Vector v = new Vector(10);
for (int i = 1; i < 100; i++) {
    Object o = new Object();
    v.add(o);
    o = null;   
}
```

在这个例子中，我们循环申请Object对象，并将所申请的对象放入一个 Vector 中，如果我们仅仅释放引用本身，那么 Vector 仍然引用该对象，所以这个对象对 GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从 Vector 中删除，最简单的方法就是将 Vector 对象设置为 null。

### Android中常见的内存泄漏汇总

- 集合类泄漏

  集合类如果仅仅有添加元素的方法，而没有相应的删除机制，导致内存被占用。如果这个集合类是全局性的变量 (比如类中的静态属性，全局性的 map 等即有静态引用或 final 一直指向它)，那么没有相应的删除机制，很可能导致集合所占用的内存只增不减。比如上面的典型例子就是其中一种情况，当然实际上我们在项目中肯定不会写这么 2B 的代码，但稍不注意还是很容易出现这种情况，比如我们都喜欢通过 HashMap 做一些缓存之类的事，这种情况就要多留一些心眼。

- 单例造成的内存泄漏

  由于单例的静态特性使得其生命周期跟应用的生命周期一样长，所以如果使用不恰当的话，很容易造成内存泄漏。比如下面一个典型的例子，

```java
public class AppManager {
  private static AppManager instance;
  private Context context;
  private AppManager(Context context) {
  this.context = context;
  }
  public static AppManager getInstance(Context context) {
    if (instance == null) {
    instance = new AppManager(context);
    }
    return instance;
  }
}
```

这是一个普通的单例模式，当创建这个单例的时候，由于需要传入一个Context，所以这个Context的生命周期的长短至关重要：

1、如果此时传入的是 Application 的 Context，因为 Application 的生命周期就是整个应用的生命周期，所以这将没有任何问题。

2、如果此时传入的是 Activity 的 Context，当这个 Context 所对应的 Activity 退出时，由于该 Context 的引用被单例对象所持有，其生命周期等于整个应用程序的生命周期，所以当前 Activity 退出时它的内存并不会被回收，这就造成泄漏了。

正确的方式应该改为下面这种方式：

```java
public class AppManager {
  private static AppManager instance;
  private Context context;
  private AppManager(Context context) {
    this.context = context.getApplicationContext();// 使用Application 的context
 }
  public static AppManager getInstance(Context context) {
    if (instance == null) {
      instance = new AppManager(context);
    }
    return instance;
  }
}
```

或者这样写，连 Context 都不用传进来了：

```java
在你的 Application 中添加一个静态方法，getContext() 返回 Application 的 context，

...

context = getApplicationContext();

...  
   /**
     * 获取全局的context
     * @return 返回全局context对象
     */
    public static Context getContext(){
        return context;
    }

public class AppManager {
  private static AppManager instance;
  private Context context;
  private AppManager() {
    this.context = MyApplication.getContext();// 使用Application 的context
  }
  public static AppManager getInstance() {
    if (instance == null) {
      instance = new AppManager();
    }
    return instance;
  }
}
```

- 匿名内部类/非静态内部类和异步线程

  - 非静态内部类创建静态实例造成的内存泄漏

    有的时候我们可能会在启动频繁的Activity中，为了避免重复创建相同的数据资源，可能会出现这种写法：

    ```java
    public class MainActivity extends AppCompatActivity {
      private static TestResource mResource = null;
      @Override
      protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(mManager == null){
          mManager = new TestResource();
        }
        //...
        }
      class TestResource {
      //...
      }
    }
    ```

这样就在Activity内部创建了一个非静态内部类的单例，每次启动Activity时都会使用该单例的数据，这样虽然避免了资源的重复创建，不过这种写法却会造成内存泄漏，因为非静态内部类默认会持有外部类的引用，而该非静态内部类又创建了一个静态的实例，该实例的生命周期和应用的一样长，这就导致了该静态实例一直会持有该Activity的引用，导致Activity的内存资源不能正常回收。正确的做法为：

将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，请按照上面推荐的使用Application 的 Context。当然，Application 的 context 不是万能的，所以也不能随便乱用，对于有些地方则必须使用 Activity 的 Context，对于Application，Service，Activity三者的Context的应用场景如下：

[![img](http://img.blog.csdn.net/20151123144226349)](javascript:;)

**其中：** NO1表示 Application 和 Service 可以启动一个 Activity，不过需要创建一个新的 task 任务队列。而对于 Dialog 而言，只有在 Activity 中才能创建

- 匿名内部类

  android开发经常会继承实现Activity/Fragment/View，此时如果你使用了匿名类，并被异步线程持有了，那要小心了，如果没有任何措施这样一定会导致泄露

  ```java
  public class MainActivity extends Activity {
    ...
    Runnable ref1 = new MyRunable();
    Runnable ref2 = new Runnable() {
        @Override
        public void run() {
  
        }
    };
     ...
  }
  ```

ref1和ref2的区别是，ref2使用了匿名内部类。我们来看看运行时这两个引用的内存：

[![img](http://img2.tbcdn.cn/L1/461/1/fb05ff6d2e68f309b94dd84352c81acfe0ae839e)](javascript:;)

可以看到，ref1没什么特别的。 但ref2这个匿名类的实现对象里面多了一个引用： this$0这个引用指向MainActivity.this，也就是说当前的MainActivity实例会被ref2持有，如果将这个引用再传入一个异步线程，此线程和此Acitivity生命周期不一致的时候，就造成了Activity的泄露。

- Handler 造成的内存泄漏

  Handler 的使用造成的内存泄漏问题应该说是最为常见了，很多时候我们为了避免 ANR 而不在主线程进行耗时操作，在处理网络任务或者封装一些请求回调等api都借助Handler来处理，但 Handler 不是万能的，对于 Handler 的使用代码编写一不规范即有可能造成内存泄漏。另外，我们知道 Handler、Message 和 MessageQueue 都是相互关联在一起的，万一 Handler 发送的 Message 尚未被处理，则该 Message 及发送它的 Handler 对象将被线程 MessageQueue 一直持有。 由于 Handler 属于 TLS(Thread Local Storage) 变量, 生命周期和 Activity 是不一致的。因此这种实现方式一般很难保证跟 View 或者 Activity 的生命周期保持一致，故很容易导致无法正确释放。

  举个例子：

  ```java
  public class SampleActivity extends Activity {
  
    private final Handler mLeakyHandler = new Handler() {
      @Override
      public void handleMessage(Message msg) {
        // ...
      }
    }
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
  
      // Post a message and delay its execution for 10 minutes.
      mLeakyHandler.postDelayed(new Runnable() {
        @Override
        public void run() { /* ... */ }
      }, 1000 * 60 * 10);
  
      // Go back to the previous Activity.
      finish();
    }
  }
  ```

在该 SampleActivity 中声明了一个延迟10分钟执行的消息 Message，mLeakyHandler 将其 push 进了消息队列 MessageQueue 里。当该 Activity 被 finish() 掉时，延迟执行任务的 Message 还会继续存在于主线程中，它持有该 Activity 的 Handler 引用，所以此时 finish() 掉的 Activity 就不会被回收了从而造成内存泄漏（因 Handler 为非静态内部类，它会持有外部类的引用，在这里就是指 SampleActivity）。

修复方法：在 Activity 中避免使用非静态内部类，比如上面我们将 Handler 声明为静态的，则其存活期跟 Activity 的生命周期就无关了。同时通过弱引用的方式引入 Activity，避免直接将 Activity 作为 context 传进去，见下面代码：

```java
public class SampleActivity extends Activity {

  /**
   * Instances of static inner classes do not hold an implicit
   * reference to their outer class.
   */
  private static class MyHandler extends Handler {
    private final WeakReference<SampleActivity> mActivity;

    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<SampleActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }

  private final MyHandler mHandler = new MyHandler(this);

  /**
   * Instances of anonymous classes do not hold an implicit
   * reference to their outer class when they are "static".
   */
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() { /* ... */ }
  };

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
  }
}
```

综述，即推荐使用静态内部类 + WeakReference 这种方式。每次使用前注意判空。

前面提到了 WeakReference，所以这里就简单的说一下 Java 对象的几种引用类型。

Java对引用的分类有 Strong reference, SoftReference, WeakReference, PhatomReference 四种。

[![img](https://gw.alicdn.com/tps/TB1U6TNLVXXXXchXFXXXXXXXXXX-644-546.jpg)](javascript:;)

在Android应用的开发中，为了防止内存溢出，在处理一些占用内存大而且声明周期较长的对象时候，可以尽量应用软引用和弱引用技术。

软/弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。利用这个队列可以得知被回收的软/弱引用的对象列表，从而为缓冲器清除已失效的软/弱引用。

假设我们的应用会用到大量的默认图片，比如应用中有默认的头像，默认游戏图标等等，这些图片很多地方会用到。如果每次都去读取图片，由于读取文件需要硬件操作，速度较慢，会导致性能较低。所以我们考虑将图片缓存起来，需要的时候直接从内存中读取。但是，由于图片占用内存空间比较大，缓存很多图片需要很多的内存，就可能比较容易发生OutOfMemory异常。这时，我们可以考虑使用软/弱引用技术来避免这个问题发生。以下就是高速缓冲器的雏形：

首先定义一个HashMap，保存软引用对象。

```java
private Map <String, SoftReference<Bitmap>> imageCache = new HashMap <String, SoftReference<Bitmap>> ();
```

再来定义一个方法，保存Bitmap的软引用到HashMap。

[![img](https://gw.alicdn.com/tps/TB1oW_FLVXXXXXuaXXXXXXXXXXX-679-717.jpg)](javascript:;)

使用软引用以后，在OutOfMemory异常发生之前，这些缓存的图片资源的内存空间可以被释放掉的，从而避免内存达到上限，避免Crash发生。

如果只是想避免OutOfMemory异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。

另外可以根据对象是否经常使用来判断选择软引用还是弱引用。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。

ok，继续回到主题。前面所说的，创建一个静态Handler内部类，然后对 Handler 持有的对象使用弱引用，这样在回收时也可以回收 Handler 持有的对象，但是这样做虽然避免了 Activity 泄漏，不过 Looper 线程的消息队列中还是可能会有待处理的消息，所以我们在 Activity 的 Destroy 时或者 Stop 时应该移除消息队列 MessageQueue 中的消息。

下面几个方法都可以移除 Message：

```java
public final void removeCallbacks(Runnable r);

public final void removeCallbacks(Runnable r, Object token);

public final void removeCallbacksAndMessages(Object token);

public final void removeMessages(int what);

public final void removeMessages(int what, Object object);
```

- 尽量避免使用 static 成员变量

  如果成员变量被声明为 static，那我们都知道其生命周期将与整个app进程生命周期一样。

  这会导致一系列问题，如果你的app进程设计上是长驻内存的，那即使app切到后台，这部分内存也不会被释放。按照现在手机app内存管理机制，占内存较大的后台进程将优先回收，因为如果此app做过进程互保保活，那会造成app在后台频繁重启。当手机安装了你参与开发的app以后一夜时间手机被消耗空了电量、流量，你的app不得不被用户卸载或者静默。 这里修复的方法是：

不要在类初始时初始化静态成员。可以考虑lazy初始化。 架构设计上要思考是否真的有必要这样做，尽量避免。如果架构需要这么设计，那么此对象的生命周期你有责任管理起来。

- 避免 override finalize()

  1、finalize 方法被执行的时间不确定，不能依赖与它来释放紧缺的资源。时间不确定的原因是：

  - 虚拟机调用GC的时间不确定
  - Finalize daemon线程被调度到的时间不确定

  2、finalize 方法只会被执行一次，即使对象被复活，如果已经执行过了 finalize 方法，再次被 GC 时也不会再执行了，原因是：

  含有 finalize 方法的 object 是在 new 的时候由虚拟机生成了一个 finalize reference 在来引用到该Object的，而在 finalize 方法执行的时候，该 object 所对应的 finalize Reference 会被释放掉，即使在这个时候把该 object 复活(即用强引用引用住该 object )，再第二次被 GC 的时候由于没有了 finalize reference 与之对应，所以 finalize 方法不会再执行。

  3、含有Finalize方法的object需要至少经过两轮GC才有可能被释放。

  详情见这里 [深入分析过dalvik的代码](http://blog.csdn.net/kai_gong/article/details/24188803)

- 资源未关闭造成的内存泄漏

  对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。

- 一些不良代码造成的内存压力

  有些代码并不造成内存泄露，但是它们，或是对没使用的内存没进行有效及时的释放，或是没有有效的利用已有的对象而是频繁的申请新内存。

  比如：

  - 构造 Adapter 时，没有使用缓存的 convertView ,每次都在创建新的 converView。这里推荐使用 ViewHolder。

### 工具分析

Java 内存泄漏的分析工具有很多，但众所周知的要数 MAT(Memory Analysis Tools) 和 YourKit 了。由于篇幅问题，我这里就只对 [MAT](http://www.eclipse.org/mat/) 的使用做一下介绍。--> [MAT 的安装](http://www.ibm.com/developerworks/cn/opensource/os-cn-ecl-ma/index.html)

- MAT分析heap的总内存占用大小来初步判断是否存在泄露

  打开 DDMS 工具，在左边 Devices 视图页面选中“Update Heap”图标，然后在右边切换到 Heap 视图，点击 Heap 视图中的“Cause GC”按钮，到此为止需检测的进程就可以被监视。

  [![img](https://gw.alicdn.com/tps/TB1pdf2LVXXXXXeXXXXXXXXXXXX-690-514.jpg)](javascript:;)

  Heap视图中部有一个Type叫做data object，即数据对象，也就是我们的程序中大量存在的类类型的对象。在data object一行中有一列是“Total Size”，其值就是当前进程中所有Java数据对象的内存总量，一般情况下，这个值的大小决定了是否会有内存泄漏。可以这样判断：

  进入某应用，不断的操作该应用，同时注意观察data object的Total Size值，正常情况下Total Size值都会稳定在一个有限的范围内，也就是说由于程序中的的代码良好，没有造成对象不被垃圾回收的情况。

  所以说虽然我们不断的操作会不断的生成很多对象，而在虚拟机不断的进行GC的过程中，这些对象都被回收了，内存占用量会会落到一个稳定的水平；反之如果代码中存在没有释放对象引用的情况，则data object的Total Size值在每次GC后不会有明显的回落。随着操作次数的增多Total Size的值会越来越大，直到到达一个上限后导致进程被杀掉。

- MAT分析hprof来定位内存泄露的原因所在

  这是出现内存泄露后使用MAT进行问题定位的有效手段。

  A)Dump出内存泄露当时的内存镜像hprof，分析怀疑泄露的类：

  [![img](https://gw.alicdn.com/tps/TB1r2zZLVXXXXcHXXXXXXXXXXXX-640-167.jpg)](javascript:;)

  B)分析持有此类对象引用的外部对象

  [![img](https://gw.alicdn.com/tps/TB17XvOLVXXXXbiXFXXXXXXXXXX-640-90.png)](javascript:;)

  C)分析这些持有引用的对象的GC路径

  [![img](https://gw.alicdn.com/tps/TB10yTwLVXXXXaRapXXXXXXXXXX-640-278.png)](javascript:;)

  D)逐个分析每个对象的GC路径是否正常

  [![img](https://gw.alicdn.com/tps/TB1CWTQLVXXXXamXFXXXXXXXXXX-640-90.png)](javascript:;)

  从这个路径可以看出是一个antiRadiationUtil工具类对象持有了MainActivity的引用导致MainActivity无法释放。此时就要进入代码分析此时antiRadiationUtil的引用持有是否合理（如果antiRadiationUtil持有了MainActivity的context导致节目退出后MainActivity无法销毁，那一般都属于内存泄露了）。

- MAT对比操作前后的hprof来定位内存泄露的根因所在

  为查找内存泄漏，通常需要两个 Dump结果作对比，打开 Navigator History面板，将两个表的 Histogram结果都添加到 Compare Basket中去

  A） 第一个HPROF 文件(usingFile > Open Heap Dump ).

  B）打开Histogram view.

  C）在NavigationHistory view里 (如果看不到就从Window >show view>MAT- Navigation History ), 右击histogram然后选择Add to Compare Basket .

  [![img](https://gw.alicdn.com/tps/TB1p1rULVXXXXbyXpXXXXXXXXXX-525-212.png)](javascript:;)

  D）打开第二个HPROF 文件然后重做步骤2和3.

  E）切换到Compare Basket view, 然后点击Compare the Results (视图右上角的红色”!”图标)。

  [![img](https://gw.alicdn.com/tps/TB1p0zKLVXXXXX.XVXXXXXXXXXX-640-98.png)](javascript:;)

  F）分析对比结果

  [![img](https://gw.alicdn.com/tps/TB1lwDMLVXXXXcUXFXXXXXXXXXX-640-115.png)](javascript:;)

  可以看出两个hprof的数据对象对比结果。

  通过这种方式可以快速定位到操作前后所持有的对象增量，从而进一步定位出当前操作导致内存泄露的具体原因是泄露了什么数据对象。

  注意：

  如果是用 MAT Eclipse 插件获取的 Dump文件，不需要经过转换则可在MAT中打开，Adt会自动进行转换。

  而手机SDk Dump 出的文件要经过转换才能被 MAT识别，Android SDK提供了这个工具 hprof-conv (位于 sdk/tools下)

  首先，要通过控制台进入到你的 android sdk tools 目录下执行以下命令：

  ./hprof-conv xxx-a.hprof xxx-b.hprof

  例如 hprof-conv input.hprof out.hprof

  此时才能将out.hprof放在eclipse的MAT中打开。

Ok，下面将给大家介绍一个屌炸天的工具 -- LeakCanary 。

### 使用 LeakCanary 检测 Android 的内存泄漏

什么是 [LeakCanary](https://github.com/square/leakcanary) 呢？为什么选择它来检测 Android 的内存泄漏呢？

别急，让我来慢慢告诉大家！

LeakCanary 是国外一位大神 Pierre-Yves Ricau 开发的一个用于检测内存泄露的开源类库。一般情况下，在对战内存泄露中，我们都会经过以下几个关键步骤：

1、了解 OutOfMemoryError 情况。

2、重现问题。

3、在发生内存泄露的时候，把内存 Dump 出来。

4、在发生内存泄露的时候，把内存 Dump 出来。

5、计算这个对象到 GC roots 的最短强引用路径。

6、确定引用路径中的哪个引用是不该有的，然后修复问题。

很复杂对吧？

如果有一个类库能在发生 OOM 之前把这些事情全部都搞定，然后你只要修复这些问题就好了。LeakCanary 做的就是这件事情。你可以在 debug 包中轻松检测内存泄露。

一起来看这个例子（摘自 LeakCanary 中文使用说明，下面会附上所有的参考文档链接）：

```java
class Cat {
}

class Box {
  Cat hiddenCat;
}
class Docker {
    // 静态变量，将不会被回收，除非加载 Docker 类的 ClassLoader 被回收。
    static Box container;
}

// ...

Box box = new Box();

// 薛定谔之猫
Cat schrodingerCat = new Cat();
box.hiddenCat = schrodingerCat;
Docker.container = box;
```

创建一个RefWatcher，监控对象引用情况。

```java
// 我们期待薛定谔之猫很快就会消失（或者不消失），我们监控一下
refWatcher.watch(schrodingerCat);
```

当发现有内存泄露的时候，你会看到一个很漂亮的 leak trace 报告:

- GC ROOT static Docker.container
- references Box.hiddenCat
- leaks Cat instance

我们知道，你很忙，每天都有一大堆需求。所以我们把这个事情弄得很简单，你只需要添加一行代码就行了。然后 LeakCanary 就会自动侦测 activity 的内存泄露了。

```java
public class ExampleApplication extends Application {
  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```

然后你会在通知栏看到这样很漂亮的一个界面:

[![img](https://corner.squareup.com/images/leakcanary/leaktrace.png)](javascript:;)

以很直白的方式将内存泄露展现在我们的面前。

### Demo

一个非常简单的 LeakCanary demo: [一个非常简单的 LeakCanary demo: https://github.com/liaohuqiu/leakcanary-demo](https://github.com/liaohuqiu/leakcanary-demo)

#### 接入

在 build.gradle 中加入引用，不同的编译使用不同的引用：

```java
 dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.3'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.3'
 }
```

#### 如何使用

使用 RefWatcher 监控那些本该被回收的对象。

```java
RefWatcher refWatcher = {...};

// 监控
refWatcher.watch(schrodingerCat);
```

LeakCanary.install() 会返回一个预定义的 RefWatcher，同时也会启用一个 ActivityRefWatcher，用于自动监控调用 Activity.onDestroy() 之后泄露的 activity。

在Application中进行配置 ：

```java
public class ExampleApplication extends Application {

  public static RefWatcher getRefWatcher(Context context) {
    ExampleApplication application = (ExampleApplication) context.getApplicationContext();
    return application.refWatcher;
  }

  private RefWatcher refWatcher;

  @Override public void onCreate() {
    super.onCreate();
    refWatcher = LeakCanary.install(this);
  }
}
```

使用 RefWatcher 监控 Fragment：

```java
public abstract class BaseFragment extends Fragment {

  @Override public void onDestroy() {
    super.onDestroy();
    RefWatcher refWatcher = ExampleApplication.getRefWatcher(getActivity());
    refWatcher.watch(this);
  }
}
```

使用 RefWatcher 监控 Activity：

```java
public class MainActivity extends AppCompatActivity {

    ......
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
            //在自己的应用初始Activity中加入如下两行代码
        RefWatcher refWatcher = ExampleApplication.getRefWatcher(this);
        refWatcher.watch(this);

        textView = (TextView) findViewById(R.id.tv);
        textView.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startAsyncTask();
            }
        });

    }

    private void async() {

        startAsyncTask();
    }

    private void startAsyncTask() {
        // This async task is an anonymous class and therefore has a hidden reference to the outer
        // class MainActivity. If the activity gets destroyed before the task finishes (e.g. rotation),
        // the activity instance will leak.
        new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... params) {
                // Do some slow work in background
                SystemClock.sleep(20000);
                return null;
            }
        }.execute();
    }


}
```

### 工作机制

1.RefWatcher.watch() 创建一个 KeyedWeakReference 到要被监控的对象。

2.然后在后台线程检查引用是否被清除，如果没有，调用GC。

3.如果引用还是未被清除，把 heap 内存 dump 到 APP 对应的文件系统中的一个 .hprof 文件中。

4.在另外一个进程中的 HeapAnalyzerService 有一个 HeapAnalyzer 使用HAHA 解析这个文件。

5.得益于唯一的 reference key, HeapAnalyzer 找到 KeyedWeakReference，定位内存泄露。

6.HeapAnalyzer 计算 到 GC roots 的最短强引用路径，并确定是否是泄露。如果是的话，建立导致泄露的引用链。

7.引用链传递到 APP 进程中的 DisplayLeakService， 并以通知的形式展示出来。

ok,这里就不再深入了，想要了解更多就到 [作者 github 主页](https://github.com/square/leakcanary) 这去哈。

### 总结

- 对 Activity 等组件的引用应该控制在 Activity 的生命周期之内； 如果不能就考虑使用 getApplicationContext 或者 getApplication，以避免 Activity 被外部长生命周期的对象引用而泄露。
- 尽量不要在静态变量或者静态内部类中使用非静态外部成员变量（包括context )，即使要使用，也要考虑适时把外部成员变量置空；也可以在内部类中使用弱引用来引用外部类的变量。
- 对于生命周期比Activity长的内部类对象，并且内部类中使用了外部类的成员变量，可以这样做避免内存泄漏：
  - 将内部类改为静态内部类
  - 静态内部类中使用弱引用来引用外部类的成员变量
- Handler 的持有的引用对象最好使用弱引用，资源释放时也可以清空 Handler 里面的消息。比如在 Activity onStop 或者 onDestroy 的时候，取消掉该 Handler 对象的 Message和 Runnable.
- 在 Java 的实现过程中，也要考虑其对象释放，最好的方法是在不使用某对象时，显式地将此对象赋值为 null，比如使用完Bitmap 后先调用 recycle()，再赋为null,清空对图片等资源有直接引用或者间接引用的数组（使用 array.clear() ; array = null）等，最好遵循谁创建谁释放的原则。
- 正确关闭资源，对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销。
- 保持对对象生命周期的敏感，特别注意单例、静态对象、全局性集合等的生命周期。





## Android布局优化之include、merge、ViewStub的使用

### 一、`<include/>`

标签在布局优化中是使用最多的一个标签了，它就是为了解决重复定义布局的问题。标签就相当于C、C++中的include头文件一样，把一些常用的底层的API封装起来，需要的时候引入即可。在一些开源的J2EE中许多XML配置文件也都会使用标签，将多个配置文件组合成为一个更为复杂的配置文件，如最常见的S2SH。

在以前Android开发中，由于ActionBar设计上的不统一以及兼容性问题，所以很多应用都自定义了一套自己的标题栏titlebar。标题栏我们知道在应用的每个界面几乎都会用到，在这里可以作为一个很好的示例来解释标签的使用。

下面是一个自定义的titlebar文件：

```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/titlebar_bg">

    <ImageView android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:src="@drawable/gafricalogo" />
</FrameLayout>
```

在应用中使用titlebar布局文件，我们通过标签,布局文件如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/app_bg"
    android:gravity="center_horizontal">

    <include layout="@layout/titlebar"/>

    <TextView android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:text="@string/hello"
              android:padding="10dp" />

    ...

</LinearLayout>
```

在标签中可以覆盖导入的布局文件root布局的布局属性（如layout_*属性）。

布局示例如下：

```xml
<include android:id="@+id/news_title"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         layout="@layout/title"/>
```

如果想使用标签覆盖嵌入布局root布局属性，必须同时覆盖layout_height和layout_width属性，否则会直接报编译时语法错误。

> *Layout parameter layout_height ignored unless layout_width is also specified on tag*

如果标签已经定义了id，而嵌入布局文件的root布局文件也定义了id，标签的id会覆盖掉嵌入布局文件root的id，如果include标签没有定义id则会使用嵌入文件root的id。

### 二、`<merge/>`

标签都是与标签组合使用的，它的作用就是可以有效减少View树的层次来优化布局。

下面通过一个简单的示例探讨一下标签的使用，下面是嵌套布局的layout_text.xml文件：

```Xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical" >
    <TextView
        android:id="@+id/textView"
        android:layout_width="match_parent"
        android:text="Hello World!"
        android:layout_height="match_parent" />
</LinearLayout>
```

一个线性布局中嵌套一个文本视图，主布局如下：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/layout_wrap"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical" >

    <include
        android:id="@+id/layout_import"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        layout="@layout/layout_text" />

</LinearLayout>
```

通过hierarchyviewer我们可以看到主布局View树的部分层级结构如下图：

[![layout_merge01](http://www.sunnyang.com/wp-content/uploads/2016/04/layout_merge01.png)](http://www.sunnyang.com/wp-content/uploads/2016/04/layout_merge01.png)

现在讲嵌套布局跟布局标签更改为，merge_text.xml布局文件如下：

```xml
<merge xmlns:android="http://schemas.android.com/apk/res/android" >

    <TextView
        android:id="@+id/textView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:text="Hello World!"/>

</merge>
```

然后将主布局标签中的layout更改为merge_text.xml，运行后重新截图如下:

[![layout_merge02](http://www.sunnyang.com/wp-content/uploads/2016/04/layout_merge02.png)](http://www.sunnyang.com/wp-content/uploads/2016/04/layout_merge02.png)

对比截图就可以发现上面的四层结构，现在已经是三层结构了。当我们使用标签的时候，系统会自动忽略merge层级，而把TextView直接放置与平级。

标签在使用的时候需要特别注意布局的类型，例如我的标签中包含的是一个LinearLayout布局视图，布局中的元素是线性排列的，如果嵌套进主布局时，include标签父布局时FrameLayout，这种方式嵌套肯定会出问题的，merge中元素会按照FrameLayout布局方式显示。所以在使用的时候，标签虽然可以减少布局层级，但是它的限制也不可小觑。

只能作为XML布局的根标签使用。当Inflate以开头的布局文件时，必须指定一个父ViewGroup，并且必须设定attachToRoot为true。

```
View android.view.LayoutInflater.inflate(int resource, ViewGroup root, boolean attachToRoot)
```

root不可少，attachToRoot必须为true。

### 三、ViewStub

在开发过程中，经常会遇到这样一种情况，有些布局很复杂但是却很少使用。例如条目详情、进度条标识或者未读消息等，这些情况如果在一开始初始化，虽然设置可见性`View.GONE`,但是在Inflate的时候View仍然会被Inflate，仍然会创建对象，由于这些布局又想到复杂，所以会很消耗系统资源。

ViewStub就是为了解决上面问题的，ViewStub是一个轻量级的View，它一个看不见的，不占布局位置，占用资源非常小的控件。

#### 定义ViewStub布局文件

下面是一个ViewStub布局文件：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/layout_wrap"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical" >

    <ViewStub
        android:id="@+id/stub_image"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inflatedId="@+id/image_import"
        android:layout="@layout/layout_image" />

    <ViewStub
        android:id="@+id/stub_text"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:inflatedId="@+id/text_import"
        android:layout="@layout/layout_text" />

</LinearLayout>
```

layout_image.xml文件如下（layout_text.xml类似）：

```Xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:id="@+id/layout_image">

    <ImageView
        android:id="@+id/imageView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

</LinearLayout>
```

#### 加载ViewStub布局文件

动态加载ViewStub所包含的布局文件有两种方式，方式一使用使用inflate()方法，方式二就是使用setVisibility(View.VISIBLE)。

示例java代码如下：

```java
private ViewStub viewStub;

protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.layout_main2);
    viewStub = (ViewStub) findViewById(R.id.stub_image);
    //viewStub.inflate();//方式一
    viewStub.setVisibility(View.VISIBLE);//方式二
    ImageView imageView = (ImageView) findViewById(R.id.imageView);
    imageView.setImageResource(R.drawable.image);
}
```

示例View层级截图如下：

[![viewstub_view](http://www.sunnyang.com/wp-content/uploads/2016/04/viewstub_view.png)](http://www.sunnyang.com/wp-content/uploads/2016/04/viewstub_view.png)

ViewStub一旦visible/inflated,它自己就不在是View试图层级的一部分了。所以后面无法再使用ViewStub来控制布局，填充布局root布局如果有id，则会默认被android:inflatedId所设置的id取代，如果没有设置android:inflatedId，则会直接使用填充布局id。

由于ViewStub这种使用后即可就置空的策略，所以当需要在运行时不止一次的显示和隐藏某个布局，那么ViewStub是做不到的。这时就只能使用View的可见性来控制了。

layout_*相关属性与include标签相似，如果使用应该在ViewStub上面使用，否则使用在嵌套进来布局root上面无效。

ViewStub的另一个缺点就是目前还不支持merge标签。

### 四、小结

Android布局优化基本上就设计上面include、merge、ViewStub三个标签的使用。在平常开发中布局推荐使用RelativeLayout，它也可以有效减少布局层级嵌套。最后了将merge和include源码附上，ViewStub就是一个View，就不贴出来了。

#### Include源码

```java
/**
 * Exercise <include /> tag in XML files.
 */
public class Include extends Activity {
    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        setContentView(R.layout.include_tag);
    }
}
```

#### Merge源码

```java
/**
 * Exercise <merge /> tag in XML files.
 */
public class Merge extends Activity {
    private LinearLayout mLayout;

    @Override
    protected void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        mLayout = new LinearLayout(this);
        mLayout.setOrientation(LinearLayout.VERTICAL);
        LayoutInflater.from(this).inflate(R.layout.merge_tag, mLayout);

        setContentView(mLayout);
    }

    public ViewGroup getLayout() {
        return mLayout;
    }
}
```





## Android 换肤相关

### 布局创建流程

以 Activity 中加载 XML 布局的过程作为切入点来了解这一流程，这部分其实和换肤功能关联性不大，但是作为基础知识还是过一遍流程吧。

我们一般都通过 setContentView 方法给 Activity 设置布局，如下所示：

```java
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main) // 核心代码
    }
}
```

#### 源码流程

跟踪 setContentView 源码到 Activity 类中：

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

最终调用了 mWindow 的 setContentView 方法，那么就从这里开始，先来分析下 mWindow 是怎么来的。此处调用的生命周期在 onCreate，那么说明 mWindow 的初始化在 onCreate 之前，直接跳到 ActivityThread 类中的 performLaunchActivity 方法查看（关于 Activity 的启动流程请自行查阅资料）：

ActivityThread.java

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    Activity activity = null;
    try {
        // 通过反射创建 Activity
        java.lang.ClassLoader cl = appContext.getClassLoader();
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
    }
    // ...
    // 注意这个 attach 方法在触发 onCreate 之前调用
    activity.attach(appContext, this, getInstrumentation(), r.token,
            r.ident, app, r.intent, r.activityInfo, title, r.parent,
            r.embeddedID, r.lastNonConfigurationInstances, config,
            r.referrer, r.voiceInteractor, window, r.configCallback,
            r.assistToken, r.shareableActivityToken);
    // ...
    // 这里是触发 Activity onCreate 生命周期
    mInstrumentation.callActivityOnCreate(activity, r.state);
    // ...
}
```

看一下 Activity attach 方法中 mWindow 初始化部分：

Activity.java

```java
final void attach(Context context, ActivityThread aThread, Instrumentation instr, 参数省略...) {
    // ...
    // mWindow 初始化
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    // ...
    // 给 mWindow 中的 mWindowManager 赋值
    mWindow.setWindowManager(
        (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    // 将上面 mWindow 中的 mWindowManager 又赋值给 Activity 的 mWindowManager
    mWindowManager = mWindow.getWindowManager();
    // ...
}
```

从上面源码中可以知道 mWindow 是 PhoneWindow，在 Activity 中调用 setContentView 就直接调用到了 PhoneWindow 中，看一下 PhoneWindow 中的源码都做了什么：

PhoneWindow.java

```
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor(); // 核心 1
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        // ...
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent); // 核心 2
    }
    // ...
}
```

上述代码主要有两处核心代码，逐个来进行分析，先记住一点，此时的 Activity 还是一片空白。

#### installDecor()

此时的 mContentParent 还是 null，进入 installDecor 方法看源码：

PhoneWindow.java

```java
private void installDecor() {
    // ...
    if (mDecor == null) {
        // 初始化 mDecor
        mDecor = generateDecor(-1);
        // ...
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        // 初始化 mContentParent
        mContentParent = generateLayout(mDecor);
        // ...
    }
}
```

mDecor 初始化代码：

PhoneWindow.java

```java
protected DecorView generateDecor(int featureId) {
    // ...
    // 直接 new DecorView 进行初始化
    // 注意第三个参数 this，将当前 PhoneWindow 对象赋值给 DecorView 中的 mWindow
    return new DecorView(context, featureId, this, getAttributes());
}
```

注意这个 DecorView 是继承自 FrameLayout 就不贴源码了，重点来看一下 mContentParent 的初始化代码：

PhoneWindow.java

```java
protected ViewGroup generateLayout(DecorView decor) {
    // ...
    // 先是一系列的属性设置贴了一些平时常用的
    // 取消标题栏
    if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
        requestFeature(FEATURE_NO_TITLE);
    } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
        // Don't allow an action bar if there is no title.
        requestFeature(FEATURE_ACTION_BAR);
    }
    // ...
    // 设置全屏
    if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
        setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
    }
    // ...
    WindowManager.LayoutParams params = getAttributes();
    // 一系列的窗口属性设置
    // 如：SDK 31 新增的高斯模糊
    if (a.getBoolean(R.styleable.Window_windowBlurBehindEnabled, false)) {
        if ((getForcedWindowFlags() & WindowManager.LayoutParams.FLAG_BLUR_BEHIND) == 0) {
            params.flags |= WindowManager.LayoutParams.FLAG_BLUR_BEHIND;
        }
    
        params.setBlurBehindRadius(a.getDimensionPixelSize(
                android.R.styleable.Window_windowBlurBehindRadius, 0));
    }
    // ...
    // 整体布局文件
    int layoutResource;
    // 根据一系列判断选择 SDK 中的布局一般默认是 R.layout.screen_simple
    if (...){
    }else if(...){
    }else{
        layoutResource = R.layout.screen_simple;
    }
    
    mDecor.onResourcesLoaded(mLayoutInflater, layoutResource);
    
    // 拿到 screen_simple.xml 布局的内容部分 （R.id.content）
    ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
    // ...
    return contentParent; // 返回
}
```

最终，是调用了 DecorView 的 onResourcesLoaded 方法，并且将 R.layout.screen_simple 布局传递过去，先来看一下布局文件源码：

R.layout.screen_simple.xml

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```

可以看出整个 Activity 默认是一个 LinearLayout 布局中包含着标题栏和内容部分，标题栏部分采用 ViewStub 形式加载，内容部分是一个空白的 FrameLayout。

接着来看 mDecor.onResourcesLoaded(mLayoutInflater, layoutResource) 源码：

DecorView.java

```java
void onResourcesLoaded(LayoutInflater inflater, int layoutResource) {
    // ...
    // 通过 LayoutInflater 将 screen_simple.xml 解析成 View
    final View root = inflater.inflate(layoutResource, null);
    if (mDecorCaptionView != null) {
        // ...
    } else {
        // 调用 addView 将 root 添加到 DecorView 上
        addView(root, 0, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
    }
    mContentRoot = (ViewGroup) root;
    // ...
}
```

onResourcesLoaded 主要将默认布局文件解析成 View 并且添加到 DecorView 上，解析的过程这里先不分析，放到后面的小节中，addView 则调用其父类 ViewGroup 的 addView。

此时 DecorView 应该是这样的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vEMApYVjEzdAIgq1qZBUbelICLfuWicx7RZc2jzNgTibarYpib7TMt6f90y15fhaT80iaj9VCs0c4XIEIrsks8uNRQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

内容部分目前还是空白，还没有解析添加我们在 Activity setContentView 传入的布局。

onResourcesLoaded 执行完成后，通过 findViewById 方法获取内容部分的 FrameLayout，并且将其返回。

installDecor 方法到此就分析完了，总结下：

1. 1. mDecor 也就是 DecorView 进行初始化
2. 2. 解析出默认布局添加到 DecorView 中，并且将内容部分 View 赋值给 mContentParent

#### mLayoutInflater.inflate(layoutResID, mContentParent)

分析完 installDecor 方法得知 mContentParent 就是承载 Activity 布局的 View，这里继续跟踪源码来看一下 inflate 做了哪些工作：

LayoutInflate.java

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    // 注意这里第三个参数 也就是 attachToRoot 为 ture
    // root 也就是 mContentParent，installDecor 中已经初始化
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    // ...
    // 获取布局文件的解析器
    XmlResourceParser parser = res.getLayout(resource);
    // ...
    return inflate(parser, root, attachToRoot);
}

public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    // ...
    // root 赋值给 result
    View result = root;
    
    // 将解析器推进到第一个START_TAG 也就是根View
    advanceToRootNode(parser); 
    // 拿到根 View 名字
    final String name = parser.getName();
    if (TAG_MERGE.equals(name)) { // merge 布局进 if
        // ...
    } else { // 普通布局进 else
        // 创建出根 View ## 后面小节着重分析这里是如何创建 View 的
        final View temp = createViewFromTag(root, name, inflaterContext, attrs);
        ViewGroup.LayoutParams params = null; // 布局参数
        if (root != null) { // root 不为 null
            params = root.generateLayoutParams(attrs); // 初始化布局参数
            if (!attachToRoot) { // atachToRoot 为 true 不进入
                temp.setLayoutParams(params);
            }
        }
        // ...
        
        // 解析布局中的其他 View 并且添加到 temp 根 View 中
        rInflateChildren(parser, temp, attrs, true);
        // ...
        
        // 将创建的根 View 添加到 root 也就是 mContentParent 中
        if (root != null && attachToRoot) {
            root.addView(temp, params);
        }
        // ...
    }
    // ...
    return result;
}
```

可以看出这里的调用将我们传入的 Activity 布局的根 View 解析出来并且添加到了 mContentParent 中，此时的 DecorView 是这样的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vEMApYVjEzdAIgq1qZBUbelICLfuWicx72uRPdssH0qic7w07lFc0JJNuy6LI05Dqk9GxkiaTyBrO9TA0NFesian6w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 

到这里为止我们的布局就全部创建完成了，但是要注意，onCreate 中调用 setContentView 后流程就走到了这里，此时布局文件虽然创建完成了，但是还并没有绘制到 Activity 中，View 的三大流程 measure、draw、layout 都还没有进行，本篇博客主要分析换肤功能，View 的绘制流程将在后续其他博客中会另行分析。

### View 的创建

在上述布局文件创建流程，在最后的 LayoutInflate 文件中的 inflate 方法里创建出了布局文件的所有 View。根布局的 View 直接通过 createViewFromTag 方法创建，直接点进去源码查看：

LayoutInflate.java

```java
private View createViewFromTag(View parent, String name, Context context, AttributeSet attrs) {
    return createViewFromTag(parent, name, context, attrs, false);
}

View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    // ...
    
    // 先通过 tryCreateView 尝试创建 View
    View view = tryCreateView(parent, name, context, attrs);
    
    if (view == null) { // 如果创建失败进入 if
        // ...
        try {
            if (-1 == name.indexOf('.')) { // 表示 sdk 中的 View (Text、Button...)
                view = onCreateView(context, parent, name, attrs);
            } else { // 表示自定义 View 或者 support 包中的 View (androidx.appcompat.widget.AppCompatButton...)
                view = createView(context, name, null, attrs);
            }
        }
        // ...
    }
    
    return view;
    // ...
}
```

上述代码的逻辑并不复杂，共有三个创建 View 的方法，逐个对其进行分析

#### tryCreateView(parent, name, context, attrs)

LayoutInflate.java

```java
public final View tryCreateView(@Nullable View parent, @NonNull String name,
    // ...
    View view;
    if (mFactory2 != null) { // 优先通过 mFactory2 创建 View
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {  // 第二选择 mFactory 创建 View
        view = mFactory.onCreateView(name, context, attrs);
    } else { // 都创建失败返回 null
        view = null;
    }
    // ...
    return view;
}
```

逻辑很简单，那么就有分别看看 mFactory2 和 mFactory 是什么：

LayoutInflate.java

```java
public interface Factory {
    @Nullable
    View onCreateView(@NonNull String name, @NonNull Context context,
            @NonNull AttributeSet attrs);
}

public interface Factory2 extends Factory {
    @Nullable
    View onCreateView(@Nullable View parent, @NonNull String name,
            @NonNull Context context, @NonNull AttributeSet attrs);
}
```

两个均是 LayoutInflate 的内部接口，本博客以 Activity 布局创建流程作为切入点分析，且 Activity 也实现了 Factory2 接口，

#### onCreateView(context, parent, name, attrs)

LayoutInflate.java

```java
public View onCreateView(@NonNull Context viewContext, @Nullable View parent,
        @NonNull String name, @Nullable AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(parent, name, attrs);
}

protected View onCreateView(View parent, String name, AttributeSet attrs)
        throws ClassNotFoundException {
    return onCreateView(name, attrs);
}

protected View onCreateView(String name, AttributeSet attrs)
        throws ClassNotFoundException {
    // 因为是sdk原生View 增加前缀 android.view. 用于反射创建 View
    return createView(name, "android.view.", attrs);
}

public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    Context context = (Context) mConstructorArgs[0];
    if (context == null) {
        context = mContext;
    }
    return createView(context, name, prefix, attrs);
}
```

可以看出 onCreateView 方法最后还是调用到了 createView 方法，仅仅是增加了 "android.view." 前缀传递过去。

#### createView(context, name, null, attrs)

```java
public final View createView(@NonNull Context viewContext, @NonNull String name,
        @Nullable String prefix, @Nullable AttributeSet attrs)
        throws ClassNotFoundException, InflateException {
    // ...
    // 优先从 sConstructorMap 中获取 View 的构造方法
    Constructor<? extends View> constructor = sConstructorMap.get(name);
    // ...
    Class<? extends View> clazz = null;
    // ...
    
    // sConstructorMap 中获取不到 则通过反射获取 View 的构造方法并且保存到 sConstructorMap 里
    if (constructor == null) {
        clazz = Class.forName(prefix != null ? (prefix + name) : name, false,
                mContext.getClassLoader()).asSubclass(View.class);
        // ...
        constructor = clazz.getConstructor(mConstructorSignature);
        constructor.setAccessible(true);
        sConstructorMap.put(name, constructor);
    } else {
        // ...
    }
    // ...
    
    // 通过反射获取的构造方法创建 View
    final View view = constructor.newInstance(args);
    
    // ...
    return view;
}
```

createView 方法主要是是通过反射的方式创建出 View，并且对 View 的构造方法做了缓存。

### Factory 和 Factory2

熟悉了 View 创建过程，总结下来就是优先使用 Factory2 去创建，创建失败则尝试使用 Factory，再失败直接反射创建，那么我们来找一下 Factory2 是什么时候设置的。

通常我们继承的 Activity 是 AppCompatActivity，在其 onCreate 方法中有这样一行代码：

AppCompatActivity.java

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    final AppCompatDelegate delegate = getDelegate();
    delegate.installViewFactory(); // 注意这行代码
    delegate.onCreate(savedInstanceState);
    super.onCreate(savedInstanceState);
}
```

先看一下这个 delegate 是什么：

AppCompatActivity.java

```java
public AppCompatDelegate getDelegate() {
    if (mDelegate == null) {
        mDelegate = AppCompatDelegate.create(this, this);
    }
    return mDelegate;
}
```

AppCompatDelegate.java

```java
public static AppCompatDelegate create(@NonNull Activity activity,
        @Nullable AppCompatCallback callback) {
    return new AppCompatDelegateImpl(activity, callback);
}
```

原来是 AppCompatDelegateImpl 的实例，直接查看其 installViewFactory 方法：

AppCompatDelegate.java

```java
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
        // 看到这里的 this 就说明 AppCompatDelegate 实现了 Factory2 接口
        LayoutInflaterCompat.setFactory2(layoutInflater, this);
    } else {
        if (!(layoutInflater.getFactory2() instanceof AppCompatDelegateImpl)) {
            Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                    + " so we can not install AppCompat's");
        }
    }
}
```

在 installViewFactory 方法中手动给 LayoutInflater 设置了 mFactory2，既然 AppCompatDelegate 实现了 Factory2，直接查看其 onCreateView 方法实现：

AppCompatDelegate.java

```java
@Override
public final View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
    return createView(parent, name, context, attrs);
}

public View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs) {
    if (mAppCompatViewInflater == null) {
        // ...
        mAppCompatViewInflater = new AppCompatViewInflater();
        // ...
    }
    // ...
    return mAppCompatViewInflater.createView(parent, name, context, attrs, inheritContext, IS_PRE_LOLLIPOP, true, VectorEnabledTintResources.shouldBeUsed());
}
```

最终又调用了 AppCompatViewInflater 的 createView 方法，继续跟踪源码：

AppCompatViewInflater.java

```java
final View createView(View parent, final String name, @NonNull Context context,
        @NonNull AttributeSet attrs, boolean inheritContext,
        boolean readAndroidTheme, boolean readAppTheme, boolean wrapContext) {
    // ...
    View view = null;
    switch (name) {
        case "TextView": // 根据名字 创建出 View
            view = createTextView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "ImageView":
            view = createImageView(context, attrs);
            verifyNotNull(view, name);
            break;
        case "Button":
            // 还有很多不贴了
            // ...
        default:
            // 没有列举出来的 走 createView 方法 返回的 null
            view = createView(context, name, attrs);
    }

    return view;
}

// 直接创建出了 appcompat 包下的 View 并没有使用反射
protected AppCompatTextView createTextView(Context context, AttributeSet attrs) {
    return new AppCompatTextView(context, attrs);
}

// 直接创建出了 appcompat 包下的 View 并没有使用反射
protected AppCompatImageView createImageView(Context context, AttributeSet attrs) {
    return new AppCompatImageView(context, attrs);
}

// createView 返回的 null 也就意味着 mFactory2 创建 View 失败了
protected View createView(Context context, String name, AttributeSet attrs) {
    return null;
}
```

从上述代码流程中可以看出，AppCompatActivity 默认设置了 Factory2，并且其实现创建 View 是直接通过 new 的方式，并没有使用反射，性能也比较好。





### Resources 浅析

熟悉了 View 创建流程，接着熟悉下资源文件的获取过程。

所谓插件化换肤，就是将皮肤资源单独打包下发给 App，App 从打包文件中获取皮肤资源进行设置。实现这个功能需要对 Android 的资源文件获取有一定的了解。

当我们的 App 打包后，资源文件会在打包进 resources.arsc 文件中，随便找一个apk，可以在 AS 中打开进行分析，如下图所示：

![图片](https://mmbiz.qpic.cn/mmbiz_png/vEMApYVjEzdAIgq1qZBUbelICLfuWicx7DBOvI5AKOBvm9oRqRh5XUkv865DH8JukrDo5ibhnEBgRnOS12ibKzxqA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以看出颜色、drawable、mipmap等等资源都类似于数据库形式存在，当我们调用 getDrawable(id) 时，会找出对应的资源返回。

接着就来分析下 Android Resource 文件是如何获取的。通常我们在 Activity 中获取一个 Drawable 资源代码是这样的：

```
getResources().getDrawable(R.drawable.xxx)
```

#### 源码分析

#### getResources()

先来分析 getResources 方法，跟踪源码会跳到 Context 中的 ：

Context.java

```
public abstract Resources getResources();
```

返回的是一个 Resources 对象。Context 的实现类是 ContextImpl，同样是在 ActivityThread 的 performLaunchActivity 方法中创建，通过 Activity 的 attach 传递给 Activity：

ActivityThread.java

```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    // ...
    ContextImpl appContext = createBaseContextForActivity(r);
    // ...
    activity.attach(appContext, this, getInstrumentation(), 参数省略...);
}

private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
    // ...
    ContextImpl appContext = ContextImpl.createActivityContext(
            this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
    // ...
    return appContext;
}
```

可以看出是通过 ContextImpl 的 createActivityContext 方法创建，查看其源码：

ContextImpl.java

```java
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
        Configuration overrideConfiguration) {
    // ...
    // 创建出 context
    ContextImpl context = new ContextImpl(null, mainThread, packageInfo, ContextParams.EMPTY,
            attributionTag, null, activityInfo.splitName, activityToken, null, 0, classLoader,
            null);
    // ...
    // ResourcesManager 单例
    final ResourcesManager resourcesManager = ResourcesManager.getInstance();
    // 在这里调用了 setResources 给 mResources 赋值
    context.setResources(resourcesManager.createBaseTokenResources(activityToken,
            packageInfo.getResDir(),
            splitDirs,
            packageInfo.getOverlayDirs(),
            packageInfo.getOverlayPaths(),
            packageInfo.getApplicationInfo().sharedLibraryFiles,
            displayId,
            overrideConfiguration,
            compatInfo,
            classLoader,
            packageInfo.getApplication() == null ? null
                    : packageInfo.getApplication().getResources().getLoaders()));
    // ...
    return context;
}
```

接着查看 ResourcesManager 的 createBaseTokenResources 方法是如何创建 Resources 的：

ResourcesManager.java

```java
public @Nullable Resources createBaseTokenResources(@NonNull IBinder token, 参数省略...) {
    try {
        // ...
        // 又调用到了 createResourcesForActivity 方法
        return createResourcesForActivity(token, key,
                /* initialOverrideConfig */ Configuration.EMPTY, /* overrideDisplayId */ null,
                classLoader, /* apkSupplier */ null);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
    }
}

private Resources createResourcesForActivity(@NonNull IBinder activityToken, 参数省略...) {
    synchronized (mLock) {
        // ...
        // 创建出了 ResourcesImpl 这里注意下一会会分析这个方法
        ResourcesImpl resourcesImpl = findOrCreateResourcesImplForKeyLocked(key, apkSupplier);
        // ...
        // 又调用到了 createResourcesForActivityLocked 方法
        return createResourcesForActivityLocked(activityToken, initialOverrideConfig,
                overrideDisplayId, classLoader, resourcesImpl, key.mCompatInfo);
    }
}

private Resources createResourcesForActivityLocked(@NonNull IBinder activityToken,
        @NonNull Configuration initialOverrideConfig, @Nullable Integer overrideDisplayId,
        @NonNull ClassLoader classLoader, @NonNull ResourcesImpl impl,
        @NonNull CompatibilityInfo compatInfo) {
    // ...
    // 创建出 Resources 对象
    Resources resources = compatInfo.needsCompatResources() ? new CompatResources(classLoader)
            : new Resources(classLoader);
    // 将上面创建的 ResourcesImpl 赋值
    resources.setImpl(impl);
    // ...
    return resources;
}
```

Resources 的创建过程中值得注意的是 ResourcesImpl 的创建，回到 findOrCreateResourcesImplForKeyLocked 方法中查看其创建过程：

ResourcesManager.java

```java
private @Nullable ResourcesImpl findOrCreateResourcesImplForKeyLocked(
        @NonNull ResourcesKey key, @Nullable ApkAssetsSupplier apkSupplier) {
    // 缓存获取
    ResourcesImpl impl = findResourcesImplForKeyLocked(key); 
    if (impl == null) {
        // 缓存获取不到 直接创建
        impl = createResourcesImpl(key, apkSupplier);
        if (impl != null) { // 放入缓存
            mResourceImpls.put(key, new WeakReference<>(impl));
        }
    }
    return impl;
}
```

继续查看 createResourcesImpl 方法：

ResourcesManager.java

```java
private @Nullable ResourcesImpl createResourcesImpl(@NonNull ResourcesKey key,
        @Nullable ApkAssetsSupplier apkSupplier) {
    // 先创建出了 
    final AssetManager assets = createAssetManager(key, apkSupplier);
    if (assets == null) {
        return null;
    }
    // ...
    // 将 AssetManager 对象作为参数传递给 ResourcesImpl 构造器
    final ResourcesImpl impl = new ResourcesImpl(assets, displayMetrics, config, daj);
    // ...
    return impl;
}
```

到这里先稍微总结下，Resource 的创建过程中会先创建出 AssetManager 并且传递给 ResourcesImpl，最后将 ResourcesImpl 再 set 给 Resources。实际上 Resources 和 ResourcesImpl 都是壳，真正做事情的是 AssetManager。

#### getDrawable()

熟悉了 Resources 的创建过程，就来跟踪下 getDrawable 看看其是如何查找资源的：

Resources.java

```java
public Drawable getDrawable(@DrawableRes int id, @Nullable Theme theme)
        throws NotFoundException {
    return getDrawableForDensity(id, 0, theme);
}

public Drawable getDrawableForDensity(@DrawableRes int id, int density, @Nullable Theme theme) {
    final TypedValue value = obtainTempTypedValue();
    try {
        // 在这里调用了 ResourcesImpl 的 getValueForDensity 方法
        final ResourcesImpl impl = mResourcesImpl;
        impl.getValueForDensity(id, density, value, true);
        return loadDrawable(value, id, density, theme);
    } finally {
        releaseTempTypedValue(value);
    }
}
```

继续查看 ResourcesImpl 的 getValueForDensity：

ResourcesImpl.java

```java
void getValueForDensity(@AnyRes int id, int density, TypedValue outValue,
        boolean resolveRefs) throws NotFoundException {
    // 最终交给了真正做事情的 AssetManager
    boolean found = mAssets.getResourceValue(id, density, outValue, resolveRefs);
    if (found) {
        return;
    }
    throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id));
}
```

看到这里就可以明白，如果想要加载皮肤包的资源，那就需要在 AssetManager 上做文章，本篇博客主要熟悉源码流程，思路在下一篇分享。

只需记住 Resources 中包含 ResourcesImpl，而 ResourcesImpl 中又包含着 AssetManager 即可。

AssertManager可以读取apk中的资源

```java
import android.content.res.AssetManager;
 
public class ResourceLoader {
 
    private AssetManager assetManager;
 
    public ResourceLoader(String apkPath) {
        this.assetManager = AssetManager.class.newInstance();
        assetManager.addAssetPath(apkPath); // 为 AssetManager 添加 APK 路径
    }
 
    // 加载资源
    public void loadResources() {
        // 获取 APK 中的资源文件列表
        String[] resourceNames = assetManager.list(""); // 获取根目录下的资源
        for (String resourceName : resourceNames) {
            // 在这里对每个资源进行操作，例如加载位图、字符串等
        }
    }
}
```

PackegeManager加载app包

```java
// 引入必要的包
import android.content.pm.PackageInfo;
import android.content.pm.PackageManager;
import android.content.Context;
 
public class APKLoader {
    private Context context;
 
    public APKLoader(Context context) {
        this.context = context;
    }
 
    // 加载 APK 文件
    public PackageInfo loadApk(String apkPath) {
        PackageManager pm = context.getPackageManager();
        PackageInfo pkgInfo = pm.getPackageArchiveInfo(apkPath, 0);
        return pkgInfo; // 返回 APK 文件的信息
    }
}
```

