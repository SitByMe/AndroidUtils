# AndroidUtils
Android功能代码集合

#### 1. 权限管理</br>
依赖：
```
implementation 'com.github.tbruyelle:rxpermissions:0.10.2'
```
**注意**：未在AndroidManifest.xml中添加的权限，均会被认定为被禁止的不再询问。</br>
使用：</br>
（1）注册RxPermissions
```
 protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RxPermissions rxPermissions = new RxPermissions(this);
        rxPermissions.setLogging(true);
    }
```
（2）调用PermissionsUtils的方法进行权限检测并监听更改
```
private void requestPermissions(final RxPermissions rxPermissions) {
        PermissionsUtils.setOnPermissionsRequestResultListener(rxPermissions, new PermissionsUtils.OnPermissionsRequestResultListener() {
            @Override
            public void allDenied() {
                ToastUtils.showLong("所有权限已被允许");
            }

            @Override
            public void hasUnDenied() {
                ToastUtils.showLong("有需要询问的权限");
            }

            @Override
            public void hasNeverAgain() {
                ToastUtils.showLong("有被禁止的权限");
            }
        }).requestEachCombined(Constants.permissions);
    }
```

#### 2. 下载管理</br>
（1）配置下载管理器
```
    /*使用DownLoadManager时只能通过DownLoadService.getDownLoadManager()的方式来获取下载管理器，不能通过new DownLoadManager()的方式创建下载管理器*/
    private DownLoadManager manager;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        this.setContentView(R.layout.activity_download_demo);
        //下载管理器需要启动一个Service,在刚启动应用的时候需要等Service启动起来后才能获取下载管理器，所以稍微延时获取下载管理器
        handler.sendEmptyMessageDelayed(1, 50);
    }

    Handler handler = new Handler(Looper.myLooper()) {
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
            /*获取下载管理器*/
            manager = DownLoadService.getDownLoadManager();
            /*设置用户ID，客户端切换用户时可以显示相应用户的下载任务*/
            manager.changeUser(Constants.USER_ROOT);
            /*断点续传需要服务器的支持，设置该项时要先确保服务器支持断点续传功能*/
            manager.setSupportBreakpoint(true);
            adapter = new DownloadListAdapter(DownloadDemoActivity.this, manager);
            listView.setAdapter(adapter);
        }
    };
```
（2）初始化管理页面
```
// 获取需要管理的数据，参数containsDownloaded为是否要包含已经下载完成的数据
listData = downLoadManager.getAllTask(containsDownloaded);
// 为下载管理任务设置监听器
downLoadManager.setAllTaskListener(new DownloadManagerListener());
```
（3）重写监听器
```
    private class DownloadManagerListener implements DownLoadListener {

        @Override
        public void onStart(SQLDownLoadInfo sqlDownLoadInfo) {

        }

        @Override
        public void onProgress(SQLDownLoadInfo sqlDownLoadInfo, boolean isSupportBreakpoint) {
            //根据监听到的信息查找列表相对应的任务，更新相应任务的进度
            for (TaskInfo taskInfo : listData) {
                if (taskInfo.getTaskID().equals(sqlDownLoadInfo.getTaskID())) {
                    taskInfo.setDownFileSize(sqlDownLoadInfo.getDownloadSize());
                    taskInfo.setFileSize(sqlDownLoadInfo.getFileSize());
                    DownloadListAdapter.this.notifyDataSetChanged();
                    break;
                }
            }
        }

        @Override
        public void onStop(SQLDownLoadInfo sqlDownLoadInfo, boolean isSupportBreakpoint) {

        }

        @Override
        public void onSuccess(SQLDownLoadInfo sqlDownLoadInfo) {
            //根据监听到的信息查找列表相对应的任务，删除对应的任务
            for (TaskInfo taskInfo : listData) {
                if (taskInfo.getTaskID().equals(sqlDownLoadInfo.getTaskID())) {
                    if (!containsDownloaded) {
                        listData.remove(taskInfo);
                    }
                    DownloadListAdapter.this.notifyDataSetChanged();
                    break;
                }
            }
        }

        @Override
        public void onError(SQLDownLoadInfo sqlDownLoadInfo) {
            //根据监听到的信息查找列表相对应的任务，停止该任务
            for (TaskInfo taskInfo : listData) {
                if (taskInfo.getTaskID().equals(sqlDownLoadInfo.getTaskID())) {
                    taskInfo.setOnDownloading(false);
                    DownloadListAdapter.this.notifyDataSetChanged();
                    break;
                }
            }
        }
    }
```
（4）继续下载和停止下载
```
// 继续下载
listData.get(position).setOnDownloading(true);
downLoadManager.startTask(listData.get(position).getTaskID());
// do something to refresh the views.

//停止下载
listData.get(position).setOnDownloading(false);
downLoadManager.stopTask(listData.get(position).getTaskID());
// do something to refresh the views.
```
