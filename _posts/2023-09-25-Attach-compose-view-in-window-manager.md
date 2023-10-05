---
layout: post
title: '[Android, Kotlin] WindowManager 에 Compose View 붙이기'
categories: Android
tags: [Android, Kotlin, Compose]
comments: true
---

## 오버레이 팝업

Android 는 앱이 실행 중 이 아닐 때, 특정 상황에서 팝업을 표시하기 위해 Foreground Service 와 window manager 를 사용하여 view 를 표시할 수 있다.

예를 들어 전화 수신 시 해당 번호가 스팸 번호인지 팝업으로 알려주는 통화 스팸 앱 처럼 말이다.

이 글에선 오버레이 팝업 사용법 이 아닌 WindowManager 에 Compose View 를 붙이는 방법과 ViewTreeLifecycleOwner not found 에러를 해결하는 방법을 소개한다.

해당 기능은 디스플레이 위에 그리기 팝업이 필요하다.

## Window Manager 에 View 삽입

Android view 를 사용할 때 Foreground service 에서 popup을 붙일때는 사용 할 view 를 inflate 하고 layou params 와 함께 add view 하여 간단히 표시 할 수 있다.

```kotlin
private fun attachView(){
    val windowManager = ContextCompat.getSystemService(this, WindowManager::class.java)

    //windowManager params 설정
    val params = WindowManager.LayoutParams(
        WindowManager.LayoutParams.MATCH_PARENT,
        WindowManager.LayoutParams.WRAP_CONTENT,
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY else
            WindowManager.LayoutParams.TYPE_SYSTEM_ALERT,
        WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH,
        PixelFormat.TRANSLUCENT
    ).apply {
        gravity = Gravity.START or Gravity.BOTTOM
        x = 0
        y = 0
    }

    // view 에 inflate 된 원하는 android view 삽입
    windowManager?.addView(view, params)
}
```

## Window Manager 에 Compose View 삽입

### Compose View 생성

Compose UI 를 사용하는 kotlin 프로젝트에서도 똑같이 Compose View 를 만들어서 삽입해 보자

```kotlin
private fun attachView(){
    val windowManager = ContextCompat.getSystemService(this, WindowManager::class.java)

    //windowManager params 설정
    val params = WindowManager.LayoutParams(
        WindowManager.LayoutParams.MATCH_PARENT,
        WindowManager.LayoutParams.WRAP_CONTENT,
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY else
            WindowManager.LayoutParams.TYPE_SYSTEM_ALERT,
        WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH,
        PixelFormat.TRANSLUCENT
    ).apply {
        gravity = Gravity.START or Gravity.BOTTOM
        x = 0
        y = 0
    }

    val view = ComposeView(this).apply{
        setContent{
        	// 사용할 Composable function 삽입
        	TestWindowPopup()
        }
    }

    windowManager?.addView(view, params)
}
```

### ViewTreeLifecycleOwner not found Exception

Compose view 를 생성하여 삽입하고 서비스를 실행하면 아래와 같은 에러가 발생한다.

```kotlin
java.lang.IllegalStateException: ViewTreeLifecycleOwner not found from androidx.compose.ui.platform.ComposeView{e2aefdc V.E...... ......I. 0,0-0,0}
                                                                                                    	at androidx.compose.ui.platform.WindowRecomposer_androidKt.createLifecycleAwareWindowRecomposer(WindowRecomposer.android.kt:352)
                                                                                                    	at androidx.compose.ui.platform.WindowRecomposer_androidKt.createLifecycleAwareWindowRecomposer$default(WindowRecomposer.android.kt:325)
                                                                                                    	at androidx.compose.ui.platform.WindowRecomposerFactory$Companion$LifecycleAware$1.createRecomposer(WindowRecomposer.android.kt:168)
                                                                                                    	at androidx.compose.ui.platform.WindowRecomposerPolicy.createAndInstallWindowRecomposer$ui_release(WindowRecomposer.android.kt:224)
                                                                                                    	at androidx.compose.ui.platform.WindowRecomposer_androidKt.getWindowRecomposer(WindowRecomposer.android.kt:300)
                                                                                                    	at androidx.compose.ui.platform.AbstractComposeView.resolveParentCompositionContext(ComposeView.android.kt:244)
                                                                                                    	at androidx.compose.ui.platform.AbstractComposeView.ensureCompositionCreated(ComposeView.android.kt:251)
                                                                                                    	at androidx.compose.ui.platform.AbstractComposeView.onAttachedToWindow(ComposeView.android.kt:283)
                                                                                                    	at android.view.View.dispatchAttachedToWindow(View.java:21291)
                                                                                                    	at android.view.ViewGroup.dispatchAttachedToWindow(ViewGroup.java:3491)
                                                                                                    	at android.view.ViewRootImpl.performTraversals(ViewRootImpl.java:2771)
                                                                                                    	at android.view.ViewRootImpl.doTraversal(ViewRootImpl.java:2286)
                                                                                                    	at android.view.ViewRootImpl$TraversalRunnable.run(ViewRootImpl.java:8948)
```

