title: FileDownloader
date: 2015-12-23 11:18:03
tags:
- Download
- Parallel
- Serial
- Project

---

[ ![Download][bintray_svg] ][bintray_url]

> Android 文件下载引擎，稳定、高效、简单易用
> 本引擎目前基于口碑很好的okhttp

### 特点

> 总之，你负责大胆高并发高频率的往下载引擎里面以队列(请求需要并行处理/串行处理)/单任务的形式放任务，引擎负责给你达预期稳定、高效的结果输出

<!-- more -->

#### 1. 稳定:

- 高并发: 兼容高频率不断入队 并行任务队列/串行任务队列/单一下载任务，甚至其中包含大量重复任务(url与path都相同)，不同队列独立运作，不同任务相对独立运作，互不干涉
- 独立进程: 在需要下载时，启动独立进程，减小对应用本身的影响，相比较`android.app.DownloadManager`，`FileDownloader`采用AIDL基于Binder更加高效，并且不会受到其他应用下载的影响。


#### 2. 冗余低:

- **默认** 通过 __文件有效性检测__ 已经下载并且下载完成的任务不重复启动下载
- 通过 __实时监控已启动的任务或排队中的任务__ ，校对准备进入的任务是否是重复任务(url与path相同)，以`warn`抛出用户层，防止重复任务冗余下载
- 通过 __本地数据库及自动断点续传__ 结合相关脏数据矫正，保证任务下载进度尽可能的被快照，只要后端带有`etag`(七牛默认会带)，无论是进程被杀，还是其他任何异常情况，下次启动自动从上次有效位置开始续传，不重复下载


#### 3. 需要注意

- 为了绝大多数使用性能考虑，目前下载引擎目前受限于int可表示的范围，而我们的回调`total`与`so far`以byte为单位回调，因此最大只能表示到`2^31-1`=2_147_483_647 = 1.99GB(ps: 如果有更大的文件下载需求，提issue，我们会进行一些巧妙的优化，利用负值区间？根据大小走特殊通道传输?)
- 暂停: paused, 恢复: 直接调用start，默认就是断点续传

#### 4. 使用okHttp并使用其中的一些默认属性

- retryOnConnectionFailure: Unreachable IP addresses/Stale pooled connections/Unreachable proxy servers
- connection/read/write time out 10s

## I. 效果

![][mix_gif]
![][parallel_gif]
![][serial_gif]
![][single_gif]
![][single_progress_gif]


## II. 使用

在项目中引用:

```
compile 'com.liulishuo.filedownloader:library:0.0.9'
```

> 假如发现IDE没有自动绑定源码到拉下来的`.class`，请下载[library-0.0.9-sources.jar][library_last_source_jar]，然后主动`Choose Sources...`进行绑定，这样方便阅读与debug。有任何问题，提issue即可。

#### 全局初始化在`Application.onCreate`中

```

public XXApplication extends Application{

    ...
    @Override
    public void onCreate() {
        // 不耗时，做一些简单初始化准备工作，不会启动下载进程
        FileDownloader.init(this);
    }

    ...
}

```

#### 启动单任务下载

```

FileDownloader.getImpl().create(url)
        .setPath(path)
        .setListener(new FileDownloadListener() {
            @Override
            protected void pending(BaseDownloadTask task, int soFarBytes, int totalBytes) {
            }

            @Override
            protected void connected(BaseDownloadTask task, String etag, boolean isContinue, int soFarBytes, int totalBytes) {
            }

            @Override
            protected void progress(BaseDownloadTask task, int soFarBytes, int totalBytes) {
            }

            @Override
            protected void blockComplete(BaseDownloadTask task) {
            }

            @Override
            protected void completed(BaseDownloadTask task) {
            }

            @Override
            protected void paused(BaseDownloadTask task, int soFarBytes, int totalBytes) {
            }

            @Override
            protected void error(BaseDownloadTask task, Throwable e) {
            }

            @Override
            protected void warn(BaseDownloadTask task) {
            }
        }).start();
```

#### 启动多任务下载

```

final FileDownloadListener queueTarget = new FileDownloadListener() {
            @Override
            protected void pending(BaseDownloadTask task, int soFarBytes, int totalBytes) {

            }

            @Override
            protected void connected(BaseDownloadTask task, String etag, boolean isContinue, int soFarBytes, int totalBytes) {

            }

            @Override
            protected void progress(BaseDownloadTask task, int soFarBytes, int totalBytes) {

            }

            @Override
            protected void blockComplete(BaseDownloadTask task) {

            }

            @Override
            protected void completed(BaseDownloadTask task) {

            }

            @Override
            protected void paused(BaseDownloadTask task, int soFarBytes, int totalBytes) {

            }

            @Override
            protected void error(BaseDownloadTask task, Throwable e) {

            }

            @Override
            protected void warn(BaseDownloadTask task) {

            }
        };

for (String url : URLS) {
    FileDownloader.getImpl().create(url)
            .setListener(queueTarget)
            .ready();
}

if(serial){
    // 串行执行该队列
    FileDownloader.getImpl().start(queueTarget, true);
}

if(parallel){
    // 并行执行该队列
    FileDownloader.getImpl().start(queueTarget, false);
}

```

