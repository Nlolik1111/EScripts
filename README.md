# EScript JSON Runtime: полная спецификация (актуальная)

Этот документ — главный справочник по JSON-скриптам `.escript.json`

## 1. Принципы
- Поддерживается **только JSON-формат** (файлы `.escript.json`).
- DSL существует только для диагностики: **источник истины — JSON**.
- Скриптовые команды **могут вызываться с любым префиксом** (не только `!!`).
- Если команда написана неточно, но смысл понятен, система пытается выполнить **ближайшую по смыслу** команду.
- Внутренние функции скрипта можно вызывать в выражениях: `get_bonus()`.
- Константы из `consts` доступны **как обычные переменные** (например, `daily_bonus`).

## 2. Структура файла
```json
{
  "imports": ["math", "text"],
  "consts": {"tax": 5},
  "functions": [/* ... */],
  "commands": [/* ... */],
  "events": [/* ... */],
  "source": "опционально"
}
```

## 3. Типы
- `number`: число (int/float)
- `string`: строка
- `bool`: логическое значение
- `array`: массив
- `object`: объект
- `null`: пустое значение
- `any`: любой тип

## 4. Выражения (минимальный Lua-стиль)
Поддерживаются:
- Литералы: числа, строки, `true`/`false`, `null`
- Переменные и константы
- Арифметика: `+ - * / %`
- Сравнения: `== != < <= > >=`
- Логика: `and`, `or`, `not`
- Индексация: `arr[0]`, `obj["key"]`
- Атрибуты: `obj.key`
- Вызов функций: `balance.get(user_id)`, `get_bonus()`

## 5. Action-блоки (JSON)
Ниже перечислены допустимые узлы действий и эквиваленты в DSL.

### 5.1 let
```json
{"let": {"name": "x", "type": "number", "value": 10}}
```
DSL: `let x: number = 10`

### 5.2 set
```json
{"set": {"name": "x", "value": 11}}
```
DSL: `set x = 11`

### 5.3 call
```json
{"call": "message.send", "args": [{"expr": "chat.id(ctx)"}, "hello"]}
```
DSL: `call message.send(chat.id(ctx), "hello")`

### 5.4 if / else
```json
{"if": "x > 0", "then": [...], "else": [...]}
```
DSL:
```
if x > 0:
  ...
else:
  ...
```

### 5.5 while
```json
{"while": "i < 10", "body": [...]}
```

### 5.6 for (итерирование)
```json
{"for": {"var": "u", "in": "users"}, "body": [...]}
```

### 5.7 for_range
```json
{"for": {"var": "i", "from": 1, "to": 10, "step": 1}, "body": [...]}
```

### 5.8 try/catch/finally
```json
{"try": [...], "catch": [...], "catch_as": "err", "finally": [...]}
```

### 5.9 return / throw / break / continue
```json
{"return": {"expr": "value"}}
```
```json
{"throw": {"expr": "error message"}}
```
```json
{"break": true}
```
```json
{"continue": true}
```

## 6. Команды, префиксы, неточности
- Скриптовые команды ищутся **по смыслу** (fuzzy-match) и **без учёта регистра**.
- Можно писать команды **с любым префиксом**: `!`, `/`, `.`, `>>>`, `#`, `?`, `//` и т.д.
- Если команда имеет пробелы в имени, система сопоставляет **по началу полной фразы**.

## 7. Пользовательские функции
- Определяются в `functions` с полями `name`, `params`, `returns`, `actions`.
- Внутри выражений можно вызывать напрямую: `get_bonus()`.
- В `call` можно вызывать по имени: `{"call": "get_bonus"}`.

## 8. События
- События в `events` исполняются планировщиком (например, `daily`).

## 9. Пространства имён и функции (полный список + мини-скрипт для каждой)

### json.*
- `json.parse(text: string) -> object|array`
  - Мини-скрипт: `{"let": {"name": "data", "type": "object", "value": {"expr": "json.parse(raw)"}}}`
- `json.stringify(value: any) -> string`
  - Мини-скрипт: `{"let": {"name": "raw", "type": "string", "value": {"expr": "json.stringify(data)"}}}`

### array.*
- `array.new(items?: array) -> array` — создать массив.
  - `{"let": {"name": "arr", "type": "array", "value": {"expr": "array.new([1,2,3])"}}}`
- `array.len(arr: array) -> number` — длина массива.
  - `{"let": {"name": "n", "type": "number", "value": {"expr": "array.len(arr)"}}}`
