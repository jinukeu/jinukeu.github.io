---
title: "[Android] MVVM 패턴 박살내기 - 1"
excerpt: "안드로이드 MVVM 패턴의 기본 개념, 사용 이유, AAC"

categories:
  - Android
tags:
  - []

permalink: /android/MVVM-패턴-박살내기-1/

toc: true
toc_sticky: true

date: 2022-08-27
last_modified_at: 2022-08-27
---
## MVVM 패턴이란?
코드를 ***M***odel, ***V***iew, ***V***iew***M***odel의 역할에 맞게 분리하서 작성하는 디자인 패턴


<br>

## 왜 MVVM을 사용할까?
### 1. 사용안하면 뭐 어쩔건데?
`MVVM 패턴`을 사용하지 않고 `액티비티/프래그먼트`에 모든 코드를 넣게 되면 어떻게 될까요?   
1. `액티비티/프래그먼트` 코드가 매우 복잡해짐
2. 스파게티 코드가 될 확률이 높아짐
3. 유지보수 & 테스트가 어려워짐
4. 다른 사람이 코드를 이해하기 어려워짐 (본인 포함)

이런 단점이 있기 때문에 코드를 분리해서 작성하는 `MVVM 패턴`을 사용한다.   

<br>

### 2. MVVM말고 다른 패턴(MVC, MVP)를 사용하면 안되나요?
`MVC`의 경우 Model, View, Controller로 코드를 관리   
하지만 View와 Model 사이의 **의존성**이 높다는 단점이 있다.   

`MVP`의 경우 Model, View, Presenter로 코드를 관리   
하지만 View와 Presenter 사이의 **의존성**이 높다는 단점이 있다.  

<br>

***아니 ... 의존성은 또 뭐고 그게 높으면 뭐 어떻게 되는데요???***  ->  [이거 ... 보고 오시죠!](https://kotlinworld.com/64)   

-> MVVM > (MVC, MVP)

<br>

### 3. 협업이 쉬워짐
(다른 디자인 패턴도 마찬가지이지만) 팀원들이 전부 `MVVM 패턴`을 적용하고 이해하고 있으면 개발 커뮤니케이션이 수월해진다!

<br>

### 4. 그리고 ...
* 각각의 부분이 독립적 -> 유닛 테스트 용이
* 모듈화가 잘 되어 있다면 내장 DB를 통채로 변경하는게 비교적 쉬움 ex) realm -> room
* `ViewModel`과 `View`가 1:n 연결이 가능 -> 뷰모델 재활용 가능
* `View`가 데이터를 실시간으로 관찰하고 자동으로 UI 갱신

<br>

## 자세히 알아보자!
### 구조
> **Model**   
> * 어플리케이션에서 사용하는 데이터와 그 데이터를 처리하는 부분   

> **View**   
> * 액티비티/프래그먼트, 사용자에게 보여지는 UI   
> * 사용자와 상호작용   
> * `ViewModel`의 데이터를 관찰하여 UI 갱신

> **ViewModel**   
> * `Model`과 `View`를 이어주는 역할. `Model`의 데이터를 가공하여 `View`에게 `data stream`을 제공

<br>

### 어떻게 구현?
AAC(Android Architecture Components)를 사용하면 비교적 쉽게 구현 가능
> **AAC**
> * 강력하고 테스트와 유지관리가 쉬운 앱을 디자인하도록 돕는 라이브러리 모음

<br>

주로 안드로이드 `MVVM 패턴`에는 `AAC`의 `LiveData`와 `ViewModel`을 사용한다.

<br>

## 주의사항
### 1. AAC의 ViewModel != MVVM의 ViewModel
어 뭐야? 둘이 같은거 아닌가요??? -> 아뇨 ... 이름만 같아요 ... ㅋㅋㅋ   
> **AAC ViewModel**   
> * `ViewModel` 클래스는 수명 주기를 고려하여 UI 관련 데이터를 저장하고 관리하도록 설계되었습니다. `ViewModel` 클래스를 사용하면 화면 회전과 같이 구성을 변경할 때도 데이터를 유지할 수 있습니다. - [공식 문서](https://developer.android.com/topic/libraries/architecture/viewmodel)

> **MVVM ViewModel**   
> * `Model`과 `View`를 이어주는 역할. `Model`의 데이터를 가공하여 `View`에게 `data stream`을 제공

<br>

그러면 어떻게 `AAC ViewModel`을 `MVVM ViewModel`처럼 사용하느냐...! 단순히 ***AAC ViewModel이 가지고 있는 데이터를 관찰가능하게 해주고 View는 데이터바인딩으로 그것을 구독하고 있으면 된다!*** 이렇게 하면 데이터를 수명 주기를 고려하여 관리할 수 있으므로 더 좋다!   

<br>

하지만 단점도 있다.   
`AAC ViewModel`은 `액티비티`에 단 하나만 존재할 수 있다. 예를 들어 `MainViewModel`을 한번 생성하면 그 `액티비티`에서 `MainViewModel`을 여러번 생성해도 하나의 객체만 사용된다. `MVVM 패턴`은 `View`와 `ViewModel`이 1:n 관계인데, `AAC ViewModel`은 이를 지킬 수 없다. 또한 구글은 `View`와 `AAC ViewModel`을 1:1 관계로 사용하고 `ViewModel`안에 여러 `Model`과 `LiveData`를 사용하도록 권장한다. 이 역시 `MVVM` 원칙을 위반한다.

<br>

### LiveData의 한계
우선 `LiveData`가 뭔지 알아보자.   
`LiveData`란 `LiveCycle`을 알고 있는 `DataType`이라 생각하면 된다. ***LiveData는 Observer 객체를 이용하여 데이터의 변화를 관찰하며 UI를 업데이트 합니다.*** 어라? `MVVM`에서 `View`의 역할은 ***ViewModel의 데이터를 관찰하여 UI를 갱신하는건데*** ... 아! `ViewModel`의 데이터를 `LiveData`로 만들면 되겠네!   
  
#### LiveData vs MutableLiveData
`LiveData`는 `read-only`이고 `MutableLiveData`는 `LiveData`를 수정할 수 있다.   
`ViewModel`를 캡슐화 하기위해 외부로 노출되는 값인 `read-only` `LiveData`를 `public`으로 내부에서는 데이터를 편집할 수 있는 `MutableLiveData`를 `private`로 만들자!   
> **캡슐화를 하는 이유**   
데이터 내용을 변경할 수 있는 기능을 외부로 노출하면 디버그하기 어렵고 데이터의 내용이 어디서 왔는지 추론하기 어려워지기 때문 [다른 이유는 링크 참고](https://velog.io/@chawani47/%EA%B0%9D%EC%B2%B4%EC%A7%80%ED%96%A5%EC%97%90%EC%84%9C-%EC%BA%A1%EC%8A%90%ED%99%94%EA%B0%80-%EC%A4%91%EC%9A%94%ED%95%9C-%EC%9D%B4%EC%9C%A0)

<br>

그래서 `LiveData`의 한계가 뭔가요?   
그건 다음 포스팅에 계속 ...






