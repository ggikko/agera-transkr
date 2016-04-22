# agera-transkr

Agera는 새로운 앱을 구축하는데 있어서 가장 적합한 코드 스타일입니다. 이 페이지는 개발자들이 기존 코드를 Agera(_Agerify_)하게 바꿀 수 있도록 몇 가지 팁을 소개합니다.  

# 업그레이드 기존 옵저버 패턴

Observer 패턴은 구현방법이 다양하기 때문에 일부 코드만 Agera-style observable-updatable 클래스 구조로 쉽게 변경할 수 있습니다. 아래에는 기존 “listenable” 클래스에 `Observable` 인터페이스를 추가하는 하나의 방법입니다.

`SomeBaseClass`를 상속함으로서, `MyListenable` 클래스는 listeners(`Listener` interface 구현체들)이 `addListener`와 `removeListener`를 통해서 추가되고 제거될 수 있도록 합니다. 이 예제에서는 single-base-class 제약조건을 피하기 위하여 update dispatcher를 사용하고 이 클래스의 두 세대를 연결하기 위해서 내부 클래스 `Bridge`를 사용합니다. 이는 모든 기존 API들을 Agera-observable 하게 만듭니다.


```java
    public final class MyListenable extends SomeBaseClass implements Observable {
    
      private final UpdateDispatcher updateDispatcher;
    
      public MyListenable() {
        // 기존 생성자 코드는 여기에...
        updateDispatcher = Observables.updateDispatcher(new Bridge());
      }
    
      // 기존 Body 코드는 여기에:
      public void addListener(Listener listener) { … }
      public void removeListener(Listener listener) { … }
    
      @Override
      public void addUpdatable(Updatable updatable) {
        updateDispatcher.addUpdatable(updatable);
      }
    
      @Override
      public void removeUpdatable(Updatable updatable) {
        updateDispatcher.removeUpdatable(updatable);
      }
    
      private final class Bridge implements ActivationHandler, Listener {
        @Override
        public void observableActivated(UpdateDispatcher caller) {
          addListener(this);
        }
    
        @Override
        public void observableDeactivated(UpdateDispatcher caller) {
          removeListener(this);
        }
    
        @Override
        public void onEvent() { // Listener구현
        
          updateDispatcher.update();
        }
      }
    }
```
# Exposing synchronous operations as repositories

자바는 본질적으로 동기(synchronous) 언어입니다. 자바에서 가장 낮은 수준의 연산들 동기로 처리 됩니다. 어떤 값을 반환하는 연산이 진행될 때, 일반적으로 다른 연산들은 진행될 수 없기 때문에, 개발자들은 연산이 진행되고 있는 쓰레드(main thread)에서 다른 함수를 호출하게 되면 주의를 받습니다.

앱의 UI에서 어떤 데이터를 blocking 함수를 호출하여 데이터를 얻으려한다고 가정해봅시다. Agera의 컴파일된 저장소는 실제 함수 호출을 백그라운드 실행자에게 쉽게 넘겨줍니다. 그리고 쓰레딩 접촉때문에, UI는 데이터를 저장소를 관찰함으로서 진행되고 있는 쓰레드(main thread)에서 자유롭게 사용할 수 있습니다. 첫째로, 함수 호출은 하나의 Agera 작업자(operator)에 의해 감싸져야합니다. 아래 코드와 같이:

```java
    public class NetworkCallingSupplier implements Supplier<Result<ResponseBlob>> {
      private final RequestBlob request = …;
    
      @Override
      public Result<ResponseBlob> get() {
        try {
           ResponseBlob blob = networkStack.execute(request); // blocking 호출
           return Result.success(blob);
        } catch (Throwable e) {
           return Result.failure(e);
        }
      }
    }
    
    Supplier<Result<ResponseBlob>> networkCall = new NetworkCallingSupplier();
    
    Repository<Result<ResponseBlob>> responseRepository =
        Repositories.repositoryWithInitialValue(Result.<ResponseBlob>absent())
            .observe() // 이벤트가 없는 소스; 활성화 작업
            .onUpdatesPerLoop() // 컴파일에 필요한 코드
            .goTo(networkingExecutor)
            .thenGetFrom(networkCall)
            .compile();
```

