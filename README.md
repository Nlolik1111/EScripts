# E‑Scripting JavaScript — Полный практический справочник

> Документ ориентирован на **практику**: как работают реальные функции (`sendMessage`, `pinMessage`, `editMessage`, экономика/банк), какие лимиты действуют и как правильно строить сценарии.

---

## Оглавление

- [1. Что такое JS e-scripting](#1-что-такое-js-e-scripting)
- [2. Модель безопасности](#2-модель-безопасности)
- [3. Загрузка скриптов и структура файла](#3-загрузка-скриптов-и-структура-файла)
- [4. Импорты: что разрешено и что запрещено](#4-импорты-что-разрешено-и-что-запрещено)
- [5. Контекст `ctx`](#5-контекст-ctx)
- [6. Telegram API (подробно)](#6-telegram-api-подробно)
  - [6.1 sendMessage](#61-sendmessage)
  - [6.2 reply](#62-reply)
  - [6.3 editMessage](#63-editmessage)
  - [6.4 deleteMessage](#64-deletemessage)
  - [6.5 pinMessage / unpinMessage](#65-pinmessage--unpinmessage)
  - [6.6 sendPrivateMessage](#66-sendprivatemessage)
  - [6.7 react](#67-react)
  - [6.8 button / inlineKeyboard](#68-button--inlinekeyboard)
- [7. Economy API (подробно)](#7-economy-api-подробно)
  - [7.1 getWallet / getBalance / getUserEconomy](#71-getwallet--getbalance--getusereconomy)
  - [7.2 credit / debit / transfer](#72-credit--debit--transfer)
  - [7.3 addBalance / subtractBalance](#73-addbalance--subtractbalance)
- [8. Bank API (подробно)](#8-bank-api-подробно)
  - [8.1 getBankBalance](#81-getbankbalance)
  - [8.2 deposit](#82-deposit)
  - [8.3 withdraw](#83-withdraw)
- [9. Stats / Time / Random / Format / Policy / i18n / Logger / Flags](#9-stats--time--random--format--policy--i18n--logger--flags)
- [10. Мульти-командные скрипты (до 10)](#10-мульти-командные-скрипты-до-10)
- [11. Префиксы команд: `!!`, `/`, `.`, `#` и другие](#11-префиксы-команд-------и-другие)
- [12. Лимиты sandbox и типичные ошибки](#12-лимиты-sandbox-и-типичные-ошибки)
- [13. Helpers / Operators / Features — как использовать правильно](#13-helpers--operators--features--как-использовать-правильно)
- [14. Готовые сценарные шаблоны](#14-готовые-сценарные-шаблоны)
- [15. Проверка и отладка](#15-проверка-и-отладка)
- [16. Примеры строк из реального кода](#16-примеры-строк-из-реального-кода)

---

## 1. Что такое JS e-scripting

Система e-scripting запускает пользовательские команды Telegram-бота в изолированной JS-песочнице.

Ключевые свойства:

1. Скрипты пишутся только на JavaScript.
2. Загрузка только `.js`.
3. В одном файле до 10 команд.
4. Выполнение в Node worker с watchdog/лимитами.
5. API к Telegram/экономике даётся через безопасные фасады `@bot/sdk/*`.

---

## 2. Модель безопасности

### Что блокируется

- Внешние импорты (например, `@nestjs/axios`, `axios`, `node:fs`).
- Доступ к файловой системе, процессам, worker_threads из скрипта.
- Произвольные сетевые вызовы.
- Небезопасные токены (`fetch`, `require`, `Date.now()` и т.д.) по политике sandbox.

### Двойная проверка импортов

Проверка идёт на двух уровнях:

1. **Компилятор** — отбрасывает скрипт до активации.
2. **Runner** — повторно валидирует на исполнении.

Это нужно, чтобы даже при ошибке/обходе одного слоя второй всё равно остановил скрипт.

---

## 3. Загрузка скриптов и структура файла

Базовый шаблон:

```js
import { defineCommand } from '@bot/sdk/commands';
import { tg } from '@bot/sdk/telegram';

export default defineCommand({
  name: 'пинг',
  description: 'Проверка ответа',
  scopes: ['telegram:send'],
  async run(ctx) {
    return tg.sendMessage(ctx.chat.id, 'pong');
  },
});
```

Мульти-командный файл:

```js
import { defineCommand, defineCommands } from '@bot/sdk/commands';

const a = defineCommand({ name: 'a', description: 'A', async run(ctx) { /* ... */ } });
const b = defineCommand({ name: 'b', description: 'B', async run(ctx) { /* ... */ } });

export default defineCommands([a, b]);
```

---

## 4. Импорты: что разрешено и что запрещено

### Разрешённые модули

- `@bot/sdk/commands`
- `@bot/sdk/telegram`
- `@bot/sdk/economy`
- `@bot/sdk/bank`
- `@bot/sdk/stats`
- `@bot/sdk/time`
- `@bot/sdk/random`
- `@bot/sdk/format`
- `@bot/sdk/policy`
- `@bot/sdk/i18n`
- `@bot/sdk/logger`
- `@bot/sdk/flags`
- `@bot/sdk/helpers`
- `@bot/sdk/operators`
- `@bot/sdk/features`

### Запрещено

```js
import { HttpModule } from '@nestjs/axios'; // ❌
import fs from 'node:fs'; // ❌
import axios from 'axios'; // ❌
```

---

## 5. Контекст `ctx`

Обычно доступно:

- `ctx.user.id`, `ctx.user.username`
- `ctx.chat.id`
- `ctx.args` — аргументы команды
- `ctx.locale`
- `ctx.script` — метаданные
- `ctx.scopes`
- `ctx.dryRun`

Контекст проходит redaction (PII/секреты вырезаются).

---

## 6. Telegram API (подробно)

### 6.1 sendMessage

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Текст', {
  replyMarkup: tg.inlineKeyboard([[tg.button('OK', 'ok')]]),
});
```

Что делает:

- Публикует сообщение в текущем чате.
- Возвращает объект с `messageId`.
- Внутри sandbox применяются лимиты на длину/количество/клавиатуры.

Типичные причины ошибки:

- попытка отправки в чужой чат,
- превышен лимит сообщений,
- слишком много кнопок.

### 6.2 reply

```js
await tg.reply('Ответ на текущее сообщение');
```

`reply` — сокращённый способ сделать ответ в том же чате.

### 6.3 editMessage

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Черновик');
await tg.editMessage(ctx.chat.id, sent.messageId, 'Готово ✅');
```

Важно:

- редактировать можно только сообщения из текущего чата;
- нужен корректный `messageId`.

### 6.4 deleteMessage

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Удалю через секунду');
await tg.deleteMessage(ctx.chat.id, sent.messageId);
```

### 6.5 pinMessage / unpinMessage

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Важное объявление');
await tg.pinMessage(ctx.chat.id, sent.messageId);
// ...позже
await tg.unpinMessage(ctx.chat.id, sent.messageId);
```

Как работает «закреп сообщений» в системе:

1. Скрипт вызывает `tg.pinMessage` с `messageId`.
2. Sandbox пишет side-effect `pinMessage`.
3. Python runtime при наличии `context.bot` преобразует synthetic ID в реальный ID отправленного сообщения.
4. Runtime вызывает API Telegram `pin_chat_message`.
5. Аналогично для `unpin_chat_message`.

Почему могло «не закрепляться» раньше:

- не совпадал `messageId` между sandbox и runtime,
- в контексте не было `context.bot`,
- сообщение удалено/недоступно.

### 6.6 sendPrivateMessage

```js
await tg.sendPrivateMessage(ctx.user.id, 'Личное уведомление');
```

Лимит DM отдельно от публичных сообщений.

### 6.7 react

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Выбери реакцию');
await tg.react(ctx.chat.id, sent.messageId, '🔥');
```

### 6.8 button / inlineKeyboard

```js
const kb = tg.inlineKeyboard([
  [tg.button('Обновить', 'cmd:refresh')],
  [tg.button('Помощь', 'cmd:help')],
]);
await tg.sendMessage(ctx.chat.id, 'Панель', { replyMarkup: kb });
```

---

## 7. Economy API (подробно)

### 7.1 getWallet / getBalance / getUserEconomy

```js
const wallet = await economy.getWallet(ctx.user.id);
const balance = await economy.getBalance(ctx.user.id);
const profile = await economy.getUserEconomy(ctx.user.id);
```

Использование:

- `getWallet` — детальная модель кошелька,
- `getBalance` — быстрое число,
- `getUserEconomy` — агрегированный профиль (кошелёк + банк и др.).

### 7.2 credit / debit / transfer

```js
await economy.credit({ amount: 10, opId: 'quest:credit:1', reason: 'quest-reward' });
await economy.debit({ amount: 3, opId: 'shop:debit:1', reason: 'purchase' });
await economy.transfer({ amount: 5, toUserId: ctx.user.id, opId: 'gift:transfer:1', reason: 'gift' });
```

Правила:

- `opId` обязателен;
- дубликаты `opId` блокируются;
- действуют квоты на объём операций.

### 7.3 addBalance / subtractBalance

```js
await economy.addBalance({ userId: ctx.user.id, amount: 2, opId: 'bonus:1', reason: 'daily-bonus' });
await economy.subtractBalance({ userId: ctx.user.id, amount: 1, opId: 'fee:1', reason: 'service-fee' });
```

Это удобные обёртки над ledger-операциями.

---

## 8. Bank API (подробно)

### 8.1 getBankBalance

```js
const bankBalance = await bank.getBankBalance(ctx.user.id);
```

### 8.2 deposit

```js
await bank.deposit({
  userId: ctx.user.id,
  amount: 50,
  opId: 'bank:deposit:1',
  reason: 'savings',
});
```

### 8.3 withdraw

```js
await bank.withdraw({
  userId: ctx.user.id,
  amount: 20,
  opId: 'bank:withdraw:1',
  reason: 'cashout',
});
```

---

## 9. Stats / Time / Random / Format / Policy / i18n / Logger / Flags

### stats

```js
const progress = await stats.getProgress(ctx.user.id);
const league = await stats.getLeagueSnapshot(ctx.user.id);
```

### time

```js
const now = time.now();
const nowIso = time.nowIso();
```

### random

```js
const value = rng.int(1, 100);
```

### format

```js
const money = fmt.money(1500, '₽');
const pct = fmt.percent(0.42);
const signed = fmt.signedMoney(-35, '₽');
```

### policy / i18n / logger / flags

```js
const allowed = await policy.canViewOtherStats(ctx.user.id, ctx.user.id);
const t = i18n(ctx.locale ?? 'ru');
logger.info('cmd.run', { user: ctx.user.id, at: time.nowIso() });
const ff = flags.enabled('quest_v2');
```

---

## 10. Мульти-командные скрипты (до 10)

- В одном файле разрешено до 10 команд.
- Каждая команда должна иметь уникальное `name`.
- Используйте `defineCommands([...])` для явной структуры.

Рекомендация:

- 1 команда = 1 сценарий (квест, профиль, банк, админ-панель, дайджест и т.д.).

---

## 11. Префиксы команд: `!!`, `/`, `.`, `#` и другие

Система нормализует токен команды:

- срезает ведущие символы (`!!`, `/`, `.`, `#`, ...),
- убирает `@botname`-суффикс,
- сопоставляет с именем команды.

Пример:

- `!!квест`, `/квест`, `.квест`, `#квест`, `квест@mybot` → одна и та же команда `квест`.

---

## 12. Лимиты sandbox и типичные ошибки

Основные лимиты:

- CPU timeout (обычный/тяжёлый режим),
- memory cap worker,
- max messages / max DM / max buttons,
- лимиты сериализации `ctx/result`,
- лимиты ledger операций.

Частые ошибки:

1. `Forbidden import` — импорт не из whitelist.
2. `Telegram rate limit exceeded` — слишком много sendMessage.
3. `Duplicate opId` — повтор одной финансовой операции.
4. `Daily ledger quota exceeded` — превышена дневная квота.

---

## 13. Helpers / Operators / Features — как использовать правильно

### Helpers (поверхностно)

`helpers.add1 ... helpers.add1200` — это параметрические математические кирпичики.
Используйте их **как компоненты формул**, а не как самоцель.

### Operators (поверхностно)

- `mul1..mul128`
- `sub1..sub128`
- `repeat1..repeat128`

Подход: строить компактные преобразования для игровой логики/метрик.

### Features (поверхностно)

- `feature1..feature500`
- `onAnyMessage`, `onReply`, `onMention`

Подход: маркеры сценарных стадий, реакций, игровых веток.

---

## 14. Готовые сценарные шаблоны

### Шаблон «квест с наградой»

```js
const score = rng.int(1, 100);
const reward = Math.max(1, score % 10);
await economy.credit({ amount: reward, opId: `quest:${ctx.user.id}:${score}`, reason: 'quest-reward' });
return tg.sendMessage(ctx.chat.id, `Квест пройден, награда +${reward}`);
```

### Шаблон «банковская операция»

```js
const before = await bank.getBankBalance(ctx.user.id);
await bank.deposit({ userId: ctx.user.id, amount: 5, opId: 'bank:d:1', reason: 'daily-save' });
const after = await bank.getBankBalance(ctx.user.id);
return tg.sendMessage(ctx.chat.id, `Банк: ${before} -> ${after}`);
```

### Шаблон «сообщение + закреп + ЛС»

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Важное объявление');
await tg.pinMessage(ctx.chat.id, sent.messageId);
await tg.sendPrivateMessage(ctx.user.id, 'Объявление закреплено');
```

---

## 15. Проверка и отладка

1. Проверить импорты на whitelist.
2. Проверить наличие `description` у каждой команды.
3. Проверить уникальность `opId` для всех экономических операций.
4. Проверить лимиты сообщений и кнопок.
5. Прогнать в `dryRun` (если поддерживается командой).

---

## 16. Примеры строк из реального кода

```js
const sent = await tg.sendMessage(ctx.chat.id, 'Доска управления', { replyMarkup: kb });
await tg.pinMessage(ctx.chat.id, sent.messageId);
await economy.credit({ amount: reward, opId: `quest:${ctx.user.id}:${score}`, reason: 'quest-reward' });
const before = await bank.getBankBalance(ctx.user.id);
await bank.deposit({ userId: ctx.user.id, amount: 2, opId: `bank:d:${ctx.user.id}`, reason: 'scout-deposit' });
const hooks = features.onAnyMessage(ctx, 'feature1');
const score = operators.mul2(helpers.add1200(1));
const allow = await policy.canViewOtherStats(ctx.user.id, ctx.user.id);
logger.info('scout.run', { user: ctx.user.id, at: time.nowIso() });
return tg.sendPrivateMessage(ctx.user.id, 'Отчёт готов');
```

---

## Краткое резюме

- Детально документируйте **реальные функции** (Telegram, Economy, Bank).
- Helpers/operators/features используйте как строительные блоки, не превращая документацию в шум.
- Держите команды сценарными, самостоятельными и проверяемыми.
