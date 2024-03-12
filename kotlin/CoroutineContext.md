


```kotlin
public interface CoroutineContext {
  /**
   * Returns the element with the given [key] from this context or `null`.
   * Keys are compared _by reference_, that is to get an element from the context the reference to its actual key
   * object must be presented to this function.
   */
  public operator fun <E : Element> get(key: Key<E>): E?
  /**
   * Accumulates entries of this context starting with [initial] value and applying [operation]
   * from left to right to current accumulator value and each element of this context.
   */
  public fun <R> fold(initial: R, operation: (R, Element) -> R): R
  /**
   * Returns a context containing elements from this context and elements from  other [context].
   * The elements from this context with the same key as in the other one are dropped.
   */
  public operator fun plus(context: CoroutineContext): CoroutineContext = ...impl...
  /**
   * Returns a context containing elements from this context, but without an element with
   * the specified [key]. Keys are compared _by reference_, that is to remove an element from the context
   * the reference to its actual key object must be presented to this function.
   */
  public fun minusKey(key: Key<*>): CoroutineContext
}
```

코루틴 컨텍스트(CoroutineContext)에는 코루틴 컨텍스트를 상속한 요소(Element) 들이 등록될 수 있고, 각 요소들이 등록 될때는 요소의 고유한 키를 기반으로 등록된다.
코루틴 컨텍스트의 구현체는 3가지 종류가 있다.

* EmptyCoroutineContext: 특별히 컨텍스트가 명시되지 않을 경우 이 singleton 객체 사용
* CombinedContext: 두 개 이상의 컨텍스트가 명시되면 컨텍스트 간 연결을 위한 컨테이너 역할을 하는 컨텍스트
* Element: 컨텍스트의 각 요소들도 코루틴 컨텍스트 구현

## CoroutineScope
CoroutineContext 하나만 멤버 속성으로 정의하고 있는 인터페이스

```kotlin
public interface CoroutineScope {
    /**
     * Context of this scope.
     */
    public val coroutineContext: CoroutineContext
}
```

우리가 사용하는 모든 코루틴 빌더들(ex. launch, async, coroutineScope, withContext)은 CoroutineScope의 확장 함수로 정의된다. 즉, 이 빌더들은 CoroutineScope의 함수들이고, 이들이 코루틴을 생성할 때는 소속된 CoroutineScope에 정의된 CoroutineContext를 기반으로 필요한 코루틴들을 생성해낸다.
CoroutineScope는 인터페이스일 뿐이며 구현을 위해서는 스코프를 정의해야한다. 편의를 위해서 코루틴 프레임워크에 미리 정의된 스코프가 존재하며, 그 중 하나가 `GlobalScope` 이다.


```kotlin
// -- in CoroutineScope.kt
object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}

// -- in CoroutineContextImpl.kt
@SinceKotlin("1.3")
public object EmptyCoroutineContext : CoroutineContext, Serializable {
    private const val serialVersionUID: Long = 0
    private fun readResolve(): Any = EmptyCoroutineContext

    public override fun <E : Element> get(key: Key<E>): E? = null
    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R = initial
    public override fun plus(context: CoroutineContext): CoroutineContext = context
    public override fun minusKey(key: Key<*>): CoroutineContext = this
    public override fun hashCode(): Int = 0
    public override fun toString(): String = "EmptyCoroutineContext"
}
```

GlobalScope는 Singleton object로써 EmptyCoroutineContext를 컨텍스트로 가지고 있다. EmptyCoroutineContext는 구현해야할 모든 CoroutineContext 멤버 함수들에 대해서 기본 구현만 정의한 컨텍스트이다. 이 기본 컨텍스트는 어떤 생명주기에 바인딩 된 Job이 정의되어 있지 않기 때문에 애플리케이션 프로세스와 동일한 생명주기를 가지게 된다.
즉, `GlobalScope.launch{}`로 실행한 코루틴은 애플리케이션이 종료되지 않는 한 필요한 만큼 실행을 계속해나간다.

```kotlin
fun main(args: Array<String>) {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)
}
```

위 예제는 메인 함수 안에서 GlobalScope.launch {} 코드블록을 이용하여 Hello, World!를 출력하는 간단한 프로그램이다. 마지막 라인에서 2초간 정지(Sleep)하는 코드가 쓰인 이유는 GlobalScope.launch {} 코드블록은 코루틴 빌더이므로 이를 통해 생성되어 실행되는 코루틴은 호출 스레드를 블록하지 않기때문에 그대로 두면 메인 함수가 종료되고 메인 함수를 실행한 메인 스레드 역시 종료되어 프로그램이 끝나게 된다.
이를 방지하기 위해 임의의 시간을 지정하여 지연시킨것이며, 이렇게 스레드를 멈추는 역할을 수행하는 함수를 중단 함수(Blocking function)라고 한다.

