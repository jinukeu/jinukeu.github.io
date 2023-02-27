---
title:  "[Kotlin] Coroutine context and dispatchers"
excerpt: "Coroutine context and dispatchers"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Coroutine-context-and-dispatchers/

toc: true
toc_sticky: true

date: 2022-08-19
last_modified_at: 2023-02-27
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)

<br>

코루틴은 코틀린 표준 라이브러리에 정의된 CoroutineContext type으로 표시되는 Context에서 실행됩니다.   

coroutine context는 다양한 요소들의 집합입니다. 주요 요소는 코루틴의 **Job**과 dispatcher입니다.

## Dispatchers and threads
coroutine context는 해당 코루틴이 어떤 thread(s)를 사용할 지 결정하는 *conroutine dispatcher*를 포함합니다. coroutine dispatcher는 코루틴 실행을 특정한 thread로 제한하거나 thread pool로 dispatch하거나 제한을 하지 않을 수 있습니다.   
launch나 async같은 모든 코루틴 빌더는 CoroutineContext 매개변수를 허용합니다. CoroutineContext는 새로운 코루틴과 다른 context 요소를 위한 dispatcher를 명시적으로 지정합니다.   
```kotlin
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
}
launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
    println("Default               : I'm working in thread ${Thread.currentThread().name}")
}
launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
    println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
}
```
    Unconfined            : I'm working in thread main @coroutine#3
    Default               : I'm working in thread DefaultDispatcher-worker-1 @coroutine#4
    main runBlocking      : I'm working in thread main @coroutine#2
    newSingleThreadContext: I'm working in thread MyOwnThread @coroutine#5

`launch {...}`은 parameters가 없기 때문에 상위 CoroutineScope의 context를 상속 받습니다. 위의 경우 main 스레드에서 동작하고 있는 main runBlocking의 context를 상속 받습니다.   

<br>

`Dispatchers.Unconfined`는 특별한 dispatcher입니다. `main` 스레드에서 실행되지만 사슬은 다른 메커니즘을 가지고 있습니다. 해당 내용은 나중에 설명하겠습니다.

<br>

default dispatcher는 명시적으로 지정된 dispatcher가 없을 때 사용됩니다. 이는 `Dispatchers.Default`이고 스레드의 공유된 백그라운드 풀을 사용합니다.

<br>

`newSingleThreadContext`는 코루틴이 실행되기 위한 스레드를 만듭니다. 전용 스레드는 아주 비싼 리소스입니다. 실제 application에서 released되고 더이상 쓸모없게 된다면 close 함수를 사용하거나 상위 레벨의 변수에 저장하고 재사용해야합니다.

<br>

## Unconfined vs confined dispatcher
`Dispatchers.Unconfined` 코루틴 dispatcher는 첫 번재 suspension 지점까지 코루틴을 caller 스레드 안에서 시작합니다. suspension 이후 suspending 함수에 의해 결정된 스레드의 코루틴을 재개합니다. unconfined dispatcher는 CPU 시간을 소비하거나 특정 스레드에 제한된 공유 데이터(UI 등)를 업데이트하지 않는 코루틴에 적합합니다.

<br>

반면 기본적으로 dispatcher는 outer CoroutineScope에서 상속 받습니다. runBlocking 코루틴을 위한 기본 dipatcher는 스레드 호출자에 제한됩니다.따라서 상속은 예측 가능한 FIFO 스케줄링으로 이 스레드에 실행을 제한하는 효과가 있습니다.
```kotlin
launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
    println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
    delay(500)
    println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
}
launch { // context of the parent, main runBlocking coroutine
    println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
    delay(1000)
    println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
}
```


    Unconfined      : I'm working in thread main
    main runBlocking: I'm working in thread main
    Unconfined      : After delay in thread kotlinx.coroutines.DefaultExecutor
    main runBlocking: After delay in thread main

그래서 `runBlocking {...}` 에서 상속받은 코루틴은 `main` 스레드에서 계속되지만, unconfined 스레드는 delay 함수가 사용하는 default executor 스레드에서 재개됩니다.   

> unconfined dispatcher는 특정한 corner case에서 도움이 되는 advanced machanism입니다. 코루틴 실행 이후에 dispatching하는 것은 필요하지 않고 예상치 못한 side-effects를 발생합니다. 코루틴의 몇몇 작업은 즉시 실행되야하기 때문입니다. unconfined dispatcher는 일반적인 코드에서 사용되선 안됩니다.

<br>

## Debugging coroutines and threads
코루틴은 하나의 스레드에서 suspend되었다가 다른 스레드에서 재개될 수 있습니다. 특별한 tooling을 가지고 있지 않다면, 심지어 single-threaded dispatcher에서도 코루틴이 무엇을 하고 있는지, 어디에 있는지 파악하기 어렵습니다.   

### Debugging with IDEA
코틀린 플러그인의 Coroutine Debugger는 코루틴 디버깅을 쉽게 만듭니다.   

