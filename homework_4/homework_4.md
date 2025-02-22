## Cистема из 0й домашки

[Схема системы из 0й ДЗ](./zero_hw_scheme.png)

Система, предложенная мной в 0й ДЗ состоит из:
- Основной сервис-монолит
- Сервис для найма воркеров
- Сервис для сотрудников отдела контроля качества
- Сервис рассылки уведомлений

Никакими доменами и bounded-contexts я не мыслил при описании системы, но выделил  модули - можно условно их приравнять к контекстам:
- найм воркеров
- ААА - аутентификация\учетные записи\аудит (про то, что это надо игнорировать прочёл уже после)
- таск трекинг
- интеграция с пекарней
- биллинг
- интеграция с Golden hat
- контроль качества
- рассылка уведомлений
- тотализатор

## Расчёт Instability
### Notification Sender
Notification Sender
* Ca = 4 (Hiring Site, Monolith, Quality Control)
* Ce = 0
* I = 0 / (0 + 4) = 0

MCF Hiring Site
* Ca = 0
* Ce = 1
* I = 1 / (1 + 0) = 1

Monolith
* Ca = 0
* Ce = 2 - (Task Tracking, Weekly Billing)
* I = 2 / (2 + 0) = 1

Quality Control Site
* Ca = 0
* Ce = 1
* I = 1 / (1 + 0) = 1

### Quality Control Site
Quality Control
* Ca = 1
* Ce = 0
* I = 0 / (1 + 0) = 0

Monolith
* Ca = 0
* Ce = 1
* I = 1


## Изменение состава сервисов\контекстов
- сервис уведомлений прекратит своё существование, логика уведомлений будет встроена в соответствующие модули. Если в процессе реализации будет выявлена общая техническая логика  - её можно инкапсулировать в библиотеку и встраивать в соответствующий модуль как зависимость
- контекст биллинга будет разделён на два bounded-contexts - расчёты с воркерами, расчёты с клиентам. И будут представлены отдельными сервисами
- контекст интеграции с Golden Hat будет объединён с расчётами с воркерами. Модуль соответственно уедет в сервис биллинга воркеров
- контекст тотализатора уже выделен -> останется вынести его в отдельный сервис
- контекст таск трекинга претерпит самые сильные изменения
    - будет выделен контекст и сервис Склада и Лояльности
    - будет выделен контекст и сервис Матчинга воркеров
    - контекст Контроля Качества будет встроен в Таск Трекинг
- модуль интеграции с пекарней будет встроен в Склад и Лояльность

В результате было:
- сервис Найма
- сервис-монолит Таск Трекер
- сервис Контроля качества
- сервис Рассылки уведомлений

Станет:
- сервис Найма и Тестирования
- сервис Матчинга воркеров
- сервис Менеджмента тасок
- сервис Склада и Лояльности
- сервис Расчётов с воркерами (+ Golden Hat)
- сервис Расчётов с клиентами (+ множество интеграций с платежными системами клиентов)
- сервис Тотализатора

[Схема финальной архитектуры](./MCF_FinalArchitecture.png)

## План реализации
- **Шаг 0.** Встраивание сервиса уведомлений в модули системы. Не является шагом распила монолита, поэтому не зависит от критерия наличия ресурсов, опыта и инфры. Более того, встроить в модули монолита будет даже проще, чем в отдельные сервисы, но в целом не имеет значения, куда. Главное, как можно скорее устранить элемент - источник нестабильности и избыточного каплинга других модулей\сервисов
- **Шаг 0.5** Встраивание сервиса контроля качества в монолит (по тем же причинам)

### 1. Нет свободных людей и ресурсов, есть инфра и опыт

- **Шаг 1.** Вынос сервиса Матчинга воркеров - самый критичный для бизнеса модуль в монолите. Сопоставим по важности с наймом (но его я сразу вынес, хех). Матчингу требуется принципиально другая СУБД - графовая -> выбираю Capture Data Changes (пользуясь схемой 4 вопросов из урока)
- **шаг 2.** Вынос один из сервисов расчётов - оба критичные с точки требований по хранению данных и безопасности (т.е. сильно отличаются по характеристикам от основного монолита). Наверное, всё-таки выберу расчёты с клиентами - там много интеграций предвидится, часто будет меняться, чаще релизить - чем раньше окажется вне монолита, тем лучше. Повышенные требования к хранению данных мешают переиспользовать такую же БД (как минимум настройки будут отличаться), поэтому выбираю CDC как стратегию переноса.
- **шаг 3.** Вынос сервис расчёта с воркерами
- **шаг 4.** Вынос высоконагруженного сервиса таск трекинга в отдельный сервис. Возможно потребуется поменять технологический стек под цели high-load'а -> Strangler Fig Application. Начать стримить таски в монолит (в биллинговые сервисы уже стримится)
- **шаг 5.** Следующий критичный элемент по логике - это Склад и Лояльность, но по скольку они уже как два модуля в монолите, то проще вынести Тотализатор и привести систему к финальному виду. Ставки асинхронно общаются с трекингом и у них хранилище попроще, поэтому имеет смысл задействовать стратегию CDC.
-
### 2. Есть люди и ресурсы, нет инфры и опыта
- **Шаг 1.** Вынос сервиса Тотализатора - самый простой и безопасный для приобретения опыта и подготовки инфры (очереди сообщений, склетоны микросервисов и проч).
- **Шаг 2.** Вынос сервиса расчёта с клиентами. Менее нагруженный, чем основной монолит, меньше ресурсов требуется для его обслуживания. Но при этом повышенные требования к хранению данных, чем раньше вынесем, тем лучше.
- **Шаг 3.** Вынос сервис расчёта с воркерами.
- **Шаг 4.** Вынос сервиса Склада и Лояльности. У складских работников свой контекст -> поменяется модель данных -> CDC
- **Шаг 5.** Вынос сервиса Матчинга.
