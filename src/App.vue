<template>
  <main class="app">
    <header class="app__header">
      <h1>To‑Do List (Server)</h1>
      <p class="muted">Vue 3 • Composition API • REST API (/api/todos)</p>
    </header>

    <section class="card">
      <form class="form" @submit.prevent="createTodo">
        <input
            v-model.trim="draft.title"
            type="text"
            placeholder="할 일을 입력하세요"
            aria-label="할 일 제목"
            required
        />
        <select v-model="draft.status" aria-label="상태">
          <option value="TODO">TODO</option>
          <option value="IN_PROGRESS">IN_PROGRESS</option>
          <option value="DONE">DONE</option>
        </select>
        <button type="submit" class="btn primary" :disabled="creating">{{ creating ? '추가중…' : '추가' }}</button>
      </form>
      <p v-if="error" class="error">⚠️ {{ error }}</p>
    </section>

    <section class="toolbar">
      <div class="filters">
        <label>
          보기:
          <select v-model="ui.filter">
            <option value="all">전체</option>
            <option value="TODO">TODO</option>
            <option value="IN_PROGRESS">IN_PROGRESS</option>
            <option value="DONE">DONE</option>
          </select>
        </label>
        <label>
          정렬:
          <select v-model="ui.sortBy">
            <option value="createdAt">생성일</option>
            <option value="updatedAt">수정일</option>
            <option value="title">제목</option>
            <option value="status">상태</option>
          </select>
        </label>
        <label>
          순서:
          <select v-model="ui.sortDir">
            <option value="desc">내림차순</option>
            <option value="asc">오름차순</option>
          </select>
        </label>
      </div>
      <input
          v-model.trim="ui.query"
          type="search"
          placeholder="검색 (제목)"
          class="search"
          aria-label="검색"
      />
    </section>

    <section class="card">
      <div v-if="loading" class="empty"><p>불러오는 중…</p></div>
      <div v-else-if="filtered.length === 0" class="empty"><p>표시할 항목이 없습니다.</p></div>

      <ul class="list" v-else>
        <li v-for="t in filtered" :key="t.id" class="item" :class="{ done: t.status === 'DONE' }">
          <div class="content">
            <template v-if="editId === t.id">
              <input v-model.trim="editDraft.title" class="edit-title" aria-label="제목 편집" />
              <div class="meta">
                <select v-model="editDraft.status" aria-label="상태 편집">
                  <option value="TODO">TODO</option>
                  <option value="IN_PROGRESS">IN_PROGRESS</option>
                  <option value="DONE">DONE</option>
                </select>
              </div>
            </template>
            <template v-else>
              <div class="title">{{ t.title }}</div>
              <div class="meta">
                <span class="status tag">{{ t.status }}</span>
                <span class="muted">생성: {{ formatDateTime(t.createdAt) }}</span>
                <span class="muted">수정: {{ formatDateTime(t.updatedAt) }}</span>
              </div>
            </template>
          </div>

          <div class="actions">
            <template v-if="editId === t.id">
              <button class="btn small" @click="saveEdit(t)" :disabled="updating">{{ updating ? '저장중…' : '저장' }}</button>
              <button class="btn small" @click="cancelEdit">취소</button>
            </template>
            <template v-else>
              <button class="btn small" @click="cycleStatus(t)">상태변경</button>
              <button class="btn small" @click="startEdit(t)">편집</button>
              <button class="btn small danger" @click="removeTodo(t.id)" :disabled="deletingId===t.id">{{ deletingId===t.id ? '삭제중…' : '삭제' }}</button>
            </template>
          </div>
        </li>
      </ul>

      <footer class="footer" v-if="todos.length">
        <span>{{ counts.TODO }} TODO • {{ counts.IN_PROGRESS }} 진행중 • {{ counts.DONE }} 완료</span>
        <div class="spacer"></div>
        <button class="btn ghost" @click="refresh">새로고침</button>
      </footer>
    </section>
  </main>
