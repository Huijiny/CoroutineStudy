#  Coroutine context and dispatchers

코틀린 표준 라이브러리에 정의 된 CoroutineContext타입의 값으로 표현되는 컨텍스트 내에서 수행된다.

코루틴 컨텍스트는 다양한 컨텍스트 요소들의 집합이다. 주된 요소로는 전에 보았던 코루틴 잡과 이번 섹션에 다룰 디스패쳐가 있다.

## 코루틴의 준비물!

### CoroutineScope

-> 코루틴을 실행할 때 사용되는 준비물들을 가지고 있다. 

코틀린 확장에서 코루틴 스코프의 한 종류들이 여러가지 형태로 존재하는데,  ViewModelScope, LifecycleScope, GlobalScope, CoroutineScope가 있다. ViewModelScope는 ViewModel에 종속되고, LifeCycleScope도 activity와 fragment의 라이프사이클에 종속되고, GlobalScope는 어플리케이션 전반에 걸쳐 있는 스코프고, CoroutineScope는 각자 자신이 원하는대로 사용하는 스코프들

CoroutineScope의 역할

1. ![스크린샷 2021-08-14 오후 12.49.47](/Users/huijinryu/Library/Application Support/typora-user-images/스크린샷 2021-08-14 오후 12.49.47.png)

1. Coroutine 실행범위 Scope제공
2. Coroutine 실행하기 위한 재료(요소)인 CoroutineContext 제공
3. CoroutineBuilder를 통해서 



### CoroutineContext

코루틴의 동작, 관리를 처리하기 위한 정보를 제공해주는 인터페이스

1. Dispatcher
2. Job
3. ExceptionHandler
4. CoroutineName

이 4가지를 알고 있으면 좋다. Dispatcher나, Job이 각각의 키가 된다.

Immutable map 형태와 비슷하다. - get, fold, plus, minusKey등의 메소들

CoroutineContext들 안에 Coroutine을 실행하기 위한 정보들을 가지고 있다. 

![스크린샷 2021-08-14 오후 12.51.54](/Users/huijinryu/Library/Application Support/typora-user-images/스크린샷 2021-08-14 오후 12.51.54.png)

Job: 실행중인 코루틴의 상태에 대한 정보들

CoroutineName: 주로 디버깅 용

ExceptionHandler : 에러처리용

#### Dispatchers

Default: 	JVM 스레드 shared pool 기준으로 동작. CPU 코어 수와 동일하지만 최소 2개를 사용

IO: Thread 공유 풀로 동작(RxJava: Schedulers.io())

Main: Android UI 처리 (RxJava: Android.main)

Unconinfed: 특정 스레드에 제한되지 않는 코루틴

newSingleThreadContext = Coroutine이 실행할 쓰레드를 직접 생성한다.

#### Job

### ![스크린샷 2021-08-14 오후 12.55.14](/Users/huijinryu/Library/Application Support/typora-user-images/스크린샷 2021-08-14 오후 12.55.14.png)

Coroutine 그 자체에 대한 정보나 코루틴이 수행하고 있는 태스크나 함수들에 대한 정보들을 담고 있다. 

job - 하나의 작업이 취소되면 전체 Coroutine 모두 취소

SupervisorJob= 하나의 작업이 취소되어도 child/parent job에 영향을 미치지 않음 -> Exception handling할 때 등장하는 친구

​							= 예제들을 많이 보고 따라해보거나 

​							= 부모 자식 관계를 독립시킬 때 사용한다.

​							= Exception handler도 스코프마다 따로 적용하고 싶을 수도 있는데 그럴 때 수퍼바이져 잡이랑 같이 사용하기도 하고 그런다.

​							

### CoroutineName

동작중인 코루틴에 직접 이름을 지정할 수 있는데 디버깅용으로 사용하라고 권장됨



### CoroutineContext간의 결합

서로 다른 Element끼리는 plus연산자로 결합이 가능

위에 알아봤던 준비물들을 모두 결합시키는 것

![스크린샷 2021-08-14 오후 12.58.07](/Users/huijinryu/Library/Application Support/typora-user-images/스크린샷 2021-08-14 오후 12.58.07.png)

아래를 보면 디스패쳐를 두 개 넣고 있는데 이러한 결합은 지원하지 않는다.

Immutable map 형태와 비슷한가? -> 그럼 저장과 제거도 가능한가? 그렇다. 유연하게 가능하다.

![스크린샷 2021-08-14 오후 12.59.59](/Users/huijinryu/Library/Application Support/typora-user-images/스크린샷 2021-08-14 오후 12.59.59.png)

