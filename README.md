# Полное руководство по EScript 2.0

Это расширенная документация для единого языка EScript. Здесь собраны правила синтаксиса, формальная грамматика, справка по встроенным функциям, рекомендации по безопасности и отладке, а также пошаговые примеры для администраторов и авторов скриптов. В тексте нет длинных тире, только короткие дефисы. Все примеры используют файлы с расширением .escript.

## 1. Быстрый старт
1. Создайте файл с именем `my_script.escript`.
2. Опишите команды через `command "имя"(параметры):`.
3. Используйте отступы в два пробела. Табуляцию не используйте.
4. Загружайте файл целиком. JSON-actions и старые обёртки не принимаются.
5. Проверяйте лог компиляции: предупреждения и ошибки выводятся сразу при загрузке.
6. Команды автоматически добавляются в `!!помощь` под префиксами, настроенными в чате.

## 2. Формальная грамматика
Ниже указана базовая структура файла. Отступы определяют вложенность блоков.
```
script       ::= (command | fn | import_stmt | const)+
command      ::= "command" NAME command_params? command_return? ":" NEWLINE block
fn           ::= "fn" NAME fn_params? fn_return? ":" NEWLINE block
command_params ::= "(" param_list? ")"
fn_params      ::= "(" param_list? ")"
param_list   ::= param ("," param)*
param        ::= NAME ":" TYPE ("=" expr)?
command_return ::= "->" TYPE
fn_return      ::= "->" TYPE
block        ::= INDENT statement+ DEDENT
statement    ::= let_stmt | set_stmt | if_stmt | while_stmt | for_stmt | foreach_stmt | try_stmt | return_stmt | break_stmt | continue_stmt | expr_stmt
let_stmt     ::= "let" NAME ":" TYPE "=" expr
set_stmt     ::= "set" NAME "=" expr
if_stmt      ::= "if" expr ":" NEWLINE block ("else:" NEWLINE block)?
while_stmt   ::= "while" expr ":" NEWLINE block
for_stmt     ::= "for" NAME "in" range_expr ":" NEWLINE block
foreach_stmt ::= "for" NAME "in" expr ":" NEWLINE block
try_stmt     ::= "try:" NEWLINE block "catch" NAME ":" NEWLINE block
return_stmt  ::= "return" expr?
expr_stmt    ::= expr
range_expr   ::= expr "to" expr ("step" expr)?
expr         ::= literals | calls | operations | names
```
Типы: number, string, bool, array, object, null, any. Псевдонимы не поддерживаются.

## 3. Типы и вывод типов
- Все параметры объявляются с типами. Пример: `command "pay"(amount: number):`.
- `let` всегда требует тип. `set` использует ранее объявленный тип переменной.
- Возвращаемый тип в командах опционален, но помогает автодополнению и тестам.
- `any` используйте только когда входные данные не контролируются.
- Рантайм проверяет соответствие типов при каждом вызове функции или команды.

## 4. Области видимости
Каждая команда и каждая пользовательская функция создают собственный слой переменных. Внутренние блоки получают доступ к переменным внешнего блока только для чтения и записи через `set`. Переменная, объявленная через `let`, видна только в текущем блоке и его дочерних блоках. `var_unset` удаляет переменную из ближайшего доступного scope. Проверка `var_exists` сообщает, определено ли имя после учёта удаления.

### Пример со scope
```
command "scope_demo"():
  let base: number = 10
  if base > 5:
    let bonus: number = 2
    set base = base + bonus
  var_unset("bonus")
  if var_exists("bonus"):
    message_send(chat_id(ctx), "bonus still alive")
  else:
    message_send(chat_id(ctx), "bonus removed")
```
## 5. Ошибки и обработка
Все ошибки имеют код, сообщение и стек. Используйте try/catch для локального перехвата. Ошибки можно пробрасывать повторно через `throw`. Компилятор выдаёт предупреждения с привязкой к строкам и подсказками. Логи попадают в файлы скриптов и консоль администратора.

### Мягкая валидация
- Нетипизированный параметр автоматически превращается в `any` и помечается предупреждением.
- Неизвестный тип подменяется на `any`, чтобы загрузка не срывалась.
- `let` без значения получает `null`, без типа — `any`; обе подстановки фиксируются предупреждениями.
- Сигнатура без скобок трактуется как команда без аргументов, хвост строки остаётся описанием.
- Предупреждения записываются вместе со скриптом и показываются админам, но не блокируют исполнение.

