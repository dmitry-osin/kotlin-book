## Глава 45. Ktor Framework

### 1. Архитектура: Корутины вместо Потоков

Классические Java-серверы (Tomcat, Jetty в стандартном Spring) работают по модели **Thread-per-request (Один запрос = Один поток)**. Если на сервер одновременно приходит 200 пользователей, сервер создает 200 физических потоков ОС. Если база данных тормозит, все 200 потоков блокируются, и 201-й пользователь получает ошибку "Сервер недоступен".

Ktor построен на **Корутинах**. Когда приходит запрос, Ktor выделяет под него не тяжелый поток, а легкую корутину.
Если микросервису нужно сходить в базу данных или сделать сетевой запрос, корутина делает `suspend` (засыпает), мгновенно освобождая поток процессора для других пользователей.

Благодаря этому один единственный физический поток в Ktor может обслуживать **десятки тысяч одновременных соединений** без `OutOfMemoryError`. Для этого в Ktor даже есть собственный движок — **CIO (Coroutine-based I/O)**.

### 2. DSL вместо Аннотаций

В Spring мы привыкли обвешивать классы аннотациями: `@RestController`, `@GetMapping("/users")`, `@PathVariable`. Проблема аннотаций в том, что ты не видишь потока выполнения — фреймворк сам ищет их с помощью медленной рефлексии.

В Ktor нет магических аннотаций. Роутинг (маршрутизация) строится с помощью **чистого Kotlin DSL** (Domain Specific Language) — вложенных функций, которые выглядят как древовидная структура. Это работает невероятно быстро и проверяется компилятором на этапе сборки.

```kotlin
// Это весь код, необходимый для запуска сервера!
fun main() {
    embeddedServer(CIO, port = 8080) {
        // DSL для роутинга
        routing {
            get("/") {
                // Вызов respondText - это suspend функция!
                call.respondText("Добро пожаловать в Ktor!")
            }
            
            get("/users/{id}") {
                // Достаем параметры прямо из контекста call
                val id = call.parameters["id"]
                call.respondText("Вы запросили пользователя $id")
            }
        }
    }.start(wait = true)
}

```

### 3. Плагины (Модульность)

Как мы уже сказали, Ktor "голый" по умолчанию. Он даже не знает, что такое JSON, и не умеет писать логи в консоль.
Чтобы научить его этому, мы используем механизм **Плагинов** (ранее назывались Features). Ты явно вызываешь функцию `install()` и настраиваешь плагин. Это гарантирует, что в твоем приложении нет ни строчки лишнего кода, который ты бы не контролировал.

Давай напишем полноценный, чистый микросервис, который будет отдавать JSON (используя библиотеку `kotlinx.serialization` из модуля 7.2).

```kotlin
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.cio.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.server.plugins.callloging.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.Serializable
import kotlinx.coroutines.delay

// Наша модель данных
@Serializable
data class User(val id: Int, val name: String, val role: String)

// Главная функция (точка входа)
fun main() {
    embeddedServer(CIO, port = 8080, module = Application::myMicroservice).start(wait = true)
}

// Модуль приложения (Extension функция на класс Application)
fun Application.myMicroservice() {
    
    // 1. Устанавливаем плагин для логгирования запросов
    install(CallLogging)
    
    // 2. Устанавливаем плагин ContentNegotiation для работы с JSON
    // Он автоматически связывает Ktor и kotlinx.serialization
    install(ContentNegotiation) {
        json() 
    }

    // 3. Настраиваем маршруты
    routing {
        get("/api/user") {
            // Имитируем долгий запрос в базу данных.
            // Это suspend-вызов! Поток не заблокируется, он пойдет обслуживать других.
            delay(500) 
            
            val user = User(id = 1, name = "Дмитрий", role = "Admin")
            
            // Ktor сам поймет, что это data class, и плагин ContentNegotiation 
            // превратит его в JSON строку {"id":1,"name":"Дмитрий","role":"Admin"}
            call.respond(user)
        }
    }
}

```

### 4. Ktor Client: Брат-близнец

У Ktor есть еще одна суперсила. Это не только серверный фреймворк, но и **HTTP-клиент**.
Если ты пишешь приложение на Android или делаешь запросы между двумя микросервисами, тебе не нужно использовать громоздкий Retrofit.

Ktor Client использует ту же самую философию корутин и плагинов, и он **мультиплатформенный**. Один и тот же код клиента будет работать на JVM, в браузере (Kotlin/JS) и на iPhone (Kotlin/Native).

```kotlin
// Пример использования Ktor Client
suspend fun fetchUser() {
    val client = HttpClient(CIO) {
        install(ContentNegotiation) { json() }
    }

    // Делаем асинхронный GET-запрос и сразу парсим JSON в объект User!
    val user: User = client.get("http://localhost:8080/api/user").body()
    println(user.name)
    
    client.close()
}

```

---

Итог:

* **Ktor** — это легковесный, асинхронный веб-фреймворк от JetBrains, спроектированный специально для экосистемы Kotlin.
* **100% Корутины:** Вместо классической блокирующей модели потоков (Thread-per-request), Ktor использует `suspend` функции. Движок CIO (Coroutine I/O) позволяет обрабатывать огромное количество соединений на минимальном количестве физических потоков ОС, что идеально для высоконагруженных микросервисов.
* **Без рефлексии и аннотаций:** Конфигурация и маршрутизация (Routing) строятся на базе типобезопасного Kotlin DSL. Это ускоряет старт приложения (отсутствует сканирование классов) и делает код максимально прозрачным.
* **Архитектура плагинов (Opt-in):** Фреймворк предельно минималистичен. Любой функционал (сериализация JSON, аутентификация JWT, CORS, логгирование) добавляется в проект явно через вызов `install(PluginName)`.
* **Мультиплатформенный клиент:** Ktor предоставляет асинхронный HTTP-клиент с аналогичной архитектурой, который компилируется под любые платформы (iOS, Android, JS, Desktop) в рамках технологии Kotlin Multiplatform (KMP).