</template>

<script setup>
import { reactive, ref, computed, onMounted } from 'vue'

// ------- Types (runtime) -------
/** @typedef {{id:number, title:string, status:'TODO'|'IN_PROGRESS'|'DONE', createdAt:string, updatedAt:string}} Todo */

// ------- State -------
const API_BASE = '/api/todos'
const todos = ref(/** @type {Todo[]} */([]))
const loading = ref(false)
const creating = ref(false)
const updating = ref(false)
const deletingId = ref(null)
const error = ref('')

const ui = reactive({ filter: 'all', sortBy: 'createdAt', sortDir: 'desc', query: '' })
const draft = reactive({ title: '', status: 'TODO' })
const editId = ref(null)
const editDraft = reactive({ title: '', status: 'TODO' })

// ------- API helper -------
async function api(path, options={}) {
  const res = await fetch(path, { headers: { 'Content-Type': 'application/json' }, ...options })
  if (!res.ok) {
    const text = await res.text().catch(()=>'')
    throw new Error(text || `HTTP ${res.status}`)
  }
  const ct = res.headers.get('content-type') || ''
  if (ct.includes('application/json')) return res.json()
  return null
}

// ------- CRUD calls -------
async function refresh() {
  loading.value = true; error.value = ''
  try {
    todos.value = await api(API_BASE)
  } catch (e) {
    error.value = e.message
  } finally { loading.value = false }
}

async function createTodo() {
  if (!draft.title) return
  creating.value = true; error.value = ''
  try {
    const res = await api(API_BASE, { method: 'POST', body: JSON.stringify({ title: draft.title, status: draft.status }) })
    // 서버가 {id} 반환 → 새 항목 조회 후 목록 갱신
    const created = await api(`${API_BASE}/${res.id}`)
    todos.value.unshift(created)
    Object.assign(draft, { title: '', status: 'TODO' })
  } catch (e) {
    error.value = e.message
  } finally { creating.value = false }
}

function startEdit(t) {
  editId.value = t.id
  Object.assign(editDraft, { title: t.title, status: t.status })
}
function cancelEdit() { editId.value = null }

async function saveEdit(t) {
  if (!editDraft.title) return
  updating.value = true; error.value = ''
  try {
    await api(`${API_BASE}/${t.id}`, { method: 'PUT', body: JSON.stringify({ title: editDraft.title, status: editDraft.status }) })
    // PUT은 204 → 개별 항목 재조회
    const updated = await api(`${API_BASE}/${t.id}`)
    const idx = todos.value.findIndex(x => x.id === t.id)
    if (idx !== -1) todos.value[idx] = updated
    editId.value = null
  } catch (e) {
    error.value = e.message
  } finally { updating.value = false }
}

async function removeTodo(id) {
  deletingId.value = id; error.value = ''
  try {
    await api(`${API_BASE}/${id}`, { method: 'DELETE' })
    const idx = todos.value.findIndex(t => t.id === id)
    if (idx !== -1) todos.value.splice(idx, 1)
  } catch (e) {
    error.value = e.message
  } finally { deletingId.value = null }
}

// 편의: 상태 순환 (TODO→IN_PROGRESS→DONE→TODO)
async function cycleStatus(t) {
  const order = ['TODO','IN_PROGRESS','DONE']
  const next = order[(order.indexOf(t.status) + 1) % order.length]
  editDraft.title = t.title; editDraft.status = next
  await saveEdit(t)
}

// ------- Derived -------
const filtered = computed(() => {
  let list = [...todos.value]
  if (ui.filter !== 'all') list = list.filter(t => t.status === ui.filter)
  if (ui.query) {
    const q = ui.query.toLowerCase()
    list = list.filter(t => t.title.toLowerCase().includes(q))
  }
  const dir = ui.sortDir === 'asc' ? 1 : -1
  list.sort((a,b) => {
    const by = ui.sortBy
    const va = a[by]
    const vb = b[by]
    return (va > vb ? 1 : va < vb ? -1 : 0) * dir
  })
  return list
})

