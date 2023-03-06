## About Coroutines

This is just a humble compendium about coroutines.

* Introduction  

Kotlin coroutines are a concurrency design pattern introduced in Kotlin 1.3 that enables developers to write asynchronous, non-blocking code in a more readable and efficient way. 

* Concurrency vs. Parallelism when talk about Coroutines

Concurrency is the ability of a program to perform multiple tasks at the same time, whereas parallelism is the ability to execute multiple tasks simultaneously on multiple processors. 

* <span style="color: #00FF00">Coroutines provide concurrency, not parallelism!</span>

Concurrency and parallelism are related but distinct concepts in computer science.

Concurrency refers to a program's ability to handle multiple tasks or processes at the same time, without necessarily executing them simultaneously. This can be achieved through techniques like coroutines, where the program can switch between tasks as needed, giving the illusion of simultaneous execution.

```kotlin
fun doLongTask() {
    GlobalScope.launch {
        // This coroutine runs asynchronously, allowing other code to execute at the same time
        // even though this function does a long operation.
        // The current thread is not blocked.
        longOperation()
    }
}
```

Parallelism, on the other hand, refers to executing multiple tasks simultaneously on multiple processors or cores. This typically requires more hardware resources and a different approach to programming than concurrency.
While coroutines provide concurrency, they do not inherently provide parallelism since they do not execute tasks simultaneously on multiple processors. However, coroutines can be combined with parallel programming techniques to achieve both concurrency and parallelism.

```kotlin
fun doLongTaskInParallel() {
    val numCores = Runtime.getRuntime().availableProcessors()
    val executor = Executors.newFixedThreadPool(numCores)
    for (i in 1..numCores) {
        executor.execute {
            longOperation()
        }
    }
    executor.shutdown()
}
```

> Kotlin coroutines can be combined with parallel programming techniques to achieve both concurrency and parallelism. 

Here are some ways to achieve this:

* Using Dispatchers: Kotlin coroutines come with a set of dispatchers that can be used to specify the execution context for coroutines. 

In the example below, there is a demonstration of concurrency:

```kotlin
// create a coroutine scope with the default dispatcher
val coroutineScope = CoroutineScope(Dispatchers.Default)

// launch multiple coroutines to run concurrently on multiple threads
coroutineScope.launch {
    val result1 = async { computeResult1() }
    val result2 = async { computeResult2() }
    val combinedResult = result1.await() + result2.await()
    updateUI(combinedResult)
}
```
The async function is used to launch two coroutines concurrently, which can execute on different threads. When the await function is called on each deferred value, the coroutine suspends its execution until the corresponding computation is complete, but allows the other coroutine to continue executing concurrently. This allows the program to make progress on multiple tasks at the same time, achieving concurrency.

* Attention: However, note that this example uses the Dispatchers.Default dispatcher, which is designed for CPU-bound work and creates a thread pool with a fixed number of threads. This means that the coroutines will execute concurrently, but not necessarily in parallel on multiple processors.

We can make an adaptation of the same example using Dispatchers.Default.asExecutor() to achieve a explicit parallelism:
```kotlin
// create a coroutine scope with a custom dispatcher that provides parallelism
val threadPool = Executors.newFixedThreadPool(2).asCoroutineDispatcher()
val coroutineScope = CoroutineScope(threadPool)

// launch multiple coroutines to run in parallel on multiple threads
coroutineScope.launch {
    val result1 = async { computeResult1() }
    val result2 = async { computeResult2() }
    val combinedResult = result1.await() + result2.await()
    updateUI(combinedResult)
}
```
In this example, we create a custom dispatcher using Executors.newFixedThreadPool(2).asCoroutineDispatcher(). This creates a thread pool with two threads that can execute coroutines in parallel on multiple processors. We then create a coroutine scope using this dispatcher and launch two coroutines using the async function to compute result1 and result2. The await function is used to wait for the completion of each computation, and the results are combined and passed to updateUI.
With this implementation, the coroutines can execute concurrently and in parallel, achieving both concurrency and parallelism.

* The same example using Dispatchers.IO:
```kotlin
// create a coroutine scope with the IO dispatcher that provides parallelism
val coroutineScope = CoroutineScope(Dispatchers.IO)

// launch multiple coroutines to run in parallel on multiple threads
coroutineScope.launch {
    val result1 = async { computeResult1() }
    val result2 = async { computeResult2() }
    val combinedResult = result1.await() + result2.await()
    withContext(Dispatchers.Main) {
        updateUI(combinedResult)
    }
}
```
Using Dispatchers.IO does not guarantee parallelism, but it can provide parallelism under certain conditions. The Dispatchers.IO dispatcher uses a thread pool that can grow or shrink dynamically based on demand, which means that it can allocate multiple threads to execute multiple coroutines in parallel. However, whether or not the coroutines actually execute in parallel depends on a number of factors, such as the available CPU cores, the workload of other applications running on the system, and the nature of the tasks being executed.

