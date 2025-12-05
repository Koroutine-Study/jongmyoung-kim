# 5장 정리 글
### [5장 정리 글](https://jongmyoung.tistory.com/entry/Kotlin-%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%BD%94%EB%A3%A8%ED%8B%B4%EC%9D%98-%EC%A0%95%EC%84%9D-5%EC%9E%A5)

# 4주차 발표 내용

## 1. Deferred 객체의 생성과 구조

`async`는 비동기 작업의 결과를 담을 수 있는 `Deferred` 객체를 반환한다.  
내부적으로 어떻게 미래의 결과를 약속하는 객체를 만들어낼까?

`async` 빌더의 구현부:
```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
```

핵심은 `DeferredCoroutine`을 생성한다는 점이다. `CoroutineStart`의 옵션이 `Lazy`라면 `LazyDeferredCoroutine`을,
그 외엔 `DeferredCoroutine`을 생성하여 반환한다.

```kotlin
private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T> {
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = onAwaitInternal as SelectClause1<T>
}

private class LazyDeferredCoroutine<T>(
    parentContext: CoroutineContext,
    block: suspend CoroutineScope.() -> T
) : DeferredCoroutine<T>(parentContext, active = false) {
    private val continuation = block.createCoroutineUnintercepted(this, this)

    override fun onStart() {
        continuation.startCoroutineCancellable(this)
    }
}
```

내부 구현을 살펴보면, `LazyDeferredCoroutine` 역시 `DeferredCoroutine`을 상속받은 서브 클래스임을 알 수 있다.  
그리고 `DeferredCoroutine`은 `AbstractCoroutine`을 상속받고 있다.

그럼 `AbstractCoroutine`은 무엇인가?
```kotlin
@OptIn(InternalForInheritanceCoroutinesApi::class)
@InternalCoroutinesApi
public abstract class AbstractCoroutine<in T>(
    parentContext: CoroutineContext,
    initParentJob: Boolean,
    active: Boolean
) : JobSupport(active), Job, Continuation<T>, CoroutineScope {
```

`JobSupport`, `Job`, `Continuation`, `CoroutineScope`를 모두 구현하는 것을 알 수 있다.

```kotlin
public open class JobSupport constructor(active: Boolean) : Job, ChildJob, ParentJob {
    final override val key: CoroutineContext.Key<*> get() = Job

    private val _state = atomic<Any?>(if (active) EMPTY_ACTIVE else EMPTY_NEW)

    /* 
    Returns current state of this job. If final state of the job is Incomplete, 
    then it is boxed into IncompleteStateBox and should be unboxed before returning to user code.
     */
    internal val state: Any? get() = _state.value
}
```

```kotlin
protected suspend fun awaitInternal(): Any? {
        // fast-path -- check state (avoid extra object creation)
        while (true) { // lock-free loop on state
            val state = this.state
            if (state !is Incomplete) {
                // already complete -- just return result
                if (state is CompletedExceptionally) { // Slow path to recover stacktrace
                    recoverAndThrow(state.cause)
                }
                return state.unboxState()

            }
            if (startInternal(state) >= 0) break // break unless needs to retry
        }
        return awaitSuspend() // slow-path
    }
```

`DeferredCoroutine`을 보면 `await` 함수가 `awaitInternal`을 호출하는 것을 확인할 수 있다.  
이때 `_state`가 현재 **완료된 상태**인지, **아직 진행 중인 상태**인지에 따라 반환 값이 달라진다.

1. 이미 완료된 경우 (Fast Path)  
`await()`을 호출했을 때 무조건 코루틴을 멈추는 것은 비효율적이다.
따라서 `state`를 체크함으로써 `JobSupport`의 상태가 대기자 리스트가 아닌 결과값으로 이미 변해 있다면,
불필요한 객체 생성이나 스레드 문맥 교환 없이 즉시 값을 반환한다. 이를 **Fast Path**라고 한다.

2. 아직 완료되지 않았을 때 (Slow Path)  
`awaitInternal` 함수 마지막 줄에 있는 `awaitSuspend()`는 작업이 아직 진행 중일 때 호출된다.  

```kotlin
private suspend fun awaitSuspend(): Any? = suspendCoroutineUninterceptedOrReturn { uCont ->
    /*
     * Custom code here, so that parent coroutine that is using await
     * on its child deferred (async) coroutine would throw the exception that this child had
     * thrown and not a JobCancellationException.
     */
    val cont = AwaitContinuation(uCont.intercepted(), this)
    // we are mimicking suspendCancellableCoroutine here and call initCancellability, too.
    cont.initCancellability()
    cont.disposeOnCancellation(invokeOnCompletion(handler = ResumeAwaitOnCompletion(cont)))
    cont.getResult()
}
```
핵심은 `suspendCoroutineUninterceptedOrReturn`을 통해 현재 코루틴을 멈추고, `Continuation` 객체를 `AwaitContinuation`이라는 노드로 포장하여
`Job`의 대기자 리스트에 등록한다는 점이다.