위의 코드 단락은 저장소가 컴파일 되기 전에 요청이 알려지거나 결코 변경되지 않는 다는 것을 보여줍니다. 이는 저장소의 활동 생명주기와 동시에 또는 동적으로 요청에 대한 변화에 응답으로서 쉽게 업그레이드 될 수 있습니다. 변경하도록 요청을 하기 위해서는 단순히 가변 저장소(mutable repository)를 사용하면 됩니다. 선택적으로, 가장 첫번째 요청이 저장소가 만들어진 후에 제공되기 위해서는 `Result`에서 요청을 감싸거나 `absent()` 메소드를 갖고있는 가변 저장소(mutable repository)를 초기 설정을 해야합니다. 가변 저장소(mutable repository)의 용법은 가변  변수(mutable variable)(선택적으로 null가능)의 사용과 유사합니다. 이런 이유에서 그 이름은 요청변수(`requestVariable`)입니다.

```java
    // MutableRepository<RequestBlob> requestVariable =
    //     mutableRepository(firstRequest);
    // 또는:
    MutableRepository<Result<RequestBlob>> requestVariable =
        mutableRepository(Result.<RequestBlob>absent());
```

그리고 공급자(supplier)에서 blocking 함수를 호출을 감싸는 대신에, 동적인 요청을 잡아내기 위해 함수를 사용하십시오.

```java
    public class NetworkCallingFunction
        implements Function<RequestBlob, Result<ResponseBlob>> {
      @Override
      public Result<ResponseBlob> apply(RequestBlob request) {
        try {
           ResponseBlob blob = networkStack.execute(request);
           return Result.success(blob);
        } catch (Throwable e) {
           return Result.failure(e);
        }
      }
    }

    Function<RequestBlob, Result<ResponseBlob>> networkCallingFunction =
        new NetworkCallingFunction();
```
업그레이드된 저장소는 다음과 같이 컴파일 될 수 있습니다 :
```java
    Result<ResponseBlob> noResponse = Result.absent();
    Function<Throwable, Result<ResponseBlob>> withNoResponse =
        Functions.staticFunction(noResponse);
    Repository<Result<ResponseBlob>> responseRepository =
        Repositories.repositoryWithInitialValue(noResponse)
            .observe(requestVariable)
            .onUpdatesPerLoop()
            // .getFrom(requestVariable) if it does not supply Result, 또는:
            .attemptGetFrom(requestVariable).orEnd(withNoResponse)
            .goTo(networkingExecutor)
            .thenTransform(networkCallingFunction)
            .compile();
```
위의 코드 단락은 작업자들(operators)에게 특정 이름을 부여 함으로서 더 가독성이 좋게하는 저장소 컴파일 표현 방법중에 하나입니다.

# Wrapping asynchronous calls in repositories

