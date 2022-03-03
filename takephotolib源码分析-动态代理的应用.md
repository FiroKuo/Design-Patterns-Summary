### 前言
上个版本修复了Android端图片选择框架`saic-takephoto`中的几个问题，其中有一个bug涉及到权限问题。H5通过桥接方法`chooseImage`拉起原生相册，进而调用`saic-takephoto`库完成相册选择或拍照。client端并没有主动发起权限校验，那肯定就是lib中做的了！跟随**方法调用**深入到lib中，却完全不见权限申请的影子！这使我顿时来了兴趣，随即阅读了下源代码，发现lib中的权限申请是通过`动态代理`来做的，顿时拨云见日。于是决定分享一下`saic-takephoto`的源代码，*算是对动态代理设计模式的一个补充*。
### 1. 入口ChoosePicActivity.launchChoosePic
```
ChoosePicActivity.launchChoosePic(
            context,
            count,//图片张数
            isCompressed,//是否压缩
            compressMaxSize,//压缩后最大尺寸
            isAlbum,//相册or拍照
            object : ChoosePicCallBack {
                override fun onChoosePic(result: JSChooseImageBean) {
                    function?.onCallBack(Gson().toJson(result))
                }

                override fun onFail(msg: String) {
                    function?.onCallBack(Gson().toJson(JSChooseImageBean().apply {
                        this.errCode = -1
                        this.images = null
                        }))
                    }
                })
```
`ChoosePicActivity`继承自`TakePhotoFragmentActivity`,后者是lib中的一个**模板**类，它的签名如下：
```
public class TakePhotoFragmentActivity extends FragmentActivity implements TakeResultListener, InvokeListener

//模板接口 图片选取结果最终会回调到ChoosePicActivity中处理
public interface TakeResultListener {
        void onHandleStart();

        void takeSuccess(TResult var1);

        void takeFail(TResult var1, String var2);

        void takeCancel();
}

//动态代理回调接口 当通过动态代理拦截方法调用的时候 会回调给impl
public interface InvokeListener {
    TPermissionType invoke(InvokeParam var1);
}
```
`TakePhotoFragmentActivity`中做的最重要的事情是初始化`TakePhotoImpl`对象：
```
//动态代理的核心
public TakePhoto getTakePhoto() {
        if (this.takePhoto == null) {
            this.takePhoto = (TakePhoto)TakePhotoInvocationHandler.of(this).bind(new TakePhotoImpl(this, this));
        }

        return this.takePhoto;
}
```
`TakePhotoImpl`实现了接口`TakePhoto`，它统一提供了对外的功能接口，如图片选取、拍照、裁剪等等：
```
public interface TakePhoto {
    void onPickMultiple(int var1);

    void onPickMultipleWithCrop(int var1, CropOptions var2);

    void onPickFromDocuments();

    void onPickFromDocumentsWithCrop(Uri var1, CropOptions var2);

    void onPickFromGallery();

    void onPickFromGalleryWithCrop(Uri var1, CropOptions var2);

    void onPickFromCapture(Uri var1);

    void onPickFromCaptureWithCrop(Uri var1, CropOptions var2);

    void onCrop(Uri var1, Uri var2, CropOptions var3) throws TException;

    void onCrop(MultipleCrop var1, CropOptions var2) throws TException;

    void permissionNotify(TPermissionType var1);

    void onEnableCompress(CompressConfig var1, boolean var2);

    void setTakePhotoOptions(TakePhotoOptions var1);

    void onCreate(Bundle var1);

    void onSaveInstanceState(Bundle var1);

    void onDestroy();

    void onActivityResult(int var1, int var2, Intent var3);
}
```
`ChoosePicActivity`的`onCreate`方法中处理了状态栏透明，设置了透明和全屏，最后调用了`initAndStart`，这个方法中配置了相册的属性、压缩参数、拍照后文件存放的位置Uri，随后调用`startChoose`发起真正的功能调用：
```
private fun startChoose() {
        if (isAlbum) {
            takePhoto.onPickMultiple(count)
        } else {
            takePhoto.onPickFromCapture(imageUri)
        }
    }
```
### 2. takePhoto.onPickMultiple(count)
这里是真正从相册中选择图片的入口：
```
public void onPickMultiple(int limit) {
        if (!TPermissionType.WAIT.equals(this.permissionType)) {
            TUtils.startActivityForResult(this.contextWrap, new TIntentWap(IntentUtils.getPickMultipleIntent(this.contextWrap, limit), 1008));
        }
}
```
`TPermissionType`是一个枚举类，对应着**权限请求状态**，如下：
```
public static enum TPermissionType {
        GRANTED("已授权"),
        DENIED("未授权"),
        WAIT("等待授权"),
        NOT_NEED("无需授权"),
        ONLY_CAMERA_DENIED("没有拍照权限"),
        ONLY_STORAGE_DENIED("没有读写SD卡权限");

        String stringValue;

        private TPermissionType(String stringValue) {
            this.stringValue = stringValue;
        }

        public String stringValue() {
            return this.stringValue;
        }
}
```
通过`TPermission`表示需要请求的权限：
```
public static enum TPermission {
        STORAGE("android.permission.WRITE_EXTERNAL_STORAGE"),
        CAMERA("android.permission.CAMERA");

        String stringValue;

        private TPermission(String stringValue) {
            this.stringValue = stringValue;
        }

        public String stringValue() {
            return this.stringValue;
        }
    }
```
那么这里的含义就是，如果当前权限请求状态不是**等待授权**，就打开相册页面选取图片：
```
public static Intent getPickMultipleIntent(TContextWrap contextWrap, int limit) {
        Intent intent = new Intent(contextWrap.getActivity(), AlbumSelectActivity.class);
        intent.putExtra("limit", limit > 0 ? limit : 1);
        return intent;
}
```
`TContextWrap`类代表着上下文`Context`，是对`Activity`和`Fragment`的包装，之所以有`Fragment`的选项是因为`startActivityForResult`可以从`Fragment`中发起，此时会回到Fragment的`onActivityResult`方法中。
`TIntentWap`包装了`Intent`和`requestCode`。
### 3. AlbumSelectActivity & ImageSelectActivity
到这里其实来到了另一个lib：`saic-multipleimageselect`，这个lib提供了自定义相册/图片选取的功能和UI——里面有两个重要的类分别是相册选择`AlbumSelectActivity`和图片选取`ImageSelectActivity`。`AlbumSelectActivity`中最重要的方法是`loadAlbums`:
```
 private void loadAlbums() {
        this.abortLoading();
        AlbumSelectActivity.AlbumLoaderRunnable runnable = new AlbumSelectActivity.AlbumLoaderRunnable();
        this.thread = new Thread(runnable);
        this.thread.start();
    }
```
这个方法开启了一个子线程加载相册：
```
private class AlbumLoaderRunnable implements Runnable {
        private AlbumLoaderRunnable() {
        }

        public void run() {
            Process.setThreadPriority(10);
            Message message;
            if (AlbumSelectActivity.this.adapter == null) {
                message = AlbumSelectActivity.this.handler.obtainMessage();
                message.what = 2001;
                message.sendToTarget();
            }

            if (!Thread.interrupted()) {
                Cursor cursor = AlbumSelectActivity.this.getApplicationContext().getContentResolver().query(Media.EXTERNAL_CONTENT_URI, AlbumSelectActivity.this.projection, (String)null, (String[])null, "date_added");
                if (cursor == null) {
                    message = AlbumSelectActivity.this.handler.obtainMessage();
                    message.what = 2005;
                    message.sendToTarget();
                } else {
                    ArrayList<Album> temp = new ArrayList(cursor.getCount());
                    HashSet<String> albumSet = new HashSet();
                    if (cursor.moveToLast()) {
                        do {
                            if (Thread.interrupted()) {
                                return;
                            }

                            String album = cursor.getString(cursor.getColumnIndex(AlbumSelectActivity.this.projection[0]));
                            String image = cursor.getString(cursor.getColumnIndex(AlbumSelectActivity.this.projection[1]));
                            int index = image.lastIndexOf(".");
                            if (index != -1 && !".gif".equals(image.substring(index))) {
                                File file = new File(image);
                                if (file.exists() && !albumSet.contains(album)) {
                                    temp.add(new Album(album, image));
                                    albumSet.add(album);
                                }
                            }
                        } while(cursor.moveToPrevious());
                    }

                    cursor.close();
                    if (AlbumSelectActivity.this.albums == null) {
                        AlbumSelectActivity.this.albums = new ArrayList();
                    }

                    AlbumSelectActivity.this.albums.clear();
                    AlbumSelectActivity.this.albums.addAll(temp);
                    message = AlbumSelectActivity.this.handler.obtainMessage();
                    message.what = 2002;
                    message.sendToTarget();
                    Thread.interrupted();
                }
            }
        }
}
```
相册加载过程中出现的“事件”都是通过**消息机制**回调到`Handler`中处理，方便展示UI的变化。相册加载是通过**MediaStore** Api做的：
```
    Cursor cursor = AlbumSelectActivity.this.getApplicationContext().getContentResolver().query(Media.EXTERNAL_CONTENT_URI, AlbumSelectActivity.this.projection, (String)null, (String[])null, "date_added");

    private final String[] projection = new String[]{"bucket_display_name", "_data"};
```
这里的`Media.EXTERNAL_CONTENT_URI`代表着`content://media/external/images/media`。众所周知，**MediaStore就是用ContentResolver的方式操作数据库**，如果你有**Root权限**，可以查看/data/data/com.android.providers.media包下的文件，通常会有两个库：`external.db`和`internal.db`，这两个数据库就是Android中的**Meida文件系统**，导出并使用SQ GUI工具打开，对学习`MediaStore`大有脾益。比方说有了**images**表，你就明白了**Content Uri和File Path**的一种转换方式：
```
//Content Uri获取Path
//***代表 media_id
	String  myImageUrl = "content://media/external/images/media/***";
       	Uri uri = Uri.parse(myImageUrl); 
        String[] proj = { MediaStore.Images.Media.DATA };   
        Cursor actualimagecursor = this.ctx.managedQuery(uri,proj,null,null,null);  
        int actual_image_column_index = actualimagecursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);   
        actualimagecursor.moveToFirst();      
        String img_path = actualimagecursor.getString(actual_image_column_index);  
     　 File file = new File(img_path);
  　 　 Uri fileUri = Uri.fromFile(file);
```
这里通过迭代的方式查询数据，并放到子线程中，非常好，但不是最好，相册一般不是很多，即内存占用不会很大，试想如果要加载图片，并且一个相册里的图片非常多，就会有两个问题：一是响应时间变长；二是占用大量内存。如何解决这个问题？使用`CursorAdapter`，直接从数据库中读数据，滑到那里加载到那里，内存中占用的对象数也只有一屏+上下两行，而不必把数据全部取出来保存到内存中。相比之下前者看起来有点不太聪明的样子。
加载完毕后会发送消息，通知显示相册列表：
```
 case 2002:
    if (AlbumSelectActivity.this.adapter == null) {
        AlbumSelectActivity.this.adapter = new CustomAlbumSelectAdapter(AlbumSelectActivity.this.getApplicationContext(), AlbumSelectActivity.this.albums);
        AlbumSelectActivity.this.gridView.setAdapter(AlbumSelectActivity.this.adapter);
        AlbumSelectActivity.this.progressBar.setVisibility(4);
        AlbumSelectActivity.this.gridView.setVisibility(0);
        AlbumSelectActivity.this.orientationBasedUI(AlbumSelectActivity.this.getResources().getConfiguration().orientation);
    } else {
        AlbumSelectActivity.this.adapter.notifyDataSetChanged();
    }
    break;
```
点击相册进行跳转，到图片选择页`ImageSelectActivity`：
```
 this.gridView.setOnItemClickListener(new OnItemClickListener() {
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                Intent intent = new Intent(AlbumSelectActivity.this.getApplicationContext(), ImageSelectActivity.class);
                intent.putExtra("album_id", ((Album)AlbumSelectActivity.this.albums.get(position)).id);
                AlbumSelectActivity.this.startActivityForResult(intent, 2000);
            }
});
```
这里传入`album_id`，同样的套路加载图片列表，不再赘述。
### 4. 选取图片回传
选择完毕后点击确定按钮返回：
```
private void sendIntent() {
        Intent intent = new Intent();
        intent.putParcelableArrayListExtra("images", this.getSelected());
        this.setResult(-1, intent);
        this.finish();
}
```
进而返回到`AlbumSelectActivity`：
```
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == 2000 && resultCode == -1 && data != null) {
            this.setResult(-1, data);
            this.finish();
        }

}
```
返回到`TakePhotoFragmentActivity`:
```
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        this.getTakePhoto().onActivityResult(requestCode, resultCode, data);
        super.onActivityResult(requestCode, resultCode, data);
    }
```
委托给`TakePhotoImpl`处理：
```
case 1008:
            this.listener.onHandleStart();
            if (resultCode == -1 && data != null) {
                ArrayList<Image> images = data.getParcelableArrayListExtra("images");
                if (this.cropOptions != null) {
                    try {
                        this.onCrop(MultipleCrop.of(TUtils.convertImageToUri(this.contextWrap.getActivity(), images), this.contextWrap.getActivity(), this.fromType), this.cropOptions);
                    } catch (TException var6) {
                        this.cropContinue(false);
                        var6.printStackTrace();
                    }
                } else {
                    this.takeResult(TResult.of(TUtils.getTImagesWithImages(images, this.fromType)));
                }
            } else {
                this.listener.takeCancel();
            }
        }
```
如果需要裁切会走裁切的逻辑，否则调用`takeResult`，这个方法中判断是否需要压缩，压缩的方式有两种，尺寸压缩和质量压缩：
```
public void compress(String imagePath, CompressImageUtil.CompressListener listener) {
        if (this.config.isEnablePixelCompress()) {
            try {
                this.compressImageByPixel(imagePath, listener);
            } catch (FileNotFoundException var4) {
                listener.onCompressFailed(imagePath, String.format("图片压缩失败,%s", var4.toString()));
                var4.printStackTrace();
            }
        } else {
            this.compressImageByQuality(BitmapFactory.decodeFile(imagePath), imagePath, listener);
        }

}
```
压缩算法不应该放在lib中默认实现，这是设计的不太好的一点，比方说我想使用Luban压缩自己压缩图片，就只能避免使用压缩配置，放在图片选取成功的回调中处理了。压缩处理完毕后，调用`handleTakeCallback`方法：
```
  private void handleTakeCallBack(TResult result, String... message) {
        if (message.length > 0) {
            this.listener.takeFail(result, message[0]);
        } else if (this.multipleCrop != null && this.multipleCrop.hasFailed) {
            this.listener.takeFail(result, this.contextWrap.getActivity().getResources().getString(string.msg_crop_failed));
        } else if (this.compressConfig != null) {
            boolean hasFailed = false;
            Iterator var4 = result.getImages().iterator();

            label31: {
                TImage image;
                do {
                    if (!var4.hasNext()) {
                        break label31;
                    }

                    image = (TImage)var4.next();
                } while(image != null && image.isCompressed());

                hasFailed = true;
            }

            if (hasFailed) {
                this.listener.takeFail(result, this.contextWrap.getActivity().getString(string.msg_compress_failed));
            } else {
                this.listener.takeSuccess(result);
            }
        } else {
            this.listener.takeSuccess(result);
        }

        this.clearParams();
}
```
最后，检查各项是否执行成功了，回调`TakeResultListener`，也就是最开始的模板接口，整个流程结束。
### 5.哎，等等...权限申请呢？
走完整个流程，我们始终没有看到权限处理的影子，见了鬼了！这正是使我感兴趣的地方！用老婆千反田爱瑠的话说：**我很好奇！**。那么，我们一定是错过了什么，重新看一下`TakePhotoImpl`创建的地方：
```
public TakePhoto getTakePhoto() {
        if (this.takePhoto == null) {
            this.takePhoto = (TakePhoto)TakePhotoInvocationHandler.of(this).bind(new TakePhotoImpl(this, this));
        }
        return this.takePhoto;
}
```
我们发现了`TakePhotoInvocationHandler`类，咦，感觉这个名词好熟悉？！为什么这个方法中没有直接new一个`TakePhotoImpl`对象返回？赶紧点击去一探究竟：
```
public class TakePhotoInvocationHandler implements InvocationHandler {
    private TakePhoto delegate;
    private InvokeListener listener;

    public static TakePhotoInvocationHandler of(InvokeListener listener) {
        return new TakePhotoInvocationHandler(listener);
    }

    private TakePhotoInvocationHandler(InvokeListener listener) {
        this.listener = listener;
    }

    public Object bind(TakePhoto delegate) {
        this.delegate = delegate;
        return Proxy.newProxyInstance(delegate.getClass().getClassLoader(), delegate.getClass().getInterfaces(), this);
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TPermissionType type = this.listener.invoke(new InvokeParam(proxy, method, args));
        if (proxy instanceof TakePhoto && !TPermissionType.NOT_NEED.equals(type)) {
            ((TakePhoto)proxy).permissionNotify(type);
        }

        return method.invoke(this.delegate, args);
    }
}
```
哦，原来如此：`of`方法保存了`InvokeListener`，这里是`TakePhotoFragmentActivity`类，`bind`方法保存了`delegate`对象，即`TakePhoto`接口的实现类`TakePhotoImpl`，然后调用`Proxy.newProxyInstance`方法，对`TakePhoto`接口**进行了动态代理**，也就是说，`getTakePhoto`方法返回的对象**并不是**`TakePhotoImpl`，而是动态生成的对象。这样，调用`TakePhoto`的所有方法，都会走到`invoke`方法，是的，就像你猜的那样，**权限处理就是在这里拦截处理的**。
### 6. 权限处理过程
下面我们分析权限处理的过程，入口就是`TakePhotoInvocationHandler`的`invoke`方法：
```
 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TPermissionType type = this.listener.invoke(new InvokeParam(proxy, method, args));
        if (proxy instanceof TakePhoto && !TPermissionType.NOT_NEED.equals(type)) {
            ((TakePhoto)proxy).permissionNotify(type);
        }

        return method.invoke(this.delegate, args);
    }
```
`InvokeParams`类封装了动态代理的参数：
```
public class InvokeParam {
    private Object proxy;
    private Method method;
    private Object[] args;
    //...
}
```
上面我们提到，这里的listener传入的是`TakePhotoFragmentActivity`，我们看它的实现：
```
public TPermissionType invoke(InvokeParam invokeParam) {
        TPermissionType type = PermissionManager.checkPermission(TContextWrap.of(this), invokeParam.getMethod());
        if (TPermissionType.WAIT.equals(type)) {
            this.invokeParam = invokeParam;
        }

        return type;
}
```
进入到`PermissionManager`类中，首先查看它的类结构：
```
public static PermissionManager.TPermissionType checkPermission(@NonNull TContextWrap contextWrap, @NonNull Method method)
public static void requestPermission(@NonNull TContextWrap contextWrap, @NonNull String[] permissions)
public static PermissionManager.TPermissionType onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults)
public static void handlePermissionsResult(Activity activity, PermissionManager.TPermissionType type, InvokeParam invokeParam, TakeResultListener listener) 
```
这个类就是专门处理权限请求的，`checkPermission`方法如下：
```
 public static PermissionManager.TPermissionType checkPermission(@NonNull TContextWrap contextWrap, @NonNull Method method) {
        String methodName = method.getName();
        boolean contain = false;
        int i = 0;

        for(int j = methodNames.length; i < j; ++i) {
            if (TextUtils.equals(methodName, methodNames[i])) {
                contain = true;
                break;
            }
        }

        if (!contain) {
            return PermissionManager.TPermissionType.NOT_NEED;
        } else {
            boolean cameraGranted = true;
            boolean storageGranted = ContextCompat.checkSelfPermission(contextWrap.getActivity(), PermissionManager.TPermission.STORAGE.stringValue()) == 0;
            if (TextUtils.equals(methodName, "onPickFromCapture") || TextUtils.equals(methodName, "onPickFromCaptureWithCrop")) {
                cameraGranted = ContextCompat.checkSelfPermission(contextWrap.getActivity(), PermissionManager.TPermission.CAMERA.stringValue()) == 0;
            }

            boolean granted = storageGranted && cameraGranted;
            if (!granted) {
                ArrayList<String> permissions = new ArrayList();
                if (!storageGranted) {
                    permissions.add(PermissionManager.TPermission.STORAGE.stringValue());
                }

                if (!cameraGranted) {
                    permissions.add(PermissionManager.TPermission.CAMERA.stringValue());
                }

                requestPermission(contextWrap, (String[])permissions.toArray(new String[permissions.size()]));
            }

            return granted ? PermissionManager.TPermissionType.GRANTED : PermissionManager.TPermissionType.WAIT;
        }
}
```
`methodNames`是`TakePhoto`接口中需要申请权限的方法名集合:
```
    private static final String[] methodNames = new String[]{"onPickFromCapture", "onPickFromCaptureWithCrop", "onPickMultiple", "onPickMultipleWithCrop", "onPickFromDocuments", "onPickFromDocumentsWithCrop", "onPickFromGallery", "onPickFromGalleryWithCrop", "onCrop"};
```
首先判断当前方法名是否在这个集合内，调用不需要处理权限的方法直接返回`PermissionManager.TPermissionType.NOT_NEED`。然后调用`ContextCompat.checkSelfPermission`**检查相机权限和存储权限**的获取情况，发起权限请求，但此时方法会直接返回：
```
granted ? PermissionManager.TPermissionType.GRANTED : PermissionManager.TPermissionType.WAIT;
```
我们先看对返回值的处理，假设现在返回`PermissionManager.TPermissionType.WAIT`，`TakePhotoFragmentActivity`中：
```
 public TPermissionType invoke(InvokeParam invokeParam) {
        TPermissionType type = PermissionManager.checkPermission(TContextWrap.of(this), invokeParam.getMethod());
        if (TPermissionType.WAIT.equals(type)) {
            this.invokeParam = invokeParam;
        }

        return type;
    }
```
这里保存了`InvokeParams`，意思就是**记录了当前试图调用的方法**，以待后续权限请求成功后继续调用；直接将type返回到`InvocationHandler`中:
```
 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        TPermissionType type = this.listener.invoke(new InvokeParam(proxy, method, args));
        if (proxy instanceof TakePhoto && !TPermissionType.NOT_NEED.equals(type)) {
            ((TakePhoto)proxy).permissionNotify(type);
        }

        return method.invoke(this.delegate, args);
 }
```
首先调用了`permissionNotify`方法，这个方法的调用又会被动态代理，但由于不在`methodNames`列表中，所以不会被拦截，**直接调用delegate的permissionNotify**，更新`TPermissionType`，方便调用方法时判断：
```
 public void onPickMultiple(int limit) {
        if (!TPermissionType.WAIT.equals(this.permissionType)) {
            TUtils.startActivityForResult(this.contextWrap, new TIntentWap(IntentUtils.getPickMultipleIntent(this.contextWrap, limit), 1008));
        }
 }
```
然后调用`method.invoke(this.delegate, args)`，也就是上边的`onPickMultiple`方法，由于此时权限类型是WAIT，所以这里直接跳过。
别忘了，此时正在发起权限请求，权限请求的结果会回调到`TakePhotoFragmentActivity`中：
```
public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        TPermissionType type = PermissionManager.onRequestPermissionsResult(requestCode, permissions, grantResults);
        PermissionManager.handlePermissionsResult(this, type, this.invokeParam, this);
}

 //委托到PermissionManager中判断权限获取情况
 public static PermissionManager.TPermissionType onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        if (requestCode == 2000) {
            boolean cameraGranted = true;
            boolean storageGranted = true;
            int i = 0;

            for(int j = permissions.length; i < j; ++i) {
                if (grantResults[i] != 0) {
                    if (TextUtils.equals(PermissionManager.TPermission.STORAGE.stringValue(), permissions[i])) {
                        storageGranted = false;
                    } else if (TextUtils.equals(PermissionManager.TPermission.CAMERA.stringValue(), permissions[i])) {
                        cameraGranted = false;
                    }
                }
            }

            if (cameraGranted && storageGranted) {
                return PermissionManager.TPermissionType.GRANTED;
            }

            if (!cameraGranted && storageGranted) {
                return PermissionManager.TPermissionType.ONLY_CAMERA_DENIED;
            }

            if (!storageGranted && cameraGranted) {
                return PermissionManager.TPermissionType.ONLY_STORAGE_DENIED;
            }

            if (!storageGranted && !cameraGranted) {
                return PermissionManager.TPermissionType.DENIED;
            }
        }

        return PermissionManager.TPermissionType.WAIT;
}
```
拿到当前权限请求结果类型后，调用:
```
PermissionManager.handlePermissionsResult(this, type, this.invokeParam, this);
```
这里的`invokeParam`就是权限请求开始之前保存的，意图调用的方法：
```
public static void handlePermissionsResult(Activity activity, PermissionManager.TPermissionType type, InvokeParam invokeParam, TakeResultListener listener) {
        String tip = null;
        switch(type) {
        case DENIED:
            listener.takeFail((TResult)null, tip = activity.getResources().getString(string.tip_permission_camera_storage));
            break;
        case ONLY_CAMERA_DENIED:
            listener.takeFail((TResult)null, tip = activity.getResources().getString(string.tip_permission_camera));
            break;
        case ONLY_STORAGE_DENIED:
            listener.takeFail((TResult)null, tip = activity.getResources().getString(string.tip_permission_storage));
            break;
        case GRANTED:
            try {
                invokeParam.getMethod().invoke(invokeParam.getProxy(), invokeParam.getArgs());
            } catch (Exception var6) {
                listener.takeFail((TResult)null, tip = activity.getResources().getString(string.tip_permission_camera_storage));
            }
        }

        if (tip != null) {
            Toast.makeText(activity.getApplicationContext(), tip, 1).show();
        }

}
```
可以看到，只有权限请求类型为`GRANTED`的时候才会继续调用：
```
    invokeParam.getMethod().invoke(invokeParam.getProxy(), invokeParam.getArgs());

```
此时正常进入方法调用流程。
### 总结
有点啰嗦，其实主要还是想说一下动态代理这一部分的处理方式，结果把整个流程都说了一下。正是因为`TakePhoto`接口中大多数方法都要进行权限处理，所以这里可以使用动态代理的方式统一对方法调用进行拦截，处理权限问题，使得代码结构更为简洁干练，是个很好的实战案例。