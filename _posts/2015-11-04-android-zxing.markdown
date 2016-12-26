---
layout: post
title:  "Android 扫描二维码的实现（简化zxing）"
date:   2015-11-04 18:00:00 +0800
categories: android
---

## zxing

哎呀呀，在`杭州2015 Hackthon`的现场，因为没有二维码签到功能，被吐槽low！这是我近期最丢脸的事啦~
于是回来就开始着手开发二维码相关的东西了。

一搜索`Google`和我们`SegmentFault`，发现在`Android`上，`Google`的`zxing`这个二维码的库比较受欢迎，好，那就是它了（我就是这么任性= =）

OK，看下`zxing`这个项目https://github.com/zxing/zxing
好像非常大啊，如何快速用起来呢？

答案就是

> 简化！

简化的话，带来的副作用就是适用性降低，比如在这个场景里面，我们不考虑横屏的情况，不考虑对摄像头进行过多的配置，不存在截图。

## 简化过程

我们看下这个项目有一个`android`目录，它里面其实就是`条码扫描器`这个app的开源代码，因为我们暂时不需要深究它是如何进行图像识别，如何帮我们从图像里解析出二维码的，所以我们就不研究`core`相关的内容（但是里面都是干货啊！图像识别的干货啊！）

![clipboard.png](/img/bVqHmc)
我们最主要就是参考这个里面的源码啦。

先把整个项目clone下来，然后把`android`这个源码导入到`Android Studio`里面去，经过小小的配置，就能搞定啦，直接运行我们就能看见这个App了。

先看下我们要做的工作是啥：

> 1. 对摄像头进行管理（为了兼容老设备，我们要用即将被Google舍弃的`android.hardware.camera`类）。
> 2. 对获取的一帧图片进行解析。
> 3. 对解析的结果进行处理。

如果不带着问题或者目的去看源码的话，会非常没头绪，所以我们一定要记得我们想要做什么。

OK，先看第一个问题，我们需要对摄像头进行管理。我们可以看到目录中有三个文件和摄像机有关

![clipboard.png](/img/bVqHm8)
就是这三个文件啦。

|类|用途|
|--|--|
|`CameraConfigurationManager`|这个类主要用于对摄像头进行一些配置，包括旋转角度、预览图尺寸等等|
|`CameraConfigurationUtils`|工具类，在`CameraConfigurationManager`调用，|
|`CameraManager`|控制摄像头的生命周期，获取预览尺寸，生成原始数据发送给解析器|

在这里，我把`CameraConfigurationUtils`拷贝过来，另外两个类合成到`CameraManager`里面，在初始化的时候，做好配置。

## 摄像头的管理（横屏改竖屏）

因为原先zxing给的demo是横屏的，这里要改成竖屏，就需要做几个配置。

### getFramingRectInPreview
在`getFramingRectInPreview`这个函数中，`cameraResolution`和`screenResolution`方向不对，所以
```java
rect.left = rect.left * cameraResolution.y / screenResolution.x;
rect.right = rect.right * cameraResolution.y / screenResolution.x;
rect.top = rect.top * cameraResolution.x / screenResolution.y;
rect.bottom = rect.bottom * cameraResolution.x / screenResolution.y;
```
这里需要把方向换一下。

### binary data
还需要更改的地方，就是在获取`帧预览`中的原始数据，需要进行一个旋转，因为`zxing`原先是对横向的图片一行一行读取的，如果我们给予纵向的数据，就必须旋转数据，或者更改读取算法。 这里更改数据可能会更加方便，
在`buildLuminanceSource`中，更改：
```
byte[] rotatedData = new byte[data.length];
for (int y = 0; y < height; y++) {
    for (int x = 0; x < width; x++)
        rotatedData[x * height + height - y - 1] = data[x + y * width];
}
int tmp = width;
width = height;
height = tmp;
```

最后记得把摄像头的预览旋转改成90°即可。
```java
 camera.setDisplayOrientation(90);
```

