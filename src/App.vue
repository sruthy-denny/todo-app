<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';

const items = ref([]);
const newItem = ref('');
const editingId = ref(null);
const editingText = ref('');
const appRef = ref(null);
let nextId = 1;

const lastDeleted = ref(null);
let undoTimeout = null;

const selectedCount = computed(() => items.value.filter(i => i.done).length);
const bulkButtonLabel = computed(() =>
  selectedCount.value === items.value.length ? 'Delete All' : `Delete Selected (${selectedCount.value})`
);

function addItem() {
  const text = newItem.value.trim();
  if (!text) return;
  items.value.push({ id: nextId++, text, done: false });
  newItem.value = '';
}

function toggleDone(item) {
  item.done = !item.done;
}

function deleteItem(id) {
  const index = items.value.findIndex(i => i.id === id);
  if (index === -1) return;
  lastDeleted.value = { item: items.value[index], index };
  items.value = items.value.filter(i => i.id !== id);

  clearTimeout(undoTimeout);
  undoTimeout = setTimeout(() => {
    lastDeleted.value = null;
  }, 5000);
}

function undoDelete() {
  if (!lastDeleted.value) return;
  if (lastDeleted.value.bulk) {
    items.value = lastDeleted.value.items;
  } else {
    items.value.splice(lastDeleted.value.index, 0, lastDeleted.value.item);
  }
  lastDeleted.value = null;
  clearTimeout(undoTimeout);
}

function handleOutsideClick() {
  items.value.forEach(item => { item.done = false; });
}

onMounted(() => {
  document.addEventListener('click', handleOutsideClick);
});

onUnmounted(() => {
  document.removeEventListener('click', handleOutsideClick);
});

function deleteSelected() {
  const toDelete = items.value.filter(i => i.done);
  if (toDelete.length < 2) return;
  if (!confirm(`Delete ${toDelete.length} selected tasks?`)) return;

  lastDeleted.value = { bulk: true, items: items.value };
  items.value = items.value.filter(i => !i.done);

  clearTimeout(undoTimeout);
  undoTimeout = setTimeout(() => {
    lastDeleted.value = null;
  }, 5000);
}

function startEdit(item) {
  editingId.value = item.id;
  editingText.value = item.text;
}

function saveEdit(item) {
  const text = editingText.value.trim();
  if (text) item.text = text;
  item.done = false;
  editingId.value = null;
}

function cancelEdit() {
  editingId.value = null;
}
</script>

<template>
  <div class="app" ref="appRef" @click.stop>
    <h1>To-Do List</h1>

    <div class="add-row">
      <input
        type="text"
        v-model="newItem"
        placeholder="Add a new task..."
        @keyup.enter="addItem"
      />
      <button class="btn-add" @click="addItem">Add</button>
    </div>

    <div v-if="selectedCount > 1" class="list-toolbar">
      <button class="btn-deselect" @click="handleOutsideClick">Clear selection</button>
      <button class="btn-clear-all" @click="deleteSelected">{{ bulkButtonLabel }}</button>
    </div>

    <ul v-if="items.length">
      <li v-for="item in items" :key="item.id">
        <input type="checkbox" :checked="item.done" @change="toggleDone(item)" />

        <template v-if="editingId === item.id">
          <div class="edit-wrapper">
            <input
              type="text"
              class="edit-input"
              v-model="editingText"
              @keyup.enter="saveEdit(item)"
              @keyup.esc="cancelEdit"
            />
            <div class="actions">
              <button class="btn-save" @click="saveEdit(item)">Save</button>
              <button class="btn-cancel" @click="cancelEdit">Cancel</button>
            </div>
          </div>
        </template>

        <template v-else>
          <span class="item-text" :class="{ done: item.done }">{{ item.text }}</span>
          <div class="actions">
            <button class="btn-edit" @click="startEdit(item)" aria-label="Edit">
              <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5Z"/><path d="m15 5 4 4"/></svg>
            </button>
            <button class="btn-delete" :disabled="!item.done" @click="deleteItem(item.id)" aria-label="Delete">
              <svg xmlns="http://www.w3.org/2000/svg" width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6"/><path d="M10 11v6"/><path d="M14 11v6"/><path d="M9 6V4a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2"/></svg>
            </button>
          </div>
        </template>
      </li>
    </ul>
    <div v-else class="empty">No tasks yet — add one above.</div>

    <div v-if="lastDeleted" class="undo-banner">
      {{ lastDeleted.bulk ? 'All tasks deleted.' : 'Task deleted.' }}
      <button class="btn-undo" @click="undoDelete">Undo</button>
    </div>
  </div>
