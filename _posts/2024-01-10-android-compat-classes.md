---
layout: post
title: '[Android, Kotlin] Android의 Compat 클래스들'
categories: [Android]
tags: [Android, Kotlin]
comments: true
banner:
     image: "/assets/images/default_android_banner.png"
---
## Compat class 가 무엇인가요?

안드로이드는 1년에 한번 SDK 메이저 업데이트가 진행되는 특성상 많은 기능들이 추가되고
deprecate되기 때문에 해당 기능들을 사용할 때 마다 디바이스의 SDK 버전을 체크 해줘야합니다.

안드로이드의 Compat class 들은 이러한 번거러움을 해소해주기 위해
SDK 버전을 체크하여 버전에 맞는 기능을 알아서 사용하도록 구현되어있으며 대부분의 메소드들은
기존의 비 Compat 메소드와 동일하게 사용할 수 있습니다.

보통 사용하는 안드로이드 관련 클래스의 이름 뒤에 Compat을 붙여서 안드로이드 스튜디오에서 검색해보면 쉽게 발견 할 수 있습니다.

많은 Compat 클래스 들이 있지만 현 시점에서 가장 많이 사용하는 min SDK 23 ~ 24 를 기준으로 자주 사용하는 것들만 아래에 모아보았습니다.

## ContextCompat

Acitivty나 fragment, Service 등의 컴포넌트의 context 와 관련된 기능들이 구현되어 있습니다.

23 보다 더 낮은 min SDK 를 사용한다면 color, string 등 resource 와 관련된 메소드들도 구현되어있으니 찾아보면 좋을 것 같습니다.