### Solve

Compose view 에는 lifeCycler Owner 가 필요한데 (일반적으로는 Activity가 그 역할을 함) 현재 없기 때문에 발생하는 에러다.

우선 LifeCycleOwner class 를 생성한다

```kotlin
class PopupLifeCycleOwner : SavedStateRegistryOwner {
    private var mLifecycleRegistry: LifecycleRegistry = LifecycleRegistry(this)
    private var mSavedStateRegistryController: SavedStateRegistryController =
        SavedStateRegistryController.create(this)

    /**
     * @return True if the Lifecycle has been initialized.
     */
    val isInitialized: Boolean
        get() = true

    override val lifecycle: Lifecycle get() = mLifecycleRegistry

    fun setCurrentState(state: Lifecycle.State) {
        mLifecycleRegistry.currentState = state
    }

    fun handleLifecycleEvent(event: Lifecycle.Event) {
        mLifecycleRegistry.handleLifecycleEvent(event)
    }

    override val savedStateRegistry: SavedStateRegistry =
        mSavedStateRegistryController.savedStateRegistry

    fun performRestore(savedState: Bundle?) {
        mSavedStateRegistryController.performRestore(savedState)
    }

    fun performSave(outBundle: Bundle) {
        mSavedStateRegistryController.performSave(outBundle)
    }
}
```

그리고 Compose view 를 생성했던 부분에서 lifeCycler Owner 와 ViewModelStoreOwner 그리고 Recomposer 를 각각 세팅해 준다.

```kotlin
private fun attachView(){
    val windowManager = ContextCompat.getSystemService(this, WindowManager::class.java)

    //windowManager params 설정
    val params = WindowManager.LayoutParams(
        WindowManager.LayoutParams.MATCH_PARENT,
        WindowManager.LayoutParams.WRAP_CONTENT,
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O)
            WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY else
            WindowManager.LayoutParams.TYPE_SYSTEM_ALERT,
        WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH,
        PixelFormat.TRANSLUCENT
    ).apply {
        gravity = Gravity.START or Gravity.BOTTOM
        x = 0
        y = 0
    }

    val view = ComposeView(this)

    // lifeCycleOwner 생성
    val lifecycleOwner = PopupLifeCycleOwner().apply {
        performRestore(null)
        handleLifecycleEvent(Lifecycle.Event.ON_CREATE)
    }

    // ViewModelStoreOwner 생성
    val viewModelStore = ViewModelStore()
    val viewModelStoreOwner = object : ViewModelStoreOwner {
        override val viewModelStore: ViewModelStore
            get() = viewModelStore
    }

    // Recomposer 생성
    val coroutineContext = AndroidUiDispatcher.CurrentThread
    val runRecomposeScope = CoroutineScope(coroutineContext)
    val reComposer = Recomposer(coroutineContext)

    view.apply {
        setViewTreeSavedStateRegistryOwner(lifecycleOwner)
        setViewTreeLifecycleOwner(lifecycleOwner)
        setViewTreeViewModelStoreOwner(viewModelStoreOwner)
        setViewCompositionStrategy(
            ViewCompositionStrategy.DisposeOnDetachedFromWindow
        )
        setContent {
            TestWindowPopup()
        }
        compositionContext = reComposer
        runRecomposeScope.launch {
            reComposer.runRecomposeAndApplyChanges()
        }
    }

    windowManager?.addView(view, params)
}
```

참조: [StackOverFlow](https://stackoverflow.com/a/70460554)
