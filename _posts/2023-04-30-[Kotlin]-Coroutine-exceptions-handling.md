---
title:  "[Kotlin] Coroutine exceptions handling"
excerpt: "Coroutine exceptions handling"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Coroutine-exceptions-handling/

toc: true
toc_sticky: true

date: 2023-04-30
last_modified_at: 2023-04-30
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/exception-handling.html)

<br>

예외 처리와 예외 취소에 대해 다룹니다. 취소된 코루틴이 중단 지점에서 CancellationException을 발생하는 것을 알고 있습니다. 그리고 코루틴 조직(machinery)에서 무시된다는 것도요. 취소 또는 같은 코루틴의 다수의 자식이 예외를 던지는 동안 예외가 발생하면 무슨 일이 생기는지 알아봅니다.   


## Exception propagation   
코루틴 빌더는 두 가지 종류가 있습니다. 예외를 자동으로 전파하거나 (launch 그리고 actor) 예외를 유저에게 부과하는 것 (async 그리고 produce). 이러한 빌더들이 root 코루틴을 생성하는데 사용한다면 (다른 코루틴의 자식이 아닌) 전자의 빌더들(launch, actor)은 예외를 uncaught 예외로 간주힙니다. (자바의 `Thread.uncaughtExceptionHandler`와 유사) 후자(async, produce)의 경우 마지막 예외를 처리하는 유저에게 의존합니다. 예를 들어, await, receive를 통해서요.   

GlobalScope를 사용하는 간단한 예시로 설명할 수 있습니다.   

```kotlin
@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val job = GlobalScope.launch { // root coroutine with launch
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException() // Will be printed to the console by Thread.defaultUncaughtExceptionHandler
    }
    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async { // root coroutine with async
        println("Throwing exception from async")
        throw ArithmeticException() // Nothing is printed, relying on user to call await
    }
    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```   

    Throwing exception from launch
    Exception in thread "DefaultDispatcher-worker-2 @coroutine#2" java.lang.IndexOutOfBoundsException
    Joined failed job
    Throwing exception from async
    Caught ArithmeticException   

## CoroutineExceptionHandler   
uncaught 예외를 콘솔에 출력하는 기본 행동을 커스텀할 수 있습니다. 루트 코루틴의 CoroutineExceptionHandler 컨텍스트 요소는 root 코루틴과 사용자 지정 예외 처리가 발생할 수 있는 모든 하위 항목을 위한 일반적인 catch 블록 처럼 사용됩니다. `Thread.uncaughtExceptionHandler`와 유사합니다. `CoroutineExceptionHandler`안의 예외로부터 recover할 수 없습니다. handler가 호출되었을 때, 코루틴은 이미 해당하는 예외와 함께 완료됩니다. 보통 핸들러는 예외를 로깅, 에러 메세지 보여주기, 종료하거나 앱을 재실행하기 위해 사용됩니다.   

`CoroutineExceptionHandler`는 uncaught 예외(다른 곳에서 처리되지 않은)에서만 발생합니다. 특히, 모든 자식 코루틴은 (다른 Job의 컨텍스트에서 생성된) 그들의 예외를 부모 코루틴에 위임합니다. 부모 코루틴은 또 그들의 부모 코루틴에 예외를 위임하고 결국 루트 코루틴까지 위임됩니다. 따라서 자식 코루틴에 설치된 `CoroutineExceptionHandler`는 결코 사용되지 않습니다. 추가로 async 빌더는 항상 모든 예외를 catch하고 결과를 Deffered 객체로 나타냅니다. 따라서 `CoroutineExceptionHandler`는 아무 효과가 없습니다.   

```kotlin
val handler = CoroutineExceptionHandler { _, exception -> 
    println("CoroutineExceptionHandler got $exception") 
}
val job = GlobalScope.launch(handler) { // root coroutine, running in GlobalScope
    throw AssertionError()
}
val deferred = GlobalScope.async(handler) { // also root, but async instead of launch
    throw ArithmeticException() // Nothing will be printed, relying on user to call deferred.await()
}
joinAll(job, deferred)
```   

    CoroutineExceptionHandler got java.lang.AssertionError   

