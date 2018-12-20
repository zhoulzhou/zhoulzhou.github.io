---
title: Android 转屏分析
date: 2018-12-20 17:13:26
tags:
- 转屏
categories:
- Android系统源码
---
转屏的代码逻辑主要在ＷＭＳ（窗口管理）这一块。

整个转屏过程可以分为以下几个步骤：

１计算出屏幕的下一个rotation;

２window 重新布局计算

一开始当然是Sensor 从驱动传上来值，然后上层来判断要不要转屏，　这部分代码位WindowOrientationListener，　WindowOrientationListener是一个抽象类。在他的内部有一个SensorEventListenerImpl　实现了SensorEventListener　用于监听Sensor

```
 private WindowOrientationListener(Context context, Handler handler, int rate) {  
       mHandler = handler;  
       mSensorManager = (SensorManager)context.getSystemService(Context.SENSOR_SERVICE);  
       mRate = rate;  
       mSensor = mSensorManager.getDefaultSensor(USE_GRAVITY_SENSOR  
               ? Sensor.TYPE_GRAVITY : Sensor.TYPE_ACCELEROMETER);  
       if (mSensor != null) {  
           // Create listener only if sensors do exist  
           mSensorEventListener = new SensorEventListenerImpl();  
       }  
   }  
```

其中USE_GRAVITY_SENSOR　等于false ，也就是说目前我们手机里转屏用的Sensor 是加速度传感器，并非重力感应器
mSensorEventListener　的 registerListener 和　unregisterListener　分别位于WindowOrientationListener的enable()  和　disable ()方法中。

```
   1. public void enable() {  
   2.            synchronized (mLock) {  
   3.            if (mSensor == null) {  
   4.                Log.w(TAG, "Cannot detect sensors. Not enabled");  
   5.                return;  
   6.            }  
   7.            if (mEnabled == false) {  
   8.                if (LOG) {  
   9.                    Log.d(TAG, "WindowOrientationListener enabled");  
  10.                }  
  11.                mSensorEventListener.resetLocked();  
  12.                mSensorManager.registerListener(mSensorEventListener, mSensor, mRate, mHandler);  
  13.                mEnabled = true;  
  14.            }  
  15.        }  
  16.    }  
  17.   
  18.    /** 
  19.     * Disables the WindowOrientationListener. 
  20.     */  
  21.    public void disable() {  
  22.   
  23.        synchronized (mLock) {  
  24.            if (mSensor == null) {  
  25.                Log.w(TAG, "Cannot detect sensors. Invalid disable");  
  26.                return;  
  27.            }  
  28.            if (mEnabled == true) {  
  29.                if (LOG) {  
  30.                    Log.d(TAG, "WindowOrientationListener disabled");  
  31.                }  
  32.                mSensorManager.unregisterListener(mSensorEventListener);  
  33.                mEnabled = false;  
  34.            }  
  35.        }  
  36.    }  
```

WindowOrientationListener 的enable() 和disbale() 的调用位于PhoneWindowManager的updateOrientationListenerLp　方法中。该方法会在screen on, screen off 和Settings数据库发生变化的时候调用。
　　下面我们回到SensorEventListenerImpl，我们知道当sensor 值发生变化时会调用onSensorChanged。在这里经过一些算法比较后。确定是不是要转屏。这部分代码我就没仔细看了，有兴趣可以自己研究。总之，再判断要转屏之后。会调用WindowOrientationListener的onProposedRotationChanged方法。

这个方法是一个抽象的方法。他的实现位于MyOrientationListener中，　MyOrientationListener是PhoneWindowManager 里面的内部类。onProposedRotationChanged就是简单调用updateRotation，　然后调用WindowManagerService的updateRotation方法，　再到updateRotationUnchecked方法

我们来看updateRotationUnchecked方法

```
   1. public void updateRotationUnchecked(boolean alwaysSendConfiguration, boolean forceRelayout) {  
   2.         if(DEBUG_ORIENTATION) Slog.v(TAG, "updateRotationUnchecked("  
   3.                    + "alwaysSendConfiguration=" + alwaysSendConfiguration + ")");  
   4.   
   5.         long origId = Binder.clearCallingIdentity();  
   6.         boolean changed;  
   7.         synchronized(mWindowMap) {  
   8.             changed = updateRotationUncheckedLocked(false);  
   9.             if (!changed || forceRelayout) {  
  10.                 getDefaultDisplayContentLocked().layoutNeeded = true;  
  11.                 performLayoutAndPlaceSurfacesLocked();  
  12.             }  
  13.         }  
  14.   
  15.         if (changed || alwaysSendConfiguration) {  
  16.             sendNewConfiguration();  
  17.         }  
  18.   
  19.         Binder.restoreCallingIdentity(origId);  
  20.     }
```

 我们可以看到updateRotationUnchecked主要做了３件事：

