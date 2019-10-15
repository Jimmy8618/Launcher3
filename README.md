# Launcher3    android 9.0
https://blog.csdn.net/yxdspirit/article/details/84634454
android P (9.0) Launcher3 去掉抽屉式,显示所有app
Launcher3 改成抽屉式需要更改的内容
1.显示所有app在桌面上
2.去掉上划展开应用列表
3.长按拖动图标去掉删除改成取消 卸载

显示所有app在桌面上
要显示所有app在桌面上,也就是workspace上,首先是要先把所有应用加到桌面,然后要考虑安装后的应用显示到桌面
先在src/com.android.launcher3.config.BaseFlags添加个boolean值

abstract class BaseFlags {

    BaseFlags() {}

    public static final boolean REMOVE_DRAWER = true;
    ...

1.显示所有应用
应用加载在src/com.android.launcher3.model.LoaderTask 里
在run里面添加方法加载所有应用

  	// second step
            TraceHelper.partitionSection(TAG, "step 2.1: loading all apps");
            loadAllApps();
            //add by jimmy
            if (FeatureFlags.REMOVE_DRAWER) {
                verifyApplications();
            }
            //end add by jimmy
            TraceHelper.partitionSection(TAG, "step 2.2: Binding all apps");
            verifyNotStopped();
            mResults.bindAllApps();


    //add by jimmy
    private void verifyApplications() {
        final Context context = mApp.getContext();
        ArrayList<Pair<ItemInfo, Object>> installQueue = new ArrayList<>();
        final List<UserHandle> profiles = mUserManager.getUserProfiles();
        for (UserHandle user : profiles) {
            final List<LauncherActivityInfo> apps = mLauncherApps.getActivityList(null, user);
            ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo> added = new ArrayList<InstallShortcutReceiver.PendingInstallShortcutInfo>();
            synchronized (this) {
                for (LauncherActivityInfo app : apps) {
                    InstallShortcutReceiver.PendingInstallShortcutInfo pendingInstallShortcutInfo = new InstallShortcutReceiver.PendingInstallShortcutInfo(app, context);
                    added.add(pendingInstallShortcutInfo);
                    installQueue.add(pendingInstallShortcutInfo.getItemInfo());
                }
            }
            if (!added.isEmpty()) {
                mApp.getModel().addAndBindAddedWorkspaceItems(installQueue);
            }
        }
    }
    //end add by jimmy

然后再修改src/com.android.launcher3.model.BaseModelUpdateTask,把return 给注释掉

   @Override
    public final void run() {
        if (!mModel.isModelLoaded()) {
            if (DEBUG_TASKS) {
                Log.d(TAG, "Ignoring model task since loader is pending=" + this);
            }
            // Loader has not yet run.
            if(!FeatureFlags.REMOVE_DRAWER){
                return;
            }
        }
        execute(mApp, mDataModel, mAllAppsList);
    }

这样 应用就可以显示到桌面上了

2.应用安装后加载到桌面
涉及到文件src/com.android.launcher3.model.PackageUpdatedTask

  final ArrayList<AppInfo> addedOrModified = new ArrayList<>();
        addedOrModified.addAll(appsList.added);
        //add by jimmy
        if(FeatureFlags.REMOVE_DRAWER){
            updateToWorkSpace(context, app, appsList);
        }
        //end add by jimmy
        appsList.added.clear();
        addedOrModified.addAll(appsList.modified);
        appsList.modified.clear();

这样第一步就完成了

去掉上划展开应用列表
这一个问题跟了好多个文件,先是从博文里提到的src/com.android.launcher3.allapps.AllAppsTransitionController
再到src/com.android.launcher3.LauncherStateManager/StateHandler
再到src/com.android.launcher3.LauncherStateManager
在AllAppsTransitionController里修改setState方法和setStateWithAnimation方法

