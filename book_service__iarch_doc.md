# Документация по разработке цифрового библиотечного сервиса

## Описание

Необходимо реализовать систему цифрового библиотечного сервиса, с несколькими возможностями для пользователей
<details> <summary>Основные бизнес-требования к системе</summary>
В библиотеке хранятся книги, у книг есть основные параметры (Название, дата издания, автор), эти книги выдаются пользователям библиотеки, имеющим абонемент с параметрами(username, ФИО, статус абонемента(действует, просрочен, заблокирован), срок действия абонемента), книги выдают по предварительной броне сотрудники библиотеки.

Для пользователей библиотеки, сервис должен давать возможность:
•  Просмотреть доступные им книги
•  Заказать бронь книги перед посещением библиотеки
•  Уведомить пользователя о наступлении срока сдачи книги

Для сотрудников библиотеки, сервис должен давать возможность:
•  Посмотреть статус абонемент пользователя, забронированные им книги, полученные им книги, срок сдачи книг.
•  Уведомление сотрудников о большой просрочке по даче книг, со стороны пользователя
•  Заблокировать абонемент пользователя при большой просрочке, находящееся у него книг
•  Предупреждение, при выдаче книги пользователю с подходящем к концу сроком действия абонемента
•  Возможность загрузки legacy абонемента пользователя, хранящегося в excel файле, где у одного пользователя на карточке может числиться множество книг
</details>

## Описание моделей данных

Для реализации системы необходимо реализовать следующую модель данных

<details> <summary>Концептуальная модель данных</summary>

![alt text](/assets/book_service__conceptual_level.png)
</details>

<details> <summary>Логическая модель данных</summary>

![alt text](/assets/book_service__logical_level.png)
</details>

## Описание компонентов системы

Для реализации системы её необходимо декомпозировать на компоненты (модули, микросервисы) в соответствии со следующей схемой

<details><summary>Схема компонентов системы </summary>

![book_service__system_design_luk6dinda](/assets/book_service__system_design.png)
</details>

## Требования к системе

### Функциональные требоваиня

#### ФТ-1. Создание метода просмотра доступных книг пользователю

 Поле       | Значение                                                 |
|------------|----------------------------------------------------------|
| Назначение | Метод должен отдавать список доступных пользователю книг |
| Метод      | GET                                                      |
| Endpoint   | /api/v1/books                                            |

![book_service__system_design_luk6dinda](/assets/sequences/book_service__get_book_sequence.png)

Тело запроса и ответа, а также заголовки и пр. описаны в Swagger-файле

<details> <summary> Код PlantUML</summary>

```uml
@startuml Get Available Books Sequence Diagram

title Просмотр доступных книг пользователем

actor "Пользователь" as User
participant "API Gateway" as Gateway
participant "UserManagementService" as UMS
participant "BookManagementService" as BMS
participant "Booking Service" as Booking
participant "Corporate IAM Service" as IAM
database "UserDatabase" as UserDB
database "BookDatabase" as BookDB
database "BookingDatabase" as BookingDB

== Аутентификация и авторизация ==
User -> Gateway: GET /api/v1/books?filter=...&page=1&size=10&order=asc
note right: Bearer Token в заголовке

Gateway -> IAM: Валидация JWT токена
activate IAM
IAM -> IAM: Проверка подписи токена\nи срока действия
IAM --> Gateway: Токен валиден + user_info
deactivate IAM

Gateway -> UMS: Получить информацию о пользователе
activate UMS
UMS -> UserDB: SELECT user, abonement WHERE user_id = ?
activate UserDB
UserDB --> UMS: User + Abonement данные
deactivate UserDB

UMS -> UMS: Проверить статус абонемента\n(ACTIVE, не просрочен)
alt Abonement заблокирован или просрочен
    UMS --> Gateway: 403 Forbidden
    Gateway --> User: 403 Forbidden - Абонемент заблокирован
else Abonement активен
    UMS --> Gateway: User данные + права доступа
    deactivate UMS
    
    == Получение списка книг ==
    Gateway -> BMS: Получить список доступных книг\n(filter, pagination, order)
    activate BMS
    
    BMS -> BookDB: SELECT books с фильтрацией и пагинацией
    activate BookDB
    note right: WHERE available_copies > 0\nOR overbooking_enabled = true
    BookDB --> BMS: Список книг с базовой информацией
    deactivate BookDB
    
    == Персонализация для пользователя ==
    BMS -> Booking: Проверить существующие резервации пользователя
    activate Booking
    Booking -> BookingDB: SELECT reservations WHERE user_id = ?\nAND status IN (PENDING, CONFIRMED, READY_FOR_PICKUP)
    activate BookingDB
    BookingDB --> Booking: Активные резервации пользователя
    deactivate BookingDB
    Booking --> BMS: Список зарезервированных книг пользователем
    deactivate Booking
    
    BMS -> BMS: Исключить уже зарезервированные\nпользователем книги из списка
    
    BMS --> Gateway: Список доступных книг с информацией:\n- Book details (title, authors, publisher)\n- Availability status\n- Available copies count\n- Overbooking info
    deactivate BMS
    
    Gateway --> User: 200 OK\nJSON массив с книгами
end

== Обработка ошибок ==
note over Gateway, BMS: В случае ошибок базы данных или сервисов:\n- 500 Internal Server Error\n- Логирование ошибки\n- Стандартизированный ErrorResponse

@enduml
```

</details>

<details> <summary>Логика работы </summary>

1. Аутентификация и авторизация

    - Пользователь отправляет запрос с JWT токеном
    - API Gateway валидирует токен через Corporate IAM Service
    - UserManagementService проверяет статус абонемента пользователя
    - ЕСЛИ абонемент пользователя заблокирован, вернуть ошибку BOOK_ACCESS_ERROR

  ```json
  // HTTP 403 Forbidden
  {
    "errorCode": "BOOK_ACCESS_ERROR",
    "errorMessage": "Невозможно получить книги, так как ваш абонемент заблокирован. Пожалуйста, свяжитесь с библиотекой для получения дополнительной информации."
  }
  ```

1. Получение базового списка книг

    - BookManagementService выполняет запрос к базе данных с учетом фильтров и пагинации
    - Получает книги, которые имеют доступные копии или включен овербукинг
    <details> <summary>Пример списка книг </summary>

    ```json
      [
        {
          "bookId": "string",
          "title": "string",
          "isbn": "string",
          "publicationDate": "2025-06-11",
          "authors": [
            {
              "authorId": "string",
              "name": "string",
              "biography": "string"
            }
          ],
          "publisher": {
            "publisherId": "string",
            "name": "string",
            "address": "string"
          },
          "totalCopies": 0,
          "availableCopies": 0,
          "reservedCopies": 0,
          "overbookingEnabled": true,
          "availabilityStatus": "AVAILABLE"
        }
      ]
    ```

    </details>
2. Персонализация для пользователя

    - Проверяются существующие резервации пользователя
    - Исключаются или помечаются уже зарезервированные книги

3. Возврат результата

    Возвращается персонализированный список доступных книг
    <details> <summary>Пример списка книг </summary>

    ```json
      [
        {
          "bookId": "string",
          "title": "string",
          "isbn": "string",
          "publicationDate": "2025-06-11",
          "authors": [
            {
              "authorId": "string",
              "name": "string",
              "biography": "string"
            }
          ],
          "publisher": {
            "publisherId": "string",
            "name": "string",
            "address": "string"
          },
          "totalCopies": 0,
          "availableCopies": 0,
          "reservedCopies": 0,
          "overbookingEnabled": true,
          "availabilityStatus": "AVAILABLE"
        }
      ]
    ```

    </details>

