## Глава 38. Строгое FP с Arrow.kt

### 1. `Either<L, R>`: Своя обработка ошибок

`Either` (Или/Или) — это фундаментальный тип в Arrow. Это коробка, в которой может лежать строго одно из двух значений:

* **`Left` (Лево):** Традиционно здесь лежит **Ошибка**. Причем это может быть *любой* твой класс, а не только `Throwable`.
* **`Right` (Право):** Здесь лежит **Успешный результат** (Right — от слова "Правильно").

Это называется **Железнодорожно-ориентированным программированием (Railway Oriented Programming)**.

* **Аналогия из жизни:** Поезд едет по путям. Он начинает движение на "Успешной" (Правой) ветке. Он проезжает станции: "Парсинг", "Проверка базы", "Списание денег". Если на любой станции происходит сбой, стрелка переводит поезд на "Ошибочную" (Левую) ветку. Поезд больше не заезжает на успешные станции, он экспрессом летит до самого конца, везя с собой информацию об ошибке.

В современных версиях Arrow работа с `Either` выглядит невероятно чисто благодаря `computation blocks` (вычислительным блокам), которые используют магию корутин (suspend) под капотом.

```kotlin
import arrow.core.*
import arrow.core.raise.either

// 1. Описываем НАШИ бизнес-ошибки (без всяких Exception!)
sealed class DomainError {
    object UserNotFound : DomainError()
    data class NotEnoughMoney(val deficit: Double) : DomainError()
}

data class User(val balance: Double)

// 2. Функции возвращают Either
fun fetchUser(id: Int): Either<DomainError, User> = 
    if (id == 1) User(100.0).right() else DomainError.UserNotFound.left()

fun charge(user: User, price: Double): Either<DomainError, String> =
    if (user.balance >= price) "Оплата прошла".right() 
    else DomainError.NotEnoughMoney(price - user.balance).left()

// 3. Вычислительный блок either { ... }
fun processPayment(userId: Int, price: Double): Either<DomainError, String> = either {
    // Метод bind() (или просто вызов функции внутри блока) "распаковывает" Right.
    // Если fetchUser вернет Left, ВЕСЬ блок мгновенно прервется 
    // и вернет эту ошибку наружу. Это заменяет кучу if-else!
    
    val user: User = fetchUser(userId).bind() 
    
    // Если мы дошли сюда, значит user точно найден. Пытаемся списать деньги.
    val receipt: String = charge(user, price).bind()
    
    receipt // Возвращаем финальный результат
}

fun main() {
    println(processPayment(1, 50.0))  // Выведет: Right(Оплата прошла)
    println(processPayment(1, 150.0)) // Выведет: Left(NotEnoughMoney(deficit=50.0))
    println(processPayment(2, 50.0))  // Выведет: Left(UserNotFound)
}

```

### 2. `Validated`: Сборщик ошибок

`Either` работает по принципу **Short-circuiting (Короткое замыкание)**. Как только функция `bind()` спотыкается о первую ошибку, она немедленно выходит.

Но что если мы валидируем форму регистрации? Пользователь ввел плохой email и слишком короткий пароль. Если мы используем `Either`, мы покажем ему ошибку почты. Он ее исправит, нажмет "Отправить", и *только тогда* узнает, что пароль тоже плохой. Это бесит. Мы хотим собрать **все** ошибки сразу.

Для этого в Arrow есть тип `Validated` (или функция `zipOrAccumulate`).

* **Аналогия из жизни:** Проверка контрольной работы. `Either` — это строгий учитель, который зачеркивает весь лист при первой же опечатке в первом абзаце и возвращает работу. `Validated` — это добрый учитель, который дочитывает сочинение до конца, подчеркивает красным карандашом **все пять ошибок** в разных местах и отдает список исправлений за один раз.

```kotlin
import arrow.core.raise.zipOrAccumulate

data class RegistrationForm(val email: String, val pass: String)

// Простые правила (возвращают списки строк с ошибками)
fun validateEmail(email: String) {
    if (!email.contains("@")) raise("В email нет символа @")
}

fun validatePassword(pass: String) {
    if (pass.length < 8) raise("Пароль короче 8 символов")
}

fun registerUser(form: RegistrationForm): Either<NonEmptyList<String>, String> = either {
    // zipOrAccumulate запускает обе проверки независимо.
    // Даже если почта упадет, он проверит пароль и сложит ошибки в список!
    zipOrAccumulate(
        { validateEmail(form.email) },
        { validatePassword(form.pass) }
    )
    "Пользователь ${form.email} успешно зарегистрирован!"
}

fun main() {
    val badForm = RegistrationForm("bad-email.com", "123")
    println(registerUser(badForm)) 
    // Выведет: Left(NonEmptyList(В email нет символа @, Пароль короче 8 символов))
}

```