CoroutineBuilder => 새로운 코루틴을 실행하는 방법을 제공

Launch는 coroutine에 대한 참조인 job 객체를 반환. return value가 없는 경우 사용

async는 결과 값을 저장하고 있는 Deferred객체를 반환. 

Job? 코루틴 그 자체에 대한 레퍼런스. 자기 자신. 실행하거나 취소하거나. 실행중이라던지 그럴 때 중간에 조절이나 확인을 하면 돼.

async? network를 통해서 데이터를 가져왔다면 결과 값이 있기 때문에 deffered를 사용하는게 좋다. 정확하게는 성공, 실패, 에러를 가지고 있다. 내부에서 실행을 했을 경우에 어떤 값이 저장되어있다. 나중에 이 Defered를 가지고 성공과 실패를 판단하는 과정이 필요하다.



## Dispatchers and thread

코루틴 컨텍스트는 연관된 코루틴들이 실행 시 사용할 스레드 혹은 스레드 풀을 결정짓는 코루틴 디스패처를 포함한다. 코루틴 디스패처는 코루틴의 실행을 특정 스레드로 한정짓거나, 특정 스레드 풀로 전달하거나, 스레드의 제한 없이 실행되도록 할 수 있다. 

모든 코루틴 빌더들(lunch, ansync)를 사용할 때 옵셔널 파라미터로 CoroutineContext를 전달할 수 있고, 이를 이용하여 새로운 코루틴들을 위한 디스패쳐나 그 이외의 컨텍스트 요소들을 지정할 수 있다.

1. **Lunch** {...} 빌더가 파라미터 없이 호출될 경우 실행 된 코루틴 스코프로부터 상속 받은 컨텍스트를 그대로 이용한다. 
2. **Dispatchers.Unconfined**는 메인 스레드에서 실행되는 것 처럼 보이지만 이 디스패처는 다른 매커니즘을 사용한다.
3. **Dispatchers.Default**는 코루틴이 GlobalScope에서 실행될 경우에 사용되며 공통으로 사용되는 백그라운드 스레드 풀을 이용한다. 즉, launch(Dispatchers.Default){...}와 GlobalScope.launch{...} 는 동일한 디스패처를 사용한다.
4. **newSingleThreadContext**는 항상 새로 실행될 코루틴을 위해 새로운 스레드를 생성한다. 하지만 이는 비싼 cost가 들기 때문에 실제 어플리케이션에서는 더이상 사용하지 않을 경우 close()함수를 이용해서 해제하거나 Top-level 변수에 저장해 놓고 애플리케이션 전반에 걸쳐 재사용 되어야 한다.
5. 

## Unconfined vs confined dispatcher

Dispatchers.Unconfined 코루틴 디스패처는 호출 스레드에서 코루틴을 시작하지만 이는 첫번 째 중단점을 만날 때까지만 그렇다. 중단점 이후에 코루틴이 resume될 때는 중단 함수를 재개한 스레드에서 수행된다. 비-한정 디스패처는 코루틴이 CPU 시간을 소모하지 않거나 공유되는 데이터(UI)를 업데이트 하지 않는 경우처럼 특정 스레드에 국한된 작업이 아닌 경우 적절하다. -> 무슨말?



코루틴 컨텍스트 요소들은 부모-자식 코루틴 계층 구조로 정의 될 때 기본적으로 부모 코루틴 스코프의 컨텍스트 요소들이 상속된다. 물론 디스패처도 컨텍스트 요소이기 때문에 동일하게 동작한다. runBlocking{} 코루틴의 기본 디스패처는 호출 스레드에 국한되기 때문에 그것을 상속하는 것은 그 스레드에 국한되도록 만드는 효과가 있고 FIFO 스케쥴링으로 수행되어진다. 



## Debugging coroutines and threads﻿(코루틴과 스레드 디버깅)

코루틴들은 한 스레드에서 중단된 후 다른 스레드에서 재개될 수 있다. 멀티 스레드 환경이 아니더라도 단일 스레드 디스패처에서조차 어떤 코루틴이 언제 어디서 수행중이었는지 밝혀내는 것은 어려운 일이다. 스레드를 사용하는 애플리케이션을 디버깅하기 위한 일반적인 방법은 각각의 로그마다 **현재 스레드 이름을 붙여 출력**하는 것이다. 이 방법은 보통 로깅 프레임워크에 의해 전반적으로 지원된다. 코루틴을 사용할 때 스레드명 만으로는 실행 컨텍스트를 제대로 설명하진 못하기 때문에 코루틴 프레임워크는 이걸 용이하게 하는 디버깅 지원 기능을 포함하고 있다. 

