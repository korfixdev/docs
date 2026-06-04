# Стилистика игровых миниапов

> **Навигация:** [index](index.md) · [concepts](concepts.md) · [config-korgames](config-korgames.md) · [client-api](client-api.md) · [api-reference](api-reference.md) · [recipes](recipes.md) · **вы здесь** · [checklist](checklist.md).

Краткие правила: как выглядеть нативно в платформе И сохранить игровое настроение.

> **Базовая стилистика миниапов** — в [miniapps/styling.md](../miniapps/styling.md) (CSS-переменные Korfix, кнопки, карточки). Здесь — только gamedev-специфика поверх.

---

## Правило №1 — `body { background: transparent }`

**Обязательно.** Iframe миниапа встраивается в панель платформы. Если поставить `background: #fff9e6` (жёлтый) на `body` — получишь жёлтый квадрат в серой панели, выглядит инородно.

Весь тематический стиль — **внутри контейнера**, не на body. Паттерн:

```html
<body>
    <div class="app">
        <!-- Нейтральные элементы: хедер, топбар, профиль-стрип -->
        <div class="topbar">...</div>
        <div class="profile-strip">...</div>

        <!-- Игровая арена — вся игровая атмосфера ТУТ -->
        <div class="game-frame">
            <h1>Coin Clicker</h1>
            <div class="coin">...</div>
        </div>
    </div>
</body>
```

```css
body {
    margin: 0; padding: 12px;
    font: 400 14px/1.5 "Open Sans", sans-serif;
    color: var(--dark, #1D1E32);
    background: transparent;  /* ← */
}

/* Здесь вся атмосфера */
.game-frame {
    background: linear-gradient(160deg, #fffdf4 0%, #fff5d7 100%);
    border: 1px solid rgba(198, 146, 20, 0.35);
    border-radius: 14px;
    padding: 16px;
    box-shadow: 0 4px 16px rgba(196, 146, 20, 0.12),
                inset 0 0 0 1px rgba(255, 255, 255, 0.7);
}
```

Эталон — `etalon-apps/coin-clicker/styles/style.css` (`.cc-frame`).

---

## Правило №1.5 — раскладка canvas-игры: центрированная колонка

iframe миниапа **авто-ресайзится под высоту контента** (платформа репортит высоту body). Для canvas-игр (Flappy, арканоиды) из этого следуют грабли:

- **Не делай `#app { height: 100% }` + `.canvas-wrap { flex: 1 }`.** Контейнер растянется на всю высоту iframe, а canvas останется фиксированной высоты. Оверлеи game-over/start (`position: absolute; inset: 0`) покроют весь растянутый контейнер и окажутся **выше канваса** (оверлей 980px, поле 640px — классический баг).
- **Оборачивай всё в центрированную колонку** (как coin-clicker):
  ```css
  #app { max-width: 380px; margin: 0 auto; }
  .canvas-wrap { position: relative; /* НЕ flex:1 */ }
  ```
  Тогда header, canvas и оверлеи — одной ширины. Поле — аккуратная портретная колонка, а не 320px, потерянные в 1000px-строке на десктопе.
- **Canvas заполняет ширину колонки**, высота — по аспекту (`ch = cw * ratio`). Контейнер по высоте равен канвасу — iframe сам подтянется под контент.
- Полноширинная раскладка делает портретный аспект (напр. 1.69) абсурдно высоким на десктопе — половина поля уезжает за экран. Узкая колонка решает и это.

---

## Правило №2 — Korfix CSS-переменные как основа

Цвета, типографика, кнопки — через платформенные tokens:

```css
:root {
    --primary:   #323C8F;   /* кнопки, ссылки */
    --primary-h: #2a327a;   /* hover/active border-bottom */
    --success:   #33BE2B;   /* ок/claim */
    --danger:    #EF233C;   /* ошибки, отрицательные */
    --dark:      #1D1E32;
    --gray2:     #78797f;   /* subtitle */
    --gray6:     #eaeaee;   /* разделители */
    --bluegray8: #dce0ef;   /* input border */
    --bluegray10: #f4f5fa;  /* фон карточки */
    --bluegray11: #f8f8fd;  /* подложка */
}
```

