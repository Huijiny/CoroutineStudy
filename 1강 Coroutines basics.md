# 1강 Coroutines basics

## What's Coroutine

코루틴은 "일시 중단이 가능한" 계산의 인스턴스로 코드의 나머지 부분과 동시에 작동하는 코드 블록을 실행한다는 점에서 쓰레드와 유사하다. 그러나 코루틴은 특정 쓰레드에 바인딩되어있지 않으며, 일시정지한 실행을 다른 스레드에서 다시 시작할 수 있다. 

>  코루틴은 라이브러리 이름같은게 아니라 개념이다! subroutine과 대비되는 개념이다.예전부터 있던 개념이고 Scala나 PHP도 코루틴을 지원하는 개념이다. 코틀린으로 들어오면서 JVM딴에 도입이 된거라 코루틴 하면 너무 코틀린 코루틴을 생각하기 보다는 코루틴이라는 개념을 떠올리는게 좋을 것!
>
> Rx같은 경우도 개념. 라이브러리가 아니다 :) Rx를 Java로 구현한게 RxJava



## Subroutine vs Coroutine

**Subroutine** 

단일 지점에서 들어와 순차적으로 진행된 뒤 return을 만나 특정지점에서 out

중간에 일시중단이 불가능한데, -> 아주 불가능은 아니다.

흔히 접하는 언어들, C, Java 등 함수호출을 했을 때 함수가 끝이 나려면 return을 만나거나 함수 안에 있는 b0ody가 종료되거나 인데, 진입하고 나오는 지점이 정해져있다. 

그 중간에 일시중지와 재개라는 개념이 없다. 일반적인 형태의 함수들이 subroutine이다. 

**Coroutine**

단일 지점에서 들어와 임의의 지점에서 나간뒤 다시 그 임의의 지점으로 들어온 뒤 다시 특정지점에서 out! 중간에 일시중단이 가능하다.



## Coroutine Basic

`lunch` 

​	코루틴 빌더이다. 

​	새 코루틴을 시작, 독립적으로 

​	코루틴 스코프 안에서만 동작한다.

​	새로운코루틴을 실행해준다. 

`delay`

​	일시중단 기능.

​	하지만 스레드가 차단되는건 아니다.

`runBlocking`

​	안에 있는 코루틴이 완료될 때까지 메인 스레드가 차단된다.

​	fun main()과 runBlocking안의 코루틴이 있는 코드와 다리역할을 해준다. 



#### * 안드로이드에서는 runBlocking을 쓰면 안된다고 알고있는데 이유가 무엇인가요?

> 메인 쓰레드에서 사용할 경우에 오랫동안 runBlocking이 되면 ANR이 일어날 수 있기 때문이다. RunBlocking은 쓰레드가 차단된다. 
>
> 테스트용으로 쓸 때 많이 쓰고, 코드에서는 쓰지말라고 합니다. 
>
> ---
>
> 태환님 블로그 참고
>
> - runBlocking은 호출한 위치를 Blocking 시킨다.
> - runBlocking 내부의 응답이 종료되기 전까지 응답을 주지 않는다.
> - runBlocking은 비동기가 아닌 동기화로 동작한다.
> - UI에서 사용하는 runBlocking은 사용하지 않아야 한다.
> - runBlocking이 필요한 케이스를 찾아야 하는데, 명확한 IO를 보장하고, 데이터의 동기화가 필요한 경우와 UnitTest에서 활용하자.

#### * 그럼 어디서 써야하나요?

> 종료를 확실히 해줄수있는 스코프에서 사용하는게 좋을 것 같다.
>
> 뷰모델에서는 뷰모델스코프, 라이프사이클 스코프에서 사용하는게 좋을 것 같다.

#### * runBlockinge대신 뭘써야하나요?

https://thdev.tech/kotlin/2020/12/15/kotlin_effective_15/

https://developer.android.com/kotlin/coroutines/coroutines-adv?hl=ko#main-safety

https://0391kjy.tistory.com/49

#### * 종료를 해야하는 이유가?

> **불필요한 리소스 낭비 말고 또 다른 이유가 무엇일까요?**
>
> > 계속 살아있어서 메모리를 잡고 있기 때문입니다.
>
> **다른이유가 있다는데 혹시 무엇일까요?**
>
> > 잘 모르겠습니다.
> >
> > 개발자가 추적할 수 없기 때문이아닐까?
> >
> > runBlocking은 예상할 수 없기 때문이 아닐까?
> >
> > onCreate()에서 runBlocking 사용하면 onCreate가 끝나지 않는다.
> >
> > **인증 부분에서는 runBlocking을 적절히 쓸 수 있지않을까?**
> >
> > >  별도로 block처리를하지않고 라이프사이클은 그대로 타고, 다른 추적할 수 있는 상황을 만드는게 좋을 듯 하다. onCreate에 의존하게 하면 안될 것 같다.

