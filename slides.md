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
