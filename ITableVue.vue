<script setup lang="ts">
import { ref, computed, watch, onMounted, type CSSProperties } from "vue";
import type { PropType, VNode } from "vue";
 
// ============ Types ============
export interface ITableItem {
  [key: string]: unknown;
}

export interface ITableHeader {
  caption: string;
  value: string; // column key
  visible?: boolean; // default: true
  sortable?: boolean; // default: true
  sortKey?: string;   // if provided, use this key for sorting
  print?: boolean;    // default: true
  width?: string;     // e.g. "160px" or "12rem"
  align?: "left" | "center" | "right";
  // Optional logical formatter when you do not want to use a slot
  format?: (row: ITableItem, cellValue: unknown) => string | number | VNode;
}

// ============ Props & Emits ============
const props = defineProps({
  headers: { type: Array as PropType<ITableHeader[]>, required: true },
  items: { type: Array as PropType<ITableItem[]>, required: true },

  title: { type: String, default: "" },
  caption: { type: String, default: "" },

  showNoData: { type: Boolean, default: false },
  titleNoData: { type: String, default: "noResults" },

  showRowNumber: { type: Boolean, default: false },
  showPrintButton: { type: Boolean, default: true },
  showExportButton: { type: Boolean, default: true },
  showColumnsButton: { type: Boolean, default: true },
  showSearch: { type: Boolean, default: true },
  showPagination: { type: Boolean, default: true },

  pageSize: { type: Number, default: 10 },
  pageSizeOptions: {
    type: Array as PropType<number[]>,
    default: () => [10, 25, 50, 100],
  },

  // Persist user column visibility; if empty string, persistence is off
  persistKey: { type: String, default: "" },

  // If true, we do not filter/sort/page on the client; we only emit events
  serverSide: { type: Boolean, default: false },

  // Unique key for a row (string key name or function)
  rowKey: {
    type: [String, Function] as PropType<
      string | ((row: ITableItem, index: number) => string | number)
    >,
    default: "",
  },

  dense: { type: Boolean, default: false },
  stickyHeader: { type: Boolean, default: true },
  stickyFirstColumn: { type: Boolean, default: false },

  // Export options
  exportFilename: { type: String, default: "" },
  exportOnlyVisible: { type: Boolean, default: true },

  // Search debounce
  debounceSearchMs: { type: Number, default: 250 },

  // Row selection
  selectable: { type: Boolean, default: false },
  selectedKeys: {
    type: Array as PropType<Array<string | number>>,
    default: () => [],
  },
});

const emit = defineEmits<{
  (e: "update:selectedKeys", value: Array<string | number>): void;
  (e: "update:currentPage", value: number): void;
  (e: "update:pageSize", value: number): void;
  (e: "columns-change", value: string[]): void;
  (e: "search-change", value: string): void;
  (e: "sort-change", payload: { sortKey: string; ascending: boolean }): void;
  (e: "row-click", payload: { row: ITableItem; index: number }): void;
}>();

// ============ State ============
const rawSearchText = ref("");
const searchQuery = ref(""); // debounced
const currentPage = ref(1);
const currentPageSize = ref(props.pageSize);
const sortKey = ref("");
const sortAscending = ref(true);

const showColumnPopup = ref(false);
const visibleColumns = ref<string[]>([]);
const selectedRowKeys = ref<Array<string | number>>([...props.selectedKeys]);

// Initialize visible columns from header.visible (default true) or persisted value
function buildInitialVisibleColumns(headers: ITableHeader[]): string[] {
  return headers.filter(h => h.visible !== false).map(h => h.value);
}
function persistenceStorageKey(): string {
  return `ITable/visibleColumns/${props.persistKey || props.title || "default"}`;
}
watch(
  () => props.headers,
  (headers) => {
    if (props.persistKey) {
      const persisted = localStorage.getItem(persistenceStorageKey());
      if (persisted) {
        try {
          visibleColumns.value = JSON.parse(persisted);
          return;
        } catch {
          // ignore JSON parse error and fall back to headers
        }
      }
    }
    visibleColumns.value = buildInitialVisibleColumns(headers || []);
  },
  { immediate: true, deep: true }
);