```
command "safe_http"():
  try:
    let resp: object = http_get("https://example.com")
    if resp.status != 200:
      throw("http_failed")
  catch err:
    log("error", "request failed", err)
    return
```
## 6. Импорт модулей и неймспейсы
- Поддерживаются относительные импорты внутри папки скрипта: `import utils.time`.
- Имена модулей формируют неймспейс. Доступ к функциям: `utils.time.now()`.
- Импорты должны идти в начале файла до объявлений команд и функций.

## 7. Литералы и коллекции
- Объекты задаются как `{ "key": "value", "num": 1 }`. Значения могут быть любыми выражениями.
- Массивы задаются как `[1, 2, 3]` или смешанными типами `[1, "a", true]`.
- Вызовы функций могут возвращать объекты и массивы, их можно распаковывать в переменные.
- Доступ к полям объекта через точку: `user.name`. К индексам массива через квадратные скобки: `arr[0]`.

## 8. Управление потоком
Условия записываются в привычной форме. Циклы `for` умеют работать с диапазоном через `to` и шаг через `step`. `foreach` перебирает любой массив или объект (ключи). Блоки `try/catch` перехватывают ошибки рантайма. `break` и `continue` работают внутри циклов.

### Пример for c диапазоном
```
command "countdown"():
  for i in 5 to 1 step -1:
    message_send(chat_id(ctx), to_string(i))
```
### Пример foreach по массиву
```
command "list_items"():
  let items: array = ["яблоко", "груша", "слива"]
  for item in items:
    message_send(chat_id(ctx), item)
```
## 9. Пользовательские функции и перегрузка
- Функции объявляются через `fn`. Можно указать значения по умолчанию.
- Перегрузка разрешена по числу и типам аргументов. Рантайм выбирает лучшую сигнатуру.
- Возвращаемый тип обязателен для явности. Если не указан, используется any.
- Функции могут быть объявлены до или после команд, но лучше группировать по смыслу.

```
fn bonus(amount: number, factor: number=1.1) -> number:
  return amount * factor

fn bonus(amount: string) -> number:
  let parsed: number|null = to_number(amount)
  if parsed == null:
    throw("invalid_amount")
  return parsed
```
## 10. Макросы и шаблоны
Макросы позволяют описывать повторяющиеся действия экономии, банов, UI или логирования. Макрос разворачивается на этапе компиляции и проверяется типами так же, как обычный код.

```
macro ensure_balance(user_id: string, amount: number):
  if not balance_can_remove(user_id, amount):
    message_send(chat_id(ctx), "Недостаточно средств")
    return false
  return true
```
## 11. Дебаггер и трассировка
- Используйте `debug_break()` для установки точки останова. Выполнение остановится и покажет текущие переменные.
- Команда `debug_step()` выполняет следующую строку и снова останавливается.
- `debug_trace()` выводит стек вызовов и значения параметров.
- Включите детальную трассировку через флаг скрипта или настройку администратора.

## 12. Безопасность и лимиты
- Каждый скрипт получает лимиты по времени, памяти и сетевым запросам.
- HTTP доступ ограничен allowlist. Попытка обратиться на другой домен завершится ошибкой.
- Планировщик имеет антиддос: `rate_limit` ограничивает частоту операций.
- Экономические операции выполняются атомарно и проверяются на идемпотентность через ключ операции.

## 13. Событийная модель
- Триггеры: `on message`, `on join`, `on leave`, `on cron` и пользовательские события через `webhook_emit`.
- Пример: `command "on_message"(ctx: object):` будет вызван при новых сообщениях, если настроено в админке.
- Для cron используйте `schedule_every` или `schedule_once` внутри инициализационного блока.

## 14. UI слой
- Кнопки и формы описываются через `ui_form` и массивы кнопок в `message_send`.
- Каждая кнопка имеет тип и полезную нагрузку. Проверяйте payload в обработчике через строгие типы.
- Ответы форм возвращаются объектом с полями по идентификаторам полей.

## 15. Совместимость и миграции
- Скрипты версии 1.x можно обернуть в макросы совместимости. Включите режим миграции в настройках, чтобы подсказки указали несовместимые конструкции.
- Все новые примеры используют единый синтаксис без `call` и без JSON-обёрток.
- При обновлении проверяйте changelog и запускайте тестовый раннер для golden-тестов.

## 16. Тестовый раннер
- Команда раннера: `escript test my_script.escript`.
- Golden-тесты описывайте в каталоге `tests/` рядом со скриптом: входные данные и ожидаемые ответы в plain-тексте.
- Раннер поддерживает мокирование HTTP, времени и балансов для детерминированных проверок.

