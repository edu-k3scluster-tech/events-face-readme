# Разработка `events-face` сервиса

Разрабатываем сервис (`events-face`), который занимается отображением мероприятий и регистрацией на них. Сервис не владеет собственными мероприятиями, а получает их от агрегатора (`events-provider`). Агрегатор предоставляет собственный API для получения информации о мероприятиях и регистрации посетителей.

## Функциональные требования

Сервис должен уметь выполнять следующие задачи:
1. Сортировать мероприятия по полю `event_time` и фильтровать их по полям `registration_deadline`, `status` и `name` (full-text search).
2. Удалять мероприятия, которые закончились более чем 7 дней назад.
3. Регистрировать посетителя на мероприятия. При асинхронной регистрации на мероприятие посетитель должен получить подтверждение (или отказ) не позднее чем через 1 час.
4. При регистрации (успешное или нет) отправлять подтверждение через сервис уведомлений (`notification-service`).
5. На стороне сервиса необходимо вести учет всех зарегестрированных посетителей для последующей аналитики.

Допустима временная несогласованность данных с `events-provider`, при этом система должна обеспечивать их согласованность в конечном итоге (eventual consistency).

## Нефункциональные требования

При разработке необходимо учитывать следующие особенности:
1. Ожидается количество запросов к:
   - эндпоинту отображения ближайших открытых событий до 10.000 RPM;
   - эндпоинту регистрации на мероприятие до 300 RPM.
2. `events-provider` гарантирует доставку изменений мероприятий в Kafka-топик с политикой `At Most Once` (в редких случаях сообщение может потеряться).
3. В доменную область `events-provider` не входит сортировка или фильтрация по разным полям. Он предоставляет API только для получения мероприятий по дате изменения (`changed_at`).
4. Сервисы `events-provider` и `notification-service` могут быть перегружены и возвращать статус-коды `429`, `504` или `500`. Сервис `events-face` должен быть устойчив к таким сбоям: временная недоступность одного из сервисов не должна ломать основную логику.
5. В случае отказов зависимых сервисов необходимо предотвратить:
   - избыточное использование ресурсов сервиса;
   - ухудшение времени отклика для пользователей;
   - цепочку отказов, которая может привести к полной деградации системы.  
   Также важно защитить отказавший сервис от перегрузки повторными запросами, чтобы ускорить его восстановление и обеспечить стабильную работу всей системы.
6. Логика работы сервиса должна быть атомарной и устойчивой к падению самого сервиса в любой момент времени.