　　　１。更新rotation 

　　　 2。通知到ＡＭＳ　Configuration　changed

        首先，我们来分析updateRotationUncheckedLocked，　
```
 int rotation = mPolicy.rotationForOrientationLw(mForcedAppOrientation, mRotation);　
 ```
 
 updateRotationUncheckedLocked第一个很重要的方法，就是mPolicy.rotationForOrientationLw(mForcedAppOrientation, mRotation);　 mForcedAppOrientation一般情况是由当前Resume状态的Activity决定的。具体实现是在PhoneWindowManager 中，作用就是根据mForcedAppOrientation和 rotation计算出新的rotation

 ```
         mRotation = rotation;
        mAltOrientation = altOrientation;
        mPolicy.setRotationLw(mRotation);
 
        mWindowsFreezingScreen = true;
        mH.removeMessages(H.WINDOW_FREEZE_TIMEOUT);
        mH.sendEmptyMessageDelayed(H.WINDOW_FREEZE_TIMEOUT, WINDOW_FREEZE_TIMEOUT_DURATION);
        mWaitingForConfig = true;
        final DisplayContent displayContent = getDefaultDisplayContentLocked();
        displayContent.layoutNeeded = true;
        final int[] anim = new int[2];
        if (displayContent.isDimming()) {
            anim[0] = anim[1] = 0;
        } else {
            mPolicy.selectRotationAnimationLw(anim);
        }
        startFreezingDisplayLocked(inTransaction, anim[0], anim[1]);
        // startFreezingDisplayLocked can reset the ScreenRotationAnimation.
        screenRotationAnimation =
                mAnimator.getScreenRotationAnimationLocked(Display.DEFAULT_DISPLAY);
```

 上面的代码主要就是设置rotation，  开始freezing screen 。  设置 freezing_time_out ， 获取转屏动画等操作。
 我们再往下看：
 
 ```
         final WindowList windows = displayContent.getWindowList();
        for (int i = windows.size() - 1; i >= 0; i--) {
            WindowState w = windows.get(i);
            if (w.mHasSurface) {
                if (DEBUG_ORIENTATION) Slog.v(TAG, "Set mOrientationChanging of " + w);
                w.mOrientationChanging = true;
                mInnerFields.mOrientationChangeComplete = false;
            }
            w.mLastFreezeDuration = 0;
        }
```

    这里设置了两个重要的属性：
	
```
 w.mOrientationChanging = true;
 mInnerFields.mOrientationChangeComplete = false;
```

注意这两个属性和转屏结束有关。

接着  如果changed true 表示，rotation 要发生变化。  则调用sendNewConfiguration  。其作用就是调用AMS 的  mActivityManager.updateConfiguration(null);

```
   void sendNewConfiguration() {
        try {
            mActivityManager.updateConfiguration(null);
        } catch (RemoteException e) {
        }
    }
```

AMS 的 updateConfiguration 方法 首先会调用 ＷＭＳ的　computeNewConfiguration　来计算出新的config.  其最终实现是在WMS的computeScreenConfigurationLocked中。 有兴趣可以具体分析。

接着ＡＭＳ会根据得到的新的config 调用　updateConfigurationLocked(values, null, false, false);       这个方法是系统更新config 的方法。任何config　changed都会调用到。做过theme 的应该都了解。　updateConfigurationLocked其主要作用就是更新Ａｐｐ的configuration  发送CONFIGURATION_CHANGED 广播 。在这个方法最后会调用到ＷＭＳ　

```
 　　if (values != null && mWindowManager != null) {
            mWindowManager.setNewConfiguration(mConfiguration);
     }
```

```
    @Override
    public void setNewConfiguration(Configuration config) {
        if (!checkCallingPermission(android.Manifest.permission.MANAGE_APP_TOKENS,
                "setNewConfiguration()")) {
            throw new SecurityException("Requires MANAGE_APP_TOKENS permission");
        }
 
        synchronized(mWindowMap) {
        	mCurConfiguration = new Configuration(config);
            if (mWaitingForConfig) {
                mWaitingForConfig = false;
                mLastFinishedFreezeSource = "new-config";
            }
            performLayoutAndPlaceSurfacesLocked();
        }
    }
```