요즘 많은 라이브러리들은 비동기 API와 내장된 쓰레딩(threading) 기능(클라이언트가 제어 또는 사용하지 못하도록 할 수 없는)들을 가지고 있습니다. 코드에서 이러한 라이브러리를 갖는 것은 전체 앱을 Agerify 하는 것이 더 도전적일 수 있습니다. 명백한 해결책은 그 라이브러리의 동기(syncrhonouse) 대안을 찾고 [[pattern demonstrated above|Incrementally-Agerifying-legacy-code#exposing-synchronous-operations-as-repositories]]을 적용하는 것입니다. 반대 해결책(안티패턴)은 백그라운드 쓰레드에서 비동기 호출을 수행하고 thread가 blocking 당할 동안 그 결과를 기다리고 결과를 동기로 반환하는 것입니다. 이 부분은 위의 명백한 해결책이 불가피할 때 적절한 제 2의 해결책에 대해서 논의합니다.


비동기 호출의 재귀 패턴은 요청-응답 구조입니다. 아래의 예시는 그 구조의 세부사항을 보여주고 있습니다. 이는 또한 끝나지 않은 작업이 취소되도록 허용해주고, 그렇지 않으면 콜백이 유발되는 쓰레드를 구체적으로 명시하지 않습니다.


```java
    interface AsyncOperator<P, R> {
      Cancellable request(P param, Callback<R> callback);
    }
    
    interface Callback<R> {
      void onResponse(R response); // 어떤 쓰레드에서도 호출될 수 있음.
    }
    
    interface Cancellable {
      void cancel();
    }
```

아래 코드의 저장소는 주어진 `비동기Operator`(`AsyncOperator`)로부터 요청에 대한 응답을 노출시킵니다. 여기서 요청은 각각의 저장소(공급자와 함께 추상화된)의 활동에 의해 결정된 파라미터의 요청을 말합니다. 이 코드는 근본적으로 비동기Operator가 이미 적절한 캐싱을 함으로써 중복된 요청은 수행을 못하게 하는 것을 보여줍니다. 

```java
    public class AsyncOperatorRepository<P, R> extends BaseObservable
        implements Repository<Result<R>>, Callback<R> {
    
      private final AsyncOperator<P, R> asyncOperator;
      private final Supplier<P> paramSupplier;
    
      private Result<R> result;
      private Cancellable cancellable;
    
      public AsyncOperatorRepository(AsyncOperator<P, R> asyncOperator,
          Supplier<P> paramSupplier) {
        this.asyncOperator = asyncOperator;
        this.paramSupplier = paramSupplier;
        this.result = Result.absent();
      }
    
      @Override
      protected synchronized void observableActivated() {
        cancellable = asyncOperator.request(paramSupplier.get(), this);
      }
    
      @Override
      protected synchronized void observableDeactivated() {
        if (cancellable != null) {
          cancellable.cancel();
          cancellable = null;
        }
      }
    
      @Override
      public synchronized void onResponse(R response) {
        cancellable = null;
        result = Result.absentIfNull(response);
        dispatchUpdate();
      }
    
      @Override
      public synchronized Result<R> get() {
        return result;
      }
    }
```

이 클래스는 요청 파라미터를 변경하게 함으로써 쉽게 업그레이드 될 수 있습니다. 그리고 처리는 초기 논고와 유사합니다: 요청파라미터가 저장소를 통해서 제공되게하고 `AsyncOperatorRepository`가 요청파라미터 변화를 관찰하게 합니다. 다음과 같이 요청파라미터의 변화를 관찰하는 것과 활성화에 있어서 어떤 진행중인 요청을 취소하거나 새로운 요청을 보냅니다 :

```java
    public class AsyncOperatorRepository<P, R> extends BaseObservable
        implements Repository<Result<R>>, Callback<R>, Updatable {

      private final AsyncOperator<P, R> asyncOperator;
      private final Repository<P> paramRepository;

      private Result<R> result;
      private Cancellable cancellable;

      public AsyncOperatorRepository(AsyncOperator<P, R> asyncOperator,
          Repository<P> paramRepository) {
        this.asyncOperator = asyncOperator;
        this.paramRepository = paramRepository;
        this.result = Result.absent();
      }

      @Override
      protected void observableActivated() {
        paramRepository.addUpdatable(this);
        update();
      }

      @Override
      protected synchronized void observableDeactivated() {
        paramRepository.removeUpdatable(this);
        cancelOngoingRequestLocked();
      }

      @Override
      public synchronized void update() {
        cancelOngoingRequestLocked();
        // 만약 paramRepository가 Result를 제공하면 상황에 맞게 조정
        cancellable = asyncOperator.request(paramRepository.get(), this);
      }

      private void cancelOngoingRequestLocked() {
        if (cancellable != null) {
          cancellable.cancel();
          cancellable = null;
        }
      }

      @Override
      public synchronized void onResponse(R response) {
        cancellable = null;
        result = Result.absentIfNull(response);
        dispatchUpdate();
      }
      
      // 전형적으로 오류가 있을 수 있는 요청을 위한 유사한 처리
      // onError(Throwable) callback): Result에서 실패를 
      //wrapping하고 dispatchUpdate() 호출

      @Override
      public synchrnonized Result

      <R> get() {
        return result;
      }
    }
```

수정중

https://github.com/ZeroBrain/agera-wiki-kr/wiki/Incrementally-Agerifying-legacy-code
