---
title: "Полуавтоматическое создание карточек для Anki"
slug: custom-yomitan-anki-setup
date: 2025-11-19T17:48:11+02:00
draft: false
toc: true
tags:
  - greek
  - anki
---

## Недостатки стандартной реализации Yomitan

Стандартные настройки и карточки Yomitan хорошо работают для большого количества поддерживаемых языков. Yomitan генерирует карточки, с одной стороны которых слово на иностранном языке, а с другой — краткое словарное определение со ссылкой на Wiktionary.
Слова в Anki через Yomitan добавляются очень удобно: с зажатой кнопкой `shift` достаточно навести на интересующее слово, чтобы увидеть его словарную карточку и тут же добавить одним кликом в запущенный Anki.

{{< image src="images/yomitan-capture.jpeg" position="center"  style="height: 200px;">}}

Получается хорошо и красиво, но на начальных этапах изучения языка для меня важнее видеть просто слово и его перевод, в идеале на родной язык, чтобы активно набирать обширный словарный запас и не тратить силы на тонкости этимологии и редкие значения.

Кроме того, для быстрого набора словарного запаса полезно учить не только перевод с изучаемого языка, но и в обратную сторону, где на лицевой стороне будет слово на родном языке. Стандартные обратные Yomitan-карточки с определениями на лицевой стороне для этого плохо подходят, поскольку нужно затратить немало усилий на анализ этого определения, чтобы быстро понять, какое слово требуется перевести.

Для не самых популярных языков, например греческого, также возникает проблема из-за того, что пара греческий-русский плохо представлена в Wiktionary, соответственно часть карточек приходится добавлять с русским и английским определением, а часть только с английским.

{{< row >}}
  {{< image src="images/standart-front.png" position="none" style="height: 400px;" >}}
  {{< image src="images/standart-back.png" position="none" style="height: 400px;" >}}
  {{< image src="images/standart-front-2.png" position="none" style="height: 400px;" >}}
  {{< image src="images/standart-back-2.png" position="none" style="height: 400px;" >}}
{{< /row >}}

## Решение

Концептуально хочется иметь карточки, которые бы удовлетворяли следующим условиям:

1. Карточка должна быть максимально простой визуально, никакого лишнего шума;
2. Доступ к информации из словаря должен быть простой и быстрый;
3. Процесс добавления новых карточек должен быть максимально прост;

Таким образом напрашивается дизайн карточки, у которой на лицевой стороне слово, а на задней — перевод плюс скрытое за неприметной кнопкой словарное определение. Все это можно реализовать стандартными средствами Anki.

В итоге процесс добавления нового слова получается таким:

1. Как и раньше выбираем интересные слова через Yomitan, но настраиваем его так, чтобы он записывал свое значение в отдельное поле `Glossary`;
2. Переходим в Anki, переходим в обзор колод, там по фильтру: `Back:` выбираем карточки, у которых нет перевода;
3. Заполняем подходящий перевод по русскому/английскому определению; кстати, для непонятных слов очень удобно запрашивать значение у любого GPT;
4. Переходим к следующей заметке из фильтра с помощью горячей клавиши `ctrl-N`;

На скриншотах внешний вид получившихся карточек:

{{< row >}}
  {{< image src="images/custom-front.png" position="none" style="height: 400px;" >}}
  {{< image src="images/custom-back.png" position="none" style="height: 400px;" >}}
  {{< image src="images/custom-dict.png" position="none" style="height: 400px;" >}}
{{< /row >}}

{{< row >}}
  {{< image src="images/custom-front-2.png" position="none" style="height: 400px;" >}}
  {{< image src="images/custom-back-2.png" position="none" style="height: 400px;" >}}
  {{< image src="images/custom-dict-2.png" position="none" style="height: 400px;" >}}
{{< /row >}}

## Техническая реализация

Добавим новый тип карточек, создадим его на основе стандартной "Basic (with reversed card)".
Также добавим туда отдельное поле `Glossary`

{{< image src="images/anki-settings-cards.png" position="center" style="height: 300px;" >}}

### Стиль