JVM 옵션에 -Dkolinx.coroutines.debug를 추가하고 아래 예제를 실행하면 Thread 이름에 추가로 코루틴의 이름까지 출력되는 것을 확인할 수 있다. 

```kotlin
val a = async {
    log("I'm computing a piece of the answer")
    6
}
val b = async {
    log("I'm computing another piece of the answer")
    7
}
log("The answer is ${a.await() * b.await()}")
```

```
[main @coroutine#2] I'm computing a piece of the answer
[main @coroutine#3] I'm computing another piece of the answer
[main @coroutine#1] The answer is 42
```

모두 메인 스레드에서 실행중인 것을 알 수 있다. 



### 스레드 간 전환(Jumping between threads)

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
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
}
```

```
[Ctx1 @coroutine#1] Started in ctx1 
[Ctx2 @coroutine#1] Working in ctx2 
[Ctx1 @coroutine#1] Back to ctx1
```



runBlocking 호출 시 명시적으로 컨텍스트를 설정하고 코루틴의 컨텍스트 전환을 위해 withContext()함수를 사용한 것인데 **결과에서 볼 수 있듯이 전환 중에서 여전히 기존 코루틴을 유지하고 있는 것을 확인할 수 있다.** 

### 컨텍스트 상의 Job (Job in the context)

```kotlin
fun main(args: Array<String>) = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")
}
```

위에서 말했듯이, job은 컨텍스트 그 자체이다. 모든 정보를 가지고 있다.



### 코루틴의 자식 코루틴들 (Children of a coroutine)

어떤 코루틴이 다른 코루틴의 코루틴 스코프 안에서 실행되면, 그것은 부모 코루틴의 CoroutineScope.coroutineContext 를 통해 컨텍스트를 상속하고, 새롭게 생성 된 자식 코루틴의 Job 은 부모 코루틴 Job 의 자식으로 생성된다. 그 결과 **부모 코루틴이 취소되면 모든 자식들 역시 재귀적으로 취소된다.**

하지만 GlobalScope에서 실행 된 코루틴들은  그들이 실행된 스코프에 연관되지 않고 독립적으로 동작한다.

다음 예제를 보면 GlobalScope 에서 실행된 코루틴은 그 자신이 실행된 스코프가 취소되어도 영향을 받지 않는 것을 확인할 수 있다.



### 부모 코루틴의 의무 (Parental responsibilities)

부모 코루틴은 자신에게 속한 모든 자식 코루틴들의 실행이 완료될 때까지 기다린다. 부모는 이를 위해 명시적으로 실행한 모든 자식들을 추적할 필요는 없으며, 자식들의 종료를 기다리기위해 Job.join() 함수를 사용할 필요도 없다.



### 디버깅을 위한 코루틴 이름 지정 (Naming coroutines for debugging)

위에서 말했듯이, 코루틴에 이름을 지정할 수 있으며 이는 보통 디버깅을 위해서 사용된다.



### 컨텍스트 요소의 병합 (Combining context elements)

종종 우리는 코루틴 컨텍스트 요소들 중 두개 이상을 동시에 지정해야 할 경우가 있다. 이 경우 우리는 +(plus) 연산자를 사용하여 코루틴 요소들을 병합할 수 있다. 위에 오형님 발표에 나와있다!



### 명시적인 Job 을 통한 취소 요청 (Cancellation via explicit job)

지금은 안드로이드 애플리케이션을 만들고있고, 안드로이드 액티비티 컨텍스트에서 데이터를 가져오거나 변경하고, 애니메이션을 수행하는 등의 동작을 위해 다양한 코루틴을 실행하고 있다고 생각해 보면, 여기서 사용된 모든 코루틴들은 액티비티가 종료될 때 함께 취소되어야 메모리 누수를 피할 수 있다.

우리는 액티비티의 생명주기와 매핑 된 Job 인스턴스를 생성함으로써 코루틴의 생명주기를 관리할 수 있다. Job 인스턴스는 액티비티가 생성될 때 Job() 팩토리 함수를 이용하여 생성되고 액티비티가 종료될 때 취소된다.



### 스레드 로컬 데이터 (Thread-local data)



## Parental responsibilities

꼭 알아야하는 개념이라고 생각한다.

예기치 못한 동작이 있을 수 있다. 

이 부분은 꼭 공부하기!