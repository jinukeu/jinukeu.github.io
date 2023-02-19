---
title:  "[Kotlin] Cancellation and timeouts"
excerpt: "Cancellation and timeouts"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Cancellation-and-timeouts/

toc: true
toc_sticky: true

date: 2023-02-18
last_modified_at: 2023-02-18
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/composing-suspending-functions.html)

<br>

## Sequential by default
두 개의 다른 곳에서 정의된 suspending functions가 있다고 가정해보자. 또한 suspending functions는 원격 서비스를 호출하거나 계산을 하는 유용한 동작을 수행한다고 가정하자. (1초간 딜레이를 하는 부분이 유용한 기능을 수행한다고 가정)   

```kotlin
suspend fun doSomethingUsefulOne(): Int {
    delay(1000L) // pretend we are doing something useful here
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    delay(1000L) // pretend we are doing something useful here, too
    return 29
}
```   

만약 두 개의 함수가 *순차적으로* 실행되기를 원한다면 - 먼저 `doSomethingUsefulOne`을 실행하고 그 다음에 `doSomethingUsefulTwo`를 수행한다. 그리고 결과 값을 합친다? 실제로는, 첫 번째 함수의 결과를 사용하여 두 번째 함수를 호출해야 하는지 또는 호출 방법을 결정할 때 이 작업을 수행합니다.   

일반 코드와 마찬가지로 코루틴의 코드는 기본적으로 *순차적*이기 때문에 일반 순차 호출을 사용합니다.   

```kotlin
val time = measureTimeMillis {
    val one = doSomethingUsefulOne()
    val two = doSomethingUsefulTwo()
    println("The answer is ${one + two}")
}
println("Completed in $time ms")
```
    The answer is 42
    Completed in 2017 ms    

## Concurrent using async
만약 `doSomethingUsefulOne`과 `doSomethingUsefulTwo`의 실행에 의존성이 없고 결과값을 더 빨리 얻고 싶다면, 두 개를 *동시에* 실행하면 됩니다. async를 사용하면 됩니다.   

개념적으로 async는 launch와 같습니다. 별개의 코루틴(light-weight 하고 모든 코루틴과 동시에 작업하는 thread)을 시작합니다. `launch`는 Job을 반환하고 결과 값이 없는 반면 `async`는 Deferred (light-weight non-blocking fucture, 결과값을 나중에 제공함을 약속) 를 반환합니다. deferred value에 `.await()`를 사용하여 최종값을 얻을 수 있습니다. 하지만 `Deferred` 역시 `Job`이기 때문에 필요하다면 취소할 수 있습니다.   

```kotlin
val time = measureTimeMillis {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    println("The answer is ${one.await() + two.await()}")
}
println("Completed in $time ms")
```
    The answer is 42
    Completed in 1017 ms   

두 개의 코루틴이 동시에 실행되기 때문에 2배 빠르다.    

## Lazily started async
async는 `start` 매개변수를 CoroutineStart.LAZY로 설정함으로써 lazy로 만들 수 있습니다. 이렇게하면 await에 의해 결과 값이 요구되거나 `Job`의 start 함수가 실행되었을 때만 코루틴이 시작됩니다.   
```kotlin
val time = measureTimeMillis {
    val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
    val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
    // some computation
    one.start() // start the first one
    two.start() // start the second one
    println("The answer is ${one.await() + two.await()}")
}
println("Completed in $time ms")
```   

    The answer is 42
    Completed in 1017 ms   

이전 예제와는 달리, 2개의 코루틴이 있지만 실행되지 않습니다. 프로그래머가 start를 호출함으로써 실행을 시작할 수 있습니다. 먼저 `one`을 시작하고 그 다음 `two`를 시작하고 각각의 코루틴이 끝날 때까지 기다립니다.   

만약 각각의 코루틴에 `start`를 호출하지 않고 `println`에서 `await`만 호출했다면, 이는 순차적인 행동을 하게 합니다. `await`은 코루틴 실행을 시작하고 끝날 때까지 기다리기 때문입니다. laziness의 use-case로 설계되지 않았습니다. `async(start = CoroutineStart.LAZY)`는 suspending functions를 포함한 computation 상황에서 표준 `lazy` 함수를 대체할 수 있습니다.    

아래 예시를 보면 쉽게 이해할 수 있습니다.

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async { doSomethingUsefulOne() }
        val two = async { doSomethingUsefulTwo() }
        
        delay(500L)
        println("무언가 중요한 작업 ...")
        
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    println("doSomethingUsefulOne Start")
    delay(1000L) // pretend we are doing something useful here
    println("doSomethingUsefulOne Finish")
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    println("doSomethingUsefulTwo Start")
    delay(1000L) // pretend we are doing something useful here, too
    println("doSomethingUsefulTwo Finish")
    return 29
}
```
    doSomethingUsefulOne Start
    doSomethingUsefulTwo Start
    무언가 중요한 작업 ...
    doSomethingUsefulOne Finish
    doSomethingUsefulTwo Finish
    The answer is 42
    Completed in 1044 ms

```kotlin
import kotlinx.coroutines.*
import kotlin.system.*

fun main() = runBlocking<Unit> {
    val time = measureTimeMillis {
        val one = async(start = CoroutineStart.LAZY) { doSomethingUsefulOne() }
    	val two = async(start = CoroutineStart.LAZY) { doSomethingUsefulTwo() }
        
        delay(500L)
        println("무언가 중요한 작업 ...")
        
        one.start() // start the first one
    	two.start() // start the second one
        println("The answer is ${one.await() + two.await()}")
    }
    println("Completed in $time ms")    
}

