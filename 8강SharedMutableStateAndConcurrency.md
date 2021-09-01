# Shared mutable state and concurrency



코루틴은 컬티 스레드 디스패처에 의해 동시에 실행되는 것처럼 Dispatchers.Default에 의 해 비동기적으로 실행된다. 하지만 동시성으로 인해 발생될 수 있는 문제점들을 가지고 있다. 

예를들어, 변경 가능한 공유 자원에 대해 접근함으로써 발생되는 동시성문제가 있다. 이를 코루틴에서 어떻게 해결할까?

```kotlin
suspend fun massiveRun(action: suspend () -> Unit) {
    val n = 100  // number of coroutines to launch
    val k = 1000 // times an action is repeated by each coroutine
    val time = measureTimeMillis {
        coroutineScope { // scope for coroutines
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("Completed ${n * k} actions in $time ms")
}
```

n개의 코루틴을 실행해 각 코루틴마다 action을 K 번 실행하는 시간을 측정하는 함수다. 

```kotlin
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

Dispatchers.Default를 이용해 멀티 스레드 환경에서 코루틴간 공유되는 변수를 증가시키는 동작을 시작하면 최종적으로 100개의 코루틴이 서로 동기화 없이 여러개의 스레드에서 동시에 카운터를 증가시킨다. 수천개의 코루틴이 동기화 없이 여러 스레드에서 counter를 동시에 증가시키기 때문에 Counter = 1000000를 출력할 가능성이 없다.



## Volatile

변수를 volatile로 선언해도 소용이 없다. volatile은 변수를 메인 메모리에 저장하겠다고 명시하는 것인데 변수에 접근할 때 캐시가 아닌 항상 메인 메모리에서 읽어오라는 뜻이다. 하지만 여러개의 스레드가 여.전.히. 변수를 동시에 읽어올 수 있으므로 문제를 해결하기 힘들다. 

```kotlin
@Volatile // 코틀린에서는 volatile을 annotation으로 선언한다.
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

> 횟수가 너무 많아서 그런 것 같다. Dispatchers.Default를 써서 그런지. 여러개의 스레드가 있어서 volatile로 해결이 안된 것 같음. 동시성 문제도 있는 것 같음. 같은 스레드에서 동시성으로 인해서 하나의 job이 읽어서 올리려고 하는 그 사이에 올리기 전에 읽어버린 것 같다. 
>
> > 선형적인 읽기 쓰기가 가능하다 -> 읽는 시점은 알 수 있는데, 메인에 쓰는 시점을 알 수 없어서 좀 보장이 안될 수도 있다. 
> >
> > 쓰기에 대한 허점이 있는 것 같다. 

## Thread-safe data structures

일반적인 해결법 중 하나로 thread-safe한 자료구조를 사용하는 방법이 있다. Thread-safe란, 동시에 최대 하나의 스레드만 변수에 접근할 수 있도록 자체적으로 제어하는 변수를 의미한다. 따라서 thread-safe한 변수를 사용하면 여러 개의 스레드가 동시에 접근해도 변수를 동기화할 수 있다. 예를들어 코틀린에는 `Int` 의 thread-safe타입인 `AtomicInteger`이 있다. `incrementAndGet`연산을 사용하면 `AtomicInteger`를 thread-safe하게 증가시킬 수 있다.

> 내부 알고리즘 동작알고리즘이 compare and swap? 임 락을 안하는 방식이다. mutex나 lock을 하는게 안전한데 그걸 안하고 계속 비교를 해서 원래 알고있는 값이 맞는지?를 확인하기 때문에 빠르다. 비교하고 맞으면 내가 수정하겠다 방식임. 호출이 되는 순간에 값을 가지고 한다. 그럼 다음에 다시 시도한다. 

```kotlin
val counter = AtomicInteger()

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.incrementAndGet()
        }
    }
    println("Counter = $counter")
}
```



>  하지만 `Atomic`으로 접근하면 접근상태나 연산이 복잡한 경우를 처리할 수 없다. 코드를 근본적으로 thread-safe하게 만들어야한다.

## Thread confinement: fine-grained

Thread confinement는 하나의 스레드를 통해서만 변수에 접근할 수 있도록 하는 해결법이다. 예를들어 안드로이드에서는 메인스레드에서만 UI를 갱신할 수 있다. 위의 코드에 적용하려면 단일 스레드 문맥에서만 변수에 접근하도록 하면 된다. 

