# 3, 4, 5장 정리 글
### [3장 정리 글](https://jongmyoung.tistory.com/entry/Kotlin-%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%BD%94%EB%A3%A8%ED%8B%B4%EC%9D%98-%EC%A0%95%EC%84%9D-3)  
### [4장 정리 글](https://jongmyoung.tistory.com/entry/Kotlin-%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%BD%94%EB%A3%A8%ED%8B%B4%EC%9D%98-%EC%A0%95%EC%84%9D-4%EC%9E%A5)  
### [5장 정리 글](https://jongmyoung.tistory.com/entry/Kotlin-%EC%BD%94%ED%8B%80%EB%A6%B0-%EC%BD%94%EB%A3%A8%ED%8B%B4%EC%9D%98-%EC%A0%95%EC%84%9D-5%EC%9E%A5)

# 2주차 발표 내용

## 1. `launch` 코루틴이 @coroutine#2인 이유

```Kotlin
fun main() = runBlocking<Unit> {
	val dispatcher = newSingleThreadContext(name = "SingleThread")
	launch(context = dispatcher) {
		println("[${Thread.currentThread().name}] 실행")
	}
}

/*
// 결과:
[SingleThread @coroutine#2] 실행
*/
```

`runBlocking` 자체가 첫 번째 코루틴(@coroutine#1)을 생성하기 때문이다.  
`launch`는 `runBlocking`의 스코프 안에서 호출되어, 그 자식으로 두 번째 코루틴을 생성하기 때문에 @coroutine#1이 아닌 @coroutine#2가 된다.

따라서 `println`을 실행하는 주체는 두 번째 코루틴이며, `newSingleThreadContext`에 의해 SingleThread라는 이름의 스레드에서 실행되므로 [SingleThread @coroutine#2]라는 결과가 나온다.

## 2. Dispatchers.Default와 Dispatchers.IO의 공유 스레드풀 분석

코루틴의 `Dispatchers.Default`와 `Dispatchers.IO`는 `Main` 디스패처와 달리 단 하나의 공유 스레드풀을 사용한다.  
이 공유 메커니즘과 초기 상태는 `kotlinx.coroutines`의 내부 코드를 통해 명확히 확인할 수 있다.

### 2.1. CoroutineScheduler의 생성

모든 것은 `Dispatchers.Default`에서 시작된다. `Default` 디스패처는 `DefaultScheduler`라는 싱글톤 객체이다.

```kotlin
public actual object Dispatchers {
    @JvmStatic
    public actual val Default: CoroutineDispatcher = DefaultScheduler
}
```

`DefaultScheduler`는 `SchedulerCoroutineDispatcher` 클래스를 상속하며, 이 `SchedulerCoroutineDispatcher`는 생성되는 시점에 
`private var coroutineScheduler = createScheduler()` 코드를 통해 `CoroutineScheduler` 객체를 생성한다.

```kotlin
// Instance of Dispatchers.Default
internal object DefaultScheduler : SchedulerCoroutineDispatcher(
    CORE_POOL_SIZE, MAX_POOL_SIZE,
    IDLE_WORKER_KEEP_ALIVE_NS, DEFAULT_SCHEDULER_NAME
)
```

```kotlin
// Instantiated in tests so we can test it in isolation
internal open class SchedulerCoroutineDispatcher(
    private val corePoolSize: Int = CORE_POOL_SIZE,
    private val maxPoolSize: Int = MAX_POOL_SIZE,
    private val idleWorkerKeepAliveNs: Long = IDLE_WORKER_KEEP_ALIVE_NS,
    private val schedulerName: String = "CoroutineScheduler",
) : ExecutorCoroutineDispatcher() {

    override val executor: Executor
        get() = coroutineScheduler

    // This is variable for test purposes, so that we can reinitialize from clean state
    private var coroutineScheduler = createScheduler()

    private fun createScheduler() =
        CoroutineScheduler(corePoolSize, maxPoolSize, idleWorkerKeepAliveNs, schedulerName)
}
```

## 2. 공유 스레드풀의 초기 상태
`CoroutineScheduler` 객체가 생성될 때, 스레드풀의 초기 상태가 결정된다.  
코드 내의 `val workers = ResizableAtomicArray<Worker>((corePoolSize + 1) * 2)` 선언은 스레드를 미리 생성하는 것이 아니라,
앞으로 생성될 `Worker 객체(스레드)`를 담아둘 비어있는 배열 공간을 미리 확보하는 작업이다.

```kotlin
    /**
     * State of worker threads.
     * [workers] is a dynamic array of lazily created workers up to [maxPoolSize] workers.
     * [createdWorkers] is count of already created workers (worker with index lesser than [createdWorkers] exists).
     * [blockingTasks] is count of pending (either in the queue or being executed) blocking tasks.
     *
     * Workers array is also used as a lock for workers' creation and termination sequence.
     *
     * **NOTE**: `workers[0]` is always `null` (never used, works as sentinel value), so
     * workers are 1-indexed, code path in [Worker.trySteal] is a bit faster and index swap during termination
     * works properly.
     *
     * Initial size is `Dispatchers.Default` size * 2 to prevent unnecessary resizes for slightly or steadily loaded
     * applications.
     */
    @JvmField
    val workers = ResizableAtomicArray<Worker>((corePoolSize + 1) * 2)
```

소스 코드의 주석에 **[workers] is a dynamic array of lazily created workers...** 라고 적혀 있는 것을 확인할 수 있다. 

`지연 생성`이라는 명시는 `CoroutineScheduler` 관리자 객체가 처음 생성될 때는 실제 워커 스레드(Worker Thread)가 0개이며,
첫 번째 작업이 요청될 때 비로소 스레드가 생성됨을 의미한다.

## 3. Dispatchers.IO가 동일 엔진을 공유하는 방식

`Dispatchers.IO` 역시 이 `CoroutineScheduler`를 공유한다.  
`IO` 디스패처는 `DefaultIoScheduler` 객체를 가리키며,
이 객체는 스케줄러를 새로 만들지 않고 내부적으로 `UnlimitedIoScheduler`의 `View`를 사용한다. (View에 대해서는 후술할 예정)

```kotlin
    @JvmStatic
    public val IO: CoroutineDispatcher get() = DefaultIoScheduler
```


```kotlin
// Dispatchers.IO
internal object DefaultIoScheduler : ExecutorCoroutineDispatcher(), Executor {

    private val default = UnlimitedIoScheduler.limitedParallelism(
        systemProp(
            IO_PARALLELISM_PROPERTY_NAME,
            64.coerceAtLeast(AVAILABLE_PROCESSORS)
        )
    )
}
```

`IO`가 동일한 스케줄러를 사용하는 것은 `UnlimitedIoScheduler`의 `dispatch` 메소드 구현을 확인하면 알 수 있다.  
`IO` 작업의 본체인 `UnlimitedIoScheduler`는 작업을 받으면, 1단계에서 생성된 `DefaultScheduler`에게 `BlockingContext`라는 태그를 붙여 작업을 직접 전달한다.

여기서 `BlockingContext`는 `Default`와 `IO` 작업의 가장 큰 차이, 즉 '해당 작업이 스레드를 블로킹할 가능성이 있는지'를 `DefaultScheduler`가 식별할 수 있도록 넘겨주는 태그이다.

```kotlin
// The unlimited instance of Dispatchers.IO that utilizes all the threads CoroutineScheduler provides
private object UnlimitedIoScheduler : CoroutineDispatcher() {

    @InternalCoroutinesApi
    override fun dispatchYield(context: CoroutineContext, block: Runnable) {
        DefaultScheduler.dispatchWithContext(block, BlockingContext, true)
    }

    override fun dispatch(context: CoroutineContext, block: Runnable) {
        DefaultScheduler.dispatchWithContext(block, BlockingContext, false)
    }

    override fun limitedParallelism(parallelism: Int, name: String?): CoroutineDispatcher {
        parallelism.checkParallelism()
        if (parallelism >= MAX_POOL_SIZE) {
            return namedOrThis(name)
        }
        return super.limitedParallelism(parallelism, name)
    }

    // This name only leaks to user code as part of .limitedParallelism machinery
    override fun toString(): String {
        return "Dispatchers.IO"
    }
}
```

결과적으로 `Default` 디스패처가 유일한 `CoroutineScheduler`를 생성하고, `IO` 디스패처는 모든 작업을 동일한 `DefaultScheduler`로 보낸다.
따라서 두 디스패처는 **동일한 스케줄러**를 공유한다.

또한, 이 스케줄러가 스레드를 관리하기 위해 단 하나의 `workers` 배열을 사용한다는 점에서 "단 하나의 스레드풀을 공유한다"는 의미로 해석할 수 있다.
이 공유 스레드풀은 애플리케이션 시작 시점에 스레드가 0개인 상태에서 시작한다.

## 3. limitedParallelism 및 스레드풀 상세 동작
위 분석을 바탕으로 `Default`와 `IO`의 동작 방식 및 `limitedParallelism`에 대한 질문들을 정리해보았다.

1. 뷰(View)란 무엇인가?  
   '스레드풀을 바라보는 하나의 통로' 정도로 이해하면 된다. 뷰는 스레드풀을 직접 소유하지 않는 가벼운 래퍼(Wrapper) 객체로,
   작업 요청을 받으면 원본 스케줄러에 위임하되 `limitedParallelism(10)`처럼 자신을 통과하는 작업의 동시 실행 개수만 제한하는 게이트키퍼 역할을 한다.

2. `IO.limitedParallelism`은 책에서 표현하는 그림을 보면 어디에도 속하지 않는 것으로 보인다. 그렇다면 공유 스레드 풀은 `Default`, `IO`, 그리고 나머지는
`IO.limitedParallelism`에 속하는 것인가?  
   `IO`의 탄력성이란, `Default`의 한도나 `IO`의 한도와는 별개의 동시 실행을 추가로 허용한다는 의미이다. 스레드풀은 하나이며, Default (8개), IO (64개), IO.limitedParallelism(20) (20개)는 모두 그 하나의 풀을 사용하는 별개의 뷰이다.
   `CoroutineScheduler`는 이 모든 요청을 받아 (8 + 64 + 20 = 92개) 스레드를 생성한다.

3. 책에서는 공유 스레드풀에서 `무한대`로 스레드를 생성할 수 있다고 했는데, 정말 literally 무한대로 찍을 수 있는건가?
   관용적인 표현이다. 스레드 개수가 `newFixedThreadPool(8)`처럼 고정된 것이 아니라, 필요에 따라 계속 늘어날 수 있음을 강조하는 의미이다.
   - 소프트웨어적 한계: `CoroutineScheduler`의 `maxPoolSize`는 약 209만 개로 설정되어 있다.
   - 현실적 한계: `maxPoolSize`에 도달하기 전에, 스레드당 필요한 스택 메모리로 인해 하드웨어/OS의 메모리 한계에 먼저 도달한다.  
     `IO.limitedParallelism`은 Default/IO의 기본 한도를 넘어 이 소프트웨어적 한계(209만 개)까지 스레드 생성을 요청할 수 있다는 의미이다.

4. 유휴 스레드는 어떻게 되는가?  
   유휴 스레드의 처리 스레드는 작업이 없다고 무한정 대기하지 않는다. `CoroutineScheduler`는 `idleWorkerKeepAliveNs`라는 타임아웃(기본값 60초)을 가진다.  
   스레드가 작업을 마친 후 60초 동안 새 작업이 할당되지 않아 유휴 상태로 남아있으면, 해당 Worker 스레드는 스스로를 종료(terminate)하고 스레드풀에서 제거된다.
   ```Kotlin
   @JvmField
   internal val IDLE_WORKER_KEEP_ALIVE_NS = TimeUnit.SECONDS.toNanos(
       systemProp("kotlinx.coroutines.scheduler.keep.alive.sec", 60L)
   )
   ```