watch(
  visibleColumns,
  (value) => {
    if (props.persistKey) {
      localStorage.setItem(persistenceStorageKey(), JSON.stringify(value));
    }
    emit("columns-change", value);
  },
  { deep: true }
);

// Keep selected keys in sync if parent controls them
watch(
  () => props.selectedKeys,
  (next) => (selectedRowKeys.value = [...next]),
  { deep: true }
);

// Debounce search input
let debounceTimer: number | undefined;
watch(
  rawSearchText,
  (text) => {
    if (debounceTimer) window.clearTimeout(debounceTimer);
    debounceTimer = window.setTimeout(() => {
      searchQuery.value = text.trim();
      resetPage();
      emit("search-change", searchQuery.value);
    }, props.debounceSearchMs);
  },
  { immediate: true }
);

// Reset page if items list size changes (client mode)
watch(
  () => props.items.length,
  () => {
    if (!props.serverSide) resetPage();
  }
);

// Clamp page when totals shrink
watch(
  () => [currentPageSize.value, () => filteredItems.value.length],
  () => {
    if (props.serverSide) return;
    const pages = totalPages.value || 1;
    if (currentPage.value > pages) currentPage.value = pages;
  },
  { deep: true }
);

// ============ Helpers ============
const stringNumberCollator = new Intl.Collator(undefined, {
  numeric: true,
  sensitivity: "base",
});

function getCellValue(row: ITableItem, keyPath: string): unknown {
  // dot.path support
  return keyPath.split(".").reduce<unknown>((acc: any, part) => acc?.[part], row);
}

function compareValues(a: unknown, b: unknown): number {
  // nulls last
  if (a == null && b == null) return 0;
  if (a == null) return 1;
  if (b == null) return -1;

  // Date support
  const aTime =
    a instanceof Date ? a.getTime() : Number.isNaN(Date.parse(String(a))) ? NaN : Date.parse(String(a));
  const bTime =
    b instanceof Date ? b.getTime() : Number.isNaN(Date.parse(String(b))) ? NaN : Date.parse(String(b));
  if (!Number.isNaN(aTime) && !Number.isNaN(bTime)) return aTime - bTime;

  // Numeric fallback (handles "1,234.50", "$99", etc.)
  const numericA = typeof a === "number" ? a : Number(String(a).replace(/[^0-9.-]+/g, ""));
  const numericB = typeof b === "number" ? b : Number(String(b).replace(/[^0-9.-]+/g, ""));
  if (!Number.isNaN(numericA) && !Number.isNaN(numericB) && String(a).trim() !== "" && String(b).trim() !== "") {
    return numericA - numericB;
  }

  return stringNumberCollator.compare(String(a), String(b));
}

function ariaSortForHeader(header: ITableHeader): "none" | "ascending" | "descending" {
  if (sortKey.value !== (header.sortKey || header.value)) return "none";
  return sortAscending.value ? "ascending" : "descending";
}

function resetPage(): void {
  currentPage.value = 1;
  emit("update:currentPage", 1);
}

function toggleSortForColumn(header: ITableHeader): void {
  if (header.sortable === false) return;
  const key = header.sortKey || header.value;
  if (sortKey.value === key) {
    sortAscending.value = !sortAscending.value;
  } else {
    sortKey.value = key;
    sortAscending.value = true;
  }
  emit("sort-change", { sortKey: sortKey.value, ascending: sortAscending.value });
  if (!props.serverSide) resetPage();
}

function toggleAllColumns(): void {
  if (visibleColumns.value.length === props.headers.length) {
    visibleColumns.value = [];
  } else {
    visibleColumns.value = props.headers.map((h) => h.value);
  }
}

