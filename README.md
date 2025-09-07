# 📊 ITable Component (Vue 3 + TypeScript + Laravel Ready)

A powerful, customizable data table component for **Vue 3** with features like search, sort, pagination, export, print, and column visibility.  
Optimized for **client-side** and **server-side (Laravel API)** usage.

---

## ✨ Features

- 🔍 **Search** with debounce  
- ↕️ **Sorting** (text, number, date)  
- 📑 **Pagination** (client & server side)  
- 📤 **Excel Export**  
- 🖨️ **Print support** (hide certain columns when printing)  
- 👁️ **Column visibility toggle** (with `localStorage` persistence)  
- 📌 **Sticky header / sticky first column**  
- ✔️ **Row selection** with bulk select  
- ⚡ **Server-side mode** (emit events for Laravel API)  
- 🎨 **Custom slots** for cells, toolbar actions, pagination, etc.  

---

## 📦 Installation

```bash
npm install xlsx
# or
yarn add xlsx
```

Place `ITable.vue` into your `components` folder.

---

## 🔧 Props

| Prop                | Type                 | Default   | Description |
|---------------------|----------------------|-----------|-------------|
| `headers`           | `ITableHeader[]`     | **req.**  | Column definitions |
| `items`             | `ITableItem[]`       | **req.**  | Table rows |
| `title`             | `string`             | `""`      | Title for print/export |
| `caption`           | `string`             | `""`      | Caption above table |
| `pageSize`          | `number`             | `10`      | Rows per page |
| `pageSizeOptions`   | `number[]`           | `[10,25,50,100]` | Dropdown options |
| `showSearch`        | `boolean`            | `true`    | Show search box |
| `showExportButton`  | `boolean`            | `true`    | Show Excel export |
| `showPrintButton`   | `boolean`            | `true`    | Show print button |
| `showColumnsButton` | `boolean`            | `true`    | Show column toggle |
| `showPagination`    | `boolean`            | `true`    | Show pagination |
| `persistKey`        | `string`             | `""`      | Save visible columns in `localStorage` |
| `serverSide`        | `boolean`            | `false`   | If true, emits events instead of client filtering |
| `rowKey`            | `string \| function` | `""`      | Unique row identifier |
| `selectable`        | `boolean`            | `false`   | Enable row selection |
| `selectedKeys`      | `Array<string\|number>` | `[]`   | Selected rows |
| `stickyHeader`      | `boolean`            | `true`    | Keep header fixed |
| `stickyFirstColumn` | `boolean`            | `false`   | Keep first column fixed |
| `dense`             | `boolean`            | `false`   | Compact row height |
| `exportFilename`    | `string`             | `"export.xlsx"` | Export filename |
| `exportOnlyVisible` | `boolean`            | `true`    | Export only visible columns |
| `debounceSearchMs`  | `number`             | `250`     | Search debounce delay |

---

## 🎯 Events

| Event                | Payload                                |
|-----------------------|----------------------------------------|
| `row-click`           | `{ row, index }`                       |
| `columns-change`      | `string[]` (visible column keys)        |
| `search-change`       | `string`                               |
| `sort-change`         | `{ sortKey: string, ascending: bool }` |
| `update:currentPage`  | `number`                               |
| `update:pageSize`     | `number`                               |
| `update:selectedKeys` | `Array<string \| number>`              |

---

## 🖼️ Example: Client-side

```vue
<script setup lang="ts">
import ITable, { ITableHeader, ITableItem } from "@/components/ITable.vue";

const headers: ITableHeader[] = [
  { caption: "ID", value: "id", sortable: true, visible: true },
  { caption: "Name", value: "name", sortable: true },
  { caption: "Email", value: "email" },
  { caption: "Created At", value: "created_at", sortable: true, print: true },
];

const rows: ITableItem[] = [
  { id: 1, name: "Alice", email: "alice@example.com", created_at: "2025-01-01" },
  { id: 2, name: "Bob", email: "bob@example.com", created_at: "2025-02-15" },
];

const selectedIds = ref<number[]>([]);
</script>

<template>
  <ITable
    :headers="headers"
    :items="rows"
    title="Users"
    caption="User Directory"
    :page-size="10"
    show-row-number
    selectable
    row-key="id"
    v-model:selectedKeys="selectedIds"
    @row-click="({ row }) => console.log('Clicked row:', row)"
  >
    <!-- Custom slot for email column -->
    <template #email="{ value }">
      <a :href="`mailto:${value}`" class="text-blue-600 underline">{{ value }}</a>
    </template>
  </ITable>
</template>
```

---

## 🖥️ Example: Server-side (Laravel API)

```vue
<template>
  <ITable
    server-side
    :headers="headers"
    :items="users.data"
    :page-size="query.pageSize"
    :show-pagination="false"
    @search-change="applySearch"
    @sort-change="applySort"
    @update:currentPage="goToPage"
    @update:pageSize="setPageSize"
  >
    <template #serverPager>
      <YourPager :meta="users.meta" @page="goToPage" />
    </template>
  </ITable>
</template>

<script setup lang="ts">
const headers = [
  { caption: "ID", value: "id" },
  { caption: "Name", value: "name" },
  { caption: "Email", value: "email" },
];

const users = ref({ data: [], meta: {} });
const query = reactive({ page: 1, pageSize: 25, search: "", sort: "id", dir: "asc" });

function applySearch(search: string) {
  query.search = search;
  fetchUsers();
}
function applySort({ sortKey, ascending }: any) {
  query.sort = sortKey;
  query.dir = ascending ? "asc" : "desc";
  fetchUsers();
}
function goToPage(page: number) {
  query.page = page;
  fetchUsers();
}
function setPageSize(size: number) {
  query.pageSize = size;
  query.page = 1;
  fetchUsers();
}
function fetchUsers() {
  // Call Laravel API with query params
}
</script>
```

---

## 🎨 Slots

- `#headerTitle` → Content above table header  
- `#toolbar-left` → Left toolbar section  
- `#toolbar-right` → Right toolbar section  
- `#serverPager` → Custom pagination for server-side mode  
- `#[columnKey]` → Custom cell rendering per column  

Example:

```vue
<template #name="{ value }">
  <strong>{{ value }}</strong>
</template>
```

---

## 📌 Tips

- Use `persistKey="users-table"` to remember visible columns across reloads.  
- Always set `rowKey="id"` (or a function) for stable selection & rendering.  
- Enable `selectable` for checkboxes & bulk actions.  
- Use `stickyHeader` for long scrollable tables.  
- For large datasets, use **server-side mode** with Laravel pagination.  

---

## 📝 License

MIT