 public void setState(LauncherState state) {
 //2018.12.12 更新 setState方法可以不用管,不然会导致有动画效果没有
  //      if(FeatureFlags.REMOVE_DRAWER)
  //          return;
        setProgress(state.getVerticalProgress(mLauncher));
        setAlphas(state, NO_ANIM_PROPERTY_SETTER);
        onProgressAnimationEnd();
    }
     @Override
    public void setStateWithAnimation(LauncherState toState,
            AnimatorSetBuilder builder, AnimationConfig config) {
        if(FeatureFlags.REMOVE_DRAWER)
            return;
        float targetProgress = toState.getVerticalProgress(mLauncher);
        if (Float.compare(mProgress, targetProgress) == 0) {
            setAlphas(toState, config.getPropertySetter(builder));
            // Fail fast
            onProgressAnimationEnd();
            return;
        }

这样可以实现上划去掉展开所有app,但是可以上划,上划后会隐藏掉所有的app


然后找到滑动的地方src/com.android.launcher3.touch.AbstractStateChangeTouchController
最后发现修改src_ui_overrides/com.android.launcher3.uioverrides.AllAppsSwipeController

   @Override
    protected boolean canInterceptTouch(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            mTouchDownEvent = ev;
        }
        if (mCurrentAnimation != null) {
            // If we are already animating from a previous state, we can intercept.
            return true;
        }
        if (AbstractFloatingView.getTopOpenView(mLauncher) != null) {
            return false;
        }
        if (!mLauncher.isInState(NORMAL) && !mLauncher.isInState(ALL_APPS)) {
            // Don't listen for the swipe gesture if we are already in some other state.
            return false;
        }
        if (mLauncher.isInState(ALL_APPS) && !mLauncher.getAppsView().shouldContainerScroll(ev)) {
            return false;
        }
        //add by jimmy
        if(FeatureFlags.REMOVE_DRAWER){
            return false;
        }
        //end add by jimmy
        return true;
    }

这样就上划不了了,但是这样会有什么其他影响暂时还没研究出来

长按拖动图标去掉删除改成取消 卸载
长按图标出现删除,卸载栏
src/com.android.launcher3.DropTargetBar
应用图标拖动的控制类
src/com.android.launcher3.dragndrop.DragController
删除,卸载类
src/com.android.launcher3.DeleteDropTarget
src/com.android.launcher3.SecondaryDropTarget
拖动应用图标或者文件夹的时候我们需要把删除改成取消,并且拖到取消后桌面不能移除图标
1.应用图标或者文件夹拖动删除改成取消
src/com.android.launcher3.DeleteDropTarget

  private void setTextBasedOnDragSource(ItemInfo item) {
        if (!TextUtils.isEmpty(mText)) {
            mText = getResources().getString(item.id != ItemInfo.NO_ID
                    ? R.string.remove_drop_target_label
                    : android.R.string.cancel);
            if(FeatureFlags.REMOVE_DRAWER){
                mText = getResources().getString(isCanDrop(item)
                        ? R.string.remove_drop_target_label
                        : android.R.string.cancel);
            }
            requestLayout();
        }
    }


mControlType 具体作用还不清楚,也一起改了

    /**
     * Set mControlType depending on the drag item.
     */
    private void setControlTypeBasedOnDragSource(ItemInfo item) {
        mControlType = item.id != ItemInfo.NO_ID ? ControlType.REMOVE_TARGET
                : ControlType.CANCEL_TARGET;
        if(FeatureFlags.REMOVE_DRAWER) {
            mControlType = isCanDrop(item) ? ControlType.REMOVE_TARGET
                    : ControlType.CANCEL_TARGET;
        }
    }

2.删除操作改为取消操作
DragController是应用图标拖动的控制类
onDriverDragEnd是拖动结束后调用的地方,drop 是执行拖动后操作

   @Override
    public void onDriverDragEnd(float x, float y) {
        DropTarget dropTarget;
        Runnable flingAnimation = mFlingToDeleteHelper.getFlingAnimation(mDragObject);
        if (flingAnimation != null) {
            dropTarget = mFlingToDeleteHelper.getDropTarget();
        } else {
            dropTarget = findDropTarget((int) x, (int) y, mCoordinatesTemp);
        }

        drop(dropTarget, flingAnimation);

        endDrag();
    }

在drop方法里对droptarget 判断 如果是拖动到删除 也就是DeleteDropTarget 就取消拖动