Игровые акценты поверх (gold для Korn, streak-orange):

```css
:root {
    --gold:     #f4c542;
    --gold-d:   #c69214;
    --gold-bg:  #fff3cd;
    --gold-txt: #856404;
    --streak:   #ff6b35;
}
```

**Полный список tokens** — [miniapps/styling.md § CSS-переменные](../miniapps/styling.md).

---

## Правило №3 — подложки не должны быть пустыми квадратами

Каждый блок — с мягкой тенью или рамкой. Пустая белая карточка в пустом сером фоне выглядит недоделанной.

```css
/* Карточка платформенная — минимум что должно быть */
.card {
    background: #fff;
    border-radius: 8px;
    padding: 16px;
    box-shadow: 0 1px 3px rgba(0,0,0,.08);
}

/* Игровая карточка — добавляем акцент */
.game-card {
    background: #fff;
    border-radius: 10px;
    padding: 16px;
    border: 1px solid rgba(198, 146, 20, 0.2);
    box-shadow: 0 2px 8px rgba(196, 146, 20, 0.08);
}

/* "Pill" контейнер (для баланса, стрика) */
.pill {
    display: inline-flex; align-items: center; gap: 8px;
    padding: 8px 14px; border-radius: 24px;
    background: linear-gradient(135deg, var(--gold-bg), #ffe28a);
    color: var(--gold-txt); font-weight: 600;
    border: 1px solid rgba(198, 146, 20, 0.25);
    box-shadow: 0 2px 6px rgba(196, 146, 20, 0.15);
}
```

---

## Правило №4 — кнопки с Korfix-паттерном

Округлые углы **3px** (не 6/8), `border-bottom: 3px solid darker-color` для визуального «нажатия»:

```css
.btn {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 8px 16px;
    border: 0; border-radius: 3px;
    font: 500 13px/1.4 "Open Sans", sans-serif;
    cursor: pointer; transition: .15s;

    background: var(--primary);
    color: #fff;
    border-bottom: 3px solid var(--primary-h);
}
.btn:hover { opacity: .92; }
.btn:active { transform: translateY(1px); border-bottom-width: 2px; }
.btn:disabled { opacity: .5; cursor: not-allowed; }

/* Успешная (claim, save) */
.btn-success {
    background: var(--success);
    border-bottom-color: #2B9C5B;
}

/* Вторичная (secondary, back) */
.btn-secondary {
    background: #fff; color: var(--primary);
    border: 1px solid var(--bluegray8);
    border-bottom: 3px solid var(--bluegray8);
}
.btn-secondary:hover { border-color: var(--primary); }

/* Золотая (акцент для Korn-related) */
.btn-gold {
    background: var(--gold); color: #5a3f00;
    border-bottom: 3px solid var(--gold-d);
}
```

---

## Правило №5 — интерактивная отзывчивость

Любой кликабельный элемент должен давать визуальный feedback при нажатии:

```css
.pill-button {
    cursor: pointer;
    transition: transform .08s, box-shadow .12s;
}
.pill-button:hover {
    box-shadow: 0 4px 10px rgba(196, 146, 20, 0.25);
}
.pill-button:active {
    transform: translateY(1px);
    box-shadow: 0 1px 3px rgba(196, 146, 20, 0.2) inset;
}

/* Игровая монетка/объект */
.clickable-object { transition: transform .1s; }
.clickable-object:active { transform: scale(0.92); }
.clickable-object.bumped { animation: bump .1s; }
@keyframes bump { 50% { transform: scale(1.2) rotate(10deg); } }
```

Без этого игра ощущается «мёртвой».

---

## Правило №6 — цифры моноширинным шрифтом

Score, таймер, суммы — `"SF Mono"`, `Menlo`, `monospace`. Не прыгают при изменении.

```css
.score, .timer, .balance-amount, .transaction-amt {
    font: 700 16px/1 "SF Mono", Menlo, Monaco, "Liberation Mono", monospace;
    color: var(--gold-txt);
}
```

