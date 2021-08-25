# Coroutine Exception handling



이 페이지에서는 예외 처리와 취소로 인한 예외에 대해서 다룬다. 

이미 취소된 코루틴은 중단점에서 CancellationException예외를 던지며 이 예외는 코루틴 체계에서 무시 된다는 것을 이미 알고 있다. 하지만 만일 취소 동작 중 예외가 발생하거나 혹은 2개 이상의 자식 코루틴들에서 동시에 예외가 발생하면 어떻게 될까?



## Exception propagation

코루틴 빌더들은 예외를 처리하는 방식에 따라 다음과 같이 크게 두 가지 타입으로 나눌 수 있다. 

- 예외를 자동으로 전파하거나 (launch나 actor)
- 사용자에게 노출하여 예외처리를 일임한다.(async, produce)

이 빌더들이 코루틴 root를 생성하는데 사용될 때 이는 다른 코루틴의 자식이 아니고, 전자의 빌더 즉 Lauth나 acrot는 자바의 `Thread.uncaughtExceptionHandler` 처럼 '처리되지 않은 예외'로 간주된다.  반면에 후자 async나 produce와 같은 빌더들은 마지막으로 예외를 처리하는 예외 처리 핸들러에게 그 처리 방식이 달려있고, 예를들어 await, receive등이 있다. 

이러한 예외 전파의 특징은 다음과 같이 글로벌 스코프에서 새로운 코루틴을 생성하는 예제로 쉽게 확인해 볼 수 있다. 

```kotlin
import kotlinx.coroutines.*

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



## CoroutineExceptionHandler

위에서 살펴본 것과 같이 처리되지 않은 예외를 콘솔로 출력하는 기본 동작을 변경 할 수 있다. 루트 코루틴(부모가 없는 최 상위 코루틴)에 CoroutineExceptionHandler 컨텍스트 요소를 설정하면 이 핸들러는 이 루트 코루틴 및 모든 자식 코루틴들을 위한 범용적인 catch 블록과 같이 사용된다. 이것은 Thread.uncaughtExceptionHandler와 유사하다. 우리는 CoroutineExceptionHandler를 이용하여 예외 상황을 복구할 수는 없다. 예외 핸들러가 호출 된 코루틴은 이미 발생한 예외로 인해 종료되었기 때문이다. **일반적으로 예외 핸들러는 예외를 로깅하거나 관련 예외 메시지를 사용자에게 보여주고 애플리케이션을 종료하거나 재시작하기 위해 사용된다.**



JVM에서는 ServiceLoader를 통해 `CoroutineExceptionHandler`를 등록함으로써 **모든 코루틴들을 위한 범용적인 예외 처리 핸들러를 재정의 할 수 있다.** 이러한 범용 예외 처리 핸들러는 적절한 예외 처리기가 등록되어 있지 않을 경우 최종적으로 사용되는 Thread.defaultUncaughtExceptionHandler와 유사하다. Android에서는 uncaughtExceptionPreHandler가 범용 코루틴 예외 처리기로 설치되어있다. 



`CoroutineExceptionHandler`는 사용자에 의해 처리되지 않은 예외에 한해서만 호출된다. 특히, 모든 자식 코루틴(다른 Job의 컨텍스트를 이용해 생성된 코루틴들)은 예외 처리를 그들의 부모 코루틴에게 위임하고, 그 부모 코루틴은 그 부모 코루틴에게 이런식으로 루트 코루틴에 도달할 때까지 예외처리를 위임한다. **그래서 자식 코루틴에게 설치된 CoroutineExceptionHandler는 절대 사용되지 않는다.** 추가적으로 `async코루틴 빌더`는 **항상 모든 예외들을 catch 해 결과 객체인 Deffered 객체를 통해 노출한다. 그러므로 이러한 코루틴 빌더의 CoroutineExceptionHandler역시 아무런 효과가 없다.** 



## Cancellation and exceptions

취소는 예외와 밀접한 관계가 있다. 코루틴은 내부적으로 취소를 위해서 `CancellationException`을 사용하고, 이러한 예외들은 모든 핸들러들에게 무시 되므로 **catch 블록에서 획득할 수 있는 부가적인 디버깅 정보 획득용으로만 사용**하는 것이 좋다. **코루틴이 `Job.cancel()`로 인해 취소되면 자신의 실행을 종료하지만 부모 코루틴에게 취소를 요청하지는 않는다.**

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
```

만약 어떤 코루틴이 `CancellationException` 이외의 예외를 만나면 그 예외를 부모 코루틴에게 전달하여 부모 코루틴도 취소하게 된다. 이 동작 방식은 재정의 될 수 없고, 구조화된 동시성을 위한 안정된 코루틴 계층을 제공하기 위해 사용된다. `CoroutineExceptionHandler` 구현은 자식 코루틴들에 의해 사용되진 않는다. 

> 예제들에서 CoroutineExceptionHandler는 항상 GlobalScope에서 생성된 코루틴에 설정되었다. 예외 핸들러를 메인함수 runBlocking{} 스코프에서 설정하는 것은 모순되어 보이는데, 메인 코루틴은 그 자식 코루틴이 예외로 인해 종료되면 핸들러가 설치되어 있더라도 항상 취소될 것이기 때문이다.