이러한 중단 함수가 현재 스레드를 멈추게 할 수 있다는 것을 코드상에 보다 명시적으로 나타내기 위해 다음과 같이 `runBlocking {}` 블록을 사용할 수 있다.
이는 주어진 블록이 완료될 때 까지 현재 스레드를 멈추는 새로운 코루틴을 생성하여 실행하는 코루틴 빌더이다.

```kotlin
fun main(args: Array<String>) {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    runBlocking {
        delay(2000L)
    }
}

// 메인함수 자체를 코루틴 빌더를 이용하여 작성
fun main(args: Array<String>) = runBlocking {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    delay(2000L)
}
```

`delay()`는 중단 함수이며 모든 중단 함수들은 코루틴 안에서만 호출될 수 있다. 지금까지의 예제에서는 GlobalScope.launch{} 로 실행된 코루틴의 수행이 완료될 때 까지 현재 스레드(main 함수)를 대기시키기 위해서 임의의 지연(2초)을 주었는데 이는 실제 프로그램에서 적절한 방법이 아니다.
이유는 내부적으로 실행죽인 코루틴(자식 코루틴)들이 작업을 완료하고 종료될 때 까지 얼마나 대기해야할지 부모 코루틴은 예측할 수 없기 때문이다.
이러한 문제를 해결하기 위해서 우리는 다음과 같이 `GlobalScope.launch {}`의 결과로 반환되는 Job 인스턴스를 이용할 수 있다.

```kotlin
fun main(args: Array<String>) = runBlocking {
    val job = GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    job.join()
}
```

메인 코루틴 안에서 두개 이상의 자식 코루틴들이 수행되고, 모든 자식 코루틴들의 종료를 기다리도록 구현해야한다면 자식 코루틴에 대응되는 모든 Job 객체들의 참조를 어딘가에 유지하고 있다가 부모 코루틴이 종료되어야 하는 시점에 실행된 모든 자식 코루틴들의 job들에 join하여 자식 코루틴들의 종료를 기다려야 할 것이다.

이러한 번거로운 일을 줄이기 위해 코루틴 스코프(Scope)를 사용할 수 있다. 모든 코루틴들은 각자의 스코프를 가진다. 그렇기 때문에 `runBlocking {}` 코루틴 빌더 등을 이용해 생성된 코루틴 블록 안에서 `launch {}` 코루틴 빌더를 이용하여 새로운 코루틴을 생성하면 현재 위치한 부모 코루틴에 `join()`을 명시적으로 호출할 필요 없이 자식 코루틴들을 실행하고 종료될 때 까지 대기할 수 있다.

---
## Scope Builder
사용자 정의 스코프가 필요하다면 `coroutineScope {}` 빌더를 이용할 수 있다.이 빌더를 통해 생성된 코루틴은 모든 자식 코루틴들이 끝날때까지 종료되지 않는 스코프를 정의하는 코루틴이다.

### runBlocking{} 과 coroutineScope{}의 차이점
runBlokcing{}과 달리 coroutineScope{}는 자식들의 종료를 기다리는 동안 현재 스레드를 블록하지 않는다.
* `runBlocking{}`: 호출한 스레드를 블로킹하면서 코루틴을 실행한다. 이 구문 내에서 실행되는 코루틴은 모두 완료될 때까지 현재 스레드를 블로킹한다.
* `coroutineScope{}`: 새 코루틴 스코프를 만들고, 이 스코프 안에서 실행되는 모든 코루틴이 완료될 때까지 기다린다. 현재 코루틴의 실행을 블로킹하지 않고, 내부의 코루틴이 완료될 때 까지만 대기한다.

```kotlin
fun main(args: Array<String>) = runBlocking {
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    coroutineScope {
        launch {
            delay(500L)
            println("Task from nested launch")
        }
        delay(100L)
        println("Task from coroutine scope")
    }
    println("Coroutine scope is over")
}
```

이 코드의 실행 순서는 다음과 같다.
1. `runBlocking` 블록 내의 `launch` 코루틴이 시작되어 200ms 동안 대기
2. 그 다음으로 `coroutineScope` 블록이 실행되며 이 안에서 또 다른 `launch` 코루틴이 시작되어 500ms 동안 대기
3. `coroutineScope` 내의 첫 번째 대기 시간(100ms)이 끝나면, 출력
4. 이후, `runBlocking` 블록 내의 첫 번째 `launch` 코루틴의 200ms 대기 시간이 끝나면 내용 출력
5. `coroutineScope` 블록 내의 모든 코루틴 작업이 완료되면 마지막 문장 출력

결과적으로 다음과 같이 출력되는 것을 확인할 수 있다.
```kotlin
Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
```
---
* 아래 블로그 링크를 읽고 정리한 문서입니다.
* https://myungpyo.medium.com/reading-coroutine-official-guide-thoroughly-part-1-7ebb70a51910
* https://myungpyo.medium.com/reading-coroutine-official-guide-thoroughly-part-1-98f6e792bd5b

