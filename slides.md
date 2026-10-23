---
theme: seriph
colorSchema: dark
background: '#32353C'
title: "Доклад 3. Регулярные выражения: грабли и то, что все пропустили"
class: text-center
drawings:
  persist: false
transition: slide-left
comark: true
duration: 40min
---

# Регулярные выражения:<br>грабли и то, что все пропустили

---
class: thesis-slide
---

# Две беды

- **За ними никто не следит.** Пишут как в 2015-м и не знают, что половину костылей уже можно выбросить.
- **Ловятся даже опытные.** Баг проявляется не сразу и выглядит как мистика.

<style>
.slidev-layout.thesis-slide p,
.slidev-layout.thesis-slide li {
  font-size: 16px;
  line-height: 1.4;
}
.slidev-layout.thesis-slide ul {
  margin-top: 0.6rem;
}
</style>

---
class: thesis-slide
---

# Кейс: поиск по логу тикета

```js
const log = `
14:02 Мария: не открывается T-10421, пишет 👍
14:03 Иван: проверь T-10421 и T-10422
`
```

- По логу надо **найти, подсветить и достать номера** вроде `T-10421`.
- В поле поиска человек может ввести **что угодно**.

---
class: thesis-slide problem-slide
---

# `test` с `/g` срабатывает через раз

```js
const TICKET = /^T-\d+$/g

function isTicketId(query) {
  return TICKET.test(query)
}

isTicketId('T-10421') // true
isTicketId('T-10421') // false
isTicketId('T-10421') // true
```

- **`lastIndex` хранится на регулярке**, не на строке. После `true` позиция в конце, следующий вызов стартует не сначала.
- **В тестах это не ловится:** модуль каждый раз поднимается заново.

---
class: thesis-slide
---

# Почему так сделано

```js
const re = /T-\d+/g
let match

while ((match = re.exec(log))) {
  highlight(match[0], match.index)
}
```

- **`exec` двигает `lastIndex`.** Нашёл совпадение — следующее ищет уже дальше.
- **`test` с `/g` крутит то же поле.** Два вызова на одной строке — как на прошлом слайде.

---
class: thesis-slide answer-slide
---

# Не смешивать `/g` и `test`

```js
function isTicketId(query) {
  return /^T-\d+$/.test(query)
}
```

- **Валидатору флаг `g` не нужен.**
- Если регулярка с `/g` уже общая — **сбросить `lastIndex`** или **собрать её внутри функции**.

---
class: thesis-slide problem-slide
---

# Деструктуризация `match` падает

```js
const [, id] = log.match(/T-(\d+)/)
```

- **Нет совпадения — `match` вернёт `null`.** `null` деструктурировать нельзя, будет `TypeError`.
- Строка выглядит аккуратно и проходит ревью. Тест, где номер всегда есть, это не поймает.

---
class: thesis-slide answer-slide
---

# Сначала проверка

```js
const [, id] = log.match(/T-(\d+)/) ?? []
```

- **`?? []`** — нет совпадения, `id` будет `undefined`, без исключения.
- Или `log.match(...)?.[1]`, или `exec` и проверка `if (!match)`.

---
class: thesis-slide problem-slide
---

# Один флаг меняет форму `match`

```js
log.match(/T-(\d+)/)
// ['T-10421', '10421', index, input]

log.match(/T-(\d+)/g)
// ['T-10421', 'T-10421']
```

- **Без `/g` — совпадение, группы, `index`.** С `/g` — плоский список строк, групп нет.
- **TypeScript не ловит:** тип один, `RegExpMatchArray | null`.

---
class: thesis-slide answer-slide
---

# Группы — у `exec` и `matchAll`

```js
const match = /T-(\d+)/.exec(log)
match?.[1] // '10421'
```

- **`match` без `/g`** — первое совпадение с группами. **`match` с `/g`** — все совпадения, только текстом.
- Если нужны все вхождения и группы — **`exec` в цикле** или **`matchAll`**.

---
class: thesis-slide problem-slide
---

# В поле поиска скобка, потом зависание

```js
new RegExp(query, 'gi')
// query === '('   → SyntaxError

new RegExp(`(${query})+$`)
// query === 'a+'  →  /(a+)+$/
```

- **Скобка для конструктора — спецсимвол**, поиск падает.
- **`(a+)+$` на паре десятков `a` вешает вкладку**, на Node — процесс. На слайде только вид шаблона, гонять его не будем.

---
class: thesis-slide answer-slide
---

# Ввод не класть в шаблон сырьём

```js
new RegExp(RegExp.escape(query), 'gi')
```

- **Экранировать, чтобы скобка осталась скобкой.** Штатный способ — `RegExp.escape`.
- **Не вкладывать плюс в плюс.** Сырой ввод в `new RegExp` не кладём.

---
class: thesis-slide problem-slide
---

# Подсветка режет эмодзи пополам

```js
'👍'.length           // 2
/^.$/.test('👍')      // false
/^[👍]$/.test('👍')   // false
```

