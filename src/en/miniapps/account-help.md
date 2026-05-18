# Help Content (account_help, service_help)

> **See also:** [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md) · [favorites-menu.md](favorites-menu.md)
> **← [Home](index.md)**

Two catalogs for contextual help: instructions for catalogs
and reference information on pages.

---

## account_help — Catalog Instructions

Instructions are tied to specific platform catalogs. They appear
as a sticky-note button in the catalog header — clicking opens a modal
with the instruction content.

### Record Structure

| Field | Type | Description |
|-------|------|-------------|
| `name` | textbox | Instruction title |
| `aliasdb` | select | Catalog the instruction is attached to (e.g. `tt_tasks`) |
| `is_main` | checkbox | Default instruction (if multiple exist for a catalog — the one with `is_main=1` is shown) |
| `cont` | textarea (wysiwyg) | Instruction text (HTML) |
| `doc` | photo | Illustration |
| `doc1` | file | Attached file |
| `youtube` | textbox | YouTube video link |

Access: by group (`from_group`). Multiple instructions per catalog are allowed.

### How It Works in the UI

When opening any catalog, an icon appears in the header:
- **Filled sticky note** (fas fa-sticky-note) — instruction exists, click opens it
- **Empty sticky note** (far fa-sticky-note) — no instruction, click creates one

### Reading Instructions

```js
// Get all instructions
const resp = await App.fetch('/db/account_help.json');

// Instructions for a specific catalog
const resp = await App.fetch('/db/account_help.json?form[aliasdb]=tt_tasks');
const instruction = resp.data[0];
console.log(instruction.name);     // "How to work with tasks"
console.log(instruction.cont);     // HTML content
console.log(instruction.youtube);  // "abcd" or full URL
```

### Creating an Instruction from a Miniapp

```js
// Create an instruction for the orders catalog
await App.fetch('/db/account_help/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'How to place an order',
        'form[aliasdb]': 'b2b_orders',
        'form[is_main]': 1,
        'form[cont]': '<h3>Step 1</h3><p>Open the orders catalog...</p>',
        'form[youtube]': 'https://youtu.be/abcd',
        submit: 1
    }
});
```

### Getting the Catalog List

```js
// Available catalogs for attaching an instruction
const schema = await App.fetch('/db/account_help/sheme.json');
const catalogs = schema.data.aliasdb.arr;
// { "tt_tasks": "Tasks (tt_tasks)", "b2b_orders": "Orders (b2b_orders)", ... }
```

---

## service_help — Page Reference Information

Help items are tied to page URLs. They appear as a "?" button
in the page header.

### Record Structure

| Field | Type | Description |
|-------|------|-------------|
| `name` | textbox | Help title |
| `show_on_pages` | textarea | Page URLs to show on (one per line) |
| `cont` | textarea (tinymce) | Help text (HTML with WYSIWYG editor) |
| `doc` | photo | Illustration |
| `doc1` | file | Attached file |
| `youtube` | textbox | YouTube video link |

### How It Works in the UI

Help items are loaded via `cmd('db/service_help')` in each catalog's footer.
If a help item exists for the current page — a "?" button (fa-question-circle) appears
in the header; clicking opens a modal with content, image, and video.

### Reading Help Items

```js
// All help items
const resp = await App.fetch('/db/service_help.json');

// Help items containing a specific URL
const resp = await App.fetch('/db/service_help.json?form[show_on_pages]=/db/tt_tasks');
```

### Creating a Help Item

```js
await App.fetch('/db/service_help/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'How to use filters',
        'form[show_on_pages]': '/db/tt_tasks\n/db/b2b_orders',
        'form[cont]': '<p>Use the filter panel...</p>',
        submit: 1
    }
});
```

---

## Use Cases for Miniapps

### Auto-create instruction on install

An app can create an instruction for the catalog it works with:

```js
async function createAppInstruction(catalog, title, content, videoUrl) {
    // Check if an instruction already exists from our app
    const resp = await App.fetch(`/db/account_help.json?form[aliasdb]=${catalog}`);
    if (resp.data.length > 0) return; // already exists

    await App.fetch('/db/account_help/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': title,
            'form[aliasdb]': catalog,
            'form[cont]': content,
            'form[youtube]': videoUrl || '',
            submit: 1
        }
    });
}
```

### Contextual help inside the miniapp

A miniapp can read existing instructions and display them
in its own UI, e.g. as tooltips or inline hints:

```js
async function getHelpForCatalog(catalog) {
    const resp = await App.fetch(`/db/account_help.json?form[aliasdb]=${catalog}`);
    return resp.data; // array of instructions
}
```

### Knowledge base

An app can build a navigable knowledge base from all instructions:

```js
const all = await App.fetchAll('/db/account_help.json');
// Group by catalog
const byModule = {};
all.data.forEach(item => {
    if (!byModule[item.aliasdb]) byModule[item.aliasdb] = [];
    byModule[item.aliasdb].push(item);
});
```

---

## Differences Between Catalogs

| | account_help | service_help |
|---|---|---|
| Bound to | Catalog (`aliasdb`) | Page URL (`show_on_pages`) |
| Icon | Sticky note (fa-sticky-note) | Question mark (fa-question-circle) |
| Multiple | Yes, per catalog (priority: `is_main`) | Yes, per page |
| Editor | wysiwyg-simple | tinymce |
| Access | By group | By group |
| Purpose | User instructions | Reference content from admin |

---

*Catalogs: `/db/account_help`, `/db/service_help`*

**← [Home](index.md)**