会更新ＷＭＳ　这边mCurConfiguration。　然后调用　performLayoutAndPlaceSurfacesLocked　对窗口重新布局绘制

在performLayoutAndPlaceSurfacesLocked 执行过程中会对每个可见的window 调用updateResizingWindows(w); 我们来看updateResizingWindows

```
 private void updateResizingWindows(final WindowState w) {
        final WindowStateAnimator winAnimator = w.mWinAnimator;
        if (w.mHasSurface && w.mLayoutSeq == mLayoutSeq) {
            w.setInsetsChanged();
            boolean configChanged = w.isConfigChanged();
            w.mLastFrame.set(w.mFrame);
            if (w.mContentInsetsChanged
                    || w.mVisibleInsetsChanged
                    || winAnimator.mSurfaceResized
                    || configChanged) {
                           
                w.mLastOverscanInsets.set(w.mOverscanInsets);
                w.mLastContentInsets.set(w.mContentInsets);
                w.mLastVisibleInsets.set(w.mVisibleInsets);
                makeWindowFreezingScreenIfNeededLocked(w);
                // If the orientation is changing, then we need to
                // hold off on unfreezing the display until this
                // window has been redrawn; to do that, we need
                // to go through the process of getting informed
                // by the application when it has finished drawing.
                if (w.mOrientationChanging) {
                    if (DEBUG_SURFACE_TRACE || DEBUG_ANIM || DEBUG_ORIENTATION) Slog.v(TAG,
                            "Orientation start waiting for draw mDrawState=DRAW_PENDING in "
                            + w + ", surface " + winAnimator.mSurfaceControl);
                    winAnimator.mDrawState = WindowStateAnimator.DRAW_PENDING;
                    if (w.mAppToken != null) {
                        w.mAppToken.allDrawn = false;
                        w.mAppToken.deferClearAllDrawn = false;
                    }
                }
                if (!mResizingWindows.contains(w)) {
                    if (DEBUG_RESIZE || DEBUG_ORIENTATION) Slog.v(TAG,
                            "Resizing window " + w + " to " + winAnimator.mSurfaceW
                            + "x" + winAnimator.mSurfaceH);
                    mResizingWindows.add(w);
                }
            } else if (w.mOrientationChanging) {
                if (w.isDrawnLw()) {
                    if (DEBUG_ORIENTATION) Slog.v(TAG,
                            "Orientation not waiting for draw in "
                            + w + ", surface " + winAnimator.mSurfaceControl);
                    w.mOrientationChanging = false;
                    w.mLastFreezeDuration = (int)(SystemClock.elapsedRealtime()
                            - mDisplayFreezeTime);
                }
            }
        }
    }
	
```

由于WMS的 mConfiguration 已经改变了。而windowstate.isConfigChaned 会返回true

```
    boolean isConfigChanged() {
        boolean configChanged = mConfiguration != mService.mCurConfiguration
                && (mConfiguration == null
                        || (mConfiguration.diff(mService.mCurConfiguration) != 0));
 
        if (mAttrs.type == TYPE_KEYGUARD) {
            // Retain configuration changed status until resetConfiguration called.
            mConfigHasChanged |= configChanged;
            configChanged = mConfigHasChanged;
        }
 
        return configChanged;
    }
```

  可以看到当转屏时 configChanged 发生了变化。 需要对window 重新布局和绘制。并且会加入   mResizingWindows.add(w);
  
```
  if (w.mContentInsetsChanged
                    || w.mVisibleInsetsChanged
                    || winAnimator.mSurfaceResized
                    || configChanged) 
```

如果都没有发生，且 w.mOrientationChanging 为true 且 w.isDrawnLw() ，已经绘制完成，则会将w.mOrientationChanging 设为false

performLayoutAndPlaceSurfacesLocked 接下来会对 mResizingWindows 里面的所有窗口调用 resize 处理。 最终会调用到应用端，让应用重新布局绘制。

performLayoutAndPlaceSurfacesLocked 的最后会调用scheduleAnimationLocked 来播放动画  动画的循环执行是在WindowAnimator animateLocked 方法中。 这个方法在每次开始的时候。都会将

```
 mBulkUpdateParams = SET_ORIENTATION_CHANGE_COMPLETE;
```

```
      final WindowList windows = mService.getWindowListLocked(displayId);
                final int N = windows.size();
                for (int j = 0; j < N; j++) {
                    windows.get(j).mWinAnimator.prepareSurfaceLocked(true);
      }
```