- `array.get(arr: array, idx: number, default?: any) -> any` — получить элемент.
  - `{"let": {"name": "first", "type": "any", "value": {"expr": "array.get(arr, 0)"}}}`
- `array.set(arr: array, idx: number, value: any) -> array` — записать элемент.
  - `{"call": "array.set", "args": [{"expr": "arr"}, 0, 99]}`
- `array.push(arr: array, value: any) -> array` — добавить в конец.
  - `{"call": "array.push", "args": [{"expr": "arr"}, "item"]}`
- `array.pop(arr: array) -> any` — взять с конца.
  - `{"let": {"name": "last", "type": "any", "value": {"expr": "array.pop(arr)"}}}`
- `array.contains(arr: array, value: any) -> bool` — содержит ли элемент.
  - `{"let": {"name": "has", "type": "bool", "value": {"expr": "array.contains(arr, 10)"}}}`
- `array.join(arr: array, sep?: string) -> string` — склеить строки.
  - `{"let": {"name": "csv", "type": "string", "value": {"expr": "array.join(arr, ',')"}}}`
- `array.slice(arr: array, start?: number, end?: number) -> array` — срез.
  - `{"let": {"name": "part", "type": "array", "value": {"expr": "array.slice(arr, 0, 3)"}}}`
- `array.shuffle(arr: array) -> array` — перемешать.
  - `{"call": "array.shuffle", "args": [{"expr": "arr"}]}`
- `array.unique(arr: array) -> array` — уникальные значения.
  - `{"let": {"name": "uniq", "type": "array", "value": {"expr": "array.unique(arr)"}}}`

### map.*
- `map.new(pairs?: array) -> object`
  - `{"let": {"name": "m", "type": "object", "value": {"expr": "map.new()"}}}`
- `map.get(obj: object, key: string, default?: any) -> any`
  - `{"let": {"name": "val", "type": "any", "value": {"expr": "map.get(m, 'k', 0)"}}}`
- `map.set(obj: object, key: string, value: any) -> object`
  - `{"call": "map.set", "args": [{"expr": "m"}, "k", 1]}`
- `map.del(obj: object, key: string) -> bool`
  - `{"let": {"name": "removed", "type": "bool", "value": {"expr": "map.del(m, 'k')"}}}`
- `map.has(obj: object, key: string) -> bool`
  - `{"let": {"name": "exists", "type": "bool", "value": {"expr": "map.has(m, 'k')"}}}`
- `map.keys(obj: object) -> array`
  - `{"let": {"name": "keys", "type": "array", "value": {"expr": "map.keys(m)"}}}`
- `map.values(obj: object) -> array`
  - `{"let": {"name": "vals", "type": "array", "value": {"expr": "map.values(m)"}}}`
- `map.merge(a: object, b: object, mode?: string) -> object`
  - `{"let": {"name": "merged", "type": "object", "value": {"expr": "map.merge(a, b, 'overwrite')"}}}`

### text.*
- `text.format(template: string, args?: array|object) -> string`
  - `{"let": {"name": "t", "type": "string", "value": {"expr": "text.format('Hi {name}', {\"name\": \"Alex\"})"}}}`
- `text.lower(text: string) -> string`
  - `{"let": {"name": "lower", "type": "string", "value": {"expr": "text.lower('ABC')"}}}`
- `text.upper(text: string) -> string`
  - `{"let": {"name": "upper", "type": "string", "value": {"expr": "text.upper('abc')"}}}`
- `text.trim(text: string) -> string`
  - `{"let": {"name": "clean", "type": "string", "value": {"expr": "text.trim('  ok  ')"}}}`
- `text.replace(text: string, old: string, new: string) -> string`
  - `{"let": {"name": "rep", "type": "string", "value": {"expr": "text.replace('a-b', '-', ':')"}}}`
- `text.split(text: string, sep: string) -> array`
  - `{"let": {"name": "parts", "type": "array", "value": {"expr": "text.split('a,b', ',')"}}}`
- `text.regex_match(text: string, pattern: string, flags?: string) -> bool`
  - `{"let": {"name": "ok", "type": "bool", "value": {"expr": "text.regex_match('abc', 'a.+')"}}}`
- `text.regex_findall(text: string, pattern: string, flags?: string) -> array`
  - `{"let": {"name": "hits", "type": "array", "value": {"expr": "text.regex_findall('a1 a2', 'a\\d+')"}}}`

### math.*
- В namespace `math` нет одиночных функций: используются автогенерируемые функции `math.add_N`, `math.sub_N`, `math.mul_N`, `math.div_N`.
  - `{"let": {"name": "x", "type": "number", "value": {"expr": "math.add_5(10)"}}}`