function toggleColumnVisibility(key: string): void {
  if (visibleColumns.value.includes(key)) {
    visibleColumns.value = visibleColumns.value.filter((k) => k !== key);
  } else {
    visibleColumns.value = [...visibleColumns.value, key];
  }
}

// Row keys (for selection and v-for)
function getRowKey(row: ITableItem, index: number): string | number {
  if (typeof props.rowKey === "function") return props.rowKey(row, index);
  if (typeof props.rowKey === "string" && props.rowKey) return (row as any)[props.rowKey] ?? index;
  return index;
}

function onRowClick(row: ITableItem, index: number): void {
  emit("row-click", { row, index });
}

// Selection
const allRowsOnPageSelected = computed<boolean>(() => {
  if (!props.selectable) return false;
  if (paginatedItems.value.length === 0) return false;
  const pageKeys = paginatedItems.value.map((row, idx) =>
    getRowKey(row, idx + (currentPage.value - 1) * currentPageSize.value)
  );
  return pageKeys.every((k) => selectedRowKeys.value.includes(k));
});

function toggleSelectAllOnPage(): void {
  const pageKeys = paginatedItems.value.map((row, idx) =>
    getRowKey(row, idx + (currentPage.value - 1) * currentPageSize.value)
  );
  if (allRowsOnPageSelected.value) {
    selectedRowKeys.value = selectedRowKeys.value.filter((k) => !pageKeys.includes(k));
  } else {
    selectedRowKeys.value = Array.from(new Set([...selectedRowKeys.value, ...pageKeys]));
  }
  emit("update:selectedKeys", selectedRowKeys.value);
}

function toggleRowSelected(row: ITableItem, index: number): void {
  const key = getRowKey(row, index);
  if (selectedRowKeys.value.includes(key)) {
    selectedRowKeys.value = selectedRowKeys.value.filter((k) => k !== key);
  } else {
    selectedRowKeys.value = [...selectedRowKeys.value, key];
  }
  emit("update:selectedKeys", selectedRowKeys.value);
}

// ============ Derived data ============
const printableHeaders = computed<ITableHeader[]>(() => props.headers.filter((h) => h.print !== false));

const filteredItems = computed<ITableItem[]>(() => {
  if (props.serverSide) return props.items;
  const query = searchQuery.value.toLowerCase();
  if (!query) return props.items;
  return props.items.filter((row) =>
    visibleColumns.value.some((key) =>
      String(getCellValue(row, key) ?? "").toLowerCase().includes(query)
    )
  );
});

const sortedItems = computed<ITableItem[]>(() => {
  if (props.serverSide) return filteredItems.value;
  if (!sortKey.value) return [...filteredItems.value];
  const direction = sortAscending.value ? 1 : -1;
  return [...filteredItems.value].sort((a, b) => {
    const valueA = getCellValue(a, sortKey.value);
    const valueB = getCellValue(b, sortKey.value);
    return direction * compareValues(valueA, valueB);
  });
});

const paginatedItems = computed<ITableItem[]>(() => {
  if (props.serverSide || !props.showPagination) return sortedItems.value;
  const start = (currentPage.value - 1) * currentPageSize.value;
  return sortedItems.value.slice(start, start + currentPageSize.value);
});

const totalPages = computed<number>(() => {
  if (props.serverSide || !props.showPagination) return 0;
  return Math.max(1, Math.ceil(sortedItems.value.length / currentPageSize.value));
});

