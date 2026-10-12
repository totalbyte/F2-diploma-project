# ParkOver — Thesis Proposal

| Поле | Значення |
| --- | --- |
| Студент | Антонюк Влад Володимирович |
| Науковий керівник | Тарновецька Ольга Юріївна |
| Спеціальність | F2 Інженерія програмного забезпечення |
| Освітній ступінь | Бакалавр |
| Статус документа | Верифіковано за допомогою генеративного ШІ, очікує відгуку керівника |

## Українська версія

### 1. Попередня робоча назва проєкту

- **UA:** Хмароорієнтована подієво-керована платформа управління міським паркуванням з моніторингом зайнятості в реальному часі, бронюванням та програмою лояльності
- **EN:** Cloud-Native Event-Driven Smart Parking Platform with Real-Time Space Availability, Reservations and Loyalty Program
- **Назва продукту:** ParkOver

### 2. Актуальність та проблематика

Зростання рівня автомобілізації міст спричиняє дефіцит паркувальних місць у центральних районах. Пошук вільного місця збільшує тривалість поїздки, інтенсивність руху, витрату пального та обсяг викидів. Наявні рішення переважно обслуговують окремі платні паркувальні майданчики та не надають актуальних даних про зайнятість вуличних паркувальних зон.

Інженерна складність задачі:

- **Доставка даних у реальному часі.** Стан паркувальних місць змінюється щохвилини; зміни необхідно доставляти великій кількості клієнтів із мінімальною затримкою.
- **Гетерогенність джерел даних.** Стан зайнятості формується з подій датчиків, паркувальних сесій водіїв і спостережень користувачів, що мають різний рівень достовірності. Обладнання кожного місця датчиками економічно недоцільне, тому для частини майданчиків дані користувачів є єдиним джерелом.
- **Достовірність і повнота даних від користувачів.** Користувачі не мають мотивації повідомляти про стан чужих місць, а отримані дані можуть бути застарілими або хибними. Система повинна отримувати дані як побічний результат власних дій водія та оцінювати їхню достовірність.
- **Конкурентний доступ до обмеженого ресурсу.** Одночасні запити на бронювання одного місця мають оброблятися без подвійного бронювання.
- **Узгодженість розподілених операцій.** Бронювання, оплата та нарахування бонусів виконуються різними сервісами і мають залишатися узгодженими у разі збою будь-якого з них.
- **Пікові навантаження.** Різке зростання кількості запитів у години пік вимагає горизонтального масштабування окремих компонентів.

### 3. Сформована ідея продукту

ParkOver — хмароорієнтована подієво-керована мікросервісна платформа з клієнтським застосунком у форматі Progressive Web Application (PWA). Платформа відображає паркувальні майданчики на інтерактивній мапі та надає дані про зайнятість місць у реальному часі.

Кожен паркувальний майданчик характеризується двома незалежними атрибутами:

- **тип оплати:** платний (з бронюванням та онлайн-оплатою) або безкоштовний;
- **джерело даних про зайнятість:** датчики паркувальних місць або дані водіїв (паркувальні сесії та спостереження з урахуванням рейтингу довіри).

Дані датчиків у межах проєкту генерує програмний симулятор, що емулює роботу апаратури за сценаріями навантаження та передає події IoT-протоколом MQTT. Формат і протокол повідомлень симулятора ідентичні до реальних пристроїв, що забезпечує підключення фізичних датчиків без змін у програмному коді системи.

Система передбачає три ролі: **водій**, **оператор паркувального майданчика** та **адміністратор**. Паркувальні майданчики реєструє оператор або адміністратор із зазначенням розташування, місткості, типу оплати, джерела даних і тарифів.

#### Архітектурні рішення

- Декомпозиція серверної частини на шість сервісів за бізнес-доменами: Identity, Parking Catalog, Occupancy, Booking, Payment, Loyalty. Кожен сервіс має власну базу даних (Database per Service) і внутрішню структуру за принципами Clean Architecture.
- Асинхронна взаємодія сервісів через брокер повідомлень RabbitMQ. Гарантовану доставку подій забезпечують патерн Transactional Outbox та ідемпотентні обробники.
- Розподілену транзакцію «бронювання → оплата → нарахування бонусів» координує оркестрована Saga з компенсуючими транзакціями.
- Сервіс зайнятості реалізовано за патерном CQRS: модель запису обробляє події датчиків і водіїв, модель читання в Redis обслуговує запити мапи.
- Єдина точка входу — API Gateway з автентифікацією, рольовою авторизацією (RBAC) та обмеженням частоти запитів (Token Bucket).
- Доставка змін клієнтам у реальному часі через SignalR із горизонтальним масштабуванням на основі Redis backplane.
- Вибір RabbitMQ замість Apache Kafka фіксується в Architecture Decision Record: обсяг подій не потребує журналу з тривалим зберіганням і повторним читанням, а RabbitMQ нативно підтримує протокол MQTT.

### 4. Детальний опис функціоналу (Core Features)

