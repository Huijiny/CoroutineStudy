# 3강 Composing suspending function

GlobalScope로일시중단함수 작성

**GlobalScope** : 앱이 실행될 때부터 종료될 때까지 실행

## 기본적으로 순차

기본적으로 순차적으로 진행된다.

## Concurrent using async

하지만 만약에 async를 사용하게 된다면?

코틀린에서는 async가 바로 실행이된다. 

lunch는 Job을 반환하고 

Async는 Deffered를 반환한다. 이 친구도 Job이기 때문에 cancel할 수 있다. 

나중에 결과를 제공하겠다는 약속을 나타내는 lightweight non-blocking future

async를 사용하여 더욱 빠르게 :)

## Lazily started async

어떤 매개변수를 `CoroutineStart.Lazy`로 설정해 비동기를 지연시킬 수 있다. 

이렇게 되면 await()가 실행되었을 때 실행된다.

one.start()로 시작이 되고, one.await는 one.start를 기다려준다.

>  start를 안넣어보면 어떻게되나요?
>
> > 2초가 걸린다. 결과가 start를 빼도 나오긴 한다. await가 실행도 시켜준다고 생각하면 된다. 
> >
> > await은 기다리는 애도 있으니까 시작하고 걸리고 시작하고 걸려서 2초가 걸리는거. start는 시작해주는애니까 동시에 시작해주기때문에 1초걸린거.

## Async - style functions

글로벌 스코프에서 구현된 두가지 함수

글로벌 스코프는 object다? => 어디에서건 글로벌 스코프를 호출하는건 동일한 인스턴스를 호출하는게 맞는 것 같다. 이렇게 따로 호출을 했을 때 하나의 묶음으로 동작하는게 아니다. 

https://thdev.tech/kotlin/2020/12/22/kotlin_effective_16/

> 추측상) GlobalScope에 async라는 호출하면 저게 분리되서 두 가지가 되는게 아닐까

> **await 전에 캔슬을 하게되면 one.cancel()으 하면 어떻게 될까?** 
>
> > -> 에러가 난다. 방지는 .isCancelled같은걸로 취소 체크를 먼저 해야한다. 
> >
> > await이라는 애가 불렸을 때 얘가 캔슬된 상태냐 안니냐.  중요.
>
> **.start()은 lazy에서만 사용하는 친구**



### Structured concurrency

글로벌 스코프에서 구현된 두가지 함수를 두가지를 하나의 스코프로 묶어주는게 좋다. 

집을 짓는다. -> 배관설치, 가구 넣고, 마루바닥 뜯어내기 등의 작업이 있을텐데 그거중에 한개라도 에러가 나면 집을 짓는다는게 안된다. 그런식으로 접근하면 되는게 structured concurrency라고 하면 된다.

예외가 발생하면 같은 스코프에 있는 모든 코루틴이 취소된다.  한개가 망가지면 다 망가지게 해주는 것!



## 질문

### 코루틴 생성시 자동으로 생성되는 스레드는 몇 개일까?

12개 정도. Core * hyper스레딩를 곱해서 12개까지 늘릴 수 있다. 

정확하게는 코어개수 -1

코어당 1개!!

	#### 1-1디폴트와 io스레드는 뭐가 다를까?

https://sandn.tistory.com/110  

io스레드로 하면 64개 근방까지 생성된다고 한다.

io는 블로킹될 수도 있어서 그런게 아닐까.. 



### coroutineScope, lunch, runBlocking의 차이

coroutineScope VS CoroutineScope

CoroutineScope는 인터페이스, coroiutineScope는 함수

코루틴스코프에서 Job들이 어떤 순서로 돌아가게될까?

큐로 동작한다고 가정 > cancel은 실제로 어떻게 동작하는걸까?

#### **cps란?** 라벨로 해주어서 중간중간 끊을 수 있게 해준다. **Continuation passing style**

Suspend function에다가 함수의 리턴타입으로 반환값에 무엇인가를 던지는 걸 해봤을텐데 

서스펜드함수는 우리가 지정한 객체를 던지는게 아니라 Continuation이라는 객체를 던져준다.

얘를 통해서 원하는 결과값을 받게끔 구조가 되어있는 것.

그걸 코틀린 컴파일러가 알아서 변환해주는거기때문에 suspend fun의 리턴타입으로 우리가 설정한 객체가 리턴타입으로 넘어간다고 생각해주는거다.

#### 그러면 이 suspend fun의 작동 순서는? 

추측한 바로는, queue로 기본적으로 동작하는 듯 함.

> 내부적으로 queue인지는 모르겠다.
>
> 코루틴 컨텍스트가 누구냐가 중요하고, 코루틴 스타트라는 객체가 있는데 여기에 옵션이 4가지가 있다. 레이지, 디폴트, 아토믹, 언디스패치드, 이거에 따라서 동작이 달라질 수도있다.
>
> bfs처럼 타고타고 들어가서 큐에 다 넣고 아래로 내려가는 형태가 아니었던 이유는 람다이기 때문이다. 

queue로 동작을 하고 이걸 cancel할 때도 있는데 cancel할때는 key값으로 hashmap 비슷하게 접근해서 cancel시켜주는걸로 알고있다. 



### suspended point 

큐에들어갔다가 이 포인트를 만나면 여기서 나온다.