```css
.card {
  font-family: arial;
  font-size: 20px;
  text-align: center;
}

.line-with-audio {
  position: relative;
  text-align: center;
  padding: 0.1em 2.2em;
  margin-bottom: 0.6em;
}


.line-text {
  display: inline-block;
}

.line-with-audio .replaybutton,
.line-with-audio .replay-button {
  position: absolute;
  right: 0.2em;
  top: 50%;
  transform: translateY(-50%);

  border: none;
  background: none;
  padding: 0;
  margin: 0;

  display: inline-flex;
  align-items: center;
  justify-content: center;

  font-size: 0.9em;
}


.line-with-audio .replaybutton svg,
.line-with-audio .replay-button svg {
  display: none !important;
}

.line-with-audio .replaybutton::before,
.line-with-audio .replay-button::before {
  content: "🗣️";
  opacity: 0.5;
  line-height: 1;
}

.glossary-only {
  margin-top: 0.4em;
}

.glossary-hint-wrapper {
  margin-top: 0.75em;
  text-align: center;
}

.glossary-hint-button {
  font-size: 0.8em;
  padding: 4px 10px;
  border-radius: 999px;
  border: none;
  background-color: rgba(128, 128, 128, 0.12);
  opacity: 0.85;
  cursor: pointer;
}

.glossary-hint-content {
  margin-top: 0.75em;
  padding: 0.6em 0.9em;
  border-radius: 0.6em;
  background-color: rgba(128, 128, 128, 0.06);
  box-shadow:
    0 2px 4px rgba(0, 0, 0, 0.12),
    0 0 0 1px rgba(0, 0, 0, 0.02);
  font-size: 0.9em;
  line-height: 1.4;
}
```

### Front/back template код для карточек

#### Card 1

Front:

```html
<div class="line-with-audio">
  <span class="line-text">
    {{Front}}
  </span>
  {{Audio}}
</div>
```

Back:

```html
{{FrontSide}}

<hr id=answer>

{{#Back}}
  <div class="line-with-audio">
    <span class="line-text">
      {{Back}}
    </span>
  </div>
{{/Back}}

{{^Back}}
  {{#Glossary}}
    <div class="glossary-only">
      {{Glossary}}
    </div>
  {{/Glossary}}
{{/Back}}

{{#Back}}
  {{#Glossary}}
    <div class="glossary-hint-wrapper">
      <button class="glossary-hint-button"
              type="button"
              onclick="var c=this.nextElementSibling;
                       this.style.display='none';
                       if (c) { c.style.display='block'; }
                       return false;">
        Словарь
      </button>

      <div class="glossary-hint-content" style="display:none;">
        {{Glossary}}
      </div>
    </div>
  {{/Glossary}}
{{/Back}}
```


#### Card 2

Front:

```html
{{#Back}}
  <span class="line-text">
    {{Back}}
  </span>
{{/Back}}

{{^Back}}
  {{#Glossary}}
    <div class="glossary-only">
      {{Glossary}}
    </div>
  {{/Glossary}}
{{/Back}}
```

Back:

```html
{{FrontSide}}

<hr id=answer>

<div class="line-with-audio">
  <span class="line-text">
    {{Front}}
  </span>
  {{Audio}}
</div>

{{^Back}}
  {{#Glossary}}
    <div class="glossary-only">
      {{Glossary}}
    </div>
  {{/Glossary}}
{{/Back}}

{{#Back}}
  {{#Glossary}}
    <div class="glossary-hint-wrapper">
      <button class="glossary-hint-button"
              type="button"
              onclick="var c=this.nextElementSibling;
                       this.style.display='none';
                       if (c) { c.style.display='block'; }
                       return false;">
        Словарь
      </button>

      <div class="glossary-hint-content" style="display:none;">
        {{Glossary}}
      </div>
    </div>
  {{/Glossary}}
{{/Back}
```

## Бонус: полезные аддоны

### HyperTTS

Вы, возможно, заметили иконку с говорящей головой на карточках.
Это аудио, автоматически сгенерированное с помощью Gemini TTS плагином HyperTTS.
Необходимость в этом возникла из-за того, что в AnkiDroid встроенный TTS читает всю заметку целиком, игнорируя скрытые элементы.
В качестве обходного решения можно было бы использовать специальные теги, выделяющие фрагменты для озвучивания, однако тогда пришлось бы прямо в шаблоне указывать язык для TTS, что сделало бы шаблон не универсальным.
Аудио генерируется в поле `Audio` одной кнопкой при редактировании заметки и затем используется в шаблоне карточки.
Если какие-либо аудиофайлы остались несгенерированными, такие карточки можно найти по запросу `Audio:` и скопом их обновить.

### backfill-anki-yomitan

Крайне удобный плагин, по названию которого можно догадаться, что он взаимодействует с Yomitan и извлекает из него словарную справку.
Это удобно в сценариях, когда карточки добавляются полностью вручную, позволяя дозаполнить поле `Glossary` одним нажатием кнопки либо сразу во время редактирования, либо позже, если карточка была добавлена с телефона.