최초 발생한 예외는 **모든 자식 코루틴이 종료된 후에야 부모 코루틴에 의해서 처리된다.**  > 왜죠!!!!?!?!?!?!?!?!?! 이해 X



```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception")
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
}
```



## Exception aggregation

만약 두 개 이상의 자식 코루틴들에서 예외가 발생한다면 어떻게 될까?

**기본적으로는 처음 발생한 예외가 우선한다.** 그러므로 처음 발생한 예외가 예외 처리 매커니즘에 의해 처리되게 된다. 그 이후에 발생하는 모든 예외들은 첫번째 예외에 숨겨져 추가되게 된다. 

취소 익셉션은, 익셉션을 전파하는데 있어서 투명하고 기본적으로 언래핑된다. 즉, 취소 이외의 예외가 발생하여 코루틴이 취소 될 경우 CoroutineExceptionHandler가 예외를 처리할 때는 취소 예외는 바로 언래핑되고, 최소 발생한 예외가 핸들러의 exception 파라미터로 바로 전달된다. 

```kotlin
val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught original $exception")
}
val job = GlobalScope.launch(handler) {
    val inner = launch {
        launch {
            launch {
                throw IOException()
            }
        }
    }
    try {
        inner.join()
    } catch (e: CancellationException) {
        println("Rethrowing CancellationException with original cause")
        throw e
    }
}
job.join()
```

```
Rethrowing CancellationException with original cause
Caught original java.io.IOException
```



## Supervision

앞서 학습한 것처럼 취소는 전체 코루틴 계층 속에서 부모-자식 코루틴들간의 관계에서 양방향으로 전파된다.



하지만 단방향 취소가 필요한 경우가 있다. 이 경우에 대한 좋은 예로는 동일 스코프에 정의된 Job을 공유하는 UI 컴포넌트가 적절하다. 만약 UI의 자식 태스크 중 하나가 실패했다고 하더라도 전체 UI를 취소 할 필요는 없다. 하지만 UI 컴포넌트가 종료되면 모든 자식들의 결과물도 필요 없게 되므로 자식들의 Job도 취소되어야 한다.



도 다른 예로는 여러개의 자식 작업을 갖는 서버 프로세스를 들 수 있는데, 이러한 서버 프로세스는 자식 작업들의 실행과 실패를 감독하고 실패한 자식 작업에 대해서는 다시 시작될 수 있도록 해야한다.



### Supervision job

이러한 목적을 위해서는 `SupervisorJob`이 사용될 수 있다. 이것은 일반적인 Job과 유사하지만 예외로 인한 취소가 부모 -> 자식 으로만 전파된다는 점은 다르다.  **=> 예시가 이해가 안감..**

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        // launch the first child -- its exception is ignored for this example (don't do this in practice!)
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("First child is failing")
            throw AssertionError("First child is cancelled")
        }
        // launch the second child
        val secondChild = launch {
            firstChild.join()
            // Cancellation of the first child is not propagated to the second child
            println("First child is cancelled: ${firstChild.isCancelled}, but second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                // But cancellation of the supervisor is propagated
                println("Second child is cancelled because supervisor is cancelled")
            }
        }
        // wait until the first child fails & completes
        firstChild.join()
        println("Cancelling supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}
```

예를 들어 이 경우,



## Supervision scope

범위를 지정하여 동시성 제어를 하기 위해서는 `coroutineScope`를 사용하는데 이러한 스코프에서 단방향으로 전파되는 취소 매커니즘을 사용하려면 `supervisorScope`를 사용하면 된다. 이것은 **취소를 단방향으로 전파하며 오직 자신이 실패했을때만 모든 자식을 취소**한다. 또한 **coroutineScope와 동일하게 자신의 종료 전에 모든 자식들의 종료를 기다린다.** 

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    try {
        supervisorScope {
            val child = launch {
                try {
                    println("Child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("Child is cancelled")
                }
            }
            // Give our child a chance to execute and print using yield
            yield()
            println("Throwing exception from scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught assertion error")
    }
}
```

```
Child is sleeping
Throwing exception from scope
Child is cancelled
Caught assertion error
```



yield() 함수는 표현식을 계속 진행하기 전에 실행을 잠시 멈추고 요소를 반환한뒤 멈춘 시점에서 다시 실행을 재개합니다.



## Exceptions in supervised coroutines

일반적인 Job과 Supervisor Job의 중요한 차이점은 예외 처리 방식에 있다. 모든 자식은 예외 처리 매커니즘을 통해서 자신의 예외를 처리해야한다. 이러한 차이점은 자식의 실패가 부모로 전파되지 않는 사실로부터 발생한다. 즉, **`supervisorScorp` 안에서 실행된 코루틴들은 그들의 스코프에 설정된 CoroutineExceptionHandler 를 사용한다는 것**을 의미한다. 

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception")
    }
    supervisorScope {
        val child = launch(handler) {
            println("Child throws an exception")
            throw AssertionError()
        }
        println("Scope is completing")
    }
    println("Scope is completed")
}
```