하지만 이 방법은 실행시간이 느린데, 값을 증가시킬 때 마다 `Dispatchers.Default`문맥을 `counterContext`로 바꿔서 실행하기 때문이다. 

```kotlin
val counterContext = newSingleThreadContext("CounterContext") // 단일 스레드 문맥
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // 이 부분만 문맥을 counterContext로 바꿔서 실행
            withContext(counterContext) {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

```
Completed 100000 actions in 2339 ms
Counter = 100000
```



## Thread confinement: coarse-grained

사실상 `Dispatchers.Default`는 사용되지 않는 문맥이다. 애초에 실행 자체를 `counterContext`에서 하면 불필요한 문맥 교환을 막을 수 있을 것이다.

```kotlin
val counterContext = newSingleThreadContext("CounterContext")
var counter = 0

fun main() = runBlocking {
    // 실행 자체를 단일 스레드 문맥에서 하면 된다.
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter = $counter")
}
```

```
Completed 100000 actions in 37 ms
Counter = 100000
```



## Mutual exclusion

Mutual exclusion이란 공유 변수에 접근하는 코드를 critical section으로 보호하여 critical section이 동시에 실행되지 않도록 막는 방법이다. 일반적으로는 `synchronized`나 `ReentrantLock`을 사용하지만 코루틴에서는 `Mutex` 를 사용하다. `lock` 과 `unlock`함수를 사용하여 critical section을 표현할 수 있다. `Mutex.lock()`등은 모두 suspending 함수이기 때문에 스레드를 Block하지 않는다. `Mutex.withLock`함수를 사용하면 자동으로 Lock을 걸고 풀어준다.

```kotlin
val mutex = Mutex()
var counter = 0

fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            // critical section
            mutex.withLock {
                counter++
            }
        }
    }
    println("Counter = $counter")
}
```

```
Completed 100000 actions in 1396 ms
Counter = 100000
```



## Actors

Actors는 코루틴의 묶음이다. 1. Private 변수와 다른 코루틴과 의사소통할 수 있는 channel로 이루어진 객체다. 변수가 여러 개 있다면 actor를 클래스로 정의하는 것이 좋지만, 지금은 변수가 하나뿐이므로 함수를 사용해서 작성할 수 있다.

`actor`코루틴 빌더를 사용하면 메세지를 수신할 Mailbox channel과 결과를 돌려줄 send channel을 간편하게 정의할 수 있다. 따라서 `actor`가 반환한 객체만을 가지고도 actor를 조작할 수 있다.

Actor를 사용하려면 먼저 actor가 처리할 메시지를 정의해야한다. 코틀린의 `sealed class`를 사용하면 좋다. 여러개의 메시지 타입을 `sealed class` 를 통해 하나로 묶고, `when`문을 이용하여 메시지를 타입에 따라 처리할 수 있다. 

```kotlin
sealed class CounterMsg // 모든 메시지 타입의 부모 클래스
object IncCounter : CounterMsg() // 변수를 1 증가시키라는 메시지
class GetCounter(val response: CompletableDeferred<Int>) : CounterMsg() // 변수의 값을 돌려달라는 메시지
```

`CompletableDeffered`를 사용하면 값 하나를 언젠가 돌려주겠다는 약속을 표현할 수 있다. Actor도 코루틴이기 때문에 본질적으로 비동기이고, 값을 즉시 반환하지 못할 경우가 있을 수 있기 때문이다. 

```kotlin
// 위에서 정의한 메시지를 처리하는 actor 정의
fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0        // 변수 (state)
    for (msg in channel) { // 들어오는 메시지를 처리한다.
        when (msg) {
            is IncCounter -> counter++
            is GetCounter -> msg.response.complete(counter)
        }
    }
}
```

```kotlin
fun main() = runBlocking<Unit> {
    val counter = counterActor()        // actor 생성
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    // actor로부터 값을 받는다.
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println("Counter = ${response.await()}")
    counter.close() // actor 종료
}
```

```
Completed 100000 actions in 988 ms
Counter = 100000
```

Actor는 메시지를 순차적으로 처리하기 때문에 자연스럽게 변수에는 한 번에 하나의 작업만 수행하고, actor는 오직 메시지를 통해서만 서로 의사소통할 수 있다.

결국, 문맥이 바뀌지 않기때문에 더 이상 스레드 문맥을 고려할 필요가 없고, 시간도 빨라진다. 

https://thinking-face.tistory.com/entry/Kotlin-Coroutines-Shared-Mutable-State-and-concurrency