  private void drop(DropTarget dropTarget, Runnable flingAnimation) {
        final int[] coordinates = mCoordinatesTemp;
        mDragObject.x = coordinates[0];
        mDragObject.y = coordinates[1];

        // Move dragging to the final target.
        if (dropTarget != mLastDropTarget) {
            if (mLastDropTarget != null) {
                mLastDropTarget.onDragExit(mDragObject);
            }
            mLastDropTarget = dropTarget;
            if (dropTarget != null) {
                dropTarget.onDragEnter(mDragObject);
            }
        }

        mDragObject.dragComplete = true;
        if (mIsInPreDrag) {
            if (dropTarget != null) {
                dropTarget.onDragExit(mDragObject);
            }
            return;
        }


        // Drop onto the target.
        boolean accepted = false;
        if (dropTarget != null) {
            dropTarget.onDragExit(mDragObject);
            if (dropTarget.acceptDrop(mDragObject)) {
                if (flingAnimation != null) {
                    flingAnimation.run();
                } else {
                    dropTarget.onDrop(mDragObject, mOptions);
                }
                accepted = true;
                //add by jimmy //2018-11-30 16:05 修改
                if (FeatureFlags.REMOVE_DRAWER && dropTarget instanceof DeleteDropTarget &&
                        isNeedCancelDrag(mDragObject.dragInfo)) {
                    cancelDrag();
                }
                //end add by jimmy
            }
        }
        final View dropTargetAsView = dropTarget instanceof View ? (View) dropTarget : null;
        mLauncher.getUserEventDispatcher().logDragNDrop(mDragObject, dropTargetAsView);
        dispatchDropComplete(dropTargetAsView, accepted);
    }

    private boolean isNeedCancelDrag(ItemInfo item){
        return (item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION ||
                item.itemType == LauncherSettings.Favorites.ITEM_TYPE_FOLDER);
    }

这样只完成了一步,看 dropTarget.onDrop(mDragObject, mOptions); 这里执行了drop操作.
看DeleteDropTarget 的父类src/com.android.launcher3.ButtonDropTarget

  @Override
    public void onDrop(final DragObject d, final DragOptions options) {
        final DragLayer dragLayer = mLauncher.getDragLayer();
        final Rect from = new Rect();
        dragLayer.getViewRectRelativeToSelf(d.dragView, from);

        final Rect to = getIconRect(d);
        final float scale = (float) to.width() / from.width();
        mDropTargetBar.deferOnDragEnd();

        Runnable onAnimationEndRunnable = () -> {
            completeDrop(d);
            mDropTargetBar.onDragEnd();
            mLauncher.getStateManager().goToState(NORMAL);
        };

        dragLayer.animateView(d.dragView, from, to, scale, 1f, 1f, 0.1f, 0.1f,
                DRAG_VIEW_DROP_DURATION,
                Interpolators.DEACCEL_2, Interpolators.LINEAR, onAnimationEndRunnable,
                DragLayer.ANIMATION_END_DISAPPEAR, null);
    }

这里做了一个动画效果,动画完后就调用了onAnimationEndRunnable
completeDrop 就是做对应的操作了,然后看DeleteDropTarget 的completeDrop 方法

  @Override
    public void completeDrop(DragObject d) {
        ItemInfo item = d.dragInfo;
        if ((d.dragSource instanceof Workspace) || (d.dragSource instanceof Folder)) {
            onAccessibilityDrop(null, item);
        }
    }

    /**
     * Removes the item from the workspace. If the view is not null, it also removes the view.
     */
    @Override
    public void onAccessibilityDrop(View view, ItemInfo item) {
        // Remove the item from launcher and the db, we can ignore the containerInfo in this call
        // because we already remove the drag view from the folder (if the drag originated from
        // a folder) in Folder.beginDrag()

        // add by jimmy ,cancel to remove app icon and folder
        if(!FeatureFlags.REMOVE_DRAWER || isCanDrop(item)) {
            mLauncher.removeItem(view, item, true /* deleteFromDb */);
            mLauncher.getWorkspace().stripEmptyScreens();
            mLauncher.getDragLayer()
                    .announceForAccessibility(getContext().getString(R.string.item_removed));
        }
        // end add by jimmy
    }

