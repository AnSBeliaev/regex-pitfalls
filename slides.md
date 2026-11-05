---
theme: seriph
colorSchema: dark
background: '#32353C'
title: "Доклад 3. Регулярные выражения: ошибки и новый синтаксис"
class: text-center
drawings:
  persist: false
transition: slide-left
comark: true
duration: 40min
---

# Регулярные выражения:<br>ошибки и новый синтаксис

---
class: thesis-slide
---

# Две беды

- **К шаблону не возвращаются.** Регулярка нужна, чтобы найти кусок текста: номер, почту, строку в логе. Она что-то находит, и внутрь никто обычно не смотрит.
- **Со стороны это не ошибка шаблона.** В обоих случаях тест зелёный.
  - На форме одна и та же строка срабатывает через раз.
  - На короткой строке в поле поиска вкладка зависает.

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

isTicketId('T-10421') // true,  lastIndex → 7
isTicketId('T-10421') // false, lastIndex → 0
isTicketId('T-10421') // true,  lastIndex → 7
```

- **`test` проверяет, есть ли совпадение,** и отвечает `true` или `false`.
- **Флаг `g` — для многих совпадений в одной длинной строке.** Новую не отличает и помнит `lastIndex`.
- **После `true` позиция сразу за найденным,** в `T-10421` это 7. Второй вызов идёт с 7, символов нет: `false`, индекс 0. Третий снова с начала.

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

- **В `log` у нас есть длинный текст и несколько номеров**, их надо найти и подсветить. Цикл крутит `exec` по одной и той же регулярке.
- **Первый круг:** `exec` ищет с `lastIndex = 0`, находит `T-10421`, `highlight` подсвечивает. Сам `exec` ставит `lastIndex` сразу за этим номером.
- **Второй круг:** та же регулярка видит новый `lastIndex` и не застревает на первом номере. Ищет дальше, находит следующий, снова сдвигает индекс.
- **Когда номера кончились,** `exec` возвращает `null`, цикл стоп, `lastIndex` снова 0. **`test` с `/g` крутит то же поле** — поэтому на прошлом слайде и получалось через раз.

---
class: thesis-slide answer-slide
---

# Не смешивать `/g` и `test`

```js
function isTicketId(query) {
  return /^T-\d+$/.test(query)
}
```

- **Нам не надо искать много номеров подряд.** Нужно только понять, номер ли тикета ввели в поиск. Пишем `test` без `/g` — между вызовами ничего не запоминается, второй раз снова `true`.
- **Нельзя убрать `/g`?** После вызова поставьте `lastIndex = 0`. Или объявите регулярку внутри функции: каждый вызов начнёт с нуля.

---
class: thesis-slide problem-slide
---

# Деструктуризация `match` падает

```js
const [, id] = log.match(/T-(\d+)/)
// из 'T-10421' в id попадёт '10421'
```

- **Строка ищет в логе номер тикета** и кладёт цифры в `id`. Скобки `(\d+)` как раз эти цифры. Запятая в `[, id]` пропускает целое `T-10421` и берёт то, что в скобках.
- **Если в логе нет `T-…`, `match` вернёт `null`.** Из `null` цифры не достать: код выбросит `TypeError`.
- **На ревью это не заметят:** строка короткая и выглядит нормально. В тесте если кладут лог с номером, `null` не поймают.

---
class: thesis-slide answer-slide check-slide
---

# Сначала проверка

```js
const [, id] = log.match(/T-(\d+)/)
// нет T-… → null, TypeError
```

````md magic-move
```js
const [, id] = log.match(/T-(\d+)/)

const [, id] = log.match(/T-(\d+)/)

const [, id] = log.match(/T-(\d+)/)
```
```js {1}
const [, id] = log.match(/T-(\d+)/) ?? []

const [, id] = log.match(/T-(\d+)/)

const [, id] = log.match(/T-(\d+)/)
```
```js {3}
const [, id] = log.match(/T-(\d+)/) ?? []

const id = log.match(/T-(\d+)/)?.[1]

const [, id] = log.match(/T-(\d+)/)
```
```js {5-7}
const [, id] = log.match(/T-(\d+)/) ?? []

const id = log.match(/T-(\d+)/)?.[1]

const match = log.match(/T-(\d+)/)
if (!match) return
const [, id] = match
```
````

<v-switch at="1">
  <template #0><p>Сверху ошибка. Снизу три копии той же строки — правим по очереди.</p></template>
  <template #1><p>Первая правка: вместо <code>null</code> пустой массив, <code>id</code> будет <code>undefined</code>.</p></template>
  <template #2><p>Вторая: нет совпадения — сразу <code>undefined</code>, без <code>TypeError</code>.</p></template>
  <template #3><p>Третья: сначала проверка, потом цифры. Длиннее, зато видно глазами.</p></template>
</v-switch>

<style>
.slidev-layout.check-slide .slidev-code,
.slidev-layout.check-slide pre {
  font-size: 15px !important;
  line-height: 22px !important;
  margin: 0.15rem 0 !important;
}
.slidev-layout.check-slide p {
  text-align: left;
  font-size: 16px;
  line-height: 1.35;
  margin: 0.35rem 0 0;
}
</style>

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

- **Один и тот же `match`, два разных ответа.** Без `/g` — первое совпадение целиком, цифры из скобок, `index` и исходный `log` в `input`. С `/g` — список всех `T-10421`, без скобок, без `index` и без `input`.
- **Без `/g` во втором элементе цифры `10421`.** С `/g` во втором элементе уже другой целый номер `T-10421`. Ждали цифры — получили не то.
- **TypeScript разницы не видит:** тип один, `RegExpMatchArray | null`. Написанию не помешает, упадёт уже в рантайме.

---
class: thesis-slide answer-slide
---

# Группы — у `exec` и `matchAll`

```js
const result = /T-(\d+)/.exec(log)
result?.[1] // '10421'
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