</details>

#### ФТ-2. Создание метода бронирования книги пользователем

| Поле       | Значение                       |
| ---------- | ------------------------------ |
| Назначение | Метод бронирования книг        |
| Метод HTTP | POST                           |
| Endpoint   | /api/v1/books/{bookId}/reserve |

![alt text](/assets/sequences/book_service__send_reservation_plant_seq.png)

Тело запроса и ответа, а также заголовки и пр. описаны в Swagger-файле

<details>

<summary>Логика работы</summary>

##### Логика работы метода бронирования книги пользователем

| Шаг | Описание |
|-----|----------|
| 1. Пользователь инициирует бронирование | **HTTP Request:**<br>`POST /api/v1/books/book-205/reserve`<br>**Headers:** `Authorization: Bearer token...`<br>См. Request JSON #1 ниже<br>**Current Date:** `2025-06-12 16:42:04 UTC` |
| 2. API Gateway направляет запрос в BookingService | **Request URL:** `POST /api/v1/reservations`<br>**Internal Headers:** `X-User-ID: mrmacgood71, X-Request-ID: reserve-456` |
| 3. BookingService запрашивает данные пользователя | **Request URL:** `GET /api/v1/users/{userId}`<br>См. Response JSON #1 ниже |
| 4. BookingService проверяет доступность книги | **Request URL:** `GET /api/v1/books/book-205`<br>См. Response JSON #2 ниже |
| 5. BookingService проверяет существующие резервации | См. SQL Query #1 ниже |
| 6. BookingService маппит данные и создает резервацию |  |
| 7. BookingService обновляет счетчики книги | **Request URL:** `PUT /api/v1/books/book-205`<br>**Body:** `{"reservedCopies": 2}` |
| 8. BookingService возвращает созданную резервацию | См. Response JSON #3 ниже |
| 9. API Gateway возвращает результат пользователю | См. Response JSON #4 ниже |

##### Request JSON #1 - Тело запроса бронирования

```json
{
  "reservationPeriodDays": 7,
  "requestedAt": "2025-06-12T16:42:04Z"
}
```

##### Response JSON #1 - Данные пользователя и абонемента

```json
{
  "userId": "mrmacgood71",
  "username": "mrmacgood71",
  "fullName": "Иванов Иван Иванович",
  "abonement": {
    "abonementId": "abon-123",
    "status": "ACTIVE",
    "endDate": "2025-12-31T23:59:59Z",
    "maxBooks": 5,
    "currentBooks": 2
  }
  ...
}
```

#### Response JSON #2 - Доступность книги

```json
{
  "bookId": "book-205",
  "title": "Clean Architecture",
  "isbn": "978-0134494166",
  "totalCopies": 3,
  "availableCopies": 1,
  "reservedCopies": 1,
  ...
}
```

#### SQL Query #1 - Проверка существующих резерваций

```sql
SELECT * FROM reservations 
WHERE user_id = 'mrmacgood71' 
  AND book_id = 'book-205' 
  AND status IN ('PENDING', 'CONFIRMED', 'READY_FOR_PICKUP')
```


#### Response JSON #3 - Созданная резервация (ответ от BookingService)
```json
{
  "reservationId": "res-789",
  "userId": "mrmacgood71",
  "bookId": "book-205",
  "status": "PENDING",
  "createdAt": "2025-06-12T16:42:04Z",
  "expiresAt": "2025-06-19T16:42:04Z"
}
```

#### Response JSON #4 - Финальный ответ пользователю (201 Created)

```json
{
  "success": true,
  "reservation": {
    "reservationId": "res-789",
    "userId": "mrmacgood71",
    "userName": "Иванов Иван Иванович",
    "book": {
      "bookId": "book-205",
      "title": "Clean Architecture",
      "isbn": "978-0134494166",
      "authors": [{"name": "Robert C. Martin"}]
    },
    "status": "PENDING",
    "priority": "NORMAL",
    "createdAt": "2025-06-12T16:42:04Z",
    "expiresAt": "2025-06-19T16:42:04Z",
    "queuePosition": 1,
    "estimatedAvailability": "2025-06-14T10:00:00Z",
    "isOverbooking": false
  }
}
```

</details>

#### ФТ-3. Создание метода просмотра статуса абонемента пользователя

Поле       | Значение
---------- | ------------------------------
Назначение | Метод отображает информацию об абонементе, его статус и дату истечения срока действия
Метод HTTP | GET
Endpoint   | /api/v1/users/{userId}/abonements

![alt text](/assets/sequences/book_service__get_abonement_status_plant_seq.png)

<details> <summary>Логика работы </summary>

##### Логика работы метода просмотра статуса абонемента пользователя

| Шаг | Значение |
|-----|----------|
| 1. Сотрудник запрашивает статус абонемента пользователя | **HTTP Request:**<br>`GET /api/v1/users/{userId}/abonements/{abonementId}`<br>**Headers:** `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`<br>**User Role:** `STAFF` (JWT токен уже валидирован, роль проверена) |
| 2. API Gateway переадресовывает запрос к UserManagementService | **Internal Request:** Получить данные пользователя и абонемента<br>**Request Headers:** `X-Gateway-ID: gateway-001, X-Request-ID: req-12345` |
| 3. UserManagementService ищет пользователя по userId | **Database Query:**<br>```sql<br>SELECT u.*, a.* FROM users u<br>LEFT JOIN abonements a ON u.user_id = a.user_id<br>WHERE u.user_id = 'user-456' AND a.abonement_id = 'abon-789'<br>``` |
| 4. Обработка случая "Пользователь не найден" | **Alternative Flow:**<br>Если пользователь не найден в БД:<br>**Response:** `404 Not Found`<br>**Error JSON:**<br>```json<br>{<br>  "errorCode": "USER_NOT_FOUND",<br>  "errorMessage": "User not found with the provided ID"<br>}<br>``` |
| 5. UserManagementService получает данные абонемента пользователя | **Database Query:**<br>```sql<br>SELECT abonement_id, user_id, issue_date, expiry_date,

</details>

#### ФТ-4. Создание метода просмотра забронированных книг пользователем

Поле       | Значение
---------- | ------------------------------
Назначение | Метод предназначен для просмотра забронированных книг пользователем
Метод HTTP | GET
Endpoint   | /api/v1/users/{userId}/reservations

![alt text](/assets/sequences/book_service__get_reserved_books_plant_seq.png)

<details>
<summary>Логика работы</summary>

##### Логика работы метода просмотра забронированных книг пользователем

| Шаг | Описание |
|-----|----------|
| 1. Пользователь запрашивает свои резервации | **HTTP Request:**<br>`GET /api/v1/users/{userId}/reservations?status=PENDING,CONFIRMED`<br>**Headers:** `Authorization: Bearer token...`<br>См. Request JSON #1 ниже<br>**Current Date:** `2025-06-12 17:04:09 UTC` |
| 2. API Gateway направляет запрос в BookingService | **Request URL:** `GET /reservations?userId={userId}`<br>**Internal Headers:** `X-User-ID: mrmacgood71, X-Request-ID: get-reservations-234` |
| 3. BookingService маппит данные и ищет резервации | См. SQL Query #1 ниже |
| 4. BookingService запрашивает информацию о книгах | **Request URL (для каждой книги):** `GET /api/v1/books/book-205`<br>См. Response JSON #2 ниже |
| 5. BookingService обогащает данные резерваций | |
| 6. BookingService возвращает обогащенные резервации | См. Response JSON #3 ниже |
| 7. API Gateway возвращает результат пользователю | См. Response JSON #4 ниже |

#### Request JSON #1 - Параметры запроса
```json
{
  "status": "PENDING,CONFIRMED",
  "page": 1,
  "size": 10
}
```

#### SQL Query #1 - Поиск резерваций пользователя
```sql
SELECT * FROM reservations 
WHERE user_id = 'mrmacgood71' 
  AND status IN ('PENDING', 'CONFIRMED')
  AND created_at >= '2025-01-01'