1. **Інтерактивна мапа паркувальних майданчиків.** Кольорова індикація статусу місць, геопошук у заданому радіусі, кластеризація маркерів, оновлення статусів у реальному часі без перезавантаження сторінки.
2. **Гібридний моніторинг зайнятості.** Прийом подій від датчиків через MQTT. Для майданчиків без датчиків — оцінка заповненості зони на основі паркувальних сесій і спостережень водіїв з перевіркою геолокації автора, рейтингом довіри та зниженням ваги спостережень з часом. Індикація джерела даних у картці майданчика.
3. **Бронювання паркувальних місць.** Бронювання місця на платному майданчику з виключенням подвійного бронювання на рівні СУБД (exclusion constraint) та автоматичним скасуванням неоплачених бронювань після закінчення відведеного часу.
4. **Керування паркувальною сесією.** Реєстрація прибуття із заявленою тривалістю стоянки, контекстний запит після прибуття за маршрутом, продовження сесії, push-сповіщення перед завершенням. Життєвий цикл сесії реалізовано як скінченний автомат: Активна → Очікує підтвердження → Непідтверджена → Завершена. На платних майданчиках з датчиками — контроль перевищення оплаченого часу: пільговий період, донарахування за підвищеним тарифом, блокування нових бронювань за наявності заборгованості.
5. **Онлайн-оплата.** Інтеграція з LiqPay (режим sandbox): формування та верифікація цифрового підпису, ідемпотентна обробка callback-повідомлень, повторні спроби та circuit breaker для викликів зовнішнього API.
6. **Програма лояльності.** Вітальна знижка за реєстрацію, знижка до дня народження, промокоди, бонусні бали за спостереження, підтверджені іншими водіями або датчиками. Сторнування бонусів у разі скасування операції.
7. **Побудова маршруту.** Розрахунок і відображення маршруту від поточного місцезнаходження водія до обраного майданчика.

### 5. Попередній інструментарій та технологічний стек

