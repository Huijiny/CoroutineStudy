# 5장 Asynchronous Flow

Flow를 최대한 다양한 형태로 사용할 수 있게 제공해주는 연산자들을 알아볼 예정.



## flowOn 연산자

downstream과 upstream 

flow는 upstream에서 downstream으로 가는데, 

upstream이랑 downstreamdㅣ 뭐냐?

계속해서 위에서 아래로 뭔가를 계속해서 추가해가는데, map, catch, 등등. 이거를 하나의 스트림이라고 보면 좋고 말그대로 Upstream은 위에 있는 stream, downstream은 아래에 있는 stream이다. 그래서 flow는 Upstream에서 downstreamdㅡ로 흘러가고, 기본적으로 다운스트림에서 업스트림은 건드릴 수 없다.



하지만 flowOn()은 아래서 위에를 영향을 끼치기 때문에 순차성을 일부 포기했다는 의미입니다. 

## Buffering

데이터를 방출하는 속도 < 방출한 데이터를 처리하는 속도 ? -> 버퍼에 들어가있다.

그럼 그 때 버퍼를 써서 데이터 방출된거를 버퍼에서 가지고있다가 처리한다.



## Conflation

 - 버퍼 사용

 - 데이터를 처리하는 동안 쌓인 데이터들 중 지난 데이터는 관심 없고, 최신의 데이터만 처리하고 싶다.

 - 데이터 흐름 중 최신의 날씨 정보만 유의미하다.

   ### 최근 값 처리

   ~Latest 연산자

   기존 연산자와 동일한 로직을 수행하지만 새로운 값이 방출되면 블록안에 실행되고 있던 처리 작업을 취소시킨다. 

   Conflation은 데이터 방출 작업과 처리 작업이 너무 느릴 때 속도를 높이기 좋은 방법

   처리 작업 중에 새로운 데이터가 방출되면 기존 처리 작업을 취소하고 새로운 데이터를 처리하는 방법도 있음.



=> 데이터를 skip하거나 빠르게 처리하는 방법

약간 rx에서 debounce와 같은 것. 안드로이드 앱 개발 입장에서 예시를 들어보자면 검색창에 검색할 때 추천 검색어가 뜨는데 내가 입력할 때마다 모든 글자에 하면 너무 힘들거고, 0.3초마다 측정해서 그 전꺼랑 다르면 방출하고 이런식으로 쓸 수 있다.

지도를 움직였을 때, 지도를 움직이고 약간의 딜레이 후에 내가 움직인 후에 마크들이 갱신되는 경험을 해봤을 텐데, 내가 지도를 움직였을때마다 지도 데이터를 갱신할필요가 없으니, 일정 시간, interval 안에서 가장 나중의 값을 기준으로 서버에 좌표를 던져서 이 구역 안에있는 정보를 줘 이런식으로 활용할 수 있다.





---

## Composing multiple flows.

여러 flow를 합치는 방법들

### zip

- 두 flow를 합치는 zip 연산자를 제공
- 1, 2, 3, 4, 5으로 숫자만 늘리면 어떻게 될까? 1-one, 2-two, 3-three 하고 멈춰있을거다.
- 수집이 필요할 때 호출이 되게 때문에 nums라는 flow는 emit된 데이터를 버퍼에 갖고 있는데 strs 를 다른 데이터를 방출할 수 있는 상태가 되면 그제서야 같이 묶이게 되는 것
- 결과적으로 출력이 되는건 1, 2, 3가되고 숫자 4, 5에 대해서는 출력이 안될거다.

### Combine

- 두 데이터가 1:1로 결합하게 되었을 때 1:1 결합을 한다.

### Flatting flows

- 수신된 값의 시퀀스 -> 각 값을 또 다른 값의 시퀀스를 요청할 수 있음

- 중첩이 가능하다. -> `Flow<Flow<String>>` 반환값을 평탄화 시키는 애들

  #### flatMapConcat

  블록 안의 내부 flow가 완료되어야만 외부 collect를 수행한다.

  내부적으로 Map에 flattenConcat 연산자가 체이닝되는 형태

  겹쳐지는 간격이 없어지는 것

  - -_-_-_ -_-=> -----------

  #### flatMapMerge

  시퀀스에 관계 없이 방출되는 데이터를 최대한 빠르게 flatten 시킨다.

  concurrency라는 파라메터를 받아서 flattenConcat()을 하던지 ChannelFlowMerge를 시키냐 이던데, concurrency의 경우 생산라인이 1개라는 뜻이니까 아까 했던 flattenConcat()을 실행시키고 아니면 ChannelFlowMerge를 시킨다는 것.

  즉, 내부 시퀀스를 Concurrency하게 방출하는 것.

  그래서 아까와 같은 코드를 돌려보면 방출이 끝나는대로 바로바로 진행된다. 그래서 순서가 보장되어있지 않다.

  #### flatMalLatest

  이전에 뭘 처리하고있었던지, 새로운 flow가 방출되면 이전 flow처리가 취소되면서 flatten

  내부적으로 transformLatest { emitAll(Transform(it))}