// Column widths (sample first N rows for perf)
const COLUMN_SAMPLE_COUNT = 25;
const columnStyleMap = computed<Record<string, CSSProperties>>(() => {
  const styleMap: Record<string, CSSProperties> = {};
  for (const header of props.headers) {
    if (header.width) {
      styleMap[header.value] = { width: header.width, whiteSpace: "nowrap" };
      continue;
    }
    const sample = props.items.slice(0, COLUMN_SAMPLE_COUNT).map((row) =>
      String(getCellValue(row, header.value) ?? "")
    );
    const maxLength = Math.max(header.value.length, ...sample.map((s) => s.length));
    const minWidthPx = maxLength * 8 + 24; // 8px/char + padding
    styleMap[header.value] = { minWidth: `${minWidthPx}px`, whiteSpace: "nowrap" };
  }
  return styleMap;
});
function getColumnStyle(key: string): CSSProperties {
  return columnStyleMap.value[key] || {};
}

// ============ Actions ============
async function exportToExcel(): Promise<void> {
  // async import to keep bundle small
  const XLSX = await import("xlsx");
  const source = props.exportOnlyVisible ? sortedItems.value : props.items;
  const columnKeys = props.exportOnlyVisible
    ? visibleColumns.value
    : props.headers.map((h) => h.value);

  const data = source.map((row) => {
    const record: Record<string, unknown> = {};
    columnKeys.forEach((key) => {
      const header = props.headers.find((h) => h.value === key);
      const headerCaption = header?.caption || key;
      record[headerCaption] = getCellValue(row, key) as unknown;
    });
    return record;
  });

  const worksheet = XLSX.utils.json_to_sheet(data);
  const workbook = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(workbook, worksheet, props.title || "Sheet1");
  const filename = props.exportFilename || `${props.title || "export"}.xlsx`;
  XLSX.writeFile(workbook, filename);
}

function printTable(): void {
  const tableHost = document.getElementById("printable-table");
  if (!tableHost) return;

  const tableClone = tableHost.cloneNode(true) as HTMLElement;

  // Remove columns with print:false
  const hiddenColumnIndexes: number[] = [];
  props.headers.forEach((header, index) => {
    if (header.print === false) {
      hiddenColumnIndexes.push(index + (props.showRowNumber ? 1 : 0) + (props.selectable ? 1 : 0));
    }
  });

  const headerRow = tableClone.querySelector("thead tr");
  if (headerRow) {
    [...hiddenColumnIndexes].reverse().forEach((colIndex) => {
      const cell = headerRow.children[colIndex];
      if (cell) cell.remove();
    });
  }
  const bodyRows = tableClone.querySelectorAll("tbody tr");
  bodyRows.forEach((row) => {
    [...hiddenColumnIndexes].reverse().forEach((colIndex) => {
      const cell = row.children[colIndex];
      if (cell) cell.remove();
    });
  });

  const popup = window.open("", "_blank");
  if (!popup) return;
  popup.document.write(`
    <!doctype html>
    <html>
    <head>
      <title>${props.title || "Table"}</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .print-caption { text-align: center; font-weight: bold; margin-bottom: 12px; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #333; padding: 8px; text-align: left; }
        th { background: #f5f5f5; }
        tr:nth-child(even) { background: #fafafa; }
        @media print {
          body { margin: 0; }
        }
      </style>
    </head>
    <body>
      ${props.caption || props.title ? `<div class="print-caption">${props.caption || props.title}</div>` : ""}
      ${tableClone.outerHTML}
    </body>
    </html>
  `);
  popup.document.close();
  popup.focus();
  setTimeout(() => {
    popup.print();
    popup.close();
  }, 250);
}
</script>