## 17. Стиль и форматтер
- Используйте два пробела. Одна инструкция в строке.
- Максимальная длина строки 100 символов. Переносите аргументы вертикально.
- Форматтер можно вызвать через `escript fmt file.escript`.
- Линтер сообщает о неиспользуемых переменных, неявных типах и запрещённых вызовах.

## 18. Префиксы и справка
- Префикс выбирается в настройках группы. Команды в коде указываются без префикса.
- В `!!помощь` скриптовые команды отображаются внизу в отдельной категории. Если скрипт выключен или с ошибкой, команды пропадают.
- Если описание не задано в сигнатуре, выводится текст "Описание отсутствует".

## 19. Справочник встроенных функций
Ниже приведены все встроенные функции с сигнатурами, описанием и примером использования. Все вызовы используют единый синтаксис `name(args)` без call.

### Переменные и типы
- **var_unset(name: string) -> void**
  - Удаляет переменную из ближайшего scope.
  - Пример: `var_unset("temp")`

- **var_exists(name: string) -> bool**
  - Проверяет, доступна ли переменная и не равна ли undefined.
  - Пример: `if var_exists("temp"):`

- **type_of(value: any) -> string**
  - Возвращает тип: number|string|bool|array|object|null|undefined.
  - Пример: `set t = type_of(value)`

- **to_number(value: any, default: number|null=null) -> number|null**
  - Безопасно приводит к числу.
  - Пример: `let n: number|null = to_number(arg)`

- **to_string(value: any) -> string**
  - Преобразует любое значение в строку.
  - Пример: `message_send(chat, to_string(value))`

- **to_bool(value: any) -> bool**
  - Приводит к bool по строкам true/false/1/0/yes/no.
  - Пример: `if to_bool(flag): log("info", "flag on")`


### JSON
- **json_parse(text: string) -> object|array|null**
  - Парсит JSON. При ошибке возвращает null и создаёт запись об ошибке.
  - Пример: `let data: object|null = json_parse(raw)`

- **json_stringify(value: any, pretty: bool=false) -> string**
  - Сериализует значение в JSON.
  - Пример: `set body = json_stringify(payload, true)`


### Массивы
- **array_new(...values: any[]) -> array**
  - Создаёт массив.
  - Пример: `let xs: array = array_new(1,2,3)`

- **array_push(arr_ref: array, value: any) -> number**
  - Добавляет элемент и возвращает длину.
  - Пример: `array_push(xs, 4)`

- **array_pop(arr_ref: array, default: any=null) -> any**
  - Снимает последний элемент или default.
  - Пример: `let last: any = array_pop(xs, null)`

- **array_get(arr: array, index: number, default: any=null) -> any**
  - Возвращает элемент по индексу, поддерживает отрицательные индексы.
  - Пример: `set first = array_get(xs, 0)`

- **array_set(arr_ref: array, index: number, value: any) -> void**
  - Устанавливает значение, растягивая массив при необходимости.
  - Пример: `array_set(xs, 5, 99)`

- **array_len(arr: array) -> number**
  - Возвращает длину.
  - Пример: `let size: number = array_len(xs)`

- **array_slice(arr: array, from: number=0, to: number=len) -> array**
  - Создаёт срез.
  - Пример: `let part: array = array_slice(xs, 1, 3)`

- **array_join(arr: array, sep: string=",") -> string**
  - Объединяет элементы в строку.
  - Пример: `message_send(chat, array_join(xs, ";"))`

- **array_unique(arr: array) -> array**
  - Удаляет дубликаты по строгому равенству.
  - Пример: `set uniq = array_unique(xs)`

- **array_shuffle(arr_ref: array, seed: number|null=null) -> array**
  - Перемешивает массив. При seed результат детерминирован.
  - Пример: `array_shuffle(xs, 42)`

- **array_contains(arr: array, value: any) -> bool**
  - Проверяет наличие элемента.
  - Пример: `if array_contains(xs, 5): ...`


### Объекты
- **map_new(pairs: array|null=null) -> object**
  - Создаёт объект. Пары можно передать как [[k,v], ...].
  - Пример: `let obj: object = map_new()`

- **map_get(obj: object, key: string, default: any=null) -> any**
  - Получает значение по ключу.
  - Пример: `let name: string = map_get(obj, "name", "?")`

- **map_set(obj_ref: object, key: string, value: any) -> void**
  - Устанавливает значение.
  - Пример: `map_set(obj, "age", 30)`

- **map_del(obj_ref: object, key: string) -> bool**
  - Удаляет ключ. Возвращает true при удалении.
  - Пример: `map_del(obj, "temp")`