### command.*
- `command.args(raw_text?: string) -> array`
  - `{"let": {"name": "args", "type": "array", "value": {"expr": "command.args()"}}}`
- `command.arg(index: number, default?: any, raw_text?: string) -> any`
  - `{"let": {"name": "first", "type": "any", "value": {"expr": "command.arg(0, null)"}}}`
- `command.parse_amount(text: string, allow_all?: bool) -> object`
  - `{"let": {"name": "parsed", "type": "object", "value": {"expr": "command.parse_amount('10k')"}}}`
- `command.parse_user(text: string) -> string|null`
  - `{"let": {"name": "uid", "type": "string", "value": {"expr": "command.parse_user('@user')"}}}`

### user.*
- `user.id(ctx?: any) -> string`
  - `{"let": {"name": "uid", "type": "string", "value": {"expr": "user.id(ctx)"}}}`
- `user.name(user_id: string) -> string`
  - `{"let": {"name": "uname", "type": "string", "value": {"expr": "user.name(user.id(ctx))"}}}`
- `user.mention(user_id: string) -> string`
  - `{"let": {"name": "mention", "type": "string", "value": {"expr": "user.mention(user.id(ctx))"}}}`
- `user.role(user_id: string) -> string`
  - `{"let": {"name": "role", "type": "string", "value": {"expr": "user.role(user.id(ctx))"}}}`

### chat.*
- `chat.id(ctx?: any) -> string`
  - `{"let": {"name": "cid", "type": "string", "value": {"expr": "chat.id(ctx)"}}}`
- `chat.title(ctx?: any) -> string`
  - `{"let": {"name": "title", "type": "string", "value": {"expr": "chat.title(ctx)"}}}`

### permissions.*
- `permissions.has(user_id: string, perm: string) -> bool`
  - `{"let": {"name": "allowed", "type": "bool", "value": {"expr": "permissions.has(user.id(ctx), 'admin')"}}}`

### members.*
- `members.list() -> array`
  - `{"let": {"name": "members", "type": "array", "value": {"expr": "members.list()"}}}`
- `members.random(count?: number) -> array`
  - `{"let": {"name": "pick", "type": "array", "value": {"expr": "members.random(3)"}}}`

### ui.*
- `ui.form(fields: array, submit?: string) -> string`
  - `{"let": {"name": "form_id", "type": "string", "value": {"expr": "ui.form([{"name":"age","label":"Возраст","type":"number"}], 'send')"}}}`
- `ui.form_submit(form_id: string) -> object`
  - `{"let": {"name": "payload", "type": "object", "value": {"expr": "ui.form_submit(form_id)"}}}`

### message.*
- `message.send(chat_id?: string, text?: string, buttons?: array, reply_to?: string) -> string`
  - `{"call": "message.send", "args": [{"expr": "chat.id(ctx)"}, "Привет!"]}`
- `message.edit(message_id: string, text: string, buttons?: array) -> bool`
  - `{"call": "message.edit", "args": ["123", "Обновлено"]}`
- `message.delete(message_id: string) -> bool`
  - `{"call": "message.delete", "args": ["123"]}`
- `message.pin(message_id: string, chat_id?: string, notify?: bool) -> bool`
  - `{"call": "message.pin", "args": ["123", {"expr": "chat.id(ctx)"}, false]}`
- `message.unpin(message_id: string, chat_id?: string) -> bool`
  - `{"call": "message.unpin", "args": ["123", {"expr": "chat.id(ctx)"}]}`

### pin.*
- `pin.set(value: string) -> bool`
  - `{"call": "pin.set", "args": ["state"]}`
- `pin.get(default?: string) -> string`
  - `{"let": {"name": "state", "type": "string", "value": {"expr": "pin.get('none')"}}}`
- `pin.clear() -> bool`
  - `{"call": "pin.clear"}`

### thread.*
- `thread.storage(key: string, value?: any) -> any`
  - `{"call": "thread.storage", "args": ["ticket", 42]}`
- `thread.storage_get(key: string, default?: any) -> any`
  - `{"let": {"name": "ticket", "type": "any", "value": {"expr": "thread.storage_get('ticket', null)"}}}`
- `thread.storage_del(key: string) -> bool`
  - `{"call": "thread.storage_del", "args": ["ticket"]}`

### time.*
- `time.now() -> number` (unix-seconds)
  - `{"let": {"name": "now", "type": "number", "value": {"expr": "time.now()"}}}`