const counts = computed(() => ({
  TODO: todos.value.filter(t => t.status==='TODO').length,
  IN_PROGRESS: todos.value.filter(t => t.status==='IN_PROGRESS').length,
  DONE: todos.value.filter(t => t.status==='DONE').length,
}))

// ------- Utils -------
function pad(n){ return String(n).padStart(2,'0') }
function formatDateTime(iso) {
  if (!iso) return ''
  const d = new Date(iso)
  return `${d.getFullYear()}-${pad(d.getMonth()+1)}-${pad(d.getDate())} ${pad(d.getHours())}:${pad(d.getMinutes())}`
}

onMounted(refresh)
</script>

<style scoped>
:root { color-scheme: light dark; }
* { box-sizing: border-box; }
body, .app { margin: 0; font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, 'Noto Sans KR', Arial, 'Apple SD Gothic Neo', 'Malgun Gothic', sans-serif; }
.app { max-width: 880px; margin: 2rem auto; padding: 0 1rem; }
.app__header { display:flex; align-items:baseline; gap:.75rem; margin-bottom: 1rem; }
.muted { opacity:.7; font-size:.9rem; }
.card { background: color-mix(in oklab, canvas, canvastext 3%); border: 1px solid color-mix(in oklab, canvastext, canvas 85%); border-radius: 16px; padding: 1rem; box-shadow: 0 2px 10px rgba(0,0,0,.03); }

.form { display: grid; grid-template-columns: 1fr max-content max-content; gap:.5rem; }
.form input[type="text"], .search, .edit-title, select { height: 40px; padding: 0 .75rem; border-radius: 10px; border:1px solid color-mix(in oklab, canvastext, canvas 80%); background: canvas; color: canvastext; }
.form .btn { height: 40px; }

.toolbar { display:flex; align-items:center; justify-content: space-between; gap:1rem; margin: 1rem 0; }
.filters { display:flex; gap:.75rem; align-items:center; }
.search { flex:1; }

.list { list-style: none; padding: 0; margin: 0; display: grid; gap: .5rem; }
.item { display: grid; grid-template-columns: 1fr max-content; align-items: center; gap:.75rem; padding:.5rem; border-radius: 12px; border:1px solid color-mix(in oklab, canvastext, canvas 90%); background: color-mix(in oklab, canvas, canvastext 2%); }
.item.done .title { text-decoration: line-through; opacity:.6 }

.content { display:flex; flex-direction: column; gap:.25rem; }
.title { font-weight: 600; }
.meta { display:flex; flex-wrap: wrap; gap:.5rem .75rem; align-items:center; font-size:.9rem; }
.status.tag { padding:.1rem .5rem; border:1px solid color-mix(in oklab, canvastext, canvas 80%); border-radius:999px; }

.actions { display:flex; gap:.5rem; }

.btn { border: 1px solid color-mix(in oklab, canvastext, canvas 80%); background: canvas; color: canvastext; border-radius: 10px; height: 34px; padding: 0 .75rem; cursor: pointer; }
.btn.primary { background: color-mix(in oklab, canvastext, canvas 85%); border-color: transparent; }
.btn.danger { border-color: transparent; background: #e53935; color: white; }
.btn.ghost { background: transparent; }
.btn.small { height: 30px; padding: 0 .6rem; }
.btn:disabled { opacity: .5; cursor: not-allowed; }

.footer { display:flex; align-items:center; gap:.75rem; margin-top:.75rem; }
.spacer { flex:1; }

.empty { text-align:center; padding: 1rem; color: color-mix(in oklab, canvastext, canvas 40%); }
.error { margin-top:.5rem; color: #b00020; }

@media (max-width: 700px) {
  .form { grid-template-columns: 1fr; }
}
</style>
