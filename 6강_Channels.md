# 6강 Channels

편하게 전송할 수 있는 도구다.

채널은 서로 다른 코루틴간의 결과값을 받을 수 있는 도구라고 생각하면 된다.



## Channel basics

채널은 BlockingQueue와 매우 흡사하다. 한 가지 중요한 차이점은 blocking의 put작업 대신 suspending send가 존재하고 blocking의 take 작업 대신 suspending receive가 존재한다는 것.

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

channel로 x*x를 보내고 receive로 받는다.



## Closing and iteration over channels

큐와는 다르게 채널은 닫음으로써 더 이상 요소가 오지 않음을 표현할 수 있다. 수신하는 부분에선 일반 for룹을 사용하여 채널로부터 요소를 수신하는 것이 편리하다.

close() -> 내부에 정의된 불린 값 => 채널에 특별한 close 토큰을 보내는 것과 같다. 이 close 토큰을 받는 즉시 반복이 중지되므로 close되기 전에 이전에 보낸 모든 요소가 수신된다는 것을 보장한다.

클로즈가 되었는지 확인도 해줄 수 있는 부분이다.

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

![Welcome to Kotlin hands-on](6강_Channels.assets/UsingChannelManyCoroutines.png)

코루틴에서 일련의 원소를 생성하는 패턴을 꽤 흔하다. 이는 concurrent 코드에서 흔히 발견할 수 있는 Producer-consumer 패턴의 한 부분이다. 우리는 producer를 추상화해서 채널을 매개변수로 가지는 함수로 만들 수 있겠지만, 이는 함수에서 결과가 반드시 되돌아와야한다는 것에서 어긋난다. 

그렇기 때문에 채널에선 생산자 측에서 바로 Produce할 수 있도록 하는 편리한 코루틴 빌더와 Producer측에서 for loop을 대체하여 사용할 수 있는 consumeEach라는 익스텐션 함수가 있다.

consumeEach는 내부적으로 모든 값들을 소비하게 된다. 1부터 5까지의 값들을 다 보여준다.

CoroutineScope이 완료가 되었을 때 채널을 클로즈 해준다. 



## Pipelines

파이프라인은 한 코루틴이 무한대의 stream 값을 생성하는 패턴이다. 유한한 토큰과 스트림 값들을 연결 연결 시켜줘서 값들을 연산할 수 있다는 내용이다.

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1
    while (true) send(x++) // infinite stream of integers starting from 1
}
```

위의 코드에서 무한의 숫자를 계속해서 생산한다.

```kotlin
fun CoroutineScope.square(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (x in numbers) send(x * x)
}
```

위의 코드에서는 스트림을 받아서 x*x를 해줌으로서 소비한다. 

```kotlin
val numbers = produceNumbers() // produces integers from 1 and on
val squares = square(numbers) // squares integers
repeat(5) {
    println(squares.receive()) // print first five
}
println("Done!") // we are done
coroutineContext.cancelChildren() // cancel children coroutines
```

모두 연결해보면 이렇게 squares.receive에서 첫 5개를 받게되고 cancel childeren을 해줌으로서 코루틴을 끊어준다. 



## Prime numbers with pipeline

```kotlin
fun CoroutineScope.numbersFrom(start: Int) = produce<Int> {
    var x = start
    while (true) send(x++) // infinite stream of integers from start
}
```

```kotlin
fun CoroutineScope.filter(numbers: ReceiveChannel<Int>, prime: Int) = produce<Int> {
    for (x in numbers) if (x % prime != 0) send(x)
}
```

위 코드에서 소수로 떨어지지 않는 애들을 필터링 한다. 

```kotlin
var cur = numbersFrom(2)
repeat(10) {
    val prime = cur.receive()
    println(prime)
    cur = filter(cur, prime)
}
coroutineContext.cancelChildren() // cancel all children to let main finish
```

그럼 2에서부터 시작해서 현재 채널에서 소수를 가져오고 각각의 소수에 대해 새로운 파이프라인 단계를 시작함으로써 파이프라인을 구축할 수 있다. 



## Fan-out(1:n)

`송신자는 하나인데, 수신자가 여러개인 경우`

여러 코루틴이 동일한 채널에서 수신되어 서로 간에 작업을 분배할 수 있다. 

예를들어, 정수를 주기적으로 생산하는 Producer 코루틴부터 알아보자면, 

```kotlin
fun CoroutineScope.produceNumbers() = produce<Int> {
    var x = 1 // start from 1
    while (true) {
        send(x++) // produce next
        delay(100) // wait 0.1s
    }
}
```

우리는 여러 개의 프로세서 코루틴을 얻을 수 있다. 이 예제에서는 id와 수신 번호만 출력하게 되는데, 

```kotlin
fun CoroutineScope.launchProcessor(
    id: Int, 
    channel: ReceiveChannel<Int>
) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }    
}
```

그리고 위의 프로세서들을 5번 실행한다면? 아래와 같은 결과가 나온다.

```kotlin
val producer = produceNumbers()
repeat(5) { launchProcessor(it, producer) }
delay(950)
producer.cancel() // cancel producer coroutine and thus kill them all
```

```
Processor #0 received 1
Processor #0 received 2
Processor #1 received 3
Processor #2 received 4
Processor #3 received 5
Processor #4 received 6
Processor #0 received 7
Processor #1 received 8
Processor #2 received 9
Processor #3 received 10
```



## Fan-in(n:1)

`송신자가 여러개이고, 수신자가 하나인 경우`

여러 코루틴이 동일한 채널로 전송될 수 있다. 예를 들어, string 타입의 채널에 delay를 사용하여 문자열을 이 채널에 반복적으로 전송하는 suspend함수가 있다. 

```kotlin
suspend fun sendString(
    channel: SendChannel<String>, 
    s: String, 
    time: Long
) {
    while (true) {
        delay(time)
        channel.send(s)
    }
}

```

```kotlin
val channel = Channel<String>()
launch { sendString(channel, "foo", 200L) }
launch { sendString(channel, "BAR!", 500L) }
repeat(6) { // receive first six
    println(channel.receive())
}
coroutineContext.cancelChildren() // cancel all children to let main finish
```

이후에 결과를 보자면,

```
foo
foo
BAR!
foo
foo
BAR!
```

이렇게 된다.



## Buffered channels

지금까지 살펴본 채널은 buffer가 없었다. 버퍼링되지 않은 채널은 송신자와 수신자가 서로 만날 때 요소들을 전송한다. 전송이 먼저 호출되면 수신은 호출될 때까지 일시 중단되고, 수신 호출이 먼저 호출될 경우 전송이 호출될 때까지 일시 중단된다. 

`Channel()` factory함수와 `produce`builder 모두 버퍼 크기를 지정하기 위해 선택적으로 `capacity` 라는 매개 변수를 사용한다. 버퍼는 송신자가 일시 중단하기 전까지 여러 요소를 전송할 수 있도록 하는데, 이는 버퍼가 가득차면 차단되는 지정된 용량의 BlockingQueue와 유사하다.



1. RendezavousChannel
2. LinkedListChannel
3. ArrayChannel
4. ConflatedChannel



## Channels are fair

채널로의 송수신 작업은 여러 코루틴에서 호출하는 순서에 따라 공정하게 이루어진다. 즉, 선착순으로 이루어지는데 예를 들자면 수신 작업을 호출하는 첫 번째 코루틴이 그 요소를 얻게 되는 것이다.

결국 Ping이 먼저 Pong보다 수행된다.



## Ticker channel