---

## Правило №7 — модалки с блюром

Для overlay-модалок (game over, shop, top scores) — полупрозрачный фон + blur:

```css
.modal {
    position: fixed; inset: 0;
    background: rgba(0, 0, 0, 0.55);
    backdrop-filter: blur(3px);
    display: flex; align-items: center; justify-content: center;
    padding: 20px; z-index: 100;
}
.modal.hidden { display: none; }
.modal-content {
    background: #fff; padding: 22px; border-radius: 12px;
    max-width: 400px; width: 100%; max-height: 80vh; overflow-y: auto;
    box-shadow: 0 10px 40px rgba(0, 0, 0, 0.25);
}
```

---

## Правило №8 — аватары круглые, с border

```css
.avatar {
    width: 44px; height: 44px; border-radius: 50%;
    object-fit: cover; background: var(--bluegray10);
    border: 2px solid var(--gold-d);  /* или --bluegray8 для не-gamedev */
    flex-shrink: 0;
}

/* Fallback для пустого аватара */
.avatar.avatar-fallback {
    background: radial-gradient(circle at 30% 30%, var(--gold), var(--gold-d));
    position: relative;
}
.avatar.avatar-fallback::after {
    content: '?';
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center;
    color: #5a3f00; font: 700 22px/1 "Open Sans", sans-serif;
}
```

---

## Правило №9 — drag-drop зона для upload

Визуальный feedback dashed border → hover → dragover-glow:

```css
.drop-zone {
    width: 120px; height: 120px; border-radius: 50%;
    background: var(--bluegray10);
    border: 2px dashed var(--bluegray8);
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; transition: .15s;
}
.drop-zone:hover {
    border-color: var(--primary); background: var(--bluegray11);
}
.drop-zone.dragover {
    border-color: var(--primary);
    background: #eef0fc;
    box-shadow: 0 0 0 4px rgba(50, 60, 143, 0.1);  /* glow */
}
```

JS:
```js
['dragenter','dragover'].forEach(ev => zone.addEventListener(ev, e => {
    e.preventDefault(); zone.classList.add('dragover');
}));
['dragleave','drop'].forEach(ev => zone.addEventListener(ev, e => {
    e.preventDefault(); zone.classList.remove('dragover');
}));
```

---

## Правило №10 — i18n-ready тексты

Все user-facing строки через `i18n.t('key')`. Даже если поддерживается только EN сейчас — завтра попросят RU, будет дешевле.

```html
<button data-i18n="game.start">Start</button>
<h1 data-i18n="app.title">My Game</h1>
<input placeholder="" data-i18n-attr="placeholder:form.hint">
```

```js
i18n.applyToDom(document);  // после init и после setLang
```

См. [project-structure.md § i18n](project-structure.md).

---

## Чеклист стилей перед деплоем

- [ ] `body { background: transparent }`
- [ ] Вся игровая атмосфера — внутри `.*-frame` или аналога
- [ ] CSS-переменные Korfix в `:root`
- [ ] Кнопки в Korfix-паттерне (`border-radius: 3px`, `border-bottom: 3px solid darker`)
- [ ] Все кнопки имеют `:hover` и `:active` состояния
- [ ] Score/таймер/цифры моноширинным шрифтом
- [ ] Модалки — `backdrop-filter: blur`
- [ ] Аватары — круглые, с fallback
- [ ] Upload-зона — dashed border + dragover glow
- [ ] Подложки с box-shadow, не плоские
- [ ] i18n готовность — `data-i18n` атрибуты

---

## Ссылки

- [miniapps/styling.md](../miniapps/styling.md) — база (что платформа даёт и ожидает)
- [etalon-apps/coin-clicker/styles/style.css](../../etalon-apps/coin-clicker/styles/style.css) — эталон игрового стиля (у тебя в проекте)
- [etalon-apps/games-hub/styles/style.css](../../etalon-apps/games-hub/styles/style.css) — эталон hub-стиля (табы, лидерборд, профиль)