    private boolean isCanDrop(ItemInfo item){
        return !(item.itemType == LauncherSettings.Favorites.ITEM_TYPE_APPLICATION ||
                item.itemType == LauncherSettings.Favorites.ITEM_TYPE_FOLDER);
    }

把删除应用图标操作去掉就好了,不让删除应用图标和文件夹

2018-12-03 12:00 更新
应用列表会有个提示上拉的跳动小动画,在src/com.android.launcher3.Launcher 的onResume里,把它去掉就好了

   @Override
    protected void onResume() {
        TraceHelper.beginSection("ON_RESUME");
        super.onResume();
        TraceHelper.partitionSection("ON_RESUME", "superCall");

        mHandler.removeCallbacks(mLogOnDelayedResume);
        Utilities.postAsyncCallback(mHandler, mLogOnDelayedResume);

        setOnResumeCallback(null);
        // Process any items that were added while Launcher was away.
        InstallShortcutReceiver.disableAndFlushInstallQueue(
                InstallShortcutReceiver.FLAG_ACTIVITY_PAUSED, this);

        // Refresh shortcuts if the permission changed.
        mModel.refreshShortcutsIfRequired();
        //Add by jimmy
        if(!FeatureFlags.REMOVE_DRAWER)
            DiscoveryBounce.showForHomeIfNeeded(this);
        //end add by jimmy
        if (mLauncherCallbacks != null) {
            mLauncherCallbacks.onResume();
        }
        UiFactory.onLauncherStateOrResumeChanged(this);

        TraceHelper.endSection("ON_RESUME");
    }

Launcher3 的Quickstep会调用DiscoveryBounce.showForOverviewIfNeeded 也会有all app跳动的动画
所以我把DiscoveryBounce里的showForHomeIfNeeded 和 showForOverviewIfNeeded 都给注释掉了

https://blog.csdn.net/yxdspirit/article/details/84665558
android 9.0 Launcher3修改PageIndicator

ndroid 9.0 Launcher3的PageIndicator 是横线,需要把它改成小圆点

先看Launcher3的xml
res/layout/launcher.xml

...

        <!-- The workspace contains 5 screens of cells -->
        <!-- DO NOT CHANGE THE ID -->
        <com.android.launcher3.Workspace
            android:id="@+id/workspace"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:layout_gravity="center"
            android:theme="@style/HomeScreenElementTheme"
            launcher:pageIndicator="@+id/page_indicator" />

        <include
            android:id="@+id/overview_panel_container"
            layout="@layout/overview_panel"
            android:visibility="gone" />

        <!-- Keep these behind the workspace so that they are not visible when
         we go into AllApps -->
        <com.android.launcher3.pageindicators.WorkspacePageIndicator
            android:id="@+id/page_indicator"
            android:layout_width="match_parent"
            android:layout_height="4dp"
            android:layout_gravity="bottom|center_horizontal"
            android:theme="@style/HomeScreenElementTheme" />

   ...
</com.android.launcher3.LauncherRootView>


这个WorkspacePageIndicator 就是主界面的PageIndicator了
在src/com.android.launcher3.pageindicators下发现还有个PageIndicatorDots
这不就是小圆点的吗
看了下它在Folder里有被调用,所以在桌面上创建了个文件夹,发现这个小圆点的PageIndicator 还不错
所以我就把WorkspacePageIndicator给替换成PageIndicatorDots