ORDER BY created_at DESC
LIMIT 10 OFFSET 0
```

#### Response JSON #2 - Детали книги
```json
{
  "bookId": "book-205",
  "title": "Clean Architecture",
  "isbn": "978-0134494166",
  "authors": [{"name": "Robert C. Martin"}],
  "publisher": {"name": "Pearson"},
  "totalCopies": 3,
  "availableCopies": 1,
  "reservedCopies": 2,
  "availabilityStatus": "LIMITED"
}
```


#### Response JSON #3 - Обогащенные резервации (ответ от BookingService)
```json
[
  {
    "reservationId": "res-789",
    "book": {
      "bookId": "book-205",
      "title": "Clean Architecture",
      "isbn": "978-0134494166",
      "authors": [{"name": "Robert C. Martin"}]
    },
    "status": "PENDING",
    "reservationDate": "2025-06-10T14:30:00Z",
    "expiryDate": "2025-06-17T14:30:00Z",
    "priority": 1,
    "queuePosition": 1,
    "estimatedAvailability": "2025-06-14T10:00:00Z"
  }
]
```

#### Response JSON #4 - Финальный ответ пользователю (200 OK)
```json
{
  [
  {
    "reservationId": "string",
    "userId": "string",
    "bookId": "string",
    "book": {
      "bookId": "string",
      "title": "string",
      "isbn": "string",
      "publicationDate": "2025-06-12",
      "authors": [
        {
          "authorId": "string",
          "name": "string",
          "biography": "string"
        }
      ],
      "publisher": {
        "publisherId": "string",
        "name": "string",
        "address": "string"
      },
      "totalCopies": 0,
      "availableCopies": 0,
      "reservedCopies": 0,
      "overbookingEnabled": true,
      "availabilityStatus": "AVAILABLE"
    },
    "reservationDate": "2025-06-12T17:37:24.435Z",
    "expiryDate": "2025-06-12T17:37:24.435Z",
    "status": "PENDING",
    "priority": 0,
    "isOverbooking": true
  }
]
}
```
</details>

#### ФТ-5. Создание метода полученные книги пользователем

Поле       | Значение
---------- | ------------------------------
Назначение | Метод предназначен для просмотра полученных пользователем книг
Метод HTTP | GET
Endpoint   | /api/v1/users/{userId}/loans

![alt text](/assets/sequences/book_service__get_user_loans_plant_seq.png)

<details>
<summary><b>Логика работы</b></summary>

##### Логика работы метода просмотра полученных книг пользователем

| Шаг | Описание |
|-----|----------|
| 1. Пользователь запрашивает свои займы | **HTTP Request:**<br>`GET /api/v1/users/{userId}/loans?status=ACTIVE,RETURNED&page=1&size=10`<br>**Headers:** `Authorization: Bearer token...`<br>**Current Date:** `2025-06-11 16:31:06 UTC` |
| 2. API Gateway валидирует пользователя | **Request URL:** `GET /api/v1/users/{userId}?type=SHORT` |
| 3. UserManagementService возвращает данные пользователя | См. Response JSON #1 ниже |
| 4. API Gateway запрашивает займы у BookingService | **Request URL:** `GET /api/v1/loans?userId=mrmacgood71` |
| 5. BookingService находит займы пользователя | **Query Logic:** Поиск займов с фильтрацией по статусу и пагинацией |
| 6. BookingService запрашивает информацию о книгах | **Request URL per Book:** `GET /api/v1/books/{bookId}` |
| 7. BookManagementService возвращает информацию о книге | См. Response JSON #2 ниже |
| 8. BookingService маппит данные между сервисами | **Mapping Logic:** Объединение займов с данными книг, расчет просрочек и статусов |
| 9. BookingService проверяет наличие просроченных книг | См. JavaScript Code #1 ниже |
| 10. BookingService возвращает обогащенный список займов | См. Response JSON #3 ниже |
| 11. API Gateway формирует финальный ответ с информацией о просрочках | См. Response JSON #4 ниже |

#### Response JSON #1 - Данные пользователя

```json
{
  "userId": "mrmacgood71",
  "username": "mrmacgood71",
  "fullName": "Иванов Иван Иванович",
  ...
}
```

#### Response JSON #2 - Информация о книге

```json
{
  "bookId": "book-101",
  "title": "JavaScript: The Good Parts",
  "isbn": "978-0596517748",
  "authors": [{"name": "Douglas Crockford"}],
  "publisher": {"name": "O'Reilly Media"}
  ...
}
```

#### Response JSON #3 - Обогащенный список займов

```json
[
  {
    "loanId": "loan-301",
    "bookId": "book-101",
    "book": {
      "title": "JavaScript: The Good Parts",
      "authors": [{"name": "Douglas Crockford"}]
    },
    "issueDate": "2025-05-25",
    "dueDate": "2025-06-08",
    "returnDate": null,
    "status": "OVERDUE",
    "daysOverdue": 3,
    "fineAmount": 15.00,
    "renewalCount": 0
  }
  ...
]
```

#### Response JSON #4 - Финальный ответ

```json
{
  [
  {
    "loanId": "string",
    "userId": "string",
    "bookId": "string",
    "book": {
      "bookId": "string",
      "title": "string",
      "isbn": "string",
      "publicationDate": "2025-06-12",
      "authors": [
        {
          "authorId": "string",
          "name": "string",
          "biography": "string"
        }
      ],
      "publisher": {
        "publisherId": "string",
        "name": "string",
        "address": "string"
      },
      "totalCopies": 0,
      "availableCopies": 0,
      "reservedCopies": 0,
      "overbookingEnabled": true,
      "availabilityStatus": "AVAILABLE"
    },
    "reservationId": "string",
    "issuedByStaffId": "string",
    "issueDate": "2025-06-12",
    "dueDate": "2025-06-12",
    "returnDate": "2025-06-12",
    "status": "ACTIVE",
    "renewalCount": 0,
    "fineAmount": 0,
    "daysOverdue": 0,
    "isOverdue": true
  }
]
}
```

##### Обработка ошибочных сценариев

| Ошибка | HTTP Code | Response Example |
|--------|-----------|------------------|
| Нет займов | 204 | `No Content - пустой ответ` |
| Неверные параметры | 400 | `{"errorCode": "INVALID_PARAMETERS", "errorMessage": "Invalid status filter provided"}` |
| Пользователь не найден | 404 | `{"errorCode": "USER_NOT_FOUND", "errorMessage": "User not found"}` |
| Сервис недоступен | 500 | `{"errorCode": "SERVICE_UNAVAILABLE", "errorMessage": "Unable to retrieve loans"}` |

##### Логика расчета просроченных книг

| Поле | Описание | Расчет |
|------|----------|---------|
| `hasOverdueBooks` | Флаг наличия просроченных книг | `true` если есть займы со статусом ACTIVE и due_date < current_date |
| `overdueLoansCount` | Количество просроченных займов | Подсчет займов с просрочкой |
| `totalOverdueFines` | Общая сумма штрафов | Сумма всех fineAmount для просроченных займов |
| `mostOverdueBook` | Самая просроченная книга | Займ с максимальным значением daysOverdue |
| `canBorrowNewBooks` | Возможность брать новые книги | `false` если есть просроченные книги |

</details>

#### ФТ-6. Создание метода срок сдачи книг

Поле       | Значение
---------- | ------------------------------
Назначение | Метод предназначен для передачи информации о сроках сдачи книг пользователем
Метод HTTP | GET
Endpoint   | /api/v1/users/{userId}/checkouts

![alt text](/assets/sequences/book_service__get_checkouts_plant_seq.png)

<details>
<summary>Логика работы</summary>

##### Логика работы метода просмотра сроков сдачи книг

| Шаг | Описание |
|-----|----------|
| 1. Пользователь запрашивает сроки сдачи | **HTTP Request:**<br>`GET /api/v1/users/{userId}/checkouts`<br>**Headers:** `Authorization: Bearer token...`<br>**Current Date:** `2025-06-11 16:35:58 UTC` |
| 3. UserManagementService возвращает данные пользователя | См. Response JSON #1 ниже |
| 11. API Gateway формирует финальный ответ | См. Response JSON #3 ниже |

#### Response JSON #1 - Данные пользователя

```json
{
  "userId": "mrmacgood71",
  "username": "Иванов Иван Иванович",
  ...
}
```


#### Response JSON #3 - Финальный ответ

```json
{
  "data": [
  {
    "loanId": "string",
    "bookId": "string",
    "reservationId": "string",
    "checkoutDate": "2025-06-12T17:34:56.602Z",
    "dueDate": "2025-06-12T17:34:56.602Z",
    "status": "ACTIVE",
    "fineAmount": 0
  }
],
}
```

</details>

#### ФТ-7. Создание метода загрузки legacy абонемента пользователя

Поле       | Значение
---------- | ------------------------------
Назначение | Метод предназначен для загрузки предыдущих абонементов пользователем
Метод HTTP | POST
Endpoint   | /api/v1/imports/legacy/abonements

![alt text](/assets/sequences/book_service__send_importing_abonements_plant_seq.png)

<details> <summary>Логика работы </summary>

## Логика работы импорта legacy абонементов пользователей

| Шаг | Описание |
|-----|----------|
| 1. Администратор инициирует загрузку файла | **HTTP Request:**<br>`POST /api/v1/imports/legacy/abonements`<br>**Content-Type:** `multipart/form-data`<br>См. Request Data #1 ниже<br>**Current Date:** `2025-06-12 17:29:08 UTC`<br>**Admin User:** `mrmacgood71` |
| 2. API Gateway направляет запрос в ImportService | **Request URL:** `POST /api/v1/imports/legacy/abonements`<br>**Internal Headers:** `X-User-ID: mrmacgood71, X-Request-ID: import-123` |
| 3. ImportService сохраняет загруженный файл | **Request URL:** `POST /internal/storage/files`<br>См. Response JSON #1 ниже |
| 4. ImportService валидирует структуру файла | См. Validation Logic #1 ниже |
| 5a. Обработка невалидного файла | См. Response JSON #2 ниже (если файл невалидный) |
| 5b. Обработка dry run режима | См. Response JSON #3 ниже (если dryRun = true) |
| 6. ImportService маппит данные между сервисами | См. JavaScript Code #1 ниже |
| 7. ImportService обрабатывает каждую запись абонемента | **Цикл обработки:** См. Processing Logic #1 ниже |
| 8. ImportService проверяет существование пользователя | **Request URL (для каждого пользователя):** `GET /api/v1/users/{userId}`<br>См. Response JSON #4 ниже |
| 9a. Обработка несуществующего пользователя | См. Error Handling #1 ниже |
| 9b. Создание/обновление legacy абонемента | **Request URL:** `POST /api/v1/users/{userId}/abonements/legacy`<br>См. Request JSON #2 ниже |
| 10. UserManagementService маппит legacy данные | См. JavaScript Code #2 ниже |
| 11a. Обработка дубликата абонемента | См. Response JSON #5 ниже (если дубликат) |
| 11b. Успешное создание абонемента | См. Response JSON #6 ниже |
| 12. ImportService финализирует импорт | См. JavaScript Code #3 ниже |
| 13. ImportService возвращает результаты импорта | См. Response JSON #7 ниже |
| 14. API Gateway возвращает результат администратору | См. Response JSON #8 ниже |
| 15. Альтернативный запрос статуса импорта | **HTTP Request:**<br>`GET /api/v1/imports/legacy/abonements/{importId}/status`<br>См. Response JSON #9 ниже |

#### Request Data #1 - Данные загрузки файла
```multipart
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="legacy_abonements.csv"
Content-Type: text/csv