- **map_keys(obj: object) -> array<string>**
  - Возвращает массив ключей.
  - Пример: `let keys: array = map_keys(obj)`

- **map_values(obj: object) -> array<any>**
  - Возвращает массив значений.
  - Пример: `let vals: array = map_values(obj)`

- **map_has(obj: object, key: string) -> bool**
  - Проверяет наличие ключа.
  - Пример: `if map_has(obj, "name"): ...`

- **map_merge(a: object, b: object, mode: string="overwrite") -> object**
  - Объединяет объекты. Режимы: overwrite, keep, deep.
  - Пример: `let merged: object = map_merge(a, b, "deep")`


### Поток
- **switch(value: any, cases: array<{when:any, actions:any}>, default_actions: array=[]) -> void**
  - Выполняет actions первой совпавшей ветки. Поддерживает match_regex.
  - Пример: `await switch(color, cases, [])`

- **while_loop(condition: expr, actions: array, max_iter: number=1000) -> void**
  - Цикл while с лимитом итераций.
  - Пример: `await while_loop(i < 10, actions)`

- **for_loop(var_name: string, from: number, to: number, step: number=1, actions: array, max_iter: number=10000) -> void**
  - Цикл for по диапазону с защитой от step=0.
  - Пример: `await for_loop("i", 0, 5, 1, actions)`

- **foreach(var_name: string, arr: array, actions: array, max_iter: number=10000) -> void**
  - Перебор элементов массива.
  - Пример: `await foreach("item", items, actions)`

- **break() -> void**
  - Останавливает текущий цикл.
  - Пример: `break()`

- **continue() -> void**
  - Переходит к следующей итерации.
  - Пример: `continue()`

- **try_catch(actions: array, catch_var: string="error", catch_actions: array) -> void**
  - Блок try/catch.
  - Пример: `await try_catch(main_actions, "err", handler)`


### Время
- **time_now() -> number**
  - Возвращает Unix time в секундах.
  - Пример: `let now: number = time_now()`

- **time_today(tz: string|null=null) -> number**
  - Индекс дня с учётом tz.
  - Пример: `let day: number = time_today(null)`

- **time_parse(text: string, tz: string|null=null) -> number|null**
  - Парсит ISO даты/времени.
  - Пример: `let ts: number|null = time_parse("2024-01-01")`

- **time_format(ts: number, format: string="YYYY-MM-DD HH:mm", tz: string|null=null) -> string**
  - Форматирует таймстамп.
  - Пример: `message_send(chat, time_format(ts))`

- **time_add_days(ts: number, days: number) -> number**
  - Добавляет дни.
  - Пример: `set future = time_add_days(now, 3)`

- **time_diff_days(a: number, b: number) -> number**
  - Разница в днях.
  - Пример: `let diff: number = time_diff_days(a, b)`


### Планировщик
- **schedule_once(ts: number, actions: any, id: string|null=null) -> string**
  - Запускает одноразовый таймер.
  - Пример: `let id: string = schedule_once(time_add_days(time_now(),1), actions)`

- **schedule_every(cron: string, actions: any, id: string|null=null, tz: string|null=null) -> string**
  - Cron планировщик.
  - Пример: `let job: string = schedule_every("0 9 * * *", actions)`

- **schedule_cancel(id: string) -> bool**
  - Отменяет задачу.
  - Пример: `schedule_cancel(job)`

- **schedule_list(prefix: string|null=null) -> array**
  - Возвращает активные задачи.
  - Пример: `let jobs: array = schedule_list(null)`


### Экономика
- **balance_get(user_id: string) -> number**
  - Возвращает баланс пользователя.
  - Пример: `let bal: number = balance_get(user_id(ctx))`

- **balance_can_remove(user_id: string, amount: number) -> bool**
  - Проверяет возможность списания.
  - Пример: `if not balance_can_remove(uid, cost): return`

- **balance_add(user_id: string, amount: number, reason: string|null=null) -> void**
  - Начисляет средства.
  - Пример: `balance_add(uid, reward, "bonus")`

- **balance_remove(user_id: string, amount: number, reason: string|null=null) -> bool**
  - Списывает с проверкой.
  - Пример: `balance_remove(uid, bet, "bet")`

- **balance_transfer_atomic(from_id: string, to_id: string, amount: number, reason: string|null=null) -> bool**
  - Атомарный перевод.
  - Пример: `balance_transfer_atomic(a,b,10,"tip")`

- **economy_lock(scope: string, key: string, seconds: number=5) -> bool**
  - Блокировка по scope: user|chat|global.
  - Пример: `economy_lock("user", uid, 5)`