附上这部分代码（比较长）
```java
public class CameraManager {

    private static final int MIN_FRAME_WIDTH = 240;
    private static final int MIN_FRAME_HEIGHT = 240;

    // 这里修正扫描大小
    private static final int MAX_FRAME_WIDTH = 675; // = 5/8 * 1920
    private static final int MAX_FRAME_HEIGHT = 1200; // = 5/8 * 1080

    private Context mContext;
    private Point mScreenResolution;
    private Point mCameraResolution;
    private Point mBestPreviewSize;
    private Point mPreviewSizeOnScreen;

    private Rect mFramingRectInPreview;
    private Rect mFramingRect;

    private Camera mCamera;
    private PreviewCallback mPreviewCallback;
    private AutoFocusManager mAutoFocusManager;

    public CameraManager(Context context) {
        mContext = context;
        mPreviewCallback = new PreviewCallback(this);
    }

    public void openDriver(Camera camera, SurfaceHolder holder) {
        mCamera = camera;

        Camera.Parameters parameters = camera.getParameters();

        CameraConfigurationUtils.setBarcodeSceneMode(parameters);
        CameraConfigurationUtils.setFocus(parameters,
                true,
                true,
                false);

        Point theScreenResolution = new Point();
        Rect rect = holder.getSurfaceFrame();
        theScreenResolution.set(rect.height(), rect.width());
        mScreenResolution = theScreenResolution;
        mCameraResolution = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, mScreenResolution);
        mBestPreviewSize = CameraConfigurationUtils.findBestPreviewSizeValue(parameters, mScreenResolution);

        boolean isScreenPortrait = mScreenResolution.x < mScreenResolution.y;
        boolean isPreviewSizePortrait = mBestPreviewSize.x < mBestPreviewSize.y;

        if (isScreenPortrait == isPreviewSizePortrait) {
            mPreviewSizeOnScreen = mBestPreviewSize;
        } else {
            mPreviewSizeOnScreen = new Point(mBestPreviewSize.y, mBestPreviewSize.x);
        }

        parameters.setPreviewSize(mBestPreviewSize.x, mBestPreviewSize.y);
        camera.setParameters(parameters);
        camera.setDisplayOrientation(90);

        Camera.Parameters afterParameters = camera.getParameters();
        Camera.Size afterSize = afterParameters.getPreviewSize();
        if (afterSize != null && (mBestPreviewSize.x != afterSize.width || mBestPreviewSize.y != afterSize.height)) {
            mBestPreviewSize.x = afterSize.width;
            mBestPreviewSize.y = afterSize.height;
        }

    }

    public synchronized Rect getFramingRect() {
        if (mFramingRect == null) {
            Point screenResolution = mScreenResolution;
            if (screenResolution == null) {
                // Called early, before init even finished
                return null;
            }

            int width = findDesiredDimensionInRange(screenResolution.x, MIN_FRAME_WIDTH, MAX_FRAME_WIDTH);
            int height = findDesiredDimensionInRange(screenResolution.y, MIN_FRAME_HEIGHT, MAX_FRAME_HEIGHT);

            int leftOffset = (screenResolution.x - width) / 2;
            int topOffset = (screenResolution.y - height) / 2;
            mFramingRect = new Rect(leftOffset, topOffset, leftOffset + width, topOffset + height);
        }
        return mFramingRect;
    }

    public Point getCameraResolution() {
        return mCameraResolution;
    }

    public synchronized Rect getFramingRectInPreview() {
        if (mFramingRectInPreview == null) {
            Rect framingRect = getFramingRect();
            if (framingRect == null) {
                return null;
            }
            Rect rect = new Rect(framingRect);
            Point cameraResolution = mCameraResolution;
            Point screenResolution = mScreenResolution;
            if (cameraResolution == null || screenResolution == null) {
                // Called early, before init even finished
                return null;
            }
            rect.left = rect.left * cameraResolution.y / screenResolution.x;
            rect.right = rect.right * cameraResolution.y / screenResolution.x;
            rect.top = rect.top * cameraResolution.x / screenResolution.y;
            rect.bottom = rect.bottom * cameraResolution.x / screenResolution.y;
            mFramingRectInPreview = rect;
        }
        return mFramingRectInPreview;
    }

    private static int findDesiredDimensionInRange(int resolution, int hardMin, int hardMax) {
        int dim = 5 * resolution / 8; // Target 5/8 of each dimension
        if (dim < hardMin) {
            return hardMin;
        }
        if (dim > hardMax) {
            return hardMax;
        }
        return dim;
    }

    /**
     * A factory method to build the appropriate LuminanceSource object based on the format
     * of the preview buffers, as described by Camera.Parameters.
     *
     * @param data A preview frame.
     * @param width The width of the image.
     * @param height The height of the image.
     * @return A PlanarYUVLuminanceSource instance.
     */
    public PlanarYUVLuminanceSource buildLuminanceSource(byte[] data, int width, int height) {
        Rect rect = getFramingRectInPreview();
        if (rect == null) {
            return null;
        }

        byte[] rotatedData = new byte[data.length];
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++)
                rotatedData[x * height + height - y - 1] = data[x + y * width];
        }
        int tmp = width;
        width = height;
        height = tmp;

        // Go ahead and assume it's YUV rather than die.
        return new PlanarYUVLuminanceSource(rotatedData, width, height, rect.left, rect.top,
                rect.width(), rect.height(), false);
    }

    /**
     * A single preview frame will be returned to the handler supplied. The data will arrive as byte[]
     * in the message.obj field, with width and height encoded as message.arg1 and message.arg2,
     * respectively.
     *
     * @param handler The handler to send the message to.
     * @param message The what field of the message to be sent.
     */
    public synchronized void requestPreviewFrame(Handler handler, int message) {
        if (mCamera != null) {
            mPreviewCallback.setHandler(handler, message);
            mCamera.setOneShotPreviewCallback(mPreviewCallback);
        }
    }

    public synchronized void startPreview() {
        mCamera.startPreview();
        mAutoFocusManager = new AutoFocusManager(mContext, mCamera);
    }

    public synchronized void stopPreview() {
        if (mAutoFocusManager != null) {
            mAutoFocusManager.stop();
            mAutoFocusManager = null;
        }
        if (mCamera != null) {
            mCamera.stopPreview();
        }
        mPreviewCallback.setHandler(null, 0);

    }

    public synchronized void closeDriver() {
        mCamera.release();
        mCamera = null;
    }
}
```

