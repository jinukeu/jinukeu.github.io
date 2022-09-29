---
title: "[Android] Intents and Intent Filters (작성 중)"
excerpt: "Intents and Intent Filters 공식 문서 번역"

categories:
  - Android
tags:
  - []

permalink: /android/Intents-and-Intent-Filters/

toc: true
toc_sticky: true

date: 2022-09-27
last_modified_at: 2022-09-27
---

> 안드로이드 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 내용이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://developer.android.com/guide/components/intents-filters)

<br>

Intent는 메시징 객체로, 다른 앱 구성 요소로부터 작업을 요청하는 데 사용할 수 있습니다. 인텐트가 구성 요소 사이의 통신을 촉진하는 데는 여러 가지 방식이 있지만 기본적인 사용 사례는 크게 세 가지로 나눌 수 있습니다.

* 액티비티 시작   
Activity는 앱 안의 단일 화면을 나타냅니다. Activity의 새 인스턴스를 시작하려면 Intent를 startActivity()로 전달하면 됩니다. Intent는 시작할 액티비티를 설명하고 모든 필수 데이터를 담습니다.   
액티비티가 완료되었을 때 결과를 수신하려면, startActivityForResult()를 호출합니다. 액티비티는 해당 결과를 이 액티비티의 onActivityResult() 콜백에서 별도의 Intent 객체로 수신합니다. 자세한 내용은 액티비티 가이드를 참조하세요.