userId,abonementNumber,startDate,endDate,status,maxBooks
user001,AB12345,2024-01-15,2024-12-31,ACTIVE,5
user002,AB12346,2024-02-01,2024-11-30,EXPIRED,3

--boundary
Content-Disposition: form-data; name="dryRun"

false
--boundary--
```

#### Response JSON #1 - Сохранение файла
```json
{
  "fileId": "file-456",
  "fileName": "legacy_abonements.csv",
  "fileSize": 1024,
  "uploadedAt": "2025-06-12T17:29:08Z",
  "status": "SAVED"
}
```

#### Response JSON #2 - Ошибка валидации файла (400 Bad Request)
```json
{
  "success": false,
  "errorCode": "INVALID_FILE_FORMAT",
  "errorMessage": "Неверный формат файла",
  "validationErrors": [
    {
      "field": "startDate",
      "error": "Неверный формат даты в строке 3"
    },
    {
      "field": "status",
      "error": "Недопустимое значение статуса 'INVALID' в строке 5"
    }
  ]
}
```

#### Response JSON #3 - Dry run завершен (200 OK)
```json
{
    "importId": 789,
    "processedUsersCount": 0,
    "blockedUsersCount": 100
    
}
```

#### Request JSON #2 - Создание legacy абонемента
```json
{
  "legacyAbonementNumber": "AB12345",
  "issueDate": "2024-01-15T00:00:00Z",
  "expiryDate": "2024-12-31T23:59:59Z",
  "status": "ACTIVE",
  "maxBooksAllowed": 5,
  "source": "LEGACY_IMPORT",
  "importBatch": "batch-789",
  "importedBy": "mrmacgood71",
  "importedAt": "2025-06-12T17:29:08Z"
}
```


#### Response JSON #5 - Дубликат абонемента (409 Conflict)
```json
{
  "success": false,
  "errorCode": "DUPLICATE_ABONEMENT",
  "errorMessage": "Абонемент для пользователя user001 уже существует",
  "existingAbonement": {
    "abonementId": "abon-existing-123",
    "legacyNumber": "AB12345",
    "status": "ACTIVE"
  }
}
```

#### Response JSON #6 - Успешное создание абонемента (201 Created)
```json
{
  "success": true,
  "abonementId": "abon-new-456",
  "userId": "user001",
  "legacyNumber": "AB12345",
  "status": "LEGACY",
  "createdAt": "2025-06-12T17:29:08Z"
}
```


#### Response JSON #7 - Результаты импорта от ImportService
```json
{
  "importId": "import-789",
  "status": "COMPLETED",
  "summary": {
    "totalRecords": 150,
    "successful": 142,
    "failed": 6,
    "duplicates": 2,
    "warnings": 3
  },
  "processingTime": "00:02:45",
  "completedAt": "2025-06-12T17:31:53Z"
}
```

#### Response JSON #8 - Финальный ответ администратору (200 OK)
```json
{
    "importId": 789,
    "processedUsersCount": 0,
    "blockedUsersCount": 0,
    "reportUrl": "/api/v1/imports/legacy/abonements/import-789/report"
  }
}
```

#### Response JSON #9 - Статус импорта (GET запрос)
```json
{
  {
  "importId": 1,
  "fileName": "string",
  "status": "PENDING",
  "totalRecords": 0,
  "successfulRecords": 0,
  "failedRecords": 0,
  "startTime": "2025-06-12T17:39:49.477Z",
  "endTime": "2025-06-12T17:39:49.477Z",
  "errors": [
    {
      "row": 0,
      "error": "string",
      "data": {}
    }
  ]
}
}
```
</details>

#### ФТ-8. Создание метода блокировки абонемента пользователя при большой просрочке, находящееся у него книг

Поле       | Значение
---------- | ------------------------------
Назначение | Метод предназначен для блокировки абонемента пользователя при большой просрочке
Метод HTTP | PUT
Endpoint   | /api/v1/users/{userId}/abonements/{abonementsId}

##### Если использовать Scheduler

![alt text](/assets/sequences/book_service__send_abonement_status_scheduler_plant_seq.png)

##### Если использовать кнопку для библиотекаря

![alt text](/assets/sequences/book_service__put_abonements_by_staff_plant_seq.png)

<details> <summary>Логика работы </summary>

##### Логика работы метода блокировки абонемента при большой просрочке
## Логика работы блокировки абонемента пользователя при большой просрочке

| Шаг | Описание |
|-----|----------|
| 1. Работник библиотеки инициирует проверку просроченных займов | **HTTP Request:**<br>`PUT /api/v1/users/{userId}/abonements/{abonementId}`<br>**Headers:** `Authorization: Bearer token...`<br>См. Request JSON #1 ниже<br>**Note:** Автоматическая проверка каждый день в 09:00 UTC<br>**Current Date:** `2025-06-12 17:43:09 UTC` |
| 2. API Gateway запрашивает конфигурацию блокировки | **Request URL:** `GET /api/v1/configs/abonement/blockPolicy`<br>См. Response JSON #1 ниже |
| 3. API Gateway запрашивает пользователей с критической просрочкой | **Request URL:** `GET /api/v1/loans/overdues?overdueDaysThreshold={overdueDaysThreshold}&refreshCache=true`<br>См. Response JSON #2 ниже |
| 4. BookingService находит займы с критической просрочкой | См. SQL Query #1 ниже |
| 5. BookingService формирует и сохраняет список пользователей | См. JavaScript Code #1 ниже |
| 6. API Gateway направляет запрос в UserManagementService | **Request URL:** `PUT /api/v1/users/{userId}/abonements/{abonementId}`<br>**Internal Headers:** `X-Staff-ID: staff-456, X-Request-ID: block-789` Body: {"block": true}|
| 7. UserManagementService проверяет текущий статус абонемента | См. JavaScript Code #2 ниже |
| 8a. Если абонемент уже заблокирован | См. Response JSON #3 ниже |
| 8b. Если абонемент активен - выполняется блокировка | См. Database Update #1 ниже |
| 9. UserManagementService отправляет уведомление о блокировке | **Request URL:** `POST /api/v1/notify?type=OVERDUE_WARNING`<br>См. Request JSON #2 ниже |
| 10. NotificationService подтверждает отправку уведомления | См. Response JSON #4 ниже |
| 11. UserManagementService возвращает результат блокировки | См. Response JSON #5 ниже |
| 12. API Gateway возвращает результат работнику библиотеки | См. Response JSON #6 ниже |

#### Request JSON #1 - Запрос блокировки абонемента
```json
{
  "status": "BLOCKED",
  "reason": "CRITICAL_OVERDUE",
  "blockedBy": "staff-456",
  "blockedAt": "2025-06-12T17:43:09Z"
}
```

#### Response JSON #1 - Конфигурация политики блокировки
```json
{
  "overdueDaysThreshold": 30,
  "autoBlockEnabled": true,
  "notificationEnabled": true,
  "blockingSchedule": "0 9 * * *",
  "warningPeriodDays": 7
}
```

#### Response JSON #2 - Пользователи с критической просрочкой
```json
[
  {
    "userId": "mrmacgood71",
    "username": "mrmacgood71",
    "fullName": "Иванов Иван Иванович",
    "abonementId": "abon-123",
    "overdueLoans": [
      {
        "loanId": "loan-456",
        "bookId": "book-205",
        "bookTitle": "Clean Architecture",
        "daysOverdue": 35,
        "fineAmount": 175.00
      }
    ],
    "totalOverdueDays": 35,
    "totalFineAmount": 175.00
  }
]
```

#### SQL Query #1 - Поиск займов с критической просрочкой
```sql
SELECT l.user_id, l.loan_id, l.book_id, l.due_date, l.fine_amount,
       DATEDIFF(CURRENT_DATE, l.due_date) as days_overdue