#### 全局接口说明(`FileDownloader.`)

> 所有的暂停，就是停止，会释放所有资源并且停到所有相关线程，下次启动的时候默认会断点续传

| 方法名 | 备注
| --- | ---
| init(Application) |  简单初始化，不会启动下载进程
| create(url:String) | 创建一个下载任务
| start(listener:FileDownloadListener, isSerial:boolean) | 启动是相同监听器的任务，串行/并行启动
| pause(listener:FileDownloadListener) | 暂停启动相同监听器的任务
| pauseAll(void) | 暂停所有任务
| pause(downloadId) | 启动downloadId的任务
| getSoFar(downloadId) | 获得下载Id为downloadId的soFarBytes
| getTotal(downloadId) | 获得下载Id为downloadId的totalBytes
| bindService(void) | 主动启动下载进程(可事先调用该方法(可以不调用)，保证第一次下载的时候没有启动进程的速度消耗)
| unBindService(void) | 主动停止下载进程(如果不调用该方法，进程闲置一段时间以后，系统调度会自动将其回收)

#### Task接口说明

| 方法名 | 备注
| --- | ---
| setPath(path:String) | 下载文件的存储绝对路径
| setListener(listener:FileDownloadListener) | 设置监听，可以以相同监听组成队列
| setCallbackProgressTimes(times:int) | 设置progress最大回调次数
| setTag(tag:Object) | 内部不会使用，在回调的时候用户自己使用
| setForceReDownload(isForceReDownload:boolean) | 强制重新下载，将会忽略检测文件是否健在
| setFinishListener(listener:FinishListener) | 结束监听，仅包含结束(over(void))的监听
| ready(void) | 用于队列下载的单任务的结束符(见上面:启动多任务下载的案例)
| start(void) | 启动下载任务
| pause(void) | 暂停下载任务(也可以理解为停止下载，但是在start的时候默认会断点续传)
| getDownloadId(void):int | 获取唯一Id(内部通过url与path生成)
| getUrl(void):String | 获取下载连接
| getCallbackProgressTimes(void):int | 获得progress最大回调次数
| getPath(void):String | 获取下载文件存储路径
| getListener(void):FileDownloadListener | 获取监听器
| getSoFarBytes(void):int | 获取已经下载的字节数
| getTotalBytes(void):int | 获取下载文件总大小
| getStatus(void):int | 获取当前的状态
| isForceReDownload(void):boolean | 是否强制重新下载
| getEx(void):Throwable | 获取下载过程抛出的Throwable
| isReusedOldFile(void):boolean | 判断是否是直接使用了旧文件(检测是有效文件)，没有启动下载
| getTag(void):Object | 获取用户setTag进来的Object
| isContinue(void):boolean | 是否成功断点续传
| getEtag(void):String | 获取当前下载获取到的ETag

#### 监听器(`FileDownloadListener`)说明

##### 一般的下载回调流程:

```
pending -> connected -> (progress <->progress) -> blockComplete -> completed
```

##### 可能会遇到以下回调而直接终止整个下载过程:

```
paused / completed / error / warn
```

##### 如果检测存在已经下载完成的文件(可以通过`isReusedOldFile`进行决策是否是该情况)(也可以通过`setForceReDownload(true)`来避免该情况):

```
blockComplete -> completed
```

##### 方法说明

| 回调方法 | 备注 | 带回数据
| --- | --- | ---
| pending | 等待，已经进入下载队列 | 数据库中的soFarBytes与totalBytes
| connected | 已经连接上 | ETag, 是否断点续传, soFarBytes, totalBytes
| progress | 下载进度回调 | soFarBytes
| blockComplete | 在完成前同步调用该方法，此时已经下载完成 | -
| completed | 完成整个下载过程 | -
| paused | 暂停下载 | soFarBytes
| error | 下载出现错误 | 抛出的Throwable
| warn | 在下载队列中(正在等待/正在下载)已经存在相同下载连接与相同存储路径的任务 | -

## III. Open Source

[lingochamp/FileDownloader](https://github.com/lingochamp/FileDownloader)

[mix_gif]: https://github.com/lingochamp/FileDownloader/raw/master/art/mix.gif
[parallel_gif]: https://github.com/lingochamp/FileDownloader/raw/master/art/parallel.gif
[serial_gif]: https://github.com/lingochamp/FileDownloader/raw/master/art/serial.gif
[single_progress_gif]: https://github.com/lingochamp/FileDownloader/raw/master/art/single_progress.gif
[single_gif]: https://github.com/lingochamp/FileDownloader/raw/master/art/single.gif
[bintray_svg]: https://api.bintray.com/packages/jacksgong/maven/FileDownloader/images/download.svg
[bintray_url]: https://bintray.com/jacksgong/maven/FileDownloader/_latestVersion
[library_last_source_jar]: https://bintray.com/artifact/download/jacksgong/maven/com/liulishuo/filedownloader/library/0.0.9/library-0.0.9-sources.jar