## Cancellation and exceptions   
취소는 예외와 밀접한 관련이 있습니다. 코루틴은 내부적으로 취소를 위해 `CancellationException`을 사용합니다. 이러한 예외는 모든 handler로부터 무시되기 때문에, `catch` 블록으로 부터 얻어지는 추가 디버그 정보를 위해 사용되어야 합니다. Job.cancel을 이용하여 코루틴이 취소되었을 때, 종료되지만 부모를 종료하진 않습니다.   

```kotlin
val job = launch {
    val child = launch {
        try {
            delay(Long.MAX_VALUE)
        } finally {
            println("Child is cancelled")
        }
    }
    yield()
    println("Cancelling child")
    child.cancel()
    child.join()
    yield()
    println("Parent is not cancelled")
}
job.join()
```   

    Cancelling child
    Child is cancelled
    Parent is not cancelled   

만약 코루틴이 `CancellationException`외의 다른 예외를 마주한다면, 해당 예외와 함께 부모를 취소합니다. 이러한 행동은 재정의될 수 없고 구조화된 동시성을 위한 안정적인 코루틴 계층을 제공하기 위해 사용됩니다. CoroutineExceptionHandler 실행은 자식 코루틴들을 위한 것이 아닙니다.   

모든 자식 코루틴이 종료되었을 때, 오리지널 예외는 부모에 의해 처리됩니다.   

```kotlin
val handler = CoroutineExceptionHandler { _, exception -> 
    println("CoroutineExceptionHandler got $exception") 
}
val job = GlobalScope.launch(handler) {
    launch { // the first child
        try {
            delay(Long.MAX_VALUE)
        } finally {
            withContext(NonCancellable) {
                println("Children are cancelled, but exception is not handled until all children terminate")
                delay(100)
                println("The first child finished its non cancellable block")
            }
        }
    }
    launch { // the second child
        delay(10)
        println("Second child throws an exception")
        throw ArithmeticException()
    }
}
job.join()
```   

    Second child throws an exception
    Children are cancelled, but exception is not handled until all children terminate
    The first child finished its non cancellable block
    CoroutineExceptionHandler got java.lang.ArithmeticException   

## Exceptions aggregation   
다수의 코루틴이 예외가 발생하여 실패했을 때, 일반적인 규칙은 "첫 번째 예외가 이긴다" 입니다. 따라서 첫 번째 예외가 처리됩니다. 모든 추가적인 예외는 첫 예외의 억제된 것으로 추가됩니다.   

```kotlin
import kotlinx.coroutines.*
import java.io.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("CoroutineExceptionHandler got $exception with suppressed ${exception.suppressed.contentToString()}")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE) // it gets cancelled when another sibling fails with IOException
            } finally {
                throw ArithmeticException() // the second exception
            }
        }
        launch {
            delay(100)
            throw IOException() // the first exception
        }
        delay(Long.MAX_VALUE)
    }
    job.join()  
}
```   

    CoroutineExceptionHandler got java.io.IOException with suppressed [java.lang.ArithmeticException]   

취소 예외는 기본적으로 투명하며 래핑(wrap)되지 않습니다.   
```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
    println("CoroutineExceptionHandler got $exception")
}
val job = GlobalScope.launch(handler) {
    val inner = launch { // all this stack of coroutines will get cancelled
        launch {
            launch {
                throw IOException() // the original exception
            }
        }
    }
    try {
        inner.join()
    } catch (e: CancellationException) {
        println("Rethrowing CancellationException with original cause")
        throw e // cancellation exception is rethrown, yet the original IOException gets to the handler  
    }
}
job.join()
```   

    Rethrowing CancellationException with original cause
    CoroutineExceptionHandler got java.io.IOException   

