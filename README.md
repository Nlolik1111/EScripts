# EScripting - кастомные скрипты/аддоны для групп

EScripting - система расширения бота через JSON-скрипты. Скрипты позволяют создавать свои команды, события, кнопки, розыгрыши, автоматические ежедневные ивенты и админ-экшены (закрепление, кик, мут). Действия выполняются строго по порядку, ровно в том виде, как описаны в скрипте. Доступно только на плане **Ультиматум**.

## Быстрый старт
1. В группе: `!!скрипты добавить "Мой скрипт"`.
2. Бот попросит файл `.escript.json`. Ответьте на сообщение файлом.
3. Бот выполнит компиляцию, покажет прогресс и результат. При успехе скрипт активируется.

## Команды EScripting
- `!!скрипты`: список скриптов со статусами (🟢/🔴) и сводка.
  - Нажатие на кнопку скрипта переключает статус.
- `!!скрипты инфо`: сводка + меню со статусами.
- `!!скрипты настроить`: меню настроек скриптов (кнопки без статуса).
  - Нажатие открывает карточку скрипта:
    - Статус
    - Ошибки за сутки
    - Автор
    - Кнопки: **Код**, **Изменить**, **Удалить**
- `!!скрипты настроить "Название"`: сразу открыть настройки.
- `!!скрипты добавить "Название"`: добавить скрипт (только администраторы экономики).

## Формат файла `.escript.json`
```json
{
  "name": "Лотерея дня",
  "timezone": "UTC+3",
  "commands": [
    {
      "name": "лотерея",
      "description": "Ежедневная розыгрыш-команда",
      "permissions": "all",
      "override": false,
      "target_mode": "optional",
      "actions": [
        {"type": "reply", "text": "🎟️ Запускаю розыгрыш..."},
        {"type": "random_pick", "count": 3, "exclude_sender": true, "var": "winners"},
        {"type": "add_balance", "target": "var:winners", "amount": {"min": 2000, "max": 5000}},
        {"type": "send", "text": "Победители: {winners}"}
      ]
    }
  ],
  "events": {
    "daily": [
      {
        "at": "09:00",
        "tz": "UTC+3",
        "actions": [
          {"type": "send", "text": "🌞 Доброе утро! Ежедневная акция включена."}
        ]
      }
    ]
  }
}
```

## Ключевые поля
| Поле | Тип | Описание |
|------|-----|----------|
| `name` | string | Название скрипта. |
| `timezone` | string | Часовой пояс скрипта (например, `UTC+3`). |
| `commands` | array | Команды скрипта (описаны ниже). |
| `events.daily` | array | Список ежедневных событий по времени `HH:MM`. |

## Команды скрипта (`commands`)
Каждая команда описывает кастомную команду, которая будет доступна в чате:
```json
{
  "name": "команда",
  "description": "Описание для !!помощь",
  "permissions": "all|moderator|admin",
  "access_level": 0,
  "override": false,
  "target_mode": "optional|required|none",
  "actions": [ ... ]
}
```

**Пояснения:**
- `permissions` - кто может вызывать команду (all/moderator/admin).
- `access_level` - уровень доступа по ролям: 0 (все), 1 (модератор+), 2 (админ+), 3 (только создатель). Опционально.
- `override` - если `true`, команда перехватывает одноимённую базовую команду бота.
- `target_mode` - управление целью (`@user`):
  - `none` - цель не нужна
  - `optional` - цель опционально
  - `required` - цель обязательна

## Действия (`actions`)
Скриптовые действия выполняются строго по порядку.

### reply/send
Отправить сообщение (reply - ответом, send - в чат).
```json
{"type": "reply", "text": "Текст"}
{"type": "send", "text": "Текст"}
```

**Кнопки**
```json
{
  "type": "reply",
  "text": "Выберите вариант",
  "buttons": [
    {"label": "Да", "actions": [{"type": "send", "text": "Подтверждено"}]},
    {"label": "Нет", "actions": [{"type": "send", "text": "Отменено"}]}
  ]
}
```

### add_balance / remove_balance
```json
{"type": "add_balance", "target": "sender", "amount": 3000}
{"type": "remove_balance", "target": "target", "amount": {"min": 1000, "max": 2000}}
```
`target`:
- `sender` - отправитель
- `target` - упомянутый пользователь
- `var:имя` - список целей из переменной

### transfer
Перевод между пользователями.
```json
{"type": "transfer", "receiver": "target", "amount": 2500}
```

### call (стандартная библиотека 1000+ функций)
Выполняет встроенную функцию. Все действия выполняются строго в порядке скрипта.
```json
{"type": "call", "func": "math.add_10", "args": [{"var": "score"}], "out": "score"}
```

**Функции-диапазоны (1000+):**
- `math.add_1..math.add_200` - прибавить константу N к первому аргументу.
- `math.sub_1..math.sub_200` - вычесть константу N из первого аргумента.
- `math.mul_1..math.mul_200` - умножить первый аргумент на N.
- `math.div_1..math.div_200` - разделить первый аргумент на N.
- `text.repeat_1..text.repeat_200` - повторить строку N раз.

