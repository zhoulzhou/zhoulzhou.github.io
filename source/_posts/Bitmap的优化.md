---
title: Bitmap的优化
date: 2019-01-03 14:19:17
tags:
- bitmap
categories:
- 性能优化
---
#### 1. Bitmap内存模型
在Android 2.3.3（API10）之前，Bitmap的像素数据存放在Native内存，而Bitmap对象本身则存放在Dalvik Heap中。Native内存中的像素数据并不会以可预测的方式进行同步回收，有可能会导致内存升高甚至OOM。而在Android3.0之后，Bitmap的像素数据也被放在了Dalvik Heap中。

#### 2. Bitmap的内存回收
Android3.0之后，并没有强调Bitmap.recycle()；而是强调Bitmap的复用：

**2.2.1 Save a bitmap for later use**
使用LruCache对Bitmap进行缓存，当再次使用到这个Bitmap的时候直接获取，而不用重走编码流程。

**2.2.2 Use an existing bitmap**
Android3.0(API 11之后)引入了BitmapFactory.Options.inBitmap字段，设置此字段之后解码方法会尝试复用一张存在的Bitmap。这意味着Bitmap的内存被复用，避免了内存的回收及申请过程，显然性能表现更佳。不过，使用这个字段有几点限制：

声明可被复用的Bitmap必须设置inMutable为true；
Android4.4(API 19)之前只有格式为jpg、png，同等宽高（要求苛刻），inSampleSize为1的Bitmap才可以复用；
Android4.4(API 19)之前被复用的Bitmap的inPreferredConfig会覆盖待分配内存的Bitmap设置的inPreferredConfig；
Android4.4(API 19)之后被复用的Bitmap的内存必须大于需要申请内存的Bitmap的内存；
Android4.4(API 19)之前待加载Bitmap的Options.inSampleSize必须明确指定为1。

#### 3. Bitmap占有多少内存？
加载一张本地资源图片，那么它占用的内存 = width * height * nTargetDensity/inDensity * nTargetDensity/inDensity * 一个像素所占的内存。

#### 4 BitmapFactory.decodeFile()
（1）若使用 decodeResource() 加载本地图片，inDensity 为加载图片所在的文件夹代表的 dpi，inTargetDensity 为目标屏幕密度（or 图片真实像素密度？），
最终 bitmap.mDensity = targetDensity。
（2）若使用 decodeStream() 则不会先记录图片信息，得到bitmap 后，直接调用 setDensityFromOptions() 方法，所以最终 bitmap.mDensity = defaultDensity() = DENSITY_DEVICE。

#### 5. Bitmap如何复用？
1、使用LruCache和DiskLruCache做内存和磁盘缓存；

2、使用Bitmap复用，同时针对版本进行兼容。
图片宽1024、高594，进行Bitmap复用的实验；

```
BitmapFactory.Options options = new BitmapFactory.Options();
// 图片复用，这个属性必须设置；
options.inMutable = true;
// 手动设置缩放比例，使其取整数，方便计算、观察数据；
options.inDensity = 320;
options.inTargetDensity = 320;
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.resbitmap, options);
// 对象内存地址；
Log.i(TAG, "bitmap = " + bitmap);
Log.i(TAG, "bitmap：ByteCount = " + bitmap.getByteCount() + ":::bitmap：AllocationByteCount = " + bitmap.getAllocationByteCount());

// 使用inBitmap属性，这个属性必须设置；
options.inBitmap = bitmap;
options.inDensity = 320;
// 设置缩放宽高为原始宽高一半；
options.inTargetDensity = 160;
options.inMutable = true;
Bitmap bitmapReuse = BitmapFactory.decodeResource(getResources(), R.drawable.resbitmap_reuse, options);
// 复用对象的内存地址；
Log.i(TAG, "bitmapReuse = " + bitmapReuse);
Log.i(TAG, "bitmap：ByteCount = " + bitmap.getByteCount() + ":::bitmap：AllocationByteCount = " + bitmap.getAllocationByteCount());
Log.i(TAG, "bitmapReuse：ByteCount = " + bitmapReuse.getByteCount() + ":::bitmapReuse：AllocationByteCount = " + bitmapReuse.getAllocationByteCount());

输出：
I/lz: bitmap = android.graphics.Bitmap@35ac9dd4
I/lz: width:1024:::height:594
I/lz: bitmap：ByteCount = 2433024:::bitmap：AllocationByteCount = 2433024
I/lz: bitmapReuse = android.graphics.Bitmap@35ac9dd4 // 两个对象的内存地址一致
I/lz: width:512:::height:297
I/lz: bitmap：ByteCount = 608256:::bitmap：AllocationByteCount = 2433024
I/lz: bitmapReuse：ByteCount = 608256:::bitmapReuse：AllocationByteCount = 2433024 // ByteCount比AllocationByteCount小
```