- **economy_unlock(scope: string, key: string) -> void**
  - Снимает блокировку.
  - Пример: `economy_unlock("user", uid)`

- **treasury_get() -> number**
  - Получает баланс казны.
  - Пример: `let bank: number = treasury_get()`

- **treasury_add(amount: number, reason: string|null=null) -> void**
  - Пополнение казны.
  - Пример: `treasury_add(50, "event")`

- **treasury_remove(amount: number, reason: string|null=null) -> bool**
  - Списание из казны.
  - Пример: `treasury_remove(20, "payout")`


### Хранилище и транзакции
- **kv_incr(key: string, delta: number=1, init: number=0) -> number**
  - Инкремент volatile ключа.
  - Пример: `let val: number = kv_incr("visits")`

- **persist_incr(user_id: string, key: string, delta: number=1, init: number=0) -> number**
  - Инкремент user storage.
  - Пример: `persist_incr(uid, "score", 1)`

- **kv_set_ttl(key: string, value: any, seconds: number) -> void**
  - Устанавливает значение с TTL.
  - Пример: `kv_set_ttl("otp", code, 300)`

- **kv_get_meta(key: string) -> object**
  - Возвращает exists, expires_at, type, size.
  - Пример: `let meta: object = kv_get_meta("otp")`

- **kv_list(prefix: string, limit: number=100, cursor: string|null=null) -> object**
  - Постраничный список ключей.
  - Пример: `let page: object = kv_list("session:", 50, null)`

- **persist_list(user_id: string, prefix: string, limit: number=100, cursor: string|null=null) -> object**
  - Листинг user storage.
  - Пример: `let page2: object = persist_list(uid, "notes:", 20, null)`

- **db_transaction(ops: array<op>) -> object**
  - Атомарный пакет операций set/del/incr/ttl.
  - Пример: `db_transaction([{"op":"set","key":"a","value":1}])`


### Текст
- **text_lower(text: string) -> string**
  - Приводит к нижнему регистру.
  - Пример: `set low = text_lower(name)`

- **text_upper(text: string) -> string**
  - Приводит к верхнему регистру.
  - Пример: `text_upper(code)`

- **text_trim(text: string) -> string**
  - Убирает пробелы по краям.
  - Пример: `text_trim(input)`

- **text_replace(text: string, from: string|regex, to: string) -> string**
  - Заменяет подстроку или regex.
  - Пример: `text_replace(msg, "foo", "bar")`

- **text_split(text: string, sep: string|regex, limit: number|null=null) -> array<string>**
  - Разбивает строку.
  - Пример: `let parts: array = text_split(msg, " ")`

- **text_regex_match(text: string, pattern: string, flags: string="") -> object|null**
  - Ищет совпадение regex.
  - Пример: `let m: object|null = text_regex_match(msg, "^id:(\d+)")`

- **text_regex_findall(text: string, pattern: string, flags: string="") -> array**
  - Находит все совпадения.
  - Пример: `let hits: array = text_regex_findall(msg, "@([a-z]+)", "gi")`

- **text_format(template: string, values: object) -> string**
  - Форматирует строку по ключам.
  - Пример: `text_format("Привет, {user}", {"user":name})`


### Команды и пользователи
- **command_args(raw_text: string|null=null) -> array<string>**
  - Разбивает аргументы, учитывая кавычки.
  - Пример: `let args: array = command_args(null)`

- **command_arg(index: number, default: any=null) -> string|null**
  - Получает аргумент по индексу.
  - Пример: `set first = command_arg(0, "?")`

- **command_parse_amount(text: string, allow_all: bool=true) -> object**
  - Парсит суммы: 10, 1k, 2.5m, all.
  - Пример: `let parsed: object = command_parse_amount(arg)`

- **command_parse_user(text: string) -> string|null**
  - Определяет пользователя по @, id или reply.
  - Пример: `let target: string|null = command_parse_user(arg)`

- **user_id(ctx) -> string**
  - Возвращает id отправителя.
  - Пример: `let uid: string = user_id(ctx)`

- **user_name(user_id: string) -> string**
  - Получает имя пользователя.
  - Пример: `user_name(uid)`

- **user_mention(user_id: string) -> string**
  - Создаёт упоминание.
  - Пример: `message_send(chat, user_mention(uid))`

- **chat_id(ctx) -> string**
  - ID текущего чата.
  - Пример: `let cid: string = chat_id(ctx)`

- **chat_title(ctx) -> string**
  - Название чата.
  - Пример: `chat_title(ctx)`