#### * 멀티쓰레드를 사용한 구현과 코루틴 사용 구현의 큰 차이점

> 멀티쓰레드는 병렬성을 가진 구현이라면 코루틴은 저런 동시성을 잘 활용할 수 있게 된다.동시성과 병렬성은 많이 다르니까 한 번 찾아보는게 좋을 것 같다. 



### Structured concurrency

코루틴은 구조적인 동시성 원칙을 가져가는데, 특정 CoroutineScope 안에서 시작된다.

코루틴 스코프에서는 수명을 지정해준다.

->실제 앱에서는 수많은 코루틴을 실행하는데, 구조화된 동시성의 원칙은  손실, 누출이 없고 보호가 잘 된다.

| 이전 문서에서는 Global Scope로 구조적 동시성을 설명해주고있다.

![스크린샷 2021-08-07 오후 2.37.18](1강 Coroutines basics.assets/스크린샷 2021-08-07 오후 2.37.18.png)



## delay와 Thread.sleep()의 차이점

Thread.sleep()에서는 suspend를 안써도된다.

delay함수는 코루틴 스코프안에서만 사용할 수 있고, Thread.sleepd은 Thread관련에서 쓰는데,

delay는 coroutine을 Blocking

Thread.sleep은 쓰레드를 Blocking



## suspend 키워드

suspend 키워드를 추가하여 함수를 분할한다.

suspend로 정의한 함수는 사용하기 전까지 동작하지 않는다.

suspend 함수는 CoroutineScope 내에서만 사용할 수 있다. 



## ScopeBuilder와 Concurrency



## Extract function refactoring(추출함수 리팩토링)

Extract function => 함수를 추출

```kotlin
suspend fun doworld() {
	delay(1000L)
  println('asdf')
}
```

## RunBlocking vs CoroutineScope

`runBlocking`

​	 -> 대시를 위해 현재스레드를 차단

​	 -> 일반함수

`coroutineScope` 

​	-> 다른용도를 위해 기본 스레드를 해제

​	-> 일시중단 함수

**이 둘의 차이가 뭐지?**

> 둘 다 끝날때까지 대기한다는 점은 동일하다. runBlocking은 현재 스레드를 블럭 후 코드가 끝날때까지 기다리게 된다. coroutineScope는 일시중단을 하고 다른 작업할 수 있는 코루틴을 찾아다닌다. 

![스크린샷 2021-08-07 오후 2.41.33](1강 Coroutines basics.assets/스크린샷 2021-08-07 오후 2.41.33.png)







![스크린샷 2021-08-07 오후 2.41.56](1강 Coroutines basics.assets/스크린샷 2021-08-07 오후 2.41.56.png)



## Coroutine is lightweight

쓰레드보다 훨!씬! 가볍다!

코루틴은 **단일 쓰레드 내에서 여러 개의 코루틴을 실행할 수 있기 때문에, 많은 양의 동시 작업을 처리**할 수 있으면서 **메모리 절약의 장점**이 있다.

기존 쓰레드는 Context-Switching(CPU가 쓰레드를 점유하면서 실행, 종료를 반복하며 메모리 소모)이 발생하기 때문에 많은 양의 쓰레드를 갖기가 어렵지만 **코루틴은 쓰레드가 아닌 루틴을 일시 중단(suspend) 하는 방식**이라 Context-Switching에 비용이 들지 않기 때문이다.

또한, 지정된 작업 범위 내에서 실행이 되기 때문에 메모리 누수를 방지할 수 있다.



# 끝으로

1. 어떤 쓰레드에서 실행할 것 인지 Dispatchers 를 정하고 (Dispatchers.Main, Dispatchers.IO, Dispatchers.Default)

2. 코루틴이 실행될 Scope를 정하고 (CoroutineScope, ViewModelScope, LifecycleScope, liveData...)

3. launch 또는 async로 코루틴을 실행 시키면 끝!

 **Context**로 **Scope**를 만들고, **Builder**를 이용하여 코루틴을 실행!



## 질문

```kotlin
val job1 = Job()
CoroutineScope(job1 + Dispatchers.Main).launch {
    // 메인 쓰레드 작업

    launch(Dispatchers.IO) {
        // 비동기 작업
    }

    withContext(Dispatchers.Default) {
        // 비동기 작업
    }
}
```

이 코드에서 withContext를 쓰는경우와 안쓰고 lunch에서 Dispatchers.IO로 바꾸는건 무슨차이일까?