        <!-- Keep these behind the workspace so that they are not visible when
         we go into AllApps -->
        <com.android.launcher3.pageindicators.PageIndicatorDots
            android:id="@+id/page_indicator"
            android:layout_width="match_parent"
            android:layout_height="4dp"
            android:layout_gravity="bottom|center_horizontal"
            android:theme="@style/HomeScreenElementTheme" />

然后修改src/com.android.launcher3.Workspace,extends PagedView改为PagedView

/**
 * The workspace is a wide area with a wallpaper and a finite number of pages.
 * Each page contains a number of icons, folders or widgets the user can
 * interact with. A workspace is meant to be used with a fixed width only.
 */
public class Workspace extends PagedView<PageIndicatorDots>
        implements DropTarget, DragSource, View.OnTouchListener,
        DragController.DragListener, Insettable, LauncherStateManager.StateHandler {
    private static final String TAG = "Launcher.Workspace";


发现src/com.android.launcher3.states.SpringLoadedState报错了,设置自动隐藏的 直接注释掉

    @Override
    public void onStateEnabled(Launcher launcher) {
        Workspace ws = launcher.getWorkspace();
        ws.showPageIndicatorAtCurrentScroll();
            //报错点  设置自动隐藏的 直接注释掉
//        ws.getPageIndicator().setShouldAutoHide(false);

        // Prevent any Un/InstallShortcutReceivers from updating the db while we are
        // in spring loaded mode
        InstallShortcutReceiver.enableInstallQueue(InstallShortcutReceiver.FLAG_DRAG_AND_DROP);
        launcher.getRotationHelper().setCurrentStateRequest(REQUEST_LOCK);
    }

    @Override
    public float getWorkspaceScrimAlpha(Launcher launcher) {
        return 0.3f;
    }

    @Override
    public void onStateDisabled(final Launcher launcher) {
    //报错点  设置自动隐藏的 直接注释掉
//        launcher.getWorkspace().getPageIndicator().setShouldAutoHide(true);

        // Re-enable any Un/InstallShortcutReceiver and now process any queued items
        InstallShortcutReceiver.disableAndFlushInstallQueue(
                InstallShortcutReceiver.FLAG_DRAG_AND_DROP, launcher);
    }

调试看的时候发现PageIndicator跑到HotSeat下面去了,这是要在HotSeat上面的

对比看了下WorkspacePageIndicator和PageIndicatorDots
然后发现让PageIndicatorDots implements Insettable 就可以实现在HotSeat上面了

先implements Insettable

public class PageIndicatorDots extends View implements Insettable, PageIndicator {
1
然后直接把WorkspacePageIndicator 的setInsets copy过来

    @Override
    public void setInsets(Rect insets) {
        DeviceProfile grid = mLauncher.getDeviceProfile();
        FrameLayout.LayoutParams lp = (FrameLayout.LayoutParams) getLayoutParams();

        if (grid.isVerticalBarLayout()) {
            Rect padding = grid.workspacePadding;
            lp.leftMargin = padding.left + grid.workspaceCellPaddingXPx;
            lp.rightMargin = padding.right + grid.workspaceCellPaddingXPx;
            lp.bottomMargin = padding.bottom;
        } else {
            lp.leftMargin = lp.rightMargin = 0;
            lp.gravity = Gravity.CENTER_HORIZONTAL | Gravity.BOTTOM;
            lp.bottomMargin = grid.hotseatBarSizePx + insets.bottom;
        }
        setLayoutParams(lp);
    }

最后在构造方法初始化mLauncher

    public PageIndicatorDots(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        mLauncher = Launcher.getLauncher(context);
        mCirclePaint = new Paint(Paint.ANTI_ALIAS_FLAG);

2018-12-04 09:06 更新
在设置里将Launcher3的缓存清掉后,有时会发生奔溃问题
检查发现PageIndicatorDots setScroll 的totalScroll有时会为0导致scrollPerPage会有分母为0的情况

  @Override
    public void setScroll(int currentScroll, int totalScroll) {
        if (mNumPages > 1) {
            if (mIsRtl) {
                currentScroll = totalScroll - currentScroll;
            }
            int scrollPerPage = totalScroll / (mNumPages - 1);
            //add by yy
            if(scrollPerPage == 0)
                return;
            //end add by yy
            int pageToLeft = currentScroll / scrollPerPage;
            int pageToLeftScroll = pageToLeft * scrollPerPage;
            int pageToRightScroll = pageToLeftScroll + scrollPerPage;

            float scrollThreshold = SHIFT_THRESHOLD * scrollPerPage;
            if (currentScroll < pageToLeftScroll + scrollThreshold) {
                // scroll is within the left page's threshold
                animateToPosition(pageToLeft);
            } else if (currentScroll > pageToRightScroll - scrollThreshold) {

2018-12-12 16:16 更新
WorkspacePageIndicator 里有两个方法其他地方可能会被调用,是用来操作动画的
在PageIndicatorDots 可以加上

    /**
     * Pauses all currently running animations.
     */
    public void pauseAnimations() {
        stopAllAnimations();
    }

    /**
     * Force-ends all currently running or paused animations.
     */
    public void skipAnimationsToEnd() {
        stopAllAnimations();
    }