- `time.today() -> string` (YYYY-MM-DD)
  - `{"let": {"name": "today", "type": "string", "value": {"expr": "time.today()"}}}`
- `time.parse(text: string, tz?: string) -> number`
  - `{"let": {"name": "ts", "type": "number", "value": {"expr": "time.parse('2024-01-01')"}}}`
- `time.format(ts: number, pattern?: string, tz?: string) -> string`
  - `{"let": {"name": "fmt", "type": "string", "value": {"expr": "time.format(time.now(), '%Y-%m-%d')"}}}`
- `time.add_days(ts: number, days: number) -> number`
  - `{"let": {"name": "tom", "type": "number", "value": {"expr": "time.add_days(time.now(), 1)"}}}`
- `time.diff_days(a: number, b: number) -> number`
  - `{"let": {"name": "diff", "type": "number", "value": {"expr": "time.diff_days(time.now(), time.add_days(time.now(), 3))"}}}`

### schedule.*
- `schedule.once(name: string, at: number, payload?: any) -> bool`
  - `{"call": "schedule.once", "args": ["daily_bonus", {"expr": "time.add_days(time.now(), 1)"}]}`
- `schedule.every(name: string, period: number, payload?: any) -> bool`
  - `{"call": "schedule.every", "args": ["tick", 3600]}`
- `schedule.cancel(name: string) -> bool`
  - `{"call": "schedule.cancel", "args": ["tick"]}`
- `schedule.list(prefix?: string) -> array`
  - `{"let": {"name": "jobs", "type": "array", "value": {"expr": "schedule.list('tick')"}}}`

### balance.*
- `balance.get(user_id?: string) -> number`
  - `{"let": {"name": "bal", "type": "number", "value": {"expr": "balance.get(user.id(ctx))"}}}`
- `balance.add(user_id: string, amount: number, reason?: string) -> number`
  - `{"call": "balance.add", "args": [{"expr": "user.id(ctx)"}, 10, "bonus"]}`
- `balance.remove(user_id: string, amount: number, reason?: string) -> number`
  - `{"call": "balance.remove", "args": [{"expr": "user.id(ctx)"}, 5, "fee"]}`
- `balance.transfer_atomic(from_id: string, to_id: string, amount: number, reason?: string) -> bool`
  - `{"call": "balance.transfer_atomic", "args": ["1", "2", 100]}`
- `balance.can_remove(user_id: string, amount: number) -> bool`
  - `{"let": {"name": "ok", "type": "bool", "value": {"expr": "balance.can_remove(user.id(ctx), 50)"}}}`

### economy.*
- `economy.lock(key?: string, ttl?: number) -> bool`
  - `{"call": "economy.lock", "args": ["bank", 60]}`
- `economy.unlock(key?: string) -> bool`
  - `{"call": "economy.unlock", "args": ["bank"]}`

### treasury.*
- `treasury.get() -> number`
  - `{"let": {"name": "treasury", "type": "number", "value": {"expr": "treasury.get()"}}}`
- `treasury.add(amount: number, reason?: string) -> number`
  - `{"call": "treasury.add", "args": [50, "donation"]}`
- `treasury.remove(amount: number, reason?: string) -> number`
  - `{"call": "treasury.remove", "args": [25, "payout"]}`

### kv.*
- `kv.incr(key: string, delta?: number, ttl?: number) -> number`
  - `{"let": {"name": "cnt", "type": "number", "value": {"expr": "kv.incr('hits', 1, 3600)"}}}`
- `kv.list(prefix: string, limit?: number, cursor?: string) -> object`
  - `{"let": {"name": "list", "type": "object", "value": {"expr": "kv.list('hits')"}}}`
- `kv.set_ttl(key: string, ttl: number) -> bool`
  - `{"call": "kv.set_ttl", "args": ["hits", 600]}`
- `kv.get_meta(key: string) -> object`
  - `{"let": {"name": "meta", "type": "object", "value": {"expr": "kv.get_meta('hits')"}}}`

### persist.*
- `persist.incr(user_id: string, key: string, delta?: number) -> number`
  - `{"let": {"name": "lvl", "type": "number", "value": {"expr": "persist.incr(user.id(ctx), 'lvl', 1)"}}}`
- `persist.list(user_id: string, prefix: string, limit?: number, cursor?: string) -> object`
  - `{"let": {"name": "plist", "type": "object", "value": {"expr": "persist.list(user.id(ctx), 'lvl')"}}}`

### db.*
- `db.transaction(actions: array) -> bool`
  - `{"call": "db.transaction", "args": [[{"call": "balance.add", "args": ["1", 10]}]]}`

