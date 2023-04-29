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

## Fan-out   
여러 코루틴이 동일한 채널에서 수신되어 작업을 서로 분배할 수 있습니다. 일정 기간을 두고 숫자를 생산하는 생산자 코루틴을 시작합시다.   

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```   

다수의 프로세서 코루틴을 만들 수 있습니다. 예시에서, id와 수신한 숫자를 출력합니다.   
```kotlin
fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }
}
```   

5개의 프로세서를 실행하고 대략 1초 동안 수행하도록 만들어봅시다.   
```kotlin
val producer = produceNumbers()
repeat(5) { launchProcessor(it, producer) }
delay(950)
producer.cancel() // cancel producer coroutine and thus kill them all
```   

출력은 다음과 유사할겁니다.   

    Processor #2 received 1
    Processor #4 received 2
    Processor #0 received 3
    Processor #1 received 4
    Processor #3 received 5
    Processor #2 received 6
    Processor #4 received 7
    Processor #0 received 8
    Processor #1 received 9
    Processor #3 received 10    

생산자 코루틴을 취소하면 채널이 닫히므로 프로세서 코루틴이 수행하는 채널을 통한 반복이 종료됩니다.   

또한 `launchProcessor` 코드 안에서 fan-out을 수행하는 for문과 함께 channel을 어떻게 이터레이트 했는지에 대해 볼 필요가 있습니다. `consumeEach`와는 다르게 이 `for`문은 완벽하게 여러 코루틴으로 부터 안전합니다. 하나의 프로세서 코루틴이 실패해도 다른 코루틴들은 여전히 채널로부터 수신하는 상태입니다. 반면 `consumeEach`로 작성된 프로세서는 항상 정상 또는 비정상적인 완료 시 기본 채널을 소비(취소)합니다.   

## Fan-in   
다수의 코루틴은 같은 채널에 send할 수 있습니다. 예를 들어 문자열 채널과 일정 delay를 가지고 특정 문자열을 채널로 send하는 suspending 함수가 있다고 가정합시다.   
```kotlin
suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```   

이제, 문자열을 전송하는 두 개의 코루틴을 실행해봅시다. (이 예시에서는 메인 코루틴의 자식으로서 메인 스레드의 컨텍스트에서 시작합니다.)     
```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    val channel = Channel<String>()
    launch { sendString(channel, "foo", 200L) }
    launch { sendString(channel, "BAR!", 500L) }
    repeat(6) { // receive first six
        println(channel.receive())
    }
    coroutineContext.cancelChildren() // cancel all children to let main finish
}

suspend fun sendString(channel: SendChannel<String>, s: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}
```   

    foo
    foo
    BAR!
    foo
    foo
    BAR!   

## Buffered channels   
지금까지의 채널은 버퍼가 없었습니다. 버퍼가 없는 채널은 sender와 receiver가 서로 만났을 때 요소를 전달했습니다. send가 먼저 발생하면, receive가 발생할 때까지 중단되어 있습니다. receive가 먼저 발생하면, send가 발생할 때까지 중단되어 있습니다.   

Channel 팩토리 함수, produce 빌더 모두 버퍼 사이즈를 구체화하는 `capacity` 매개 변수를 선택적으로 가질 수 있습니다. 버퍼는 sender가 suspending 되기 전에 여러 개의 요소를 보낼 수 있게 허용합니다. 구체적인 용량을 가지고 있고 버퍼가 다 찼을 때 block 되는 `BlockingQueue`와 비슷합니다.   

다음 코드의 수행 결과를 봐주세요.   
```kotlin
val channel = Channel<Int>(4) // create buffered channel
val sender = launch { // launch sender coroutine
    repeat(10) {
        println("Sending $it") // print before sending each element
        channel.send(it) // will suspend when buffer is full
    }
}
// don't receive anything... just wait....
delay(1000)
sender.cancel() // cancel sender coroutine
```   

    Sending 0
    Sending 1
    Sending 2
    Sending 3
    Sending 4    

## Channels are fair   
채널로 전송 및 수신 작업은 여러 코루틴의 호출 순서와 관련하여 공정합니다. 이들은 선착순으로 제공됩니다. 예를 들어, 수신을 호출하는 첫 번째 코루틴이 요소를 얻습니다. 다음 예제에서는 두 코루틴 "ping" 및 "pong"이 공유 "table" 채널에서 "ball" 개체를 수신하고 있습니다.   

```kotlin
data class Ball(var hits: Int)

fun main() = runBlocking {
    val table = Channel<Ball>() // a shared table
    launch { player("ping", table) }
    launch { player("pong", table) }
    table.send(Ball(0)) // serve the ball
    delay(1000) // delay 1 second
    coroutineContext.cancelChildren() // game over, cancel them
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) { // receive the ball in a loop
        ball.hits++
        println("$name $ball")
        delay(300) // wait a bit
        table.send(ball) // send the ball back
    }
}
```   

    ping Ball(hits=1)
    pong Ball(hits=2)
    ping Ball(hits=3)
    pong Ball(hits=4)    

## Ticker channels   
Ticker 채널은 이 채널에서 마지막으로 소모된 이후 지정된 delay 패스가 있을 때마다 Unit을 생성하는 특수 랑데부 채널입니다. 윈도우 설정 및 기타 시간 의존적 처리를 수행하는 복잡한 시간 기반 생산 파이프라인 및 운영자를 만드는 유용한 구성 요소입니다. 이 채널을 만들기 위해 ticker 메소드 사용하며 더이상 필요 없을 경우 ReceiveChannel 의 cancel 호출하면 된다.   

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking<Unit> {
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0) // create ticker channel
    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement") // no initial delay

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() } // all subsequent elements have 100ms delay
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    // Emulate large consumption delays
    println("Consumer pauses for 150ms")
    delay(150)
    // Next element is available immediately
    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")
    // Note that the pause between `receive` calls is taken into account and next element arrives faster
    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() } 
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel() // indicate that no more elements are needed
}
```   

    Initial element is available immediately: kotlin.Unit
    Next element is not ready in 50 ms: null
    Next element is ready in 100 ms: kotlin.Unit
    Consumer pauses for 150ms
    Next element is available immediately after large consumer delay: kotlin.Unit
    Next element is ready in 50ms after consumer pause in 150ms: kotlin.Unit   

티커는 가능한 소비자 일시 중지를 인식하며, 기본적으로 일시 중지가 발생할 경우 다음에 생성되는 요소 지연을 조정하여 생성되는 요소의 고정 속도를 유지하려고 합니다.   

선택적으로 TickerMode와 동일한 모드 매개 변수입니다.FIXED_DELay를 지정하여 요소 간에 고정된 지연을 유지할 수 있습니다.