- **user_role(user_id: string) -> string**
  - Роль пользователя.
  - Пример: `if user_role(uid) == "admin": ...`

- **permissions_has(user_id: string, perm: string) -> bool**
  - Проверяет права.
  - Пример: `if permissions_has(uid, "ban"): ...`

- **members_random(chat_id: string, count: number=1, filter: object|null=null) -> array<string>**
  - Выбирает случайных участников.
  - Пример: `let picks: array = members_random(chat_id(ctx), 2, {"not_bot":true})`

- **members_list(chat_id: string, filter: object|null=null, limit: number=200, cursor: string|null=null) -> object**
  - Листинг участников.
  - Пример: `let page: object = members_list(chat_id(ctx), null, 50, null)`


### UI и сообщения
- **ui_form(fields: array, title: string="") -> object**
  - Создаёт форму ввода.
  - Пример: `let form: object = ui_form([{"type":"text","name":"code"}], "Введите код")`

- **message_send(chat_id: string, text: string, buttons: array|null=null, reply_to: string|null=null) -> string**
  - Отправляет сообщение, возвращает message_id.
  - Пример: `let mid: string = message_send(chat_id(ctx), "hi", null, null)`

- **message_edit(message_id: string, text: string, buttons: array|null=null) -> bool**
  - Редактирует сообщение.
  - Пример: `message_edit(mid, "updated", null)`

- **message_delete(message_id: string) -> bool**
  - Удаляет сообщение.
  - Пример: `message_delete(mid)`

- **pin_set(chat_id: string, message_id: string) -> bool**
  - Закрепляет сообщение.
  - Пример: `pin_set(chat_id(ctx), mid)`

- **pin_clear(chat_id: string) -> bool**
  - Снимает закреп.
  - Пример: `pin_clear(chat_id(ctx))`

- **pin_get(chat_id: string) -> string|null**
  - Получает id закрепа.
  - Пример: `let pid: string|null = pin_get(chat_id(ctx))`

- **thread_storage(scope_id: string, key: string, value: any, ttl: number|null=null) -> void**
  - Сохраняет состояние ветки.
  - Пример: `thread_storage(chat_id(ctx), "step", 1, 600)`

- **thread_storage_get(scope_id: string, key: string, default: any=null) -> any**
  - Читает состояние ветки.
  - Пример: `let step: any = thread_storage_get(chat_id(ctx), "step", 0)`

- **thread_storage_del(scope_id: string, key: string) -> bool**
  - Удаляет состояние.
  - Пример: `thread_storage_del(chat_id(ctx), "step")`


### HTTP и крипто
- **http_get(url: string, headers: object|null=null, timeout_ms: number=5000) -> object**
  - HTTP GET. Возвращает status, headers, body.
  - Пример: `let resp: object = http_get("https://api.example.com")`

- **http_post(url: string, json: any, headers: object|null=null, timeout_ms: number=5000) -> object**
  - HTTP POST с JSON.
  - Пример: `http_post("https://api", {"a":1}, null, 3000)`

- **http_request(method: string, url: string, headers: object|null=null, body: any=null, timeout_ms: number=5000) -> object**
  - Универсальный HTTP.
  - Пример: `http_request("PUT", url, {"X":"1"}, body, 2000)`

- **webhook_emit(event: string, payload: any) -> bool**
  - Отправляет вебхук на разрешённый URL.
  - Пример: `webhook_emit("payment", {"id":123})`

- **crypto_hmac_sha256(text: string, secret: string, output: string="hex") -> string**
  - HMAC SHA256.
  - Пример: `let sig: string = crypto_hmac_sha256(body, secret, "hex")`

- **crypto_sha256(text: string, output: string="hex") -> string**
  - SHA256.
  - Пример: `crypto_sha256("abc")`

- **rate_limit(key: string, per_seconds: number, limit: number) -> bool**
  - Возвращает true если лимит не превышен.
  - Пример: `if not rate_limit("cmd:uid", 60, 5): return`


### Логи и метрики
- **log(level: string, message: string, data: any|null=null) -> void**
  - Структурный лог.
  - Пример: `log("info", "started", {"cmd":command})`

- **assert_(condition: bool, message: string="assert_failed") -> void**
  - Выбрасывает ошибку если условие ложно.
  - Пример: `assert_(user != null, "user_required")`

- **metrics_incr(name: string, value: number=1, tags: object|null=null) -> void**
  - Инкремент метрики.
  - Пример: `metrics_incr("bets", 1, {"game":"slots"})`