## Supervision   
이전에 공부했듯이, 취소는 코루틴의 전체 계층을 통해 전파되는 양방향 관계이다. 단방향 취소가 필요한 경우를 살펴보자.   

좋은 예시는 UI 컴포넌트 안에 job이 정의된 경우이다. UI 자식 작업 중 하나가 실패해도 전체 UI 컴포넌트를 취소할 필요가 없으나 UI 컴포넌트가 파괴되고(job도 취소됨), 그러면 모든 자식 job의 결과는 필요없기 때문에 전부 취소해야한다.   

다른 예시는 서버 작업이 다른 여러 개의 자식 job을 만들고 그들의 실행을 감독, 실패를 추적하고 실패된 것들을 다시 시작할 때이다.   

### Supervision job   
SupervisorJob은 이러한 목적을 위해 사용됩니다. 아래로만 전파되는 예외를 가진 일반적인 Job과 유사합니다.   

```kotlin
val supervisor = SupervisorJob()
with(CoroutineScope(coroutineContext + supervisor)) {
    // launch the first child -- its exception is ignored for this example (don't do this in practice!)
    val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
        println("The first child is failing")
        throw AssertionError("The first child is cancelled")
    }
    // launch the second child
    val secondChild = launch {
        firstChild.join()
        // Cancellation of the first child is not propagated to the second child
        println("The first child is cancelled: ${firstChild.isCancelled}, but the second one is still active")
        try {
            delay(Long.MAX_VALUE)
        } finally {
            // But cancellation of the supervisor is propagated
            println("The second child is cancelled because the supervisor was cancelled")
        }
    }
    // wait until the first child fails & completes
    firstChild.join()
    println("Cancelling the supervisor")
    supervisor.cancel()
    secondChild.join()
}
```   

    The first child is failing
    The first child is cancelled: true, but the second one is still active
    Cancelling the supervisor
    The second child is cancelled because the supervisor was cancelled   

### Supervision scope   
coroutineScope대신 supervisorScope를 scoped concurrency를 위해 사용할 수 있습니다. 취소를 단방향으로만 전달하고 supervisorScope가 실패한 경우에만 모든 자식들을 취소합니다. 또한 coroutineScope처럼 모든 자식이 완료될때까지 기다립니다.   
```kotlin
try {
    supervisorScope {
        val child = launch {
            try {
                println("The child is sleeping")
                delay(Long.MAX_VALUE)
            } finally {
                println("The child is cancelled")
            }
        }
        // Give our child a chance to execute and print using yield 
        yield()
        println("Throwing an exception from the scope")
        throw AssertionError()
    }
} catch(e: AssertionError) {
    println("Caught an assertion error")
}
```   

    The child is sleeping
    Throwing an exception from the scope
    The child is cancelled
    Caught an assertion error   

#### Exceptions in supervised coroutines   
regular와 supervisor job의 또다른 중요한 차이점은 예외 핸들링이다. 모든 자식은 예외 핸들링 메커니즘을 통해 예외를 스스로 처리한다는 것입니다. 이런 차이점은 자식의 실패가 부모로 전파되지 않는다는 사실에서 옵니다. supervisorScope 안에서 직접적으로 실행된 코루틴은 CoroutineExceptionHandler를 **사용해야만 합니다.** 이런 CoroutineExceptionHandler는 그들의 범위에 설치되며 루트 코루틴과 같은 동작을 수행합니다.   

```kotlin
val handler = CoroutineExceptionHandler { _, exception -> 
    println("CoroutineExceptionHandler got $exception") 
}
supervisorScope {
    val child = launch(handler) {
        println("The child throws an exception")
        throw AssertionError()
    }
    println("The scope is completing")
}
println("The scope is completed")
```   

    The scope is completing
    The child throws an exception
    CoroutineExceptionHandler got java.lang.AssertionError
    The scope is completed