- **Без `u` регулярка смотрит кодовые единицы**, не символы. У пальца их две.
- Подсветка по `index` и длине совпадения **отрезает эмодзи пополам**.

---
class: thesis-slide answer-slide
---

# Флаг `u`

```js
/^.$/u.test('👍')     // true
/^[👍]$/u.test('👍')  // true
```

- **Флаг `u` — смотреть символами.** Точка берёт весь палец.
- Для номеров тикетов разницы нет. Для лога с эмодзи — есть.

---
class: thesis-slide problem-slide
---

# `replace` без `/g` меняет первое

```js
log.replace(/T-\d+/, '<mark>$&</mark>')
// только первое T-10421

log.replaceAll(/T-\d+/, '<mark>$&</mark>')
// TypeError — нет флага g
```

- **`replace` без `/g` молча меняет одно.** Иван просил два номера, подсвечен один.
- **`replaceAll` с регуляркой без `/g` бросает `TypeError`.** С регуляркой флаг `g` обязателен.

---
class: thesis-slide problem-slide
---

# Доллар в строке замены

```js
log.replace(/T-\d+/g, note)
// note === '$1' → подставится группа, не текст

log.replace(/T-\d+/g, () => note)
```

- **`$&`, `$1`, `$$` во втором аргументе — подстановки**, не буквальный текст.
- Если замена от человека — **функция**: она вернёт строку как есть.

---
class: thesis-slide problem-slide
---

# Точка не берёт перевод строки

```js
/T-10421.*T-10422/.test(log)    // false
/T-10421.*T-10422/s.test(log)   // true
```

- **Точка без `s` не совпадает с переводом строки.** Между двумя тикетами как раз перенос.
- **`^` и `$` без `m` — края всей строки**, с `m` — каждой. Это разные флаги, их путают.

---
class: thesis-slide
---

# Жадный и ленивый

```js
log.match(/T-.*\d+/)
// от первого T- до последнего 10422

log.match(/T-.*?\d+/)
// 'T-10421'
```

- **`.*` берёт как можно больше**, `.*?` — как можно меньше.
- На короткой строке оба совпадают. На логе с двумя тикетами **подсветка разъезжается на полсообщения**.

---
class: thesis-slide
---

# Что появилось и все пропустили

Тот же поиск по логу. Статусы как во втором докладе, для нашего `defaults` — в нём ещё Chrome 109.

- **Используем** — браузеры из списка уже умеют.
- **С полифилом** — метод лёгкий, в список всё равно кладём.
- Процент с caniuse **сам по себе не ответ.**

---
class: thesis-slide answer-slide
---

# `matchAll`

```js
for (const match of log.matchAll(/T-(\d+)/g)) {
  match[1]    // '10421'
  match.index
}
```

- **Вместо цикла с `exec` и `lastIndex`.** Совпадение, группы и `index` на каждом шаге, регулярку между вызовами не трогаем.
- Флаг `g` обязателен, без него `TypeError`. В нашем списке есть. **Используем.**

---
class: thesis-slide answer-slide
---

# Флаг `d`

```js
for (const match of log.matchAll(/T-(\d+)/gd)) {
  const [start, end] = match.indices[0]
}
```

- **`indices` — откуда по куда вырезать** для подсветки. Раньше считали `index` плюс длина.
- В Chrome с 90-го, в нашем 109 уже есть. **Используем.**

---
class: thesis-slide answer-slide
---

# Именованные группы

```js
const match = /T-(?<id>\d+)/.exec(log)
match.groups.id // '10421'

log.replace(/T-(?<id>\d+)/g, '/tickets/$<id>')
```

- **`groups.id` вместо `match[1]`.** Скобку в шаблоне подвинули — имя на месте.
- В замене то же имя: `$<id>`. С 2018 года, в списке давно. **Используем.**

---
class: thesis-slide answer-slide
---

# Lookbehind

```js
for (const match of log.matchAll(/(?<=T-)\d+/g)) {
  match[0] // '10421'
}
```

- **Совпадение — только цифры, если слева `T-`.** Раньше брали группу и выкидывали префикс руками.
- `(?<!...)` — наоборот, не брать цифры, если это номер тикета. С 2018 года. **Используем.**

---
class: thesis-slide answer-slide
---

# `RegExp.escape`

```js
new RegExp(RegExp.escape(query), 'gi')
// query === '('  → ищем скобку, поиск не падает
```

- **Скобка и точка остаются литералами.** Самодельная функция со StackOverflow то и дело забывает дефис или слэш.
- Метод, в Chrome с 136-го. В нашем `defaults` ещё 109. **С полифилом.**

---
class: thesis-slide answer-slide
---

# Флаг `v`

```js
/\p{Emoji}/u.test('👍')       // true
/\p{RGI_Emoji}/v.test('👨‍💻') // весь эмодзи целиком
```

- **`v` вместо `u`:** множества символов и составные эмодзи. С `u` на одной регулярке смешать нельзя.
- Синтаксис, в Chrome с 112-го. В нашем 109 нет. **Не пишем**, пока список не догонит.
