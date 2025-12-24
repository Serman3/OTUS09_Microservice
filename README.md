# OTUS09_Microservice
Представить в виде одной или нескольких диаграмм архитектуру приложения для игры Космический бой.  Цель: разбить Игру Космический бой на набор взаимодействующих между собой микросервисов и приложений.
## Event Storming
![event storming](img/event_storming.jpg)
## C4 model
C4 включает четыре уровня представления:

1. Context: высокоуровневый взгляд на систему. Показывает приложения и пользователей, без технических деталей.

2. Container: углубляет представление системы, описывая основные части, или "контейнеры" (backend-приложение, веб-приложение, мобильного приложение, базы данных, файловая система), которые входят в состав системы. На этом уровне определены функции каждого контейнера, технологические решения по языкам программирования, протоколы взаимодействия.

3. Component: детализирует каждый контейнер, описывая его компоненты и их взаимодействие.

4. Code: наиболее детальный уровень, описывающий внутреннюю структуру каждого компонента. Часто используются UML-диаграммы для его описания. Не обязателен.

### Context
![c4_context](img/c4_context.jpg)
### Container
![c4_container](img/c4_container.jpg)
## Endpoints и взаимодействие микросервисов
### 1. Auth Service (/auth/*)
* POST /auth/register - регистрация пользователя
* POST /auth/login - вход в систему
* POST /auth/refresh - обновление токена
* POST /auth/logout - выход из системы
### 2. User Service (/users/*)
* GET /users/{id} - получение профиля
* PUT /users/{id} - обновление профиля
* GET /users/{id}/stats - статистика игрока
* GET /users/search - поиск игроков
### 3. Tournament Service (/tournaments/*)
* POST /tournaments - создание турнира
* GET /tournaments - список турниров
* GET /tournaments/{id} - детали турнира
* POST /tournaments/{id}/register - регистрация в турнир
* PUT /tournaments/{id}/status - изменение статуса турнира
### 4. Battle Service (/battles/*)
* POST /battles - создание боя
* GET /battles/{id} - информация о бое
* POST /battles/{id}/submit - отправка программы
* GET /battles/{id}/replay - получение записи боя
### 5. Rating Service (/ratings/*)
* GET /ratings/user/{id} - рейтинг пользователя
* GET /ratings/tournament/{id} - рейтинг турнира
* POST /ratings/calculate - пересчет рейтинга
* POST /ratings/create - создание рейтинга
* PUT /ratings/{id}/update - обновление рейтинга
### 6. Notification Service (/notifications/*)
* GET /notifications/ws - WebSocket для уведомлений
* POST /notifications - отправка уведомления
* GET /notifications/user/{id} - уведомления пользователя
### 7. Battle Engine (внутренний API)
* POST /engine/battle/start - запуск боя
* GET /engine/battle/{id}/status - статус выполнения
* POST /engine/validate - валидация программы агента
## Sequence Diagrams (Частично)
### 1. Полный цикл регистрации и аутентификации
```mermaid
sequenceDiagram
    participant User as Пользователь
    participant Web as Веб-клиент
    participant GW as API Gateway
    participant Auth as Auth Service
    participant UserS as User Service
    participant Event as Event Store
    participant Broker as Message Broker
    participant Cache as Redis Cache
    participant DB_Auth as Auth DB
    participant DB_User as User DB

    User->>Web: Заполняет форму регистрации
    Web->>GW: POST /auth/register
    GW->>Auth: register(email, password)
    
    Auth->>DB_Auth: Проверить существование email
    alt Email уже существует
        Auth-->>GW: 409 Conflict
        GW-->>Web: Ошибка
        Web-->>User: "Email уже используется"
    else Email свободен
        Auth->>DB_Auth: Создать запись пользователя
        DB_Auth-->>Auth: User ID
        Auth->>Cache: Сохранить сессию
        Auth->>Event: emit UserRegisteredEvent
        Auth-->>GW: 201 Created + JWT токен
        
        GW->>UserS: POST /users (создать профиль)
        UserS->>DB_User: Создать профиль пользователя
        DB_User-->>UserS: Профиль создан
        UserS->>Event: emit UserProfileCreatedEvent
        
        Event->>Broker: publish UserRegisteredEvent
        Broker->>Broker: distribute to subscribers
        
        GW-->>Web: Успешная регистрация
        Web->>Web: Сохранить JWT в localStorage
        Web-->>User: "Регистрация успешна"
    end
```
![sd_1](img/sd_registartion.png)
### 2. Создание турнира организатором
```mermaid
sequenceDiagram
    participant Org as Организатор
    participant Admin as Админ-панель
    participant GW as API Gateway
    participant Auth as Auth Service
    participant Tourn as Tournament Service
    participant Battle as Battle Service
    participant Rating as Rating Service
    participant Notif as Notification Service
    participant Event as Event Store
    participant Broker as Message Broker
    participant DB_Tourn as Tournament DB

    Org->>Admin: Заполняет форму создания турнира
    Admin->>GW: POST /tournaments (с JWT)
    
    GW->>Auth: validateToken(JWT)
    Auth-->>GW: user_id, role="organizer"
    
    GW->>Tourn: createTournament(tournament_data, user_id)
    Tourn->>DB_Tourn: Сохранить турнир
    DB_Tourn-->>Tourn: tournament_id
    
    Tourn->>Event: emit TournamentCreatedEvent
    Tourn->>Broker: publish tournament.created
    
    par Параллельная обработка
        Broker->>Battle: tournament.created
        Battle->>Battle: Зарезервировать ресурсы для матчей
        Battle->>Event: emit BattleResourcesReservedEvent
        
        Broker->>Rating: tournament.created
        Rating->>Rating: Инициализировать рейтинг турнира
        Rating->>Event: emit TournamentRatingInitializedEvent
        
        Broker->>Notif: tournament.created
        Notif->>Notif: Уведомить подписчиков на новые турниры
        Notif->>Event: emit TournamentNotificationsSentEvent
    end
    
    Tourn-->>GW: 201 Created + tournament_id
    GW-->>Admin: Успешно
    Admin-->>Org: Турнир создан
```
![sd_2](img/sd_create_tournament.png)
### 3. Регистрация игрока на турнир
```mermaid
sequenceDiagram
    participant Player as Игрок
    participant Web as Веб-клиент
    participant GW as API Gateway
    participant Auth as Auth Service
    participant Tourn as Tournament Service
    participant User as User Service
    participant Rating as Rating Service
    participant Notif as Notification Service
    participant Event as Event Store
    participant Broker as Message Broker
    participant DB_Tourn as Tournament DB
    participant Cache as Redis Cache

    Player->>Web: Нажимает "Зарегистрироваться"
    Web->>GW: POST /tournaments/{id}/register
    
    GW->>Auth: validateToken(JWT)
    Auth-->>GW: user_id
    
    GW->>Tourn: registerForTournament(tournament_id, user_id)
    
    Tourn->>Cache: Получить кеш турнира
    alt Турнир в кеше
        Cache-->>Tourn: Данные турнира
    else
        Tourn->>DB_Tourn: Получить турнир
        DB_Tourn-->>Tourn: Данные турнира
        Tourn->>Cache: Кешировать на 5 минут
    end
    
    Tourn->>Tourn: Проверить условия регистрации
    
    alt Условия не выполнены
        Tourn-->>GW: 400 Bad Request
        GW-->>Web: Ошибка
        Web-->>Player: "Нельзя зарегистрироваться"
    else Условия выполнены
        Tourn->>DB_Tourn: Добавить участника
        Tourn->>Event: emit TournamentRegistrationEvent
        
        Tourn->>User: getUserProfile(user_id)
        User-->>Tourn: Профиль игрока
        
        Tourn->>Rating: getRating(user_id)
        Rating-->>Tourn: Рейтинг игрока
        
        Tourn->>Broker: publish tournament.player_registered
        
        Broker->>Notif: tournament.player_registered
        Notif->>Notif: Отправить подтверждение регистрации
        Notif->>Event: emit RegistrationNotificationSentEvent
        
        Tourn-->>GW: 200 OK
        GW-->>Web: Успешно
        Web-->>Player: "Вы зарегистрированы"
    end
```
![sd_3](img/sd_player_register_to_tournamnet.png)
### 4. Матчмейкинг и создание боя
```mermaid
sequenceDiagram
    participant Scheduler as Tournament Scheduler
    participant Tourn as Tournament Service
    participant Battle as Battle Service
    participant User as User Service
    participant Engine as Battle Engine
    participant Event as Event Store
    participant Broker as Message Broker
    participant DB_Battle as Battle DB

    Note over Scheduler,Engine: Автоматическое создание боя по расписанию турнира
    
    Scheduler->>Tourn: getNextRoundMatches(tournament_id)
    Tourn-->>Scheduler: Список пар игроков
    
    loop Для каждой пары игроков
        Scheduler->>Battle: createBattle(player1_id, player2_id, tournament_id)
        
        Battle->>User: getPlayerDetails(player1_id)
        User-->>Battle: Профиль игрока 1
        
        Battle->>User: getPlayerDetails(player2_id)
        User-->>Battle: Профиль игрока 2
        
        Battle->>DB_Battle: Создать запись боя
        DB_Battle-->>Battle: battle_id
        
        Battle->>Event: emit BattleCreatedEvent
        Battle->>Broker: publish battle.created
        
        Broker->>Engine: battle.created
        Engine->>Engine: Подготовить симуляцию
        Engine->>Event: emit BattlePreparationCompletedEvent
        
        Battle-->>Scheduler: battle_id создан
    end
    
    Scheduler->>Tourn: updateTournamentBracket(tournament_id, battles)
    Tourn->>Event: emit TournamentRoundScheduledEvent
```
![sd_4](img/sd_create_battle_scheduler.png)
### 5. Исполнение боя в Battle Engine
```mermaid
sequenceDiagram
    participant Engine as Battle Engine
    participant Battle as Battle Service
    participant Agent1 as Agent 1
    participant Agent2 as Agent 2
    participant Rating as Rating Service
    participant Replay as Replay Service
    participant Event as Event Store
    participant Broker as Message Broker
    participant Storage as Object Storage
    participant DB_Battle as Battle DB

    Note over Engine,DB_Battle: Этап 1: Инициализация
    
    Engine->>Storage: Загрузить код агента 1
    Storage-->>Engine: Код агента 1
    
    Engine->>Storage: Загрузить код агента 2
    Storage-->>Engine: Код агента 2
    
    Engine->>Engine: Инициализировать игровое поле
    Engine->>Event: emit BattleStartedEvent
    
    Note over Engine,Engine: Этап 2: Игровой цикл
    
    loop Каждый игровой тик (1..N)
        Engine->>Agent1: executeTurn(game_state)
        Agent1-->>Engine: Действие агента 1
        
        Engine->>Agent2: executeTurn(game_state)
        Agent2-->>Engine: Действие агента 2
        
        Engine->>Engine: Применить физику и правила
        Engine->>Engine: Обновить игровое состояние
        
        alt Условие победы выполнено
            Engine->>Engine: Определить победителя
            break
        end
    end
    
    Note over Engine,Storage: Этап 3: Завершение
    
    Engine->>Engine: Сгенерировать реплей
    Engine->>Storage: Сохранить реплей
    Storage-->>Engine: replay_url
    
    Engine->>DB_Battle: Сохранить результаты
    Engine->>Event: emit BattleCompletedEvent
    Engine->>Broker: publish battle.completed
    
    Broker->>Battle: battle.completed
    Battle->>Battle: Обновить статус боя
    
    Broker->>Rating: battle.completed
    Rating->>Rating: Обновить рейтинги игроков
    
    Broker->>Replay: battle.completed
    Replay->>Replay: Обработать реплей для CDN
```
![sd_5](img/battle_engine_scheduler.png)
### 6. Обновление рейтинга после боя (Saga)
```mermaid
sequenceDiagram
    participant Broker as Message Broker
    participant Rating as Rating Service
    participant Tourn as Tournament Service
    participant User as User Service
    participant Notif as Notification Service
    participant Event as Event Store
    participant DB_Rating as Rating DB
    participant Cache as Redis Cache
    participant DB_User as User DB

    Broker->>Rating: battle.completed event
    
    Rating->>DB_Rating: Получить текущие рейтинги
    DB_Rating-->>Rating: rating_player1, rating_player2
    
    Rating->>Rating: Рассчитать новые рейтинги (формула Elo)
    Note right of Rating: ΔR = K × (S - E)<br/>Новый рейтинг = Старый + ΔR
    
    Rating->>DB_Rating: Обновить рейтинги игроков
    Rating->>DB_Rating: Записать в историю изменений
    Rating->>Event: emit RatingsUpdatedEvent
    
    par Параллельные действия
        Rating->>Cache: Обновить кеш лидерборда
        Rating->>User: updateUserStats(player_id, new_rating)
        User->>DB_User: Обновить статистику
        
        Rating->>Tourn: updateTournamentResults(battle_id, winner_id)
        Tourn->>Tourn: Обновить турнирную таблицу
        
        Rating->>Broker: publish ratings.updated
        Broker->>Notif: ratings.updated
        Notif->>Notif: Отправить уведомления игрокам
    end
    
    Rating->>Event: emit RatingProcessingCompletedEvent
```
![sd_6](img/update_raiting_after_battler_saga.png)
### 7. Просмотр реплея боя
```mermaid
sequenceDiagram
    participant User as Пользователь
    participant Web as Веб-клиент
    participant GW as API Gateway
    participant Auth as Auth Service
    participant Battle as Battle Service
    participant Replay as Replay Service
    participant Cache as Redis Cache
    participant CDN as CDN
    participant Storage as Object Storage
    participant DB_Battle as Battle DB

    User->>Web: Кликает на "Смотреть реплей"
    Web->>GW: GET /battles/{id}/replay
    
    GW->>Auth: validateToken(JWT)
    Auth-->>GW: user_id
    
    GW->>Battle: getBattleReplay(battle_id, user_id)
    
    Battle->>Cache: Проверить кеш реплея
    alt Реплей в кеше
        Cache-->>Battle: replay_data
    else
        Battle->>DB_Battle: Получить информацию о бое
        DB_Battle-->>Battle: battle_metadata
        
        Battle->>Replay: generateReplayUrl(battle_id)
        Replay->>Storage: Получить реплей
        Storage-->>Replay: replay_file
        
        Replay->>CDN: Загрузить в CDN
        CDN-->>Replay: cdn_url
        
        Replay->>Cache: Кешировать на 24 часа
        Replay-->>Battle: replay_info
    end
    
    Battle-->>GW: replay_url + metadata
    GW-->>Web: Данные реплея
    
    Web->>CDN: Запросить видео-поток
    CDN-->>Web: Stream видео
    
    Web->>Web: Воспроизвести реплей
    Web-->>User: Показать реплей
```
![sd_7](img/video_transalter.png)
### 8. Полный турнирный цикл (Swiss система)
```mermaid
sequenceDiagram
    participant Scheduler as Tournament Scheduler
    participant Tourn as Tournament Service
    participant Rating as Rating Service
    participant Battle as Battle Service
    participant Engine as Battle Engine
    participant Broker as Message Broker
    participant Event as Event Store
    participant DB_Tourn as Tournament DB

    Note over Scheduler,DB_Tourn: Начало турнира
    
    Scheduler->>Tourn: startTournament(tournament_id)
    Tourn->>DB_Tourn: Обновить статус на "active"
    Tourn->>Event: emit TournamentStartedEvent
    
    loop Для каждого раунда (1..N)
        Note over Tourn,Rating: Пайринг раунда
        
        Tourn->>Rating: getCurrentRatings(tournament_id)
        Rating-->>Tourn: Рейтинги всех участников
        
        Tourn->>Tourn: Выполнить пайринг Swiss системы
        Tourn->>DB_Tourn: Сохранить пары раунда
        
        loop Для каждой пары
            Tourn->>Battle: createBattle(player1, player2)
            Battle->>Engine: scheduleBattle(battle_data)
            Engine-->>Battle: battle_scheduled
            Battle-->>Tourn: battle_created
        end
        
        Note over Engine,Broker: Ожидание завершения всех боев раунда
        
        par Для каждого боя
            Engine->>Engine: Симуляция боя
            Engine->>Broker: publish battle.completed
            Broker->>Rating: battle.completed
            Rating->>Rating: Обновить рейтинги
            Broker->>Tourn: battle.completed
            Tourn->>Tourn: Обновить результаты
        end
        
        Tourn->>Tourn: Проверить завершение раунда
        Tourn->>Event: emit TournamentRoundCompletedEvent
    end
    
    Note over Tourn,Event: Завершение турнира
    
    Tourn->>Tourn: Рассчитать финальные места
    Tourn->>DB_Tourn: Сохранить результаты
    Tourn->>Event: emit TournamentCompletedEvent
    Tourn->>Broker: publish tournament.completed
```
![sd_8](img/all_touranment_loop.png)
## Узкие места и проблемы масштабирования
### 1. Проблема: Нагрузка на Battle Engine
**Описание:** Игровые симуляции требуют значительных вычислительных ресурсов, особенно при массовых турнирах.</br>

