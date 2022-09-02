---
title: "[Android] Activity (작성중)"
excerpt: "Activity 기본 개념"

categories:
  - Android
tags:
  - []

permalink: /android/Activity-Basic/

toc: true
toc_sticky: true

date: 2022-09-02
last_modified_at: 2022-09-02
---

> 안드로이드 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 내용이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://developer.android.com/guide/components/activities/intro-activities)

`Activity` 클래스는 Android 앱의 중요한 컴포넌트입니다. `Activity`가 실행되고 합쳐지는 방법은 플랫폼 어플리케이션 모델의 근본적인 부분입니다. `main()` 메서드에서 앱이 실행되는 프로그램과는 달리, 안드로이드 시스템은 `Activity` 인스턴스 안에서 특정한 생명주기에 맞는 callback 메서드를 실행함으로써 코드를 시작합니다.

<br>

이 문서는 Activities 개념을 소개하고 어떻게 동작하는지에 대한 가벼운 가이드를 제공합니다.


<br>

## The concept of activities
모바일 앱 환경은 데스크톱 앱 환경과 달리 사용자와 앱 상호작용이 항상 동일한 위치에서 시작되지 않습니다. 대신 사용자는 앱을 다른 화면에서 시작할 수 있습니다. 예를 들어 홈 화면에서 이메일 앱을 열면 이메일 목록이 표시될 수 있고 반대로 SNS 앱을 사용하고 있는 상태에서 이메일 앱을 실행하면 이메일을 작성 화면으로 바로 이동할 수 있습니다.

<br>

`Activity` 클래스는 이 패러다임이 가능하도록 설계되었습니다. A앱이 B앱을 작동시켰을 때, A앱은 B앱의 전체를 호출하지 않고 B앱의 activity를 호출합니다. 이런 방식으로 activity는 앱과 사용자의 상호작용을 위한 진입점 역할을 합니다. activity는 Activity 클래스의 서브클래스로 구현됩니다.   

<br>

activity는 UI를 그리는 window를 제공합니다. window는 일반적으로 화면 전체를 채우지만 화면보다 작고 다른 window 위에 떠 있을 수 있습니다. 일반적으로 하나의 activity는 하나의 화면을 구현합니다. 예를 들어 activity 중 하나는 환경설정 화면을 다른 activity는 사진 선택 화면을 구현할 수 있습니다.

<br>

대부분의 앱은 여러 화면을 포함하고 있습니다. 즉, 여러 activity로 구성됩니다. 일반적으로 *기본 activity*가 지정되며 기본 activity는 사용자가 앱을 실행할 때 표시되는 첫 번째 화면입니다. 각 activity는 또 다른 activity를 실행할 수 있습니다. 예를 들어 기본 activity는 받은 편지함을 표시하고 이메일 작성 및 이메일 상세보기 화면(activity)을 실행할 수 있습니다.

<br>

activity는 앱의 일관된 사용자 환경을 형성하기 위해 함께 작동하지만, 각 activity는 다른 activity에 느슨하게 결합됩니다. 일반적으로 activity 간에는 최소한의 종속성만 있습니다. 실제로 activity는 흔히 다른 앱의 activity를 시작할 수 있습니다. 예를 들어, 브라우저 앱은 SNS 앱의 공유 activity를 실행할 수 있습니다.

<br>

activity를 사용하려면 앱의 manifest에 activity 관련 정보를 등록하고 activity 생명 주기를 적절히 관리해야합니다. 이 문서의 나머지 부분에는 이러한 주제를 소개합니다. 

<br>

## manifest 구성
manifest에 activity 및 관련된 특정 속성을 선언해야 앱에서 activity를 사용할 수 있습니다. 

<br>

### Activity 선언
Activity를 선언하려면 manifest 파일을 열고 `<activity>` 요소 `<application>` 요소의 하위 요소로 추가해야 합니다. 예를 들면 다음과 같습니다.
```
    <manifest ... >
      <application ... >
          <activity android:name=".ExampleActivity" />
          ...
      </application ... >
      ...
    </manifest >
    
```
이 요소의 유일한 필수 속성은 activity의 클래스 이름을 지정하는 android:name입니다. 또한 라벨, 아이콘 또는 UI 테마와 같은 activity 특성을 정의하는 속성도 추가할 수 있습니다. 이러한 속성 및 기타 속성에 관한 자세한 내용은 `<activity>` 요소 참조 문서에서 확인할 수 있습니다.

<br>

> 참고: 앱을 게시한 후에는 활동 이름을 변경해서는 안 됩니다. 활동 이름을 변경하면 앱 바로가기와 같은 일부 기능이 작동하지 않을 수 있습니다. 게시 후 피해야 할 변경사항에 관한 자세한 내용은 변경할 수 없는 사항을 참조하세요.

<br>

### 인텐트 필터 선언
인텐트 필터는 Android 플랫폼의 매우 강력한 기능입니다. 인턴트 필터는 *명시적* 요청 뿐만 아니라 *암시적* 요청을 기반으로도 activity를 실행하는 기능을 제공합니다. 예를 들어 명시적 요청은 'Gmail 앱에서 이메일 보내기 activity를 시작'하도록 시스템에 지시할 수 있습니다. 이와 반대로 암시적 요청은 '작업을 할 수 있는 activity으로 이메일 보내기 화면을 시작'하도록 시스템에 지시합니다. 시스템 UI 에서 사용자에게 작업을 실행할 때 어떤 앱을 사용할지 묻는 메시지가 표시되면 바로 인텐트 필터가 작동한 것입니다.

<br>

`<activity>`요소에서 `<intent-filter>` 속성을 선언함으로써 이 기능을 활용할 수 있습니다. 이 요소의 정의에는 `<activity>`요소와 선택적으로 `<category>`요소 또는 `<data>` 요소가 포함됩니다. 이러한 요소를 결합하여 activity가 응답할 수 있는 인텐트 유형을 지정할 수 있습니다. 예를 들어 다음 코드 스니펫은 텍스트 데이터를 전송하고 다른 activity들의 요청을 수신하는 activity를 구성하는 방법을 보여줍니다.

```
    <activity android:name=".ExampleActivity" android:icon="@drawable/app_icon">
        <intent-filter>
            <action android:name="android.intent.action.SEND" />
            <category android:name="android.intent.category.DEFAULT" />
            <data android:mimeType="text/plain" />
        </intent-filter>
    </activity>
```

<br>

예시에서 `<action>` 요소는 activity가 데이터를 전송하도록 지정합니다. `<category>` 요소를 `DEFAULT`로 선언하면 activity가 실행 요청을 수신할 수 있습니다. `<data>` 요소는 activity가 전송할 수 있는 데이터 유형을 지정합니다. 