### Collector try-catch

- try-catch문 사용 가능

### Everything is caught

- emitter, 중간 연산자, 터미널 연산자 등 모든 부분에서의 예외를 잡을 수 있다.
- 중간연산자로 정의하는 부분
- 터미널 연산자로 실행하는 부분에서 try-catch로 실행하는 부분

### Exception transparency

flow는 예외에 있어 투명해야한다 -> 디버깅 난이도와 직결

Try-catch문으로 감싸서 모든 에러를 catch할 수 있음이 보장되는 장점이 있다.

투명성을 보장하고 싶으면 -> catch를 사용할 수 있다.

throw: 예외를 다시 던지기 가능

emit: 새로운 값을 다시 방출



### Transparent catch

- catch 도 중간 연산자로 업스트림의 예외들만 잡아내면서 투명성 보장한다.
- 그래서 그 아래 collect문에서 에러는 못잡는다.

### Catching declartively

OnEach를 사용해라.



## Flow completion

- flow는 방출된 데이터들을 정상적으로 모두 처리했거나 예외가 발생했으 ㄹ때 모두 완료한것으로 간주. 보통 해당 처리가 모두 끝나면 특정한 작업 수행 가능

### Imperative finally block



### Declarative handling

- 명시적인 방법으로, onCompletion 중간 연산자가 있다.
- 모두 끝나면 Done을 출력해주세요.
- 람다식에서 Nullable한 Throwable이 이 파라미터로 전달되기 때문에 collect가 정상적으로 완료되었는지, 예외가 발생해서 종료된 것인지.

``` 
simple()
.onCompletion{println("DONE")}
.collect{ value -> println(value)}
```



### Successful completion

- catch -> 업스트림에서만 예외 찾기 가능
- onCompletion -> 업스트림과 다운 스트림 모두에서 발생하는 예외를 주시한다.
- flow가 완전히 성공적으로 완료된 경우에만 Null을 수신한다.
- 다운스트림에서 예외가 발생했기 때문에 flow가 완료되지 못했고, onCompletion의 파라미터가 null이 아니게 되었다. 

## Imperative versus declarative

- 모두 숙지후에 자신의 스타일과 상황에 맞게 사용하자!



## Launching flow

- 어떤 source로부터 전달되는 비동기적인 event를 flow를 이용해서 쉽게 표현할 수 있다.
- 이 경우 event가 발생할 때마다 특정 동작을 처리할 수 있도록 하고, 그 이후의 다른 코드는 계속 진행되도록 리스너와 같은 기능이 필요하다.
- onEach는 중간 연산자이므로 자체적으로 데이터를 처리(collect)할 수 없다.
  - 비어 있는 collect를 호출하자



### LanchIn 연산자를 대신 사용

concurrency하게 실행된다. 

코드 들어가보면 Lunch가 있으니까 각각의 애들이 각각의 런치에서 실행되는 것

위 예제의 경우 this를 사용했기 때문에 flow가 완료되어야 상위 runBlocking도 종료될 것.

다른 Lifetime을 갖는 entity의 scope를 받아 사용했다면 해당 lifetime이 종료되면서 해당 flow가 cancel될 것.

lunchIn은 Job을 리턴하기 때문에 필요에 따라 스코프의 취소 없이도 특정 flow만 취소할 수 있고, join을 이용해서 대기시킬 수도 있다.



### Flow cancellation checks

Flow Builder는 데이터 방출 시 `ensureActive`를 수행한다.

​	현재 스코프가 active상태인지 확인

​	액티브 상태가 아니면 CancellationException을 발생

즉 반복문을 통한 데이터 방출은 취소가 가능하다.



## Flow and Reactive Streams

위에서 언급한 내용들은 Reactive streams 개념과 비슷한 것 같다.

Flow의 궁극적인 목표

최대한 간단한 디자인, Kotlin 스럽고 suspension 친화적인 

