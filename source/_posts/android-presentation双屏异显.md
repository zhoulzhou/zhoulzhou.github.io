---
title: android presentation双屏异显
date: 2018-12-19 13:53:07
tags:
- presentation
- 双屏异显
categories:
- Android系统源码
---
要为辅助显示屏创建独特的内容:

1. 您需要扩展Presentation类，并实现onCreate()回调方法。在onCreate()中，调用setContentView()来指定您要在辅助显示屏上显示的UI。

2. 作为Dialog类的扩展，Presentation类提供了一个区域，在其中，您的应用可以在辅助显示屏上显示不同的UI。

获取显示Presentation的辅助显示屏：

1. 可以使用DisplayManager或者MediaRouter的API。其中，使用DisplayManagerAPI可以使您获得当前连接的所有显示屏的枚举，而MediaRouter则可以使您直接访问系统为Presentation设置的默认显示输出。

2. 可以给MediaRouter.getSelectedRoute()传一个ROUTE_TYPE_LIVE_VIDEO来获得演示的默认显示器。它将返回一个MediaRouter.RouteInfo对象，描述了系统为视频演示所选择的当前路由。如果MediaRouter.RouteInfo不空，调用getPresentationDisplay()即可获取当前连接的显示屏对象：Display。

3. 然后，您可以将这个Display对象传入Presentation的构造函数。调用Presentation.show()方法，演示就会出现在辅助显示屏上了。

为在运行时即时检测新接入的显示器，需要先创建一个MediaRouter.SimpleCallback的实例。在其中，您需要实现onRoutePresentationDisplayChanged()回调方法。当新的显示器连接时，系统会调用这个回调方法。

```
private final MediaRouter.SimpleCallback mMediaRouterCallback =
            new MediaRouter.SimpleCallback() {
        @Override
        public void onRouteSelected(MediaRouter router, int type, RouteInfo info) {
            Log.d(TAG, "onRouteSelected: type=" + type + ", info=" + info);
            updatePresentation();
        }

        @Override
        public void onRouteUnselected(MediaRouter router, int type, RouteInfo info) {
            Log.d(TAG, "onRouteUnselected: type=" + type + ", info=" + info);
            updatePresentation();
        }

        @Override
        public void onRoutePresentationDisplayChanged(MediaRouter router, RouteInfo info) {
            Log.d(TAG, "onRoutePresentationDisplayChanged: info=" + info);
            updatePresentation();
        }
    };
```

```
private void updatePresentation() {
        // Get the current route and its presentation display.
        MediaRouter.RouteInfo route = mMediaRouter.getSelectedRoute(
                MediaRouter.ROUTE_TYPE_LIVE_VIDEO);
        Display presentationDisplay = route != null ? route.getPresentationDisplay() : null;

        // Dismiss the current presentation if the display has changed.
        if (mPresentation != null && mPresentation.getDisplay() != presentationDisplay) {
            Log.i(TAG, "Dismissing presentation because the current route no longer "
                    + "has a presentation display.");
            mPresentation.dismiss();
            mPresentation = null;
        }

        // Show a new presentation if needed.
        if (mPresentation == null && presentationDisplay != null) {
            Log.i(TAG, "Showing presentation on display: " + presentationDisplay);
            mPresentation = new DemoPresentation(this, presentationDisplay);
            mPresentation.setOnDismissListener(mOnDismissListener);
            try {
                mPresentation.show();
            } catch (WindowManager.InvalidDisplayException ex) {
                Log.w(TAG, "Couldn't show presentation!  Display was removed in "
                        + "the meantime.", ex);
                mPresentation = null;
            }
        }

        // Update the contents playing in this activity.
        updateContents();
    }
```

```
private void updateContents() {
        // Show either the content in the main activity or the content in the presentation
        // along with some descriptive text about what is happening.
        if (mPresentation != null) {
            mInfoTextView.setText(getResources().getString(
                    R.string.presentation_with_media_router_now_playing_remotely,
                    mPresentation.getDisplay().getName()));
            mSurfaceView.setVisibility(View.INVISIBLE);
            mSurfaceView.onPause();
            if (mPaused) {
                mPresentation.getSurfaceView().onPause();
            } else {
                mPresentation.getSurfaceView().onResume();
            }
        } else {
            mInfoTextView.setText(getResources().getString(
                    R.string.presentation_with_media_router_now_playing_locally,
                    getWindowManager().getDefaultDisplay().getName()));
            mSurfaceView.setVisibility(View.VISIBLE);
            if (mPaused) {
                mSurfaceView.onPause();
            } else {
                mSurfaceView.onResume();
            }
        }
    }
```

为针对辅助显示进一步优化Presentation的UI，您需要为您的应用或activity指定标签为android:presentationTheme主题。

**请留意**
注意：public Presentation (Context outerContext, Display display) 这个初始化函数，这里的outerContext必须要activity，一个activity的context，虽然presentation会创建自己的context，但是它是在这个参数context之上的，而且这个activity跳转后presentation就消失了，如果说用getApplicationContext()就直接报错，也不行。

连接到用户移动设备的屏幕往往有较大的屏幕尺寸和不同的屏幕密度。由于屏幕特征可能不同，您需要为大型显示设备提供特定优化的资源。如果您需要从Presentation中获取额外的资源，调用getContext().getResources()来获取针对这种显示的资源对象。这样，您的应用就可以根据辅助显示器的尺寸和密度提供最合适的资源了。