- **`(`, `[`, `?`, `+` в вводе — спецсимволы шаблона.** Конструктор ждёт закрывающую скобку, не находит — `SyntaxError`, поиск падает.
- **Плюс внутри плюса — ReDoS.** Движок перебирает миллионы разрезов строки. На 20–30 символах вкладка зависает, на Node процессор уходит в 100%. Гонять на докладе не будем.

---
class: thesis-slide answer-slide
---

# Ввод не класть в шаблон сырьём

```js
new RegExp(RegExp.escape(query), 'gi')
```

- **Экранировать ввод.** `(` станет литералом. Штатный способ — `RegExp.escape`. Для нашего `defaults` — **с полифилом**.
- **Не вкладывать плюс в плюс:** `(a+)+`, `(\w+)*`. Если нужен просто текст — `includes`, без регулярки.

---
class: thesis-slide problem-slide
---

# Подсветка режет эмодзи пополам

```js
'👍'.length              // 2
/^.$/.test('👍')         // false — точка берёт одну единицу
/^[👍]$/.test('👍')      // false — в скобках две половинки, не символ
/^[👍]$/.test('\uD83D')  // true  — одна суррогатная половина
```

- **Без `u` регулярка смотрит кодовые единицы**, не символы. У пальца их две.
- Подсветка по `index` и длине совпадения **отрезает эмодзи пополам** — в интерфейсе кракозябра.

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
// $& — найденный текст. Без /g подсветится только первое T-10421

log.replaceAll(/T-\d+/, '<mark>$&</mark>')
// TypeError — нет флага g
```

- **`replace` без `/g` молча меняет одно.** В логе два номера, на экране подсвечен один. Предупреждения не будет.
- **`replaceAll` с регуляркой без `/g` бросает `TypeError`.** Метод называется «заменить все», поэтому флаг `g` обязателен: забыли — ошибка сразу, а не тихий баг.

---
class: thesis-slide problem-slide
---

# Доллар в строке замены

```js
log.replace(/T-(\d+)/g, note)
// note === '$1' → вместо текста цифры из скобок: 10421

log.replace(/T-(\d+)/g, () => note)
// note как есть: доллары внутри не трогает
```

- **Второй аргумент `replace` — не всегда буквальный текст.** `$&`, `$1`, `$$` в нём — подстановки.
- **Если `note` пришёл из инпута или API** — передавайте функцию: она вставит строку как есть. Иначе `$1` в тексте пользователя перепишет данные.

---
class: thesis-slide problem-slide
---

# Точка не берёт перевод строки

```js
// в log: ...T-10421...\n...T-10422...
/T-10421.*T-10422/.test(log)    // false — точка не берёт \n
/T-10421.*T-10422/s.test(log)   // true  — флаг s разрешает
```

- **Точка по умолчанию — любой символ, кроме перевода строки.** Флаг `s` как раз про точку: с ним `.*` перешагивает через `\n`.
- **Флаг `m` про другое: якоря `^` и `$`.** Без `m` — края всего текста, с `m` — начало и конец каждой строки. Время в начале строки лога — `m`, кусок текста через перенос — `s`.

---
class: thesis-slide
---

# Жадный и ленивый

```js
log.match(/T-.*\d+/s)
// от первого T- до последнего 10422: текст, 👍 и второй номер внутри