- **error_last() -> object|null**
  - Возвращает последнюю ошибку рантайма.
  - Пример: `let last = error_last()`

## 20. Практические рецепты

### Hello
```
command "hello"():
  message_send(chat_id(ctx), "Привет")
```

### Пари
```
command "bet"(amount: number):
  if not balance_can_remove(user_id(ctx), amount):
    message_send(chat_id(ctx), "Нет средств")
    return
  balance_remove(user_id(ctx), amount, "bet")
  message_send(chat_id(ctx), "Ставка принята")
```

### Форма ввода
```
command "ask"():
  let form: object = ui_form([{"type":"text","name":"city","title":"Город"}], "Укажите город")
  message_send(chat_id(ctx), "Форма отправлена")
```

### Расписание
```
command "daily"():
  schedule_every("0 10 * * *", [{"do":"message_send","args":[chat_id(ctx),"Доброе утро"]}])
```

### HTTP
```
command "check"():
  let resp: object = http_get("https://api.github.com")
  message_send(chat_id(ctx), to_string(resp.status))
```
## 21. Чеклист перед загрузкой
- Файл сохранён с суффиксом .escript.
- Команды имеют описания. Если нет, будет показано "Описание отсутствует".
- Отступы только два пробела.
- Сигнатуры параметров указаны с типами.
- Нет запретных доменов в HTTP запросах.
- Тестовый раннер проходит все сценарии.
- Макросы разворачиваются без ошибок.
- Проверены лимиты памяти и времени.

## 22. Лучшие практики
- Разделяйте код по модулям и используйте неймспейсы.
- Используйте строгие типы вместо any.
- Сохраняйте идемпотентные ключи для экономических операций.
- Логируйте каждое внешнее обращение и ошибку.
- Не держите блокировки дольше необходимого времени.
- Ставьте seed при тестируемом рандоме.
- Используйте форматтер перед коммитом.

## 23. Антипаттерны
- Хранить секреты в открытом виде в скрипте.
- Писать вложенные if без вынесения в функции.
- Использовать error_last как контроль потока.
- Игнорировать результат balance_remove.
- Ссылаться на переменные после var_unset.

## 24. Миграция 1.x -> 2.x
Шаги миграции:
- Удалите JSON actions. Оставьте только .escript.
- Замените конструкции call/type/actions на чистые выражения.
- Объявите типы параметров и возвращаемых значений.
- Проверьте справку команд: они теперь берут описания из сигнатур.
- Запустите линтер и форматтер.
- Обновите макросы и убедитесь, что они не используют устаревшие поля.

## 25. Подробные примеры по категориям
### Пример 1: Работа с коллекциями 1
```
command "demo_1"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 2: Работа с коллекциями 2
```
command "demo_2"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 3: Работа с коллекциями 3
```
command "demo_3"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 4: Работа с коллекциями 4
```
command "demo_4"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 5: Работа с коллекциями 5
```
command "demo_5"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 6: Работа с коллекциями 6
```
command "demo_6"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 7: Работа с коллекциями 7
```
command "demo_7"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 8: Работа с коллекциями 8
```
command "demo_8"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 9: Работа с коллекциями 9
```
command "demo_9"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 10: Работа с коллекциями 10
```
command "demo_10"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 11: Работа с коллекциями 11
```
command "demo_11"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 12: Работа с коллекциями 12
```
command "demo_12"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 13: Работа с коллекциями 13
```
command "demo_13"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 14: Работа с коллекциями 14
```
command "demo_14"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

### Пример 15: Работа с коллекциями 15
```
command "demo_15"():
  let xs: array = [1,2,3,4]
  array_shuffle(xs, 123)
  let text: string = array_join(xs, ", ")
  message_send(chat_id(ctx), text)
```

## 26. Подробные примеры по экономике
### Экономика 1
```
command "payout_1"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_1")
      treasury_add(amount, "payout_1")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 2
```
command "payout_2"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_2")
      treasury_add(amount, "payout_2")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 3
```
command "payout_3"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_3")
      treasury_add(amount, "payout_3")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 4
```
command "payout_4"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_4")
      treasury_add(amount, "payout_4")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 5
```
command "payout_5"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_5")
      treasury_add(amount, "payout_5")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 6
```
command "payout_6"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_6")
      treasury_add(amount, "payout_6")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 7
```
command "payout_7"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_7")
      treasury_add(amount, "payout_7")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 8
```
command "payout_8"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_8")
      treasury_add(amount, "payout_8")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 9