In general, Dispatchers.IO is optimized for I/O-bound tasks, such as network requests or disk operations, that involve waiting for external resources and are typically not CPU-intensive. In such cases, the coroutines can suspend their execution while waiting for the I/O operations to complete, allowing other coroutines to execute in parallel on different threads.

However, for CPU-bound tasks that do not involve waiting for external resources, using Dispatchers.Default or a custom thread pool with a fixed number of threads may be more appropriate to achieve parallelism, as this can allocate a fixed number of threads that can execute the coroutines in parallel on multiple CPU cores.

* Using parallel collections: Kotlin provides parallel versions of its collections library, such as asFlow().parallel(), which can be used to process large amounts of data in parallel using multiple coroutines. This can provide both concurrency and parallelism by executing tasks concurrently on multiple threads and processors:
```kotlin
// create a list of data to process in parallel using coroutines
val dataList = listOf("data1", "data2", "data3", "data4", "data5")

// create a flow from the data list and parallelize its processing
val resultFlow = dataList.asFlow()
    .onEach { Log.d(TAG, "processing $it on thread ${Thread.currentThread().name}") }
    .parallel()
    .map { processData(it) }
    .sequential()

// collect the result flow and update the UI with the combined result
coroutineScope.launch {
    val combinedResult = resultFlow.reduce { acc, result -> acc + result }
    updateUI(combinedResult)
}
```
* Using actors: Actors are a concurrency design pattern that can be used with coroutines to achieve parallelism. Actors can handle concurrent requests and update shared state in a thread-safe manner, allowing multiple coroutines to execute simultaneously without interfering with each other:
```kotlin
// create an actor to handle concurrent requests and update shared state
class SharedStateActor : CoroutineScope by CoroutineScope(Dispatchers.Default) {
    private var sharedState = 0
    
    // define a message type to update the shared state
    sealed class Message {
        data class Update(val value: Int) : Message()
    }
    
    // define the actor behavior to handle incoming messages
    private val actor = actor<Message> {
        for (message in channel) {
            when (message) {
                is Message.Update -> sharedState += message.value
            }
        }
    }
    
    // define a function to send messages to the actor
    fun updateSharedState(value: Int) {
        actor.offer(Message.Update(value))
    }
    
    // define a function to retrieve the current shared state
    fun getSharedState(): Int = sharedState
    
    // clean up the actor when the scope is cancelled
    fun cleanUp() {
        actor.cancel()
    }
}

// create an instance of the shared state actor
val sharedStateActor = SharedStateActor()

// launch multiple coroutines to update the shared state concurrently
coroutineScope.launch {
    for (i in 1..10) {
        launch {
            sharedStateActor.updateSharedState(i)
        }
    }
    
    // wait for all coroutines to complete and update the UI with the shared state
    coroutineScope.launch {
        delay(1000) // wait for all coroutines to complete
        val sharedState = sharedStateActor.getSharedState()
        updateUI(sharedState)
    }
}

// clean up the shared state actor when the scope is cancelled
coroutineScope.launch {
    coroutineScope.coroutineContext.cancelChildren()
    sharedStateActor.cleanUp()
}
```
* So, you could ask: its usual use such parallelism approach in an android mobile application?

In general, parallelism can be beneficial when performing CPU-bound tasks that can be split into multiple independent tasks that can run in parallel. For example, if your application needs to download and process multiple large files concurrently, using parallelism can speed up the processing time.

However, parallelism also has its downsides, such as increased resource usage and potential synchronization issues. In a mobile device, if not managed in a good way, It could be little dangerous, since many android devices have limited memory resources…

* More stuff about coroutines

Kotlin coroutines are a powerful feature that enables developers to write asynchronous code. Ok, you already know that!

Here are some key ideas about such stack:

* Suspend Functions: a suspend function is a function that can be paused and resumed later without blocking the current thread. These functions are the building blocks of coroutines and can be identified by the suspend modifier in their signature.
```kotlin
suspend fun fetchData(): String {
    delay(1000) // This is a built-in suspend function that pauses the coroutine for a specified time
    return "Data from the network"
}
```
* Continuation Passing Style: coroutines use continuation passing style (CPS) to achieve their asynchronous behavior. CPS is a programming technique that passes the control flow of a program to a callback function when an operation is not yet complete.
```kotlin
fun doLongOperation(callback: (result: Int) -> Unit) {
    // This function does a long operation and then passes the result to the callback function
    GlobalScope.launch {
        delay(1000)
        callback(42)
    }
}
```
* Coroutine Context: a set of rules that define how a coroutine should behave. It specifies which thread or thread pool a coroutine should run on, as well as other properties such as exception handling and cancellation behavior. 
```kotlin
val coroutineContext = Dispatchers.IO + CoroutineName("fetchData")

fun fetchData(): String = runBlocking(coroutineContext) {
    // This coroutine runs on the IO thread pool and has a name of "fetchData"
    // This makes it easier to debug and reason about the code.
    delay(1000)
    "Data from the network"
}
```
* Launching a Coroutine: we can use launch function from the kotlinx.coroutines package. This function creates a new coroutine and runs it in the background.
```kotlin
fun doLongOperation() {
    GlobalScope.launch {
        // This coroutine runs in the background and doesn't block the UI thread.
        delay(1000)
        withContext(Dispatchers.Main) {
            // This code runs on the UI thread after the coroutine is finished.
            updateUI()
        }
    }
}
```
* Suspending a Coroutine: when a coroutine encounters a suspend function, it can pause its execution and return control to the calling code. The calling code can then continue to execute until the suspend function is ready to resume.
```kotlin
suspend fun fetchData(): String {
    // This coroutine pauses until the network request is complete.
    val result = withContext(Dispatchers.IO) {
        networkRequest()
    }
    return result
}
```
* Coroutine Scope: scope is an object that manages the lifecycle of coroutines. It ensures that all launched coroutines are cancelled when the scope is cancelled.  
```kotlin
class MyActivity : AppCompatActivity(), CoroutineScope {
    private lateinit var job: Job

    override val coroutineContext: CoroutineContext
        get() = Dispatchers.Main + job

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        job = Job()
        setContentView(R.layout.activity_main)
    }

    override fun onDestroy() {
        super.onDestroy()
        job.cancel()
    }

    fun fetchData() {
        launch {
            // This coroutine is managed by the activity's coroutine scope,
            // so it will be cancelled when the activity is destroyed.
            delay(1000)
            updateUI()
        }
    }
}
```
* Cancellation: coroutines can be cancelled using the cancel function or by throwing a CancellationException. When a coroutine is cancelled, all of its children coroutines are also cancelled.
```kotlin
fun doLongOperation() {
    val job = GlobalScope.launch {
        delay(5000)
        updateUI()
    }
    // This code cancels the coroutine after 1000 milliseconds.
    GlobalScope.launch {
        delay(1000)
        job.cancel()
    }
}
```
* Error Handling: in coroutines is done using try/catch blocks or by propagating exceptions up the call stack. You can also use the CoroutineExceptionHandler to handle uncaught exceptions in coroutines.
```kotlin  
 fun fetchData() {
    GlobalScope.launch(CoroutineExceptionHandler { _, exception ->
        // This code runs when an unhandled exception occurs in the coroutine.
        showErrorToast(exception)
    }) {
        try {
            val result = networkRequest()
            updateUI
        } catch (e: IOException) {
            // This code runs when a network error occurs.
            showNetworkErrorToast(e)
        } catch (e: Exception) {
            // This code runs when any other exception occurs.
            showErrorToast(e)
        }
    }
}
```
* Dispatchers: what it is?

Earlier I talked about using dispatchers to get concurrency. But what is Dispatchers?

Kotlin coroutines Dispatchers are a mechanism for controlling where and how coroutines run. A coroutine dispatcher is responsible for scheduling coroutines to run on a particular thread or thread pool.

You can specify a dispatcher to control where the coroutine runs. Dispatchers provide a way to abstract away the details of threading and concurrency, so you can focus on writing asynchronous code without worrying about the underlying implementation.

So... what are the types of Dispatchers?

* Main - runs the coroutine on the main thread of the Android UI thread.
* IO - optimized for disk or network IO operations, runs on a shared thread pool that is designed to handle IO tasks.
* Default - optimized for CPU-intensive tasks, runs on a shared thread pool that is designed to handle CPU-bound tasks.
* Unconfined - runs the coroutine on the current thread until the first suspension point, after which it resumes on a different thread.

We can also create your own custom dispatcher by implementing the CoroutineDispatcher interface, but to be preetty honest, this is unusual.

------------------------------------------------------------------------------------------------------------------------------------------------

## Some examples of Coroutines Dispatchers used in open source projects