FROM book_loans l
WHERE l.status = 'ACTIVE'
  AND DATEDIFF(CURRENT_DATE, l.due_date) > 30
  AND l.user_id = 'mrmacgood71'
ORDER BY days_overdue DESC
```

#### Response JSON #3 - Ответ при уже заблокированном абонементе (409 Conflict)
```json
{
  "errorCode": "ABONEMENT_ALREADY_BLOCKED",
  "errorMessage": "Абонемент уже заблокирован",
  "abonementId": "abon-123",
  "userId": "mrmacgood71",
  "blockedAt": "2025-06-10T09:00:00Z",
  "blockedBy": "staff-123"
}
```

#### Database Update #1 - Блокировка абонемента
```sql
UPDATE abonements 
SET status = 'BLOCKED',
    blocked_at = '2025-06-12 17:43:09',
    blocked_by = 'staff-456',
    block_reason = 'CRITICAL_OVERDUE',
    updated_at = CURRENT_TIMESTAMP
WHERE abonement_id = 'abon-123'
  AND user_id = 'mrmacgood71'
  AND status = 'ACTIVE'
```

#### Request JSON #2 - Уведомление о блокировке
```json
{
  "type": "OVERDUE_WARNING",
  "userId": "mrmacgood71",
  "title": "Абонемент заблокирован",
  "message": "Ваш абонемент заблокирован из-за критической просрочки книг. Обратитесь в библиотеку для решения вопроса.",
  "priority": "HIGH",
  "metadata": {
    "abonementId": "abon-123",
    "totalOverdueDays": 35,
    "totalFineAmount": 175.00,
    "blockedAt": "2025-06-12T17:43:09Z"
  }
}
```

#### Response JSON #4 - Подтверждение отправки уведомления
```json
{
  "notificationId": "notif-789",
  "status": "SENT",
  "message": "Уведомление о блокировке абонемента отправлено",
  "sentAt": "2025-06-12T17:43:09Z"
}
```

#### Response JSON #5 - Результат блокировки от UserManagementService
```json
{
  "success": true,
  "abonementId": "abon-123",
  "userId": "mrmacgood71",
  "previousStatus": "ACTIVE",
  "currentStatus": "BLOCKED",
  "blockedAt": "2025-06-12T17:43:09Z",
  "blockedBy": "staff-456",
  "blockReason": "CRITICAL_OVERDUE",
  "notificationSent": true
}
```

#### Response JSON #6 - Финальный ответ работнику библиотеки (200 OK)
```json
{
  "success": true,
  "message": "Обработка завершена",
  "processedUsers": [
    {
      "userId": "mrmacgood71",
      "username": "mrmacgood71",
      "fullName": "Максим Макгуд",
      "abonementId": "abon-123",
      "action": "BLOCKED",
      "blockedAt": "2025-06-12T17:43:09Z",
      "overdueInfo": {
        "totalOverdueDays": 35,
        "totalFineAmount": 175.00,
        "overdueLoansCount": 1
      },
      "notificationSent": true
    }
  ],
  "totalProcessed": 1,
  "processedAt": "2025-06-12T17:43:09Z"
}
```

## Альтернативные сценарии

#### Response JSON #7 - Ответ при отсутствии пользователей для блокировки (204 No Content)
```json
{
  "message": "Пользователи с критической просрочкой не найдены",
  "checkedAt": "2025-06-12T17:43:09Z",
  "overdueDaysThreshold": 30
}
```
</details>

#### ФТ-9. Создание метода уведомления сотрудников о большой просрочке по даче книг, со стороны пользователя

Поле       | Значение
---------- | ------------------------------
Назначение | Метод уведомления сотрудников о просрочке сдачи книг
Страница | /staff/notifications

При переходе работника библиотеки на свою домашнюю страницу, необхдоимо сделать запрос на /api/v1/loans/overdues для получения списка пользователей с просрочками по сдаче книг.
Необходимо использовать Lazyloading, запрос должен быть асинхронен.
При удачном запросе иконка нотификаций в header страницы должна стать проиграть анимацию и запустить компонент notification (например: Notification из Ant Design)

![alt text](/assets/sequences/book_service__get_overdue_warning_notify_plant_seq.png)

#### ФТ-10. Создание метода выдачи книги пользователю с уведомлением подходящем к концу сроком действия абонемента

Поле       | Значение
---------- | ------------------------------
Назначение | Метод выдачи книги пользователю с уведомлением подходящем к концу сроком действия абонемента
Метод HTTP | POST
Endpoint   | /api/v1/loans

![alt text](/assets/sequences/book_service__get_checkout_expiration_plant_seq.png)

<details> <summary>Логика работы</summary>


| Шаг | Описание |
|-----|----------|
| 1. Библиотекарь запрашивает выдачу книги | **HTTP Request:**<br>`POST /api/v1/loans`<br>**Headers:** `Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`<br>**Request Body:** См. Request JSON #1 ниже |
| 2. API Gateway передает запрос BookingService | **Internal Request:** `POST /api/v1/loans`<br>**Headers:** `X-Librarian-ID: lib-456, X-Request-ID: req-789`<br>**Body:** Данные запроса без изменений |
| 3. BookingService проверяет статус абонемента пользователя | **Request URL:** `GET /api/v1/users/mrmacgood71?type=SHORT`<br>**Internal Headers:** `X-Service: BookingService, X-Request-Type: abonement-check` |
| 4. UserManagementService находит и анализирует абонемент | См. SQL Query #1 ниже |
| 5. UserManagementService возвращает данные абонемента | См. Response JSON #2 ниже |
| 6. BookingService проверяет доступность книги | **Request URL:** `GET /api/v1/books/book-205` |
| 7. BookManagementService проверяет статус книги | См. SQL Query #2 ниже |
| 8. BookManagementService возвращает информацию о книге | См. Response JSON #3 ниже |
| 9. BookingService создает займ с флагом предупреждения | См. SQL Query #3 ниже |
| 10. BookingService возвращает предупреждение об истечении абонемента | См. Response JSON #4 ниже |
| 11. API Gateway передает предупреждение библиотекарю | **HTTP Response (409 Conflict):** Передача того же JSON с предупреждением |
| 12. Библиотекарь подтверждает выдачу книги несмотря на предупреждение | **HTTP Request:** `POST /api/v1/loans?type=IGNORE_ABONEMENT_EXPIRATION_WARNING`<br>**Headers:** `Authorization: Bearer {token}`<br>**Request Body:** См. Request JSON #5 ниже |
| 13. BookingService подтверждает займ и отправляет уведомление | См. SQL Query #4 и Notification JSON #6 ниже |
| 14. BookingService возвращает успешный результат | См. Response JSON #7 ниже |

#### Request JSON #1 - Запрос на выдачу книги
```json
{
  "userId": "mrmacgood71",
  "bookId": "book-205",
  "reservationId": "res-12345",
  "dueDays": 14
}
```
SQL Query #1 - Поиск абонемента пользователя
```SQL