**Решение:**
* Горизонтальное масштабирование Battle Engine с использованием очередей (Kafka)
* Балансировка нагрузки через Service Mesh
* Автоскейлинг на основе метрик CPU/памяти
* Кеширование результатов симуляций в Redis
### 2. Проблема: Задержки в обновлении рейтинга
**Описание:** Расчет рейтинга для регулярных турниров может создавать пиковую нагрузку.</br>
**Решение:**
* Асинхронная обработка обновлений рейтинга через Message Broker
* Пакетная обработка ночью для не-реaltime обновлений
### 3. Проблема: Хранение и отдача реплеев боев
**Описание:** Видеозаписи боев занимают много места и создают нагрузку на сеть.</br>
**Решение:**
* Использование CDN для раздачи статики
* Компрессия реплеев на лету
* Потоковая передача вместо скачивания целиком
* Хранение "горячих" реплеев в кеше, "холодных" - в S3 Glacier
* Использовать сторонний сервис

### 4. Согласованность данных между микросервисами
**Описание:** Сервисы имеют разные бд, распределенные транзакции теряют атомарность.</br>
**Решение:**
* Event Sourcing + Saga Pattern
* В сервисах добавить компенсирующие транзакции
* Получаем консистентность по итогу 

## Компоненты с часто меняющимися требованиями
### 1. Battle Engine (Игровой движок)
**Описание:** Балансировка игры, новые типы кораблей, изменение правил физики.</br>
**Решение:**
* SOLID
* Выделение интерфейсов для игровых объектов
* Использование dependency injection
* Конфигурация через JSON/YAML файлы
### 2. Rating Service (Рейтинговая система)
**Описание:** Изменение формул расчета, введение новых лиг, сезонные сбросы.</br>
**Решение:**
* SOLID
* Фабричный метод для создания калькуляторов рейтинга
### 3. Tournament Service (Турнирная система)
**Описание:** Новые форматы турниров, разные системы сеток, специальные правила.</br>
**Решение:**
* SOLID
* Абстрактный класс Tournament с шаблонными методами
* Конкретные реализации для разных форматов
* Система правил как отдельный компонент
## Описание решения
Разработанная микросервисная архитектура для игры "Космический бой" разделяет систему на 7 основных сервисов, каждый из которых отвечает за определенную бизнес-область:

1. Сервисы ядра (Auth, User, Tournament) обеспечивают базовую функциональность приложения.
2. Игровой движок (Battle Engine) выделен в отдельный сервис для изоляции ресурсоемких операций. (Должен быть описан в c4 модели в компонентах)
3. Вспомогательные сервисы (Rating, Notification) реализуют дополнительные функции.

### Ключевые архитектурные решения:
* API Gateway служит единой точкой входа, обеспечивая аутентификацию, лимиты запросов и маршрутизацию.
* Message Broker позволяет асинхронное взаимодействие между сервисами, уменьшая связность, транзакции.
* Разделение баз данных - каждый сервис управляет своей схемой данных.
* * Разделение кешей - некоторые сервисы используют кеш.
### Преимущества подхода:
* Масштабируемость: Каждый сервис можно масштабировать независимо.
* Устойчивость к отказам: Падение одного сервиса не приводит к остановке всей системы.
* Гибкость разработки: Команды могут работать над разными сервисами независимо.
* Технологическая гетерогенность: Возможность использовать разные языки/технологии для разных сервисов.
### Недостатки и компромиссы:
* Сложность развертывания: Требуется orchestration (Kubernetes).
* Сложность входа
* Сетевая задержка: Межсервисные вызовы добавляют latency.
* Сложность отладки: Требуется распределенная трассировка.
### Рекомендации по внедрению:
* Начать с монолита, выделяя сервисы по мере необходимости.
* Использовать feature flags для постепенного развертывания изменений.
* Внедрить comprehensive мониторинг с самого начала.
* Использовать contract testing для обеспечения совместимости API.
* Данная архитектура обеспечивает баланс между гибкостью, производительностью и сложностью поддержки, позволяя системе эволюционировать вместе с требованиями к игре.