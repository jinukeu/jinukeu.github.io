---
title:  "[Kotlin] Coroutines basics"
excerpt: "코틀린 기초에 대해 정리"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Coroutines-basics/

toc: true
toc_sticky: true

date: 2022-08-04
last_modified_at: 2023-02-08
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/coroutines-basics.html)

<br>

*코루틴*은 suspend 가능한 computation 인스턴스 입니다. 코루틴 block 안의 코드가 그 외의 코드와 동시에 작동한다는 점에서 개념적으로는 thread와  유사합니다. 그러나, 코루틴은 특정 스레드에 묶여 있지 않습니다. 예를 들어 코루틴은 A 스레드에서 중단(suspend)되고 B 스레드에서 재개될 수 있습니다.   

코루틴은 경량 스레드라고 생각될 수 있습니다. 하지만 실제로 사용할 때 스레드와 중요한 차이점이 많이 있습니다. 

```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { // launch a new coroutine and continue
        delay(1000L) // non-blocking delay for 1 second (default time unit is ms)
        println("World!") // print after delay
    }
    println("Hello") // main coroutine continues while a previous one is delayed
}
```

    Hello
    World!

<br>

`launch`는 coroutine builder입니다.   
코드의 나머지 부분과 동시에 독립적으로 작동하는 새로운 코루틴을 실행합니다. 독립적으로 작동하기 때문에 `Hello`가 먼저 출력됩니다.

`delay`는 *suspending function*입니다. 코루틴을 특정 시간동안 중단시킵니다.   
코루틴을 중단시켜도 스레드는 *block*되지 않습니다. 다른 코루틴의 실행을 허용하고 실행된 코루틴이 해당 스레드를 사용합니다.

`runBlocking`은 coroutine builder입니다.   
runBlocking안에 있는 코루틴 코드를 non-coroutine world인 `fun main()`와연결합니다.
또한 { } 안은 `CoroutineScope`입니다.   
`launch`의 경우 `CoroutineScope`안에서만 선언할 수 있습니다.   

`runBlocking` 이름은 `runBlocking`을 실행하는 스레드가 `runBlocking`이 호출되는 기간 동안 *block*됨을 의미합니다. (`runBlocking { ... }`안의 모든 코루틴이 실행 완료될 때까지 *block* 됩니다.) `runBlocking`은 보통 application의 최상단에서 사용되고 실제 코드 안에서는 거의 사용하지 않는 것을 확인할 수 있습니다. 스레드는 expensive resources이고 이러한 스레드들을 blocking하는 것은 비효율적이고 바람직하지 않기 때문입니다. 

<br>

## Structured concurrency
새로운 코루틴은 특정한 CoroutineScope안에서 실행되어야 한다는 **structured concurrency** 원칙을 따릅니다. 특정한 CoroutineScope는 코루틴의 생명주기의 범위를 정합니다. 위 예제 코드에서 runBlocking이 runBlocking에 해당하는 범위를 지정한 것을 알 수 있습니다. 따라서 위 예제 코드는 `World!`가 출력된 이후에야 종료됩니다.   

실제 앱에서 많은 코루틴을 실행합니다. Structured concurrency는 코루틴이 lost, leak됨을 방지합니다. outer scope는 모든 자식 코루틴이 완료되기 전까지 완료될 수 없습니다. Structured concurrency는 또한 코드 에러가 적절하게 보고 되고 lost되지 않게합니다.

<br>

## Extract function refactoring
```kotlin
fun main() = runBlocking { // this: CoroutineScope
    launch { doWorld() }
    println("Hello")
}

// this is your first suspending function
suspend fun doWorld() {
    delay(1000L)
    println("World!")
}
```
launch안의 코드를 별개의 함수로 분리해봅시다. 일반적인 함수와 다르게 `suspend` modifier가 붙은 것을 확인할 수 있습니다. 이를 **suspending function**이라 부릅니다. suspending function은 일반적인 함수와 마찬가지로 코루틴 내부에서 사용할 수 있습니다. suspending function의 특징은 다른 suspending functions함수(`delay`)를 사용할 수 있다는 것입니다.      

<br>

## Scope builder
runBlocking 말고도 다른 coroutineScope builder를 통해 `coroutine scope`를 제공할 수 있습니다. 이러한 coroutineScope builder는 실행된 자식(children)이 완료될 때까지 종료되지 않습니다.   

`runBlocking`과 `coroutineScope`는 서로 자식 코루틴이 종료될 때까지 기다린다는 점에서 유사합니다. 차이점은 `runBlocking`은 종료될 때까지 현재 thread를 멈추게(block)하는 반면 `coroutineScope`는 단순히 중단(suspend)하며 block하지 않고 thread를 다른 usage로 release합니다. 또한 runBlocking은 일반 함수이고 coroutineScope는 suspending 함수입니다.

모든 `suspending function`에서 `coroutineScope`를 사용할 수 있습니다. 단, suspend function은 무조건 CoroutineScope안에서 실행되야합니다.   

<br>

## Scope builder and concurrency
여러 개의 동시 작업을 위해 `coroutineScope builder`를 `suspending function`에 사용할 수 있습니다. 동시에 발생하는 2개의 코루틴을 실행해봅시다.

```kotlin
// Sequentially executes doWorld followed by "Done"
fun main() = runBlocking {
    doWorld()
    println("Done")
}

// Concurrently executes both sections
suspend fun doWorld() = coroutineScope { // this: CoroutineScope
    launch {
        delay(2000L)
        println("World 2")
    }
    launch {
        delay(1000L)
        println("World 1")
    }
    println("Hello")
}
```
    Hello
    World 1
    World 2
    Done
<br>

`launch { ... }`안의 코드는 *동시에* 실행됩니다. 1초 후에 `World 1`을 출력하고 2초 후에 `World 2`를 출력합니다.  `doWorld` 안의 coroutineScope는 두 개의 `launch`가 종료된 이후에 끝납니다. 따라서 `doWorld`가 return(종료)된 이후에 `Done`이 출력됩니다.

> coroutineScope 자체가 코루틴을 실행해주는거라 생각했는데 아니었다. 정말 **범위**만 지정해주는구나 ... 예상 했던 실행 결과는 Hello -> Done -> World 1 -> World 2 였다.

<br>


## An explicit job
`launch` coroutine builder는 `Job` 객체를 리턴합니다.
`Job`객체는 실행된 coroutine을 다룰 수 있으며 coroutine의 종료를 기다릴 수 있습니다.

```kotlin
val job = launch { // launch a new coroutine and keep a reference to its Job
    delay(1000L)
    println("World!")
}
println("Hello")
job.join() // wait until child coroutine completes
println("Done")
```

    Hello
    World!
    Done

<br>

## Coroutines are light-weight
코루틴은 JVM threads보다 자원을 덜 사용합니다. **`thread`를 사용하여 JVM의 가용 메모리를 고갈시키는 코드는 `코루틴`을 사용하여 리소스 제한 없이 실행될 수 있습니다.** 즉, 스레드를 사용하면 메모리 부족이 생기는 코드를 코루틴에서는 사용할 수 있습니다.   
다음 코드는 100000개의 코루틴을 실행하지만 메모리를 거의 사용하지 않습니다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    repeat(100_000) { // launch a lot of coroutines
        launch {
            delay(5000L)
            print(".")
        }
    }
}
```   

위 코드를 threads를 사용한다면 (`runBlocking`을 지우고 `launch`를 `thread`로 바꾸고 `delay` 대신 `Thread.sleep`를 사용하는 방식으로) out-of-memory error가 발생합니다.