### http.*
- `http.get(url: string, headers?: object) -> object`
  - `{"let": {"name": "resp", "type": "object", "value": {"expr": "http.get('https://example.com')"}}}`
- `http.post(url: string, data?: any, headers?: object) -> object`
  - `{"let": {"name": "resp", "type": "object", "value": {"expr": "http.post('https://example.com', {"name": "test"})"}}}`
- `http.request(method: string, url: string, data?: any, headers?: object) -> object`
  - `{"let": {"name": "resp", "type": "object", "value": {"expr": "http.request('PUT', 'https://example.com', {"ok": true})"}}}`

### crypto.*
- `crypto.sha256(text: string) -> string`
  - `{"let": {"name": "hash", "type": "string", "value": {"expr": "crypto.sha256('data')"}}}`
- `crypto.hmac_sha256(key: string, text: string) -> string`
  - `{"let": {"name": "hmac", "type": "string", "value": {"expr": "crypto.hmac_sha256('k', 'data')"}}}`

### webhook.*
- `webhook.emit(name: string, payload?: any) -> bool`
  - `{"call": "webhook.emit", "args": ["event", {"value": 1}]}`

### config.*
- `config.get(section: string, key: string, default?: any) -> any`
  - `{"let": {"name": "val", "type": "any", "value": {"expr": "config.get('prefix', 'list', [])"}}}`
- `config.set(section: string, key: string, value: any) -> bool`
  - `{"call": "config.set", "args": ["prefix", "list", ["!!", "!"]]}`
- `config.del(section: string, key: string) -> bool`
  - `{"call": "config.del", "args": ["prefix", "list"]}`
- `config.list(section: string, prefix?: string) -> array`
  - `{"let": {"name": "keys", "type": "array", "value": {"expr": "config.list('prefix')"}}}`

### alias.*
- `alias.set(command: string, alias: string) -> bool`
  - `{"call": "alias.set", "args": ["balance", "bal"]}`
- `alias.get(alias: string) -> string|null`
  - `{"let": {"name": "cmd", "type": "string", "value": {"expr": "alias.get('bal')"}}}`
- `alias.list() -> array`
  - `{"let": {"name": "alist", "type": "array", "value": {"expr": "alias.list()"}}}`

### role.*
- `role.set(user_id: string, role: string) -> bool`
  - `{"call": "role.set", "args": [{"expr": "user.id(ctx)"}, "moderator"]}`
- `role.get(user_id: string) -> string`
  - `{"let": {"name": "role", "type": "string", "value": {"expr": "role.get(user.id(ctx))"}}}`
- `role.list() -> array`
  - `{"let": {"name": "roles", "type": "array", "value": {"expr": "role.list()"}}}`

### locale.*
- `locale.set(value: string) -> bool`
  - `{"call": "locale.set", "args": ["ru"]}`
- `locale.get(default?: string) -> string`
  - `{"let": {"name": "loc", "type": "string", "value": {"expr": "locale.get('ru')"}}}`

### currency.*
- `currency.set(value: object) -> bool`
  - `{"call": "currency.set", "args": [{"name": "Монеты", "icon": "🪙"}]}`
- `currency.get() -> object`
  - `{"let": {"name": "curr", "type": "object", "value": {"expr": "currency.get()"}}}`

### income.*
- `income.source_set(name: string, value: object) -> bool`
  - `{"call": "income.source_set", "args": ["daily", {"amount": 10}]}`
- `income.source_get(name: string) -> object|null`
  - `{"let": {"name": "inc", "type": "object", "value": {"expr": "income.source_get('daily')"}}}`

### reward.*
- `reward.formula_set(name: string, formula: string) -> bool`
  - `{"call": "reward.formula_set", "args": ["daily", "base*1.2"]}`
- `reward.formula_get(name: string) -> string|null`
  - `{"let": {"name": "formula", "type": "string", "value": {"expr": "reward.formula_get('daily')"}}}`

### cap.*
- `cap.set(name: string, value: number) -> bool`
  - `{"call": "cap.set", "args": ["balance", 100000]}`
- `cap.get(name: string, default?: number) -> number`
  - `{"let": {"name": "cap", "type": "number", "value": {"expr": "cap.get('balance', 0)"}}}`

### cooldown.*
- `cooldown.set(key: string, seconds: number) -> bool`
  - `{"call": "cooldown.set", "args": ["daily", 86400]}`
- `cooldown.check(key: string) -> bool`
  - `{"let": {"name": "ready", "type": "bool", "value": {"expr": "cooldown.check('daily')"}}}`