Итого: **1000+** функций, все работают по одной простой схеме: функция принимает массив `args`,
использует первый аргумент и возвращает результат. При необходимости сохраняйте результат в `out`.

### add_treasury / remove_treasury
Работа с казной группы.
```json
{"type": "add_treasury", "amount": 5000}
{"type": "remove_treasury", "amount": 3000}
```

### random_pick
Выбирает случайных участников чата.
```json
{"type": "random_pick", "count": 3, "exclude_sender": true, "var": "winners"}
```

### chance
Выполняет случайную ветку.
```json
{
  "type": "chance",
  "percent": 50,
  "then": [{"type": "send", "text": "Повезло!"}],
  "else": [{"type": "send", "text": "Не повезло."}]
}
```

### set
Сохраняет значение в переменную.
```json
{"type": "set", "var": "myvar", "value": "123"}
```

### math
Математика над переменными.
```json
{"type": "math", "var": "score", "op": "+", "value": 10}
{"type": "math", "var": "score", "op": "*", "value": {"var": "multiplier"}}
```

### if
Условные ветки.
```json
{
  "type": "if",
  "condition": {"left": {"var": "score"}, "op": "gte", "right": 10},
  "then": [{"type": "send", "text": "Уровень: профи"}],
  "else": [{"type": "send", "text": "Уровень: новичок"}]
}
```
Операторы условий:
`equals`, `not_equals`, `gt`, `lt`, `gte`, `lte`, `contains`, `role_is`.

### cooldown
Ограничивает повторения команды.
```json
{
  "type": "cooldown",
  "key": "promo",
  "seconds": 3600,
  "per": "user",
  "on_cooldown": [{"type": "reply", "text": "Подожди час."}]
}
```

### retry
Сбрасывает кулдаун команды для отправителя (если ввод был ошибочным).
```json
{"type": "retry"}
```

### kv_set / kv_get
KV-хранилище скрипта (глобально для чата).
```json
{"type": "kv_set", "key": "phase", "value": "stage1"}
{"type": "kv_get", "key": "phase", "out": "phase_val"}
```

### persist_set / persist_get
Персистентные переменные для конкретного игрока.
```json
{"type": "persist_set", "key": "wins", "value": 3}
{"type": "persist_get", "key": "wins", "out": "wins_val"}
```

### pin_info
Получить message_id текущего закрепа (если есть).
```json
{"type": "pin_info", "out": "pinned_id"}
```

### stop
Останавливает выполнение цепочки.
```json
{"type": "stop"}
```

### pin
Отправляет и закрепляет сообщение (если у бота есть права).
```json
{"type": "pin", "text": "Важное сообщение"}
```

### unpin
Снять все закрепления.
```json
{"type": "unpin"}
```

### kick / mute / unmute
Админ-действия (бот должен быть администратором).
```json
{"type": "kick", "target": "target"}
{"type": "mute", "target": "target"}
{"type": "unmute", "target": "target"}
```

### ban_chat / unban_chat
Блокировка пользователя в чате (бот должен быть администратором).
```json
{"type": "ban_chat", "target": "target", "seconds": 3600}
{"type": "unban_chat", "target": "target"}
```

### ban_economy / unban_economy
Экономический бан (запрет команд экономки).
```json
{"type": "ban_economy", "target": "target"}
{"type": "unban_economy", "target": "target"}
```

### set_role
Меняет роль пользователя экономики.
```json
{"type": "set_role", "target": "target", "role": "moderator"}
```

### edit / delete_message
Редактировать текущее сообщение или удалить его.
```json
{"type": "edit", "text": "Обновлённый текст"}
{"type": "delete_message"}
```

## Переменные в тексте
В тексте можно использовать плейсхолдеры:
- `{user}` - имя отправителя
- `{target}` - имя цели
- `{winners}` - список победителей (из random_pick)
- `{sender_role}` - роль отправителя (default/moderator/admin/banned)

## Ошибки и предупреждения
Компиляция пишет лог:
- **Ошибки** блокируют загрузку.
- **Предупреждения** не блокируют, но могут вызвать проблемы.

Лог можно получить кнопкой **«Получить лог»** после компиляции.

## Пример: команда-розыгрыш
```json
{
  "name": "Розыгрыши",
  "commands": [
    {
      "name": "розыгрыш",
      "description": "Раздать призы 2 участникам",
      "permissions": "moderator",
      "actions": [
        {"type": "reply", "text": "🎉 Розыгрыш стартует!"},
        {"type": "random_pick", "count": 2, "exclude_sender": true, "var": "winners"},
        {"type": "add_balance", "target": "var:winners", "amount": {"min": 2500, "max": 6000}},
        {"type": "send", "text": "Победители: {winners}"}
      ]
    }
  ]
}
```

## Пример: ежедневный ивент
```json
{
  "name": "Дневной бонус",
  "events": {
    "daily": [
      {
        "at": "12:00",
        "actions": [
          {"type": "send", "text": "☀️ Ежедневная активность активна!"},
          {"type": "random_pick", "count": 3, "var": "winners"},
          {"type": "add_balance", "target": "var:winners", "amount": 3000},
          {"type": "send", "text": "Бонусы получили: {winners}"}
        ]
      }
    ]
  }
}
```