* 서비스 시작   
Service는 사용자 인터페이스 없이 백그라운드에서 작업을 수행하는 구성 요소입니다. Android 5.0(API 레벨 21) 이상부터는 JobScheduler로 서비스를 시작할 수 있습니다. JobScheduler에 대한 자세한 내용은 [API-reference documentation](https://developer.android.com/reference/android/app/job/JobScheduler?hl=ko)을 참조하세요.   
Android 5.0(API 레벨 21) 이하 버전은 Service 클래스의 메서드를 사용하면 서비스를 시작할 수 있습니다. 서비스를 시작하여 일회성 작업을 수행하도록 하려면(예: 파일 다운로드) Intent를 startService()에 전달하면 됩니다. Intent는 시작할 서비스를 설명하고 모든 필수 데이터를 담고 있습니다.   
서비스가 클라이언트-서버 인터페이스로 디자인된 경우, 다른 구성 요소로부터 서비스에 바인딩하려면 Intent를 bindService()에 전달하면 됩니다. 자세한 내용은 (서비스)[https://developer.android.com/guide/components/services?hl=ko] 가이드를 참조하세요.

* 브로드캐스트 전달   
브로드캐스트는 모든 앱이 수신할 수 있는 메시지입니다. 시스템은 시스템이 부팅될 때 또는 기기가 충전을 시작할 때 등 시스템 이벤트에 대한 다양한 브로드캐스트를 전달합니다. Intent를 sendBroadcast() 또는 sendOrderedBroadcast()에 전달하면 다른 앱에 브로드캐스트를 전달할 수 있습니다.

이 페이지의 나머지에서는 인텐트의 작동 원리와 사용 방법을 설명합니다. 관련 정보는 (다른 앱과의 상호작용)[https://developer.android.com/training/basics/intents?hl=ko] 및 (콘텐츠 공유)[https://developer.android.com/training/sharing?hl=ko]를 참조하세요.

<br>

## 인텐트 유형
인텐트에는 두 가지 유형이 있습니다.

* **명시적 인텐트(Explicit intents)**는 인텐트를 충족하는 애플리케이션이 무엇인지 지정합니다. 이를 위해 대상 앱의 패키지 이름 또는 완전히 자격을 갖춘 구성 요소 클래스 이름을 제공합니다. 명시적 인텐트는 일반적으로 앱 안에서 구성 요소를 시작할 때 씁니다. 시작하고자 하는 액티비티 또는 서비스의 클래스 이름을 알고 있기 때문입니다. 예를 들어, 사용자 작업에 응답하여 새로운 액티비티를 시작하거나 백그라운드에서 파일을 다운로드하기 위해 서비스를 시작하는 것 등이 여기에 해당됩니다.
* **암시적 인텐트(Implicit intents)**는 특정 구성 요소의 이름을 대지 않지만, 그 대신 수행할 일반적인 작업을 선언하여 다른 앱의 구성 요소가 이를 처리할 수 있도록 해줍니다. 예를 들어 사용자에게 지도에 있는 한 위치를 표시하고자 하는 경우, 암시적 인텐트를 사용하여 해당 기능을 갖춘 다른 앱이 지정된 위치를 지도에 표시하도록 요청할 수 있습니다.   

그림 1은 액티비티를 시작할 때 인텐트를 사용하는 법을 나타낸 것입니다. Intent 객체가 특정 액티비티 구성 요소를 명시적으로 지정하면 시스템이 해당 구성 요소를 즉시 시작합니다.

![그림 1](https://developer.android.com/static/images/components/intent-filters_2x.png?hl=ko)
그림 1. 암시적(Implicit) 인텐트가 시스템을 통해 전달되어 다른 액티비티를 시작하는 방법. [1] 액티비티 A가 작업 설명이 있는 Intent를 생성하여 이를 startActivity()에 전달합니다. [2] Android 시스템이 모든 앱에서 해당 인텐트와 일치하는 인텐트 필터를 검색합니다. 일치하는 항목을 찾으면, [3] 시스템이 해당 액티비티의 onCreate() 메서드를 호출하여 일치하는 액티비티(액티비티 B)를 시작하고 Intent를 전달합니다.  


암시적 인텐트를 사용하면 Android 시스템에서 시작할 적절한 구성 요소를 찾습니다. 이때 인텐트의 내용을 기기에 있는 다른 여러 앱의 매니페스트 파일에서 선언된 인텐트 필터와 비교하는 방법을 사용합니다. 해당 인텐트와 일치하는 인텐트 필터가 있으면 시스템에서 해당 구성 요소를 시작하고 Intent 객체를 전달합니다. 호환되는 인텐트 필터가 여러 개인 경우, 시스템에서 대화상자를 표시하여 사용자가 어느 앱을 사용할지 직접 선택할 수 있게 합니다.   


인텐트 필터란 앱의 매니페스트 파일에 들어 있는 표현으로, 해당 구성 요소가 수신하고자 하는 인텐트의 유형을 나타냅니다. 예를 들어 액티비티에 대한 인텐트 필터를 선언하면 다른 여러 앱이 특정한 종류의 인텐트를 가지고 액티비티를 직접 시작할 수 있습니다. 마찬가지로 액티비티에 대한 인텐트 필터를 선언하지 않는 경우, 이것은 명시적인 인텐트로만 시작할 수 있습니다.   

> 주의 : 앱의 보안을 보장하려면 Service를 시작할 때에는 항상 명시적 인텐트만 사용하고 서비스에 대한 인텐트 필터는 선언하지 마세요. 암시적 인텐트를 사용하여 서비스를 시작하면 보안 위험을 초래합니다. 인텐트에 어느 서비스가 응답할지 확신할 수 없고, 사용자는 어느 서비스가 시작되는지 볼 수 없기 때문입니다. Android 5.0(API 레벨 21)부터 시스템은 개발자가 암시적 인텐트로 bindService()를 호출하면 예외를 발생시킵니다.   

<br>

## 인텐트 빌드
Intent 객체에는 Android 시스템이 어느 구성 요소를 시작할지 판별하는 데 사용하는 정보가 담겨 있습니다(예를 들어 정확한 구성 요소 이름 또는 인텐트를 수신해야 하는 구성 요소 카테고리 등). 또한 수신자 구성 요소가 작업을 적절히 수행하기 위해 사용할 정보(예: 수행할 작업 및 조치를 취할 데이터 위치 등)도 이 안에 담겨 있습니다.

Intent에 포함된 기본 사항은 다음과 같습니다.   

### 구성 요소 이름
시작할 구성 요소의 이름입니다.
이것은 선택 항목이지만, 인텐트를 명시적인 것으로 만들어주는 중요한 정보입니다. 다시 말해 이 인텐트는 구성 요소 이름이 정의한 앱 구성 요소에만 전달되어야 한다는 뜻입니다. 구성 요소 이름이 없으면 해당 인텐트는 암시적이며, 인텐트를 수신해야 하는 구성 요소는 다른 인텐트 정보를 기반으로 시스템이 결정합니다(예를 들어 작업, 데이터 및 카테고리. 아래 설명 참조). 앱에서 특정한 구성 요소를 시작해야 하는 경우에는 구성 요소 이름을 지정해야 합니다.   

> 참고 : Service를 시작하는 경우, 항상 구성 요소 이름을 지정해야 합니다. 그렇지 않으면 인텐트에 어느 서비스가 응답할지 확신할 수 없고, 사용자도 어느 서비스가 시작되는지 볼 수 없게 됩니다.   

Intent의 필드는 ComponentName 객체로, 이것을 지정하려면 대상 구성 요소의 완전히 정규화된 클래스 이름(앱의 패키지 이름 포함)을 사용해야 합니다. 예를 들어 com.example.ExampleActivity를 쓰세요. 구성 요소 이름을 설정하려면 setComponent(), setClass(), setClassName()을 사용하거나 또는 Intent 생성자를 사용합니다.   

### 작업
수행할 일반적인 작업을 나타내는 문자열입니다(예: 보기 또는 선택).   

브로드캐스트 인텐트의 경우, 이것은 이미 실행되어 보고되고 있는 작업입니다. 이 작업은 대체로 나머지 인텐트의 구조를 결정합니다. 특히, 데이터와 엑스트라에 포함되는 정보가 결정됩니다.   


본인의 앱 내에 있는 인텐트가 사용할 작업(또는 다른 앱이 사용하여 본인의 앱 안의 구성 요소를 호출하게 할 작업)을 직접 지정할 수도 있지만, 보통은 Intent 클래스나 다른 프레임워크 클래스가 정의한 작업 상수를 지정해야 합니다. 다음은 액티비티를 시작하는 데 쓰이는 보편적인 작업입니다.

#### ACTION_VIEW
이 작업은 액티비티가 사용자에게 표시할 수 있는 어떤 정보를 가지고 있을 때 startActivity()가 있는 인텐트에서 사용합니다. 예를 들어 갤러리 앱에서 볼 사진이나 지도 앱에서 볼 주소 등이 이에 해당됩니다.
#### ACTION_SEND
공유 인텐트라고도 하며, 사용자가 다른 앱을 통해 공유할 수 있는 데이터를 가지고 있을 때 startActivity()가 있는 인텐트에서 사용해야 합니다. 예를 들어 이메일 앱 또는 소셜 공유 앱 등이 이에 해당됩니다.
일반적인 작업을 정의하는 상수에 대한 자세한 내용은 [Intent](https://developer.android.com/reference/android/content/Intent?hl=ko) 클래스 참조 문서를 확인하세요. 다른 작업은 Android 프레임워크의 다른 곳에서 정의됩니다. 예를 들어 시스템의 설정 앱에서 특정 화면을 여는 작업의 경우 Settings에서 정의됩니다.

인텐트에 대한 작업을 지정하려면 setAction() 또는 Intent 생성자를 사용하면 됩니다.

나름의 작업을 직접 정의하는 경우, 앱의 패키지 이름을 접두어로 포함시켜야 합니다. 이는 다음 예시에서 확인할 수 있습니다.   

```kotlin
const val ACTION_TIMETRAVEL = "com.example.action.TIMETRAVEL"
```   

### 데이터
작업을 수행할 데이터 및/또는 해당 데이터의 MIME 유형을 참조하는 URI(Uri 객체)입니다. 제공된 데이터의 유형을 나타내는 것은 일반적으로 인텐트의 작업입니다. 예를 들어 인텐트가 ACTION_EDIT인 경우, 데이터에 편집할 문서의 URI가 들어 있어야 합니다.
인텐트를 생성할 때는 URI 외에 데이터 유형(데이터의 MIME 유형)을 지정하는 것이 중요한 경우가 많습니다. 예를 들어 이미지를 표시할 수 있는 액티비티라면 오디오 파일은 재생하지 못할 것입니다(URI 형식은 비슷할지라도). 데이터의 MIME 유형을 지정하면 Android 시스템이 인텐트를 수신하기 가장 좋은 구성 요소를 찾는 데 도움이 됩니다. 다만 때로는 MIME 유형을 URI를 통해 추론할 수 있습니다. 특히 데이터가 content: URI일 때 추론하기 쉽습니다. content: URI는 데이터가 기기에 위치하고 ContentProvider가 제어한다는 것을 의미하며, 따라서 데이터 MIME 유형이 시스템에 표시됩니다.

데이터 URI만 설정하려면 setData()를 호출하세요. MIME 유형만 설정하려면 setType()을 호출하세요. 필요한 경우, setDataAndType()을 사용하여 두 가지를 모두 명시적으로 설정할 수 있습니다.   

> 주의: URI와 MIME 유형을 둘 다 설정하고자 하는 경우, setData() 및 setType()을 호출하면 안 됩니다. 이 둘은 서로의 값을 무효화하기 때문입니다. URI와 MIME 유형을 둘 모두 설정하려면 항상 setDataAndType()을 사용하세요.  

### 카테고리
인텐트를 처리해야 하는 구성 요소의 종류에 관한 추가 정보를 담은 문자열입니다. 인텐트 안에는 카테고리 설명이 얼마든지 들어 있을 수 있지만, 대부분의 인텐트에는 카테고리가 없어도 됩니다. 다음은 몇 가지 보편적인 카테고리입니다.
#### CATEGORY_BROWSABLE
대상 액티비티가 웹브라우저를 통해 시작되도록 허용하고 이미지, 이메일 메시지 등의 링크로 참조된 데이터를 표시하게 합니다.
#### CATEGORY_LAUNCHER
이 액티비티가 작업의 최초 액티비티이며, 시스템의 애플리케이션 시작 관리자에 목록으로 게재됩니다.
카테고리의 전체 목록을 보려면 Intent 클래스 설명을 참조하세요.

카테고리는 addCategory()로 지정할 수 있습니다.

위에 나열된 이러한 속성(구성 요소 이름, 작업, 데이터 및 카테고리)은 인텐트를 정의하는 특성을 나타냅니다. Android 시스템은 이와 같은 속성을 읽어 어느 앱 구성 요소를 시작해야 할지 확인할 수 있습니다. 그러나 인텐트는 앱 구성 요소로 확인되는 방법에 영향을 미치지 않는 추가 정보도 담고 있을 수 있습니다. 인텐트가 제공할 수 있는 기타 정보는 다음과 같습니다.   

### 엑스트라
요청된 작업을 수행하는 데 필요한 추가 정보가 담긴 키-값 쌍입니다. 몇몇 작업이 특정한 종류의 데이터 URI를 사용하는 것과 마찬가지로, 몇몇 작업은 특정한 엑스트라도 사용합니다.
다양한 putExtra() 메서드로 엑스트라 데이터를 추가할 수 있습니다. 각 메서드는 키 이름과 값, 이렇게 두 개의 매개변수를 취합니다. 또한 모든 엑스트라 데이터를 포함한 Bundle 객체를 만든 다음 Bundle을 Intent에 putExtras()로 삽입할 수도 있습니다.

예를 들어 ACTION_SEND로 이메일을 전송할 인텐트를 생성하는 경우 받는 사람 수신자를 지정할 때 EXTRA_EMAIL 키를 사용한 다음, 제목은 EXTRA_SUBJECT 키로 지정하면 됩니다.

Intent 클래스는 표준화된 데이터 유형에 대해 여러 가지 EXTRA_* 상수를 지정합니다. 나름의 엑스트라 키를 선언해야 하는 경우(본인의 앱이 수신할 인텐트에 대하여), 앱의 패키지 이름을 접두어로 포함해야 합니다.

```kotlin
const val EXTRA_GIGAWATTS = "com.example.EXTRA_GIGAWATTS"
```

> 주의 : 다른 앱이 수신할 것으로 생각하는 인텐트를 전송할 때는 Parcelable 또는 Serializable 데이터를 사용하지 마세요. 앱이 Bundle 객체에 있는 데이터에 액세스하려고 시도했지만 파슬링되거나 직렬화된 클래스에 액세스하지 못하면 시스템이 RuntimeException을 발생시킵니다.   

### 플래그
플래그는 Intent 클래스에서 정의되고, 인텐트에 대한 메타데이터와 같은 기능을 합니다. 이런 플래그는 Android 시스템에 액티비티를 시작할 방법에 대한 지침을 줄 수도 있고(예를 들어 액티비티가 어느 작업에 소속되어야 하는지 등) 액티비티를 시작한 다음에 어떻게 처리해야 하는지도 알려줄 수 있습니다(예를 들어 해당 액티비티가 최근 액티비티 목록에 소속되는지 여부).
자세한 내용은 [setFlags()](https://developer.android.com/reference/android/content/Intent#setFlags(int)) 메서드를 참조하세요.   

## 명시적 인텐트 예시
명시적 인텐트는 앱 내의 특정 액티비티나 서비스 등 특정한 앱 구성 요소를 시작하는 데 사용하는 인텐트입니다. 명시적 인텐트를 생성하려면 Intent 객체에 대한 구성 요소 이름을 정의합니다. 다른 인텐트 속성은 모두 선택 사항입니다.

예를 들어 앱 안에 DownloadService라는 서비스를 구축했다고 가정해보겠습니다. 이 서비스는 웹상에서 파일을 다운로드하도록 설계되었습니다. 이 서비스를 시작하려면 다음과 같은 코드를 사용합니다.   
```kotlin
// Executed in an Activity, so 'this' is the Context
// The fileUrl is a string URL, such as "http://www.example.com/image.png"
val downloadIntent = Intent(this, DownloadService::class.java).apply {
    data = Uri.parse(fileUrl)
}
startService(downloadIntent)
```
Intent(Context, Class) 생성자가 앱에 Context를 제공하고 구성 요소에 Class 객체를 제공합니다. 이처럼 이 인텐트는 앱 내의 DownloadService 클래스를 명시적으로 시작합니다.

서비스를 빌드하고 시작하는 방법에 대한 자세한 내용은 (서비스)[https://developer.android.com/guide/components/services] 가이드를 참조하세요.   

## 암시적 인텐트 예시
암시적 인텐트는 작업을 지정하여 기기에서 해당 작업을 수행할 수 있는 모든 앱을 호출할 수 있도록 합니다. 본인의 앱은 작업을 수행할 수 없지만 다른 앱은 그 작업을 수행할 수 있는 가능성이 있고, 사용자가 어떤 앱을 사용할지 선택하기를 원할 경우에 암시적 인텐트가 유용합니다.

예를 들어 사용자가 다른 사람들과 공유했으면 하는 콘텐츠를 가지고 있는 경우, ACTION_SEND 작업이 있는 인텐트를 생성한 다음 공유할 콘텐츠를 지정하는 엑스트라를 추가하면 됩니다. 해당 인텐트로 startActivity()를 호출하면 사용자가 어느 앱을 통해 콘텐츠를 공유할지 선택할 수 있습니다.
```kotlin
// Create the text message with a string.
val sendIntent = Intent().apply {
    action = Intent.ACTION_SEND
    putExtra(Intent.EXTRA_TEXT, textMessage)
    type = "text/plain"
}

// Try to invoke the intent.
try {
    startActivity(sendIntent)
} catch (e: ActivityNotFoundException) {
    // Define what your app should do if no activity can handle the intent.
}
```
startActivity()를 호출하면 시스템이 설치된 앱을 모두 살펴보고 이런 종류의 인텐트를 처리할 수 있는 앱이 어느 것인지 알아봅니다(ACTION_SEND 작업이 있는 인텐트이며 "텍스트/일반" 데이터가 담긴 것). 이것을 처리할 수 있는 앱이 하나뿐이면, 해당 앱이 즉시 열리고 이 앱에 인텐트가 주어집니다. 인텐트를 허용하는 액티비티가 여러 개인 경우, 시스템은 그림 2에서와같이 대화상자를 표시하여 사용자에게 앱을 선택하게 합니다.

다른 앱을 시작하는 방법에 대한 자세한 내용은 [사용자를 다른 앱에 보내기](https://developer.android.com/training/basics/intents/sending)를 참조하세요.

![그림 2](https://developer.android.com/static/images/training/basics/intent-chooser.png)
그림 2   

### 앱 선택기 강제 적용
암시적 인텐트에 응답하는 앱이 두 개 이상인 경우, 사용자가 어느 앱을 사용할지 선택하고 이 작업에 대한 기본 앱으로 지정할 수 있습니다. 이는 사용자가 이제부터 계속 같은 앱을 사용할 것으로 추정되는 작업을 수행할 때 좋습니다. 예를 들어 웹 페이지를 열 때가 이에 해당합니다(대다수 사용자가 웹브라우저는 하나만 사용하는 것을 선호합니다).

그러나 인텐트에 응답할 수 있는 앱이 여러 개이고 사용자가 매번 다른 앱을 사용하기를 원할 수도 있는 경우라면, 선택기 대화상자를 명시적으로 표시해야 합니다. 선택기 대화상자는 사용자가 작업에 어떤 앱을 사용할지 매번 물어봅니다(사용자는 작업에 사용할 기본 앱을 선택할 수 없습니다). 예를 들어 앱이 ACTION_SEND 작업으로 "공유"를 수행하는 경우, 사용자는 각자의 현재 상황에 따라 여러 가지 다른 앱을 사용하여 공유하기를 원할 수도 있습니다. 따라서 항상 선택기 대화상자를 사용해야 합니다(그림 2 참조).

선택기를 표시하려면 createChooser()를 사용하여 Intent를 만들고 startActivity()에 전달합니다. 이 예시는 createChooser() 메서드에 전달된 인텐트에 응답하는 앱의 목록을 대화상자에 표시하며, 제공된 텍스트를 대화상자 제목으로 사용합니다.   

```kotlin
val sendIntent = Intent(Intent.ACTION_SEND)
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
val title: String = resources.getString(R.string.chooser_title)
// Create intent to show the chooser dialog
val chooser: Intent = Intent.createChooser(sendIntent, title)

// Verify the original intent will resolve to at least one activity
if (sendIntent.resolveActivity(packageManager) != null) {
    startActivity(chooser)
}
```
   

## 안전하지 않은 인텐트 시작 감지
당신의 앱은 앱 내부 구성 요소를 이동하거나 다른 앱 대신 작업을 수행하기 위해 인텐트를 실행할 수 있습니다. 플랫폼 보안을 향상시키기 위해, 안드로이드 12(API level 31) 이상부터는 만약 당신의 앱이 안전하지 않은 인텐트 실행을 수행한다면 경고를 해주는 디버깅 기능을 제공합니다. 예를 들어 다른 인텐트에서 extra로 전달된 인텐트를 호출하는 안전하지 않은 *nested 인텐트*를 당신의 앱이 실행할 수 있습니다.   

만약 앱이 다음 두 개의 작업을 수행한다면, 시스템은 안전하지않은 인텐트 실행을 김자하고 StrictMode 위반을 발생시킵니다.   
1. 앱이 전달된 인텐트의 extras로부터의 nested intent를 unparcels했을 때
2. 앱이 즉시 nested 인텐트를 사용하여 앱 구성 요소를 시작할 때, 예를 들어 startActivity(), startService() 또는 bindService()로 인텐트를 전달할 때   

이런 상황을 감지하고 앱을 변화시키는 방법은 (Android Nesting Intents)[https://medium.com/androiddevelopers/android-nesting-intents-e472fafc1933]를 읽어보세요.   

<br>

### 안전하지 않은 인텐트 실행 확인
앱에서 안전하지 않은 인텐트를 확인하기 위해, VmPolicy를 설정할 때 detextUnsafeIntentLaunch()를 호출하세요. 다음 코드 스니펫에서 확인할 수 있습니다. 만약 앱이 StrictMode violation을 감지한다면 잠재적으로 민감한 정보를 보호하기 위해 앱 실행을 멈추고 싶을 것이다.   

> 노트 : 만약 VmPolicy 정의에서 안드로이드 12를 타겟팅한다면 detectUnsafeIntentLaunch() 메서드는 자동으로 실행됩니다.   
```kotlin
fun onCreate() {
    StrictMode.setVmPolicy(VmPolicy.Builder()
        // Other StrictMode checks that you've previously added.
        // ...
        .detectUnsafeIntentLaunch()
        .penaltyLog()
        // Consider also adding penaltyDeath()
        .build())
}
```   

### 인텐트를 더 책임감 있게 사용
안전하지 않은 인텐트 실행과 StrictMode 위반을 방지하기 위해 다음 best practices를 따르세요.   

인텐트안에서 필수적인 extras만 복사하고 
