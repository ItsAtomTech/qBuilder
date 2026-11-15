# qBuilder

> A lightweight JavaScript utility for building, managing, and sending paginated HTTP queries with filter and state persistence support.

**Version:** 1.3  
**Type:** Vanilla JavaScript (no dependencies)  
**Typical Use Case:** Table/list views with search, filters, sorting, and pagination — commonly paired with a PHP or Python (Flask) backend.

---

## Table of Contents

- [Overview](#overview)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [API Reference](#api-reference)
  - [Core Methods](#core-methods)
  - [Utility Methods](#utility-methods)
  - [Helper Functions](#helper-functions)
- [Usage Examples](#usage-examples)
- [Custom Parameters](#custom-parameters)
- [Error Handling](#error-handling)
- [Notes](#notes)

---

## Overview

`qBuilder` is a singleton object that abstracts away the repetitive boilerplate of building `FormData` payloads, managing pagination state, and sending `XMLHttpRequest` calls. It also provides optional localStorage-based state persistence, a human-readable filter formatter, and a lightweight DOM data-binding utility.

---

## Getting Started

Include the script in your HTML file:

```html
<script src="request_builder.js"></script>
```

Set the server address and fire your first query:

```javascript
qBuilder.server_address = '/api/records';
qBuilder.sendQuery(function(response) {
    let data = JSON.parse(response.responseText);
    console.log(data);
});
```

---

## Configuration

These properties can be set on `qBuilder` before calling `sendQuery()`:

| Property | Type | Default | Description |
|---|---|---|---|
| `server_address` | `string` | `undefined` | **Required.** The endpoint URL to send requests to. |
| `request_method` | `string` | `'POST'` | HTTP method. Accepts `'POST'` or `'GET'`. |
| `page` | `number` | `1` | Current page number. |
| `maxPage` | `number` | `-1` | Maximum page limit. `-1` means no upper bound. |
| `row_limit` | `number` | `100` | Number of rows to fetch per request. |
| `returnType` | `string` | `'normal'` | Response format hint sent to the server. |
| `search` | `string` | `undefined` | Search keyword. Omitted from payload if not set. |
| `sort` | `string` | `undefined` | Sort column. Omitted from payload if not set. |
| `order_by` | `string` | `undefined` | Sort direction (e.g. `'asc'`, `'desc'`). Omitted if not set. |
| `filters` | `object` | `{}` | Key-value filter pairs. Always included as a JSON string in the payload. |

---

## API Reference

### Core Methods

#### `sendQuery(func, customURL, customParams, errorHandler)`

Builds a `FormData` payload from the current state and sends it via `XMLHttpRequest`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `func` | `function` | `undefined` | Callback invoked on a successful response (`status 200`). Receives the XHR object. You should JSON parse the response Text etc. |
| `customURL` | `string` | `undefined` | Overrides `server_address` for this request only. |
| `customParams` | `array` | `[]` | Array of `{ name, value }` or `{ name, data }` objects appended to the payload. |
| `errorHandler` | `function` | `undefined` | Callback invoked on `status 0` or `status 404`. |

```javascript
qBuilder.sendQuery(
    function(res) { console.log(JSON.parse(res.responseText)); },
    undefined,
    [{ name: 'extra_flag', value: 1 }],
    function(err) { console.error('Request failed', err); }
);
```

Or alternative callback and is more easy to understand:

```javascript
let params = [{ name: 'extra_flag', value: 1 }];

qBuilder.sendQuery(
    myFunction,
    undefined,
    params,
    function(err) { console.error('Request failed', err); }
);

function myFunction(data){
	//Do Something --


}

```

---

#### `sendPromise(func, customURL, customParams, errorHandler)`

A Promise-based wrapper around `sendQuery()`. Resolves on success, rejects on error.

```javascript
qBuilder.sendPromise(function(res) {
    let data = JSON.parse(res.responseText);
    renderTable(data);
}).then(() => {
    console.log('Done');
}).catch(err => {
    console.error('Failed:', err);
});

// Or with async/await:
let res = await qBuilder.sendPromise();
let data = JSON.parse(res.responseText);
```

---

#### `paginate(dir, send)`

Increments or decrements the current page, clamped within `[1, maxPage]`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `dir` | `string` | — | `'prev'` or `'next'` |
| `send` | `boolean` | `false` | If `true`, automatically calls `sendQuery()` after changing the page. |

**Returns:** The new page number.

```javascript
// Move to next page and send immediately
qBuilder.paginate('next', true);
```

---

### Utility Methods

#### `limitRange(value, min, max)`

Clamps a value between a minimum and maximum.

| Parameter | Type | Description |
|---|---|---|
| `value` | `number` | Value to clamp. |
| `min` | `number` | Minimum allowed value. |
| `max` | `number` | Maximum allowed value. `-1` disables the upper bound. |

```javascript
qBuilder.limitRange(150, 1, 100); // Returns 100
qBuilder.limitRange(0, 1, 100);   // Returns 1
```

---

#### `saveParamsLocal(params)`

Saves selected `qBuilder` properties to `localStorage` under the key `'paramStored'`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `params` | `array` | `['page', 'sort', 'search', 'filters', 'order_by']` | List of property names to save. |

```javascript
qBuilder.saveParamsLocal(); // Saves all defaults
qBuilder.saveParamsLocal(['page', 'filters']); // Save specific keys only
```

---

#### `loadParamsLocal(params, retain)`

Restores properties from `localStorage`. Removes the stored entry after loading unless `retain` is `true`.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `params` | `array` | `['page', 'sort', 'search', 'filters', 'order_by']` | List of property names to restore. |
| `retain` | `boolean` | `false` | If `true`, does not remove the stored entry after loading. |

**Returns:** `true` if data was found and loaded, `false` otherwise.

```javascript
let loaded = qBuilder.loadParamsLocal();
if (loaded) {
    loadDataBind(qBuilder); // Sync UI
}
```

---

#### `formatFilters(filters, parentKey, returnKeys)`

Converts the `filters` object into a human-readable array. Skips `sort`, `strict_status`, and any filter with the value `'all'`. Supports nested filter objects.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `filters` | `object` | `this.filters` | The filter object to format. |
| `parentKey` | `string` | `''` | Used internally for nested recursion. |
| `returnKeys` | `boolean` | `false` | If `true`, returns `{ key, value }` objects instead of plain strings. |

```javascript
qBuilder.filters = { status: 'active', college: 'engineering' };

qBuilder.formatFilters();
// Returns: ['Status: Active', 'College: Engineering']

qBuilder.formatFilters(qBuilder.filters, undefined, true);
// Returns: [{ key: 'status', value: 'Status: Active' }, ...]
```

---

### Helper Functions

These are standalone functions defined outside `qBuilder`.

---

#### `monitorChanges(key, loader)`

Polls `localStorage` every 2 seconds for a reload signal. When the key value is `'true'`, it triggers a `sendQuery()` call and removes the flag. Useful for cross-tab or cross-page reload signaling.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `key` | `string` | `'shouldReload'` | The `localStorage` key to watch. |
| `loader` | `function` | `undefined` | Callback passed to `sendQuery()`. Falls back to `loadRecords` if defined globally. |

```javascript
monitorChanges('shouldReload', loadRecords);
```

---

#### `accessProperty(obj, propString)`

Safely accesses a deeply nested object property using dot notation.

```javascript
let val = accessProperty(responseData, 'student.info.name');
// Equivalent to: responseData?.student?.info?.name
```

---

#### `loadDataBind(objRoot)`

Finds all elements with a `[databind]` attribute in the DOM and sets their `.value` to the matching property on `objRoot` using dot notation.

```html
<input databind="page" />
<input databind="filters.status" />
```

```javascript
loadDataBind(qBuilder); // Fills inputs with current qBuilder state
```

---

#### `loadSavedParams(retain)`

Calls `loadParamsLocal()` and, if successful, syncs the UI via `loadDataBind()`.

```javascript
loadSavedParams(false); // Load and clear localStorage
loadSavedParams(true);  // Load and keep localStorage entry
```

---

#### `removeFilter(key)`

Deletes a key from `qBuilder.filters`, rebinds the UI, and optionally refreshes filter ribbons if `filter_apply_list()` and `generateFilterRibbons()` are defined.

```javascript
removeFilter('status'); // Removes the 'status' filter and updates the UI
```

---

## Usage Examples

### Basic Paginated Table

```javascript
qBuilder.server_address = '/api/students';
qBuilder.row_limit = 25;
qBuilder.filters = { college: 'engineering', status: 'active' };

qBuilder.sendQuery(function(res) {
    let data = JSON.parse(res.responseText);
    renderTable(data.records);
    qBuilder.maxPage = data.max_page;
});
```

### Search with Filters

```javascript
qBuilder.search = document.getElementById('searchInput').value;
qBuilder.filters.status = 'on_probation';
qBuilder.page = 1;
qBuilder.sendQuery(loadRecords);
```

### Async/Await with sendPromise

```javascript
async function fetchRecords() {
    try {
        let res = await qBuilder.sendPromise();
        let data = JSON.parse(res.responseText);
        renderTable(data.records);
    } catch (err) {
        console.error('Fetch failed:', err);
    }
}
```

### Preserve State Across Navigation

```javascript
// Before leaving the page
qBuilder.saveParamsLocal();

// On return
loadSavedParams(false);
qBuilder.sendQuery(loadRecords);
```

---

## Custom Parameters

Extra parameters can be passed per request without permanently modifying `qBuilder`'s state:

```javascript
qBuilder.sendQuery(loadRecords, undefined, [
    { name: 'export_format', value: 'csv' },
    { name: 'include_archived', data: true }
]);
```

Both `value` and `data` keys are supported for the parameter value.

---

## Error Handling

`sendQuery` handles three response states:

| Status | Behavior |
|---|---|
| `200` | Calls `func(xhr)` with the XHR object. |
| `0` | Calls `createDialogue('error', ...)` and `errorHandler(xhr)` if provided. |
| `404` | Calls `createDialogue('error', ...)` and `errorHandler(xhr)` if provided. |

> **Note:** `createDialogue()` is expected to be defined externally in your project.

---

## Notes

- `qBuilder` is a **singleton object literal**. If multiple independent query builders are needed on the same page, consider refactoring it into a class or factory function.
- `saveParamsLocal` and `loadParamsLocal` rely on `localStorage`, which is **not available** in some server-side rendering or private browsing contexts.
- The `formatFilters` method calls `preProcessEvent()` internally if it is defined globally — this is an optional hook for custom filter value transformations.
- `monitorChanges` starts an interval that runs indefinitely. Store the return value if you need to stop it later via `clearInterval()`.