| Категорія | Технології |
| --- | --- |
| Мови програмування | C#, TypeScript, SQL |
| Архітектурні патерни | Microservices, Database per Service, Event-Driven Architecture (Publish/Subscribe), API Gateway, Saga (оркестрація), Transactional Outbox, CQRS, Clean Architecture, State Machine, RBAC, Retry, Circuit Breaker |
| Фреймворки та бібліотеки (backend) | ASP.NET Core (.NET 10), Entity Framework Core, SignalR, MassTransit, YARP, Microsoft.Extensions.Http.Resilience |
| Фреймворки та бібліотеки (frontend) | React, Vite, Tailwind CSS, shadcn/ui, Framer Motion, MapLibre GL JS, vite-plugin-pwa |
| Бази даних та сховища | PostgreSQL з розширенням PostGIS; Redis (модель читання, кеш, SignalR backplane, розподілені блокування) |
| Брокер повідомлень | RabbitMQ з MQTT Plugin |
| Безпека | OAuth 2.0, OpenID Connect, JWT, Azure Key Vault, TLS для MQTT, SonarQube (статичний аналіз), Trivy (сканування образів) |
| Тестування та якість | xUnit, Testcontainers, Playwright, k6, SonarQube Quality Gate |
| Спостережуваність (Observability) | OpenTelemetry, Prometheus, Grafana, Jaeger, Serilog |
| DevOps-інструменти | Git, GitHub, GitHub Actions, Docker, Docker Compose, Azure Container Apps (автомасштабування KEDA), Azure Container Registry, OpenTofu |
| Документування | C4 Model, Architecture Decision Records, OpenAPI, AsyncAPI, Markdown (Docs-as-Code) |
| Зовнішні інтеграції | LiqPay API (sandbox), OpenStreetMap, OpenRouteService, Web Push API |
| Емуляція обладнання | Програмний симулятор IoT-датчиків паркувальних місць (C#) |

## English Version

### 1. Preliminary Project Title

- **UA:** Хмароорієнтована подієво-керована платформа управління міським паркуванням з моніторингом зайнятості в реальному часі, бронюванням та програмою лояльності
- **EN:** Cloud-Native Event-Driven Smart Parking Platform with Real-Time Space Availability, Reservations and Loyalty Program
- **Product name:** ParkOver

### 2. Relevance and Problem Statement

Growing car ownership in cities causes a shortage of parking spaces in central districts. Searching for a free space increases trip duration, traffic intensity, fuel consumption and emissions. Existing solutions mostly serve individual paid parking facilities and do not provide up-to-date occupancy data for on-street parking zones.

Engineering challenges:

- **Real-time data delivery.** Parking space states change every minute; updates must reach a large number of clients with minimal latency.
- **Heterogeneous data sources.** Occupancy is derived from sensor events, driver parking sessions and user observations with different levels of reliability. Equipping every space with sensors is not cost-effective, so for some facilities user data is the only available source.
- **Reliability and completeness of user data.** Users have no incentive to report the state of other drivers' spaces, and the data received may be outdated or false. The system must collect data as a by-product of the driver's own actions and assess its reliability.
- **Concurrent access to a limited resource.** Simultaneous requests to book the same space must be processed without double booking.
- **Consistency of distributed operations.** Booking, payment and bonus accrual are performed by different services and must remain consistent if any of them fails.
- **Peak loads.** Sharp growth in request volume during rush hours requires horizontal scaling of individual components.

### 3. Product Concept

ParkOver is a cloud-native event-driven microservice platform with a client application delivered as a Progressive Web Application (PWA). The platform displays parking facilities on an interactive map and provides real-time space occupancy data.

Each parking facility is described by two independent attributes:

- **payment type:** paid (with booking and online payment) or free;
- **occupancy data source:** parking space sensors or driver data (parking sessions and observations weighted by a trust rating).

Within the project, sensor data is generated by a software simulator that emulates hardware behaviour under load scenarios and publishes events via the MQTT IoT protocol. The simulator's message format and protocol are identical to those of real devices, which allows physical sensors to be connected without changes to the system code.

The system defines three roles: **driver**, **parking facility operator** and **administrator**. Parking facilities are registered by an operator or administrator, specifying location, capacity, payment type, data source and tariffs.

#### Architectural Decisions

- The backend is decomposed into six services by business domain: Identity, Parking Catalog, Occupancy, Booking, Payment, Loyalty. Each service owns its database (Database per Service) and follows Clean Architecture internally.
- Services communicate asynchronously via the RabbitMQ message broker. Guaranteed event delivery is ensured by the Transactional Outbox pattern and idempotent consumers.
- The distributed transaction "booking → payment → bonus accrual" is coordinated by an orchestrated Saga with compensating transactions.
- The Occupancy service implements CQRS: the write model processes sensor and driver events, while a read model in Redis serves map queries.
- A single entry point is provided by an API Gateway with authentication, role-based authorization (RBAC) and rate limiting (Token Bucket).
- Real-time updates are delivered to clients via SignalR, horizontally scaled with a Redis backplane.
- The choice of RabbitMQ over Apache Kafka is recorded in an Architecture Decision Record: the event volume does not require a long-retention replayable log, and RabbitMQ natively supports MQTT.

### 4. Core Features

1. **Interactive parking map.** Colour-coded space status, geospatial search within a given radius, marker clustering, real-time status updates without page reload.
2. **Hybrid occupancy monitoring.** Ingestion of sensor events via MQTT. For facilities without sensors, zone occupancy is estimated from driver parking sessions and observations, with author geolocation verification, a trust rating and time-based decay of observation weight. The data source is indicated on the facility card.
3. **Parking space booking.** Booking a space at a paid facility, with double booking prevented at the database level (exclusion constraint) and automatic cancellation of unpaid bookings after the allotted time.
4. **Parking session management.** Arrival registration with a declared parking duration, a contextual prompt upon arrival via route, session extension, push notifications before expiry. The session lifecycle is implemented as a finite state machine: Active → Awaiting Confirmation → Unconfirmed → Completed. At paid facilities with sensors, overstay control is applied: grace period, surcharge at an increased tariff, and blocking of new bookings while debt is outstanding.
5. **Online payment.** LiqPay integration (sandbox mode): digital signature generation and verification, idempotent callback processing, retries and a circuit breaker for external API calls.
6. **Loyalty program.** Welcome discount on registration, birthday discount, promo codes, bonus points for observations confirmed by other drivers or sensors. Bonus reversal when an operation is cancelled.
7. **Route planning.** Calculation and display of a route from the driver's current location to the selected facility.

### 5. Preliminary Toolset and Technology Stack

| Category | Technologies |
| --- | --- |
| Programming languages | C#, TypeScript, SQL |
| Architectural patterns | Microservices, Database per Service, Event-Driven Architecture (Publish/Subscribe), API Gateway, Saga (orchestration), Transactional Outbox, CQRS, Clean Architecture, State Machine, RBAC, Retry, Circuit Breaker |
| Frameworks and libraries (backend) | ASP.NET Core (.NET 10), Entity Framework Core, SignalR, MassTransit, YARP, Microsoft.Extensions.Http.Resilience |
| Frameworks and libraries (frontend) | React, Vite, Tailwind CSS, shadcn/ui, Framer Motion, MapLibre GL JS, vite-plugin-pwa |
| Databases and storage | PostgreSQL with PostGIS; Redis (read model, cache, SignalR backplane, distributed locks) |
| Message broker | RabbitMQ with MQTT Plugin |
| Security | OAuth 2.0, OpenID Connect, JWT, Azure Key Vault, TLS for MQTT, SonarQube (static analysis), Trivy (image scanning) |
| Testing and quality | xUnit, Testcontainers, Playwright, k6, SonarQube Quality Gate |
| Observability | OpenTelemetry, Prometheus, Grafana, Jaeger, Serilog |
| DevOps tools | Git, GitHub, GitHub Actions, Docker, Docker Compose, Azure Container Apps (KEDA autoscaling), Azure Container Registry, OpenTofu |
| Documentation | C4 Model, Architecture Decision Records, OpenAPI, AsyncAPI, Markdown (Docs-as-Code) |
| External integrations | LiqPay API (sandbox), OpenStreetMap, OpenRouteService, Web Push API |
| Hardware emulation | Software simulator of IoT parking space sensors (C#) |