SELECT a.abonement_id, a.user_id, a.status, a.issue_date, a.expiry_date, 
       DATEDIFF(a.expiry_date, CURDATE()) as days_until_expiry
FROM abonements a 
WHERE a.user_id = 'mrmacgood71' AND a.status = 'ACTIVE' 
LIMIT 1
```
Response JSON #2 - Данные абонемента
```JSON

{
  "abonementId": "abon-456",
  "userId": "mrmacgood71",
  "status": "ACTIVE",
  "issueDate": "2024-06-12",
  "expiryDate": "2025-06-17",
  "daysUntilExpiry": 5,
  "isExpired": false,
  "isExpiringSoon": true,
  "warningLevel": "HIGH",
  "maxBooksAllowed": 3,
  "currentBooksCount": 1
}
```
SQL Query #2 - Проверка статуса книги
```SQL

SELECT b.book_id, b.title, b.isbn, b.status, b.location, 
       r.reservation_id, r.status as reservation_status
FROM books b
WHERE b.book_id = 'book-205'
```
Response JSON #3 - Информация о книге
```JSON

{
  "bookId": "book-205", 
  "title": "Clean Architecture", 
  "isbn": "978-0134494166", 
  "status": "AVAILABLE", 
  "location": "Shelf-A-15", 
  "authors": [{"name": "Robert C. Martin"}], 
  "reservation": {
    "reservationId": "res-12345", 
    "status": "CONFIRMED", 
    "userId": "mrmacgood71"
  }, 
  "isAvailableForLoan": true
}
```
SQL Query #3 - Создание займа с предупреждением
```SQL

INSERT INTO book_loans (
  loan_id, user_id, book_id, reservation_id, issue_date, due_date,
  status, abonement_expiry_warning, warning_level, created_by
) VALUES (
  'loan-789', 'mrmacgood71', 'book-205', 'res-12345', '2025-06-12',
  '2025-06-26', 'PENDING_CONFIRMATION', true, 'HIGH', 'lib-456'
)
```
Response JSON #4 - Предупреждение об истечении абонемента (409 Conflict)
```JSON

{
  "errorCode": "ABONEMENT_EXPIRY_WARNING",
  "errorMessage": "Абонемент пользователя истекает через 5 дней",
  "warningData": {
    "userId": "mrmacgood71",
    "username": "mrmacgood71",
    "abonementId": "abon-456",
    "expiryDate": "2025-06-17",
    "daysUntilExpiry": 5,
    "warningLevel": "HIGH",
    "loanId": "loan-789",
    "bookTitle": "Clean Architecture"
  },
  "actions": {
    "confirm": "POST /api/v1/loans/loan-789/confirm",
    "cancel": "DELETE /api/v1/loans/loan-789",
    "renew_abonement": "POST /api/v1/users/mrmacgood71/abonements/renew"
  }
}
```
Request JSON #5 - Подтверждение выдачи с игнорированием предупреждения
```JSON

{
  "acknowledgeWarning": true,
  "warningType": "ABONEMENT_EXPIRY_WARNING",
  "librarianNote": "Пользователь предупрежден об истечении абонемента"
}
```
SQL Query #4 - Подтверждение займа
```SQL

UPDATE book_loans
SET status = 'ACTIVE', 
    confirmed_at = NOW(), 
    confirmed_by = 'lib-456', 
    librarian_note = 'Пользователь предупрежден об истечении абонемента'
WHERE loan_id = 'loan-789'
```
Notification JSON #6 - Уведомление пользователю
```JSON