**Debug** tool window는 **Coroutines** 탭을 포함하고 있습니다. 해당 탭에서 실행 중이고 suspend된 코루틴의 정보를 찾을 수 있습니다. 코루틴들은 실행 중인 dispatcher에 그룹화되어 있습니다.   
![](https://kotlinlang.org/docs/images/coroutine-idea-debugging-1.png)   

coroutine debugger로 다음과 같은 작업을 할 수 있습니다.   

* 각 코루틴의 상태 체크
* running 그리고 suspend된 코루틴의 local, captured 값 확인
* coroutine creation stack 전부, 코루틴 안의 call stack 확인. stack은 변수의 모든 frames을 포함하고 있음. 이것들은 standard debugging 중에 유실될 수 있음.
* 각 코루틴의 상태와 stack에 대한 모든  보고서를 얻을 수 있다. **Coroutines** 탭을 우클릭한 후 **Get Coroutines Dump**를 클릭하면 된다.   

코루틴 디버깅을 시작하기 위해, breakpoints를 설정하고 application을 debug 모드로 실행해라.   

### Debugging using logging   
Coroutine Debugger 없이 스레드가 있는 application을 디버깅하는 다른 방법은 각각의 log 문에 thread name을 출력하는 것이다. 해당 기능은 logging이 되는 frameworks에서 모두 지원한다. coroutines를 사용할 때, 스레드 이름 만으로는 많은 정보를 제공하지 않습니다. 그래서 `kotlinx.coroutines`는 debugging facilties를 제공합니다.   

JVM 옵션에서 `-DKotlinx.coroutines.debug`와 함께 코드를 실행해보세요.
```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking<Unit> {
    val a = async {
        log("I'm computing a piece of the answer")
        6
    }
    val b = async {
        log("I'm computing another piece of the answer")
        7
    }
    log("The answer is ${a.await() * b.await()}")    
}
```   

`runBlocking`안의 메인 코루틴(#1)과 `a`, `b` deferred 값을 계산하는 2, 3번째 코루틴(#2, #3)이 있습니다. 이것들은 전부 `runBlocking`안의 context안에서 실행되고 main thread에 한정되어 있습니다. 출력 결과는 다음과 같습니다.   

    [main @coroutine#2] I'm computing a piece of the answer
    [main @coroutine#3] I'm computing another piece of the answer
    [main @coroutine#1] The answer is 42   

`log` 함수는 대괄호와 함께 thread 이름을 출력합니다. 현재 실행 중인 코루틴이 추가된 `main` 스레드 식별자를 확인할 수 있습니다. 이 식별자는 debugging mode가 켜져있을 때 모든 생성된 코루틴에 연속해서 할당됩니다.

## Jumping between threads
`-Dkotlinx.coroutines.debug` JVM 옵션과 함께 다음 코드를 실행하라   
```kotlin
newSingleThreadContext("Ctx1").use { ctx1 ->
    newSingleThreadContext("Ctx2").use { ctx2 ->
        runBlocking(ctx1) {
            log("Started in ctx1")
            withContext(ctx2) {
                log("Working in ctx2")
            }
            log("Back to ctx1")
        }
    }
}
```   
새로운 테크닉들을 보여주고 있습니다. 하나는 `runBlocking`이 명백하게 구체화된 context에서 사용하는 것과 `withContext`를 사용하여 같은 코루틴 내에 존재하면서 코루틴의 context를 바꾼 것입니다.   

    [Ctx1 @coroutine#1] Started in ctx1
    [Ctx2 @coroutine#1] Working in ctx2
    [Ctx1 @coroutine#1] Back to ctx1   

이 예제는 코틀린 표준 라이브러리의 `use` 함수를 사용했습니다. `use` 함수는 `newSingleThreadContext`로 만들어진 threads가 더이상 필요하지 않을 때 release합니다. 


## Children of a coroutine
다른 코루틴의 CoroutineScope안에서 새로운 코루틴이 실행되면 context를 CoroutineScope.coroutineContext를 통해 상속받고 새로운 코루틴의 Job은 부모 코루틴 Job의 자식이 됩니다. 부모 코루틴이 취소되면 모든 자식 역시 재귀적으로 취소됩니다.

<br>

그러나, 부모-자식 관계는 다음 두 경우에 의해 재정의될 수 있습니다.

1. 코루틴이 실행될 때(예를 들어, GlobalScope.launch) 다른 scope가 명백하게 지정되면 Job을 부모 scope로 부터 상속 받지 않습니다.
2. 다른 Job 객체가 새로운 코루틴의 context로 통과되면(아래 예시 참고) 부모 scope의 Job을 재정의합니다.

두 경우에 실행된 코루틴은 scope에 묶여있지 않습니다. 독립적으로 실행되고 작동합니다.

{% highlight kotlin %}
// launch a coroutine to process some kind of incoming request
val request = launch {
    // it spawns two other jobs
    launch(Job()) { 
        println("job1: I run in my own Job and execute independently!")
        delay(1000)
        println("job1: I am not affected by cancellation of the request")
    }
    // and the other inherits the parent context
    launch {
        delay(100)
        println("job2: I am a child of the request coroutine")
        delay(1000)
        println("job2: I will not execute this line if my parent request is cancelled")
    }
}
delay(500)
request.cancel() // cancel processing of the request
println("main: Who has survived request cancellation?")
delay(1000) // delay the main thread for a second to see what happens
{% endhighlight kotlin %}
    job1: I run in my own Job and execute independently!
    job2: I am a child of the request coroutine
    main: Who has survived request cancellation?
    job1: I am not affected by cancellation of the request

<br>

## Parental responsibilities
부모 코루틴은 항상 자식 코루틴이 전부 완료될 때까지 기다립니다. 부모 코루틴은 모든 자식 코루틴을 추적할 필요가 없고 자식들을 기다리기 위해 Job.join을 사용할 필요가 없습니다.   
{% highlight kotlin %}
// launch a coroutine to process some kind of incoming request
val request = launch {
    repeat(3) { i -> // launch a few children jobs
        launch  {
            delay((i + 1) * 200L) // variable delay 200ms, 400ms, 600ms
            println("Coroutine $i is done")
        }
    }
    println("request: I'm done and I don't explicitly join my children that are still active")
}
request.join() // wait for completion of the request, including all its children
println("Now processing of the request is complete")
{% endhighlight kotlin %}
    request: I'm done and I don't explicitly join my children that are still active
    Coroutine 0 is done
    Coroutine 1 is done
    Coroutine 2 is done
    Now processing of the request is complete

<br>

## Combining context elements
때때로 coroutine context에 여러개의 요소를 정의할 필요가 있습니다. 이때 + 연산자를 사용할 수 있습니다. 예를들어, dispatcher과 이름을 동시에 지정할 수 있습니다.
{% highlight kotlin %}
launch(Dispatchers.Default + CoroutineName("test")) {
    println("I'm working in thread ${Thread.currentThread().name}")
}
{% endhighlight kotlin %}

<br>

## Coroutine scope
application이 lifecycle object를 가지고 있지만 object는 코루틴이 아니라고 가정해봅시다. 예를 들어 Android application을 작성하고 있고 데이터를 가져오거나 업데이트하는 비동기 작업을 수행하기 위해 다양한 코루틴을 context안에서 실행하고 있습니다. 메모리 누출을 막기 위해 모든 코루틴들은 액티비티가 destoryed될 때 취소되야합니다. 물론 contexts와 jobs를 액티비티의 생명주기와 코루틴에 맞게 다루겠지만, **kotlinx.coroutines**는 CoroutineScope를 캡슐화하는 추상화를 제공합니다. 모든 코루틴 빌더가 코루틴의 확장으로 선언되므로 코루틴 scope에 대해 이미 잘 알고 있어야합니다.

<br>

액티비티의 생명주기와 얽매인 CoroutineScope 객체를 생성함으로써 코루틴의 생명주기를 관리할 수 있습니다. CoroutineScope() 또는 MainScope()의 factory functions를 이용하여 CoroutineScope를 생성할 수 있습니다. 전자의 경우 일반적인 목적의 scope이지만 후자의 경우 UI applications를 위한 scope를 만들며 Dispatchers.Main을 기본 dispatcher로 사용합니다.

{% highlight kotlin %}
class Activity {
    private val mainScope = MainScope()

    fun destroy() {
        mainScope.cancel()
    }
    // to be continued ...
{% endhighlight kotlin %}
이제 scope가 정의된 액티비티의 scope안에서 코루틴을 실행할 수 있습니다. 
{% highlight kotlin %}
// class Activity continues
    fun doSomething() {
        // launch ten coroutines for a demo, each working for a different time
        repeat(10) { i ->
            mainScope.launch {
                delay((i + 1) * 200L) // variable delay 200ms, 400ms, ... etc
                println("Coroutine $i is done")
            }
        }
    }
} // class Activity ends
{% endhighlight kotlin %}
main 함수에서 액티비티를 만들고 doSomething 함수를 호출합니다. 그리고 액티비티를 500ms 후에 destroy합니다. 이는 doSomething으로 시작된 모둔 코루틴을 취소합니다. 더 이상 메세지가 print되지 않음을 통해 확인할 수 있습니다.
{% highlight kotlin %}
val activity = Activity()
activity.doSomething() // run test function
println("Launched coroutines")
delay(500L) // delay for half a second
println("Destroying activity!")
activity.destroy() // cancels all coroutines
delay(1000) // visually confirm that they don't work
{% endhighlight kotlin %}
    Launched coroutines
    Coroutine 0 is done
    Coroutine 1 is done
    Destroying activity!