즉, 미완료 상태의 `Deferred`는 `_state` 변수를 통해 결과를 기다리는 대기자들의 목록을 관리한다.

그렇다면 `async` 블록의 작업이 끝나고 결과값이 반환되면 어떤 일이 벌어지나?  
`Deferred`는 내부적으로 작업 완료 시 `makeCompleting` 로직을 수행하며, 이때 `_state` 변수의 역할이 바뀐다.
1. 상태 덮어쓰기 (Atomic Swap): 기존에 `_state`가 들고 있던 대기자 리스트가 최종 결과값으로 교체된다.  
(이후 들어오는 await 요청은 Fast Path를 통해 즉시 값을 가져간다.)

```kotlin
// 1. 네트워크 요청은 딱 1번 실행됨
val deferredUser = async { api.fetchUser() } 

launch { 
    // 2. A 작업에서 user 정보가 필요함
    val user = deferredUser.await() // 여기서 기다렸다가 결과 받음
    updateUI(user)
}

launch { 
    // 3. B 작업에서도 user 정보가 필요함
    // A가 await 할 때 이미 작업이 끝났다면, B는 기다리지 않고 'Fast Path'로 즉시 값을 가져감
    val user = deferredUser.await() 
    saveToDb(user)
}
```

2. 대기자 깨우기 (Resume): 방금 전까지 `_state`에 있던 대기자 리스트의 노드들을 순회한다.  
각 노드에 저장된 `Continuation`의 `resume`을 호출하여, 잠들어 있던 코루틴들을 깨우고 결과값을 전달한다.

그런데 여기서 한 가지 의문이 생긴다.  
`async` 블록이 끝났다는 사실을 `Deferred`는 어떻게 알고 `makeCompleting`을 호출하는 것일까?

`DeferredCoroutine`은 `AbstractCoroutine`을 상속받고, `AbstractCoroutine`은 `Continuation` 인터페이스를 구현하고 있다.

```kotlin
public abstract class AbstractCoroutine<in T>() : JobSupport(active), Job, Continuation<T> {
    public final override fun resumeWith(result: Result<T>) {
        val state = makeCompletingOnce(result.toState())
        if (state === COMPLETING_WAITING_CHILDREN) return
        afterResume(state)
    }
    /*
     AbstractCoroutine은 Continuation 인터페이스를 구현하고 있다. 즉, this는 Continuation
     */
    public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }
}
```
1. 시작 시점: `async`가 시작될 때, 자기 자신을 `completion` 컨티뉴에이션으로 등록한다.
2. 종료 시점: `async` 블록의 코드가 실행을 마치면, 컴파일러가 생성한 코드는 자동으로 `completion.resumeWith(결과)`를 호출한다.
3. 완료 처리: 이때 호출되는 `resumeWith`가 내부적으로 `makeCompletingOnce`를 실행한다.
이로써 `JobSupport`의 상태가 '대기 중'에서 '완료'로 확정되며, 대기열에 잠들어 있던 코루틴들이 깨어나 결과값을 수령하게 되는 것이다.

### 결?론?
`Deferred`의 동작 원리는 `JobSupport` 클래스 내부의 `_state` 변수를 활용한 다형성과 원자적 상태 전이로 설명할 수 있다.
1. `Deferred`의 구현체인 `JobSupport`는 상태 관리를 위해 `_state`라는 `atomic<Any?>` 타입의 변수를 사용한다. `Any?` 타입으로 선언되어 있어 런타임에 참조하는 객체의 타입을 동적으로 변경할 수 있다.
2. `NodeList` 참조 작업이 완료되기 전, `_state`는 `Incomplete` 인터페이스를 구현한 객체를 참조한다. 이 객체는 `await()` 호출로 인해 일시 중단된 코루틴들의 실행 문맥을 연결 리스트 구조로 관리하는 헤드 노드 역할을 수행한다.
3. `Result Value` 참조 작업이 종료되면, `makeCompleting` 로직을 통해 `CAS(Compare-And-Swap)` 연산이 수행된다. 이때 `_state` 변수가 참조하던 객체는 대기자 리스트(NodeList)에서 최종 연산 결과값으로 원자적으로 교체된다.
이후, 교체되기 전 리스트에 있던 대기자(Continuation)들을 순회하며 `resume`을 호출하여 결과를 전달하고 깨운다.
4. 즉, `Deferred`는 단일 메모리 공간의 참조 대상을 생명주기에 따라 리스너 관리 객체에서 데이터 객체로 형변환하여 사용하는 것