suspend fun doSomethingUsefulOne(): Int {
    println("doSomethingUsefulOne Start")
    delay(1000L) // pretend we are doing something useful here
    println("doSomethingUsefulOne Finish")
    return 13
}

suspend fun doSomethingUsefulTwo(): Int {
    println("doSomethingUsefulTwo Start")
    delay(1000L) // pretend we are doing something useful here, too
    println("doSomethingUsefulTwo Finish")
    return 29
}
```
    무언가 중요한 작업 ...
    doSomethingUsefulOne Start
    doSomethingUsefulTwo Start
    doSomethingUsefulOne Finish
    doSomethingUsefulTwo Finish
    The answer is 42
    Completed in 1526 ms

## Async-style functions
GlobalScope 참조를 사용하여 비동기 코루틴 빌더를 사용하여 `doSomethingUsfulOne`과 `doSomethingUsfulTwo`를 비동기적으로 호출하는 비동기식 함수를 정의하여 구조화된 동시성을 제거할 수 있습니다. 이러한 함수들을 "...Async" 처럼 접미사를 붙여 이름을 지어 비동기적인 계산을 시작하고 결과값을 얻기 위해서 deferred value를 사용해야함을 강조합니다.   

> GlobalScope는 사소한 일로도 역효과를 내는 섬세한 API입니다. (그 중 하나는 아래에서 설명합니다.) 따라서 GlopeScope를 사용하려면 `@OptIn(DelicateCoroutinesApi::class)`를 명시적으로 적어야합니다.   

```kotlin
// The result type of somethingUsefulOneAsync is Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulOneAsync() = GlobalScope.async {
    doSomethingUsefulOne()
}

// The result type of somethingUsefulTwoAsync is Deferred<Int>
@OptIn(DelicateCoroutinesApi::class)
fun somethingUsefulTwoAsync() = GlobalScope.async {
    doSomethingUsefulTwo()
}
```   

`xxxAsync` 함수들은 *suspending* 함수가 **아닙니다.** 어디에서든지 사용할 수 있습니다. 그러나, 보통 asnchronous 실행임을 암시합니다. (여기서는 *동시성*을 의미)   

코루틴 바깥에서 실행하는 예시입니다.   

```kotlin
// note that we don't have `runBlocking` to the right of `main` in this example
fun main() {
    val time = measureTimeMillis {
        // we can initiate async actions outside of a coroutine
        val one = somethingUsefulOneAsync()
        val two = somethingUsefulTwoAsync()
        // but waiting for a result must involve either suspending or blocking.
        // here we use `runBlocking { ... }` to block the main thread while waiting for the result
        runBlocking {
            println("The answer is ${one.await() + two.await()}")
        }
    }
    println("Completed in $time ms")
}
```   

> 비동기 함수가 있는 이 프로그래밍 스타일은 다른 프로그래밍 언어에서 인기 있는 스타일이기 때문에 설명용으로만 제공됩니다. 코틀린 코루틴과 함께 이 스타일을 사용하는 것은 아래에 설명된 이유로 **매우 권장하지 않습니다.**   

만약 `val one = somethingUsefulOneAsync()`와 `one.await()` 사이에 에러가 있고 프로그램이 exception을 발생시키고, 프로그램에 의해 수행되고 있던 작업은 중단된다고 가정하자. 일반적으로, global error-handler가 exception을 catch하고 개발자들을 위해 로그를 남기고 보고한다. 그렇지않으면 프로그램은 다른 작업을 계속할 수 있다. 하지만, 시작된 작업이 중단되었음에도 불구하고 `somethingUsefulOneAsync`는 여전히 백그라운드에서 동작한다. 이런 문제는 아래 섹션에서 나오는 것처럼 structured concurrency에서는 발생하지 않는다.   

## Structured concurrency with async
Concurrent using async 예제를 가져와서 함수를 추출해보자. 추출하려는 함수는 `doSomethingUsefulOne`과 `doSomethingUsefulTwo`를 동시에 수행하고 결과값들의 합을 리턴한다. async 코루틴 빌더가 CoroutineScope의 extension으로 정의되었기 때문에, async는 scope안에 있어야하고 scope는 coroutineScope 함수가 제공합니다.
```kotlin
suspend fun concurrentSum(): Int = coroutineScope {
    val one = async { doSomethingUsefulOne() }
    val two = async { doSomethingUsefulTwo() }
    one.await() + two.await()
}
```   

이렇게하면, 만약 `concurrentSum` 함수 내부에서 뭔가 잘못되어 exception이 발생하면 해당 범위에 있는 모든 실행 중인 코루틴은 취소됩니다.   

```kotlin
val time = measureTimeMillis {
    println("The answer is ${concurrentSum()}")
}
println("Completed in $time ms")
```   
    The answer is 42
    Completed in 1017 ms   

Cancellation은 항상 코루틴 계층으로 전파됩니다.   

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    try {
        failedConcurrentSum()
    } catch(e: ArithmeticException) {
        println("Computation failed with ArithmeticException")
    }
}

suspend fun failedConcurrentSum(): Int = coroutineScope {
    val one = async<Int> { 
        try {
            delay(Long.MAX_VALUE) // Emulates very long computation
            42
        } finally {
            println("First child was cancelled")
        }
    }
    val two = async<Int> { 
        println("Second child throws an exception")
        throw ArithmeticException()
    }
    one.await() + two.await()
}
```

    Second child throws an exception
    First child was cancelled
    Computation failed with ArithmeticException