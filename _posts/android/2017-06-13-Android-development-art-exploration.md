---
layout: post
category: Android
title:  "《Android开发艺术探索》笔记"
tags: [Android,View]
---

[TOC]

本书是一本Android进阶类书籍，采用理论、源码和实践相结合的方式来阐述高水准的Android应用开发要点。本书从三个方面来组织内容。

1. 介绍Android开发者不容易掌握的一些知识点

2. 结合Android源代码和应用层开发过程，融会贯通，介绍一些比较深入的知识点

3. 介绍一些核心技术和Android的性能优化思想


# 1 Activity的生命周期和启动模式


## 1.1 Activity的生命周期全面分析

用户正常使用情况下的生命周期 & 由于Activity被系统回收或者设备配置改变导致Activity被销毁重建情况下的生命周期。



### 1.1.1 典型情况下的生命周期分析

Activity的生命周期和启动模式

![](http://images2015.cnblogs.com/blog/776204/201511/776204-20151130161336030-1114688010.png)

1. Activity第一次启动：onCreate->onStart->onResume。

2. Activity切换到后台（ 用户打开新的Activity或者切换到桌面） ，onPause->onStop(如果新Activity采用了透明主题，则当前Activity不会回调onstop)。

3. Activity从后台到前台，重新可见，onRestart->onStart->onResume。

4. 用户退出Activity，onPause->onStop->onDestroy。

5. onStart开始到onStop之前，Activity可见。onResume到onPause之前，Activity可以接受用户交互。

6. 在新Activity启动之前，栈顶的Activity需要先onPause后，新Activity才能启动。所以不能在onPause执行耗时操作。

7. onstop中也不可以太耗时，资源回收和释放可以放在onDestroy中。



### 1.1.2 异常情况下的生命周期分析

1 系统配置变化导致Activity销毁重建



例如Activity处于竖屏状态，如果突然旋转屏幕，由于系统配置发生了改变，Activity就会被销

毁并重新创建。

在异常情况下系统会在onStop之前调用onSaveInstanceState来保存状态。Activity重新创建后，会在onStart之后调用onRestoreInstanceState来恢复之前保存的数据。

![](http://7xs99u.com1.z0.glb.clouddn.com/image/png/%E5%BC%82%E5%B8%B8%E6%83%85%E5%86%B5%E4%B8%8BActivity%E7%9A%84%E9%87%8D%E5%BB%BA%E8%BF%87%E7%A8%8B.png)

保存数据的流程： Activity被意外终止，调用onSaveIntanceState保存数据-> Activity委托Window，Window委托它上面的顶级容器一个ViewGroup（ 可能是DecorView） 。然后顶层容器在通知所有子元素来保存数据。

> 这是一种委托思想，Android中类似的还有：View绘制过程、事件分发等。



系统只在Activity异常终止的时候才会调用 onSaveInstanceState 和onRestoreInstanceState 方法。其他情况不会触发。



2 资源内存不足导致低优先级的Activity被回收

三种Activity优先级:前台- 可见非前台 -后台，从高到低。

如果一个进程没有四大组件，那么将很快被系统杀死。因此，后台工作最好放入service中。



android:configChanges="orientation" 在manifest中指定 configChanges 在系统配置变化后不重新创建Activity，也不会执行 onSaveInstanceState 和onRestoreInstanceState 方法，而是调用 onConfigurationChnaged 方法。

[附：系统配置变化项目](http://www.programgo.com/article/81032141506/)



configChanges 一般常用三个选项：

1. locale 系统语言变化

2. keyborardHidden 键盘的可访问性发生了变化，比如用户调出了键盘

3. orientation 屏幕方向变化



## 1.2 Activity的启动模式



### 1.2.1 Activity的LaunchMode

Android使用栈来管理Activity。

1. standard

每次启动都会重新创建一个实例，不管这个Activity在栈中是否已经存在。谁启动了这个Activity，那么Activity就运行在启动它的那个Activity所在的栈中。

用Application去启动Activity时会报错，原因是非Activity的Context没有任务栈。解决办法是为待启动Activity制定FLAG_ACTIVITY_NEW_TASH标志位，这样就会为它创建一个新的任务栈。

2. singleTop

如果新Activity位于任务栈的栈顶，那么此Activity不会被重新创建，同时回调 onNewIntent 方法。onCreate和onStart方法不会被执行。

3. singleTask

这是一种单实例模式。如果不存在activity所需要的任务栈，则创建一个新任务栈和新Activity实例；如果存在所需要的任务栈，不存在实例，则新创建一个Activity实例；如果存在所需要的任务栈和实例，则不创建，调用onNewIntent方法。同时使该Activity实例之上的所有Activity出栈。

[参考：taskAffinity标识Activity所需要的任务栈](https://gold.xitu.io/entry/57ac05858ac247005fec2ca1)

4. singleIntance

单实例模式。具有singleTask模式的所有特性，同时具有此模式的Activity只能独自位于一个任务栈中。



假设两个任务栈，前台任务栈为12，后台任务栈为XY。Y的启动模式是singleTask。现在请求Y，整个后台任务栈会被切换到前台。如图所示：

![](https://developer.android.com/images/fundamentals/diagram_backstack_singletask_multiactivity.png)

设置启动模式

1. manifest中 设置下的 android:launchMode 属性。

2. 启动Activity的 intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK); 。

3. 两种同时存在时，以第二种为准。第一种方式无法直接为Activity添加FLAG_ACTIVITY_CLEAR_TOP标识，第二种方式无法指定singleInstance模式。

4. 可以通过命令行 adb shell dumpsys activity 命令查看栈中的Activity信息。



### 1.2.2 Activity的Flags

这些FLAG可以设定启动模式、可以影响Activity的运行状态。

- FLAG_ACTIVITY_NEW_TASK

为Activity指定“singleTask”启动模式。

- FLAG_ACTIVITY_SINGLE_TOP

为Activity指定“singleTop"启动模式。

- FLAG_ACTIVITY_CLEAR_TOP 

具有此标记位的Activity启动时，同一个任务栈中位于它上面的Activity都要出栈，一般和FLAG_ACTIVITY_NEW_TASK配合使用。

- FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS 

如果设置，新的Activity不会在最近启动的Activity的列表(就是安卓手机里显示最近打开的Activity那个系统级的UI)中保存。等同于在xml中指定android:exludeFromRecents="true"属性。



## 1.3 IntentFilter的匹配规则

Activity调用方式

1. 显示调用 明确指定被启动对象的组件信息，包括包名和类名

2. 隐式调用 不需要明确指定组件信息，需要Intent能够匹配目标组件中的IntentFilter中所设置的过滤信息。



匹配规则

- IntentFilter中的过滤信息有action、category、data。

- 只有一个Intent同时匹配action类别、category类别、data类别才能成功启动目标Activity。

- 一个Activity可以有多个intent-filter，一个Intent只要能匹配任何一组intent-filter即可成功启动对应的Activity。



** action **

action是一个字符串,匹配是指与action的字符串完全一样,区分大小写。

一个intent-filter可以有多个aciton，只要Intent中的action能够和任何一个action相同即可成功匹配。

Intent中如果没有指定action，那么匹配失败。



** category **

category是一个字符串。

Intent可以没有category，但是如果你一旦有category，不管有几个，每个都必须与intent-filter中的其中一个category相同。

系统在 startActivity 和 startActivityForResult 的时候，会默认为Intent加上 android.intent.category.DEFAULT 这个category，所以为了我们的activity能够接收隐式调用，就必须在intent-filter中加上 android.intent.category.DEFAULT 这个category。



** data **

data的匹配规则与action一样，如果intent-filter中定义了data，那么Intent中必须要定义可匹配的data。

intent-filter中data的语法：

        <data android:scheme="string"
        android:host="string"
        android:port="string"
        android:path="string"
        android:pathPattern="string"
        android:pathPrefix="string"
        android:mimeType="string"/>

Intent中的data有两部分组成：mimeType和URI。mimeType是指媒体类型，比如

image/jpeg、audio/mpeg4-generic和video/等，可以表示图片、文本、视频等不同的媒

体格式。



URI的结构：

        <scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]

实际例子

        content://com.example.project:200/folder/subfolder/etc
		
        http://www.baidu.com:80/search/info

scheme：URI的模式，比如http、file、content等，默认值是 file 。

host：URI的主机名

port：URI的端口号

path、pathPattern和pathPrefix：这三个参数描述路径信息。

path、pathPattern可以表示完整的路径信息，其中pathPattern可以包含通配符 * ，表示0个或者多个任意字符。

pathPrefix只表示路径的前缀信息。



过滤规则的uri为空时，有默认值content和file，因此intent设置uri的scheme部分必须为content或file。

Intent指定data时，必须调用 setDataAndType 方法， setData 和 setType 会清除另一方的值。

对于service和BroadcastReceiver也是同样的匹配规则，不过对于service最好使用显式调用。



隐式调用需注意

- 当通过隐式调用启动Activity时，没找到对应的Activity系统就会抛出 android.content.ActivityNotFoundException 异常，所以需要判断是否有Activity能够匹配我们的隐式Intent。

- 采用 PackageManager 的 resloveActivity 方法或Intent 的 resloveActivity 方法

        public abstract List<ResolveInfo> queryIntentActivityies(Intent intent,int flags);

        public abstract ResolveInfo resloveActivity(Intent intent,int flags);



	以上的第二个参数使用 MATCH_DEFAULT_ONLY ，这个标志位的含义是仅仅匹配那些在

intent-filter中声明了 android.intent.category.DEFAULT 这个category的Activity。因为如果把不含这个category的Activity匹配出来了，由于不含DEFAULT这个category的Activity是无法接受隐式Intent的从而导致startActivity失败。



- 下面的action和category用来表明这是一个入口Activity，并且会出现在系统的应用列表中，二者缺一不可。

        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />



# 2 IPC机制


## 2.1 Android IPC 简介

- IPC即Inter-Process Communication，含义为进程间通信或者跨进程通信，是指两个进程之间进行数据交换的过程。

- 线程是CPU调度的最小单元，是一种有限的系统资源。进程一般指一个执行单元，在PC和移动设备上是指一个程序或者应用。进程与线程是包含与被包含的关系。一个进程可以包含多个线程。最简单的情况下一个进程只有一个线程，即主线程（ 例如Android的UI线程） 。

- 任何操作系统都需要有相应的IPC机制。如Windows上的剪贴板、管道和邮槽；Linux上命名管道、共享内容、信号量等。Android中最有特色的进程间通信方式就是binder，另外还支持socket。contentProvider是Android底层实现的进程间通信。

- 在Android中，IPC的使用场景大概有以下：

	- 有些模块由于特殊原因需要运行在单独的进程中。
	- 通过多进程来获取多份内存空间。
	- 当前应用需要向其他应用获取数据。

## 2.2 Android中的多进程模式

### 2.2.1 开启多进程模式

在Android中使用多线程只有一种方法：给四大组件在Manifest中指定 android:process 属性。这个属性的值就是进程名。这意味着不能在运行时指定一个线程所在的进程。

tips：使用 adb shell ps 或 adb shell ps|grep 包名 查看当前所存在的进程信息。

两种进程命名方式的区别

1. “：remote”

“：”的含义是指在当前的进程名前面附加上当前的包名，完整的进程名为“com.example.c2.remote"。这种进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中。

2. "com.example.c2.remote"

这是一种完整的命名方式。这种进程属于全局进程，其他应用可以通过ShareUID方式和它跑在同一个进程中。





### 2.2.2 多线程模式的运行机制

Android为每个进程都分配了一个独立的虚拟机，不同虚拟机在内存分配上有不同的地址空间，导致不同的虚拟机访问同一个类的对象会产生多份副本。例如不同进程的Activity对静态变量的修改，对其他进程不会造成任何影响。所有运行在不同进程的四大组件，只要它们之间需要通过内存在共享数据，都会共享失败。四大组件之间不可能不通过中间层来共享数据。


多进程会带来以下问题：

1. 静态成员和单例模式完全失效。

2. 线程同步锁机制完全失效。

这两点都是因为不同进程不在同一个内存空间下，锁的对象也不是同一个对象。

3. SharedPreferences的可靠性下降。

SharedPreferences底层是 通过读/写XML文件实现的，并发读/写会导致一定几率的数据丢失。

4. Application会多次创建。

由于系统创建新的进程的同时分配独立虚拟机，其实这就是启动一个应用的过程。在多进程模式中，不同进程的组件拥有独立的虚拟机、Application以及内存空间。

多进程相当于两个不同的应用采用了SharedUID的模式



实现跨进程的方式有很多：

1. Intent传递数据。

2. 共享文件和SharedPreferences。

3. 基于Binder的Messenger和AIDL。

4. Socket等



## 2.3 IPC基础概念介绍

主要介绍 Serializable 、 Parcelable 、 Binder 。Serializable和Parcelable接口可以完成对象的序列化过程，我们通过Intent和Binder传输数据时就需要Parcelabel和Serializable。还有的时候我们需要对象持久化到存储设备上或者通过网络传输到其他客户端，也需要Serializable完成对象持久化。



### 2.3.1 Serializable接口

Serializable 是Java提供的一个序列化接口（ 空接口） ，为对象提供标准的序列化和反序列化操作。只需要一个类去实现 Serializable 接口并声明一个 serialVersionUID 即可实现序列化。

	private static final long serialVersionUID = 8711368828010083044L

serialVersionUID也可以不声明。如果不手动指定 serialVersionUID 的值，反序列化时如果当前类有所改变（ 比如增删了某些成员变量） ，那么系统就会重新计算当前类的hash值并更新 serialVersionUID 。这个时候当前类的 serialVersionUID 就和序列化数据中的serialVersionUID 不一致，导致反序列化失败，程序就出现crash。

静态成员变量属于类不属于对象，不参与序列化过程，其次 transient 关键字标记的成员变量也不参与序列化过程。

通过重写writeObject和readObject方法可以改变系统默认的序列化过程。



### 2.3.2 Parcelable接口

Parcel内部包装了可序列化的数据，可以在Binder中自由传输。序列化过程中需要实现的功能有序列化、反序列化和内容描述。

序列化功能由 writeToParcel 方法完成,最终是通过 Parcel 的一系列writer方法来完成。

       @Override

        public void writeToParcel(Parcel out, int flags) {

        out.writeInt(code);

        out.writeString(name);

        }

反序列化功能由 CREATOR 来完成，其内部表明了如何创建序列化对象和数组，通过 Parcel 的一系列read方法来完成。



    public static final Creator<Book> CREATOR = new Creator<Book>() {

    @Override

    public Book createFromParcel(Parcel in) {

    return new Book(in);

    } 

	@Override

    public Book[] newArray(int size) {

    return new Book[size];

    }

    };

    protected Book(Parcel in) {

    code = in.readInt();

    name = in.readString();

    }



在Book(Parcel in)方法中，如果有一个成员变量是另一个可序列化对象，在反序列化过程中需要传递当前线程的上下文类加载器，否则会报无法找到类的错误。
      
       book = in.readParcelable(Thread.currentThread().getContextClassLoader());

内容描述功能由 describeContents 方法完成，几乎所有情况下都应该返回0，仅当当前对象中存在文件描述符时返回1。



    public int describeContents() {

    return 0;

    }

Serializable 是Java的序列化接口，使用简单但开销大，序列化和反序列化过程需要大量I/O操作。而 Parcelable 是Android中的序列化方式，适合在Android平台使用，效率高但是使用麻烦。 Parcelable 主要在内存序列化上，Parcelable 也可以将对象序列化到存储设备中或者将对象序列化后通过网络传输，但是稍显复杂，推荐使用 Serializable 。


### 2.3.3 Binder

Binder是Android中的一个类，实现了 IBinder 接口。从IPC角度说，Binder是Andoird的一种跨进程通讯方式，Binder还可以理解为一种虚拟物理设备，它的设备驱动是/dev/binder。从Android Framework角度来说，Binder是 ServiceManager 连接各种Manager( ActivityManager· 、 WindowManager )和相应 ManagerService 的桥梁。从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService时，服务端返回一个包含服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务器端提供的服务或者数据（ 包括普通服务和基于AIDL的服务）。

![](http://gityuan.com/images/binder/prepare/IPC-Binder.jpg)

> Binder通信采用C/S架构，从组件视角来说，包含Client、Server、ServiceManager以及binder驱动，其中ServiceManager用于管理系统中的各种服务。

> 图中的Client,Server,Service Manager之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与Binder驱动进行交互的，从而实现IPC通信方式。其中Binder驱动位于内核空间，Client,Server,Service Manager位于用户空间。Binder驱动和Service Manager可以看做是Android平台的基础架构，而Client和Server是Android的应用层，开发人员只需自定义实现client、Server端，借助Android的基本平台架构便可以直接进行IPC通信。

> http://gityuan.com/2015/10/31/binder-prepare/



Android中Binder主要用于 Service ，包括AIDL和Messenger。普通Service的Binder不涉及进程间通信，Messenger的底层其实是AIDL，所以下面通过AIDL分析Binder的工作机制。



** 由系统根据AIDL文件自动生成.java文件 **

1. Book.java

表示图书信息的实体类，实现了Parcelable接口。

2. Book.aidl

Book类在AIDL中的声明。

3. IBookManager.aidl

定义的管理Book实体的一个接口，包含 getBookList 和 addBook 两个方法。尽管Book类和IBookManager位于相同的包中，但是在IBookManager仍然要导入Book类。

4. IBookManager.java

系统为IBookManager.aidl生产的Binder类，在 gen 目录下。

IBookManager继承了 IInterface 接口，所有在Binder中传输的接口都需要继IInterface接口。结构如下：

	- 声明了 getBookList 和 addBook 方法，还声明了两个整型id分别标识这两个方法，用于标识在 transact 过程中客户端请求的到底是哪个方法。

	- 声明了一个内部类 Stub ，这个 Stub 就是一个Binder类，当客户端和服务端位于同一进程时，方法调用不会走跨进程的 transact 。当二者位于不同进程时，方法调用需要走 transact 过程，这个逻辑有 Stub 的内部代理类 Proxy 来完成。

    - 这个接口的核心实现就是它的内部类 Stub 和 Stub 的内部代理类 Proxy 。



** Stub和Proxy类的内部方法和定义 **

![](http://upload-images.jianshu.io/upload_images/1944615-3c92d9d160957e78.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. DESCRIPTOR

Binder的唯一标识，一般用Binder的类名表示。

2. asInterface(android.os.IBinder obj)

将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象，如果C/S位于同一进

程，此方法返回就是服务端的Stub对象本身，否则返回的就是系统封装后的Stub.proxy对

象。

3. asBinder

返回当前Binder对象。

4. onTransact

这个方法运行在服务端的Binder线程池中，由客户端发起跨进程请求时，远程请求会通过

系统底层封装后交由此方法来处理。该方法的原型是

        java public Boolean onTransact(int code,Parcelable data,Parcelable reply,int flags)

	1. 服务端通过code确定客户端请求的目标方法是什么，

	2. 接着从data取出目标方法所需的参数，然后执行目标方法。

	3. 执行完毕后向reply写入返回值（ 如果有返回值） 。

	4. 如果这个方法返回值为false，那么服务端的请求会失败，利用这个特性我们可以来做权限验证。

5. Proxy#getBookList 和Proxy#addBook

这两个方法运行在客户端，内部实现过程如下：

	1. 首先创建该方法所需要的输入型对象Parcel对象_data，输出型Parcel对象_reply和返回值对象List。

	2. 然后把该方法的参数信息写入_data（ 如果有参数）

	3. 接着调用transact方法发起RPC（ 远程过程调用） ，同时当前线程挂起

	4. 然后服务端的onTransact方法会被调用知道RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。



AIDL文件不是必须的，之所以提供AIDL文件，是为了方便系统为我们生成IBookManager.java，但我们完全可以自己写一个。



** linkToDeath和unlinkToDeath **

如果服务端进程异常终止，我们到服务端的Binder连接断裂。但是，如果我们不知道Binder连接已经断裂，那么客户端功能会受影响。通过linkTODeath我们可以给Binder设置一个死亡代理，当Binder死亡时，我们就会收到通知。

1. 声明一个 DeathRecipient 对象。 DeathRecipient 是一个接口，只有一个方法 binderDied ，当Binder死亡的时候，系统就会回调 binderDied 方法，然后我们就可以重新绑定远程服务。

        private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient(){

        @Override

        public void binderDied(){

        if(mBookManager == null){

        return;

        } 

        mBookManager.asBinder().unlinkToDeath(mDeathRecipient,0);

        mBookManager = null;

        // TODO：这里重新绑定远程Service

        }

        }

2. 在客户端绑定远程服务成功后，给binder设置死亡代理：

        mService = IBookManager.Stub.asInterface(binder);

        binder.linkToDeath(mDeathRecipient,0);

3. 另外，可以通过Binder的 isBinderAlive 判断Binder是否死亡。



## 2.4 Android中的IPC方式

主要有以下方式：

1. Intent中附加extras

2. 共享文件

3. Binder

4. ContentProvider

5. Socket



### 2.4.1 使用Bundle

四大组件中的三大组件（ Activity、Service、Receiver） 都支持在Intent中传递 Bundle 数据。

Bundle实现了Parcelable接口，因此可以方便的在不同进程间传输。当我们在一个进程中启动了另一个进程的Activity、Service、Receiver，可以再Bundle中附加我们需要传输给远程进程的消息并通过Intent发送出去。被传输的数据必须能够被序列化。



### 2.4.2 使用文件共享

我们可以序列化一个对象到文件系统中的同时从另一个进程中恢复这个对象。

1. 通过 ObjectOutputStream / ObjectInputStream 序列化一个对象到文件中，或者在另一个进程从文件中反序列这个对象。注意：反序列化得到的对象只是内容上和序列化之前的对象一样，本质是两个对象。

2. 文件并发读写会导致读出的对象可能不是最新的，并发写的话那就更严重了 。所以文件共享方式适合对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读写问题。

3. SharedPreferences 底层实现采用XML文件来存储键值对。系统对它的读/写有一定的缓存策略，即在内存中会有一份 SharedPreferences 文件的缓存，因此在多进程模式下，系统对它的读/写变得不可靠，面对高并发读/写时 SharedPreferences 有很大几率丢失数据，因此不建议在IPC中使用 SharedPreferences 。



### 2.4.3 使用Messenger

Messenger可以在不同进程间传递Message对象。是一种轻量级的IPC方案，底层实现是AIDL。它对AIDL进行了封装，使得我们可以更简便的进行IPC。

![](http://img.blog.csdn.net/20160828161207521)

具体使用时，分为服务端和客户端：

1. 服务端：创建一个Service来处理客户端请求，同时创建一个Handler并通过它来创建一个

Messenger，然后再Service的onBind中返回Messenger对象底层的Binder即可。

        private final Messenger mMessenger = new Messenger (new xxxHandler());

2. 客户端：绑定服务端的Sevice，利用服务端返回的IBinder对象来创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，消息类型是 Message 。如果需要服务端响应，则需要创建一个Handler并通过它来创建一个Messenger（ 和服务端一样） ，并通过 Message 的 replyTo 参数传递给服务端。服务端通过Message的 replyTo 参数就可以回应客户端了。



总而言之，就是客户端和服务端 拿到对方的Messenger来发送 Message 。只不过客户端通过bindService 而服务端通过 message.replyTo 来获得对方的Messenger。



Messenger中有一个 Hanlder 以串行的方式处理队列中的消息。不存在并发执行，因此我们不用考虑线程同步的问题。



### 2.4.4 使用AIDL

如果有大量的并发请求，使用Messenger就不太适合，同时如果需要跨进程调用服务端的方法，Messenger就无法做到了。这时我们可以使用AIDL。



流程如下：

1. 服务端需要创建Service来监听客户端请求，然后创建一个AIDL文件，将暴露给客户端的接口在AIDL文件中声明，最后在Service中实现这个AIDL接口即可。

2. 客户端首先绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。





** AIDL支持的数据类型： **

1. 基本数据类型、String、CharSequence

2. List：只支持ArrayList，里面的每个元素必须被AIDL支持

3. Map：只支持HashMap，里面的每个元素必须被AIDL支持

4. Parcelable

5. 所有的AIDL接口本身也可以在AIDL文件中使用



自定义的Parcelable对象和AIDL对象，不管它们与当前的AIDL文件是否位于同一个包，都必须显式import进来。



如果AIDL文件中使用了自定义的Parcelable对象，就必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。



        package com.ryg.chapter_2.aidl;

        parcelable Book;

AIDL接口中的参数除了基本类型以外都必须表明方向in/out。AIDL接口文件中只支持方法，不支持声明静态常量。建议把所有和AIDL相关的类和文件放在同一个包中，方便管理。

        

        void addBook(in Book book);

 AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接时，管理数据的集合直接采用 CopyOnWriteArrayList 来进行自动线程同步。类似的还有 ConcurrentHashMap 。



因为客户端的listener和服务端的listener不是同一个对象，所以 RecmoteCallbackList 是系统专门提供用于删除跨进程listener的接口，支持管理任意的AIDL接口，因为所有AIDL接口都继承自 IInterface 接口。



        public class RemoteCallbackList<E extends IInterface>

它内部通过一个Map接口来保存所有的AIDL回调，这个Map的key是 IBinder 类型，value是 Callback 类型。当客户端解除注册时，遍历服务端所有listener，找到和客户端listener具有相同Binder对象的服务端listenr并把它删掉。



==客户端RPC的时候线程会被挂起，由于被调用的方法运行在服务端的Binder线程池中，可能很耗时，不能在主线程中去调用服务端的方法。==



** 权限验证 **

默认情况下，我们的远程服务任何人都可以连接，我们必须加入权限验证功能，权限验证失败则无法调用服务中的方法。通常有两种验证方法：

1. 在onBind中验证，验证不通过返回null

验证方式比如permission验证，在AndroidManifest声明：

		<permission

        android:name="com.rgy.chapter_2.permisson.ACCESS_BOOK_SERVICE"

        android:protectionLevel="normal"/>

[Android自定义权限和使用权限](http://blog.csdn.net/reboot123/article/details/14451123)

		public IBinder onBind(Intent intent){

        int check = checkCallingOrSelefPermission("com.ryq.chapter_2.permission.ACCESS_BOOK_SERVICE");

        if(check == PackageManager.PERMISSION_DENIED){

        	return null;

        }

        return mBinder;

        }

这种方法也适用于Messager。

2. 在onTransact中验证，验证不通过返回false

可以permission验证，还可以采用Uid和Pid验证。



### 2.4.5 使用ContentProvider

==ContentProvider是四大组件之一，天生就是用来进程间通信。和Messenger一样，其底层实现是用Binder。==



系统预置了许多ContentProvider，比如通讯录、日程表等。要RPC访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。



创建自定义的ContentProvider，只需继承ContentProvider类并实现 onCreate 、 query 、 update 、 insert 、 getType 六个抽象方法即可。getType用来返回一个Uri请求所对应的MIME类型，剩下四个方法对应于CRUD操作。这六个方法都运行在ContentProvider进程中，除了 onCreate 由系统回调并运行在主线程里，其他五个方法都由外界调用并运行在Binder线程池中。



ContentProvider是通过Uri来区分外界要访问的数据集合，例如外界访问ContentProvider中的表，我们需要为它们定义单独的Uri和Uri_Code。根据Uri_Code，我们就知道要访问哪个表了。



==query、update、insert、delete四大方法存在多线程并发访问，因此方法内部要做好线程同步。==若采用SQLite并且只有一个SQLiteDatabase，SQLiteDatabase内部已经做了同步处理。若是多个SQLiteDatabase或是采用List作为底层数据集，就必须做线程同步。



### 2.4.6 使用Socket

Socket也称为“套接字”，分为流式套接字和用户数据报套接字两种，分别对应于TCP和UDP协议。Socket可以实现计算机网络中的两个进程间的通信，当然也可以在本地实现进程间的通信。我们以一个跨进程的聊天程序来演示。



在远程Service建立一个TCP服务，然后在主界面中连接TCP服务。服务端Service监听本地端口，客户端连接指定的端口，建立连接成功后，拿到 Socket 对象就可以向服务端发送消息或者接受服务端发送的消息。

[本例的客户端和服务端源代码](https://github.com/singwhatiwanna/android-art-res/tree/master/Chapter_2/src/com/ryg/chapter_2/socket)



除了采用TCP套接字，也可以用UDP套接字。实际上socket不仅能实现进程间的通信，还可以实现设备间的通信（只要设备之间的IP地址互相可见）。



## 2.5 Binder连接池

前面提到AIDL的流程是：首先创建一个service和AIDL接口，接着创建一个类继承自AIDL接口中的Stub类并实现Stub中的抽象方法，客户端在Service的onBind方法中拿到这个类的对象，然后绑定这个service，建立连接后就可以通过这个Stub对象进行RPC。



那么如果项目庞大，有多个业务模块都需要使用AIDL进行IPC，随着AIDL数量的增加，我们不能无限制地增加Service，我们需要把所有AIDL放在同一个Service中去管理。

![](http://upload-images.jianshu.io/upload_images/667368-b564d4bdd7af3141?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 服务端只有一个Service，把所有AIDL放在一个Service中，不同业务模块之间不能有耦合

- 服务端提供一个 queryBinder 接口，这个接口能够根据业务模块的特征来返回响应的Binder对象给客户端

- 不同的业务模块拿到所需的Binder对象就可以进行RPC了



[BinderPool源码](https://github.com/singwhatiwanna/android-art-res/tree/master/Chapter_2/src/com/ryg/chapter_2/binderpool)



## 2.6 选用合适的IPC方式

![](http://images2015.cnblogs.com/blog/757858/201604/757858-20160421103323491-1740712324.png)



# 3 View的事件体系

本章介绍View的事件分发和滑动冲突问题的解决方案。

## 3.1 view的基础知识

View的位置参数、MotionEvent和TouchSlop对象、VelocityTracker、GestureDetector和Scroller对象。

### 3.1.1什么是view

View是Android中所有控件的基类，View的本身可以是单个空间，也可以是多个控件组成的一组控件，即ViewGroup，ViewGroup继承自View，其内部可以有子View，这样就形成了View树的结构。



### 3.1.2 View的位置参数

View的位置主要由它的四个顶点来决定，即它的四个属性：top、left、right、bottom，分别表示View左上角的坐标点（ top，left） 以及右下角的坐标点（ right，bottom） 。

![](http://img2016.itdadao.com/d/file/tech/2016/11/02/it289208021201281.png)

同时，我们可以得到View的大小：

        

    width = right - left

    height = bottom - top

而这四个参数可以由以下方式获取：

    

    Left = getLeft();

    Right = getRight();

    Top = getTop();

    Bottom = getBottom();

Android3.0后，View增加了x、y、translationX和translationY这几个参数。其中x和y是View左上角的坐标，而translationX和translationY是View左上角相对于容器的偏移量。他们之间的换算关系如下：

    

    x = left + translationX;

    y = top + translationY;



top,left表示原始左上角坐标，而x,y表示变化后的左上角坐标。在View没有平移时，x=left,y=top。==View平移的过程中，top和left不会改变，改变的是x、y、translationX和translationY。==

### 3.1.3 MotionEvent和TouchSlop

** MotionEvent **

事件类型

- ACTION_DOWN 手指刚接触屏幕

- ACTION_MOVE 手指在屏幕上移动

- ACTION_UP 手指从屏幕上松开



点击事件类型

- 点击屏幕后离开松开，事件序列为DOWN->UP

- 点击屏幕滑动一会再松开，事件序列为DOWN->MOVE->...->MOVE->UP 



通过MotionEven对象我们可以得到事件发生的x和y坐标，我们可以通过getX/getY和getRawX/getRawY得到。它们的区别是：getX/getY返回的是相对于当前View左上角的x和y坐标，getRawX/getRawY返回的是相对于手机屏幕左上角的x和y坐标。



** TouchSloup **

TouchSloup是系统所能识别出的被认为是滑动的最小距离，这是一个常量，与设备有关，可通过以下方法获得：



	ViewConfiguration.get(getContext()).getScaledTouchSloup().

当我们处理滑动时，比如滑动距离小于这个值，我们就可以过滤这个事件（系统会默认过滤），从而有更好的用户体验。

### 3.1.4 VelocityTracker、GestureDetector和Scroller

** VelocityTracker **

速度追踪，用于追踪手指在滑动过程中的速度，包括水平放向速度和竖直方向速度。使用方法：

- 在View的onTouchEvent方法中追踪当前单击事件的速度

        VelocityRracker velocityTracker = VelocityTracker.obtain();

        velocityTracker.addMovement(event);

- 计算速度，获得水平速度和竖直速度

        velocityTracker.computeCurrentVelocity(1000);

        int xVelocity = (int)velocityTracker.getXVelocity();

        int yVelocity = (int)velocityTracker.getYVelocity();

注意，获取速度之前必须先计算速度，即调用computeCurrentVelocity方法，这里指的速度是指一段时间内手指滑过的像素数，1000指的是1000毫秒，得到的是1000毫秒内滑过的像素数。速度可正可负：速度 = （ 终点位置 - 起点位置） / 时间段

- 最后，当不需要使用的时候，需要调用clear()方法重置并回收内存：

		velocityTracker.clear();

		velocityTracker.recycle();



** GestureDetector **

手势检测，用于辅助检测用户的单击、滑动、长按、双击等行为。使用方法：

- 创建一个GestureDetector对象并实现OnGestureListener接口，根据需要，也可实现OnDoubleTapListener接口从而监听双击行为：

        GestureDetector mGestureDetector = new GestureDetector(this);

        //解决长按屏幕后无法拖动的现象

        mGestureDetector.setIsLongpressEnabled(false);

- 在目标View的OnTouchEvent方法中添加以下实现：

        boolean consume = mGestureDetector.onTouchEvent(event);

        return consume;

- 实现OnGestureListener和OnDoubleTapListener接口中的方法

![](http://img.blog.csdn.net/20160322205838021)

其中常用的方法有：onSingleTapUp(单击)、onFling(快速滑动)、onScroll(拖动)、onLongPress(长按)和onDoubleTap（ 双击）。建议：如果只是监听滑动相关的，可以自己在onTouchEvent中实现，如果要监听双击这种行为，那么就使用GestureDetector。



** Scroller **

弹性滑动对象，用于实现View的弹性滑动。其本身无法让View弹性滑动，需要和View的computeScroll方法配合使用才能完成这个功能。使用方法：



    Scroller scroller = new Scroller(mContext);

    //缓慢移动到指定位置

    private void smoothScrollTo(int destX,int destY){

        int scrollX = getScrollX();

        int delta = destX - scrollX;

        //1000ms内滑向destX,效果就是慢慢滑动

        mScroller.startScroll(scrollX,0,delta,0,1000);

        invalidata();

    } 

    @Override

    public void computeScroll(){

        if(mScroller.computeScrollOffset()){

        scrollTo(mScroller.getCurrX,mScroller.getCurrY());

        postInvalidate();

        }

    }





## 3.2 View的滑动

三种方式实现View滑动

### 3.2.1 使用scrollTo/scrollBy

scrollBy实际调用了scrollTo，它实现了基于当前位置的相对滑动，而scrollTo则实现了绝对滑动。



==scrollTo和scrollBy只能改变View的内容位置而不能改变View在布局中的位置。滑动偏移量mScrollX和mScrollY的正负与实际滑动方向相反，即从左向右滑动，mScrollX为负值，从上往下滑动mScrollY为负值。==



### 3.2.2 使用动画

使用动画移动View，主要是操作View的translationX和translationY属性，既可以采用传统的View动画，也可以采用属性动画，如果使用属性动画，为了能够兼容3.0以下的版本，需要采用开源动画库nineolddandroids。 如使用属性动画：(View在100ms内向右移动100像素)



		ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration(100).start();

	

### 3.2.3 改变布局属性

通过改变布局属性来移动View，即改变LayoutParams。



### 3.2.4 各种滑动方式的对比

- scrollTo/scrollBy：操作简单，适合对View内容的滑动；

- 动画：操作简单，主要适用于没有交互的View和实现复杂的动画效果；

- 改变布局参数：操作稍微复杂，适用于有交互的View。



## 3.3 弹性滑动



### 3.3.1 使用Scroller

使用Scroller实现弹性滑动的典型使用方法如下：



    Scroller scroller = new Scroller(mContext);

    //缓慢移动到指定位置

    private void smoothScrollTo(int destX,int dextY){

        int scrollX = getScrollX();

        int deltaX = destX - scrollX;

        //1000ms内滑向destX，效果就是缓慢滑动

        mScroller.startSscroll(scrollX,0,deltaX,0,1000);

        invalidate();

    } 

    @override

    public void computeScroll(){

        if(mScroller.computeScrollOffset()){

        scrollTo(mScroller.getCurrX(),mScroller.getCurrY());

        postInvalidate();

        }

    }

从上面代码可以知道，我们首先会构造一个Scroller对象，并调用他的startScroll方法，该方法并没有让view实现滑动，只是把参数保存下来，我们来看看startScroll方法的实现就知道了：



    public void startScroll(int startX,int startY,int dx,int dy,int duration){

        mMode = SCROLL_MODE;

        mFinished = false;

        mDuration = duration;

        mStartTime = AnimationUtils.currentAminationTimeMills();

        mStartX = startX;

        mStartY = startY;

        mFinalX = startX + dx;

        mFinalY = startY + dy;

        mDeltaX = dx;

        mDeltaY = dy;

        mDurationReciprocal = 1.0f / (float)mDuration;

    }

可以知道，startScroll方法的几个参数的含义，startX和startY表示滑动的起点，dx和dy表示的是滑动的距离，而duration表示的是滑动时间，注意，这里的滑动指的是View内容的滑动，在startScroll方法被调用后，马上调用invalidate方法，这是滑动的开始，invalidate方法会导致View的重绘，在View的draw方法中调用computeScroll方法，computeScroll又会去向Scroller获取当前的scrollX和scrollY；然后通过scrollTo方法实现滑动，接着又调用postInvalidate方法进行第二次重绘，一直循环，直到computeScrollOffset()方法返回值为false才结束整个滑动过程。 我们可以看看computeScrollOffset方法是如何获得当前的scrollX和scrollY的：



    public boolean computeScrollOffset(){

        ...

        int timePassed = (int)(AnimationUtils.currentAnimationTimeMills() - mStartTime);

        if(timePassed < mDuration){

            switch(mMode){

            case SCROLL_MODE:

            final float x = mInterpolator.getInterpolation(timePassed * mDuratio

            nReciprocal);

            mCurrX = mStartX + Math.round(x * mDeltaX);

            mCurrY = mStartY + Math.round(y * mDeltaY);

            break;

            ...

            }

        } 

        return true;

    }

到这里我们就基本明白了，computeScroll向Scroller获取当前的scrollX和scrollY其实是通过计算时间流逝的百分比来获得的，每一次重绘距滑动起始时间会有一个时间间距，通过这个时间间距Scroller就可以得到View当前的滑动位置，然后就可以通过scrollTo方法来完成View的滑动了。



### 3.3.2 通过动画

动画本身就是一种渐近的过程，因此通过动画来实现的滑动本身就具有弹性。实现也很简单：



    ObjectAnimator.ofFloat(targetView,"translationX",0,100).setDuration(100).start()

    ;    

    //当然，我们也可以利用动画来模仿Scroller实现View弹性滑动的过程：

    final int startX = 0;

    final int deltaX = 100;

    ValueAnimator animator = ValueAnimator.ofInt(0,1).setDuration(1000);

    animator.addUpdateListener(new AnimatorUpdateListener(){

        @override

        public void onAnimationUpdate(ValueAnimator animator){

        float fraction = animator.getAnimatedFraction();

        mButton1.scrollTo(startX + (int) (deltaX * fraction) , 0);

        }

    });

    animator.start();

上面的动画本质上是没有作用于任何对象上的，他只是在1000ms内完成了整个动画过程，利用这个特性，我们就可以在动画的每一帧到来时获取动画完成的比例，根据比例计算出View所滑动的距离。采用这种方法也可以实现其他动画效果，我们可以在onAnimationUpdate方法中加入自定义操作。



### 3.3.3 使用延时策略

延时策略的核心思想是通过发送一系列延时信息从而达到一种渐近式的效果，具体可以通过Hander和View的postDelayed方法，也可以使用线程的sleep方法。 下面以Handler为例：



    private static final int MESSAGE_SCROLL_TO = 1;

    private static final int FRAME_COUNT = 30;

    private static final int DELATED_TIME = 33;

    private int mCount = 0;

    @suppressLint("HandlerLeak")

    private Handler handler = new handler(){

        public void handleMessage(Message msg){

        switch(msg.what){

            case MESSAGE_SCROLL_TO:

            mCount ++ ;

            if (mCount <= FRAME_COUNT){

                float fraction = mCount / (float) FRAME_COUNT;

                int scrollX = (int) (fraction * 100);

                mButton1.scrollTo(scrollX,0);

                mHandelr.sendEmptyMessageDelayed(MESSAGE_SCROLL_TO , DELAYED_TIME);

                } 

            break;

            default : break;

            }

        }

    }



## 3.4 View的事件分发机制



### 3.4.1 点击事件的传递规则

点击事件是MotionEvent。首先我们先看看下面一段伪代码，通过它我们可以理解到点击事件的传递规则：



    public boolean dispatchTouchEvent (MotionEvent ev){

    boolean consume = false;

    if (onInterceptTouchEvnet(ev){

    	consume = onTouchEvent(ev);

    } else {

    	consume = child.dispatchTouchEnvet(ev);

    } 

    return consume;

    }

上面代码主要涉及到以下三个方法：

- public boolean dispatchTouchEvent(MotionEvent ev); 

这个方法用来进行事件的分发。如果事件传递给当前view，则调用此方法。返回结果表示是否消耗此事件，受onTouchEvent和下级View的dispatchTouchEvent方法影响。

- public boolean onInterceptTouchEvent(MotionEvent ev); 

这个方法用来判断是否拦截事件。在dispatchTouchEvent方法中调用。返回结果表示是否拦截。

- public boolean onTouchEvent(MotionEvent ev); 

这个方法用来处理点击事件。在dispatchTouchEvent方法中调用，返回结果表示是否消耗事件。如果不消耗，则在同一个事件序列中，当前View无法再次接收到事件。



![](http://7xijtp.com1.z0.glb.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202015-11-14%2015.54.15.png)

点击事件的传递规则：对于一个根ViewGroup，点击事件产生后，首先会传递给他，这时候就会调用他的dispatchTouchEvent方法，如果Viewgroup的onInterceptTouchEvent方法返回true表示他要拦截事件，接下来事件就会交给ViewGroup处理，调用ViewGroup的onTouchEvent方法；如果ViewGroup的onInteceptTouchEvent方法返回值为false，表示ViewGroup不拦截该事件，这时事件就传递给他的子View，接下来子View的dispatchTouchEvent方法，如此反复直到事件被最终处理。



当一个View需要处理事件时，如果它设置了OnTouchListener，那么onTouch方法会被调用，如果onTouch返回false，则当前View的onTouchEvent方法会被调用，返回true则不会被调用，同时，在onTouchEvent方法中如果设置了OnClickListener，那么他的onClick方法会被调用。==由此可见处理事件时的优先级关系： onTouchListener > onTouchEvent >onClickListener==



关于事件传递的机制，这里给出一些结论：

1. 一个事件系列以down事件开始，中间包含数量不定的move事件，最终以up事件结束。

2. 正常情况下，一个事件序列只能由一个View拦截并消耗。

3. 某个View拦截了事件后，该事件序列只能由它去处理，并且它的onInterceptTouchEvent

不会再被调用。

4. 某个View一旦开始处理事件，如果它不消耗ACTION_DOWN事件（ onTouchEvnet返回false） ，那么同一事件序列中的其他事件都不会交给他处理，并且事件将重新交由他的父元素去处理，即父元素的onTouchEvent被调用。

5. 如果View不消耗ACTION_DOWN以外的其他事件，那么这个事件将会消失，此时父元素的onTouchEvent并不会被调用，并且当前View可以持续收到后续的事件，最终消失的点击事件会传递给Activity去处理。

6. ViewGroup默认不拦截任何事件。

7. View没有onInterceptTouchEvent方法，一旦事件传递给它，它的onTouchEvent方法会被调用。

8. View的onTouchEvent默认消耗事件，除非他是不可点击的（ clickable和longClickable同时为false） 。View的longClickable属性默认false，clickable默认属性分情况（如TextView为false，button为true）。

9. View的enable属性不影响onTouchEvent的默认返回值。

10. onClick会发生的前提是当前View是可点击的，并且收到了down和up事件。

11. 事件传递过程总是由外向内的，即事件总是先传递给父元素，然后由父元素分发给子View，通过requestDisallowInterceptTouchEvent方法可以在子元素中干预父元素的分发过程，但是ACTION_DOWN事件除外。



### 3.4.2 事件分发的源码解析

略

## 3.5 滑动冲突

在界面中，只要内外两层同时可以滑动，这个时候就会产生滑动冲突。滑动冲突的解决有固定的方法。



### 3.5.1 常见的滑动冲突场景

![](http://static.oschina.net/uploads/img/201511/01205227_RmWK.jpg)

1. 外部滑动和内部滑动方向不一致；

比如viewpager和listview嵌套，但这种情况下viewpager自身已经对滑动冲突进行了处理。

2. 外部滑动方向和内部滑动方向一致；

3. 上面两种情况的嵌套。

只要解决1和2即可。



### 3.5.2 滑动冲突的处理规则

对于场景一，处理的规则是：当用户左右（ 上下） 滑动时，需要让外部的View拦截点击事件，当用户上下（ 左右） 滑动的时候，需要让内部的View拦截点击事件。根据滑动的方向判断谁来拦截事件。



对于场景二，由于滑动方向一致，这时候只能在业务上找到突破点，根据业务需求，规定什么时候让外部View拦截事件，什么时候由内部View拦截事件。



场景三的情况相对比较复杂，同样根据需求在业务上找到突破点。



### 3.5.3 滑动冲突的解决方式

** 外部拦截法 **

所谓外部拦截法是指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，否则就不拦截。下面是伪代码：



    public boolean onInterceptTouchEvent (MotionEvent event){

    boolean intercepted = false;

    int x = (int) event.getX();

    int y = (int) event.getY();

    switch (event.getAction()) {

    case MotionEvent.ACTION_DOWN:

        intercepted = false;

        break;

    case MotionEvent.ACTION_MOVE:

        if (父容器需要当前事件） {

        intercepted = true;

        } else {

        intercepted = flase;

        } 

        break;

        } 

    case MotionEvent.ACTION_UP:

        intercepted = false;

        break;

    default : 

    	break;

    } 

    mLastXIntercept = x;

    mLastYIntercept = y;

    return intercepted;

针对不同冲突，只需修改父容器需要当前事件的条件即可。其他不需修改也不能修改。

- ACTION_DOWN:必须返回false。因为如果返回true，后续事件都会被拦截，无法传递给子View。

- ACTION_MOVE：根据需要决定是否拦截

- ACTION_UP：必须返回false。如果拦截，那么子View无法接受up事件，无法完成click操作。而如果是父容器需要该事件，那么在ACTION_MOVE时已经进行了拦截，根据上一节的结论3，ACTION_UP不会经过onInterceptTouchEvent方法，直接交给父容器处理。



** 内部拦截法 ** 

内部拦截法是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要此事件就直接消耗，否则就交由父容器进行处理。这种方法与Android事件分发机制不一致，需要配合requestDisallowInterceptTouchEvent方法才能正常工作。下面是伪代码：



    public boolean dispatchTouchEvent ( MotionEvent event ) {

    int x = (int) event.getX();

    int y = (int) event.getY();

    switch (event.getAction) {

    case MotionEvent.ACTION_DOWN:

        parent.requestDisallowInterceptTouchEvent(true);

        break;

    case MotionEvent.ACTION_MOVE:

        int deltaX = x - mLastX;

        int deltaY = y - mLastY;

        if (父容器需要此类点击事件) {

        	parent.requestDisallowInterceptTouchEvent(false);

        } 

        break;

    case MotionEvent.ACTION_UP:

        break;

    default : 

    	break;

    } 

    mLastX = x;

    mLastY = y;

    return super.dispatchTouchEvent(event);

    }

==除了子元素需要做处理外，父元素也要默认拦截除了ACTION_DOWN以外的其他事件，这样当子元素调用parent.requestDisallowInterceptTouchEvent(false)方法时，父元素才能继续拦截所需的事件。==因此，父元素要做以下修改：



    public boolean onInterceptTouchEvent (MotionEvent event) {

        int action = event.getAction();

        if(action == MotionEvent.ACTION_DOWN) {

            return false;

        } else {

            return true;

        }

    }



优化滑动体验：



	mScroller.abortAnimation();

外部拦截法实例：[HorizontalScrollViewEx](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_3/src/com/ryg/chapter_3/ui/HorizontalScrollViewEx.java)



# 4 View的工作原理

主要内容

- View的工作原理

- 自定义View的实现方式

- 自定义View的底层工作原理，比如View的测量流程、布局流程、绘制流程

- View常见的回调方法，比如构造方法、onAttach.onVisibilityChanged/onDetach等



## 4.1 初识ViewRoot和DecorView

ViewRoot的实现是 ViewRootImpl 类，是连接WindowManager和DecorView的纽带，View的三大流程（ mearsure、layout、draw） 均是通过ViewRoot来完成。当Activity对象被创建完毕后，会将DecorView添加到Window中，同时创建 ViewRootImpl 对象，并将ViewRootImpl 对象和DecorView建立连接，源码如下：



	root = new ViewRootImpl(view.getContext(),display);

	root.setView(view,wparams, panelParentView);

View的绘制流程是从ViewRoot的performTraversals开始的

![](https://jakkypan.gitbooks.io/android-develop-art-discovery/content/QQ20151224-1@2x.png)

1. measure用来测量View的宽高

2. layout来确定View在父容器中的位置

3. draw负责将View绘制在屏幕上



performTraversals会依次调用 performMeasure 、 performLayout 和performDraw 三个方法，这三个方法分别完成顶级View的measure、layout和draw这三大流程。其中 performMeasure 中会调用 measure 方法，在 measure 方法中又会调用 onMeasure 方法，在 onMeasure 方法中则会对所有子元素进行measure过程，这样就完成了一次measure过程；子元素会重复父容器的measure过程，如此反复完成了整个View数的遍历。另外两个过程同理。



- Measure完成后, 可以通过getMeasuredWidth 、getMeasureHeight 方法来获取View测量后的宽/高。特殊情况下，测量的宽高不等于最终的宽高，详见后面。

- Layout过程决定了View的四个顶点的坐标和实际View的宽高，完成后可通过 getTop 、 getBotton 、 getLeft 和 getRight 拿到View的四个定点坐标。



DecorView作为顶级View，其实是一个 FrameLayout ，它包含一个竖直方向的 LinearLayout ，这个 LinearLayout 分为标题栏和内容栏两个部分。

!



    <div  align="center">

    <img src="http://images2015.cnblogs.com/blog/500720/201609/500720-20160925174505236-1295369287.png" width = "150" height = "200" alt="图片" align=center />

    </div>



在Activity通过setContextView所设置的布局文件其实就是被加载到内容栏之中的。这个内容栏的id是 R.android.id.content ，通过

``` 

ViewGroup content = findViewById(R.android.id.content);

``` 



可以得到这个contentView。View层的事件都是先经过DecorView，然后才传递到子View。





## 4.2 理解MeasureSpec

MeasureSpec决定了一个View的尺寸规格。但是父容器会影响View的MeasureSpec的创建过程。系统将View的 LayoutParams 根据父容器所施加的规则转换成对应的MeasureSpec，然后根据这个MeasureSpec来测量出View的宽高。



### 4.2.1 MeasureSpec

MeasureSpec代表一个32位int值，高2位代表SpecMode（ 测量模式） ，低30位代表SpecSize（ 在某个测量模式下的规格大小） 。

SpecMode有三种：

- UNSPECIFIED ：父容器不对View进行任何限制，要多大给多大，一般用于系统内部

- EXACTLY：父容器检测到View所需要的精确大小，这时候View的最终大小就是SpecSize所指定的值，对应LayoutParams中的 match_parent 和具体数值这两种模式

- AT_MOST ：对应View的默认大小，不同View实现不同，View的大小不能大于父容器的SpecSize，对应 LayoutParams 中的 wrap_content



### 4.2.2 MeasureSpec和LayoutParams的对应关系

对于DecorView，其MeasureSpec由窗口的尺寸和其自身的LayoutParams共同确定。而View的MeasureSpec由父容器的MeasureSpec和自身的LayoutParams共同决定。



View的measure过程由ViewGroup传递而来，参考ViewGroup的 measureChildWithMargins 方法，通过调用子元素的 getChildMeasureSpec 方法来得到子元素的MeasureSpec，再调用子元素的 measure 方法。

![](http://www.qingpingshan.com/uploads/allimg/160920/0SJ34H7-3.png)

parentSize是指父容器中目前可使用的大小。



1. 当View采用固定宽/高时（ 即设置固定的dp/px） ,不管父容器的MeasureSpec是什么，View的MeasureSpec都是EXACTLY模式，并且大小遵循我们设置的值。

2. 当View的宽/高是 match_parent 时，View的MeasureSpec都是EXACTLY模式并且其大小等于父容器的剩余空间。

3. 当View的宽/高是 wrap_content 时，View的MeasureSpec都是AT_MOST模式并且其大小不能超过父容器的剩余空间。

4. 父容器的UNSPECIFIED模式，一般用于系统内部多次Measure时，表示一种测量的状态，一般来说我们不需要关注此模式。









## 4.3 View的工作流程



### 4.3.1 measure过程

** View的measure过程 **



直接继承View的自定义控件需要重写 onMeasure 方法并设置 wrap_content （ 即specMode是 AT_MOST 模式） 时的自身大小，否则在布局中使用 wrap_content 相当于使用 match_parent 。对于非 wrap_content 的情形，我们沿用系统的测量值即可。



	  @Override

	protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);

        int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);

        int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);

        int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

          // 在 MeasureSpec.AT_MOST 模式下，给定一个默认值mWidth,mHeight。默认宽高灵活指定

          //参考TextView、ImageView的处理方式

          //其他情况下沿用系统测量规则即可

        if (widthSpecMode == MeasureSpec.AT_MOST

                && heightSpecMode == MeasureSpec.AT_MOST) {

            setMeasuredDimension(mWith, mHeight);

        } else if (widthSpecMode == MeasureSpec.AT_MOST) {

            setMeasuredDimension(mWith, heightSpecSize);

        } else if (heightSpecMode == MeasureSpec.AT_MOST) {

            setMeasuredDimension(widthSpecSize, mHeight);

        }

	}

** ViewGroup的measure过程 **



ViewGroup是一个抽象类，没有重写View的 onMeasure 方法，但是它提供了一个 measureChildren 方法。这是因为不同的ViewGroup子类有不同的布局特性，导致他们的测量细节各不相同，比如 LinearLayout 和 RelativeLayout ,因此ViewGroup没办法同一实现 onMeasure方法。



measureChildren方法的流程：

1. 取出子View的 LayoutParams

2. 通过 getChildMeasureSpec 方法来创建子元素的 MeasureSpec

3. 将 MeasureSpec 直接传递给View的measure方法来进行测量



通过LinearLayout的onMeasure方法里来分析ViewGroup的measure过程：

1. LinearLayout在布局中如果使用match_parent或者具体数值，测量过程就和View一致，即高度为specSize

2. LinearLayout在布局中如果使用wrap_content，那么它的高度就是所有子元素所占用的高度总和，但不超过它的父容器的剩余空间

3. LinearLayout的的最终高度同时也把竖直方向的padding考虑在内



View的measure过程是三大流程中最复杂的一个，measure完成以后，通过 getMeasuredWidth/Height 方法就可以正确获取到View的测量后宽/高。在某些情况下，系统可能需要多次measure才能确定最终的测量宽/高，所以在onMeasure中拿到的宽/高很可能不是准确的。



==如果我们想要在Activity启动的时候就获取一个View的宽高，怎么操作呢？==因为View的measure过程和Activity的生命周期并不是同步执行，无法保证在Activity的 onCreate、onStart、onResume 时某个View就已经测量完毕。所以有以下四种方式来获取View的宽高：

1. Activity/View#onWindowFocusChanged

onWindowFocusChanged这个方法的含义是：VieW已经初始化完毕了，宽高已经准备好了，需要注意：它会被调用多次，当Activity的窗口得到焦点和失去焦点均会被调用。

2. view.post(runnable)

通过post将一个runnable投递到消息队列的尾部，当Looper调用此runnable的时候，View也初始化好了。

3. ViewTreeObserver

使用 ViewTreeObserver 的众多回调可以完成这个功能，比如OnGlobalLayoutListener 这个接口，当View树的状态发送改变或View树内部的View的可见性发生改变时，onGlobalLayout 方法会被回调，这是获取View宽高的好时机。需要注意的是，伴随着View树状态的改变， onGlobalLayout 会被回调多次。

4. view.measure(int widthMeasureSpec,int heightMeasureSpec)

手动对view进行measure。需要根据View的layoutParams分情况处理：

	- match_parent：

无法measure出具体的宽高，因为不知道父容器的剩余空间，无法测量出View的大小

	- 具体的数值（ dp/px）:

            int widthMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);

            int heightMeasureSpec = MeasureSpec.makeMeasureSpec(100,MeasureSpec.EXACTLY);

            view.measure(widthMeasureSpec,heightMeasureSpec);

	- wrap_content：

            int widthMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);

            // View的尺寸使用30位二进制表示，最大值30个1，在AT_MOST模式下，我们用View理论上能支持的最大值去构造MeasureSpec是合理的

            int heightMeasureSpec = MeasureSpec.makeMeasureSpec((1<<30)-1,MeasureSpec.AT_MOST);

            view.measure(widthMeasureSpec,heightMeasureSpec);



### 4.3.2 layout过程

layout的作用是ViewGroup用来确定子View的位置，当ViewGroup的位置被确定后，它会在onLayout中遍历所有的子View并调用其layout方法，在 layout 方法中， onLayout 方法又会被调用。



View的 layout 方法确定本身的位置，源码流程如下： 

1. setFrame 确定View的四个顶点位置，即确定了View在父容器中的位置

2. 调用 onLayout 方法，确定所有子View的位置，和onMeasure一样，onLayout的具体实现和布局有关，因此View和ViewGroup均没有真正实现 onLayout 方法。



以LinearLayout的 onLayout 方法为例：

1. 遍历所有子View并调用 setChildFrame 方法来为子元素指定对应的位置

2. setChildFrame 方法实际上调用了子View的 layout 方法，形成了递归



==View的测量宽高和最终宽高的区别：==

在View的默认实现中，View的测量宽高和最终宽高相等，只不过测量宽高形成于measure过程，最终宽高形成于layout过程。但重写view的layout方法可以使他们不相等。

### 4.3.3 draw过程

View的绘制过程遵循如下几步：

1. 绘制背景 drawBackground(canvas)

2. 绘制自己 onDraw

3. 绘制children dispatchDraw 遍历所有子View的 draw 方法

4. 绘制装饰 onDrawScrollBars



ViewGroup会默认启用 setWillNotDraw 为ture，导致系统不会去执行 onDraw ，所以自定义ViewGroup需要通过onDraw来绘制内容时，必须显式的关闭 WILL_NOT_DRAW 这个优化标记位，即调用 setWillNotDraw(false);



## 4.4 自定义View



### 4.4.1 自定义View的分类

** 继承View 重写onDraw方法 **

通过 onDraw 方法来实现一些不规则的效果，这种效果不方便通过布局的组合方式来达到。这种方式需要自己支持 wrap_content ，并且padding也要去进行处理。



** 继承ViewGroup派生特殊的layout **

实现自定义的布局方式，需要合适地处理ViewGroup的测量、布局这两个过程，并同时处理子View的测量和布局过程。



** 继承特定的View子类（ 如TextView、Button） **

扩展某种已有的控件的功能，比较简单，不需要自己去管理 wrap_content 和padding。



** 继承特定的ViewGroup子类（ 如LinearLayout） **

比较常见，实现几种view组合一起的效果。与方法二的差别是方法二更接近底层实现。



### 4.4.2 自定义View须知

1. 直接继承View或ViewGroup的控件， 需要在onmeasure中对wrap_content做特殊处理。指定wrap_content模式下的默认宽/高。

2. 直接继承View的控件，如果不在draw方法中处理padding，那么padding属性就无法起作用。直接继承ViewGroup的控件也需要在onMeasure和onLayout中考虑padding和子元素margin的影响，不然padding和子元素的margin无效。

3. 尽量不要用在View中使用Handler，因为没必要。View内部提供了post系列的方法，完全可以替代Handler的作用。

4. View中有线程和动画，需要在View的onDetachedFromWindow中停止。当View不可见时，也需要停止线程和动画，否则可能造成内存泄漏。

5. View带有滑动嵌套情形时，需要处理好滑动冲突



### 4.4.3 自定义View实例

- 继承View重写onDraw方法：[CircleView](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_4/src/com/ryg/chapter_4/ui/CircleView.java)



>自定义属性设置方法：

1. 在values目录下创建自定义属性的XML，如attrs.xml。

		<?xml version="1.0" encoding="utf-8"?>

        <resources>

            <declare-styleable name="CircleView">

                <attr name="circle_color" format="color" />

            </declare-styleable>

        </resources>

2. 在View的构造方法中解析自定义属性的值并做相应处理，这里我们解析circle_color。

		public CircleView(Context context, AttributeSet attrs, int defStyleAttr) {

            super(context, attrs, defStyleAttr);

            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.CircleView);

            mColor = a.getColor(R.styleable.CircleView_circle_color, Color.RED);

            a.recycle();

            init();

    	}

3. 在布局文件中使用自定义属性

        <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"

        xmlns:app="http://schemas.android.com/apk/res-auto"

        android:layout_width="match_parent"

        android:layout_height="match_parent"

        android:background="#ffffff"

        android:orientation="vertical" >

        <com.ryg.chapter_4.ui.CircleView

            android:id="@+id/circleView1"

            android:layout_width="wrap_content"

            android:layout_height="100dp"

            android:layout_margin="20dp"

            android:background="#000000"

            android:padding="20dp"

            app:circle_color="@color/light_green" />

    	</LinearLayout>

[Android中的命名空间](http://blog.qiji.tech/archives/3744)



- 继承ViewGroup派生特殊的layout：[HorizontalScrollViewEx](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_4/src/com/ryg/chapter_4/ui/HorizontalScrollViewEx.java)

onMeasure方法中，首先判断是否有子元素，没有的话根据LayoutParams中的宽高做相应处理。然后判断宽高是不是wrap_content，如果宽是，那么HorizontalScrollViewEx的宽就是所有所有子元素的宽度之和。如果高是wrap_content，HorizontalScrollViewEx的高度就是第一个子元素的高度。同时要处理padding和margin。

onLayout方法中，在放置子元素时候也要考虑padding和margin。



### 4.4.4 自定义View的思想

- 掌握基本功，比如View的弹性滑动、滑动冲突、绘制原理等

- 面对新的自定义View时，对其分类并选择合适的实现思路。



# 5 理解RemoteViews

什么是远程view呢？它和远程service一样，RemoteViews可以在其他进程中显示。我们可以跨进程更新它的界面。在Android中，主要有两种场景：通知栏和桌面小部件。



本章先简单介绍通知栏和桌面小部件应用，接着分析RemoteViews内部机制，最后分析RemoteViews的意义并给出一个实例。



## 5.1 RemoteViews的应用

通知栏主要是通过NotificationManager的notify方法实现。桌面小部件是通过APPWidgetProvider来实现。APPWidgetProvider本质是一个广播。RemoteViews运行在系统的SystemServer进程。

### 5.1.1 RemoteViews在通知栏的应用

我们用到自定义通知，首先要提供一个布局文件，然后通过RemoteViews来加载，可以自定义通知的样式。更新view时，通过RemoteViews提供的一系列方法。如果给一个控件加点击事件，要使用PendingIntent。

### 5.1.2 RemoteViews在桌面小部件的应用

AppWidgetProvider是实现桌面小部件的类，本质是一个BroadcastReceiver。开发步骤如下：

1. 定义小部件界面 [代码](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/res/layout/widget.xml)

2. 定义小部件配置信息 [代码](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/res/xml/appwidget_provider_info.xml)

3. 定义小部件实现类，继承AppWidgetProvider [代码](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/MyAppWidgetProvider.java)

上面的例子实现了一个简单地桌面小部件，在小部件上显示一张图片，点击后会旋转一周。

4. 在AndroidManifest.mxl中声明小部件

		receiver android:name=".MyAppWidgetProvider" >

            <meta-data

                android:name="android.appwidget.provider"

                android:resource="@xml/appwidget_provider_info" >

            </meta-data>



            <intent-filter>

                <action android:name="com.ryg.chapter_5.action.CLICK" />

                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />

            </intent-filter>

        </receiver>

第一个action用于识别小部件的单击，第二个action作为小部件的标识必须存在。



AppWidgetProvider除了onUpdate方法，还有一系列方法。这些方法会自动被onReceive方法调用。当广播到来以后，AppWidgetProvider会自动根据广播的action通过onReceive方法分发广播。

- onEnable：该小部件第一次添加到桌面时调用，添加多次只在第一次调用

- onUpdate：小部件被添加或者每次小部件更新时调用，更新时机由updatePeriodMillis指定，每个周期小部件都会自动更新一次。

- onDeleted：每删除一次桌面小部件都会调用一次

- onDisabled：最后一个该类型的桌面小部件被删除时调用

- onReceive：内置方法，用于分发具体事件给以上方法



### 5.1.3 PendingIntent概述

PendingIntent表示一种处于待定的状态的intent。典型场景是RemoteViews添加单击事件。通过send和cancel方法来发送和取消待定intent。



PendingIntent支持三种待定意图：



|  |  |

|--------|--------|

|  static  PendingIntent    |  getActivity(Context context, int requestCode, Intent intent, int flags)获得一个PendingIntent，效果相当于Context.startActivity(Intent)      |

|  static  PendingIntent    |  getService(Context context, int requestCode, Intent intent, int flags)获得一个PendingIntent，效果相当于Context.startService(Intent)      |

|  static  PendingIntent    |  getBroadcast(Context context, int requestCode, Intent intent, int flags)获得一个PendingIntent，效果相当于Context.sendBroadcast(Intent)      |

其中requestCode多数情况下设为0即可，requestCode会影响flags的效果。



PendingIntent的匹配规则：

如果两个PendingIntent，它们内部的Intent相同且requestCode也相同，那这两个PendingIntent就是相同的。



Intent的匹配规则：

如果两个intent的ComponentName和intent-filter相同，那么这两个intent相同。Extras不参与匹配过程。



flags参数的含义

- FLAG_ONE_SHOT

当前的PendingIntent只能被使用一次，然后就会被自动cancel，如果后续还有相同的PendingIntent，它们的send方法会调用失败。对于通知栏来说，同类的通知只能使用一次，后续的通知将无法打开。

- FLAG_NO_CREATE

当前的PendingIntent不会主动创建，如果当前PendingIntent之前不存在（匹配的PendingIntent），那么获取PendingIntent失败。这个flag很少使用。

- FLAG_CANCEL_CURRENT

当前的PendingIntent如果存在（匹配的PendingIntent），那么它们都会被cancel，然后系统创建一个新的PendingIntent。对于通知栏来说，那些被cancel的消息单击后将无法打开。

- FLAG_UPDATE_CURRENT

当前PendingIntent如果已经存在（匹配的PendingIntent），那么它们都会被更新。即intent中的extras会被替换成最新的。



举例：

在`manager.notify(id,notification)`中，如果id是常量，那么多次调用notify只能弹出一个通知，后续的通知会把前面的通知完全替代。而如果每次id都不同，那么会弹出多个通知。

如果id每次都不同且PendingIntent不匹配，那么flags不会对通知之间造成干扰。

如果id不同且PendingIntent匹配：

1. 如果采用了FLAG_ONE_SHOT标记位，那么后续通知中的PendingIntent会和第一条通知完全一致，包括extras，单击任何一条通知后，剩下的通知均无法再打开，当所有的通知被清除后，会再次重复这一过程。

2. 如果采用FLAG_CANCEL_CURRENT，那么只有最新的通知可以打开。

3. 如果采用FLAG_UPDATE_CURRENT，那么之前弹出的通知中的PendingIntent会被更新，与最新一条的通知完全一致，包括extras，并且这些通知都可以打开。



## 5.2 RemoteViews的内部机制

** 构造方法 **



    public RemoteViews(String packageName,int layoutId)

第一个参数是当前应用的包名，第二个参数是待加载的布局文件。

RemoteViews并不支持所有的view类型，支持类型如下：

- Layout：FrameLayout、LinearLayout、RelativeLayout、GridLayout

- View：AnalogClock、Button、Chronometer、ImageButton、ImageView、ProgressBar、TextView、ViewFlipper,ListView,GridView、StackView、AdapterViewFlipper、ViewStub。

- RemoteViews不支持以上view的子类



访问RemoteViews的view元素，必须通过一系列set方法完成：



|方法名|作用|

|--|--|

|setTextViewText(int viewId,CharSequence text)|设置TextView的文本内容 第一个参数是TextView的id 第二个参数是设置的内容|
|setTextViewTextSize(int viewId,int units,float size)	|设置TextView的字体大小 第二个参数是字体的单位
|setTextColor(int viewId,int color)	|设置TextView字体颜色
|setImageViewResource(int viewId,int srcId)	|设置ImageView的图片
|setInt(int viewId,String methodName,int value)	|反射调用View对象的参数类型为Int的方法 比如上述的setImageViewResource的方法内部就是这个方法实现 因为srcId为int型参数
|setLong setBoolean	|类似于setInt
|setOnClickPendingIntent(int viewId,PendingIntent pendingIntent)|	添加点击事件的方法

大部分set方法是通过反射来完成的。



** RemoteViews内部机制 **

通知栏和小组件分别由NotificationManager(NM)和AppWidgetManager(AWM)管理，而NM和AWM通过Binder分别和SystemService进程中的NotificationManagerService以及AppWidgetService中加载的，而它们运行在系统的SystemService中，这就和我们进程构成了跨进程通讯。



首先RemoteViews会通过Binder传递到SystemService进程，因为RemoteViews实现了Parcelable接口，因此它可以跨进程传输，系统会根据RemoteViews的包名等信息拿到该应用的资源；然后通过LayoutInflater去加载RemoteViews中的布局文件。接着系统会对View进行一系列界面更新任务，这些任务就是之前我们通过set来提交的。set方法对View的更新并不会立即执行，会记录下来，等到RemoteViews被加载以后才会执行。



为了提高效率，系统没有直接通过Binder去支持所有的View和View操作。而是提供一个Action概念，Action同样实现Parcelable接口。系统首先将View操作封装到Action对象并将这些对象跨进程传输到SystemService进程，接着SystemService进程执行Action对象的具体操作。远程进程通过RemoteViews的apply方法来进行View的更新操作，RemoteViews的apply方法会去遍历所有的Action对象并调用他们的apply方法。这样避免了定义大量的Binder接口，也避免了大量IPC操作。

![](https://jakkypan.gitbooks.io/android-develop-art-discovery/content/QQ20160122-0@2x.png)

apply和reApply的区别在于：apply会加载布局并更新界面，而reApply则只会更新界面。RemoteViews在初始化界面时会调用apply方法，后续更新界面调用reApply方法。



关于单击事件，RemoteViews中只支持发起PendingIntent，不支持onClickListener那种模式。setOnClickPendingIntent用于给普通的View设置单击事件，不能给集合（ListView/StackView）中的View设置单击事件（开销大，系统禁止了这种方式）。如果要给ListView/StackView中的item设置单击事件，必须将setPendingIntentTemplate和setOnClickFillInIntent组合使用才可以。



## 5.3 RemoteViews的意义

当一个应用需要更新另一个应用的某个界面，我们可以选择用AIDL来实现，但如果更新比较频繁，效率会有问题，同时AIDL接口就可能变得很复杂。如果采用RemoteViews就没有这个问题，但RemoteViews仅支持一些常用的View，如果界面的View都是RemoteViews所支持的，那么就可以考虑采用RemoteViews。

demo [A](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/DemoActivity_1.java)  [、B](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_5/src/com/ryg/chapter_5/DemoActivity_2.java)



# 6 Android的Drawable

Drawable表示的是一种可以在Canvas上进行绘制的抽象概念，它的种类有很多，最常见的就是颜色和图片。优点：使用简单，比自定义View成本低很多，非图片类型的Drawable占用空间较小。本章中，首先描述Drawable的层次关系，接着介绍Drawable的分类，最后介绍自定义Drawable相关的知识。



## 6.1 Drawable简介

Drawable有很多种，都表示图像的概念，但不全是图片，通过颜色也可以构造出各式各样的图像效果。实际开发中，Drawable常被用来作为View的背景使用。Drawable一般是通过XML来定义的，Drawable是所有Drawable对象的基类。



Drawable的内部宽、高这个参数比较重要，通过getIntrinsicWidth/getIntrinsicHeight这两个方法获取。但并不是所有Drawable都有宽高；图片Drawable的内部宽/高就是图片的宽/高，但是颜色形成的Drawable并没有宽/高的概念。



## 6.2 Drawable的分类

常见的有BitmapDrawable、ShapeDrawable、LayerDrawable以及StateListDrawable等。

### 6.2.1 BitmapDrawable

表示的就是一张图片，可以直接引用原始图片即可，也可以通过XML描述它，从而设置更多效果。



	<?xml version="1.0" encoding="utf-8"?>

	<bitmap / nine-patch

    xmlns:android="http://schemas.android.com/apk/res/android"

    android:src="@[package:]drawable/drawable_resource"

    android:antialias=["true" | "false"]

    android:dither=["true" | "false"]

    android:filter=["true" | "false"]

    android:gravity=["top" | "bottom" | "left" | "right" | "center_vertical" |

                      "fill_vertical" | "center_horizontal" | "fill_horizontal" |

                      "center" | "fill" | "clip_vertical" | "clip_horizontal"]

    android:tileMode=["disabled" | "clamp" | "repeat" | "mirror"] />

属性分析

- android：src

图片资源id

- android:antialias

是否开启图片抗锯齿功能。开启后会让图片变得平滑，同时也会一定程度上降低图片的清晰度，建议开启；

- android:dither

是否开启抖动效果。当图片的像素配置和手机屏幕像素配置不一致时，开启这个选项可以让高质量的图片在低质量的屏幕上还能保持较好的显示效果，建议开启。



- android:filter

是否开启过滤效果。当图片尺寸被拉伸或压缩时，开启过滤效果可以保持较好的显示效果，建议开启；



- android:gravity

当图片小于容器的尺寸时，设置此选项可以对图片进行定位。

![](http://s2.sinaimg.cn/mw690/002GIKNbgy6H9KVGKrva1&690)



- android:tileMode

平铺模式，有四种选项["disabled" | "clamp" | "repeat" | "mirror"]。当开启平铺模式后，gravity属性会被忽略。repeat是指水平和竖直方向上的平铺效果；mirror是指在水平和竖直方向上的镜面投影效果；clamp是指图片四周的像素会扩展到周围区域，这个比较特别。



** NinePatchDrawable **

表示一张.9格式的图片，它和BitmapDrawable都表示一张图片。用XML描述的方式也和BitmapDrawable一样。在bitmap标签中也可以使用.9图。



### 6.2.2 ShapeDrawable

可以理解为通过颜色来构造的图形，可以是纯色或渐变的图形。



	<?xml version="1.0" encoding="utf-8"?>

	<shape xmlns:android="http://schemas.android.com/apk/res/android"

    android:shape="rectangle">

    <corners

        android:bottomLeftRadius="10dp"

        android:bottomRightRadius="10dp"

        android:radius="5dp"

        android:topLeftRadius="10dp"

        android:topRightRadius="10dp" />



    <gradient

        android:angle="0"

        android:centerColor="#cccccc"

        android:centerX="100"

        android:centerY="20"

        android:endColor="#abcdef"

        android:gradientRadius="100dp"

        android:startColor="#000000"

        android:type="linear"

        android:useLevel="false" />



    <solid android:color="#cccccc" />



    <stroke

        android:width="1dp"

        android:color="#cccccc"

        android:dashGap="2dp"

        android:dashWidth="50dp" />

</shape>

属性分析

- android:shape 

表示图片的形状，选项：rectangle（矩形）、oval（椭圆）、line（横线）、ring（圆环）。默认值是矩形，另外line和ring这两个选项必须通过<stroke>标签来指定宽度和颜色，否则看不到效果。

其中，ring有其特殊的5种属性



|属性|含义|

|--|--|

|android:innerRadius	|圆环的内半径，和innerRadiusRatio同时存在，以innerRadius为准

|android:thickness	|圆环的厚度，外半径减去内半径的大小，和android:thinknessRatio同时存在，以thickness为准

|android:innerRadiusRatio	|内半径占整个Drawable的宽度比例，默认值为9，如果为n，内半径=宽度/n

|android:thicknessRatio	|厚度占整个Drawable宽度的比例，默认值为3，如果为n，厚度=宽度/n

|android:useLevel	|一般都用false

- `<corners>`

表示shape的四个角的角度（圆角程度）。只适用于矩形shape。其中android:radius是同时为4个角设置相同的角度，优先级较低，会被topLeftRadius这种具体指定角度的属性所覆盖。



- `<gradient> `

与`<solid>`标签相互排斥的，其中solid表示纯色填充，而gradient表示渐变效果；gradient有如下几个属性:



	1. android:angle——渐变的角度，默认为0，其值必须是45的倍数，0表示从左往右，90表示从下到上。

	2. android:centerX 渐变的中心点的横坐标

	3. android:centerY    渐变的中心点的纵坐标；

    4. android:startColor    渐变的起始色

    5. android:centerColor    渐变的中间色

    6. android:endColor    渐变的结束色

	7. android:gradientRadius 渐变半径，仅当android:type="radial"时有效。

	8. android:type 渐变的类型，有linear(线性渐变）、radial(镜像渐变）、swepp(扫描线渐变）三种，默认是线性渐变。

- `<solid>`

表示纯色填充，通过android:color即可指定shape中填充的颜色。



- `<stroke>`

 Shape的描边，有如下属性：



	1. android:width 描边的宽度

	2. android:color 描边的颜色

	3. android:dashWidth 组成虚线的线段的宽度

	4. android:dashGap 组成虚线之间的间距。dashWidth和dashGap有任何一个为0，虚线效果都不能生效。

- `<padding>`

表示空白，但不是shape的空白，而是包含它的View的空白。

- `<size>`

 shape的大小，有两个属性：android:width和android:height,分别表示shape的宽高。通过<size>标签指定宽高后，ShapeDrawable就有固定宽/高了。但是作为view的背景来说，shape还是会被拉伸或者缩小为view的大小。



更多参考：[Android样式的开发:shape篇](http://keeganlee.me/post/android/20150830)



### 6.2.3 LayerDrawable

它表示一种层次化的Drawable集合，通过将不同的Drawable放置在不同层后达到一种叠加效果。



	<?xml version="1.0" encoding="utf-8"?>

	<layer-list xmlns:android="http://schemas.android.com/apk/res/android">

    <item

        android:id="@+id/res_haimei1"

        android:bottom="10dp"

        android:drawable="@mipmap/haimei1"

        android:left="10dp"

        android:right="10dp"

        android:top="10dp" />



    <item

        android:id="@+id/res_icon"

        android:width="30dp"

        android:height="30dp"

        android:drawable="@mipmap/ic_launcher"

        android:gravity="center" />

</layer-list>



###  6.2.4 StateListDrawable

对应`<selector>`标签。它表示Drawable集合，每个Drawable对应View的一种状态，这样系统就会根据View的状态来选择合适的Drawable。主要用于设置可点击View的背景。



	<?xml version="1.0" encoding="utf-8"?>

	<selector xmlns:android="http://schemas.android.com/apk/res/android" 

    android:constantSize="false" 

    android:dither="true" 

    android:variablePadding="false">

    <item android:drawable="@mipmap/ic_launcher" android:state_pressed="true" />

    <item android:drawable="@mipmap/haimei1" android:state_pressed="false" />

</selector>

属性分析

- android:constantSize

StateListDrawable的固有大小是否随着其状态的变化而变化，因为不同的Drawable有不同的固有大小。true表示固有大小保持不变，这时它的固有大小是内部所有Drawable的固有大小的最大值。默认值为false。

- android:dither

是否开启抖动效果，默认true

- android:variablePadding

StateListDrawable的padding是否随着状态变化而变化。true表示变化，false表示padding是内部所有Drawable的padding的最大值。默认为false。



view的常见状态



|状态|含义|

|--|--|

|android:state_pressed	|按下状态，Button按下之后没有松开

|android:state_focused	|View获取了焦点

|android:state_selected|	用户选择了View，如RadioButton

|android:state_checked	|用户选中了View，适用于CheckBox

|android:state_enable	|View处于可用状态

默认的item一般放在最后并且不添加任何状态，这样当系统在之前的item无法选择的时候，就会匹配默认的item，因为item的默认状态不附带任何状态，所以它可以适配任何状态。



###  6.2.5 LevelListDrawable

对应`<level-list>`标签。同样表示Drawable集合，集合中的每个Drawable都会有一个等级的概念，根据等级不同来切换对于的Drawable。当它作为View的背景时，可以通过Drawable的setLevel方法来设置不同的等级从而切换具体的Drawable。level的值从0-10000，默认为0。



	

    <?xml version="1.0" encoding="utf-8"?>

     <level-list xmlns:android="http://schemas.android.com/apk/res/android">

        <item android:maxLevel="0" android:drawable="@drawable/ic_playmethod_normal" />

        <item android:maxLevel="1" android:drawable="@drawable/ic_playmethod_repeat_list" />

        <item android:maxLevel="2" android:drawable="@drawable/ic_playmethod_repeat_one" />

        <item android:maxLevel="3" android:drawable="@drawable/ic_playmethod_random" />

    </level-list>



### 6.2.6 TransitionDrawable

对应`<transition>`标签。用来实现两个Drawable之间淡入淡出的效果。



    <?xml version="1.0" encoding="utf-8"?>

    <transition xmlns:android="http://schemas.android.com/apk/res/android">

        <item android:drawable="@mipmap/haimei2" />

        <item android:drawable="@mipmap/haimei3" />

    </transition>



    TransitionDrawable drawable = (TransitionDrawable) imageView.getBackground();

    drawable.startTransition(1000);

startTransition和reverseTransition方法实现淡入淡出的效果以及它的逆过程。



### 6.2.7 InsetDrawable

对应于`<inset>`标签。它可以将其他Drawable内嵌到自己当中，并可以在四周留下一定的间距。当一个View希望自己的背景比自己的实际区域小的时候，可以采用InsetDrawable来实现。通过LayerDrawable也可以实现。



    <?xml version="1.0" encoding="utf-8"?>

    <inset xmlns:android="http://schemas.android.com/apk/res/android"

        android:drawable="@mipmap/haimei1"

        android:insetBottom="10dp"

        android:insetLeft="10dp"

        android:insetRight="10dp"

        android:insetTop="10dp">

        <shape android:shape="rectangle">

            <solid android:color="#abcdef" />

        </shape>

    </inset>

其中，inset中的shape距离view边界为10dp。



### 6.2.8 ScaleDrawable

 ScaleDrawable对应于xml文件中的`<scale>`标签，可以根据自己的level将指定的drawable缩放到一定比例。

 

    <?xml version="1.0" encoding="utf-8"?>

    <scale xmlns:android="http://schemas.android.com/apk/res/android"

        android:drawable="@color/blue"

        android:level="1"

        android:scaleGravity="center"

        android:scaleHeight="20%"

        android:scaleWidth="20%" />

其中，android:scaleGravity属性相当于gravity属性。android:scaleHeight/scaleWidth 表示Drawable的缩放比例。



缩放公式： `w -= (int) (w * (10000 - level) * mState.mScaleWidth / 10000)`

可见，level越大，Drawable看起来越大；scaleHeight/scaleWidth越大，Drawable看起来越小。注意的是，level设置为0时，Drawable不可见。level不应超过10000。



### 6.2.9 ClipDrawable

ClipDrawabe对应于`<clip>`标签，他可以根据自己当前的等级(level)来裁剪一个Drawable，裁剪方向可以通过Android:clipOrientation和android:gravity两个属性共同控制。



    <?xml version="1.0" encoding="utf-8"?> 

    <clip xmlns:android="http://schemas.android.com/apk/res/android" 

    android:clipOrientation="vertical\horizontal" 

    android:drawable="@drawable/bitmapdrawable" 

    android:gravity="bottom|top|left|right|center|fill|center_vertical|center_horizontal|fill_vertical|fill_horizontal|clip_vertical|clip_horizontal" />

clipOrientation表示裁剪方向。gravity需要和clipOrientation一起才能发挥作用。如下所示：



|选项| 含义|

|--|--|

|top |将内部的Drawable放在容器的顶部，不改变大小，如果为竖直裁剪，就从底部开始裁剪

|bottom |将内部的Drawable放在容器的底部，不改变大小，如果为竖直裁剪，就从顶部开始裁剪

|left |默认值。内部Drawable放在容器左边，不改变大小，如果为水平裁剪，就从右边开始裁剪。

|right |内部Drawable放在容器右边，不改变大小，如果为水平裁剪，就从左边开始裁剪

|center_vertical |Drawable在容器中竖直居中，不改变大小，竖直裁剪的时候上下同时开始裁剪

|fill_vertical |Drawable在竖直方向填充容器，如果为竖直裁剪，仅当ClipDrawable的等级为0（level=0，完全不可见）时，才会有裁剪行为

|center_horizontal |Drawable水平居中，不改变大小，水平裁剪的时候从左右两边开始裁剪

|fill_horizontal |Drawable在水平方向填充，如果为水平裁剪，仅当ClipDrawable等级=0的时候，才能有裁剪行为。

|center Drawable|在水平和竖直方向居中，不改变大小，水平裁剪的时候从左右开始裁剪，竖直裁剪的时候从上下开始裁剪。

|fill Drawable|在竖直和水平方向填充容器，仅当level=0的时候才有裁剪行为

|clip_vertical |附加选项，竖直方向的裁剪，少使用

|clip_horizontal |附加选项，水平方向的裁剪，少使用

使用步骤：

1. 定义ClipDrawable

2. 布局文件引用

3. 代码控制level

        ImageView imageClip = (ImageView) findViewById(R.id.image_clip); 

        ClipDrawable drawable = (ClipDrawable) imageClip.getDrawable(); 

        drawable.setLevel(5000);



level=0的时候，表示完全裁剪，level=10000的时候表示完全不裁剪，level=5000的时候表示裁剪了一半。即等级越大，裁剪的区域越小。



## 6.3 自定义Drawable

在第5章中，我们分析了View的工作原理，系统会调用Drawable的draw方法绘制view的背景。所以我们可以通过重写Drawable的draw方法来自定义Drawable。但是，通常我们没必要自定义Drawable，因为自定义Drawable无法在XML中使用。只有在特殊情况下可以使用自定义Drawable。

[圆形自定义Drawable demo，半径随着view的变化而变化](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_6/src/com/ryg/chapter_6/ui/CustomDrawable.java)

- draw、setAlpha、setColorFilter和getOpacity这几个方法都是必须要实现的，其中draw是最重要的。当自定义Drawable有固有大小的时候最好重写getIntrinsicWidth和getIntrinsicHeight这两个方法，因为会影响到view的wrap_content布局。上面的例子中，没有重写，其内部大小为-1。

- 内部大小不等于Drawable的实际区域大小，Drawable实际区域的大小可以通过getBounds方法来得到，一般来说它和View的尺寸相同。







































































# 7 Android动画深入分析

Android动画分为三种：

1. View动画

2. 帧动画

3. 属性动画



本章学习内容：

- 介绍View动画和自定义View动画

- View动画一些特殊的使用场景

- 对属性动画全面性的介绍

- 使用动画的一些注意事项



## 7.1 View动画

View动画的作用对象是View，支持四种动画效果：

1. 平移

2. 缩放

3. 旋转

4. 透明



### 7.1.1 View动画的种类

上述四种变换效果对应着Animation四个子类： TranslateAnimation 、 ScaleAnimation 、 RotateAnimation 和 AlphaAnimation 。这四种动画皆可以通过XML定义，也可以通过代码来动态创建。



|名称|标签|子类|效果

|--|--|--|--|

|平移动画|`<translate>`|TranslateAnimation|移动View

|缩放动画|`<scale>`|ScaleAnimation|放大或缩小View

|旋转动画|`<rotate>`|RotateAnimation|旋转View

|透明度动画|`<alpha>`|AlphaAnimation|改变View的透明度



xml定义动画



    <?xml version="1.0" encoding="utf-8"?>

    <set xmlns:android="http://schemas.android.com/apk/res/android"

    android:duration="300"

      //动画插值器，影响动画的播放速度

    android:interpolator="@android:anim/accelerate_interpolator"

      //表示集合中的动画是否和集合共享一个插值器

    android:shareInterpolator="true" >

    //透明度动画，对应 AlphaAnimation 类，可以改变 View 的透明度

      <alpha

            android:duration="3000"

            android:fromAlpha="0.0"

            android:toAlpha="1.0" />

              //旋转动画，对应着 RotateAnimation ,它可以使 View 具有旋转的动画效果

        <rotate

            android:duration="2000"

            android:fromDegrees="0"

            android:interpolator="@android:anim/accelerate_decelerate_interpolator"

            android:pivotX="50%"

            android:pivotY="50%"

            android:startOffset="3000"

            android:toDegrees="180" />

           <!--通过设置第一个alpha动画播放3s后启动rotate动画实现组合动画，如果不设置startOffset则同时播放

        pivotX:表示旋转时候的相对轴的坐标点，即围绕哪一点进行旋转，默认情况下轴点是 View 中心

        -->

          //平移动画，对应 TranslateAnimation 类，可以使 View 完成垂直或者水平方向的移动效果。

        <translate

            android:fromXDelta="500"

            android:toXDelta="0" />

              //缩放动画，对应 ScaleAnimation 类，可以使 View 具有放大和缩小的动画效果。

        <scale

            android:duration="1000"

            android:fromXScale="0.0"

            android:fromYScale="0.0"

            android:interpolator="@android:anim/accelerate_decelerate_interpolator"

            android:pivotX="50"

            android:pivotY="50"

            android:toXScale="2"

            android:toYScale="2" />

        </set>

1. <set> 标签表示动画集合，对应AnimationSet类，可以包含一个或若干个动画，内部还可以嵌套其他动画集合。

	- android:interpolator 表示动画集合所采用的插值器，插值器影响动画速度，比如非匀速动画就需要通过插值器来控制动画的播放过程。

	- android:shareInterpolator 表示集合中的动画是否和集合共享同一个插值器，如果集合不指定插值器，那么子动画就需要单独指定所需的插值器或默认值。

2. `<translate> `、` <scale> `、 `<rotate>` 、 `<alpha>` 这几个子标签分别代表四种变换效果。

3. android:fillAfter属性表示动画结束以后， View 是否停留在结束动画的位置，如果为 false ， View 会回到动画开始的位置。这个参数在动画 XML 文件的 `</set>` 节点中设置或在程序 Java 代码中进行设置：`setFillAfter(true)`。

4. 定义完View动画的xml后，通过以下代码应用动画：

        Aniamation anim = AnimationUtils.loadAnimation(context,R.anim.animation_test);

        view.startAnimation(anim);



代码动态创建动画



    AlphaAnimation alphaAnimation = new AlphaAnimation(0,1);

    alphaAnimation.setDuration(1500);

    view.startAnimation(alphaAnimation);

通过 setAnimationListener 给 View 动画添加过程监听



    public static interface AnimationListener {

        void onAnimationStart(Animation animation);

        void onAnimationEnd(Animation animation);

        void onAnimationRepeat(Animation animation);

    }



### 7.1.2 自定义View动画

除了系统提供的四种动画外，我们可以根据需求自定义动画，自定义一个新的动画只需要继承 Animation 这个抽象类，然后重写它的 inatialize 和 applyTransformation 这两个方法，在 initialize 方法中做一些初始化工作，在 Transformation 方法中进行矩阵变换即可，很多时候才有 Camera 来简化矩阵的变换过程，其实自定义动画的主要过程就是矩阵变换的过程，矩阵变换是数学上的概念，需要掌握该方面知识方能轻松实现自定义动画，例子可以参考 Android 的 APIDemos 中的一个自定义动画 [Rotate3dAnimation](https://github.com/appium/android-apidemos/blob/master/src/io/appium/android/apis/animation/Rotate3dAnimation.java) ，这是一个可以围绕 Y 轴旋转并同时沿着 Z 轴平移从而实现类似一种 3D 效果的动画。



### 7.1.3 帧动画

帧动画是顺序播放一组预先定义好的图片，使用简单，但容易引起OOM，所以在使用帧动画时应尽量避免使用过多尺寸较大的图片。系统提供了另一个类 AnimationDrawble 来使用帧动画，使用的时候，需要通过 XML 定义一个 AnimationDrawble ，如下：



    //\res\drawable\frame_animation_list.xml

    <?xml version="1.0" encoding="utf-8"?>

    <!--

        根标签为 animation-list，其中 oneshot 代表着是否只展示一遍，设置为 false 会不停的循环播放动画

        根标签下，通过 item 标签对动画中的每一个图片进行声明

        android:duration 表示展示所用的该图片的时间长度

     -->

    <animation-list xmlns:android="http://schemas.android.com/apk/res/android"

        android:oneshot="false">

        <item

            android:drawable="@drawable/one"

            android:duration="2000"/>

        <item

            android:drawable="@drawable/two"

            android:duration="2000"/>

        <item

            android:drawable="@drawable/three"

            android:duration="2000"/>

    </animation-list



## 7.2 View动画的特殊使用场景

View 动画除了可以实现的四种基本的动画效果外，还可以在一些特殊的场景下使用，比如在 ViewGroup 中可以控制子元素的出场效果，在 Activity 中可以实现不同 Activity 之间的切换效果。



### 7.2.1 LayoutAnimation

作用于ViewGroup，为ViewGroup指定一个动画，当它的子元素出场时都会具有这种动画效

果，一般用在ListView上。



    //res/anim/layout_animation.xml

    <?xml version="1.0" encoding="utf-8"?>

    <layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"

        android:delay="0.5"

        android:animationOrder="normal"

        android:animation="@anim/zoom_in">

    </layoutAnimation>

- android:delay

 表示子元素开始动画的延时时间，取值为子元素入场动画时间 duration 的倍数，比如子元素入场动画时间周期为 300ms ，那么 0.5 表示每个子元素都需要延迟 150ms 才能播放入场动画，即第一个子元素延迟 150ms 开始播放入场动画，第二个子元素延迟 300ms 开始播放入场动画，依次类推进行。

- android:animationOrder 

表示子元素动画的开场顺序，normal(正序)、reverse(倒序)、random(随机)。

- 为 ViewGroup 指定属性

`android:layoutAnimation="@anim/layout_animation"`



通过 LayoutAnimationController 来实现



    //用于控制子 view 动画效果

    LayoutAnimationController layoutAnimationController= new LayoutAnimationController(AnimationUtils.loadAnimation(this,R.anim.zoom_in));

    layoutAnimationController.setOrder(LayoutAnimationController.ORDER_NORMAL);

    listView.setLayoutAnimation(layoutAnimationController);

    listView.startLayoutAnimation();

### 7.2.2 Activity的切换效果

我们可以自定义Activity的切换效果，主要通过overridePendingTransition(int enterAnim , int exitAnim) 方法。该方法必须要在 startActivity(intent) 和 finish() 方法之后调用才会有效。



    //启动新的Activity带动画

    Intent intent=new Intent(MainActivity.this,Main2Activity.class);

    startActivity(intent);

    overridePendingTransition(R.anim.zoom_in,R.anim.zoom_out);

    //退出Activity本身带动画

    @Override

    public void finish() {

        super.finish();

        overridePendingTransition(R.anim.zoom_in,R.anim.zoom_out);

    }

Fragment 也可以添加切换动画，通过 FragmentTransation 中的 setCustomAnimations() 方法来实现切换动画，这个动画需要的是 View 动画，不能使用属性动画，因为属性动画也是 API11 才引入的，不兼容。



## 7.3 属性动画

属性动画是 API 11 引入的新特性，属性动画可以对任何对象做动画，甚至还可以没有对象。可以在一个时间间隔内完成对象从一个属性值到另一个属性值的改变。==与View动画相比，属性动画几乎无所不能，只要对象有这个属性，它都能实现动画效果。==API11以下可以通过 nineoldandroids 库来兼容以前版本。



属性动画有ValueAnimator、ObjectAnimator和AnimatorSet等概念。其中ObjectAnimator继承自ValueAnimator，AnimatorSet是动画集合。



举例：

1.  改变一个对象 TranslationY 属性，让其沿着 Y 轴平移一段距离

    	private void translateViewByObjectAnimator(View targetView){

        //TranslationY 目标 View 要改变的属性

        //ivShow.getHeight() 要移动的距离

            ObjectAnimator objectAnimator=ObjectAnimator.ofFloat(targetView,"TranslationY",ivShow.getHeight());

            objectAnimator.start();

    	}

2. 改变一个对象的背景色属性，3秒内从0xFFFF8080到0xFF8080FF渐变，无限循环且有反转效果

   	 private void changeViewBackGroundColor(View targetView){

            ValueAnimator valueAnimator=ObjectAnimator.ofInt(targetView,"backgroundColor", Color.RED,Color.BLUE);

            valueAnimator.setDuration(3000);

            //设置估值器，该处插入颜色估值器

            valueAnimator.setEvaluator(new ArgbEvaluator());

            //无限循环

            valueAnimator.setRepeatCount(ValueAnimator.INFINITE);

            //反转模式

            valueAnimator.setRepeatMode(ValueAnimator.REVERSE);

            valueAnimator.start();

    	}

3. 动画集合，5 秒内对 View 旋转、平移、缩放和透明度进行了改变

        private void startAnimationSet(View targetView){

            AnimatorSet animatorSet=new AnimatorSet();

            animatorSet.playTogether(ObjectAnimator.ofFloat(targetView,"rotationX",0,360),

                     //旋转

                    ObjectAnimator.ofFloat(targetView,"rotationY",0,360),

                    ObjectAnimator.ofFloat(targetView,"rotation",0,-90),

                     //平移

                    ObjectAnimator.ofFloat(targetView,"translationX",0,90),

                    ObjectAnimator.ofFloat(targetView,"translationY",0,90),

                     //缩放

                    ObjectAnimator.ofFloat(targetView,"scaleX",1,1.5f),

                    ObjectAnimator.ofFloat(targetView,"scaleY",1,1.5f),

                     //透明度

                    ObjectAnimator.ofFloat(targetView,"alpha",1,0.25f,1));

                    animatorSet.setDuration(3000).start();

        }

也可以通过在xml中定义在 res/animator/ 目录下。具体如下：

        \res\animator\value_animator.xml

        <?xml version="1.0" encoding="utf-8"?><!--set 标签对应着 AnimatorSet-->

        <set xmlns:android="http://schemas.android.com/apk/res/android"

            android:ordering="together">

            <!--对应着 ObjectAnimator-->

            <objectAnimator

                android:propertyName="x"

                android:repeatCount="infinite"

                android:repeatMode="reverse"

                android:startOffset="10"

                android:valueTo="300"

                android:valueType="floatType" />

            <!--其中propertyName 属性设置为translationX ，valueType 设置为floatType 可以正常启动

               如果 valueType 设置为 intType 将报错,即属性类型必须为 floatType 类型，并且android:propertyName="translationX" 表示移动 300 ，而  android:propertyName="x"表示移动到300 ，是两个不同属性-->

            <!--startOffset 指定延迟多久开始播放动画-->

            <!--valueType 表示指定的 android:propertyName 所指定属性的类型，intType 表示指定属性是整型的，

             如果指定属性为颜色，那么不需要指定 valueType 属性，系统会自动处理

             repeatCount 表示动画循环次数，默认值为0，-1 表示无限循环-->

           

            <!--对应着 ValueAnimator-->

           <animator

            android:duration="300"

            android:valueFrom="0"

            android:valueTo="360"

            android:startOffset="10"

            android:repeatCount="infinite"

            android:repeatMode="reverse"

            android:valueType="intType"/>

        </set>

使用动画

        AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator(context , R.animator.ani

        m);

        set.setTarget(view);

        set.start();

实际开发中建议使用代码实现属性动画。很多时候一个属性的起始值是无法提前确定的。



### 7.3.2 理解差值器和估值器

1. 时间插值器（ TimeInterpolator） 的作用是根据时间流逝的百分比来计算出当前属性值改变的百分比，系统预置的有LinearInterpolator（ 线性插值器：匀速动画） ，AccelerateDecelerateInterpolator（ 加速减速插值器：动画两头慢中间快） ，DecelerateInterpolator(减速插值器：动画越来越慢） 。

2. 估值器（类型估值算法， TypeEvaluator） 的作用是根据当前属性改变的百分比来计算改变后的属性值。系统预置有IntEvaluator(针对整型属性) 、FloatEvaluator(浮点型属性) 、ArgbEvaluator(针对 Color 属性)。



![](https://raw.githubusercontent.com/yongyu0102/WeeklyBlogImages/master/phase8/aninatorTimeLine.png)

如图所示，表示一个匀速动画，采用了线性插值器和整型估值算法，在 40ms 内，View 的 X 属性实现了从 0 到 40 的变化。



属性动画要求对象的该属性有 set 和 get（可选） 方法，插值器和估值算法除了系统提供的外，我们还可以自己定义，插值器或者估值算法都是一个接口，且内部只有一个方法，我们只需要派生一个类实现该接口即可，然后就可以做出千变万化的动画效果了。具体而言是：自定义插值器需要实现 Interpolator 或者 TimeInterpolator ，自定义估值算法需要实现 TypeEvaluator 。如果要对其他类型（非int,float,color)做动画，必须要自定义类型估值算法。



### 7.3.3 属性动画的监听器

属性动画提供了监听器用于监听动画的播放过程，主要有如下两个接口：AnimatorUpdateListener 和 AnimatorListener 。



    public static interface AnimatorListener {

    void onAnimationStart(Animator animation); //动画开始

    void onAnimationEnd(Animator animation); //动画结束

    void onAnimationCancel(Animator animation); //动画取消

    void onAnimationRepeat(Animator animation); //动画重复播放

    }

为了方便开发，系统提供了AnimatorListenerAdapter类，它是AnimatorListener的适配器类，可以有选择的实现以上4个方法。



    public static interface AnimatorUpdateListener {

    void onAnimationUpdate(ValueAnimator animation);

    }

AnimatorUpdateListener会监听整个动画的过程，动画由许多帧组成的，每播放一帧，onAnimationUpdate就会调用一次。利用这个特性，我们可以做一些特殊的事情。



### 7.3.4 对任意属性做动画

属性动画原理：属性动画要求动画作用的对象提供 get 方法和 set 方法，属性动画根据外界传递该属性的初始值和最终值以动画的效果去多次调用 set 方法，每次传递给 set 方法的值都不一样，确切的来说是随着时间的推移，所传递的值越来越接近最终值。总结一下，我们对 object 对象属性 abc 做动画，如果想要动画生效，要同时满足两个条件：

1. object 必须要提供 setAbc() 方法，如果动画的时候没有传递初始值，那么还要提供 getAbc() 方法，因为系统要去取 abc 属性的初始值（如果这条不满足，程序直接crash）。

2. object 的 setAbc() 对属性 abc 所做的改变必须能够通过某种方法反应出来（即最终体现了 UI 的变化），比如会带来 UI 的改变之类（如果这条不满足，动画无效果，但是程序不会crash）。



我们给 Button 的 width 属性做动画无效果但是没有crash的原因就是 Button 内部提供了 setWidth 和 getWidth 方法，但是这个 setWidth 方法并不是改变 UI 大小的，而是用来设置最大宽度和最小宽度的。对于上面属性动画的两个条件来说，这个例子只满足了条件 1 而未满足条件 2。



针对上面问题，官方文档给出了 3 种解决方法：

1. 请给你的对象加上get和set方法，如果你有权限的话

 对于SDK或者其他第三方类库的类无法加上的

2. 用一个类来包装原始对象，间接为其提供get和set方法

        /**

         * 将 Button 沿着 X 轴方向放大

         * @param button

         */

        private void performAnimationByWrapper(View button){

            ViewWrapper viewWrapper=new ViewWrapper(button);

            ObjectAnimator.ofInt(viewWrapper,"width",800)

                    .setDuration(5000)

                    .start();

        }

         private class ViewWrapper {

                private View targetView;

                public ViewWrapper(View targetView) {

                    this.targetView = targetView;

                }

                public int getWidth() {

                    //注意调用此函数能得到 View 的宽度的前提是， View 的宽度是精准测量模式，即不可以是 wrap_content

                    //否则得不到正确的测量值

                    return targetView.getLayoutParams().width;

                }

                public void setWidth(int width) {

                  //重写设置目标 view 的布局参数，使其改变大小

                    targetView.getLayoutParams().width = width;

                  //view 大小改变需要调用重新布局

                    targetView.requestLayout();

                }

            }

3. 采用ValueAnimator，监听动画过程，自己实现属性的改变

ValueAnimator 本身不作用于任何对象，也就是说直接使用它没有任何动画效果（所以系统提供了它的子类 ObjectAnimator 供我们直接使用，作用于对象直接执行动画效果，而 ValueAnimator 只是提供改变一个值的过程，并能监听到整个值的改变过程，我们基于这个过程可以自己去实现动画效果，在这个过程中做想要达到的效果，自己去实现）。它可以对一个值做动画，然后我们可以监听其动画过程，在动画过程中修改我们对象的属性，这样我们自己就实现了对对象做了动画。

        //new 一个整型估值器，用于下面比例值计算使用（可以自己去计算，这里直接使用系统的）

        private IntEvaluator intEvaluator = new IntEvaluator();

        private void performAnimatorByValue(final View targetView, final int start, final int end) {

            ValueAnimator valueAnimator = ValueAnimator.ofInt(1, 100);

            valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {

                @Override

                public void onAnimationUpdate(ValueAnimator animation) {

                    //获取当前动画进度值

                    int currentValue = (int) animation.getAnimatedValue();

                    //获取当前进度占整个动画比例

                    int fraction = (int) animation.getAnimatedFraction();

                    //直接通过估值器根据当前比例计算当前 View 的宽度,然后设置给 View

                    targetView.getLayoutParams().width = intEvaluator.evaluate(fraction, start, end);

                    targetView.requestLayout();

                }

            });

            valueAnimator.setDuration(5000)

                    .start();

        }



### 7.3.5 属性动画的工作原理

属性动画需要运行在有Looper的线程中，系统通过反射调用被作用对象get/set方法。



## 7.4 使用动画的注意事项

1. OOM问题

使用帧动画时，当图片数量较多且图片分辨率较大的时候容易出现OOM，需注意，尽量避免使用帧动画。

2. 内存泄漏

使用无限循环的属性动画时，在Activity退出时即使停止，否则将导致Activity无法释放从而造成内存泄露。

3. 兼容性问题

动画在3.0以下的系统存在兼容性问题，特殊场景可能无法正常工作，需做好适配工作。

4. View动画的问题

 View动画是对View的影像做动画，并不是真正的改变了View的状态，因此有时候会出现动画完成后View无法隐藏（ setVisibility(View.GONE） 失效） ，这时候调用 view.clearAnimation() 清理View动画即可解决。

5. 不要使用px

使用px会导致不同设备上有不同的效果。

6. 动画元素的交互

View动画是对View的影像做动画，View的真实位置没有变动，动画完成后的新位置是无法触发点击事件的。属性动画是真实改变了View的属性，所以动画完成后的位置可以接受触摸事件。

7. 硬件加速

使用动画的过程中，使用硬件加速可以提高动画的流畅度。



# 8 理解Window和WindowMananger

Window是一个抽象类，具体实现是 PhoneWindow 。不管是 Activity 、 Dialog 、 Toast 它们的视图都是附加在Window上的，因此Window实际上是View的直接管理者。WindowManager 是外界访问Window的入口，通过WindowManager可以创建Window，而Window的具体实现位于 WindowManagerService 中，WindowManager和WindowManagerService的交互是一个IPC过程。



## 8.1 Window和WindowManager

下面代码演示了通过WindowManager添加Window的过程：



    mWindowManager = (WindowManager) getSystemService(Context.WINDOW_SERVICE);

    mFloatingButton = new Button(this);

    mFloatingButton.setText("click me");

    mLayoutParams = new WindowManager.LayoutParams(

    LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0,

    PixelFormat.TRANSPARENT);

    mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL

    | LayoutParams.FLAG_NOT_FOCUSABLE

    | LayoutParams.FLAG_SHOW_WHEN_LOCKED;

    mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;

    mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;

    mLayoutParams.x = 100;

    mLayoutParams.y = 300;

    mFloatingButton.setOnTouchListener(this);

    mWindowManager.addView(mFloatingButton, mLayoutParams);

上述代码将一个button添加到屏幕坐标为（100,300）的位置上。WindowManager的flags和type这两个属性比较重要。

Flags代表Window的属性，控制Window的显示特性

1. FLAG_NOT_FOCUSABLE 

在此模式下，Window不需要获取焦点，也不需要接收各种输入事件，这个标记同时会启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层具有焦点的Window。

2. FLAG_NOT_TOUCH_MODAL 

在此模式下，系统将当前Window区域以外的点击事件传递给底层的Window，当前Window区域内的单击事件则自己处理。一般需要开启此标记。

3. FLAG_SHOW_WHEN_LOCKED

 开启此模式Window将显示在锁屏界面上。



type参数表示Window的类型

1. 应用Window：Activity

2. 子Window： 如Dialog

3. 系统Window ：如Toast和系统状态栏

Window是分层的，每个Window对应一个z-ordered，层级大的会覆盖在层级小的上面，和HTM的z-index概念一样。在三类Window中，应用Window的层级范围是1~99，子Window的层级范围是1000~1999，系统Window的层级范围是2000~2999，这些值对应WindowManager.LayoutParams的type参数。一般系统Window选用 TYPE_SYSTEM_OVERLAY 或者 TYPE_SYSTEM_ERROR （ 同时需要权限 `<uses-permission

android:name="android.permission.SYSTEM_ALERT_WINDOW" />` ） 。



WindowManager提供的功能很简单，常用的只有三个方法：

1. 添加View

2. 更新View

3. 删除View



这个三个方法定义在 ViewManager 中，而WindowManager继承了ViewManager。



        public interface ViewManager

        {

            public void addView(View view, ViewGroup.LayoutParams params);

            public void updateViewLayout(View view, ViewGroup.LayoutParams params);

            public void removeView(View view);

        }



如何拖动window？

给view设置onTouchListener：`mFloatingButton.setOnTouchListener(this)。`在onTouch方法中更新view的位置，这个位置根据手指的位置设定。







## 8.2 Window的内部机制

Window是一个抽象的概念，每一个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系。因此Window并不是实际存在的，它是以View的形式存在的。所以WindowManager的三个方法都是针对View的，说明View才是Window存在的实体。在实际使用中无法直接访问Window，必须通过WindowManager来访问Window。



### 8.2.1 Window的添加过程

Window的添加过程需要通过WindowManager的addView()来实现, 而WindowManager是一个接口, 它的真正实现是WindowManagerImpl类。



WindowManagerImpl并没有直接实现Window的三大操作, 而是全部交给了WindowManagerGlobal来处理. WindowManagerGlobal以工厂的形式向外提供自己的实例. 而WindowManagerImpl这种工作模式就典型的桥接模式, 将所有的操作全部委托给WindowManagerGlobal来实现.



1. 检查所有参数是否合法, 如果是子Window那么还需要调整一些布局参数.

2. 创建ViewRootImpl并将View添加到列表中.

3. 通过ViewRootImpl来更新界面并完成Window的添加过程. 

这个过程是通过ViewRootImpl的setView()来完成的. View的绘制过程是由ViewRootImpl来完成的, 在内部会调用requestLayout()来完成异步刷新请求. 而scheduleTraversals()实际上是View绘制的入口. 接着会通过WindowSession完成Window的添加过程(Window的添加过程是一次IPC调用). 最终会通过WindowManagerService来实现Window的添加.



WindowManagerService内部会为每一个应用保留一个单独的Session.



### 8.2.2 Window的删除过程

Window 的删除过程和添加过程一样, 都是先通过WindowManagerImpl后, 在进一步通过WindowManagerGlobal的removeView()来实现的.



方法内首先通过findViewLocked来查找待删除的View的索引, 这个过程就是建立数组遍历, 然后调用removeViewLocked来做进一步的删除.



这里通过ViewRootImpl的die()完成来完成删除操作. die()方法只是发送了请求删除的消息后就立刻返回了, 这个时候View并没有完成删除操作, 所以最后会将其添加到mDyingViews中, mDyingViews表示待删除的View的列表.



die方法中只是做了简单的判断, 如果是异步删除那么就发送一个MSG_DIE的消息, ViewRootImpl中的Handler会处理此消息并调用doDie(); 如果是同步删除, 那么就不发送消息直接调用doDie()方法.



在doDie()方法中会调用dispatchDetachedFromWindow()方法, 真正删除View的逻辑在这个方法内部实现. 其中主要做了四件事:

1. 垃圾回收的相关工作, 比如清除数据和消息,移除回调.

2. 通过Session的remove方法删除Window: mWindowSession.remove(mWindow), 这同样是一个IPC过程, 最终会调用WMS的removeWindow()方法.

3. 调用View的dispatchDetachedFromWindow()方法, 内部会调用View的onDetachedFromWindow()以及onDetachedFromWindowInternal(). 而对于onDetachedFromWindow()就是在View从Window中移除时, 这个方法就会被调用, 可以在这个方法内部做一些资源回收的工作. 比如停止动画,停止线程

4. 调用WindowManagerGlobal#doRemoveView方法刷新数据, 包括mRoots, mParams, mDyingViews, 需要将当前Window所关联的这三类对象从列表中删除.



### 8.2.3 Window的更新过程

WindowManagerGlobal#updateViewLayout()方法做的比较简单, 它需要更新View的LayoutParams并替换掉老的LayoutParams, 接着在更新ViewRootImpl中的LayoutParams. 这一步主要是通过setLayoutParams()方法实现.



在ViewRootImpl中会通过scheduleTraversals()来对View重新布局, 包括测量,布局,重绘. 除了View本身的重绘以外, ViewRootImpl还会通过WindowSession来更新Window的视图, 这个过程最后由WMS的relayoutWindow()实现同样是一个IPC过程.



## 8.3 Window的创建过程

由之前的分析可以知道，View是Android中视图的呈现方式，但是View不能单独存在，必须附着在Window这个抽象的概念上面，因此有视图的地方就有Window。这些视图包括：Activity、Dialog、Toast、PopUpWindow、菜单等等。



### 8.3.1 Activity的Window创建过程

Activity的大体启动流程: 最终会由ActivityThread中的PerformLaunchActivity()来完成整个启动过程, 这个方法内部会通过类加载器创建Activity的实例对象, 并调用其attach()方法为其关联运行过程中所依赖的一系列上下文环境变量。



在attach()方法里, 系统会创建Activity所属的Window对象并为其设置回调接口, Window对象的创建是通过PolicyManager#makeNewWindow()方法实现. 由于Activity实现了Window的CallBack接口, 因此当Window接收到外界的状态改变的时候就会回调Activity方法. 比如说我们熟悉的onAttachedToWindow(), onDetachedFromWindow(), dispatchTouchEvent()等等。



Activity将具体实现交给了Window处理, 而Window的具体实现就是PhoneWindow, 所以只需要看PhoneWindow的相关逻辑。分为以下几步

1. 如果没有DecorView, 那么就创建它. 由installDecor()-->generateDecor()触发

2. 将View添加到DecorView的mContentParent中

3. 回调Activity的onContentChanged()通知activity视图已经发生改变



这个时候DecorView已经被创建并初始化完毕, Activity的布局文件也已经添加成功到DecorView的mContentParent中. 但是这个时候DecorView还没有被WindowManager正式添加到Window中. 虽然早在Activity的attach方法中window就已经被创建了, 但是这个时候由于DecorView并没有被WindowManager识别, 所以这个时候的Window无法提供具体功能, 因为他还无法接收外界的输入信息.



在ActivityThread#handleResumeActivity()方法中, 首先会调用Activity#onResume(), 接着会调用Activity#makeVisible(), 正是在makeVisible方法中, DecorView真正的完成了添加和显示这两个过程。

### 8.3.2 Dialog的Window创建过程

Dialog的Window的创建过程和Activity类似, 有如下几步

1. 创建Window

Dialog的创建后的实际就是PhoneWindow, 这个过程和Activity的Window创建过程一致

2. 初始化DecorView并将Dialog的视图添加到DecorView中

这个过程也类似, 都是通过Window去添加指定的布局文件.

3. 将DecorView添加到Window中并显示

在Dialog的show方法中, 会通过WindowManager将DecorView添加Window中.



普通的Dialog有一个特殊之处, 那就是必须采用Activity的Content, 如果采用Application的Content, 那么就会报错. 报的错是没有应用token所导致的, 而应用token一般只有Activity才拥有.



还有一种方法. 系统Window比较特殊, 他可以不需要token, 因此只需要指定对话框的Window为系统类型就可以正常弹出对话框.



    //JAVA 给Dialog的Window改变为系统级的Window

    dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ERROR);

    //XML 声明权限

    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>



### 8.3.3 Toast的Window创建过程

Toast和Dialog不同, 它的工作过程就稍显复杂. 首先Toast也是基于Window来实现的. 但是由于Toast具有定时取消的功能, 所以系统采用了Handler. 在Toast的内部有两类IPC过程, 第一类是Toast访问NotificationManagerService()后面简称NMS. 第二类是NotificationManagerService回调Toast里的TN接口.



Toast属于系统Window, 它内部的视图有两种方式指定, 一种是系统默认的样式, 另一种是通过setView方法来指定一个自定义View. 不管如何, 他们都对应Toast的一个View类型的内部成员mNextView. Toast内部提供了cancel和show两个方法. 分别用于显示和隐藏Toast. 他们内部是一个IPC过程.



显示和隐藏Toast都是需要通过NMS来实现的. 由于NMS运行在系统的进程中, 所以只能通过远程调用的方式来显示和隐藏Toast. 而TN这个类是一个Binder类. 在Toast和NMS进行IPC的过程中, 当NMS处理Toast的显示或隐藏请求时会跨进程回调TN的方法. 这个时候由于TN运行在Binder线程池中, 所以需要通过Handler将其切换到当前主线程. 所以由其可知, Toast无法在没有Looper的线程中弹出, 因为Handler需要使用Looper才能完成切换线程的功能.



对于非系统应用来说, 最多能同时存在对Toast封装的ToastRecord上限为50个. 这样做是为了防止DOS(Denial of Service). 如果不这样, 当通过大量循环去连续的弹出Toast, 这将会导致其他应用没有机会弹出Toast, 那么对于其他应用的Toast请求, 系统的行为就是拒绝服务, 这就是拒绝服务攻击的含义.



在ToastRecord被添加到mToastQueue()中后, NMS就会通过showNextToastLocked()方法来显示当前的Toast.



Toast的显示是由ToastRecord的callback来完成的. 这个callback实际上就是Toast中的TN对象的远程Binder. 通过callback来访问TN中的方法是需要跨进程的. 最终被调用的TN中的方法会运行在发起Toast请求的应用的Binder线程池.



Toast的隐藏也会通过ToastRecord的callback完成的.同样是一次IPC过程. 方式和Toast显示类似.



以上基本说明Toast的显示和影响过程实际上是通过Toast中的TN这个类来实现的. 他有两个方法show(), hide(). 分别对应着Toast的显示和隐藏. 由于这两个方法是被NMS以跨进程的方式调用的, 因此他们运行在Binder线程池中. 为了将执行环境切换到Toast请求所在线程中, 在他们内部使用了handler。



TN的handleShow中会将Toast的视图添加到Window中.

TN的handleHide中会将Toast的视图从Window中移除.



** 以上三节的总结 **

1. 在创建视图并显示出来时，首先是通过创建一个Window对象，然后通过WindowManager对象的 addView(View view, ViewGroup.LayoutParams params); 方法将 contentView 添加到Window中，完成添加和显示视图这两个过程。

2. 在关闭视图时，通过WindowManager来移除DecorView， mWindowManager.removeViewImmediate( view); 。

3. Toast比较特殊，具有定时取消功能，所以系统采用了Handler，内部有两类IPC过程：

	- Toast访问 NotificationManagerService

	- NotificationManagerService 回调Toast里的 TN 接口



显示和隐藏Toast都通过NotificationManagerService（ NMS） 来实现，而NMS运行在系统进程中，所以只能通过IPC来进行显示/隐藏Toast。而TN是一个Binder类，在Toast和NMS进行IPC的过程中，当NMS处理Toast的显示/隐藏请求时会跨进程回调TN中的方法，这时由于TN运行在Binder线程池中，所以需要通过Handler将其切换到当前线程（ 即发起Toast请求所在的线程） ，然后通过WindowManager的 addView/removewView 方法真正完成显示和隐藏Toast。





# 9 四大组件的工作过程

本章的意义在于加深对四大组件工作方式的认识，有助于加深对Android整体的体系结构的认识。很多情况下，只有对Android的体系结构有一定认识，在实际开发中才能写出优秀的代码。 读者对四大组件的工作过程有一个感性的认识并且能够给予上层开发一些指导意义。



## 9.1 四大组件的运行状态

Android的四大组件除了BroadcastReceiver以外，都需要在AndroidManifest文件注册，BroadcastReceiver可以通过代码注册。调用方式上，除了ContentProvider以外的三种组件都需要借助intent。



** Activity **

是一种展示型组件，用于向用户直接地展示一个界面，并且可以接收用户的输入信息从而进行交互，扮演的是一个前台界面的角色。Activity的启动由intent触发，有隐式和显式两种方式。一个Activity可以有特定的启动模式，finish方法结束Activity运行。



** Service **

是一种计算型组件，在后台执行一系列计算任务。它本身还是运行在主线程中的，所以耗时的逻辑仍需要单独的线程去完成。Activity只有一种状态：启动状态。而service有两种：启动状态和绑定状态。当service处于绑定状态时，外界可以很方便的和service进行通信，而在启动状态中是不可与外界通信的。Service可以停止, 需要灵活采用stopService和unBindService



** BroadcastReceiver **

是一种消息型组件，用于在不同的组件乃至不同的应用之间传递消

息。

- 静态注册

在清单文件中进行注册广播, 这种广播在应用安装时会被系统解析, 此种形式的广播不需要应用启动就可以接收到相应的广播.

- 动态注册

需要通过Context.registerReceiver()来实现, 并在不需要的时候通过Context.unRegisterReceiver()来解除广播. 此种形态的广播要应用启动才能注册和接收广播. 在实际开发中通过Context的一系列的send方法来发送广播, 被发送的广播会被系统发送给感兴趣的广播接收者, 发送和接收的过程的匹配是通过广播接收者的`<intent-filter>`来描述的.可以实现低耦合的观察者模式, 观察者和被观察者之间可以没有任何耦合. 但广播不适合来做耗时操作.



** ContentProvider **

是一种数据共享型组件，用于向其他组件乃至其他应用共享数据。在它内部维持着一份数据集合, 这个数据集合既可以通过数据库来实现, 也可以采用其他任何类型来实现, 例如list或者map. ContentProvider对数据集合的具体实现并没有任何要求.要注意处理好内部的insert, delete, update, query方法的线程同步, 因为这几个方法是在Binder线程池被调用.



## 9.2 Activity的工作过程

![](https://hujiaweibujidao.github.io/images/androidart_activity.png)

1. Activity的所有 startActivity 重载方法最终都会调用 startActivityForResult 。

2. 调用 mInstrumentation.execStartActivity.execStartActivity() 方法。

3. 代码中启动Activity的真正实现是由ActivityManagerNative.getDefault().startActivity()方法完成的. ActivityManagerService简称AMS. AMS继承自ActivityManagerNative(), 而ActivityManagerNative()继承自Binder并实现了IActivityManager这个Binder接口, 因此AMS也是一个Binder, 它是IActivityManager的具体实现.ActivityManagerNative.getDefault()本质是一个IActivityManager类型的Binder对象, 因此具体实现是AMS.

4. 在ActivityManagerNative中, AMS这个Binder对象采用单例模式对外提供, Singleton是一个单例封装类. 第一次调用它的get()方法时会通过create方法来初始化AMS这个Binder对象, 在后续调用中会返回这个对象. 

5. AMS的startActivity()过程

	1. checkStartActivityResult () 方法检查启动Activity的结果（ 包括检查有无在

manifest注册）

	2. Activity启动过程经过两次转移, 最后又转移到了mStackSupervisor.startActivityMayWait()这个方法, 所属类为ActivityStackSupervisor. 在startActivityMayWait()内部又调用了startActivityLocked()这里会返回结果码就是之前checkStartActivityResult()用到的。

	3. 方法最后会调用startActivityUncheckedLocked(), 然后又调用了ActivityStack#resumeTopActivityLocked(). 这个时候启动过程已经从ActivityStackSupervisor转移到了ActivityStack类中.

![](http://img.blog.csdn.net/20160428014427476?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

6. 在最后的 ActivityStackSupervisor. realStartActivityLocked() 中，调用了 app.thread.scheduleLaunchActivity() 方法。 这个app.thread是ApplicationThread 类型，继承于 IApplicationThread 是一个Binder类，内部是各种启动/停止 Service/Activity的接口。

7. 在ApplicationThread中， scheduleLaunchActivity() 用来启动Activity，里面的实现就是发送一个Activity的消息（ 封装成 从ActivityClientRecord 对象） 交给Handler处理。这个Handler有一个简洁的名字 H 。

8. 在H的 handleMessage() 方法里，通过 handleLaunchActivity() 方法完成Activity对象的创建和启动,并且ActivityThread通过handleResumeActivity()方法来调用被启动的onResume()这一生命周期方法。PerformLaunchActivity()主要完成了如下几件事：

	1. 从ActivityClientRecord对象中获取待启动的Activity组件信息

	2. 通过 Instrumentation 的 newActivity 方法使用类加载器创建Activity对象

	3. 通过 LoadedApk 的makeApplication方法尝试创建Application对象，通过类加载器实现（ 如果Application已经创建过了就不会再创建）

	4. 创建 ContextImpl 对象并通过Activity的 attach 方法完成一些重要数据的初始化（ContextImpl是一个很重要的数据结构, 它是Context的具体实现, Context中的大部分逻辑都是由ContentImpl来完成的. ContextImpl是通过Activity的attach()方法来和Activity建立关联的,除此之外, 在attach()中Activity还会完成Window的创建并建立自己和Window的关联, 这样当Window接收到外部输入事件收就可以将事件传递给Activity.）

	5. 通过 mInstrumentation.callActivityOnCreate(activity, r.state) 方法调用Activity的 onCreate 方法



## 9.3 Service的工作过程

1. 启动状态：执行后台计算

2. 绑定状态：用于其他组件与Service交互



两种状态是可以共存的



### 9.3.1 Service的启动过程

![](http://hujiaweibujidao.github.io/images/androidart_service1.png)

1. Service的启动从 ContextWrapper 的 startService 开始

2. 在ContextWrapper中，大部分操作通过一个 ContextImpl 对象mBase实现

3. 在ContextImpl中， mBase.startService() 会调用 startServiceCommon 方法，而

startServiceCommon方法又会通过 ActivityManagerNative.getDefault() （ 实际上就是AMS） 这个对象来启动一个服务。

4. AMS会通过一个 ActiveService 对象（ 辅助AMS进行Service管理的类，包括Service的启动,绑定和停止等） mServices来完成启动Service： mServices.startServiceLocked() 。

5. 在mServices.startServiceLocked()最后会调用 startServiceInnerLocked() 方法：将Service的信息包装成一个 ServiceRecord 对象，ServiceRecord一直贯穿着整个Service的启动过程。通过 bringUpServiceLocked() 方法来处理，bringUpServiceLocked()又调用了 realStartServiceLocked() 方法，这才真正地去启动一个Service了。

6. realStartServiceLocked()方法的工作如下：

	1. app.thread.scheduleCreateService() 来创建Service并调用其onCreate()生命周期方法

	2. sendServiceArgsLocked() 调用其他生命周期方法，如onStartCommand()

	3. app.thread对象是 IApplicationThread 类型，实际上就是一个Binder，具体实现是ApplicationThread继承ApplictionThreadNative

7. 具体看Application对Service的启动过程app.thread.scheduleCreateService()：通过 sendMessage(H.CREATE_SERVICE , s) ，这个过程和Activity启动过程类似，同时通过发送消息给Handler H来完成的。

8. H会接受这个CREATE_SERVICE消息并通过ActivityThread的 handleCreateService() 来完成Service的最终启动。

9. handleCreateService()完成了以下工作：

	1. 通过ClassLoader创建Service对象

	2. 创建Service内部的Context对象

	3. 创建Application，并调用其onCreate()（ 只会有一次）

	4. 通过 service.attach() 方法建立Service与context的联系（ 与Activity类似）

	5. 调用service的 onCreate() 生命周期方法，至此，Service已经启动了

	6. 将Service对象存储到ActivityThread的一个ArrayMap中



### 9.3.2 Service的绑定过程

![](http://hujiaweibujidao.github.io/images/androidart_service2.png)

和service的启动过程类似的：

1. Service的绑定是从 ContextWrapper 的 bindService 开始

2. 在ContextWrapper中，交给 ContextImpl 对象 mBase.bindService()

3. 最终会调用ContextImpl的 bindServiceCommon 方法，这个方法完成两件事：

	- 将客户端的ServiceConnection转化成 ServiceDispatcher.InnerConnection 对象。ServiceDispatcher连接ServiceConnection和InnerConnection。这个过程通过 LoadedApk 的 getServiceDispatcher 方法来实现，将客户端的ServiceConnection和ServiceDispatcher的映射关系存在一个ArrayMap中。

	- 通过AMS来完成Service的具体绑定过程 ActivityManagerNative.getDefault().bindService()

4. AMS中，bindService()方法再调用 bindServiceLocked() ，bindServiceLocked()再调用 bringUpServiceLocked() ，bringUpServiceLocked()又会调用 realStartServiceLocked() 。

5. AMS的realStartServiceLocked()会调用 ActiveServices 的requrestServiceBindingLocked() 方法，最终是调用了ServiceRecord对象r的 app.thread.scheduleBindService() 方法。

6. ApplicationThread的一系列以schedule开头的方法，内部都通过Handler H来中转：scheduleBindService()内部也是通过 sendMessage(H.BIND_SERVICE , s)

7. 在H内部接收到BIND_SERVICE这类消息时就交给 ActivityThread 的handleBindService() 方法处理：

	1. 根据Servcie的token取出Service对象

	2. 调用Service的 onBind() 方法，至此，Service就处于绑定状态了。

	3. 这时客户端还不知道已经成功连接Service，需要调用客户端的binder对象来调用客户端的ServiceConnection中的 onServiceConnected() 方法，这个通过 ActivityManagerNative.getDefault().publishService() 进行。ActivityManagerNative.getDefault()就是AMS。

8. AMS的publishService()交给ActivityService对象 mServices 的 publishServiceLocked() 来处理，核心代码就一句话 c.conn.connected(r.name,service) 。对象c的类型是 ConnectionRecord ，c.conn就是ServiceDispatcher.InnerConnection对象，service就是Service的onBind方法返回的Binder对象。

9. c.conn.connected(r.name,service)内部实现是交给了mActivityThread.post(new RunnConnection(name ,service,0)); 实现。ServiceDispatcher的mActivityThread是一个Handler，其实就是ActivityThread中的H。这样一来RunConnection就经由H的post方法从而运行在主线程中，因此客户端ServiceConnection中的方法是在主线程中被回调的。

10. RunConnection的定义如下：

	- 继承Runnable接口， run() 方法的实现也是简单调用了ServiceDispatcher的 doConnected 方法。

	- 由于ServiceDispatcher内部保存了客户端的ServiceConntion对象，可以很方便地调用ServiceConntion对象的 onServiceConnected 方法。

	- 客户端的onServiceConnected方法执行后，Service的绑定过程也就完成了。

	- 根据步骤8、9、10service绑定后通过ServiceDispatcher通知客户端的过程可以说明ServiceDispatcher起着连接ServiceConnection和InnerConnection的作用。 至于Service的停止和解除绑定的过程，系统流程都是类似的。





## 9.4 BroadcastReceiver的工作过程

简单回顾一下广播的使用方法, 首先定义广播接收者, 只需要继承BroadcastReceiver并重写onReceive()方法即可. 定义好了广播接收者, 还需要注册广播接收者, 分为两种静态注册或者动态注册. 注册完成之后就可以发送广播了.

### 9.4.1 广播的注册过程

![](http://hujiaweibujidao.github.io/images/androidart_broadcastreceiver1.png)

1. 动态注册的过程是从ContextWrapper#registerReceiver()开始的. 和Activity或者Service一样. ContextWrapper并没有做实际的工作, 而是将注册的过程直接交给了ContextImpl来完成.

2. ContextImpl#registerReceiver()方法调用了本类的registerReceiverInternal()方法.

3. 系统首先从mPackageInfo获取到IIntentReceiver对象, 然后再采用跨进程的方式向AMS发送广播注册的请求. 之所以采用IIntentReceiver而不是直接采用BroadcastReceiver, 这是因为上述注册过程中是一个进程间通信的过程. 而BroadcastReceiver作为Android中的一个组件是不能直接跨进程传递的. 所有需要通过IIntentReceiver来中转一下.

4. IIntentReceiver作为一个Binder接口, 它的具体实现是LoadedApk.ReceiverDispatcher.InnerReceiver, ReceiverDispatcher的内部同时保存了BroadcastReceiver和InnerReceiver, 这样当接收到广播的时候, ReceiverDispatcher可以很方便的调用BroadcastReceiver#onReceive()方法. 这里和Service很像有同样的类, 并且内部类中同样也是一个Binder接口.

5. 由于注册广播真正实现过程是在AMS中, 因此跟进AMS中, 首先看registerReceiver()方法, 这里只关心里面的核心部分. 这段代码最终会把远程的InnerReceiver对象以及IntentFilter对象存储起来, 这样整个广播的注册就完成了.



### 9.4.2 广播的发送和接收过程

![](http://www.qingpingshan.com/uploads/allimg/160817/1619104351-1.png)

广播的发送有几种：普通广播、有序广播和粘性广播，他们的发送/接收流程是类似的，因此

只分析普通广播的实现。

1. 广播的发送和接收, 本质就是一个过程的两个阶段. 广播的发送仍然开始于ContextImpl#sendBroadcase()方法, 之所以不是Context, 那是因为Context#sendBroad()是一个抽象方法. 和广播的注册过程一样, ContextWrapper#sendBroadcast()仍然什么都不做, 只是把事情交给了ContextImpl去处理.

2. ContextImpl里面也几乎什么都没有做, 内部直接向AMS发起了一个异步请求用于发送广播.

3. 调用AMS#broadcastIntent()方法，继续调用broadcastIntentLocked()方法。

4. 在broadcastIntentLocked()内部, 会根据intent-filter查找出匹配的广播接收者并经过一系列的条件过滤. 最终会将满足条件的广播接收者添加到BroadcastQueue中, 接着BroadcastQueue就会将广播发送给相应广播接收者.

5. BroadcastQueue#scheduleBroadcastsLocked()方法内并没有立即发送广播, 而是发送了一个BROADCAST_INTENT_MSG类型的消息, BroadcastQueue收到消息后会调用processNextBroadcast()方法。

6. 无序广播存储在mParallelBroadcasts中, 系统会遍历这个集合并将其中的广播发送给他们所有的接收者, 具体的发送过程是通过deliverToRegisteredReceiverLocked()方法实现. deliverToRegisteredReceiverLocked()负责将一个广播发送给一个特定的接收者, 它的内部调用了performReceiverLocked方法来完成具体发送过程.

7. performReceiverLocked()方法调用的ApplicationThread#scheduleRegisteredReceiver()实现比较简单, 它通过InnerReceiver来实现广播的接收

8. scheduleRegisteredReceiver()方法中，receiver.performReceive()中的receiver对应着IIntentReceiver类型的接口. 而具体的实现就是ReceiverDispatcher$InnerReceiver. 这两个嵌套的内部类是所属在LoadedApk中的。

9. 又调用了LoadedApk$ReceiverDispatcher#performReceive()的方法.在performReceiver()这个方法中, 会创建一个Args对象并通过mActivityThread的post方法执行args中的逻辑. 而这些类的本质关系就是:

	- Args: 实现类Runnable

	- mActivityThread: 是一个Handler, 就是ActivityThread中的mH. mH就是ActivityThread$H. 这个内部类H以前说过.

10. 实现Runnable接口的Args中BroadcastReceiver#onReceive()方法被执行了, 也就是说应用已经接收到了广播, 同时onReceive()方法是在广播接收者的主线程中被调用的.



android 3.1开始就增添了两个标记为. 分别是FLAG_INCLUDE_STOPPED_PACKAGES, FLAG_EXCLUDE_STOPPED_PACKAGES. 用来控制广播是否要对处于停止的应用起作用.

- FLAG_INCLUDE_STOPPED_PACKAGES: 包含停止应用, 广播会发送给已停止的应用.

- FLAG_EXCLUDE_STOPPED_PACKAGES: 不包含已停止应用, 广播不会发送给已停止的应用



在android 3.1开始, 系统就为所有广播默认添加了FLAG_EXCLUDE_STOPPED_PACKAGES标识。 当这两个标记共存的时候以FLAG_INCLUDE_STOPPED_PACKAGES(非默认项为主).



应用处于停止分为两种

- 应用安装后未运行

- 被手动或者其他应用强停



开机广播同样受到了这个标志位的影响. 从Android 3.1开始处于停止状态的应用同样无法接受到开机广播, 而在android 3.1之前处于停止的状态也是可以接收到开机广播的.



更多[参考1](http://szysky.com/2016/08/16/%E3%80%8AAndroid-%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2%E3%80%8B-09-%E5%9B%9B%E5%A4%A7%E7%BB%84%E4%BB%B6%E7%9A%84%E5%B7%A5%E4%BD%9C%E8%BF%87%E7%A8%8B/)[、参考2](http://www.apkbus.com/blog-705730-60429.html)[、参考3](http://www.qingpingshan.com/rjbc/az/123206.html)



## 9.5 ContentProvider的工作机制

ContentProvider是一种内容共享型组件, 它通过Binder向其他组件乃至其他应用提供数据. 当ContentProvider所在的进程启动时, ContentProvider会同时启动并发布到AMS中. 要注意:这个时候ContentProvider的onCreate()方法是先于Application的onCreate()执行的,这一点在四大组件是少有的现象.

![](http://images2015.cnblogs.com/blog/902091/201612/902091-20161215233358729-353773763.png)

1. 当一个应用启动时，入口方法是ActivityThread的main方法，其中创建ActivityThread的实例并创建主线程的消息队列；

2. ActivityThread的attach方法中会远程调用ActivityManagerService的attachApplication，并将ApplicationThread提供给AMS，ApplicationThread主要用于ActivityThread和AMS之间的通信；

3. ActivityManagerService的attachApplication会调用ApplicationThread的bindApplication方法，这个方法会通过H切换到ActivityThread中去执行，即调用handleBindApplication方法；

4. handleBindApplication方法会创建Application对象并加载ContentProvider，注意是先加载ContentProvider，然后调用Application的onCreate方法。

5.  ContentProvider启动后, 外界就可以通过它所提供的增删改查这四个接口来操作ContentProvider中的数据源, 这四个方法都是通过Binder来调用的, 外界无法直接访问ContentProvider, 它只能通过AMS根据URI来获取到对应的ContentProvider的Binder接口IContentProvider, 然后再通过IContentProvider来访问ContentProvider中的数据源.



ContentProvider的android:multiprocess属性决定它是否是单实例，默认值是false，也就是默认是单实例。当设置为true时，每个调用者的进程中都存在一个ContentProvider对象。



当调用ContentProvider的insert、delete、update、query方法中的任何一个时，如果ContentProvider所在的进程没有启动的话，那么就会触发ContentProvider的创建，并伴随着ContentProvider所在进程的启动。



以query调用为例

![](http://hujiaweibujidao.github.io/images/androidart_contentprovider.png)

1. 首先会获取IContentProvider对象, 不管是通过acquireUnstableProvider()方法还是直接通过acquireProvider()方法, 他们的本质都是一样的, 最终都是通过acquireProvider方法来获取ContentProvider.

2. ApplicationContentResolver#acquireProvider()方法并没有处理任何逻辑, 它直接调用了ActivityThread#acquireProvider()

3. 从ActivityThread中查找是否已经存在了ContentProvider了, 如果存在那么就直接返回. ActivityThread中通过mProviderMap来存储已经启动的ContentProvider对象, 这个集合的存储类型ArrayMap<ProviderKey, ProviderClientRecord> mProviderMap. 如果目前ContentProvider没有启动, 那么就发送一个进程间请求给AMS让其启动项目目标ContentProvider, 最后再通过installProvider()方法来修改引用计数.

4. AMS是如何启动ContentProvider的呢?首先会启动ContentProvider所在的进程, 然后再启动ContentProvider. 启动进程是由AMS#startProcessLocked()方法来完成, 其内部主要是通过Process#start()方法来完成一个新进程的启动, 新进程启动后其入口方法为ActivityThread#main()方法。

5. ActivityThread#main()是一个静态方法, 在它的内部首先会创建ActivityThread实例并调用attach()方法来进行一系列初始化, 接着就开始进行消息循环. ActivityThread#attach()方法会将Application对象通过AMS#attachApplication方法跨进程传递给AMS, 最终AMS会完成ContentProvider的创建过程.

6. AMS#attachApplication()方法调用了attachApplication(), 然后又调用了ApplicationThread#bindApplication(), 这个过程也属于进程通信.bindApplication()方法会发送一个BIND_APPLICATION类型的消息给mH, 这是一个Handler, 它收到消息后会调用ActivityThread#handleBindApplication()方法.

7. ActivityThread#handlerBindApplication()则完成了Application的创建以及ContentProvider 可以分为如下四个步骤:

	1. 创建ContentProvider和Instrumentation

	2. 创建Application对象

	3. 启动当前进程的ContentProvider并调用onCreate()方法. 主要内部实现是installContentProvider()完成了ContentProvider的启动工作, 首先会遍历当前进程的ProviderInfo的列表并一一调用installProvider()方法来启动他们, 接着将已经启动的ContentProvider发布到AMS中, AMS会把他们存储在ProviderMap中, 这样一来外部调用者就可以直接从AMS中获取到ContentProvider. installProvider()内部通过类加载器创建的ContentProvider实例并在方法中调用了attachInfo(), 在这内部调用了ContentProvider#onCreate()

	4. 调用Application#onCreate()



经过了上述的四个步骤, ContentProvider已经启动成功, 并且其所在的进程的Application也已经成功, 这意味着ContentProvider所在的进程已经完成了整个的启动过程, 然后其他应用就可以通过AMS来访问这个ContentProvider了.



当拿到了ContentProvider以后, 就可以通过它所提供的接口方法来访问它. 这里要注意: 这里的ContentProvider并不是原始的ContentProvider. 而是ContentProvider的Binder类型对象IContentProvider, 而IContentProvider的具体实现是ContentProviderNative和ContentProvider.Transport. 后者继承了前者.



如果还用query方法来解释流程: 那么最开始其他应用通过AMS获取到ContentProvider的Binder对象就是IContentProvider. 而IContentProvider的实际实现者是ContentProvider.Transport. 因此实际上外部应用调用的时候本质上会以进程间通信的方式调用ContentProvider.Transport的query()方法。





# 10 Android的消息机制

Android的消息机制主要是指Handler的运行机制。从开发的角度来说，Handler是Android消息机制的上层接口。Handler的运行需要底层的 MessageQueue 和 Looper 的支撑。

- MessageQueue是一个消息队列，内部存储了一组消息，以队列的形式对外提供插入和删除的工作，内部采用单链表的数据结构来存储消息列表。

- Lopper会以无限循环的形式去查找是否有新消息，如果有就处理消息，否则就一直等待着。

- ThreadLocal并不是线程，它的作用是在每个线程中存储数据。Handler通过ThreadLocal可以获取每个线程中的Looper。

- 线程是默认没有Looper的，使用Handler就必须为线程创建Looper。我们经常提到的主线程，也叫UI线程，它就是ActivityThread，被创建时就会初始化Looper。



## 10.1 Android的消息机制概述

![](http://wujingchao.com/assets/Handler%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6.png)

- Handler的主要作用是将某个任务切换到Handler所在的线程中去执行。为什么Android要提供这个功能呢？这是因为Android规定访问UI只能通过主线程，如果子线程访问UI,程序可能会导致ANR。那我们耗时操作在子线程执行完毕后，我们需要将一些更新UI的操作切换到主线程当中去。所以系统就提供了Handler。

- 系统为什么不允许在子线程中去访问UI呢？ 因为Android的UI控件不是线程安全的，多线程并发访问可能会导致UI控件处于不可预期的状态，为什么不加锁？因为加锁机制会让UI访问逻辑变得复杂；其次锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的执行。所以Android采用了高效的单线程模型来处理UI操作。

- Handler创建时会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper就会报错。Handler可以通过post方法发送一个Runnable到消息队列中，也可以通过send方法发送一个消息到消息队列中，其实post方法最终也是通过send方法来完成。

- MessageQueue的enqueueMessage方法最终将这个消息放到消息队列中，当Looper发现有新消息到来时，处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法就会被调用，注意**Looper是运行Handler所在的线程中的**，这样一来业务逻辑就切换到了Handler所在的线程中去执行了。



## 10.2 Android的消息机制分析



### 10.2.1 ThreadLocal的工作原理

ThreadLocal是一个线程内部的数据存储类，通过它可以在指定线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数据，对于其他线程来说无法获得数据。



在某些特殊的场景下，ThreadLocal可以轻松实现一些很复杂的功能。Looper、ActivityThread以及AMS都用到了ThreadLocal。当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal。



对于Handler来说，它需要获取当前线程的Looper,而Looper的作用于就是线程并且不同的线程具有不同的Looper，通过ThreadLocal可以轻松实现线程中的存取。 



ThreadLocal的另一个使用场景是可以让监听器作为线程内的全局对象而存在，在线程内部只要通过get方法就可以获取到监听器。如果不采用ThreadLocal，只能采用函数参数调用和静态变量的方式。而第一种方式在调用栈很深时很糟糕，第二种方式不具有扩展性，比如同时多个线程执行。



虽然在不同线程访问同一个ThreadLocal对象，但是获得的值却是不同的。不同线程访问同一个ThreadLoacl的get方法，ThreadLocal的get方法会从各自的线程中取出一个数组，然后再从数组中根据当前ThreadLocal的索引去查找对应的Value值。

ThreadLocal的set方法：



    public void set(T value) {

        Thread currentThread = Thread.currentThread();

        //通过values方法获取当前线程中的ThreadLoacl数据——localValues

        Values values = values(currentThread);

        if (values == null) {

            values = initializeValues(currentThread);

        } 

        values.put(this, value);

    }

1. 在 localValues 内部有一个数组： private Object[] table ，ThreadLocal的值就存在这个数组中。

2. ThreadLocal的值在table数组中的存储位置总是ThreadLocal的reference字段所标识的对象的下一个位置。



ThreadLocal的get方法：



    	public T get() {

        // Optimized for the fast path.

        Thread currentThread = Thread.currentThread();

        Values values = values(currentThread);//找到localValues对象

        if (values != null) {

            Object[] table = values.table;

            int index = hash & values.mask;

            if (this.reference == table[index]) {//找到ThreadLocal的reference对象在table数组中的位置

            	return (T) table[index + 1];//reference字段所标识的对象的下一个位置就是ThreadLocal的值

            }

        } else {

        	values = initializeValues(currentThread);

        } 

        return (T) values.getAfterMiss(this);

    }

从ThreadLocal的set/get方法可以看出，它们所操作的对象都是当前线程的localValues对象的table数组，因此在不同线程中访问同一个ThreadLocal的set/get方法，它们ThreadLocal的读/写操作仅限于各自线程的内部。理解ThreadLocal的实现方式有助于理解Looper的工作原理。



### 10.2.2 消息队列的工作原理

消息队列指的是MessageQueue，主要包含两个操作：插入和读取。读取操作本身会伴随着删除操作。



MessageQueue内部通过一个单链表的数据结构来维护消息列表，这种数据结构在插入和删除上的性能比较有优势。



插入和读取对应的方法分别是：enqueueMessage和next方法。

enqueueMessage()的源码实现主要操作就是单链表的插入操作

next()的源码实现也是从单链表中取出一个元素的操作，next()方法是一个无线循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next()方法会返回这条消息并将其从单链表中移除。



### 10.2.3 Looper的工作原理

Looper在Android的消息机制中扮演着消息循环的角色，具体来说就是它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立即处理，否则就一直阻塞在那里。



通过Looper.prepare()方法即可为当前线程创建一个Looper，再通过Looper.loop()开启消息循环。prepareMainLooper()方法主要给主线程也就是ActivityThread创建Looper使用的，本质也是通过prepare方法实现的。



Looper提供quit和quitSafely来退出一个Looper，区别在于quit会直接退出Looper，而quitSafely会把消息队列中已有的消息处理完毕后才安全地退出。 Looper退出后，这时候通过Handler发送的消息会失败，Handler的send方法会返回false。

在子线程中，如果手动为其创建了Looper，在所有事情做完后，应该调用Looper的quit方法来终止消息循环，否则这个子线程就会一直处于等待状态；而如果退出了Looper以后，这个线程就会立刻终止，因此建议不需要的时候终止Looper。

loop()方法会调用MessageQueue的next()方法来获取新消息，而next是是一个阻塞操作，但没有信息时，next方法会一直阻塞在那里，这也导致loop方法一直阻塞在那里。如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息：msg.target.dispatchMessage(msg),这里的msg.target是发送这条消息的Handler对象，这样Handler发送的消息最终又交给Handler来处理了。



### 10.2.4 Handler的工作原理

Handler的工作主要包含消息的发送和接收过程。通过post的一系列方法和send的一系列方法来实现。



Handler发送过程仅仅是向消息队列中插入了一条消息。MessageQueue的next方法就会返回这条消息给Looper，Looper拿到这条消息就开始处理，最终消息会交给Handler的dispatchMessage()来处理，这时Handler就进入了处理消息的阶段。



handler处理消息的过程

![](http://upload-images.jianshu.io/upload_images/2534721-cde6d86b8220f4f3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



Handler的构造方法

1. 派生Handler的子类

2. 通过Callback

`Handler handler = new Handler(callback)`

其中，callback接口定义如下

        public interface Callback{

            public boolean handleMessage(Message msg);

        }

3. 通过Looper

		public Handler(Looper looper){

        this(looper,null,false);

        }



## 10.3 主线程的消息循环

Android的主线程就是ActivityThread，主线程的入口方法为main(String[] args)，在main方法中系统会通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue，并通过Looper.loop()来开启主线程的消息循环。



ActivityThread通过ApplicationThread和AMS进行进程间通信，AMS以进程间通信的方式完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityTread中去执行，即切换到主线程中去执行。四大组件的启动过程基本上都是这个流程。



> Looper.loop()，这里是一个死循环，如果主线程的Looper终止，则应用程序会抛出异常。那么问题来了，既然主线程卡在这里了，（1）那Activity为什么还能启动；（2）点击一个按钮仍然可以响应？ 
>
> 问题1：startActivity的时候，会向AMS（ActivityManagerService）发一个跨进程请求（AMS运行在系统进程中），之后AMS启动对应的Activity；AMS也需要调用App中Activity的生命周期方法（不同进程不可直接调用），AMS会发送跨进程请求，然后由App的ActivityThread中的ApplicationThread会来处理，ApplicationThread会通过主线程线程的Handler将执行逻辑切换到主线程。重点来了，主线程的Handler把消息添加到了MessageQueue，Looper.loop会拿到该消息，并在主线程中执行。这就解释了为什么主线程的Looper是个死循环，而Activity还能启动，因为四大组件的生命周期都是以消息的形式通过UI线程的Handler发送，由UI线程的Looper执行的。
>
> 问题2：和问题1原理一样，点击一个按钮最终都是由系统发消息来进行的，都经过了Looper.loop()处理。 问题2详细分析请看原书作者的[Android中MotionEvent的来源和ViewRootImpl](http://blog.csdn.net/singwhatiwanna/article/details/50775201)。



# 11 Android的线程和线程池

在Android系统，线程主要分为主线程和子线程，主线程处理和界面相关的事情，而子线程一般用于执行耗时操作。AsyncTask底层是线程池；IntentService/HandlerThread底层是线程；



在Android中，线程的形态有很多种：

1. AsyncTask封装了线程池和Handler。

2. HandlerThread是具有消息循环的线程，内部可以使用handler

3. IntentService是一种Service，内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出。由于它是一种Service，所以不容易被系统杀死



操作系统中，线程是操作系统调度的最小单元，同时线程又是一种受限的系统资源，其创建和销毁都会有相应的开销。同时当系统存在大量线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的并发，除非线程数量小于等于CPU的核心数。



频繁创建销毁线程不明智，使用线程池是正确的做法。线程池会缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。



## 11.1 主线程和子线程

主线程主要处理界面交互逻辑，由于用户随时会和界面交互，所以主线程在任何时候都需要有较高响应速度，则不能执行耗时的任务；



android3.0开始，网络访问将会失败并抛出NetworkOnMainThreadException这个异常，这样做是为了避免主线程由于被耗时操作所阻塞从而现ANR现象

## 11.2 Android中的线程形态



### 11.2.1 AsyncTask

AsyncTask是一种轻量级的异步任务类, 他可以在线程池中执行后台任务, 然后把执行的进度和最终的结果传递给主线程并在主线程更新UI. 从实现上来说. AsyncTask封装了Thread和Handler, 通过AsyncTask可以更加方便地执行后台任务,但是AsyncTask并不适合进行特别耗时的后台任务，对于特别耗时的任务来说, 建议使用线程池。



AsyncTask就是一个抽象的泛型类. 这三个泛型的意义

1. Params：参数的类型

2. Progress：后台任务的执行进度的类型

3. Result：后台任务的返回结果的类型



如果不需要传递具体的参数, 那么这三个泛型参数可以用Void来代替.



四个方法 ：

- onPreExecute()

在主线程执行, 在异步任务执行之前, 此方法会被调用, 一般可以用于做一些准备工作

- doInBackground()

 在线程池中执行, 此方法用于执行异步任务, 参数params表示异步任务的输入参数. 在此方法中可以通过publishProgress()方法来更新任务的进度, publishProgress()方法会调用onProgressUpdate()方法. 另外此方法需要返回计算结果给onPostExecute()

- onProgressUpdate()

 在主线程执行,在异步任务执行之后, 此方法会被调用, 其中result参数是后台任务的返回值, 即doInBackground()的返回值.

- onPostExecute()

 在主线程执行, 在异步任务执行之后, 此方法会被调用, 其中result参数是后台任务的返回值, 即doInBackground的返回值.



除了上述的四种方法,还有onCancelled(), 它同样在主线程执行, 当异步任务被取消时, onCancelled()方法会被调用, 这个时候onPostExecute()则不会被调用.



AsyncTask在使用过程中有一些条件限制

1. AsyncTask的类必须在主线程被加载, 这就意味着第一次访问AsyncTask必须发生在主线程, 这个问题不是绝对, 因为在Android 4.1及以上的版本已经被系统自动完成. 在Android 5.0的源码中, 可以看到ActivityThread#main()会调用AsyncTask#init()方法.

2. AsyncTask的对象必须在主线程中创建.

3. execute方法必须在UI线程调用.

4. 不要在程序中直接调用onPreExecute(), onPostExecute(), doInBackground和onProgressUpdate()

5. 一个AsyncTask对象只能执行一次, 即只能调用一次execute()方法, 否则会报运行时异常.

6. 在Android 1.6之前, AsyncTask是串行执行任务的; Android 1.6的时候AsyncTask开始采用线程池里处理并行任务; 但是Android 3.0开始, 为了避免AsyncTask带来的并发错误, AsyncTask又采用了一个线程来串行的执行任务. 尽管如此在3.0以后, 仍然可以通过AsyncTask#executeOnExecutor()方法来并行执行任务.



### 11.2.2 AsyncTask的工作原理

AsyncTask中有两个线程池(SerialExecutor和THREAD_POOL_EXECUTOR)和一个Handler(InternalHandler), 其中线程池SerialExecutor用于任务的排列, 而线程池THREAD_POOL_EXECUTOR用于真正的执行任务, 而InternalHandler用于将执行环境从线程切换到主线程, 其本质仍然是线程的调用过程.



AsyncTask的排队过程：首先系统会把AsyncTask#Params参数封装成FutureTask对象, FutureTask是一个并发类, 在这里充当了Runnable的作用. 接着这个FutureTask会交给SerialExecutor#execute()方法去处理. 这个方法首先会把FutureTask对象插入到任务队列mTasks中, 如果这个时候没有正在活动AsyncTask任务, 那么就会调用SerialExecutor#scheduleNext()方法来执行下一个AsyncTask任务. 同时当一个AsyncTask任务执行完后, AsyncTask会继续执行其他任务直到所有的任务都执行完毕为止, 从这一点可以看出, 在默认情况下, AsyncTask是串行执行的



### 11.2.3 HandlerThread

HandlerThread继承了Thread, 它是一种可以使用Handler的Thread, 它的实现也很简单, 就是run方法中通过Looper.prepare()来创建消息队列, 并通过Looper.loop()来开启消息循环, 这样在实际的使用中就允许在HandlerThread中创建Handler.



从HandlerThread的实现来看, 它和普通的Thread有显著的不同之处. 普通的Thread主要用于在run方法中执行一个耗时任务; 而HandlerThread在内部创建了消息队列, 外界需要通过Handler的消息方式来通知HandlerThread执行一个具体的任务. HandlerThread是一个很有用的类, 在Android中一个具体使用场景就是IntentService.



由于HandlerThread#run()是一个无线循环方法, 因此当明确不需要再使用HandlerThread时, 最好通过quit()或者quitSafely()方法来终止线程的执行.



### 11.2.4 IntentService

IntentSercie是一种特殊的Service，继承了Service并且是抽象类，任务执行完成后会自动停止，优先级远高于普通线程，适合执行一些高优先级的后台任务； IntentService封装了HandlerThread和Handler

1. onCreate方法自动创建一个HandlerThread

2. 用它的Looper构造了一个Handler对象mServiceHandler，这样通过mServiceHandler发送的消息都会在HandlerThread执行；

3. IntentServiced的onHandlerIntent方法是一个抽象方法，需要在子类实现，onHandlerIntent方法执行后，stopSelt(int startId)就会停止服务，如果存在多个后台任务，执行完最后一个stopSelf(int startId)才会停止服务。



## 11.3 Android线程池

优点：

1. 重用线程池的线程，减少线程创建和销毁带来的性能开销

2. 控制线程池的最大并发数，避免大量线程互相抢系统资源导致阻塞

3. 提供定时执行和间隔循环执行功能



Android中的线程池的概念来源于Java中的Executor, Executor是一个接口, 真正的线程池的实现为ThreadPoolExecutor.Android的线程池 大部分都是通 过Executor提供的工厂方法创建的。 ThreadPoolExecutor提供了一系列参数来配制线程池, 通过不同的参数可以创建不同的线程池. 而从功能的特性来分的话可以分成四类.



### 11.3.1 ThreadPoolExecutor

ThreadPoolExecutor是线程池的真正实现, 它的构造方法提供了一系列参数来配置线程池, 这些参数将会直接影响到线程池的功能特性.



    public ThreadPoolExecutor(int corePoolSize,

                     int maximumPoolSize,

                     long keepAliveTime,

                     TimeUnit unit,

                     BlockingQueue<Runnable> workQueue) {

        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,

        Executors.defaultThreadFactory(), defaultHandler);

	}

- corePoolSize: 线程池的核心线程数, 默认情况下, 核心线程会在线程池中一直存活, 即使都处于闲置状态. 如果将ThreadPoolExecutor#allowCoreThreadTimeOut属性设置为true, 那么闲置的核心线程在等待新任务到来时会有超时的策略, 这个时间间隔由keepAliveTime属性来决定. 当等待时间超过了keepAliveTime设定的值那么核心线程将会终止.
- maximumPoolSize: 线程池所能容纳的最大线程数, 当活动线程数达到这个数值之后, 后续的任务将会被阻塞.
- keepAliveTime: 非核心线程闲置的超时时长, 超过这个时长, 非核心线程就会被回收. allowCoreThreadTimeOut这个属性为true的时候, 这个属性同样会作用于核心线程.
- unit: 用于指定keepAliveTime参数的时间单位, 这是一个枚举, 常用的有TimeUtil.MILLISECONDS(毫秒), TimeUtil.SECONDS(秒)以及TimeUtil.MINUTES(分)
- workQueue: 线程池中的任务队列, 通过线程池的execute方法提交的Runnable对象会存储在这个参数中.
- threadFactory: 线程工厂, 为线程池提供创建新线程的功能. ThreadFactory是一个接口.



ThreadPoolExecutor执行任务大致遵循如下规则:

1. 如果线程池中的线程数量未达到核心线程的数量, 那么会直接启动一个核心线程来执行任务.

2. 如果线程池中的线程数量已经达到或者超过核心线程的数量, 那么任务会被插入到任务队列中排队等待执行.

3. 如果在步骤2中无法将任务插入到任务队列中, 这通常是因为任务队列已满, 这个时候如果线程数量未达到线程池的规定的最大值, 那么会立刻启动一个非核心线程来执行任务.

4. 如果步骤3中的线程数量已经达到最大值的时候, 那么会拒绝执行此任务,ThreadPoolExecutor会调用RejectedExecution方法来通知调用者.



AsyncTask的THREAD_POOL_EXECUTOR线程池配置:

1. 核心线程数等于CPU核心数+1

2. 线程池最大线程数为CPU核心数的2倍+1

3. 核心线程无超时机制，非核心线程的闲置超时时间为1秒

4. 任务队列容量是128



### 11.3.2 线程池的分类

** FixedThreadPool **

通过Executor#newFixedThreadPool()方法来创建. 它是一种线程数量固定的线程池, 当线程处于空闲状态时, 它们并不会被回收, 除非线程池关闭了. 当所有的线程都处于活动状态时, 新任务都会处于等待状态, 直到有线程空闲出来. 由于FixedThreadPool只有核心线程并且这些核心线程不会被回收, 这意味着它能够更加快速地响应外界的请求.



** CachedThreadPool **

通过Executor#newCachedThreadPool()方法来创建. 它是一种线程数量不定的线程池, 它只有非核心线程, 并且其最大值线程数为Integer.MAX_VALUE. 这就可以认为这个最大线程数为任意大了. 当线程池中的线程都处于活动的时候, 线程池会创建新的线程来处理新任务, 否则就会利用空闲的线程来处理新任务. 线程池中的空闲线程都有超时机制, 这个超时时长为60S, 超过这个时间那么空闲线程就会被回收.



和FixedThreadPool不同的是, CachedThreadPool的任务队列其实相当于一个空集合, 这将导致任何任务都会立即被执行, 因为在这种场景下SynchronousQueue是无法插入任务的. SynchronousQueue是一个非常特殊的队列, 在很多情况下可以把它简单理解为一个无法存储元素的队列. 在实际使用中很少使用.这类线程比较适合执行大量的耗时较少的任务



** ScheduledThreadPool **

通过Executor#newScheduledThreadPool()方法来创建. 它的核心线程数量是固定的, 而非核心线程数是没有限制的, 并且当非核心线程闲置时会立刻被回收掉. 这类线程池用于执行定时任务和具有固定周期的重复任务



** SingleThreadExecutor **

通过Executor#newSingleThreadPool()方法来创建. 这类线程池内部只有一个核心线程, 它确保所有的任务都在同一个线程中按顺序执行. 这类线程池意义在于统一所有的外界任务到一个线程中, 这使得在这些任务之间不需要处理线程同步的问题



# 12 Bitmap的加载和Cache

主要介绍：

1. 如何高效地加载一个Bitmap

2. Android中常用的缓存策略

3. 如何优化列表的卡顿



## 12.1 Bitmap的高效加载

先来简单介绍一下如何加载一个Bitmap, Bitmap在android中指的是一张图片, 可以是png格式也可以是jpg等其他常见的图片格式.



那么如何加载一个图片?首先BitmapFactory类提供了四种方法: decodeFile(), decodeResource(), decodeStream(), decodeByteArray(). 分别用于从文件系统, 资源文件, 输入流以及字节数组加载出一个Bitmap对象. 其中decodeFile和decodeResource又间接调用了decodeStream()方法, 这四类方法最终是在Android的底层实现的, 对应着BitmapFactory类的几个native方法.



高效加载的Bitmap的核心思想:采用BitmapFactory.Options来加载所需尺寸的图片. 比如说一个ImageView控件的大小为300*300. 而图片的大小为800*800. 这个时候如果直接加载那么就比较浪费资源, 需要更多的内存空间来加载图片, 这不是很必要的. 这里我们就可以先把图片按一定的采样率来缩小图片在进行加载. 不仅降低了内存占用,还在一定程度上避免了OOM异常. 也提高了加载bitmap时的性能.



而通过Options参数来缩放图片: 主要是用到了inSampleSize参数, 即采样率.



- 如果是inSampleSize=1那么和原图大小一样,

- 如果是inSampleSize=2那么宽高都为原图1/2, 而像素为原图的1/4, 占用的内存大小也为原图的1/4

- 如果是inSampleSize=3那么宽高都为原图1/3, 而像素为原图的1/9, 占用的内存大小也为原图的1/9

- 以此类推…..



要知道Android中加载图片具体在内存中的占有的大小是根据图片的像素决定的, 而与图片的实际占用空间大小没有关系.而且如果要加载mipmap下的图片, 还会根据不同的分辨率下的文件夹进行不同的放大缩小.



列举现在有一张图片像素为:1024*1024, 如果采用ARGB8888(四个颜色通道每个占有一个字节,相当于1点像素占用4个字节的空间)的格式来存储.(这里不考虑不同的资源文件下情况分析) 那么图片的占有大小就是1024*1024*4那现在这张图片在内存中占用4MB.

如果针对刚才的图片进行inSampleSize=2, 那么最后占用内存大小为512*512*4, 也就是1MB



采样率的数值必须是大于1的整数是才会有缩放效果, 并且采样率同时作用于宽/高, 这将导致缩放后的图片以这个采样率的2次方递减, 即内存占用缩放大小为1/(inSampleSize的二次方). 如果小于1那么相当于=1的时候. 在官方文档中指出, inSampleSize的取值应该总是为2的指数, 比如1,2,4,8,16,32…如果外界传递inSampleSize不为2的指数, 那么系统会向下取整并选择一个最接近的2的指数来代替. 比如如果inSampleSize=3,那么系统会选择2来代替. 但是这条规则并不作用于所有的android版本, 所以可以当成一个开发建议



整理一下开发中代码流程:



1. 将BitmapFactory.Options的inJustDecodeBounds参数设置为true并加载图片.

2. 从BitmapFactory.Options取出图片的原始宽高信息, 他们对应于outWidth和outHeight参数

3. 根据采样率的规则并结合目标View的所需大小计算出采样率inSampleSize

4. 将BitmapFactory.Options的inJustDecodeBounds参数设为false, 然后重新加载.



inJustDecodeBounds这个参数的作用就是在加载图片的时候是否只是加载图片宽高信息而不把图片全部加载到内存. 所以这个操作是个轻量级的.



通过这些步骤就可以整理出以下的工具加载图片类调用decodeFixedSizeForResource()即可.



    public class MyBitmapLoadUtil {

        /**

         * 对一个Resources的资源文件进行指定长宽来加载进内存, 并把这个bitmap对象返回

         *

         * @param res   资源文件对象

         * @param resId 要操作的图片id

         * @param reqWidth 最终想要得到bitmap的宽度

         * @param reqHeight 最终想要得到bitmap的高度

         * @return 返回采样之后的bitmap对象

         */

        public static Bitmap decodeFixedSizeForResource(Resources res, int resId, int reqWidth, int reqHeight){

            // 首先先指定加载的模式 为只是获取资源文件的大小

            BitmapFactory.Options options = new BitmapFactory.Options();

            options.inJustDecodeBounds = true;

            BitmapFactory.decodeResource(res, resId, options);

            //Calculate Size  计算要设置的采样率 并把值设置到option上

            options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

            // 关闭只加载属性模式, 并重新加载的时候传入自定义的options对象

            options.inJustDecodeBounds = false;

            return BitmapFactory.decodeResource(res, resId, options);

        }

        /**

         *  一个计算工具类的方法, 传入图片的属性对象和 想要实现的目标大小. 通过计算得到采样值

         */

        private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {

            //Raw height and width of image

            //原始图片的宽高属性

            final int height = options.outHeight;

            final int width = options.outWidth;

            int inSampleSize = 1;

            // 如果想要实现的宽高比原始图片的宽高小那么就可以计算出采样率, 否则不需要改变采样率

            if (reqWidth < height || reqHeight < width){

                int halfWidth = width/2;

                int halfHeight = height/2;

                // 判断原始长宽的一半是否比目标大小小, 如果小那么增大采样率2倍, 直到出现修改后原始值会比目标值大的时候

                while((halfHeight/inSampleSize) >= reqHeight && (halfWidth/inSampleSize) >= reqWidth){

                    inSampleSize *= 2;

                }

            }

            return inSampleSize;

        }

    }



## 12.2 Android中的缓存策略

当程序第一次从网络上加载图片后，将其缓存在存储设备中，下次使用这张图片的时候就不用再从网络从获取了。很多时候为了提高应用的用户体验，往往还会把图片在内存中再缓存一份，因为从内存中加载图片比存储设备中快。一般情况会把图片存一份到内存中，一份到存储设备中，如果内存中没找到就去存储设备中找，还没有找到就从网络上下载。



缓存策略包含缓存的添加、获取和删除操作。不管是内存还是存储设备，缓存大小都是有限制的。如何删除旧的缓存并添加新的缓存，就对应缓存算法。



目前常用的一种缓存算法是LRU(Least Recently Used), 最近最少使用算法. 核心思想: 当缓存存满时, 会优先淘汰那些近期最少使用的缓存对象. 采用LRU算法的缓存有两种: LruCache和DiskLruCache,**LruCache用于实现内存缓存, DiskLruCache则充当了存储设备缓存**, 当组合使用后就可以实现一个类似ImageLoader这样的类库.




### 12.2.1 LruCache

LruCache是Android 3.1所提供的一个缓存类, 通过support-v4兼容包可以兼容到早期的Android版本



LruCache是一个泛型类, 它内部采用了一个LinkedHashMap以强引用的方式存储外界的缓存对象, 其提供了get和put方法来完成缓存的获取和添加的操作. 当缓存满了时, LruCache会移除较早使用的缓存对象, 然后在添加新的缓存对象. 普及一下各种引用的区别:

- 强引用: 直接的对象引用

- 软引用: 当一个对象只有软引用存在时, 系统内存不足时此对象会被gc回收

- 弱引用: 当一个对象只有弱引用存在时, 对象会随下一次gc时被回收



LruCache是线程安全的。

LruCache 典型初始化过程：



        int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

        int cacheSize = maxMemory / 8;

        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {

            @Override

            protected int sizeOf(String key, Bitmap value) {

                return value.getRowBytes() * value.getHeight() / 1024;

            }

        };

这里只需要提供缓存的总容量大小(一般为进程可用内存的1/8)并重写 sizeOf 方法即可.sizeOf方法作用是计算缓存对象的大小。这里大小的单位需要和总容量的单位（这里是kb）一致，因此除以1024。一些特殊情况下，需要重写LruCache的entryRemoved方法，LruCache移除旧缓存时会调用entryRemoved方法，因此可以在entryRemoved中完成一些资源回收工作（如果需要的话）。



还有获取和添加方法，都比较简单：



    mMemoryCache.get(key)

    mMemoryCache.put(key,bitmap)

通过remove方法可以删除一个指定的对象。



从Android 3.1开始，LruCache称为Android源码的一部分。



### 12.2.2 DiskLruCache

DiskLruCache用于实现磁盘缓存，DiskLruCache得到了Android官方文档推荐，但它不属于Android SDK的一部分，[源码在这里](https://android.googlesource.com/platform/libcore/+/android-4.1.1_r1/luni/src/main/java/libcore/io/DiskLruCache.java)。



** DiskLruCache的创建 **

DiskLruCache并不能通过构造方法来创建, 他提供了open()方法用于创建自身, 如下所示



    public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)

- File directory: 表示磁盘缓存在文件系统中的存储路径. 可以选择SD卡上的缓存目录, 具体是指/sdcard/Andriod/data/package_name/cache目录, package_name表示当前应用的包名, 当应用被卸载后, 此目录会一并删除掉. 也可以选择data目录下. 或者其他地方. 这里给出的建议:如果应用卸载后就希望删除缓存文件的话 , 那么就选择SD卡上的缓存目录, 如果希望保留缓存数据那就应该选择SD卡上的其他目录.

- int appVersion: 表示应用的版本号, 一般设为1即可. 当版本号发生改变的时候DiskLruCache会清空之前所有的缓存文件, 在实际开发中这个实用性不大.

- int valueCount: 表示单个节点所对应的数据的个数, 一般设为1.

- long maxSize: 表示缓存的总大小, 比如50MB, 当缓存大小超出这个设定值后, DiskLruCache会清除一些缓存而保证总大小不大于这个设定值.





        //初始化DiskLruCache，包括一些参数的设置

        public void initDiskLruCache

        {

            //配置固定参数

            // 缓存空间大小

            private static final long DISK_CACHE_SIZE = 1024 * 1024 * 50;

            //下载图片时的缓存大小

            private static final long IO_BUFFER_SIZE = 1024 * 8;

            // 缓存空间索引，用于Editor和Snapshot，设置成0表示Entry下面的第一个文件

            private static final int DISK_CACHE_INDEX = 0;



            //设置缓存目录

            File diskLruCache = getDiskCacheDir(mContext, "bitmap");

            if(!diskLruCache.exists())

                diskLruCache.mkdirs();

            //创建DiskLruCache对象，当然是在空间足够的情况下

            if(getUsableSpace(diskLruCache) > DISK_CACHE_SIZE)

            {

                try

                {

                    mDiskLruCache = DiskLruCache.open(diskLruCache, 

                            getAppVersion(mContext), 1, DISK_CACHE_SIZE);

                    mIsDiskLruCache = true;

                }catch(IOException e)

                {

                    e.printStackTrace();

                }

            }

    	}



   	 //上面的初始化过程总共用了3个方法

   	 //设置缓存目录

    	public File getDiskCacheDir(Context context, String uniqueName) {

            String cachePath;

            if (Environment.MEDIA_MOUNTED.equals(Environment

                    .getExternalStorageState())

                    || !Environment.isExternalStorageRemovable()) {

                cachePath = context.getExternalCacheDir().getPath();

            } else {

                cachePath = context.getCacheDir().getPath();

            }

            return new File(cachePath + File.separator + uniqueName);

 	   }

	

    	// 获取可用的存储大小

   	 @TargetApi(VERSION_CODES.GINGERBREAD)

    	private long getUsableSpace(File path) {

            if (Build.VERSION.SDK_INT >= VERSION_CODES.GINGERBREAD)

                return path.getUsableSpace();

            final StatFs stats = new StatFs(path.getPath());

            return (long) stats.getBlockSize() * (long) stats.getAvailableBlocks();

   	 }

  	  //获取应用版本号，注意不同的版本号会清空缓存

   	 public int getAppVersion(Context context) {

            try {

                PackageInfo info = context.getPackageManager().getPackageInfo(

                        context.getPackageName(), 0);

                return info.versionCode;

            } catch (NameNotFoundException e) {

                e.printStackTrace();

            }

            return 1;

   	 }



** DiskLruCache的缓存添加 **

DiskLruCache的缓存添加的操作是通过Editor完成的, Editor表示一个缓存对象的编辑对象.



如果还是缓存图片为例子, 每一张图片都通过图片的url为key, 这里由于url可能会有特殊字符所以采用url的md5值作为key. 根据这个key就可以通过edit()来获取Editor对象, 如果这个缓存对象正在被编辑, 那么edit()就会返回null. 即DiskLruCache不允许同时编辑一个缓存对象.



当用.edit(key)获得了Editor对象之后. 通过editor.newOutputStream(0)就可以得到一个文件输出流. 由于之前open()方法设置了一个节点只能有一个数据. 所以在获得输出流的时候传入常量0即可.



有了文件输出流, 可以当网络下载图片时, 图片就可以通过这个文件输出流写入到文件系统上.最后，要通过Editor中commit()来提交写操作, 如果下载中发生异常, 那么使用Editor中abort()来回退整个操作.



** DiskLruCache的缓存查找 **

和缓存的添加过程类似, 缓存查找过程也需要将url转换成key, 然后通过DiskLruCache#get()方法可以得到一个Snapshot对象, 接着在通过Snapshot对象即可得到缓存的文件输入流, 有了文件输入流, 自然就可以得到Bitmap对象. 为了避免加载图片出现OOM所以采用压缩的方式. 在前面对BitmapFactory.Options的使用说明了. 但是这中方法对FileInputStream的缩放存在问题. 原因是FileInputStream是一种有序的文件流, 而两次decodeStream调用会影响文件的位置属性, 这样在第二次decodeStream的时候得到的会是null. 针对这一个问题, 可以通过文件流来得到它所对应的文件描述符, 然后通过BitmapFactory.decodeFileDescription()来加载一张缩放后的图片.



    /**

         * 磁盘缓存的读取

         * @param url

         * @param reqWidth

         * @param reqHeight

         * @return

     */

    private Bitmap loadBitmapFromDiskCache(String url, int reqWidth, int reqHeight) throws IOException

    {

        if(Looper.myLooper() == Looper.getMainLooper())

            Log.w(TAG, "it's not recommented load bitmap from UI Thread");

        if(mDiskLruCache == null)

            return null;



        Bitmap bitmap = null;

        String key = hashKeyForDisk(url);

        Snapshot snapshot = mDiskLruCache.get(key);

        if(snapshot != null)

        {

            FileInputStream fileInputStream = (FileInputStream) snapshot.getInputStream(DISK_CACHE_INDEX);

            FileDescriptor fd = fileInputStream.getFD();

            bitmap = mImageResizer.decodeSampleBitmapFromFileDescriptor(fd, reqWidth, reqHeight);



            if(bitmap != null)

                addBitmapToMemoryCache(key, bitmap);



        }

        return bitmap;      

    }

### 12.2.3 ImageLoader的实现

一个好的ImageLoader应该具备以下几点:



- 图片的压缩
- 网络拉取
- 内存缓存
- 磁盘缓存
- 图片的同步加载
- 图片的异步加载



** 图片压缩功能 **

[ImageResizer](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_12/src/com/ryg/chapter_12/loader/ImageResizer.java)

** 内存缓存和磁盘缓存 **

[ImageLoader](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_12/src/com/ryg/chapter_12/loader/ImageLoader.java) 

** 同步加载和异步加载的接口设计 **

[ImageLoader](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_12/src/com/ryg/chapter_12/loader/ImageLoader.java)  173行



异步加载过程：

1. bindBitmap先尝试从内存缓存读取图片，如果没有会在线程池中调用loadBitmap方法。获取成功将图片封装为LoadResult对象通过mMainHandler向UI线程发送消息。选择线程池和Handler来提供并发能力和异步能力。

2. 为了解决View复用导致的列表错位问题，在给ImageView设置图片之前都会检查它的url有没有发生改变，如果改变就不再给它设置图片。（76行）



## 12.3 ImageLoader的使用



### 12.3.1 照片墙效果

[实现照片墙效果](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_12/src/com/ryg/chapter_12/MainActivity.java)，如果图片都需要是正方形；这样做很快，自定义一个ImageView，重写onMeasure方法。



    @Override

    protected void onMeasure(int widthMeasureSpec,int heightMeasureSpec){

    	super.onMeasure(widthMeasureSpec,widthMeasureSpec);//将原来的参数heightMeasureSpec换成widthMeasureSpec

    }



### 12.3.2 优化列表的卡顿现象

1. 不要在getView中执行耗时操作，不要在getView中直接加载图片。

2. 控制异步任务的执行频率：如果用户刻意频繁上下滑动，getView方法会不停调用，从而产生大量的异步任务。可以考虑在列表滑动停止加载图片；给ListView或者GridView设置 setOnScrollListener 并在 OnScrollListener 的 onScrollStateChanged 方法中判断列表是否处于滑动状态，如果是的话就停止加载图片。

3. 大部分情况下，可以使用硬件加速解决莫名卡顿问题，通过设置 android:hardwareAccelerated="true" 即可为Activity开启硬件加速。



[本章内容更多参考](http://www.jianshu.com/p/765640fe474a)

# 13 综合技术

本章主要讲解:

1. CrashHandler来监视App的crash信息

2. 通过Google的multiDex方案解决Android方法数超过65536的问题

3. Android动态加载dex

4. 反编译



## 13.1 使用CrashHandler来获取应用的crash信息

如何检测崩溃并了解详细的crash信息？ 首先需实现一个uncaughtExceptionHandler对象，在它的uncaughtException方法中获取异常信息并将其存储到SD卡或者上传到服务器中，然后调用Thread的setDefaultUncaughtExceptionHandler为当前进程的所有线程设置异常处理器。



[CrashHandler源码](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_13/CrashTest/src/com/ryg/crashtest/CrashHandler.java)



在Application初始化的时候为线程设置CrashHandler，这样之后，Crash就会通过我们自己的异常处理器来处理异常了。



    public class BaseApplication extends Application {

        @Override

        public void onCreate() {

            super.onCreate();

            CrashHandler crashHandler = CrashHandler.getInstance();

            crashHandler.init(this);

        } 

    }

## 13.2 使用multidex来解决方法数越界

Android中单个dex文件所能包含的最大方法数为65536, 这包含了FrameWork, 依赖的jar包以及应用本身的代码中的所有方法. 会爆出:

		

    com.android.dex.DexIndexOverflowException: method ID not in[0, 0xffff] :65536



可能在一些低版本的手机, 即使没有超过方法数的上限却还是出现错误



    E/dalvikvm: Optimization failed

    E/installd: dexopt failed on '/data/dalvik-cache/.....'

这个现象, 首先dexpot是一个程序, 应用在安装时, 系统会通过dexopt来优化dex文件, 在优化过程中dexopt采用一个固定大小的缓冲区来存储应用中所有方法消息, 这个缓冲区就是linearAlloc. LinearAlloc缓冲区在新版本的Android系统中大小为8MB或者16MB. 在Android 2.2和2.3中却只有5MB. 这是如果方法过多, 即使方法数没有超过65535也有可能会因为存储空间失败而无法安装.



解决方案

- 插件化: 是一套重量级的技术方案, 通过将一个dex拆分成两个或者多个dex,可以在一定程度上解决方法数的越界问题. 但是还有兼容性问题需要考虑, 所以需要权衡是否需要使用这个方案.
- multidex: 这是Google在2014年提出的解决方案.在Android5.0之前需要引入Google提供的android-support-multidex.jar；从5.0开始系统默认支持了multidex，它可以从apk文件中加载多个dex文件。



使用步骤：

1. 修改对应工程目录下的build.gradle文件，在defaultConfig中添加multiDexEnabled

true这个配置项。

2. 在build.gradle的dependencies中添加multidex的依赖：compile

'com.android.support:multidex:#1.0.0'

3. 代码中加入支持multidex功能。

	1. 第一种方案，在manifest文件中指定Application为MultiDexApplication。

	2. 第二种方案，让应用的Application继承MultiDexApplication。

	3. 第三种方案，重写 attachBaseContext 方法，这个方法比onCreate还要先执行。

            public class BaseApplication extends Application {
                @Override
                protected void attachBaseContext(Context base) {
                    super.attachBaseContext(base);
                    MultiDex.install(this);
                }
            }

采用上面的配置项后，如果这个应用方法数没有越界，那么Gradle是不会生成多个dex文件的，当方法数越界后，Gradle就会在apk中打包2个或多个dex文件。当需要指定主dex文件中所包含的类，这时候就需要通过--multi-dex-list来选项来实现这个功能。



    //在对应工程目录下的build.gradle文件，加入

    afterEvaluate {
    println "afterEvaluate"
    tasks.matching {
    it.name.startsWith('dex')
    }.each { dx ->
    def listFile = project.rootDir.absolutePath + '/app/maindexlist.txt'
    println "root dir:" + project.rootDir.absolutePath
    println "dex task found: " + dx.name
    if (dx.additionalParameters == null) {
       dx.additionalParameters = []
    }
    dx.additionalParameters += '--multi-dex'
    dx.additionalParameters += '--main-dex-list=' + listFile
    dx.additionalParameters += '--minimal-main-dex'
    }
    } 

maindexlist.txt



    com/ryg/multidextest/TestApplication.class
    com/ryg/multidextest/MainActivity.class
    // multidex 这9个类必须在主Dex中
    android/support/multidex/MultiDex.class
    android/support/multidex/MultiDexApplication.class
    android/support/multidex/MultiDexExtractor.class
    android/support/multidex/MultiDexExtractor$1.class
    android/support/multidex/MultiDex$V4.class
    android/support/multidex/MultiDex$V14.class
    android/support/multidex/MultiDex$V19.class
    android/support/multidex/ZipUtil.class
    android/support/multidex/ZipUtil$CentralDirectory.class

需要注意multidex的jar中的9个类必须要打包到主dex中，因为Application的attachBaseContext方法中需要用到MultiDex.install(this)需要用到MultiDex。



Multidex的缺点：

1. 启动速度会降低，由于应用启动时会加载额外的dex文件，这将导致应用的启动速度降低，甚至产生ANR现象。

2. 因为Dalvik linearAlloc的bug，可以导致使用multidex的应用无法在Android4.0之前的手机上运行，需要做大量兼容性测试。



## 13.3 Android动态加载技术

动态加载也叫插件化. 当项目越来越大的时候, 可以通过插件化来减轻应用的内存和CPU占用. 还可以实现热插拔, 即可以在不发布新版本的情况下更新某些模块.



[学习一下作者的插件化开源框架：dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)



各种插件化方案都需要解决3个基础性问题

>宿主和插件的概念:宿主是指普通的apk, 而插件一般指经过处理的dex或者apk. 在主流的插件化框架中多采用经过处理的apk来作为插件, 处理方式往往和编译以及打包环节有关, 另外很多插件化框架都需要用到代理Activity的概念, 插件Activity的启动大多数是借助一个代理Activity来实现.



** 资源访问 **

插件中凡是以R开头的资源文件都不能访问。

Activity的工作主要是通过ContextImpl完成的，Activity中有一个mBase的成员变量，它的类型就是ContextImpl。Context有两个获取资源的抽象方法getAsssets()和getResources();只要实现这两个方法就可以解决资源问题。



    protected void loadResources() {  

        try {  

            AssetManager assetManager = AssetManager.class.newInstance();  

            Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);  

            addAssetPath.invoke(assetManager, mDexPath);  

            mAssetManager = assetManager;  

        } catch (Exception e) {  

            e.printStackTrace();  

        }  

        Resources superRes = super.getResources();  

        mResources = new Resources(mAssetManager, superRes.getDisplayMetrics(),  

                superRes.getConfiguration());  

        mTheme = mResources.newTheme();  

        mTheme.setTo(super.getTheme());  

    }

从loadResources()的实现看出，加载资源的方法是反射，通过调用AssetManager的addAssetPath方法，我们可以将一个apk中的资源加载到Resources对象中。传递的路径可以是zip或资源目录，因此直接将apk的路径传给它，资源就加载到AssetManager了。然后再通过AssetManager创建一个新的Resources对象，通过这个对象就可以访问插件apk中的资源了。



接着在代理Activity中实现getAssets()和getResources()。关于代理Activity参考作者的插件化开源框架。



    @Override  

    public AssetManager getAssets() {  

        return mAssetManager == null ? super.getAssets() : mAssetManager;  

    }  



    @Override  

    public Resources getResources() {  

        return mResources == null ? super.getResources() : mResources;  

    }

** Activity的生命周期管理 **

为什么会有这个问题，其实很好理解，apk被宿主程序调起以后，apk中的activity其实就是一个普通的对象，不具有activity的性质，因为系统启动activity是要做很多初始化工作的，而我们在应用层通过反射去启动activity是很难完成系统所做的初始化工作的，所以activity的大部分特性都无法使用包括activity的生命周期管理，这就需要我们自己去管理。



[详细内容参考作者的开源框架介绍](https://github.com/singwhatiwanna/dynamic-load-apk)。



** ClassLoader的管理 **

为了避免多个ClassLoader加载了同一个类所引发的类型转换错误。将不同插件的ClassLoader存储在一个HashMap中。





## 13.4 反编译初步

1. 使用dex2jar和jd-gui反编译apk

2. 使用apktool对apk进行二次打包



以上网上资料特别多，不赘述。



# 14 JNI与NDK编程

Java JNI本意为Java Native Interface(java本地接口), 是为方便java调用C或者C++等本地代码所封装的一层接口. 由于Java的跨平台性导致本地交互能力的不好, 一些和操作系统相关的特性Java无法完成, 于是Java提供了JNI专门用于和本地代码交互.



NDK是android所提供的一个工具合集, 通过NDK可以在Android中更加方便地通过JNI来访问本地代码. NDK还提供了交叉编译工具, 开发人员只需要简单的修改mk文件就可以生成特定的CPU平台的动态库. 好处如下:

1. 代码的保护。由于apk的java层代码很容易被反编译，而C/C++库反编译难度较大。

2. 可以方便地使用C/C++开源库。

3. 便于移植，用C/C++写的库可以方便在其他平台上再次使用

4. 提供程序在某些特定情形下的执行效率，但是并不能明显提升Android程序的性能。



## 14.1 JNI的开发流程

** 在Java中声明natvie方法 **

[创建一个类](https://github.com/singwhatiwanna/android-art-res/blob/master/Chapter_14/JniTest/com/ryg/JniTest.java)

生命了两个native方法：get和set(String)。这是需要在JNI实现的方法。JniTest头部有一个加载动态库的过程， 加载so库名称填入的虽然是jni-test, 但是so库全名称应该是libjni-test.so,这是加载so库的规范。



** 编辑Java源文件得到class文件, 然后通过javah命令导出JNI头文件 **

在包的的根路径, 进行命令操作



    javac com/szysky/note/androiddevseek_14/JNITest.java

    javah com.szysky.note.androiddevseek_14.JNITest

执行之后会在, 操作的路径下生成一个com_szysky_note_androiddevseek_14_JNITest.h头文件, 这个就是第二步生成的东西.

- 函数名:格式遵循:Java_包名_类名_方法名包名之间的.分割全部替换成_分割.

- 参数: jstring是代表String类型参数. 具体的类型关系后面会说明.

	- JNIEnv *: 表示一个指向JNI环境的指针, 可以通过它来访问JNI提供的方法.

	- jobject: 表示java对象中的this.

	- JNIEXPORT和JNICALL: 这是JNI种所定义的宏, 可以在jni.h这个头文件查到





    #ifdef __cplusplus

    extern "C" {

    #endif

而这个宏定义是必须的, 作用是指定extern”C”内部的函数采用C语言的命名风格来编译. 如果设定那么当JNI采用`C++`来实现时, 由于`C/C++`编译过程对函数的命名风格不同, 这将导致JNI在链接时无法根据函数名找到具体的函数, 那么JNI调用肯定会失效.



** 用C/C++实现natvie方法 **

JNI方法是指的Java中声明的native方法, 这里可以选择c++和c来实现. 过程都是类似的. 只有少量的区别, 这里两种都实现一下.



在工程的主目录创建一个子目录, 名称任意, 然后将之前通过javah命令生成的.h头文件复制到创建的目录下, 接着创建test.cpp和test.c两个文件，实现如下:



test.app

    #include "com_szysky_note_androiddevseek_14_JNITest.h"

    #include <stdio.h>

    JNIEXPORT jstring JNICALL Java_com_szysky_note_androiddevseek_114_JNITest_get(JNIEnv *env, jobject thiz){

        printf("执行在c++文件中 get方法\n");

        return env->NewStringUTF("Hello from JNI .");

    }

    JNIEXPORT void JNICALL Java_com_szysky_note_androiddevseek_114_JNITest_get(JNIEnv *env, jobject thiz, jstring string){

        printf("执行在c++文件中 set方法\n");

        char* str = (char*) env->GetStringUTFChars(string, NULL);

        printf("\n, str");

        env->ReleaseStringUTFChars(string, str);

    }

test.c

    #include "com_szysky_note_androiddevseek_14_JNITest.h"

    #include <stdio.h>

    JNIEXPORT jstring JNICALL Java_com_szysky_note_androiddevseek_114_JNITest_get(JNIEnv *env, jobject thiz){

        printf("执行在c文件中 get方法\n");

        return (*env)->NewStringUTF("Hello from JNI .");

    JNIEXPORT void JNICALL Java_com_szysky_note_androiddevseek_114_JNITest_get(JNIEnv *env, jobject thiz, jstring string){

        printf("执行在c文件中 set方法\n");

        char* str = (char*) (*env)->GetStringUTFChars(env, string, NULL);

        printf("%s\n, str");

        (*env)->ReleaseStringUTFChars(env, string, str);

    }}

其实C\C++在实现上很相似, 但是对于env的操作方式有所不同.



    C++: env->ReleaseStringUTFChars(string, str);

    C:  (*env)->ReleaseStringUTFChars(env, string, str);



** 编译so库并在java中调用 **

so库的编译这里采用gcc. 命令cd到放置刚才生成`c/c++`的目录下.

使用如下命令:



    gcc -shared -I /user/lib/jvm/java-7-openjdk-amd64/include -fPIC test.cpp -o libjni-test.so

    gcc -shared -I /user/lib/jvm/java-7-openjdk-amd64/include -fPIC test.c -o libjni-test.so

/user/lib/jvm/java-7-openjdk-amd64是本地jdk的安装路径，libjni-test.so是生产的so库的名字。Java中通过：`System.loadLibrary("jni-test")`加载，其中`lib`和`.so`不需要指出。



切换到主目录，通过Java指令执行Java程序：`java -Djava.library.path=jni com.ryg.JniTest`。其中`-Djava.library.path=jni`指明了so库的路径。



## 14.2 NDK的开发流程

** 下载并配置NDK **

下载好NDK开发包，并且配置好NDK的全局变量。

** 创建一个Android项目，并声明所需的native方法 **



    public static native String getStringFromC();

** 实现Android项目中所声明的native方法 **

1. 生成C/C++的头文件

i. 打开控制台，用cd命令切换到当前项目当前目录

ii. 使用javah命令生成头文件

        javah -classpath bin\classes;C:\MOX\AndroidSDK\platforms\android-23\android.jar -d jni cn.hudp.hellondk.MainActivity

说明：bin\classes 为项目的class文件的相对路径 ； C:\MOX\AndroidSDK\platforms\android-23\android.jar 为android.jar的全路径，因为我们的的Activity使用到了Android SDK，所以生成头文件时需要他; -d jni就是生成的头文件输出到项目的jni文件夹下； 最后跟的cn.hudp.hellondk.MainActivity是native方法所在的类的包名和类名。

2. 编写修改对应的android.mk文件（ mk文件是NDK开发所用到的配置文件）



        # Copyright (C) 2009 The Android Open Source Project
        # #
        Licensed under the Apache License, Version 2.0 (the "License");
        # you may not use this file except in compliance with the License.
        # You may obtain a copy of the License at
        # #
        http://www.apache.org/licenses/LICENSE-2.0
        # #
        Unless required by applicable law or agreed to in writing, software
        # distributed under the License is distributed on an "AS IS" BASIS,
        # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
        # See the License for the specific language governing permissions and
        # limitations under the License.
        # L

        OCAL_PATH := $(call my-dir)
        include $(CLEAR_VARS)

        ## 对应Java部分 System.loadLibrary(String libName) 的libname
        LOCAL_MODULE := hello

        ## 对应c/c++的实现文件名
        LOCAL_SRC_FILES := hello.c
        include $(BUILD_SHARED_LIBRARY)

3. 编写Application.mk，来指定需生成的平台对应的动态库，这里是全平台支持，也可以特殊指定。目前常见的架构平台有armeabi、x86和mips。其中移动设备主要是armeabi，因此大部分apk中只包含armeabi的so库。

		 APP_ABI := all

** 切换到jni目录的父目录，然后通过ndk-build命令编译产生so库 **

ndk-build 命令会默认指定jni目录为本地源码的目录



将编译好的so库放到Android项目中的 app/src/main/jniLbis 目录下，或者通过如下app的gradle设置新的存放so库的目录：



    android{
        ……
        sourceSets.main{
        	jniLibs.srcDir 'src/main/jni_libs'
        }
    }

还可以通过 defaultConfig 区域添加NDK选项



    android{
        ……
        defaultConfig{
            ……
            ndk{
                moduleName "jni-test"
            }
        }
    }
	
还可以在 productFlavors 设置动态打包不同平台CPU对应的so库进apk（ 缩小APK体积）



    gradle

    android{
        ……
        productFlavors{
            arm{
                ndk{
                    adiFilter "armeabi"
                }
            } 
            x86{
                ndk{
                    adiFilter "x86"
                }
            }
        }

    }



** 在Android中调用 **



    public class MainActivity extends Activity {

        public static native String getStringFromC();

        static{//在静态代码块中调用所需要的so文件，参数对应.so文件所对应的LOCAL_MODULE；

            System.loadLibrary("hello");

        }

        @Override

        protected void onCreate(Bundle savedInstanceState) {

            super.onCreate(savedInstanceState);

            setContentView(R.layout.activity_main);

            //在需要的地方调用native方法

            Toast.makeText(getApplicationContext(), get(), Toast.LENGTH_LONG).show();

        }

    }

[更多参考](http://szysky.com/2016/08/26/%E3%80%8AAndroid-%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2%E3%80%8B-14-JNI%E5%92%8CNDK%E7%BC%96%E7%A8%8B/)

## 14.3 JNI的数据类型和类型签名

JNI的数据类型包含两种: 基本类型和引用类型.



基本类型主要有jboolean, jchar, jint等, 和Java中的数据类型对应如下:



|JNI类型	|Java类型	|描述
|--|--|--|--
|jboolean|	boolean	|无符号8位整型
|jbyte	|byte	|无符号8位整型
|jchar|	char	|无符号16位整型
|jshort	|short	|有符号16位整型
|jint	|int	|32位整型
|jlong	|long	|64位整型
|jfloat	|float	|32位浮点型
|jdouble	|double	|64位浮点型
|void	|void|	无类型


JNI中的引用类型主要有类, 对象和数组. 他们和Java中的引用类型的对应关系如下:


|JNI类型	|Java类型|	描述|
|--|--|--|
|jobject|	Object|	Object类型
|jclass	|Class	|Class类型
|jstring	|String	|String类型
|jobjeckArray|	Object[]	|对象数组
|jbooleanArray	|boolean[]	|boolean数组
|jbyteArray	|byte[]	|byte数组
|jcharArray	|char[]	|char数组
|jshortArray	|short[]	|short数组
|jintArray	|int[]	|int数组
|jlongArray	|long[]	|long数组
|jfloatArray|	float[]	|float数组
|jdoubleArray	|double[]	|double数组
|jthrowable	|Throwable	|Throwable


JNI的类型签名标识了一个特定的Java类型, 这个类型既可以是类也可以是方法, 也可以是数据类型.


类的签名比较简单, 它采用L+包名+类型+;的形式, 只需要将其中的.替换为/即可. 例如java.lang.String, 它的签名为Ljava/lang/String;, 末尾的;也是一部分.


基本数据类型的签名采用一系列大写字母来表示, 如下:


|Java类型|	签名|	Java类型|	签名|	Java类型|	签名|
|--|--|--|--|--|--|
|boolean	|Z|	byte	|B	|char	|C
|short	|S	|int	|I	|long	|J
|float|	F|	double	|D	|void	|V

基本数据类型的签名基本都是单词的首字母, 但是boolean除外因为B已经被byte占用, 而long的表示也被Java类签名占用. 所以不同.


而对象和数组, 对象的签名就是对象所属的类签名, 数组的签名[+类型签名例如byte数组. 首先类型为byte,所以签名为B然后因为是数组那么最终形成的签名就是[B.例如如下各种对应:

    char[]      [C
    float[]     [F
    double[]    [D
    long[]      [J
    String[]    [Ljava/lang/String;
    Object[]    [Ljava/lang/Object;

如果是多维数组那么就根据数组的维度多少来决定[的多少, 例如int[][]那么就是[[I


方法的签名为(参数类型签名)+返回值类型签名。

- 方法boolean fun(int a, double b, int[] c). 参数类型的签名是连在一起, 那么按照方法的签名规则就是(ID[I)Z
- 方法:void fun(int a, String s, int[] c), 那么签名就是(ILjava/lang/String;[I)V
- 方法:int fun(), 对应签名()I
- 方法:int fun(float f), 对应签名(F)I


## 14.4 JNI调用Java方法的流程

JNI调用java方法的流程是先通过类名找到类, 然后在根据方法名找到方法的id, 最后就可以调用这个方法了. 如果是调用Java的非静态方法, 那么需要构造出类的对象后才可以调用它。



演示一下调用静态的方法

1. 首先在java中声明要被调用的静态方法. 这里触发的时机是一个按钮的点击,自行添加

        static{
                System.loadLibrary("jni-test");
            }
        /**
         * 定义一个静态方法 , 提供给JNI调用
         */
        public static void methodCalledByJni(String fromJni){
            Log.e("susu", "我是从JNI被调用的消息,  JNI返回的值是:"+fromJni );
        }
        // 定义调用本地方法, 好让本地方法回调java中的方法
        public native void callJNIConvertJavaMethod();
        @Override
        public void onClick(View view) {
            switch (view.getId()){
                case R.id.btn_jni2java:
                    // 调用JNI的方法
                    callJNIConvertJavaMethod();
                    break;
            }
        }

2. 在JNI的test.cpp中添加一个c的函数用来处理调用java的逻辑, 并提供一个方法供java代码调起来触发. 一共两个方法。

        // 定义调用java中的方法的函数
        void callJavaMethod( JNIEnv *env, jobject thiz){
            // 先找到要调用的类
            jclass clazz = env -> FindClass("com/szysky/note/androiddevseek_14/MainActivity");
            if (clazz == NULL){
                printf("找不到要调用方法的所属类");
                return;
            }
            // 获取java方法id
            // 参数二是调用的方法名,  参数三是方法的签名
            jmethodID id = env -> GetStaticMethodID(clazz, "methodCalledByJni", "(Ljava/lang/String;)V");
            if (id == NULL){
                printf("找不到要调用方法");
                return;
            }
            jstring msg = env->NewStringUTF("我是在c中生成的字符串");
            // 开始调用java中的静态方法
            env -> CallStaticVoidMethod(clazz, id, msg);
        }
        void Java_com_szysky_note_androiddevseek_114_MainActivity_callJNIConvertJavaMethod(JNIEnv *env, jobject thiz){
            printf("调用c代码成功, 马上回调java中的代码");
            callJavaMethod(env, thiz);
        }

稍微说明一下, 程序首先根据类名com/szysky/note/androiddevseek_14/MainActivity找到类, 然后在根据方法名methodCalledByJni找到方法, 并传入方法对应签名(Ljava/lang/String;), 最后通过JNIEnv对象的CallStaticVoidMethod()方法来完成最终调用。

最后只要在Java_com_szysky_note_androiddevseek_114_MainActivity_callJNIConvertJavaMethod方法中调用callJavaMethod方法即可.

流程就是–> 按钮触发了点击的onClikc –> 然后Java中会调用JNI的callJNIConvertJavaMethod() –> JNI的callJNIConvertJavaMethod()方法内部会调用具体实现回调Java中的方法callJavaMethod() –> 方法最终通过CallStaticVoidMethod()调用了Java中的methodCalledByJni()来接收一个参数并打印一个log。


结果图:

![](http://szysky.com/2016/08/26/%E3%80%8AAndroid-%E5%BC%80%E5%8F%91%E8%89%BA%E6%9C%AF%E6%8E%A2%E7%B4%A2%E3%80%8B-14-JNI%E5%92%8CNDK%E7%BC%96%E7%A8%8B/JNIcallJava.png)

生成so库的文件保存在git中的app/src/main/backup目录下一个两个版本代码, 第一个就是第二小节中的NDK开发代码, 第二个就是第四小节的代码就是目前的. 而so库是最新的, 包含了所有的JNI代码生成的库文件。


JNI调用Java的过程和Java中方法的定义有很大关联, 针对不同类型的java方法, JNIEnv提供了不同的接口去调用, 更为细节的部分要去开发中或者去网站去了解更多.


# 15 Android性能优化

Android设备作为一种移动设备，不管是内存还是CPU的性能都受到了一定的限制，也意味着Android程序不可能无限制的使用内存和CPU资源，过多的使用内存容易导致OOM，过多的使用CPU资源容易导致手机变得卡顿甚至无响应（ANR）。这也对开发人员提出了更高的要求。 本章主要介绍一些有效的性能优化方法。主要包括布局优化、绘制优化、内存泄漏优化、响应速度优化、ListView优化、Bitmap优化、线程优化等；同时还介绍了ANR日志的分析方法。

Google官方的Android性能优化典范专题短视频课程是学习Android性能优化极佳的课程，目前已更新到第五季； [youku地址](http://v.youku.com/v_show/id_XMTQ4MDU3Nzc3Mg==.html?f=26771407&from=y1.7-1.3)

## 15.1 Android的性能优化方法

### 15.1.1 布局优化

布局优化的思想就是尽量减少布局文件的层级，这样绘制界面时工作量就少了，那么程序的性能自然就高了。


- 删除无用的控件和层级
- 有选择的使用性能较低的ViewGroup，如果布局中既可以使用Linearlayout也可以使用RelativeLayout，那就是用LinearLayout，因为RelativeLayout功能比较复杂，它的布局过程需要花费更多的CPU时间。
有时候通过LinearLayou无法实现产品效果，需要通过嵌套来完成，这种情况还是推荐使用RelativeLayout，因为ViewGroup的嵌套相当于增加了布局的层级，同样降低程序性能。
- 采用标签、标签和ViewStub
  - include标签
 
	<include>标签用于布局重用，可以将一个指定的布局文件加载到当前布局文件中。<include>只支持android:layout开头的属性，当然android:id这个属性是个特例；如果指定了android:layout这种属性，那么要求android:layoutwidth和android:layout_height必须存在，否则android:layout属性无法生效。如果<include>指定了id属性，同时被包含的布局文件的根元素也指定了id属性，会以<include>指定的这个id属性为准。
  
  - merge标签

	<merge>标签一般和<include>标签一起使用从而减少布局的层级。如果当前布局是一个竖直方向的LinearLayout，这个时候被包含的布局文件也采用竖直的LinearLayout，那么显然被包含的布局文件中的这个LinearLayout是多余的，通过<merge>标签就可以去掉多余的那一层LinearLayout。

  - ViewStub

	ViewStub意义在于按需加载所需的布局文件，因为实际开发中，有很多布局文件在正常情况下是不会现实的，比如网络异常的界面，这个时候就没必要在整个界面初始化的时候将其加载进来，在需要使用的时候再加载会更好。在需要加载ViewStub布局时：

			((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
            //或者
            View importPanel = ((ViewStub)findViewById(R.id.stub_import)).inflate();

当ViewStub通过setVisibility或者inflate方法加载后，ViewStub就会被它内部的布局替换掉，ViewStub也就不再是整个布局结构的一部分了。

### 15.1.2 绘制优化

View的onDraw方法要避免执行大量的操作；

- onDraw中不要创建大量的局部对象，因为onDraw方法会被频繁调用，这样就会在一瞬间产生大量的临时对象，不仅会占用过多内存还会导致系统频繁GC，降低程序执行效率。
- onDraw也不要做耗时的任务，也不能执行成千上万的循环操作，尽管每次循环都很轻量级，但大量循环依然十分抢占CPU的时间片，这会造成View的绘制过程不流畅。根据Google官方给出的标准，View绘制保持在60fps是最佳的，这也就要求每帧的绘制时间不超过16ms(1000/60)；所以要尽量降低onDraw方法的复杂度。


### 15.1.3 内存泄露优化

内存泄露是最容易犯的错误之一，内存泄露优化主要分两个方面；一方面是开发过程中避免写出有内存泄露的代码，另一方面是通过一些分析工具如LeakCanary或MAT来找出潜在的内存泄露继而解决。

- 静态变量导致的内存泄露

 比如Activity内，一静态Conext引用了当前Activity，所以当前Activity无法释放。或者一静态变量，内部持有了当前Activity，Activity在需要释放的时候依然无法释放。

- 单例模式导致的内存泄露

 比如单例模式持有了Activity，而且也没用解注册的操作。因为单例模式的生命周期和Application保存一致，生命周期比Activity要长，这样一来就导致Activity对象无法及时被释放。

- 属性动画导致的内存泄露 

属性动画中有一类无限循环的动画，如果在Activity播放了此类动画并且没有在onDestroy中去停止动画，那么动画会一直播放下去，并且这个时候Activity的View会被动画持有，而View又持有了Activity，最终导致Activity无法释放。解决办法是在Activity的onDrstroy中调用animator.cancel()来停止动画。



### 15.1.4 响应速度优化和ANR日志分析

响应速度优化的核心思想就是避免在主线程中去做耗时操作，将耗时操作放在其他线程当中去执行。Activity如果5秒无法响应屏幕触摸事件或者键盘输入事件就会触发ANR，而BroadcastReceiver如果10秒还未执行完操作也会出现ANR。

当一个进程发生ANR以后系统会在/data/anr的目录下创建一个文件traces.txt，通过分析该文件就能定位出ANR的原因。

通过一个例子来了解如何去分析文件, 首先在onCreate()添加如下代码, 让主线程等待一个锁,然后点击返回5秒后会出现ANR。



    @Override
    protected void onCreate(Bundle savedInstanceState) {
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_main);
       // 以下代码是为了模拟一个ANR的场景来分析日志
       new Thread(new Runnable() {
           @Override
           public void run() {
               testANR();
           }
       }).start();
       SystemClock.sleep(10);
       initView();
    }
    /**
    *  以下两个方法用来模拟出一个稍微不好发现的ANR
    */
    private synchronized void testANR(){
       SystemClock.sleep(3000 * 1000);
    }
    private synchronized void initView(){}


这样会出现ANR, 然后导出/data/anr/straces.txt文件. 因为内容比较多只贴出关键部分

    DALVIK THREADS (15):

    "main" prio=5 tid=1 Blocked
      | group="main" sCount=1 dsCount=0 obj=0x73db0970 self=0xf4306800
      | sysTid=19949 nice=0 cgrp=apps sched=0/0 handle=0xf778d160
      | state=S schedstat=( 151056979 25055334 199 ) utm=5 stm=9 core=1 HZ=100
      | stack=0xff5b2000-0xff5b4000 stackSize=8MB
      | held mutexes=
      at com.szysky.note.androiddevseek_15.MainActivity.initView(MainActivity.java:0)
      - waiting to lock <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity) 
      - held by thread 15
      at com.szysky.note.androiddevseek_15.MainActivity.onCreate(MainActivity.java:42)

这段可以看出最后指明了ANR发生的位置在ManiActivity的42行. 并且通过上面看出initView方法正在等待一个锁<0x2fbcb3de>锁的类型是一个MainActivity对象. 并且这个锁已经被线程id为15(tid=15)的线程持有了. 接下来找一下线程15

    "Thread-404" prio=5 tid=15 Sleeping
      | group="main" sCount=1 dsCount=0 obj=0x12c00f80 self=0xeb95bc00
      | sysTid=19985 nice=0 cgrp=apps sched=0/0 handle=0xef34be80
      | state=S schedstat=( 391248 0 1 ) utm=0 stm=0 core=2 HZ=100
      | stack=0xe2bfe000-0xe2c00000 stackSize=1036KB
      | held mutexes=
      at java.lang.Thread.sleep!(Native method)
      - sleeping on <0x2e3896a7> (a java.lang.Object)
      at java.lang.Thread.sleep(Thread.java:1031)
      - locked <0x2e3896a7> (a java.lang.Object)
      at java.lang.Thread.sleep(Thread.java:985)
      at android.os.SystemClock.sleep(SystemClock.java:120)
      at com.szysky.note.androiddevseek_15.MainActivity.testANR(MainActivity.java:50)
      - locked <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity)


tid = 15 就是相关信息如上, 首行已经标出线程的状态为Sleeping, 原因在50行, 就是SystemClock.sleep(3000 * 1000);这句话. 也就是testANR(). 而最后一行也表明了持有的locked<0x2fbcb3de>就是主线程在等待的那个锁对象.

### 15.1.5 ListView优化和Bitmap优化

ListView/GridView优化：

1. 采用ViewHolder避免在getView中执行耗时操作

2. 其次通过列表的滑动状态来控制任务的执行频率，比如快速滑动时不是和开启大量异步任务

3. 最后可以尝试开启硬件加速使得ListView的滑动更加流畅。



Bitmap优化：主要是想是根据需要对图片进行采样显示，详细请参考12章。



### 15.1.6 线程优化

主要思想就是采用线程池, 避免程序中存在大量的Thread. 线程池可以重用内部的线程, 避免了线程创建和销毁的性能开销. 同时线程池还能有效的控制线程的最大并发数, 避免了大量线程因互相抢占系统资源从而导致阻塞现象的发生.详细参考第11章的内容。



### 15.1.7 一些性能优化的小建议

1. 避免创建过多的对象，尤其在循环、onDraw这类方法中，谨慎创建对象；

2. 不要过多的使用枚举，枚举占用的内存空间比整形大。[Android 中如何使用annotion替代Enum](http://szysky.com/2016/05/20/Android-%E4%B8%AD%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8annotion%E6%9B%BF%E4%BB%A3Enum/)

3. 常量使用static final来修饰；

4. 使用一些Android特有的数据结构，比如 SparseArray 和 Pair 等，他们都具有更好的性能；

5. 适当的使用软引用和弱引用；

6. 采用内存缓存和磁盘缓存；

7. 尽量采用静态内部类，这样可以避免非静态内部类隐式持有外部类所导致的内存泄露问题。



## 15.2 内存泄漏分析工具MAT

MAT全程Eclipse Memory Analyzer, 是一个内存泄漏分析工具. 下载后解压即可. 下载地址http://www.eclipse.org/mat/downloads.php. 这里仅简单说一下. 这个我没有手动去实践, 就当个记录, 因为现在Android Studio可以直接分析hprof文件.



可以手动写一个会造成内存泄漏的代码, 然后打开DDMS, 然后选中要分析的进程, 然后单击Dump HPROF file这个按钮. 等一小段会生成一个文件. 这个文件不能被MAT直接识别. 需要使用Android SDK中的工具进行格式转换一下.这个工具在platform-conv文件夹下



`hprof-conv 要转换的文件名 输出的文件名 `文件名的签名有包名.



然后打开MAT通过菜单打开转换后的这个文件. 这里常用的就有两个



- Histogram: 可以直观的看出内存中不同类型的buffer的数量和占用内存大小

- Dominator Tree: 把内存中的对象按照从大到小的顺序进行排序, 并且可以分析对象之间的引用关系, 内存泄漏分析就是通过这个完成的.



分析内存泄漏的时候需要分析Dominator Tree里面的内存信息, 一般会不直接显示出来, 可以按照从大到小的顺序去排查一遍. 如果发生了了泄漏, 那么在泄漏对象处右键单击Path To GC Roots->exclude wake/soft references. 可以看到最终是什么对象导致的无法释放. 刚才的操作之所以排除软引用和弱引用是因为,大部分情况下这两种类型都可以被gc回收掉,所以基本也就不会造成内存泄漏.



同样这里也可以使用搜索功能, 假如我们手动模拟了内存泄漏, 泄漏的对象就是Activity那么我们back退出重进循环几次, 会发现其实很多个Activit对象.



## 15.3 提高程序的可维护性

** 提高可读性 **

1. 命名规范

2. 代码之间排版需留出合理的空白来区分不同的代码块

3. 针对非常关键的代码添加注释。



** 代码的层级性 **

不要把一段业务逻辑放在一个方法或者一个类中全部实现，要把它分成几个子逻辑，然后每个子逻辑做自己的事情，这样即显得代码层级分明，这样利于提高程序的可扩展性。



** 程序的扩展性 **

由于很多时候在开发过程中无法保证已经做好的需求不在后面的版本发生更改, 因此在写程序的时候要时刻考虑到扩展的问题, 考虑如果这个逻辑以后发生了改变那么哪些需要修改, 以及怎样在以后修改的时候降低工作量, 而面向扩展编程可以让程序具有很好的扩展性.



恰当的使用设计模式可以提高代码的可维护性和可扩展性，Android程序容易遇到性能瓶颈，要控制设计的度，不能太牵强，避免过度设计。作者推荐查看《 大话设计模式》 和《 Android源码设计模式解析和实战》 这两本书来学习设计模式。



本笔记整理自:   https://www.gitbook.com/book/tom510230/android_ka_fa_yi_shu_tan_suo/details

参考文章：http://szysky.com/tags/#笔记、http://blog.csdn.net/player_android/article/category/6577498