- `cooldown.remaining(key: string) -> number`
  - `{"let": {"name": "left", "type": "number", "value": {"expr": "cooldown.remaining('daily')"}}}`
- `cooldown.clear(key: string) -> bool`
  - `{"call": "cooldown.clear", "args": ["daily"]}`

### ratelimit.*
- `ratelimit.hit(key: string, limit: number, period: number) -> bool`
  - `{"let": {"name": "ok", "type": "bool", "value": {"expr": "ratelimit.hit('spam', 5, 60)"}}}`
- `ratelimit.remaining(key: string) -> number`
  - `{"let": {"name": "rem", "type": "number", "value": {"expr": "ratelimit.remaining('spam')"}}}`

### tax.*
- `tax.set(value: number) -> bool`
  - `{"call": "tax.set", "args": [5]}`
- `tax.get(default?: number) -> number`
  - `{"let": {"name": "tax", "type": "number", "value": {"expr": "tax.get(0)"}}}`

### shop.*
- `shop.item_set(code: string, value: object) -> bool`
  - `{"call": "shop.item_set", "args": ["vip", {"price": 1000}]}`
- `shop.item_get(code: string) -> object|null`
  - `{"let": {"name": "item", "type": "object", "value": {"expr": "shop.item_get('vip')"}}}`
- `shop.item_del(code: string) -> bool`
  - `{"call": "shop.item_del", "args": ["vip"]}`
- `shop.item_list(prefix?: string) -> array`
  - `{"let": {"name": "items", "type": "array", "value": {"expr": "shop.item_list('v')"}}}`

### lootbox.*
- `lootbox.set(code: string, value: object) -> bool`
  - `{"call": "lootbox.set", "args": ["box1", {"rewards": [1,2]}]}`
- `lootbox.get(code: string) -> object|null`
  - `{"let": {"name": "box", "type": "object", "value": {"expr": "lootbox.get('box1')"}}}`

### transfer.*
- `transfer.rules_set(value: object) -> bool`
  - `{"call": "transfer.rules_set", "args": [{"tax": 2}]}`
- `transfer.rules_get() -> object`
  - `{"let": {"name": "rules", "type": "object", "value": {"expr": "transfer.rules_get()"}}}`

### antifarm.*
- `antifarm.set(value: object) -> bool`
  - `{"call": "antifarm.set", "args": [{"min_delay": 5}]}`
- `antifarm.get() -> object`
  - `{"let": {"name": "anti", "type": "object", "value": {"expr": "antifarm.get()"}}}`

### feature.*
- `feature.set(name: string, value: bool) -> bool`
  - `{"call": "feature.set", "args": ["market", true]}`
- `feature.get(name: string, default?: bool) -> bool`
  - `{"let": {"name": "enabled", "type": "bool", "value": {"expr": "feature.get('market', false)"}}}`

### audit.*
- `audit.log(event: string, payload?: object) -> bool`
  - `{"call": "audit.log", "args": ["purchase", {"item": "vip"}]}`

### metric.*
- `metric.inc(name: string, value?: number) -> bool`
  - `{"call": "metric.inc", "args": ["views", 1]}`

### metrics.*
- `metrics.incr(name: string, value?: number) -> bool`
  - `{"call": "metrics.incr", "args": ["views", 1]}`

### Глобальные функции (без префикса)
- `active_chat_id() -> number`
  - `{"let": {"name": "cid", "type": "number", "value": {"expr": "active_chat_id()"}}}`
- `assert_(cond: bool, message?: string) -> bool`
  - `{"call": "assert_", "args": [{"expr": "balance.get(user.id(ctx)) > 0"}, "Баланс пуст"]}`
- `log(value: any) -> bool`
  - `{"call": "log", "args": ["debug"]}`
- `error_last() -> object`
  - `{"let": {"name": "err", "type": "object", "value": {"expr": "error_last()"}}}`
- `prefix_get(default?: array) -> array`
  - `{"let": {"name": "pref", "type": "array", "value": {"expr": "prefix_get(['!!'])"}}}`
- `prefix_set(prefixes: array) -> bool`
  - `{"call": "prefix_set", "args": [["!!", "!"]]}`
- `rate_limit(key: string, limit: number, period: number) -> bool`
  - `{"let": {"name": "ok", "type": "bool", "value": {"expr": "rate_limit('msg', 3, 60)"}}}`
- `start_balance_get(default?: number) -> number`
  - `{"let": {"name": "start", "type": "number", "value": {"expr": "start_balance_get(0)"}}}`
