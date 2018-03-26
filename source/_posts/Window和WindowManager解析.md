title: Window和WindowManager解析
date: 2015-10-28 15:35:51
tags: 
	- Android
	- Window机制
categories:
	- Android
	- 源码分析
---
&emsp;这几天阅读了《Android开发艺术探索》的关于Window和WindowManager的章节,特此写一片博文来整理和总结一下学到的知识.
&emsp;说到Window,大家都会想到所有的视图,包括Activity,Dialog,Toast,它们实际上都是附加在Window上的,Window是这些视图的管理者.今天我们就来具体将一下这些视图是如何附加在Window上的,Window有是如何管理这些视图的.
#### Window的属性和类别
&emsp;当我们通过WindowManager添加Window时,可以通过WindowManger.LayoutParams来确定Window的属性和类别.其中Flags参数标示Window的属性,我们列出几个比较常见的属性:

- `FLAG_NOT_FOCUSABLE` 这个参数表示Window不需要获取焦点,也不需要接收任何输入事件

- `FLAG_NOT_TOUCH_MODAL` 这个参数表示当前Window区域之外的点击事件传递给底层Window,区域之内的点击事件自己处理,一般默认开启

- `FLAG_SHOW_WHEN_LOCKED` 这个属性可以让Window显示在锁屏界面上
&emsp;Window不仅有属性,还有类型.Type参数表示Window的类型,分别为应用Window(activity对应的),子window(dialog对应的),和系统Window(Toast和系统通知栏).Window是分层的,每个window都有z-ordered,层级大的window会覆盖层级小的window,其大小关系为系统window>子window>应用window.所以系统window总会显示在最上边,但是使用系统window是需要声明相应的权限的.这一点需要注意.
#### WindowManager
&emsp;我们先来看一下WindowManager的接口,对其接口函数的了解有助于我们更好的理解Window的类别和属性.
&emsp;WindowManger实现了ViewManager这个接口,所提供的主要函数只有三个:
```
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
```

&emsp;而且通过阅读源码,我们会发现所有的操作都是交由WindowManagerGloalal来进行.之后的小节我会依次介绍.这一节先讲一下它的比较重要的成员变量.
 ```
// 存储所有window所对应的view
    private final ArrayList<View> mViews = new ArrayList<View>();
    // 存在window所对应的viewRootImpl
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    // 存储了所有window对应的布局参数
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    // 存储了那些正在被删除的view对象,调用了removeVIew,但是没有完成的
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

#### Window的添加过程
&emsp;这是WindowManagerGlobal的对应接口

```
	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) 
```
&emsp;创建ViewRootImpl,并将View添加到相应的列表中

```
	// 创建ViewRootImpl,然后将下述对象添加到列表中
	root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);//设置Params
        mViews.add(view);//window列表添加
        mRoots.add(root);//ViewRootImpl列表添加
        mParams.add(wparams);//布局参数列表添加
```

&emsp;通过ViewRootImpl来更新界面完成window的添加过程

```
	// 添加啦!!!!!!!!这是通过ViewRootImpl的setView来完成
	root.setView(view, wparams, panelParentView);
```

&emsp;在ViewRootImpl的setView函数中,会调用requestLayout来完成异步刷新,然后在requestLayout
中调用scheduleTraversals来进行view绘制.

```
	@Override
    	public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals(); // 实际View绘制的入口
        }
    }

```

&emsp;最后通过WindowSession来完成Window的添加过程,它是一个Binder对象,通过IPC调用来添加window.
&emsp;所以,Window的添加请求就交给WindowManagerService去处理,在其内部为每个应用保留一个单独的Session.

#### Window的删除过程


```	
    public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true); //先找到view的index
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```

&emsp;removeView先通过findViewLocked来查找待删除的View的索引,然后用removeViewLocked来做进一步删除. 
```

    private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index); //获得当前的view的viewRootImpl
        View view = root.getView();

        if (view != null) { //先让imm下降
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate); //die方法只是发送一个请求删除的消息之后就就返回
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);//加入dyingView
            }
        }
    }

```
&emsp;在WindowManager中提供了两种删除接口removeVIew()和removeViewImmediate(),它们分别表示异步和同步删除.而异步操作中会调用die函数,来发送一个MSG_DIE消息来异步删除,ViewRootImpl的Handler会调用doDie(),而如果是同步删除,那么就直接调用doDie(),然后在removeView函数中把View添加到mDyingViews中.

#### Window的更新
```
	public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
       	.....
	final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
        view.setLayoutParams(wparams);
        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);//这是主要的方法
        }
    }
```
&emsp;在setLayoutParams中会调用scheduleTraversals来重新绘制.
#### Window的创建过程
&emsp;不同类型的Window的创建过程不同,这里我只来讲一下Activity的Window的创建过程.在Window的启动过程中,会调用attach()函数来为其关联运行过程中所依赖的一系列上下文环境变量.

```
    mWindow = PolicyManager.makeNewWindow(this);
    mWindow.setCallback(this);

```

&emsp;Window对象是通过PolicyManager的makeNewWindow方法实现的,由于Activity实现了Window的Callback接口,因此当Window接收到外界的状态改变时就会回调Activity的对应方法.而我们去追寻Window的具体实现类,会发现它就是PhoneWindow,而Activity中最常用的setContentView方法的具体操作都是在PhoneWindow的相应方法中实现的.

&emsp;如果没有DecorView,那么就创建它.DecorView是一个FrameLayout,是Activity中的顶级View,一般包括标题栏和内容栏,而且内容栏的id为android.R.id.content,而DecorView的创建过程由installDecor完成,内部会通过generateDecor方法来直接创建DecorView

```
	if (mContentParent == null) { //如何没有DecorView,那么就新建一个
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
  	if (mDecor == null) {
            mDecor = generateDecor(); //直接new出一个DecorView返回
            ......
        }
        if (mContentParent == null) {
            //[window] 这一步也是很重要的.
            mContentParent = generateLayout(mDecor); 
            .......
            }
        .......

```

&emsp;而在generateLayout中
```
	//根据不同的style生成不同的decorview啊
        View in = mLayoutInflater.inflate(layoutResource, null);
        // 加入到deco中,所以应该是其第一个child
        decor.addView(in, new ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT));
        mContentRoot = (ViewGroup) in; 
        //给DecorView的第一个child是mContentView
        // 这是获得所谓的content
        ViewGroup contentParent = (ViewGroup)findViewById(ID_ANDROID_CONTENT);
```
  &emsp;将View添加到DecorView的mContentParent中,这步只需要一条语句就可,具体内部细节就不说啦.
```
	//第二步,将layout添加到mContentParent
        mLayoutInflater.inflate(layoutResID, mContentParent);
```

&emsp;然后就是回调Acitivity的onContentChanged方法通知Activity视图已经改变了.
```

	final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }

```
&emsp;经历了上述三个步骤,DecorView已经创建并初始化完毕,并且Activity的布局文件已经成功添加到DecorView的mContentParent中,但是DecorView并没有添加到WindowManager中去,也无法接收外界的输入,只有到Acitivity的makeVisible()被调用时,DecorView才真正完成了添加和显示过程.
```

    //DecorView正式添加并显示
    void makeVisible() {
       if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }

```
#### 总结
&emsp;这篇博文主要是读书笔记式的总结,本来想写一些自己的东西,但是研究的太浅,并且语言组织上还是有不足,以后还需要注意和继续努力啦.