### 3. `Option` vs `Nullable` (?)

В других языках (Java, Scala) класс `Option` (или `Optional`) жизненно необходим, чтобы избежать знаменитой `NullPointerException`. Он представляет собой коробку, в которой может быть значение (`Some`), а может быть пустота (`None`).

Но мы пишем на Kotlin. У нас есть великолепная система Nullable-типов на уровне компилятора (`String?`).

Поэтому разработчики Arrow приняли гениальное решение: они **не заставляют** тебя оборачивать переменные в объекты `Option`. Вместо этого Arrow предоставляет мощные DSL-блоки (например, `nullable { ... }`), которые позволяют работать с обычными типами со знаком вопроса так же элегантно, как с `Either`.

*(Справка: Класс `Option` в Arrow все еще существует, но он применяется в узких специфических случаях, например, когда нужно сохранить "ничто" как реальный объект внутри дженерик-коллекции `List<Option<String>>`, но в 95% случаев в Kotlin используется родной `?`)*.

### 4. Оптика (Optics / Lenses): Хирургия данных

Один из главных принципов FP — использование **Неизменяемых объектов (Immutable Data Classes)**. Это защищает от гонок данных (Data Races) и багов состояния.

Но у неизменяемости есть минус. Если у тебя есть глубоко вложенный объект, его очень больно обновлять (копировать).

```kotlin
// Три уровня вложенности
data class City(val name: String)
data class Address(val street: String, val city: City)
data class Employee(val name: String, val address: Address)

val emp = Employee("Джон", Address("Ленина", City("Москва")))

// Стандартный Kotlin - боль копирования Матрешки!
val updatedEmp = emp.copy(
    address = emp.address.copy(
        city = emp.address.city.copy(name = "Питер")
    )
)

```

**Оптика (Lenses - Линзы)** решает эту проблему. Линза — это специальный сгенерированный объект (фокус), который знает, как проложить маршрут от самого верхнего класса до нужного свойства на любой глубине вложенности, и безопасно создать копию всего дерева.

* **Аналогия из жизни:** Лазерный скальпель (или Оптический прицел). Вместо того чтобы полностью разбирать матрешку руками на каждом уровне, чтобы перекрасить самую маленькую фигурку внутри, ты берешь Линзу. Она мгновенно фокусируется на нужной точке сквозь все слои и выдает тебе совершенно новую, перекрашенную матрешку.

При использовании плагина Arrow Optics компилятор сгенерирует для тебя эти "линзы" автоматически (в виде companion properties).

```kotlin
// Подключаем плагин и аннотацию
@optics data class City(val name: String) { companion object }
@optics data class Address(val street: String, val city: City) { companion object }
@optics data class Employee(val name: String, val address: Address) { companion object }

fun main() {
    val emp = Employee("Джон", Address("Ленина", City("Москва")))

    // Магия Оптики! Мы прокладываем путь до нужного свойства через точки 
    // и вызываем метод modify() (или set()).
    val updatedEmp = Employee.address.city.name.modify(emp) { it.replace("Москва", "Питер") }
    
    // Код выглядит так, как будто мы меняем var-переменную, 
    // но на самом деле мы создаем полностью новую цепочку immutable-объектов!
}

```

---

Итог:

* Библиотека **Arrow.kt** привносит строгие функциональные абстракции в Kotlin, позволяя строить надежные, тестируемые архитектуры без использования "взрывных" исключений (`Exceptions`).
* **`Either<Left, Right>`** — ключевой тип для Railway Oriented Programming. Позволяет описать ожидаемые ошибки предметной области через типизированный канал (Left), изолируя их от канала успешного выполнения (Right). Вычислительные блоки `either { ... }` заменяют вложенные проверки и `flatMap` цепочки на плоский, читаемый код.
* **`Validated` (и методы `zipOrAccumulate`)** используются, когда необходимо не прервать выполнение при первой ошибке, а собрать все ошибки воедино (например, при валидации форм).
* **Оптика (`Optics`, `Lenses`)** решает проблему изменения (глубокого копирования) слоистых неизменяемых `data`-классов. Она предоставляет сгенерированные "пути" до вложенных свойств, позволяя модифицировать их лаконично, не прибегая к каскаду методов `.copy()`.</Left,></Option</L,>