- `start_balance_set(value: number) -> bool`
  - `{"call": "start_balance_set", "args": [100]}`
- `timezone_get(default?: string) -> string`
  - `{"let": {"name": "tz", "type": "string", "value": {"expr": "timezone_get('Europe/Moscow')"}}}`
- `timezone_set(value: string) -> bool`
  - `{"call": "timezone_set", "args": ["Europe/Moscow"]}`
- `day_boundary_get(default?: number) -> number`
  - `{"let": {"name": "hour", "type": "number", "value": {"expr": "day_boundary_get(0)"}}}`
- `day_boundary_set(value: number) -> bool`
  - `{"call": "day_boundary_set", "args": [4]}`
- `to_bool(value: any) -> bool`
  - `{"let": {"name": "flag", "type": "bool", "value": {"expr": "to_bool('true')"}}}`
- `to_number(value: any) -> number`
  - `{"let": {"name": "num", "type": "number", "value": {"expr": "to_number('12.5')"}}}`
- `to_string(value: any) -> string`
  - `{"let": {"name": "txt", "type": "string", "value": {"expr": "to_string(100)"}}}`
- `type_of(value: any) -> string`
  - `{"let": {"name": "t", "type": "string", "value": {"expr": "type_of([1,2])"}}}`
- `var_exists(name: string) -> bool`
  - `{"let": {"name": "ok", "type": "bool", "value": {"expr": "var_exists('x')"}}}`
- `var_unset(name: string) -> bool`
  - `{"call": "var_unset", "args": ["x"]}`
- `break_loop()` / `continue_loop()` / `try_catch(...)` — служебные функции для движка.

## 10. Каталог автогенерируемых функций (300+)
Ниже — **300 имён функций**, полностью валидных и рабочих (часть из 1000 доступных):
```
math.add_1
math.add_2
math.add_3
math.add_4
math.add_5
math.add_6
math.add_7
math.add_8
math.add_9
math.add_10
math.add_11
math.add_12
math.add_13
math.add_14
math.add_15
math.add_16
math.add_17
math.add_18
math.add_19
math.add_20
math.add_21
math.add_22
math.add_23
math.add_24
math.add_25
math.add_26
math.add_27
math.add_28
math.add_29
math.add_30
math.add_31
math.add_32
math.add_33
math.add_34
math.add_35
math.add_36
math.add_37
math.add_38
math.add_39
math.add_40
math.add_41
math.add_42
math.add_43
math.add_44
math.add_45
math.add_46
math.add_47
math.add_48
math.add_49
math.add_50
math.add_51
math.add_52
math.add_53
math.add_54
math.add_55
math.add_56
math.add_57
math.add_58
math.add_59
math.add_60
math.add_61
math.add_62
math.add_63
math.add_64
math.add_65
math.add_66
math.add_67
math.add_68
math.add_69
math.add_70
math.add_71
math.add_72
math.add_73
math.add_74
math.add_75
math.add_76
math.add_77
math.add_78
math.add_79
math.add_80
math.add_81
math.add_82
math.add_83
math.add_84
math.add_85
math.add_86
math.add_87
math.add_88
math.add_89
math.add_90
math.add_91
math.add_92
math.add_93
math.add_94
math.add_95
math.add_96
math.add_97
math.add_98
math.add_99
math.add_100
math.sub_1
math.sub_2
math.sub_3
math.sub_4
math.sub_5
math.sub_6
math.sub_7
math.sub_8
math.sub_9
math.sub_10
math.sub_11
math.sub_12
math.sub_13
math.sub_14
math.sub_15
math.sub_16
math.sub_17
math.sub_18
math.sub_19
math.sub_20
math.sub_21
math.sub_22
math.sub_23
math.sub_24
math.sub_25
math.sub_26
math.sub_27
math.sub_28
math.sub_29
math.sub_30
math.sub_31
math.sub_32
math.sub_33
math.sub_34
math.sub_35
math.sub_36
math.sub_37
math.sub_38
math.sub_39
math.sub_40
math.sub_41
math.sub_42
math.sub_43
math.sub_44
math.sub_45
math.sub_46
math.sub_47
math.sub_48
math.sub_49
math.sub_50
math.sub_51
math.sub_52
math.sub_53
math.sub_54
math.sub_55
math.sub_56
math.sub_57
math.sub_58
math.sub_59
math.sub_60
math.sub_61
math.sub_62
math.sub_63
math.sub_64
math.sub_65
math.sub_66
math.sub_67
math.sub_68
math.sub_69
math.sub_70
math.sub_71
math.sub_72
math.sub_73
math.sub_74
math.sub_75
math.sub_76
math.sub_77
math.sub_78
math.sub_79
math.sub_80
math.sub_81
math.sub_82
math.sub_83
math.sub_84
math.sub_85
math.sub_86
math.sub_87
math.sub_88
math.sub_89
math.sub_90
math.sub_91
math.sub_92
math.sub_93
math.sub_94
math.sub_95
math.sub_96
math.sub_97
math.sub_98
math.sub_99
math.sub_100
math.mul_1
math.mul_2
math.mul_3
math.mul_4
math.mul_5
math.mul_6
math.mul_7
math.mul_8
math.mul_9
math.mul_10
math.mul_11
math.mul_12
math.mul_13
math.mul_14
math.mul_15
math.mul_16
math.mul_17
math.mul_18
math.mul_19
math.mul_20
math.mul_21
math.mul_22
math.mul_23
math.mul_24
math.mul_25
math.mul_26
math.mul_27
math.mul_28
math.mul_29
math.mul_30
math.mul_31
math.mul_32
math.mul_33
math.mul_34
math.mul_35
math.mul_36
math.mul_37
math.mul_38
math.mul_39
math.mul_40
math.mul_41
math.mul_42
math.mul_43
math.mul_44
math.mul_45
math.mul_46
math.mul_47
math.mul_48
math.mul_49
math.mul_50
math.mul_51
math.mul_52
math.mul_53
math.mul_54
math.mul_55
math.mul_56
math.mul_57
math.mul_58
math.mul_59
math.mul_60
math.mul_61
math.mul_62
math.mul_63
math.mul_64
math.mul_65
math.mul_66
math.mul_67
math.mul_68
math.mul_69
math.mul_70
math.mul_71
math.mul_72
math.mul_73
math.mul_74
math.mul_75
math.mul_76
math.mul_77
math.mul_78
math.mul_79
math.mul_80
math.mul_81
math.mul_82
math.mul_83
math.mul_84
math.mul_85
math.mul_86
math.mul_87
math.mul_88
math.mul_89
math.mul_90
math.mul_91
math.mul_92
math.mul_93
math.mul_94
math.mul_95
math.mul_96
math.mul_97
math.mul_98
math.mul_99
math.mul_100
```

