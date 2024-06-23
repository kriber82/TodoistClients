# Kotlin Client

## Code & Jar generation
In your command line/shell:
```sh
cd kotlin
./gradlew build
```

## Usage

Currently the client is not available on mavenCentral (you're welcome to create a PR to add this). Thus, you have to include it in your project manually, for now.

### Step 1: Build or download the jar
You have two options for acquiring the jar:
- Download a CI build of the jar ([Github Actions](https://github.com/kriber82/TodoistClients/actions) -> Click recent build -> Scroll to Artifacts section -> Click Download Icon)
- Build your own version of the jar (see previous section)

### Step 2: Place the jar in your project
I suggest placing it in the following folder: [project-root]/libs

### Step 3: Add dependencies

In your build.gradle:
```gradle
dependencies {
    implementation files("libs/todoist-rest-v2-client-kotlin-0.1.0.jar")
    implementation 'com.squareup.moshi:moshi-kotlin:1.15.1'
    implementation 'com.squareup.moshi:moshi-adapters:1.15.1'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
}
```

### Step 3: Use in your production code

In your kotlin code:
```kotlin
fun printAllTasks() {
    val todoistApiToken = System.getenv("TODOIST_API_TOKEN").orEmpty()
    if (todoistApiToken.isEmpty())
        throw Exception("Provide your Todoist API token in the environment variable TODOIST_API_TOKEN")
    ApiClient.accessToken = todoistApiToken

    val tasksApi = TasksApi()
    val allActiveTasks = tasksApi.getActiveTasks()

    println(allActiveTasks)
}
```

### Step 4: Use in your test code

The best way, I have found so far to test against the generated client is using okhttp mockserver with a custom dispatcher.

In your build.gradle:
```gradle
dependencies {
    testImplementation 'com.squareup.okhttp3:mockwebserver:4.12.0'
}
```

In your test files, you can initialize the fake server like so:
```kotlin
class TestContext() {

    lateinit var mockServer: TodoistApiServerFake
    lateinit var tasksApi: TasksApi

    @Before
    fun setupTodoistApiWithRestMock() {
        mockServer = TodoistApiMockWebServer()
        mockServer.start()

        tasksApi = TasksApi(basePath = mockServer.baseUrl())
    }

    @After
    fun stopWebserver() {
        mockServer.stop()
    }

}
```

Here is a highly incomplete TodoistApiServerFake.kt. The excerpt below is deliberately incomplete, as I currently only add the methods I need for my production code. If my implementation becomes more complete at some point, I might share it in this repo (of course PRs doing so are highly welcome). I suggest locating this file somewhere under src/test/kotlin/.
```kotlin
import okhttp3.mockwebserver.Dispatcher
import okhttp3.mockwebserver.MockResponse
import okhttp3.mockwebserver.MockWebServer
import okhttp3.mockwebserver.RecordedRequest
import java.nio.charset.Charset

class TodoistApiServerFake {
    private val mockWebServer = MockWebServer()
    private val dispatcher: Dispatcher

    private val taskJsons = mutableListOf<String>()

    init {
        dispatcher = object : Dispatcher() {
            override fun dispatch(request: RecordedRequest): MockResponse {
                return when (request.path) {
                    "/rest/v2/tasks" -> handleTaskRequest(request)
                    ...
                    else -> handleUnknownRequest(request)
                }
            }
        }
        mockWebServer.dispatcher = dispatcher
    }

    fun start() {
        mockWebServer.start()
    }

    fun baseUrl(): String {
        return mockWebServer.url("/rest/v2/").toString()
    }

    fun stop() {
        mockWebServer.shutdown()
    }

    private fun handleTaskRequest(request: RecordedRequest): MockResponse {
        when (request.method) {
            "POST" -> {
                val body = request.body.readString(Charset.defaultCharset())
                taskJsons.add(body)
                return MockResponse().setBody(body)
            }
            "GET" -> {
                return MockResponse().setBody(taskJsons.joinToString(prefix = "[", postfix = "]"))
            }
            ...
        }
        return handleUnknownRequest(request)
    }

    ...

    private fun handleUnknownRequest(request: RecordedRequest): MockResponse {
        println("Unexpected request: $request")
        return MockResponse().setResponseCode(404);
    }

}
```