[Google Android 의 ContextCompat Reference](https://developer.android.com/reference/androidx/core/content/ContextCompat)

### getSystemService(Context, Class)

`Context` 에서는 SDK 23 에서 부터 추가된 메소드로 `Context.SystemServiceName` 으로 시스템 서비스를
가져오는 것이 아닌 해당 시스템 서비스 클래스를 매개변수로 가져올 때 사용합니다.

기존 `Context` 를 사용하여 `getSystemService()` 를 호출한 리턴값은 구현부의 `@Nullable` 주석이 있음에도 불구하고 NonNull Class 가 리턴되는데
`ContextCompat` 을 사용하면 정상적인 `Nullable` 리턴타입으로 리턴됩니다.

구현부의 주석을 살펴보면 예약된 시스템 서비스가 아닌 클래스나 클래스 이름을 매개변수로 사용했을때 또는
앱이 인스턴트 앱인 경우 null이 리턴 될 수 있다고 되어있습니다.

일반적인 상황에서는 Null 걱정을 하지 않아도 되기 때문에 인스턴트 앱이 아니라면
Null 을 무시하고 `NullPointerException` 을 던질지 Null 대비 처리를 할것인지는 개발자에게 달려있습니다.

```Kotlin
//before
val wm = context.getSystemService(Context.WINDOW_SERVICE) as WindowManager
// or
val wm2 = context.getSystemService(WindowManager::class.java)

//ContextCompat
val wm3 = ContextCompat.getSystemService(context, WindowManager::class.java)
```

* Context 의 getSystemService 는 리턴값이 `Nonnull` 로 리턴됩니다.
    ![](/assets/images/2024-01-10/NullableSystemService.png)

    하지만 실제 `getSystemService()` 의 java 구현 코드를 확인하면 실제로는 nullable 로 리턴되는것에 주의해야 합니다.
    ![](/assets/images/2024-01-10/NullableButNotCollect.png)

* ContextCompat 를 사용하면 리턴값이 `Nullable` 로 리턴됩니다.
    ![](/assets/images/2024-01-10/NonNullSystemService.png)

### checkSelfPermission()

SDK 33 에 추가된 `Manifest.permission.POST_NOTIFICATIONS` 권한을 이전 버전의 기기에서 알아서 체크 해 줍니다.

### startForegroundService()

포그라운드 서비스를 실행할때 사용합니다.

context 의 `startForegroundService()` 는 SDK 26부터 추가되었습니다.

### getDataDir()

앱 내부 디렉터리의 경로를 반환해줍니다.

context 의 `getDataDir()` 메소드는 SDK 24 부터 추가되었습니다.

## IntentCompat, BundleCompat
Intent 의 extraData 인 bundle 로 부터 `Parcleable Class` 를 가져올 때 사용됩니다. 

### getPacrcelableExtra()

기존 Intent 및 Bundle 에서 사용하던 타입 추정 `getParcelableExtra(String)` 메소드는
SDK33 부터 deprecate 되고 타입 클래스를 함께 사용하는 `getParcelableExtra(String, Class)` 메소드가 추가되었습니다.

이는 getParcelableArrayExtra() 및 getParcelableArrayListExtra 에도 똑같이 적용됩니다.

```Kotlin
// before deprecated
val extraUri = intent.getParcelableExtra<Uri>(Const.EXTRA_MESSAGE_IMG_URI)
val extraUri2 = intent.getParcelableArrayExtra(Const.EXTRA_MESSAGE_IMG_URI)
val extraUri3 = intent.getParcelableArrayListExtra<Uri>(Const.EXTRA_MESSAGE_IMG_URI)

// new after SDK 33
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    val newExtra = intent.getParcelableExtra(Const.EXTRA_MESSAGE_IMG_URI, Uri::class.java)
    val newExtra2 = intent.getParcelableArrayExtra(Const.EXTRA_MESSAGE_IMG_URI, Uri::class.java)
    val newExtra3 = intent.getParcelableArrayListExtra(Const.EXTRA_MESSAGE_IMG_URI, Uri::class.java)
}

// use Compat
val compatExtra = IntentCompat.getParcelableExtra(intent, Const.EXTRA_MESSAGE_IMG_URI, Uri::class.java)
val compatExtra2 = IntentCompat.getParcelableArrayExtra(intent, Const.EXTRA_MESSAGE_IMG_URI,Uri::class.java)
val compatExtra3 = IntentCompat.getParcelableArrayListExtra(intent, Const.EXTRA_MESSAGE_IMG_URI,Uri::class.java)
```

## ShareCompat

액티비티 간에 데이터를 공유하는 Intent를 쉽게 만들어 주는 클래스로
이 클래스는 Compat 클래스 라기 보다 Util 클래스에 가깝다고 볼 수 있습니다.

.IntentBuild(Context) 를 사용하여 Builder 패턴을 통해 intent를 생성하고
.startChooser() 를 통해 intent 를 전달할 activity chooser 를 실행시켜 줍니다.

```kotlin
// Image 공유 예시
ShareCompat.IntentBuilder(context).apply{
	setType("image/*")
	setStream(imageUri)
	setChooserTitle("Image Share")
}.startChooser()
```

## NotificationManagerCompat

알림 및 알림채널 생성을 위한 `NotificationManager` 시스템 클래스의 Compat 클래스로 단독으로 사용하기 보단
후술할 notification 과 channel 의 Compat 클래스와 함께 사용됩니다.

`NotificationManagerCompat` 은 ContextCompat 의 `getSystemClass()` 가 아닌 `.from(Context)` 메소드로 생성할 수 있습니다.
```
// before
val notyMgr = ContextCompat.getSystemService(context, NotificationManager::class.java)

// compat
val notyMgrCompat = NotificationManagerCompat.from(context)
```

## NotificationChannelCompat

notification channel 자체가 SDK 26 부터 사용되었기 때문에 
해당 클래스는 이전 SDK 버전에서는 아무 동작도 하지 않습니다.

기존 `NotificationChannel` 클래스와 다르게 Builder 패턴을 사용하고 있으며 
.build() 메소드로 채널을 생성하여 위의 `NotificationManagerCompat` 을 통해 채널을 등록할 수 있습니다.

```
val channel = NotificationChannelCompat.Builder(channelId, importance).apply {
    setName(channelName)
    setLightsEnabled(true)
    setLightColor(Color.GREEN)
    setShowBadge(true)
}.build()

// notificationManagerCompat 으로 channel 생성
val notyMgr = NotificationManagerCompat.from(context)
notyMgr.createNotificationChannel(channel)
```

## NotificationCompat

`Notification` 은 SDK 버전이 한단계 한단계 오를때 마다 기능들이 추가될 뿐만 아니라 
여러 알림 스타일 까지 각 SDK 버전 마다 흩어져 있기 때문에 왠만하면 Compat클래스를 사용하는것이 좋습니다.

`NotificationChannelCompat` 과 마찬가지로 Builder 패턴을 사용합니다.

알림 위의 `NotificationManageCompat` 의 `notify()` 를 통해 발행할 수 있습니다.

부재중 전화 알림을 만드는 예시입니다.

```Kotlin
val builder = NotificationCompat.Builder(
    context,
    NotificationChannelManager.NOTIFICATION_CHANNEL_ID
)

builder.color = ContextCompat.getColor(
    context,
    R.color.notification_color
)
builder.setGroup(GROUP_STRING)

val pendingIntent = makeDialIntent(context, isBusiness, number, requestCode)
var count = getNotyCount(context, requestCode)
val sender = getContactSender(isBusiness, number) + " (" + ++count + ")"

builder.setContentIntent(pendingIntent)
builder.setSmallIcon(R.drawable.ic_missed_call_noty)
builder.setContentTitle(context.getString(R.string.missed_call))
builder.setContentText(sender)
builder.priority = NotificationCompat.PRIORITY_HIGH
builder.setAutoCancel(true)
builder.setCategory(NotificationCompat.CATEGORY_MISSED_CALL)

val callPending = makeCallActionIntent(context, number, subId, requestCode)
val msgPending = makeMsgActionIntent(context, number, subId, requestCode)

builder.addAction(R.drawable.ic_call_white, context.getString(R.string.call), callPending)
builder.addAction(
    R.drawable.ic_balloon_white,
    context.getString(R.string.message),
    msgPending
)

val notification = builder.build()
val notyMgr = NotificationManagerCompat.from(context)
notyMgr.notify(requestCode, notification)
notyMgr.notify(SUMMERY_NOTY_ID, buildSummery(context))
```