log.match(/T-.*?\d+/s)
// 'T-10421' — остановился на первом номере
```

- **`.*` жадный: берёт как можно больше.** Добежит до конца и только потом отдаст хвост под `\d+`.
- **`.*?` ленивый: берёт как можно меньше.** Нашёл первый номер — сразу стоп.
- На короткой строке с одним тикетом оба совпадают. На реальном логе **подсветка сжирает полсообщения**.

---
class: thesis-slide
---

# Что появилось и все пропустили

Дальше восемь вещей. Не каталог: каждая закрывает боль из первой части — обход лога, подсветка, номер из скобок, ввод, эмодзи. Фильтр тот же: наш `defaults`, в нём ещё Chrome 109.

- **Используем** — браузеры из списка, включая 109, уже умеют.
- **С полифилом** — метод лёгкий, полифил кладём и пишем.
- **Не пишем** — новый синтаксис, которого в 109 нет, а сборка сама не перепишет.
- Процент с caniuse **сам по себе не ответ:** «96% в мире» не спасает, если ваши пользователи на Chrome 109.

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

- **Лучшее от обеих форм `match`:** все совпадения, и при этом цифры из скобок и `index` на каждом шаге. Компромисс «либо всё, либо детали» больше не нужен.
- **Вместо цикла с `exec`.** Итератор сам идёт по логу. На слайде литерал `/…/g` — каждый вызов новый объект, `lastIndex` с нуля.
- Флаг `g` обязателен, без него `TypeError`. В нашем списке с 2020-го. **Используем.**

---
class: thesis-slide answer-slide
---

# Флаг `d`

```js
for (const match of log.matchAll(/T-(\d+)/gd)) {
  const [start, end] = match.indices[0] // весь T-10421
  const [from, to] = match.indices[1]   // только 10421
}
```

- **`indices` — пары `[start, end]` для подсветки.** Раньше: `index` плюс длина, для скобок ещё `indexOf` внутри совпадения — при повторе цифр легко взять не то место.
- **Границы сразу под `slice` / `substring`:** `end` не входит в отрезок, единицу вычитать не надо.
- В Chrome с 90-го, в нашем 109 уже есть. **Используем.**

---
class: thesis-slide answer-slide
---

# Именованные группы

```js
const match = /T-(?<id>\d+)/d.exec(log)
const { id } = match?.groups ?? {}     // '10421'
match?.indices.groups.id               // [start, end]

log.replace(/T-(?<id>\d+)/g, '/tickets/$<id>')
```

- **`groups.id` вместо `match[1]`.** В `/T-(\d+)/` цифры — это `match[1]`. Переписали на `/(ticket )?T-(\d+)/` — цифры уже `match[2]`, а старый код всё ещё читает `match[1]`. Имя `id` от порядка скобок не зависит.
- **В замене то же имя:** `$<id>`. С флагом `d` координаты тоже по имени: `indices.groups.id`.
- С 2018 года, в списке давно. **Используем.**

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

---
class: thesis-slide answer-slide
---

# Модификатор внутри выражения

```js
/T-\d+.*(?i:проверь)/s.test(log)
// true и для «ПРОВЕРЬ»: префикс T- как написан
```

- **Регистр только у куска шаблона**, не у всей регулярки. Раньше либо флаг `i` на всё, либо два шаблона.
- Синтаксис 2025 года, в нашем 109 нет. **Не пишем**, пока список не догонит.

---
class: thesis-slide answer-slide
---

# Свойства юникода

```js
log.match(/\p{Emoji}/u)  // '👍'
/\p{Letter}/u.test('М')  // true
```

- **`\p{Emoji}` вместо самодельного диапазона.** Без флага `u` или `v` это ошибка разбора.
- С `u` в списке с 2018 года. **Используем.** Составные эмодзи — уже история флага `v`.

---
class: thesis-slide catalog-slide
---

# Один поиск

````md magic-move
```js
function highlight(log, query) {
  const re = new RegExp(query)
  const [, id] = log.match(/T-(\d+)/)
  return log.replace(re, '<mark>$&</mark>')
}
```
```js {2,4}
function highlight(log, query) {
  const re = new RegExp(RegExp.escape(query), 'gu')
  const [, id] = log.match(/T-(\d+)/)
  return log.replace(re, (chunk) => '<mark>' + chunk + '</mark>')
}
```
```js {2-9}
function highlight(log, query) {
  const re = new RegExp(RegExp.escape(query), 'gdu')
  let out = ''
  let last = 0
  for (const m of log.matchAll(re)) {
    const [start, end] = m.indices[0]
    out += log.slice(last, start) + '<mark>' + log.slice(start, end) + '</mark>'
    last = end
  }
  return out + log.slice(last)
}
```
````

<v-switch at="1">
  <template #0><p>Сырой ввод, первое вхождение, match без проверки.</p></template>
  <template #1><p>Скобка не ломает поиск, подсвечены оба тикета, эмодзи целый.</p></template>
  <template #2><p>Подсветка по indices. lastIndex больше не общий.</p></template>
</v-switch>

<style>
.slidev-layout.catalog-slide .slidev-code,
.slidev-layout.catalog-slide pre {
  font-size: 15px !important;
  line-height: 22px !important;
  margin: 0.15rem 0 !important;
}
.slidev-layout.catalog-slide p {
  text-align: left;
  font-size: 16px;
  line-height: 1.35;
  margin: 0.35rem 0 0;
}
</style>

---
class: thesis-slide links-slide
---

# Ссылки

- **Не мешать `/g` и `test`.** Не деструктурировать `match` без запасного пути. Не класть сырой ввод в `new RegExp`.
- **[regex101](https://regex101.com/)** — отладчик, видно сколько шагов занимает матч.
- **[recheck](https://github.com/makenowjust-labs/recheck)** — ищет суперлинейный перебор.
- **[MDN, регулярки](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions)** и **[RegExp.escape](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape)**.
- **[caniuse](https://caniuse.com/)** и `npx browserslist` — как во втором докладе: процент сам по себе не ответ.