{
  "userId": "mrmacgood71",
  "type": "ABONEMENT_EXPIRY_WARNING",
  "title": "Внимание: Ваш абонемент истекает",
  "message": "Ваш абонемент истекает 17.06.2025. Продлите его в ближайшее время.",
}
```
Response JSON #7 - Успешный результат (200 OK)
```JSON

{
  "success": true,
  "loanId": "loan-789",
  "message": "Книга выдана с предупреждением об истечении абонемента",
  "loanDetails": {
    "userId": "mrmacgood71",
    "bookId": "book-205",
    "bookTitle": "Clean Architecture",
    "issueDate": "2025-06-12",
    "dueDate": "2025-06-26",
    "status": "ACTIVE",
    "warning": {
      "type": "ABONEMENT_EXPIRY_WARNING",
      "message": "Абонемент истекает 17.06.2025",
      "daysUntilExpiry": 5
    }
  }
}
```
<details> <summary>Возможные ошибки</summary>

|Ошибка	|HTTP Code	|Response Example|
|---|---|---|
|Пользователь не найден|	404|	{"errorCode": "USER_NOT_FOUND", "errorMessage": "Пользователь не найден"}|
|Книга недоступна	|400	|{"errorCode": "BOOK_UNAVAILABLE", "errorMessage": "Книга недоступна для выдачи"}|
|Превышен лимит книг|	400|	{"errorCode": "LOAN_LIMIT_EXCEEDED", "errorMessage": "Превышен лимит одновременно взятых книг"}|
|Неверная резервация|	400|	{"errorCode": "INVALID_RESERVATION", "errorMessage": "Резервация не найдена или не принадлежит пользователю"}|
|Сервис недоступен	|500|	{"errorCode": "SERVICE_UNAVAILABLE", "errorMessage": "Временная недоступность сервиса"}|
</details> 

</details>

### ФТ-11. Создание метода уведомления пользователя о наступлении срока сдачи книги

Поле       | Значение
---------- | ------------------------------
Назначение | Метод нотификации о сроке сдачи книги
Метод HTTP | POST
Endpoint   | /api/v1/notify

![alt text](/assets/sequences/book_service__send_loan_notify_plant_seq.png)

<details> <summary>Логика работы </sumdmary>

##### Логика работы метода уведомления о наступлении срока сдачи книги

| Шаг | Значение |
|-----|----------|
| 1. Планировщик инициирует отправку уведомлений | **HTTP Request:** `POST /api/v1/notify` **Headers:** `Authorization: Bearer system_token, X-Service: scheduler`<br>**Body:**<br>```json{  "type": "due_date_reminder", "scheduledDate": "2025-06-12T19:55:46.877Z"}```<br>** |
| 2. API Gateway направляет запрос в NotificationService | **Request URL:** `POST /api/v1/notify `<br> ***Headers:** `Authorization: Bearer system_token, X-Service: scheduler`<br>**Body:**<br>```json{  "type": "due_date_reminder", "scheduledDate": "2025-06-12T19:55:46.877Z"}```<br>** |
| 3. NotificationService запрашивает займы у BookingService | **Request URL:** `GET /api/v1/loans/overdues?overdueDaysThreshhold=0`<br>|
| 4. BookingService находит займы со сроком сдачи завтра | **Query Logic:**<br>```sql<br>SELECT * FROM book_loans <br>WHERE due_date = '2025-06-12' <br>  AND status = 'ACTIVE'<br>  AND return_date IS NULL<br>``` |
| 5. BookingService возвращает список займов | **Loans Response:**<br>```json<br>[<br>  {<br>    "loanId": "loan-408",<br>    "userId": "mrmacgood71",<br>    "bookId": "book-408",<br>    "issueDate": "2025-06-10",<br>    "dueDate": "2025-06-12",<br>    "status": "ACTIVE"<br>  }<br>]<br>``` |
| 6. NotificationService запрашивает данные пользователя | **Request URL:** `GET /api/v1/users/{userId}?type=SHORT`<br>**Response:**<br>```json<br>{<br>  "userId": "mrmacgood71",<br>  "fullName": "Максим Макгуд",<br>  "email": "maxim@example.com",<br>  "phone": "+7-900-123-45-67",<br>  "notificationPreferences": {<br>    "emailEnabled": true,<br>    "smsEnabled": false,<br>    "dueDateReminders": true,<br>    "preferredTime": "09:00"<br>  }<br>}<br>``` |
| 7. NotificationService запрашивает информацию о книге | **Request URL:** `GET /api/v1/books/book-408`<br>**Response:**<br>```json<br>{<br>  "bookId": "book-408",<br>  "title": "Design Patterns",<br>  "isbn": "978-0201633612",<br>  "authors": [{"name": "Erich Gamma"}]<br>}<br>``` |
| 8. NotificationService маппит данные и подготавливает уведомление | |
| 9. NotificationService отправляет email уведомление | **Email Service Request:**<br>```json<br>{<br>  "to": "maxim@example.com",<br>  "subject": "Напоминание о возврате книги",<br>  "template": "due_date_reminder",<br>  "data": {<br>    "userName": "Максим Макгуд",<br>    "bookTitle": "Design Patterns",<br>    "dueDate": "12.06.2025",<br>    "libraryUrl": "https://library.example.com"<br>  }<br>}<br>``` |
| 10. NotificationService записывает лог уведомления |  |
| 11. NotificationService возвращает статистику | **Response to Gateway:**<br>```json<br>{<br>  "processedLoans": 1,<br>  "notificationsSent": 1,<br>  "channels": {<br>    "email": 1,<br>    "sms": 0<br>  },<br>  "failures": 0<br>}<br>``` |
| 12. API Gateway возвращает результат планировщику | **Final Response (200 OK):**<br>```json<br>{<br>  "success": true,<br>  "type": "due_date_reminder",<br>  "processedAt": "2025-06-11T16:47:08Z",<br>  "targetDate": "2025-06-12",<br>  "statistics": {<br>    "totalLoansFound": 1,<br>    "usersNotified": 1,<br>    "notificationsSent": 1,<br>    "channelBreakdown": {<br>      "email": 1,<br>      "sms": 0,<br>      "push": 0<br>    },<br>    "failures": []<br>  },<br>  "details": [<br>    {<br>      "userId": "mrmacgood71",<br>      "userName": "Максим Макгуд",<br>      "loanId": "loan-408",<br>      "bookTitle": "Design Patterns",<br>      "dueDate": "2025-06-12",<br>      "channelsSent": ["email"],<br>      "sentAt": "2025-06-11T16:47:08Z"<br>    }<br>  ]<br>}<br>``` |

##### Обработка ошибочных сценариев

| Ошибка | HTTP Code | Response Example |
|--------|-----------|------------------|
| Нет займов для уведомлений | 204 | `{"message": "No loans found requiring notification", "targetDate": "2025-06-12"}` |
| Неверный тип уведомления | 400 | `{"errorCode": "INVALID_NOTIFICATION_TYPE", "errorMessage": "Unsupported notification type"}` |
| Ошибка отправки уведомления | 207 | `{"success": true, "partialFailures": 2, "details": [...]}` |
| Сервис недоступен | 500 | `{"errorCode": "SERVICE_UNAVAILABLE", "errorMessage": "Unable to process notifications"}` |

</details>

### Нефункциональные требоваиня

#### НФТ-1. Реализация логирования методов сервиса

Необходимо реализовать требования к логированию с использованием ELK

1. Уровни логирования

```
ERROR   - Критические ошибки, требующие немедленного внимания
WARN    - Предупреждения о потенциальных проблемах
INFO    - Информационные сообщения о ключевых операциях
DEBUG   - Детальная отладочная информация
TRACE   - Максимально детальная трассировка выполнения
```

2. Структурированное логирование

Обязательные поля в каждом лог-сообщении:

```json