```
command "payout_9"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_9")
      treasury_add(amount, "payout_9")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

### Экономика 10
```
command "payout_10"(amount: number):
  if not economy_lock("user", user_id(ctx), 5):
    message_send(chat_id(ctx), "Повторите позже")
    return
  try:
    if balance_can_remove(user_id(ctx), amount):
      balance_remove(user_id(ctx), amount, "case_10")
      treasury_add(amount, "payout_10")
      message_send(chat_id(ctx), "Готово")
    else:
      message_send(chat_id(ctx), "Недостаточно")
  catch err:
    log("error", "fail", err)
  finally:
    economy_unlock("user", user_id(ctx))
```

## 27. UI паттерны
### Форма 1
```
command "form_1"():
  let form: object = ui_form([
    {"type":"text","name":"field","title":"Поле 1"},
    {"type":"number","name":"amount","title":"Сумма"}
  ], "Форма 1")
  message_send(chat_id(ctx), "Отправьте форму")
```

### Форма 2
```
command "form_2"():
  let form: object = ui_form([
    {"type":"text","name":"field","title":"Поле 2"},
    {"type":"number","name":"amount","title":"Сумма"}
  ], "Форма 2")
  message_send(chat_id(ctx), "Отправьте форму")
```

### Форма 3
```
command "form_3"():
  let form: object = ui_form([
    {"type":"text","name":"field","title":"Поле 3"},
    {"type":"number","name":"amount","title":"Сумма"}
  ], "Форма 3")
  message_send(chat_id(ctx), "Отправьте форму")
```

### Форма 4
```
command "form_4"():
  let form: object = ui_form([
    {"type":"text","name":"field","title":"Поле 4"},
    {"type":"number","name":"amount","title":"Сумма"}
  ], "Форма 4")
  message_send(chat_id(ctx), "Отправьте форму")
```

### Форма 5
```
command "form_5"():
  let form: object = ui_form([
    {"type":"text","name":"field","title":"Поле 5"},
    {"type":"number","name":"amount","title":"Сумма"}
  ], "Форма 5")
  message_send(chat_id(ctx), "Отправьте форму")
```

## 28. Трассировка и диагностика
- Диагностический совет 1: включайте debug_trace перед сложными циклами.
- Диагностический совет 2: включайте debug_trace перед сложными циклами.
- Диагностический совет 3: включайте debug_trace перед сложными циклами.
- Диагностический совет 4: включайте debug_trace перед сложными циклами.
- Диагностический совет 5: включайте debug_trace перед сложными циклами.

## 29. Золотые тесты
- Тест 1: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 2: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 3: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 4: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 5: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 6: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 7: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 8: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 9: проверяйте синтаксис макросов и стабильность форматтера.
- Тест 10: проверяйте синтаксис макросов и стабильность форматтера.

## 30. Полезные сниппеты для администраторов
```
command "admin_1"():
  log("info", "admin check 1")
  message_send(chat_id(ctx), "ok 1")
```

```
command "admin_2"():
  log("info", "admin check 2")
  message_send(chat_id(ctx), "ok 2")
```

```
command "admin_3"():
  log("info", "admin check 3")
  message_send(chat_id(ctx), "ok 3")
```

```
command "admin_4"():
  log("info", "admin check 4")
  message_send(chat_id(ctx), "ok 4")
```

```
command "admin_5"():
  log("info", "admin check 5")
  message_send(chat_id(ctx), "ok 5")
```

```
command "admin_6"():
  log("info", "admin check 6")
  message_send(chat_id(ctx), "ok 6")
```

```
command "admin_7"():
  log("info", "admin check 7")
  message_send(chat_id(ctx), "ok 7")
```

```
command "admin_8"():
  log("info", "admin check 8")
  message_send(chat_id(ctx), "ok 8")
```

```
command "admin_9"():
  log("info", "admin check 9")
  message_send(chat_id(ctx), "ok 9")
```

```
command "admin_10"():
  log("info", "admin check 10")
  message_send(chat_id(ctx), "ok 10")
```

## 31. Наблюдаемость
- Метрика 1: metrics_incr("custom_1", 1, {{"source":"doc"}})
- Метрика 2: metrics_incr("custom_2", 1, {{"source":"doc"}})
- Метрика 3: metrics_incr("custom_3", 1, {{"source":"doc"}})
- Метрика 4: metrics_incr("custom_4", 1, {{"source":"doc"}})
- Метрика 5: metrics_incr("custom_5", 1, {{"source":"doc"}})

## 32. Дополнительные замечания
Повторяйте практику: читать лог компиляции, запускать форматтер, обновлять тесты. Чем больше примеров вы пишете, тем легче поддерживать скрипты без ошибок.

