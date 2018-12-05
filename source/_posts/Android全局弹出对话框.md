---
title: Android全局弹出对话框
date: 2018-12-05 16:19:50
tags:
- 对话框
- WindowManager
categories:
- ANDROID
---
项目中为了实现账号多设备登录的监听 一个账号在别的设备登录时在该设备上需要弹出对话框提示 故而用到全局对话框

#### 方案一
1、在开发中有时会用到全局弹出对话框但必须在manifest中申请权限：
><uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />

2、创建Dialog

```
AlertDialog.Builder builder=new AlertDialog.Builder(this);
  builder.setIcon(R.drawable.logo_mini);
  builder.setTitle("下线通知").setMessage("该账号在另一台Android设备上登录。")
  .setPositiveButton("重新登陆", new DialogInterface.OnClickListener() {

   @Override
   public void onClick(DialogInterface dialog, int which) {
    //do somthing
   }
  }).setNegativeButton("退出",new DialogInterface.OnClickListener() {

   @Override
   public void onClick(DialogInterface dialog, int which) {
    Intent i=new Intent(CoreService.this,LoginActivity.class);
    i.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    startActivity(i);
   }
  });
  alertDialog = builder.create();
  alertDialog.setCancelable(false);
  alertDialog.setCanceledOnTouchOutside(false);
  alertDialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
  alertDialog.show();
```

 注意要设置Dialog的Window类型为WindowManager.LayoutParams.TYPE_SYSTEM_ALERT。
 
#### 方案二
 采用方案一带来的问题：

用户在安装应用时会询问用户是否授权问题

同时在小米手机上默认是禁止系统弹出框的，应用中的系统弹出框将不能够弹出

那能不能不申请系统权限，又能弹出提示框提示用户呢？

这里妨转换思路，既然系统弹出框弹不出，我们就不采用系统弹出框，给他弹个Activity替代。但这个时候注意，要再Service是中或者ApplicationContext中startActivity时要添加额外标志ntent.FLAG_ACTIVITY_NEW_TASK：
```
Intent i=new Intent(this,ActDialog.class);
i.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
startActivity(i);
//代码中ActDialog其实是一个Activity把主题给它应用@android:style/Theme.Dialog
//把activity做的像一个Dialog的风格
```
这样解决了权限申请和小米手机全局Dialog默认无法展示的问题。

最后补充一下在方案一中也可以不申请权限就能弹出对话框，将弹出Window type 改为LayoutParams.TYPE_TOAST，但是这种类型的弹出框是不能接受事件处理的。

#### 使用WindowManager实现全局对话框
```
/**
  * 显示弹出框
  *
  * @param context
  */
 public static void showPopupWindow(final Context context, final DialogCallback callback) {
  // 获取WindowManager
  final WindowManager mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);


  final WindowManager.LayoutParams params = new WindowManager.LayoutParams();
  // 类型
  params.type = WindowManager.LayoutParams.TYPE_SYSTEM_ALERT;
  // 设置flag
  params.flags = WindowManager.LayoutParams.FLAG_ALT_FOCUSABLE_IM | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
  // 如果设置了WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE，弹出的View收不到Back键的事件
  // 不设置这个弹出框的透明遮罩显示为黑色
  params.format = PixelFormat.TRANSLUCENT;
  // FLAG_NOT_TOUCH_MODAL不阻塞事件传递到后面的窗口
  // 设置 FLAG_NOT_FOCUSABLE 悬浮窗口较小时，后面的应用图标由不可长按变为可长按
  // 不设置这个flag的话，home页的划屏会有问题
  params.width = WindowManager.LayoutParams.WRAP_CONTENT;
  params.height = WindowManager.LayoutParams.WRAP_CONTENT;
  params.gravity = Gravity.CENTER;
  TextView textView = new TextView(context);
  textView.setText("sfgsfdsfbsadfbasdfg");
  textView.setTextSize(100);

  final View mView = LayoutInflater.from(context).inflate(R.layout.item_dialog_exit, null);
  TextView tv_itemdialog_title = (TextView) mView.findViewById(R.id.tv_itemdialog_title);
  TextView tv_itemdialog_ok = (TextView) mView.findViewById(R.id.tv_itemdialog_ok);
  TextView tv_itemdialog_close = (TextView) mView.findViewById(R.id.tv_itemdialog_close);

  tv_itemdialog_ok.setText("重新登录");
  tv_itemdialog_close.setText("退出登录");
  tv_itemdialog_title.setText("该账户在其他设备登录,若不是您在操作,请及时修改密码以防泄露信息");
  tv_itemdialog_ok.setOnClickListener(new View.OnClickListener() {
   @Override
   public void onClick(View v) {
    // 隐藏弹窗
    mWindowManager.removeView(mView);
    callback.onPositive();
   }
  });

  tv_itemdialog_close.setOnClickListener(new View.OnClickListener() {
   @Override
   public void onClick(View v) {
    mWindowManager.removeView(mView);
    callback.onNegative();
   }
  });

  mWindowManager.addView(textView, params);
 }
```

我们这个的显示和当前的运行环境基本上是无关的，当前是什么activity或者是桌面，使用这个底层的结果给你的编程带来很大的灵活性，**但是要注意，显示出来就要销毁掉，这个是必须的，销毁其实就是一个remove。**

但是仅仅这么写会有个问题，就是view显示在最上层，但是后面的view无法获得焦点，当你touch的坐标超出上次view的范围时

```
mParams = new WindowManager.LayoutParams();   
mWm.updateViewLayout(view, mParams);
```

需要后面的view获得焦点时：

```
mParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;    
mWm.updateViewLayout(view, mParams);
```