{
  "timestamp": "2025-06-10T09:24:30.123Z",
  "level": "INFO",
  "service": "book-service",
  "version": "1.0.0",
  "environment": "production",
  "instance_id": "book-service-pod-abc123",
  "correlation_id": "req-uuid-12345",
  "user_id": "mrmacgood71",
  "session_id": "sess-uuid-67890",
  "operation": "book.reserve",
  "message": "Book reserved successfully",
  "execution_time_ms": 245,
  "additional_data": {}
}

```

3. Бизнес-логирование

Критические бизнес-операции (уровень INFO):

    Резервирование книг
    Выдача и возврат книг
    Блокировка/разблокировка абонементов
    Импорт legacy данных
    Изменение настроек овербукинга
    Отправка уведомлений

Пример лога резервирования:

```json

{
  "timestamp": "2025-06-10T09:24:30.123Z",
  "level": "INFO",
  "operation": "book.reserve",
  "user_id": "mrmacgood71",
  "book_id": "book-123",
  "reservation_id": "res-456",
  "is_overbooking": true,
  "message": "Book reserved with overbooking",
  "metadata": {
    "book_title": "Clean Code",
    "available_copies": 0,
    "total_reservations": 15,
    "overbooking_percentage": 20
  }
}
```

4. Безопасность и аудит

```
Логирование событий безопасности (уровень WARN/ERROR):

    Неудачные попытки аутентификации
    Попытки доступа без прав
    Подозрительная активность (много запросов за короткое время)
    Изменение критических настроек
```

Пример лога безопасности:

```json
{
  "timestamp": "2025-06-10T09:24:30.123Z",
  "level": "WARN",
  "category": "security",
  "event": "unauthorized_access_attempt",
  "user_id": "unknown",
  "ip_address": "192.168.1.100",
  "endpoint": "/staff/users/123/status",
  "user_agent": "curl/7.68.0",
  "message": "Attempt to access staff endpoint without proper role"
}
```

5. Производительность

```
Логирование медленных операций (уровень WARN):

    Запросы к БД > 1000ms
    API запросы > 2000ms
    Операции импорта > 30s
```

6. Интеграции

Логирование внешних вызовов:

    Запросы к системе уведомлений
    Взаимодействие с IdP (Keycloak)
    Обращения к внешним сервисам

#### НФТ-2 Создание метрик для мониторинга Grafana

2.1 Системные метрики

Инфраструктурные метрики:

```YAML

# CPU и память
- cpu_usage_percent
- memory_usage_percent
- memory_usage_mb
- garbage_collection_duration_ms

# Диск и сеть
- disk_usage_percent
- network_in_bytes_per_sec
- network_out_bytes_per_sec

# JVM метрики (если Java)
- jvm_heap_used_mb
- jvm_heap_max_mb
- jvm_gc_collection_seconds
```

2.2 Бизнес-метрики

Ключевые показатели библиотеки:

```YAML

# Книги и резервирования
- books_total_count
- books_available_count
- books_reserved_count
- books_issued_count
- reservations_active_count
- reservations_overbooked_count
- overbooking_utilization_percent

# Пользователи
- users_active_count
- abonements_active_count
- abonements_expiring_7days_count
- abonements_blocked_count

# Операции
- book_reservations_per_hour
- book_issues_per_hour
- book_returns_per_hour
- notifications_sent_per_hour
- legacy_imports_success_rate
```

2.3 Производительность API

HTTP метрики:

```YAML

# Время ответа
- http_request_duration_seconds (гистограмма)
- http_request_duration_p95_seconds
- http_request_duration_p99_seconds

# Пропускная способность
- http_requests_total (счетчик с лейблами: method, endpoint, status)
- http_requests_per_second

# Ошибки
- http_requests_4xx_total
- http_requests_5xx_total
- http_error_rate_percent
```

2.4 База данных

Метрики БД:

```YAML

- db_connections_active
- db_connections_idle
- db_query_duration_seconds
- db_slow_queries_total
- db_deadlocks_total
- db_connection_pool_exhausted_total
```

2.5 Кастомные метрики

Специфичные для библиотеки:

```YAML

# Система уведомлений
- notifications_pending_count
- notifications_failed_count
- notifications_delivery_rate_percent

# Овербукинг
- overbooking_violations_count
- books_with_overbooking_enabled_count
```

#### НФТ-3 Создание механизма алертинга

1. Критические алерты (P1 - немедленная реакция)

```t
# Доступность сервиса
- name: "Service Down"
  condition: "http_requests_total == 0 for 2 minutes"
  severity: "critical"
  notification: ["pager", "email", "slack"]

# Высокий уровень ошибок
- name: "High Error Rate"
  condition: "http_error_rate_percent > 10% for 5 minutes"
  severity: "critical"
  notification: ["pager", "email"]

# Производительность
- name: "High Response Time"
  condition: "http_request_duration_p95_seconds > 5 seconds for 5 minutes"
  severity: "critical"
  notification: ["pager", "email"]

# База данных
- name: "Database Connection Pool Exhausted"
  condition: "db_connection_pool_exhausted_total > 0"
  severity: "critical"
  notification: ["pager", "email"]
```

2. Важные алерты (P2 - реакция в течение часа)

```code
# Система резервирования
- name: "Overbooking Violations"
  condition
```

#### НФТ-4 Создание правил безопасности

Необходимо реализовать механизм безопасности с помощью поддержки протокола OAuth 2.0 и передачей пары Bearer-токенов при вызове каждого метода

При вызове каждого метода необхдимо добавить заголовок форматом

``` http
Authorization: Bearer {access_token}
```

Если access_token сгорел, то необходимо запросить новую пару токенов через refresh_token и повторить запрос:

``` curl
curl -X POST "http://{keycloak_url}/realms/{realm}/protocol/openid-connect/token" \
  -d "client_id=CLIENT_ID" \
  -d "client_secret=CLIENT_SECRET" \
  -d "grant_type=refresh_token" \
  -d "refresh_token={refresh_token}"
```

### Системные требования

#### СТ-1 Создание технической инфраструктуры для реализации компонентной схемы системы

1. Необходимо реализовать 4 ландшафта системы: dev, test, predProd, prod
2. Для каждого ландшафта необходимо:
2.1 Заказать ресурсы по форме (http://{resourcesUrl}.ru), где необходимо определить следующие контейнеры
БД 4cpu, 8ram, 200 SSD
BookingService 4cpu, 8ram, 10 SSD
....
2.2 Получить сетевые доступы и права: (http://{networkPolicies}.ru)

#### СТ-2 Создание таблиц и баз данных

Для реализации логической и/или физической модели системы необходимо реализовать таблицы в базах данных/схемах согласно логической схеме данных

### Ограничения

1. Было принятно решение, что блокировка абонемента пользователя из-за просрочки равноценно предупреждениям по просрочке сдачи книг пользователями. Соответственно, реализация требования "Уведомление сотрудников о большой просрочке по даче книг, со стороны пользователя" входит в состав метода блокировки абонемента пользователя по Scheduler, в виде отдельного списка/окна на ui/рабочего email сотрудника.
2. Было принято решение не описывать требования к клиентской части приложения, ограничившись, что для реализации клиентской части, необходимо удовлетворять требованиям, что были описаны для actor в диаграммах последовательности
3. Создание моделей запроса и ответа не были описаны в IARCH, т к markdown не позволяет добавлять Swagger (как это, например, это делает Confluence), поэтому пришлось разделять их на два файла.