</template>

<style scoped>
.app {
  --bg: rgba(45, 25, 50, 0.55);
  --card: #ffffff;
  --text: #ffffff;
  --muted: #e8d8ec;
  --accent: #4f46e5;
  --border: rgba(255, 255, 255, 0.25);
  width: 480px;
  min-height: 420px;
  margin: 0;
  padding: 40px;
  background: var(--bg);
  backdrop-filter: blur(6px);
  border-radius: 12px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: var(--text);
  box-sizing: border-box;
}
h1 { font-size: 32px; margin: 0 0 16px; text-align: center; }
.add-row { display: flex; gap: 8px; margin-bottom: 16px; }
input[type="text"] {
  flex: 1;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1px solid var(--border);
  background: var(--card);
  color: #1d1d1f;
  font-size: 15px;
  min-width: 0;
}
button {
  border: none;
  border-radius: 8px;
  padding: 10px 14px;
  font-size: 14px;
  cursor: pointer;
  font-weight: 500;
}
.btn-add {
  background: linear-gradient(135deg, #C33764, #8E44AD);
  color: white;
}
.btn-save { background: #E91E8C; color: white; }
.btn-edit, .btn-delete {
  background: transparent;
  color: #1a1a2e;
  padding: 6px 8px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  font-weight: 700;
}
.btn-edit svg, .btn-delete svg {
  stroke-width: 2.5;
}
.btn-edit {
  color: #2b1a4a;
}
.btn-delete {
  color: #ff3b30;
}
.btn-delete:disabled {
  opacity: 0.35;
  cursor: not-allowed;
}
.btn-cancel {
  background: rgba(255, 255, 255, 0.4);
  color: #3a2e42;
  padding: 6px 10px;
  font-weight: 600;
}
.btn-delete:hover { color: #d90429; }
.btn-edit:hover { color: var(--accent); }

.list-toolbar {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-bottom: 10px;
}
.btn-deselect {
  background: transparent;
  border: 1px solid transparent;
  color: var(--muted);
  padding: 6px 12px;
  font-size: 13px;
  font-weight: 600;
  text-decoration: underline;
}
.btn-deselect:hover { color: #ffffff; }
.btn-clear-all {
  background: transparent;
  border: 1px solid rgba(255, 255, 255, 0.4);
  color: var(--muted);
  padding: 6px 12px;
  font-size: 13px;
  font-weight: 600;
}
.btn-clear-all:hover {
  color: #ffffff;
  border-color: #ffffff;
  background: rgba(255, 255, 255, 0.1);
}

ul { list-style: none; margin: 0; padding: 0; }
li {
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 10px 12px;
  margin-bottom: 8px;
  background: #2DD4C4;
}
.item-text { flex: 1; font-size: 16px; font-weight: 700; word-break: break-word; color: #000000; }
.item-text.done { opacity: 0.55; }
.edit-wrapper {
  display: flex;
  flex-direction: column;
  gap: 8px;
  flex: 1;
  min-width: 0;
}
.edit-input {
  flex: 1;
  min-width: 0;
  padding: 6px 8px;
  border-radius: 6px;
  border: 1px solid var(--accent);
  background: var(--bg);
  color: var(--text);
  font-size: 15px;
}
.actions { display: flex; gap: 2px; flex-shrink: 0; }
.empty { color: var(--muted); text-align: center; padding: 24px 0; font-size: 14px; }
.undo-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: rgba(255, 255, 255, 0.15);
  color: var(--text);
  padding: 10px 14px;
  border-radius: 8px;
  font-size: 14px;
  margin-top: 12px;
}
.btn-undo {
  background: transparent;
  color: #fff;
  text-decoration: underline;
  font-weight: 600;
  padding: 0;
}
</style>

<style>
body{
  margin: 0;
  background: linear-gradient(135deg, #C33764, #8E44AD);
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}
</style>