## 获取图像数据
`Camera`提供这么一个函数：`setOneShotPreviewCallback`，看名字就知道，它是会在某个时间点，回调你给的接口，然后传入一个二进制的图像数组，让你去解析，这正是我们想要的东西，所以我们看下这个`PreviewCallback`，它只有一个函数，
```java
@Override
public void onPreviewFrame(byte[] data, Camera camera) {
}
```

很显然，第一个参数就是图像数组了，我们看看`zxing`里面是怎么处理这个数据的。

### 解析数据
```java
@Override
public void onPreviewFrame(byte[] data, Camera camera) {
    Point cameraResolution = mCameraManager.getCameraResolution();
    Handler thePreviewHandler = mPreviewHandler;
    if (cameraResolution != null && thePreviewHandler != null) {
        Message message = thePreviewHandler.obtainMessage(mPreviewMessage, cameraResolution.x,
                cameraResolution.y, data);
        message.sendToTarget();
        mPreviewHandler = null;
    } else {
    }
}
```

`zxing`是发送到另外一个handler，也就是说，这个解析数据的过程比较浪费时间，所以要开线程来解决这个事。
我们跟踪这个`previewHanlder`可以发现，它是一个`DecodeHandler`，其中有个重要的`decode`函数

```java
private void decode(byte[] data, int width, int height) {
    long start = System.currentTimeMillis();
    Result rawResult = null;
    PlanarYUVLuminanceSource source = activity.getCameraManager().buildLuminanceSource(data, width, height);
    if (source != null) {
        BinaryBitmap bitmap = new BinaryBitmap(new HybridBinarizer(source));
        try {
            rawResult = multiFormatReader.decodeWithState(bitmap);
        } catch (ReaderException re) {
            // continue
        } finally {
            multiFormatReader.reset();
        }
    }

    Handler handler = activity.getHandler();
    if (rawResult != null) {
        // Don't log the barcode contents for security.
        long end = System.currentTimeMillis();
        Log.d(TAG, "Found barcode in " + (end - start) + " ms");
        if (handler != null) {
            Message message = Message.obtain(handler, R.id.decode_succeeded, rawResult);
            Bundle bundle = new Bundle();
            bundleThumbnail(source, bundle);
            message.setData(bundle);
            message.sendToTarget();
        }
    } else {
        if (handler != null) {
            Message message = Message.obtain(handler, R.id.decode_failed);
            message.sendToTarget();
        }
    }
}
```

这里用到我们刚刚重写的`buildLuminanceSource`这个函数了，可以理解这个函数是对我们的原始数据做一个包装，来给解析器读取。

解析器的配置我们可以看看`DecodeThread`这个类（专门用于解析图像的线程）

```java
...
 decodeFormats = EnumSet.noneOf(BarcodeFormat.class);
decodeFormats.addAll(DecodeFormatManager.PRODUCT_FORMATS);
decodeFormats.addAll(DecodeFormatManager.INDUSTRIAL_FORMATS);
decodeFormats.addAll(DecodeFormatManager.QR_CODE_FORMATS);
decodeFormats.addAll(DecodeFormatManager.DATA_MATRIX_FORMATS);
mHints.put(DecodeHintType.POSSIBLE_FORMATS, decodeFormats);
```
这里我们可以为我们的扫描器加更多格式的支持（比如条形码、二维码、XX码）

`zxing`是使用List来管理这些解析器，然后每次读取数据，都经过这些解析器过滤一遍，如果解析成功就有结果，否则就没有结果。

最后我们跟踪`R.id.decode_success`找到`CaptureActivityHandler`
这里的`handleMessage`中有
```java
  case R.id.decode_succeeded:
        state = State.SUCCESS;
        Bundle bundle = message.getData();
        Bitmap barcode = null;
        float scaleFactor = 1.0f;
        if (bundle != null) {
            byte[] compressedBitmap = bundle.getByteArray(DecodeThread.BARCODE_BITMAP);
            if (compressedBitmap != null) {
                barcode = BitmapFactory.decodeByteArray(compressedBitmap, 0, compressedBitmap.length, null);
                // Mutable copy:
                barcode = barcode.copy(Bitmap.Config.ARGB_8888, true);
            }
            scaleFactor = bundle.getFloat(DecodeThread.BARCODE_SCALED_FACTOR);
        }
        activity.handleDecode((Result) message.obj, barcode, scaleFactor);
        break;
```

我们读取`Result`对象的text成员变量就是我们想要的二维码信息了！最后的工作当然是解析我们的二维码中的URI，然后进行后续的工作啦~

> *PS: 附带二维码扫描的SegmentFault for Android版本 即将上线！！*

> 欢迎关注我[Github](https://github.com/geminiwen) 以及 [weibo](http://weibo.com/coffeesherk/home?leftnav=1)、[@Gemini](/u/gemini)