<template>
  <div class="table-container my-2">
    <div v-if="props.items.length">
      <!-- Controls -->
      <div class="mb-4 flex flex-wrap justify-between items-center gap-4">
        <div class="flex items-center gap-3">
          <!-- Search -->
          <div class="relative" v-if="showSearch">
            <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none">
              <svg class="h-4 w-4 text-gray-400 dark:text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"></path>
              </svg>
            </div>
            <input
              v-model="rawSearchText"
              :placeholder="t('search') || 'Search...'"
              class="block w-full pl-10 pr-3 py-2 border border-gray-300 dark:border-gray-600 rounded-lg bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100 placeholder-gray-500 dark:placeholder-gray-400 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
          </div>

          <!-- Column visibility -->
          <div class="relative" v-if="showColumnsButton">
            <button
              @click="showColumnPopup = !showColumnPopup"
              type="button"
              class="group relative inline-flex items-center gap-2 px-4 py-2 bg-gradient-to-r from-purple-200 to-blue-200 dark:from-purple-700 dark:to-blue-700 hover:from-purple-300 hover:to-blue-300 dark:hover:from-purple-800 dark:hover:to-blue-800 text-purple-800 dark:text-blue-100 font-medium rounded-lg shadow-lg hover:shadow-xl transform hover:-translate-y-0.5 transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-purple-300 dark:focus:ring-purple-700 focus:ring-offset-2 dark:focus:ring-offset-gray-800">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M9 17V7m0 10a2 2 0 01-2 2H5a2 2 0 01-2-2V7a2 2 0 012-2h2a2 2 0 012 2m0 10a2 2 0 002 2h2a2 2 0 002-2M9 7a2 2 0 012-2h2a2 2 0 012 2m0 10V7m0 10a2 2 0 002 2h2a2 2 0 002-2V7a2 2 0 00-2-2h-2a2 2 0 00-2 2">
                </path>
              </svg>
              <span class="text-sm font-semibold">{{ t("columns") || "Columns" }}</span>
              <svg class="w-3 h-3 transition-transform" :class="{ 'rotate-180': showColumnPopup }" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7"/>
              </svg>
            </button>

            <!-- Popup -->
            <div
              v-if="showColumnPopup"
              class="absolute top-full left-0 mt-2 bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-xl shadow-2xl z-50 min-w-72 backdrop-blur-sm"
            >
              <div class="p-5">
                <div class="mb-4 flex justify-between items-center">
                  <span class="text-sm text-gray-600 dark:text-gray-400">
                    {{ t("visible") }} {{ visibleColumns.length }} {{ t("of") }} {{ props.headers.length }}
                  </span>
                  <button
                    @click="toggleAllColumns"
                    class="text-sm font-medium text-blue-600 dark:text-blue-400 hover:text-blue-800 dark:hover:text-blue-300 transition-colors"
                  >
                    {{ visibleColumns.length === props.headers.length ? (t("hideAll") || "Hide All") : (t("showAll") || "Show All") }}
                  </button>
                </div>

                <div class="space-y-1 max-h-64 overflow-y-auto custom-scrollbar">
                  <label
                    v-for="header in props.headers"
                    :key="header.value"
                    class="flex items-center gap-3 p-3 hover:bg-gray-50 dark:hover:bg-gray-700 rounded-lg cursor-pointer transition-colors group"
                  >
                    <input
                      type="checkbox"
                      :value="header.value"
                      v-model="visibleColumns"
                      class="w-4 h-4 text-blue-600 border-2 border-gray-300 dark:border-gray-600 rounded focus:ring-blue-500 focus:ring-2 bg-white dark:bg-gray-700"
                    />
                    <span class="text-sm font-medium text-gray-700 dark:text-gray-300">
                      {{ header.caption }}
                      <span v-if="header.print === false"
                            class="ml-2 text-xs bg-red-100 dark:bg-red-900 text-red-600 dark:text-red-400 px-2 py-0.5 rounded-full">Print Hidden</span>
                    </span>
                  </label>
                </div>
              </div>
            </div>
          </div>

          <!-- Custom toolbar slot (right next to search/columns) -->
          <slot name="toolbar-left" />
        </div>

        <!-- Right actions -->
        <div class="flex items-center gap-3">
          <!-- Page size -->
          <div v-if="showPagination" class="flex items-center gap-2">
            <span class="text-sm text-gray-600 dark:text-gray-400">{{ t("rowsPerPage") || "Rows" }}</span>
            <select
              class="px-2 py-2 text-sm border border-gray-300 dark:border-gray-600 rounded-lg bg-white dark:bg-gray-800 dark:text-gray-100"
              v-model.number="currentPageSize"
              @change="emit('update:pageSize', currentPageSize); resetPage();"
            >
              <option v-for="size in pageSizeOptions" :key="size" :value="size">{{ size }}</option>
            </select>
          </div>

          <button
            v-if="showExportButton"
            @click="exportToExcel"
            class="bg-green-100 hover:bg-green-200 dark:bg-green-900 dark:hover:bg-green-800 text-green-700 dark:text-green-200 px-4 py-2 rounded-lg flex items-center gap-2 transition-all duration-200 hover:shadow-lg transform hover:-translate-y-0.5 focus:outline-none focus:ring-2 focus:ring-green-400 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
          >
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                d="M12 10v6m0 0l-3-3m3 3l3-3m2 8H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
            </svg>
            {{ t("exportExcel") || "Export Excel" }}
          </button>

          <button
            v-if="showPrintButton"
            @click="printTable"
            class="bg-blue-100 hover:bg-blue-200 dark:bg-blue-900 dark:hover:bg-blue-800 text-blue-700 dark:text-blue-200 px-4 py-2 rounded-lg flex items-center gap-2 transition-all duration-200 hover:shadow-lg transform hover:-translate-y-0.5 focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-offset-2 dark:focus:ring-offset-gray-800"
          >
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                d="M17 17h2a2 2 0 002-2v-4a2 2 0 00-2-2H5a2 2 0 00-2 2v4a2 2 0 002 2h2m2 4h6a2 2 0 002-2v-4a2 2 0 00-2-2H9a2 2 0 00-2 2v4a2 2 0 002 2zm8-12V5a2 2 0 00-2-2H9a2 2 0 00-2 2v4h10z"/>
            </svg>
            {{ t("print") || "Print" }}
          </button>

          <!-- Allow extra custom buttons -->
          <slot name="toolbar-right" />
        </div>
      </div>

      <!-- Printable Table Container -->
      <div
        id="printable-table"
        class="overflow-x-auto rounded-lg border border-gray-200 dark:border-gray-700"
        role="grid"
        :aria-rowcount="sortedItems.length"
      >
        <slot name="headerTitle"></slot>

        <table class="w-full border-collapse">
          <thead>
            <tr
              class="bg-gradient-to-r from-gray-50 to-gray-100 dark:from-gray-800 dark:to-gray-750 border-b border-gray-200 dark:border-gray-600"
            >
              <th v-if="selectable"
                  class="px-4 py-3 text-center text-xs font-bold text-gray-700 dark:text-gray-200 uppercase tracking-wider border-r border-gray-200 dark:border-gray-600"
                  :class="{'sticky top-0 z-10 bg-gray-100 dark:bg-gray-700': stickyHeader}"
              >
                <input type="checkbox" :checked="allRowsOnPageSelected" @change="toggleSelectAllOnPage" aria-label="Select all on page" />
              </th>

              <th v-if="props.showRowNumber"
                  class="px-4 py-3 text-center text-xs font-bold text-gray-700 dark:text-gray-200 uppercase tracking-wider border-r border-gray-200 dark:border-gray-600 bg-gray-100 dark:bg-gray-700"
                  :class="{'sticky top-0 z-10 bg-gray-100 dark:bg-gray-700': stickyHeader}"
              >
                #
              </th>

              <th
                v-for="header in props.headers"
                :key="header.value"
                v-show="visibleColumns.includes(header.value)"
                :style="getColumnStyle(header.value)"
                :aria-sort="ariaSortForHeader(header)"
                :class="[
                  'px-4 py-3 select-none text-left border-r border-gray-200 dark:border-gray-600 last:border-r-0',
                  stickyHeader ? 'sticky top-0 z-10 bg-gray-100 dark:bg-gray-700' : '',
                  header.align === 'center' ? 'text-center' : header.align === 'right' ? 'text-right' : 'text-left',
                ]"
              >
                <button
                  type="button"
                  class="w-full flex items-center justify-between focus:outline-none"
                  :class="header.sortable === false ? 'cursor-default' : 'cursor-pointer hover:opacity-80'"
                  @click="toggleSortForColumn(header)"
                >
                  <span class="text-xs font-bold text-gray-700 dark:text-gray-200 uppercase tracking-wider">
                    {{ header.caption }}
                    <span v-if="header.print === false" class="ml-1 text-gray-400 dark:text-gray-500 print-hide" title="Hidden on print">
                      <svg class="w-4 h-4 inline-block" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                              d="M13.875 18.825A10.05 10.05 0 0112 19c-4.478 0-8.268-2.943-9.543-7a9.97 9.97 0 011.563-3.029m5.858.908a3 3 0 114.243 4.243M9.878 9.878l4.242 4.242M9.88 9.88l-3.29-3.29m7.532 7.532l3.29 3.29M3 3l3.59 3.59m0 0A9.953 9.953 0 0112 5c4.478 0 8.268 2.943 9.543 7a10.025 10.025 0 01-4.132 5.411m0 0L21 21"/>
                      </svg>
                    </span>
                  </span>
                  <span v-if="sortKey === (header.sortKey || header.value)" class="text-blue-600 dark:text-blue-400 font-bold text-sm">
                    {{ sortAscending ? "↑" : "↓" }}
                  </span>
                </button>
              </th>
            </tr>
          </thead>

          <tbody class="divide-y divide-gray-200 dark:divide-gray-700">
            <tr
              v-for="(row, rowIndexOnPage) in paginatedItems"
              :key="getRowKey(row, rowIndexOnPage + (currentPage - 1) * currentPageSize)"
              class="hover:bg-blue-50 dark:hover:bg-gray-700/50 transition-all duration-150 group"
              @click="onRowClick(row, rowIndexOnPage)"
            >
              <td v-if="selectable"
                  class="px-4 py-3 text-center text-sm text-gray-500 dark:text-gray-400 font-medium border-r border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-800/50"
              >
                <input
                  type="checkbox"
                  :checked="selectedRowKeys.includes(getRowKey(row, rowIndexOnPage + (currentPage - 1) * currentPageSize))"
                  @change="toggleRowSelected(row, rowIndexOnPage + (currentPage - 1) * currentPageSize)"
                  aria-label="Select row"
                />
              </td>

              <td v-if="props.showRowNumber"
                  class="px-4 py-3 text-center text-sm text-gray-500 dark:text-gray-400 font-medium border-r border-gray-200 dark:border-gray-700 bg-gray-50 dark:bg-gray-800/50 group-hover:bg-blue-100 dark:group-hover:bg-gray-600/50">
                {{ rowIndexOnPage + 1 + (currentPage - 1) * currentPageSize }}
              </td>

              <td
                v-for="header in props.headers"
                :key="header.value"
                v-show="visibleColumns.includes(header.value)"
                :style="getColumnStyle(header.value)"
                class="px-4 py-3 text-sm text-gray-900 dark:text-gray-100 border-r border-gray-200 dark:border-gray-700 last:border-r-0"
                :class="[
                  props.dense ? 'py-2' : 'py-3',
                  header.align === 'center' ? 'text-center' : header.align === 'right' ? 'text-right' : 'text-left',
                ]"
              >
                <slot :name="header.value" :row="row" :value="getCellValue(row, header.value)">
                  <!-- Fallback to formatter or raw value -->
                  <template v-if="header.format">
                    {{ header.format(row, getCellValue(row, header.value)) }}
                  </template>
                  <template v-else>
                    {{ getCellValue(row, header.value) as any }}
                  </template>
                </slot>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Pagination -->
      <div class="mt-4 flex justify-between items-center" v-if="showPagination && !serverSide">
        <div class="text-sm text-gray-600 dark:text-gray-400">
          {{ t("showing") || "Showing" }}
          <span class="font-medium text-gray-900 dark:text-gray-100">{{ (currentPage - 1) * currentPageSize + 1 }}</span>
          -
          <span class="font-medium text-gray-900 dark:text-gray-100">{{ Math.min(currentPage * currentPageSize, sortedItems.length) }}</span>
          {{ t("of") || "of" }}
          <span class="font-medium text-gray-900 dark:text-gray-100">{{ sortedItems.length }}</span>
          {{ t("results") || "results" }}
        </div>

        <div class="flex items-center gap-2">
          <button
            @click="currentPage = currentPage - 1; emit('update:currentPage', currentPage)"
            :disabled="currentPage === 1"
            class="px-3 py-2 text-sm border border-gray-300 dark:border-gray-600 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500 dark:text-gray-300 dark:hover:text-gray-100"
          >
            {{ t("prev") || "Prev" }}
          </button>

          <span class="px-3 py-2 text-sm text-gray-600 dark:text-gray-400">
            {{ t("page") || "Page" }}
            <span class="font-medium text-gray-900 dark:text-gray-100">{{ currentPage }}</span>
            {{ t("of") || "of" }}
            <span class="font-medium text-gray-900 dark:text-gray-100">{{ totalPages }}</span>
          </span>

          <button
            @click="currentPage = currentPage + 1; emit('update:currentPage', currentPage)"
            :disabled="currentPage >= totalPages"
            class="px-3 py-2 text-sm border border-gray-300 dark:border-gray-600 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-50 dark:hover:bg-gray-700 transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500 dark:text-gray-300 dark:hover:text-gray-100"
          >
            {{ t("next") || "Next" }}
          </button>
        </div>
      </div>

      <!-- Server-side consumer can provide their own pager via slot -->
      <slot v-else name="serverPager" />
    </div>

    <!-- No Data Message -->
    <div v-else-if="!props.items.length && props.showNoData" class="text-center mt-8 py-8 text-gray-500 dark:text-gray-400">
      <div class="text-lg">{{ t(props.titleNoData) || t("noData") }}</div>
    </div>

    <!-- No Results Message -->
    <div v-else-if="!filteredItems.length" class="text-center py-4 text-gray-500 dark:text-gray-400 border-2 border-dotted border-gray-400 rounded-lg">
      <div class="text-lg">{{ t(props.titleNoData) || "No results found for your search" }}</div>
    </div>

    <!-- Overlay to close popup when clicking outside -->
    <div v-if="showColumnPopup" @click="showColumnPopup = false" class="fixed inset-0 z-40"></div>
  </div>
</template>

<style scoped>
.table-container { position: relative; }

/* Custom scrollbar for column popup */
.custom-scrollbar::-webkit-scrollbar { width: 4px; }
.custom-scrollbar::-webkit-scrollbar-track { background: rgb(243 244 246); border-radius: 4px; }
.custom-scrollbar::-webkit-scrollbar-thumb { background: rgb(209 213 219); border-radius: 4px; }

/* Dark theme gradient support */
.dark .bg-gradient-to-r.from-gray-50.to-gray-100 {
  background: linear-gradient(to right, rgb(31, 41, 55), rgb(17, 24, 39));
}

/* Optional sticky first column (when you enable stickyFirstColumn you may set a class on the first <td>) */
.sticky-first-col { position: sticky; left: 0; z-index: 5; background: inherit; }

/* Print simplifications */
@media print {
  .table-container > *:not(#printable-table) { display: none !important; }
  #printable-table { overflow: visible !important; border: none !important; border-radius: 0 !important; }
  thead { display: table-header-group; }
  tfoot { display: table-footer-group; }
  .print-hide { display: none !important; }
  th, td { background: white !important; color: black !important; }
  th { background: #f5f5f5 !important; }
}
</style>
