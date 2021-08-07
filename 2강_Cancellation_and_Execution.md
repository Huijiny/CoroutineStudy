# 2강 Cancellation and Execution

만든 코루틴을 어떻게 취소하고 어떻게 타임아웃할지 알아봅시다.



## Job

Job객체 => Lunch로 만들면 잡객체로 전환을 해준다. 이 잡객체를 이용해서 코루틴을 취소하거나, job을 완료할 때까지 메인함수가 종료되지 않는다.

Coroutines의 Job은 coroutines의 상태를 가지고있다. 

[블로그참조](https://thdev.tech/kotlin/2019/04/08/Init-Coroutines-Job/)

![스크린샷 2021-08-07 오후 9.48.35](2강_Cancellation_and_Execution.assets/스크린샷 2021-08-07 오후 9.48.35.png)

![coroutine-job](https://thdev.tech/images/posts/2019/04/Init-Coroutines-Job/coroutine-job.png)

## 아래 예시에 나오는 start(), join(), cancel()

- start : 현재의 coroutine의 동작 상태를 체크하며, 동작 중인 경우 true, 준비 또는 완료 상태이면 false를 return 한다.
- join : 현재의 coroutine 동작이 끝날 때까지 대기한다. 

- cancel : 현재 coroutine을 즉시 종료하도록 유도만 하고 대기하지 않는다. 다만 타이트하게 동작하는 단순 루프에서는 delay가 없다면 종료하지 못한다.
- cancelAndJoin : 현재 coroutine에 종료하라는 신호를 보내고, 정상 종료할 때까지 대기한다.

![스크린샷 2021-08-07 오후 3.11.16](2강_Cancellation_and_Execution.assets/스크린샷 2021-08-07 오후 3.11.16.png)

![스크린샷 2021-08-07 오후 3.12.56](2강_Cancellation_and_Execution.assets/스크린샷 2021-08-07 오후 3.12.56.png)

취소가 정상적으로 먹힌 케이스

#### * Join()의 역할 

> join을 하면 1000
>
> cancel하면 캔슬이 되었는지 안되었는지를 모르지만, join은 캔슬이 되어있는지 아닌지를 알 수 있다.
>
> 이 예시에서는 job의 repeat이 있는데 while문으로 돌리게된다면 job에 캔슬을 넣어도 while문을 전부 다 돌거다.
>
> > 정정: repeat은 suspend함수가 아니라 while이랑 거기서거기
> >
> > 중요한건 내부적으로 suspension point(딜레이나, 일드같은 함수)같이 suspend가 가능한 함수가 있어야지 정상적인 캔슬이 가능하다. 
>
> cancel vs cancelAndJoin -> 캔슬이 제대로 완료가 됐냐를 확인하냐 안하냐는 차이다. 

## Cancellation is cooperative (취소는 협력적이다.)

**취소를 위해서는 협의나 규약이 있어야한다.** 

cancel() & join()

### Making computation code cancellable

delay라는 함수를 while문 안에 넣어준다.

delay는 suspend 함수다.

**결국 suspend함수가 안에 있어야지** cancel이 정상적으로 동작한다.

즉, 취소상태를 명시적으로 확인한다.

공식적으로 두 가지 방법이 있다.

#### 1. Yield()

delay를 빼고 yield()를 넣었더니 while문에서 취소가 가능한다.

얘는 try, catch로 잡아내서 익셉션을 잡아낼 수 있다.

예외처리를 하거나 등을 진행할 수 있다. 

> 프로세스의 우선순위를 정해주는. 얘 말고 더 중요한애 있으면 걔 해! 라는 함수
>
> 코루틴 쪽은 새로운 개념, 용어, 함수가 등장하면 좀 더 집중해서 찾아보면 좋을 것 같아요.

#### 2. isActive

`CoroutineScope.isActive`

isActive는 코루틴 스코프 내부에 있는 변수다. 코루틴이 동작중인지 아닌지를 판단할 수 있는 변수다.

>  조금 더 정확하게는 코루틴 스코프 외부에서 사용하기 편하게끔 extension 변수다. getters보면 코루틴 내에 있긴 한데 null-safety하게 제공해준다고 생각하면 편하다. 

while문에서 코루틴이 살아있으면 동작하고, 죽으면 탈출을 하게 될 것이다. 

https://thdev.tech/kotlin/2019/04/08/Init-Coroutines-Job/  

## Closing resources with finally

코루틴을 어떤 이유에서든 종료할 때 리소스를 어떻게 해제할지를 결정한다.

try도 그렇고 catch도 그렇고 finally에 도착하는데 그렇기 때문에 finally에서 리소스를 해제해주면 된다. 

**만약에 catch를 지우게 된다면?**

> try하다가 catch에서 익셉션 로그가 없고 바로 finaly로빠진다.
>
> 그래서 익셉션에 의해서 크래쉬가 나진 않았다. 이유는, lunch내부에서 익셉션을 잡고 있다. 그래서 시스템에서 에러를 발생시키진않는다. 본인이 원해서 로그를 찍지 않는 이상 사실 finally만 달아도 된다.

## Run non-cancellable block

코루틴이 끝난 다음에 코루틴을 또 만나는 코드

잠깐 멈췄다가 다시 코루틴이 시작한다. 

코루틴한테 멈춰!  > 아 잠깐만 하나만 더  ㅠ 이느낌

cancel되는 코루틴에서 block해야하는 경우 `withContext` 를 사용해!

**왜 finally에서 다시 코루틴을 실행시키고, withContext가 무엇이죠?**

> finally에서 보통 suspend함수는 별로 안쓴다. suspend 함수중에서 값을 받고싶을때 많이 쓴다.
>
> 서버에 실행실패 로그 보낼 때 쓰일 것 같다.
>
> delay는 코루틴을 정지를 시킨다. 함수는 즉각적으로 실행된다. 그래서 withContext를 finally내부에서 지연 가능한 시점에서 쓰게되는거다. 
>
> withContext를 쓸 때, Lunch라는 스코프안에서 다른 스코프를 열거나 디스패쳐io를 태우거나 다른 
>
> 스코프 내부에서 디스패쳐를 바꿔가면서 사용할 때 사용하면 좋다.
>
> io관련하다가 ui관련 애를 처리하려고 메인을 작업하려고 디스패쳐를 바꿔줘야한다던지 이런경우일 것 같아요.
>
> 
>
> ---
>
> withContext관련 공식문서 참고하자면 디스패처를 이용해 현재 스레드를 잠깐 바꿔주는 친구인 것 같다. 
>
> ```kotlin
> suspend fun fetchDocs() {                      // Dispatchers.Main
>     val result = get("developer.android.com")  // Dispatchers.Main
>     show(result)                               // Dispatchers.Main
> }
> 
> suspend fun get(url: String) =                 // Dispatchers.Main
>     withContext(Dispatchers.IO) {              // Dispatchers.IO (main-safety block)
>         /* perform network IO here */          // Dispatchers.IO (main-safety block)
>     }                                          // Dispatchers.Main
> }
> ```

## Timeout

대부분 시간제한 때문에 코루틴 실행이 취소되는 경우가 많다.

위에까지의 예시에서는 lunch로 job을 받아서 캔슬처리를 했다.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
}	
---
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms

```

여기서는 runblocking 내에서 바로 timeout을 처리를 했다. 그래서 익셉션이 터지는 것이다.

이걸 해소하려면 withTimeoutOrNull을 사용하면 Exception이 나지 않는다. 

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // will get cancelled before it produces this result
    }
    println("Result is $result")
}
---
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```



`withTimeout` => Timeoout cancellationException

`withTimeoutOrNull` => Exception 대신 null 발행

## Asynchrounous timeout and resources

withTimeout이라는게 비동기로 처리가 되는데, withTime이 반환되기전에 어디서든 타임아웃이 발생할 수 있다. 스코프내에서 리소스를 가지고있을 때 , 외부에서 건드리게되면 이슈가 발생할 수 있다. 

두개이 차이는 

```kotlin
var acquired = 0

class Resource {
    init { acquired++ } // Acquire the resource
    fun close() { acquired-- } // Release the resource
}

fun main() {
    runBlocking {
        repeat(100_000) { // Launch 100K coroutines
            launch { 
                val resource = withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    Resource() // Acquire a resource and return it from withTimeout block     
                }
                resource.close() // Release the resource
            }
        }
    }
    // Outside of runBlocking all coroutines have completed
    println(acquired) // Print the number of resources still acquired
}
```



위에는 60초가 지나면 이친구를 수행하다말고 타임아웃을 할건데, 지금은 딜레이수가 더 적지만 더 크면 리소스할당 전에 withTimeout으로 타임아웃이되는데 아래서 할당을 해제한대 -> ??이상한코드다.

```kotlin
runBlocking {
    repeat(100_000) { // Launch 100K coroutines
        launch { 
            var resource: Resource? = null // Not acquired yet
            try {
                withTimeout(60) { // Timeout of 60 ms
                    delay(50) // Delay for 50 ms
                    resource = Resource() // Store a resource to the variable if acquired      
                }
                // We can do something else with the resource here
            } finally {  
                resource?.close() // Release the resource if it was acquired
            }
        }
    }
}
// Outside of runBlocking all coroutines have completed
println(acquired) // Print the number of resources still acquired
```

그걸 방지하기 위해서 아래는 위에서 리소스를 선언을 미리 해주고, withTimeout을 걸어놓고 아래서 딜레이걸어서 Resource를 할당해줘도 딜레이 타임아웃에 높게 걸려도 finally에서 클로즈 가능하다!