* [ChatGPT](https://github.com/skydoves/chatgpt-android):

```kotlin
-> @Module
@InstallIn(SingletonComponent::class)
internal object DispatchersModule {

  @Provides
  @Dispatcher(ChatGPTDispatchers.IO)
  fun providesIODispatcher(): CoroutineDispatcher = Dispatchers.IO
} 

-> internal class GPTMessageRepositoryImpl @Inject constructor(
  @Dispatcher(ChatGPTDispatchers.IO) private val ioDispatcher: CoroutineDispatcher,
  private val chatGptService: ChatGPTService
) : GPTMessageRepository {
 
 
-> init {
    viewModelScope.launch {
      gptChannelRepository.streamUserFlow().collect { user ->
        user?.let { gptChannelRepository.joinTheCommonChannel(it) }
      }
    }
  }   
```
* [nowinandroid](https://github.com/android/nowinandroid):

```kotlin  
class FakeNewsRepository @Inject constructor(
    @Dispatcher(IO) private val ioDispatcher: CoroutineDispatcher,
    private val datasource: FakeNiaNetworkDataSource,
) : NewsRepository {
 
override suspend fun doWork(): Result = withContext(ioDispatcher) {
        traceAsync("Sync", 0) {
            analyticsHelper.logSyncStarted()

            // First sync the repositories in parallel
            val syncedSuccessfully = awaitAll(
                async { topicRepository.sync() },
                async { newsRepository.sync() },
            ).all { it }

            analyticsHelper.logSyncFinished(syncedSuccessfully)

            if (syncedSuccessfully) {
                Result.success()
            } else {
                Result.retry()
            }
        }
} 

@OptIn(ExperimentalSerializationApi::class)
    override suspend fun getNewsResources(ids: List<String>?): List<NetworkNewsResource> =
        withContext(ioDispatcher) {
            assets.open(NEWS_ASSET).use(networkJson::decodeFromStream)
}

@OptIn(ExperimentalSerializationApi::class)
    override suspend fun getTopics(ids: List<String>?): List<NetworkTopic> =
        withContext(ioDispatcher) {
            assets.open(TOPICS_ASSET).use(networkJson::decodeFromStream)
}
```
* [firefox](https://github.com/mozilla-mobile/firefox-android):

```kotlin 
-> val sessionManager = sessionManager(engine).apply {
    // We launch a coroutine on the main thread. Once a snapshot has been restored
    // we want to continue with it on the main thread.
    GlobalScope.launch(Dispatchers.Main) {
        // We restore on the IO dispatcher to not block the main thread:
        val snapshot = async(Dispatchers.IO) {
            val bundle = sessionStorage.restore()
            // If we got a bundle then restore the snapshot from it
            bundle.restoreSnapshot(engine)
        }

        // If we got a snapshot then restore it now:
        snapshot.await()?.let { sessionManager.restore(it) }
    }
} 

-> open class FenixApplication : LocaleAwareApplication(), Provider {

  // We avoid blocking the main thread on startup by calling into Glean on the background thread.
  @OptIn(DelicateCoroutinesApi::class)
  GlobalScope.launch(Dispatchers.IO) {
    PerfStartup.applicationOnCreate.accumulateSamples(listOf(durationMillis))
  } 

   // We avoid blocking the main thread on startup by setting startup metrics on the background thread.
   val store = components.core.store
   GlobalScope.launch(Dispatchers.IO) {
   setStartupMetrics(store, settings())
   }
 
    @OptIn(DelicateCoroutinesApi::class) // GlobalScope usage
    private fun restoreBrowserState() = GlobalScope.launch(Dispatchers.Main) {
        val store = components.core.store
        val sessionStorage = components.core.sessionStorage

        components.useCases.tabsUseCases.restore(sessionStorage, settings().getTabTimeout())

        // Now that we have restored our previous state (if there's one) let's setup auto saving the state while
        // the app is used.
        sessionStorage.autoSave(store)
            .periodicallyInForeground(interval = 30, unit = TimeUnit.SECONDS)
            .whenGoingToBackground()
            .whenSessionsChange()
    } 

-> AddonDetailsFragment

    override fun showUpdaterDialog(addon: Addon) {
        viewLifecycleOwner.lifecycleScope.launch(Main) {
            val updateAttempt = withContext(IO) {
                updateAttemptStorage.findUpdateAttemptBy(addon.id)
            }
            updateAttempt?.showInformationDialog(requireContext())
        }
    }
```
 <br>
 
## Doubts that may arise

* Use GlobalScope.launch its a good approach?

Generally not a good approach. The reason is that GlobalScope creates a global scope for coroutines that is not tied to any particular component or lifecycle. This means that coroutines launched using GlobalScope continue running even if the calling component or lifecycle is destroyed, which can lead to memory leaks and other problems.
 
* When to use withContext?
* When to use lifecycleScope.launch?
* viewModelScope.launch -> this can helps to refactor something?
* We can use Dispatchers.IO inside Dispatchers.Main and vice versa?
* Where to use Dispatchers.Main?
* Where to use Dispatchers.IO?