#### 6. Bitmap如何压缩？
BitmapFactory.Options.inSampleSize
内存压缩：
解码图片时，设置BitmapFactory.Options类的inJustDecodeBounds属性为true，可以在Bitmap不被加载到内存的前提下，获取Bitmap的原始宽高。而设置BitmapFactory.Options的inSampleSize属性可以真实的压缩Bitmap占用的内存，加载更小内存的Bitmap。
设置inSampleSize之后，Bitmap的宽、高都会缩小inSampleSize倍。例如：一张宽高为2048x1536的图片，设置inSampleSize为4之后，实际加载到内存中的图片宽高是512x384。占有的内存就是0.75M而不是12M，足足节省了15倍。

备注：inSampleSize值的大小不是随便设、或者越大越好，需要根据实际情况来设置。
以下是设置inSampleSize值的一个示例：

```
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
        int reqWidth, int reqHeight) {
    // 设置inJustDecodeBounds属性为true，只获取Bitmap原始宽高，不分配内存；
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);
    // 计算inSampleSize值；
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
    // 真实加载Bitmap；
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}

public static int calculateInSampleSize(
            BitmapFactory.Options options, int reqWidth, int reqHeight) {
    // Raw height and width of image
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;
    if (height > reqHeight || width > reqWidth) {
        final int halfHeight = height / 2;
        final int halfWidth = width / 2;
        // 宽和高比需要的宽高大的前提下最大的inSampleSize
        while ((halfHeight / inSampleSize) >= reqHeight
                && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }
    return inSampleSize;
}

```

这样使用：mImageView.setImageBitmap(decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));

备注：
inSampleSize比1小的话会被当做1，任何inSampleSize的值会被取接近2的幂值。

#### BitmapFactory.Options
说明：用于解码Bitmap时的各种参数控制。
几个重要参数
inBitmap：在解析Bitmap时重用该Bitmap，但是必须相同大小的Bitmap & inMutable = true 才可重用；
inMutable ：配置Bitmap是否可更改，如每隔几个像素给Bmp添加一条直线；
inPreferredConfig：Config颜色位数，默认值为Bitmap.Config.ARGB_888；
inDither：是否抖动，默认false（Android Depracated）;
inPremultiplied：默认true，一般不改变其值。
inPurgeable：当存储像素内存空间 在系统内存不足时 是否可被回收（Android L Deprecated）;
inInputShareable：是否可以共享一个 InputStream （Android L Deprecated）;
inPreferQualityOverSpeed：为true时会优先保证 Bitmap 质量，其次是解码速度（Android N Deprecated）;
inTempStorage：解码时的临时空间，建议 16K；
inJustDecodeBounds：为true时仅返回 Bitmap 宽高等属性，返回bmp=null，为false时才返回占内存的 bmp；
inSampleSize：表示 Bitmap 的压缩比例，值必须 > 1 & 是2的幂次方。inSampleSize = 2 时，表示压缩宽高各1/2，最后返回原始图1/4大小的Bitmap；
inDensity：表示 Bitmap 像素密度；
inTargetDensity：表示 Bitmap 最终的像素密度；
inScreenDensity：表示当前屏幕的像素密度；
inScaled：默认为true，是否支持缩放，设置为true时，Bitmap将以 inTargetDensity 的值进行缩放；
outputWidth：返回的 Bitmap的宽；
outputHeight：返回的 Bitmap的高。