## 11. Объёмный пример скрипта
```json
{
  "imports": ["text", "json"],
  "consts": {"daily_bonus": 25, "tax": 5},
  "functions": [
    {
      "name": "calc_bonus",
      "params": [{"name": "base", "type": "number"}],
      "returns": "number",
      "actions": [
        {"let": {"name": "bonus", "type": "number", "value": {"expr": "base + daily_bonus"}}},
        {"return": {"expr": "bonus"}}
      ]
    }
  ],
  "commands": [
    {
      "name": "бонус",
      "description": "Выдать разовый бонус",
      "actions": [
        {"let": {"name": "uid", "type": "string", "value": {"expr": "user.id(ctx)"}}},
        {"let": {"name": "bonus", "type": "number", "value": {"expr": "calc_bonus(10)"}}},
        {"call": "balance.add", "args": [{"expr": "uid"}, {"expr": "bonus"}, "bonus"]},
        {"call": "message.send", "args": [{"expr": "chat.id(ctx)"}, {"expr": "text.format('✅ {name}, +{bonus}', {\"name\": user.name(uid), \"bonus\": bonus})"}]}
      ]
    },
    {
      "name": "баланс игрока",
      "description": "Показать баланс",
      "actions": [
        {"let": {"name": "uid", "type": "string", "value": {"expr": "command.parse_user(command.arg(0, ''))"}}},
        {"if": "uid == null", "then": [{"set": {"name": "uid", "value": {"expr": "user.id(ctx)"}}}]},
        {"let": {"name": "bal", "type": "number", "value": {"expr": "balance.get(uid)"}}},
        {"call": "message.send", "args": [{"expr": "chat.id(ctx)"}, {"expr": "text.format('💰 {name}: {bal}', {\"name\": user.name(uid), \"bal\": bal})"}]}
      ]
    }
  ],
  "events": [
    {
      "name": "daily",
      "actions": [
        {"let": {"name": "members", "type": "array", "value": {"expr": "members.list()"}}},
        {"for": {"var": "u", "in": "members"},
         "body": [
           {"call": "balance.add", "args": [{"expr": "u"}, {"expr": "daily_bonus"}, "daily"]}
         ]
        }
      ]
    }
  ]
}
```
