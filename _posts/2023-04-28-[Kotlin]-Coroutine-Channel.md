---
title:  "[Kotlin] Coroutine Channel"
excerpt: "Coroutine Channel"

categories:
  - Kotlin
tags:
  - [코루틴]

permalink: /kotlin/Coroutine-Channel/

toc: true
toc_sticky: true

date: 2023-04-28
last_modified_at: 2023-04-28
---
> 코루틴 공식 문서를 번역하고 내용을 조금 변경하거나 내용을 추가한 게시글입니다. 잘못된 번역이 있을 수 있습니다.
> [참고한 공식 문서 바로가기](https://kotlinlang.org/docs/channels.html)

<br>

지연 값(Deferred values)을 사용하면 서로 다른 코루틴들이 손쉽게 하나의 값을 공유할 수 있습니다. 채널을 이용하면 서로 다른 코루틴들간에 데이터 스트림을 공유할 수 있습니다.   

## Channel basics   
Channel은 `BlockingQueue`와 개념적으로 아주 유사합니다. 차이점은 `put` 대신 suspending `send`를, `take` 대신 suspending `receive`를 사용합니다.   

```kotlin
val channel = Channel<Int>()
launch {
    // this might be heavy CPU-consuming computation or async logic, we'll just send five squares
    for (x in 1..5) channel.send(x * x)
}
// here we print five received integers:
repeat(5) { println(channel.receive()) }
println("Done!")
```   

    1
    4
    9
    16
    25
    Done!   

## Closing and iteration over channels   
Queue와는 달리 channel을 닫아 더 이상 요소가 오지 않음을 나타낼 수 있습니다. 일반적인 `for`문을 사용하면 channel로부터 값을 쉽게 받을 수 있습니다.   

개념적으로 close는 특수한 close 토큰을 채널로 보냅니다. 이터레이션은 close 토큰을 수신하자마자 멈춥니다. 따라서 close 전에 전송된 모든 요소가 수신된다는 보장이 있습니다.   

```kotlin
val channel = Channel<Int>()
launch {
    for (x in 1..5) channel.send(x * x)
    channel.close() // we're done sending
}
// here we print received values using `for` loop (until the channel is closed)
for (y in channel) println(y)
println("Done!")
```   

## Building channel producers   
코루틴이 일련의 요소를 생성하는 패턴은 꽤 흔합니다. **proudcer-consumer** 패턴은 종종 동시성 코드에서 찾을 수 있습니다. 이런 생성자를 채널을 매개변수로 받는 함수로 추상화할 수 있습니다.   

produce라는 코루틴 빌더로 이를 수행할 수 있습니다. 또한 consumeEach 익스텐션 함수를 사용해 for문을 대체할 수 있습니다.   

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun CoroutineScope.produceSquares(): ReceiveChannel<Int> = produce {
    for (x in 1..5) send(x * x)
}

fun main() = runBlocking {
    val squares = produceSquares()
    squares.consumeEach { println(it) }
    println("Done!")
}
```   

## Pipelines   
파이프라인은 하나의 코루틴이 무한대의 값 스트림을 생성하는 패턴입니다.   
```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```   

그리고 또 다른 코루틴이나 코루틴은 그 스트림을 소비하고, 약간의 처리를 하고, 다른 결과를 만들어냅니다. 아래 예제에서는 숫자를 제곱으로 만듭니다.   
```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```   
```kotlin
val numbers = produceNumbers() // produces integers from 1 and on
val squares = square(numbers) // squares integers
repeat(5) {
    println(squares.receive()) // print first five
}
println("Done!") // we are done
coroutineContext.cancelChildren() // cancel children coroutines
```   

## Prime numbers with pipeline   
코루틴 파이프라인을 사용하여 소수를 생성하는 파이프라인을 극단적으로 살펴보겠습니다. 우선 일련의 무한한 숫자를 생성합니다.    
```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```   

다음 파이프라인 단계에서는 들어오는 숫자 스트림을 필터링하여 지정된 소수로 구분되는 모든 숫자를 제거합니다:
```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```   

cancelChildren() 확장 함수를 사용하여 모든 자식 코루틴들을 종료시킬 수 있습니다.

```kotlin
var cur = numbersFrom(2)
repeat(10) {
    val prime = cur.receive()
    println(prime)
    cur = filter(cur, prime)
}
coroutineContext.cancelChildren() // cancel all children to let main finish
```   

iterator를 사용하여 동일한 동작을 수행할 수 있습니다. channel을 사용한 파이프라인의 장점은 Dispatchers.Default 컨텍스트에서 작동한다면 여러 개의 CPU 코어를 사용할 수 있기 때문입니다.   


