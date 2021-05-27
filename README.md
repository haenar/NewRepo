![Release notification for slack](https://github.com/severgroup-tt/xqa_oknetwork/workflows/Release%20notification%20for%20slack/badge.svg)
![Pull request notification for slack](https://github.com/severgroup-tt/xqa_oknetwork/workflows/Pull%20request%20notification%20for%20slack/badge.svg)
[![ktlint](https://img.shields.io/badge/code%20style-%E2%9D%A4-FF4081.svg)](https://ktlint.github.io/)


TalentTech: XQA-OkNetwork
=====================

##### Легковесный http клиент с удобными методами для тестирования. Простой тест выглядит следующим образом:
```kotlin
restClient().get(url = "http://example.com/users")
    .shouldBe(
        Condition.codeEquals(200),
        Condition.bodyParamEquals("count", 100)
    )
```

Оглавление
----------
- [Начало работы](#начало-работы)
- [Создание http-клиента](#создание-http-клиента)
- [Формирование тела запроса](#формирование-тела-запроса)
- [Отправка запроса и получение ответа](#отправка-запроса-и-получение-ответа)
- [Получение параметров из ответа](#получение-параметров-из-ответа)
    - [Получение отдельных параметров из ответа](#получение-отдельных-параметров-из-ответа)
    - [Методы получения тела ответа](#методы-получения-тела-ответа)
- [Проверка параметров ответа](#проверка-параметров-ответа)
- [Конфигурирование клиента](#конфигурирование-клиента)
    - [Базовый URL](#базовый-URL)
    - [Примеры использования конфигурации](#примеры-использования-конфигурации)
- [Пример использование библиотеки](#пример-использование-библиотеки)


Начало работы
-------------
Библиотека доступна в локальном репозитории [Artifactory](http://130.193.49.249:8082/ui/packages).

#### Установка зависимости

#### Gradle

В файл `gradle.properties` добавить строки (предварительно запросив логин и пароль от artifactory у команды xqa):
```properties
#artifactory properties
artifactory_user=<login>
artifactory_password=<pass>
artifactory_url=http://130.193.49.249:8082/artifactory/talenttech
```

В файл `dependencies.gradle` добавить:
```groovy
repositories {
    maven {
        url "${artifactory_url}"
        credentials {
            username "${artifactory_user}"
            password "${artifactory_password}"
        }
    }
}

dependencies {
    implementation(group: 'ru.talenttech.xqa', name: 'oknetwork', version: '0.1.1')
}
```

#### Maven
1. В папке с Maven, `${user.home}/.m2` или `${maven.home}/conf`, добавить прикрепленный файл [settings.xml](https://burning-heart.atlassian.net/wiki/download/attachments/1573814335/settings.xml?api=v2) (внутри все уже отредактировано).   
Или отредактировать уже имеющийся у вас файл вручную:
- в `<servers>...</servers>` (предварительно запросив логин и пароль от artifactory)
```xml
<server>
    <id>artifactory</id>
    <username>login</username>
    <password>pass</password>
</server>
```
- в `<profiles>...</profiles>`:
```xml
<profile>
    <repositories>
        <repository>
            <id>artifactory</id>
            <url>http://130.193.49.249:8082/artifactory/talenttech</url>
        </repository>
    </repositories>
</profile>
```

2. Далее, в проекте, в файл `pom.xml` добавить в `<dependencies>...</dependencies>`:
```xml
<dependency>
    <groupId>ru.talenttech.xqa</groupId>
    <artifactId>oknetwork</artifactId>
    <version>0.1.1</version>
</dependency>
```

##### Если Maven запускается внутри Docker
Дополнительно к предыдущему пункту:
1. В корень проекта положить файл settings.xml
2. В Dockerfile добавить строчки
```dockerfile
RUN mkdir -p /root/.m2 \
    && mkdir /root/.m2/repository
    
COPY settings.xml /root/.m2
```


Создание http-клиента
---------------------
Клиент можно вызывать каждый раз новый или записать его в переменную, например, чтобы сконфигурировать и переиспользовать.  
Их есть три вида:
- REST-клиент *(содержит методы GET, POST, PUT, DELETE и PATCH, по умолчанию установлен content-type - JSON)*
```kotlin
restClient()
// или
val client = restClient()
```
- SOAP-клиент *(содержит метод POST и по умолчанию установлен content-type - XML)*
```kotlin
soapClient()
// или
val client = soapClient()
```
- RPC-клиент *(содержит метод POST и по умолчанию установлен content-type - PROTOBUF)*
```kotlin
rpcClient()
// или
val client = rpcClient()
```

Формирование тела запроса
-------------------------
Параметры запроса:
- url - адрес запроса
- urlParams - данные, которые будут подставлены в url, вместо параметров указанных в фигурных скобках
- queryParams - параметры запроса, которые передаются в url 
- headers - различные заголовки (хедеры)
- contentType - указание в каком формате будет передаваться тело запроса
- body - тело запроса

*Чаще всего, удобнее передавать параметры запроса напрямую в клиент, "на лету", подробнее см. [Отправка запроса и получение ответа](#отправка-запроса-и-получение-ответа)*

```kotlin
val request = Request(
            url = "https://api.exapmle.com/company/{companyId}/user/{userId}",
            urlParams = mutableListOf("21", "342"),
            queryParams = mutableMapOf("per_page" to "30", "offset" to "0"),
            headers = mutableMapOf(
                "Accept-Language" to "ru",
                "Accept-Charset" to "utf-8"
            ),
            contentType = ContentType.JSON,
            body = """{"name": "John", "age": 25}""",
        )
```

Так же легко можно добавить заголовок:
```kotlin
request.addHeader("Accept-Language", "ru")
```

Если заголовков много, можно использовать коллекцию:
```kotlin
request.addHeaders(
        "Accept-Language" to "ru",
        "Accept-Charset" to "utf-8"
    )
```

Или удалить заголовок:
```kotlin
request.removeHeader("Accept-Language")
```

Существуют ситуации, когда часть url адреса должна быть динамическая и необходимо подставлять различные данные в определенные эндпоинты.
Для это нужно в url указать их в фигурных скобках, например: https://api.exapmle.com/company/{companyId}/user/{userId},
и передавать параметры пути в том же порядке, в котором они указаны в url.  
Это можно сделать следующим образом:
 ```kotlin
val request = Request(
    url = "https://api.exapmle.com/company/{companyId}/user/{userId}",
    urlParams = mutableListOf("5, 34")
    )
// или
request.addUrlParams("5", "34")
 ```

Параметр body принимает несколько вариантов тела запроса:
- String c JSON или XML
- Объект класса (POJO), для сериализации в JSON
- Объект класса, сгенерированного их proto-файлов, для передачи в формате protobuf
- `MultipartBody()` - для передачи файлов в формате multipart/form-data

Отправка запроса и получение ответа
-----------------------------------
Для отправки запроса поддерживаются методы: GET, POST, PUT, PATCH и DELETE.   
Методы возвращают ответ с типом Response.
```kotlin
restClient().get(request)
restClient().post(request)
restClient().put(request)
restClient().patch(request)
restClient().delete(request)
```

Так же, есть возможность формировать запрос "на лету", при этом параметры в методах аналогичны параметрам для [тела запроса](#формирование-тела-запроса):
```kotlin
restClient().post(
            url = "https://api.exapmle.com/company/{companyId}/user/{userId}",
            urlParams = mutableListOf("5, 34"),
            headers = mutableMapOf(
                "Accept-Language" to "ru",
                "Accept-Charset" to "utf-8"
            ),
            contentType = ContentType.JSON,
            body = """{"name": "John", "age": 25}""",
        )
```
Чтобы получить ответ как переменную, достаточно просто определить куда будет внесен результат запроса
```kotlin
val response = restClient().get(request)
```
Получение параметров из ответа
------------------------------
Ответ (класс Response) содержит параметры:
- url - адрес запроса
- protocol - протокол передачи данных
- code - статус-код ответа (200, 404, 500)
- headers - заголовки ответа
- body - тело ответа

#### Получение отдельных параметров из ответа
```kotlin
val url = response.url
val protocol = response.protocol
val code = response.code
val headers = response.headers
val body = response.body
```

#### Методы получения тела ответа
*Пример ответа:*
```json
{
    "name": "John",
    "surname": "Lennon",
    "age": 80,
    "contacts": {
      "email": "example@mail.com",
      "phone": "05551234567"
    }
}
```
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<root>
  <name>John</name>
  <surname>Lennon</surname>
  <age>80</age>
  <contacts>
    <email>example@mail.com</email>
    <phone>05551234567</phone>
  </contacts>
</root>
```

**Для JSON**
- десериализация всего тела ответа в объект класса
```kotlin
val user = response.body(User::class.java)

val userName = user.name
val userAge = user.age
val userEmail = user.contacts.email
```
- десериализация вложенного json в объект класса, с использованием jsonPath
```kotlin
val userContacts = response.body("user.contacts", Contacts::class.java)
val userEmail = contacts.email
val userPhone = contacts.phone
```

**Универсальный метод, для JSON, XML или HTML**

Получение данных из конкретного параметры в jsonPath или xmlPath.  
Метод для получения параметров из JSON, XML или HTML. Если в тело ответа не будет содержать контент с одним их этих типов, то по умолчанию он будет воспринят как JSON.
Тип переменной, в которую должен быть записать параметр из тела ответа, должен быть определен заранее, во время ее объявления.
```kotlin
val userName = response.body("user.name")
val userAge = response.body("user.age")
val userEmail = response.body("user.contacts.email")
```

**Для PROTOBUF**

Десериализация protobuf выполняется с использованием его собственного метода `parseFrom()`, 
который принимает тело ответа в формате `ByteArray`:
```kotlin
val user = Authentication.GetUserResponse.parseFrom(response.body as ByteArray)
``` 
Проверка параметров ответа
--------------------------------
Для проверки параметров ответа можно использовать метод `shouldBe()` и передать в него условия для проверки:
- Condition.codeEquals(200) - проверка статус кода
- Condition.codeNotEquals() - проверка, что статус код не равен указанному
- Condition.headerAvailable("content-encoding") - проверка, что в ответе есть конкретный заголовок
- Condition.headerValueEquals("content-type", "application/json") - проверка значения в определенном заголовке
- Condition.bodyParamEquals("id", 123) - проверка определенного параметра в теле, через json-path
- Condition.bodyEqualsByPath("company.settings", Settings) - сравнение вложенного json с объектов
- Condition.bodyEquals(Company) - сравнение всего тела ответа с объектом
```kotlin
client().get(url = "https://api.exapmle.com/company")
        .shouldBe(
                Condition.codeEquals(200),
                Condition.headerValueEquals("content-type", "application/json"),
                Condition.bodyParamEquals("id", 123),
                Condition.bodyParamEquals("name", "Компания"),
                Condition.bodyParamEquals("active", true)
        )
```
Так же, можно сначала выполнить проверки, а потом необходимый параметр ответа, например тело или заголовок, записать в переменную.
К примеру, перед получением тела ответа, необходимо убедиться, что нам пришел верный статус код и тип контента: 
```kotlin
val body = client().get(url = "https://api.exapmle.com/company")
        .shouldBe(
                Condition.codeEquals(200),
                Condition.headerValueEquals("content-type", "application/json")
        )
        .body(Company::class.java)
```

Конфигурирование клиента
------------------------
Конфигурированием можно воспользоваться, если необходимо добавлять какие-то данные ко всем запросам.
    
Есть два способа конфигурирования:
1. На уровне библиотеки - данные будут использоваться для каждого клиента
2. На уровне клиента - данные будут использоваться только в клиенте, для которого они указаны.  
 
*Если данные указаны на обоих уровнях, то уровень клиента будет приоритетнее. Т.е. при отправке запроса, клиент сначала 
смотрит, имеются ли для него уникальные данные. Если есть, то использует их, а если нет, то использует данные из глобального конфига.* 

В конфиге можно указать следующие параметры:
- Базовый URL (baseUrl) - если первая часть url повторяется в каждом запросе, например схема и домен: https://example.com
- Постоянные заголовки (permanentHeaders) - если необходимо добавить заголовок ко всем запросам
- Аутентификация (authenticator) - если сервер закрыт basic-аутентификацией
- Вывод логов (isLogging) - по умолчанию выключен. Включать, если необходимо выводить в консоль данные из запросов и ответов, 
- Разрешить редирект при получении статус кода 301 (followRedirects) - по умолчанию выключен. Включать, если в ручке включен редирект на другой хост или эндпоинты
- Разрешить редирект с http на https (followProtocolRedirects) - по умолчанию выключен. Включать, если в апи включен редирект с http на https
- Тайм-аут на ожидание ответа (timeOut) - по умолчанию равен 10 секунд. Включать, если сервер отвечает дольше 10 секунд. Значение задается в секундах.
- Добавление запросов и проверок из метода `shouldBe()` в Allure отчет (addLogsInAllureReport) - по умолчанию выключен. Будет работать только, если генерация Allure отчета настроена в проекте использующем OkNetwwork

#### Базовый URL
Как это работает:
- если в базовом url есть схема и домен, а в запросе только путь, то они объединяются (https://example.com + /company/users)
- если базовый url НЕ указан, то используется только url из запроса
- если базовый url указан, а url из запроса НЕТ, то используется только базовый
- если url из запроса указан полностью, со схемой и доменом: https://new.example.com/company/user, то базовый url не используется 
*(на случай, если нужно выполнить запрос по другому адресу, но из этого же клиента)*

#### Примеры использования конфигурации
На уровне библиотеки
```kotlin
OkNetwork.baseUrl = "https://example.com"
OkNetwork.permanentHeaders = mutableMapOf("Accept-Language" to "ru")
OkNetwork.authentication = BasicAuthentication("userName", "pass")
OkNetwork.isLogging = true
OkNetwokr.followRedirects = true
OkNetwork.followProtocolRedirects = true
OkNetwork.timeOut = 20
OkNetwork.addLogsInAllureReport = true
```
На уровне клиента
```kotlin
val client = restClient(
    baseUrl = "https://example.com",
    permanentHeaders = mutableMapOf("Accept-Language" to "ru"),
    authentication = BasicAuthentication("userName", "pass"),
    isLogging = true,
    followRedirects = true,
    followProtocolRedirects = true,
    timeOut = 20,
  addLogsInAllureReport = true
)  
```

Так же, конфиг можно создать отдельно и передать его в клиент, как аргумент:
```kotlin
val config = OkNetworkConfig.Builder()
                .baseUrl("https://example.com")
                .permanentHeaders(mutableMapOf("Accept-Language" to "ru"))
                .authentication(BasicAuthentication("userName", "pass"))
                .isLogging(true)
                .followRedirects(true)
                .followProtocolRedirects(true)
                .timeOut(20)
                .addLogsInAllureReport(true)
                .build()

val client = restClient(config)
```

Пример использование библиотеки
-------------------------------
```kotlin
@Before
fun setUp() {
    // Конфигурируем клиент на уровне библиотеки
    OkNetwork.baseUrl = "https://example.com"
    OkNetwork.permanentHeaders = mutableMapOf("Authorization" to "sf4GFd.31skE.fG48m")
    OkNetwork.isLogging = true
}

@Test
fun exampleTest() {
    // Отправляем запрос для проверки данных юзера, проверяем статус-код и параметр в json,
    // и десериализуем весь json в объект класса User()
    val user = restClient().get("/user")
        .shouldBe(
            codeEquals(200),
            bodyParamEquals("name", "John")
        )
        .body(User::class.java)


    // Создаем отдельный клиент со своим конфигом для обращений в админку
    val adminClient = restClient(
        baseUrl = "https://admin.com",
        permanentHeaders = mutableMapOf("token" to "123adaf32552fgGgsd425FS")
    )

    // Меняем имя в объекте юзера
    user.name = "Ben"

    // Обращаемся в админку для редактирования юзера
    adminClient.put(
        url = "users/{userID}",
        urlParams = mutableListOf(user.id), // В url подставляем id юзера, полученное выше
        body = user // Объект будет сериализован в json
    )

    // Отправляем повторный запрос к юзеру и проверяем, что имя изменилось
    restClient().get("/user")
        .shouldBe(
            bodyParamEquals("name", "Ben")
        )

    // Затем удаляем юзера в админке
    adminClient.delete(
        url = "users/{userID}",
        urlParams = mutableListOf(user.id),
    )
    
    // И проверяем что он удален
    restClient().get("/user")
        .shouldBe(
            codeEquals(404)
        )
}
```
