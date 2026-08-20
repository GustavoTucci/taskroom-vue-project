<script setup lang="ts">
import { computed, onMounted, ref, watch } from 'vue'

type ColumnId = 'backlog' | 'doing' | 'done'
type Priority = 'Alta' | 'Média' | 'Baixa'

interface Task {
  id: number
  title: string
  description: string
  priority: Priority
  tag: string
  column: ColumnId
}

const columns = [
  { id: 'backlog' as ColumnId, title: 'A fazer', color: 'coral' },
  { id: 'doing' as ColumnId, title: 'Em andamento', color: 'gold' },
  { id: 'done' as ColumnId, title: 'Concluído', color: 'mint' },
]

const defaultTasks: Task[] = [
  { id: 1, title: 'Mapear fluxo do usuário', description: 'Entender os pontos principais da experiência.', priority: 'Alta', tag: 'Pesquisa', column: 'backlog' },
  { id: 2, title: 'Criar wireframes', description: 'Rascunhar as telas essenciais do produto.', priority: 'Média', tag: 'Design', column: 'backlog' },
  { id: 3, title: 'Configurar componentes', description: 'Montar a base visual reutilizável em Vue.', priority: 'Alta', tag: 'Desenvolvimento', column: 'doing' },
  { id: 4, title: 'Revisar tipografia', description: 'Conferir hierarquia e legibilidade dos textos.', priority: 'Baixa', tag: 'Design', column: 'done' },
]

const storedTasks = localStorage.getItem('vue-kanban-tasks')
const tasks = ref<Task[]>(storedTasks ? JSON.parse(storedTasks) : defaultTasks)
const search = ref('')
const priorityFilter = ref<'Todas' | Priority>('Todas')
const draggedTaskId = ref<number | null>(null)
const dragOverColumn = ref<ColumnId | null>(null)
const isFormOpen = ref(false)
const newTask = ref({ title: '', description: '', priority: 'Média' as Priority, tag: 'Geral' })
const selectedTask = ref<Task | null>(null)
const theme = ref<'dark' | 'light'>((localStorage.getItem('vue-kanban-theme') as 'dark' | 'light') || 'dark')
const boardAccent = ref(localStorage.getItem('vue-kanban-accent') || '#e85e46')
const isLoading = ref(true)
const feedback = ref('')
const pointerDragId = ref<number | null>(null)
const pointerDidMove = ref(false)
const suppressClick = ref(false)
let feedbackTimer: number | undefined

watch(tasks, (value) => localStorage.setItem('vue-kanban-tasks', JSON.stringify(value)), { deep: true })
watch(theme, (value) => localStorage.setItem('vue-kanban-theme', value))
watch(boardAccent, (value) => localStorage.setItem('vue-kanban-accent', value))

onMounted(() => {
  window.setTimeout(() => { isLoading.value = false }, 350)
})

const visibleTasks = computed(() => tasks.value.filter((task) => {
  const matchesSearch = `${task.title} ${task.description} ${task.tag}`.toLowerCase().includes(search.value.toLowerCase())
  const matchesPriority = priorityFilter.value === 'Todas' || task.priority === priorityFilter.value
  return matchesSearch && matchesPriority
}))

const completedCount = computed(() => tasks.value.filter((task) => task.column === 'done').length)

function tasksForColumn(column: ColumnId) {
  return visibleTasks.value.filter((task) => task.column === column)
}

function addTask() {
  if (!newTask.value.title.trim()) {
    showFeedback('Dê um nome à tarefa antes de adicionar.')
    return
  }
  tasks.value.unshift({ id: Date.now(), title: newTask.value.title.trim(), description: newTask.value.description.trim() || 'Sem descrição', priority: newTask.value.priority, tag: newTask.value.tag.trim() || 'Geral', column: 'backlog' })
  newTask.value = { title: '', description: '', priority: 'Média', tag: 'Geral' }
  isFormOpen.value = false
  showFeedback('Tarefa adicionada ao quadro.')
}

function deleteTask(id: number) {
  tasks.value = tasks.value.filter((task) => task.id !== id)
  if (selectedTask.value?.id === id) selectedTask.value = null
  showFeedback('Tarefa excluída.')
}

function moveTask(column: ColumnId) {
  const task = tasks.value.find((item) => item.id === draggedTaskId.value)
  if (task && task.column !== column) {
    task.column = column
    showFeedback(`Tarefa movida para ${columns.find((item) => item.id === column)?.title}.`)
  }
  draggedTaskId.value = null
  pointerDragId.value = null
  dragOverColumn.value = null
}

function startDragging(id: number) {
  draggedTaskId.value = id
}

function showFeedback(message: string) {
  feedback.value = message
  window.clearTimeout(feedbackTimer)
  feedbackTimer = window.setTimeout(() => { feedback.value = '' }, 2600)
}

function openTask(task: Task) {
  if (suppressClick.value) {
    suppressClick.value = false
    return
  }
  selectedTask.value = task
}

function startPointerDrag(id: number) {
  pointerDragId.value = id
  draggedTaskId.value = id
  pointerDidMove.value = false
}

function updatePointerDrag(event: PointerEvent, markMoved = true) {
  if (pointerDragId.value === null) return
  if (markMoved) pointerDidMove.value = true
  const target = document.elementFromPoint(event.clientX, event.clientY)?.closest('[data-column]') as HTMLElement | null
  dragOverColumn.value = (target?.dataset.column as ColumnId | undefined) || null
}

function finishPointerDrag(event: PointerEvent) {
  if (pointerDragId.value === null) return
  updatePointerDrag(event, false)
  suppressClick.value = pointerDidMove.value
  if (dragOverColumn.value) moveTask(dragOverColumn.value)
  else {
    pointerDragId.value = null
    draggedTaskId.value = null
    pointerDidMove.value = false
  }
}

function updateCardTilt(event: MouseEvent) {
  const card = event.currentTarget as HTMLElement
  const bounds = card.getBoundingClientRect()
  const rotateX = ((event.clientY - bounds.top) / bounds.height - 0.5) * -5
  const rotateY = ((event.clientX - bounds.left) / bounds.width - 0.5) * 7
  card.style.setProperty('--rotate-x', `${rotateX}deg`)
  card.style.setProperty('--rotate-y', `${rotateY}deg`)
}

function resetCardTilt(event: MouseEvent) {
  const card = event.currentTarget as HTMLElement
  card.style.setProperty('--rotate-x', '0deg')
  card.style.setProperty('--rotate-y', '0deg')
}
</script>

<template>
  <main class="app-shell" :class="`theme-${theme}`" :style="{ '--accent': boardAccent }">
    <header class="topbar">
      <div class="brand"><span class="brand-mark">+</span><span>Taskroom</span></div>
      <div class="topbar-actions"><span class="date-label">Terça-feira, 19 de agosto</span><button class="theme-toggle" :aria-label="theme === 'dark' ? 'Ativar tema claro' : 'Ativar tema escuro'" @click="theme = theme === 'dark' ? 'light' : 'dark'">{{ theme === 'dark' ? '☼' : '☾' }}</button><button class="avatar" aria-label="Perfil">GT</button></div>
    </header>

    <section class="workspace-heading">
      <div>
        <p class="eyebrow">Workspace / Produto</p>
        <h1>Planeje. Construa. Entregue.</h1>
        <p class="subtitle">Um espaço calmo para acompanhar o que está acontecendo no seu projeto.</p>
      </div>
      <button class="primary-button" @click="isFormOpen = !isFormOpen"><span>+</span> Nova tarefa</button>
    </section>

    <section v-if="isFormOpen" class="task-form" aria-label="Nova tarefa">
      <input v-model="newTask.title" autofocus placeholder="Nome da tarefa" @keyup.enter="addTask" />
      <input v-model="newTask.description" placeholder="Descrição breve" />
      <input v-model="newTask.tag" placeholder="Etiqueta" />
      <select v-model="newTask.priority"><option>Alta</option><option>Média</option><option>Baixa</option></select>
      <button class="primary-button" @click="addTask">Adicionar</button>
    </section>

    <section class="toolbar">
      <label class="search-box"><span>⌕</span><input v-model="search" placeholder="Buscar tarefas..." /></label>
      <div class="filters"><span class="filter-label">Prioridade</span><button v-for="option in ['Todas', 'Alta', 'Média', 'Baixa']" :key="option" :class="{ active: priorityFilter === option }" @click="priorityFilter = option as typeof priorityFilter">{{ option }}</button></div>
      <label class="accent-picker" title="Personalizar cor do quadro"><span>Cor</span><input v-model="boardAccent" type="color" aria-label="Cor de destaque do quadro" /></label>
      <span class="progress">{{ completedCount }} de {{ tasks.length }} concluídas</span>
    </section>

    <section v-if="isLoading" class="board loading-board" aria-label="Carregando quadro"><article v-for="column in columns" :key="column.id" class="column skeleton-column"><div class="skeleton-line"></div><div class="skeleton-card"></div><div class="skeleton-card"></div></article></section>
    <section v-else class="board">
      <article v-for="column in columns" :key="column.id" :data-column="column.id" :class="['column', { 'is-drop-target': dragOverColumn === column.id } ]" @dragover.prevent="dragOverColumn = column.id" @dragleave="dragOverColumn = null" @drop="moveTask(column.id)">
        <div class="column-heading"><div><span :class="['status-dot', column.color]"></span><h2>{{ column.title }}</h2></div><span class="count">{{ tasksForColumn(column.id).length }}</span></div>
        <div class="task-list">
          <div v-for="task in tasksForColumn(column.id)" :key="task.id" :class="['task-card', { 'is-dragging': draggedTaskId === task.id } ]" draggable="true" @click="openTask(task)" @dragstart="startDragging(task.id)" @dragend="draggedTaskId = null; dragOverColumn = null" @pointerdown.stop="startPointerDrag(task.id)" @pointermove.stop="updatePointerDrag" @pointerup.stop="finishPointerDrag" @pointercancel="draggedTaskId = null; pointerDragId = null; dragOverColumn = null" @mousemove="updateCardTilt" @mouseleave="resetCardTilt">
            <div class="task-card-top"><span :class="['priority', task.priority.toLowerCase()]">{{ task.priority }}</span><button class="delete-button" aria-label="Excluir tarefa" @click.stop="deleteTask(task.id)">×</button></div>
            <h3>{{ task.title }}</h3><p>{{ task.description }}</p><span class="tag">{{ task.tag }}</span>
          </div>
          <div v-if="tasksForColumn(column.id).length === 0" class="empty-state">Solte uma tarefa aqui</div>
        </div>
      </article>
    </section>
    <Transition name="toast"><p v-if="feedback" class="feedback" role="status">{{ feedback }}</p></Transition>
    <Transition name="modal"><div v-if="selectedTask" class="modal-backdrop" @click.self="selectedTask = null"><section class="task-modal" role="dialog" aria-modal="true" aria-labelledby="task-modal-title"><button class="modal-close" aria-label="Fechar detalhes" @click="selectedTask = null">×</button><span :class="['priority', selectedTask.priority.toLowerCase()]">{{ selectedTask.priority }}</span><h2 id="task-modal-title">{{ selectedTask.title }}</h2><p>{{ selectedTask.description }}</p><dl><div><dt>Etiqueta</dt><dd>{{ selectedTask.tag }}</dd></div><div><dt>Status</dt><dd>{{ columns.find((column) => column.id === selectedTask?.column)?.title }}</dd></div></dl></section></div></Transition>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=Space+Grotesk:wght@500;600;700&display=swap');

:root { color: #252525; background: #17191c; font-family: 'DM Sans', sans-serif; font-synthesis: none; }
* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; background: #17191c; }
button, input, select { font: inherit; }
button { cursor: pointer; }
.app-shell { --accent-soft: color-mix(in srgb, var(--accent) 20%, transparent); min-height: 100vh; padding: 0 6vw 56px; color: #f3f0e9; background: radial-gradient(circle at 92% -4%, var(--accent-soft), transparent 24%), radial-gradient(circle at 8% 45%, rgba(56, 143, 151, .14), transparent 24%), #17191c; background-image: radial-gradient(rgba(255,255,255,.045) 1px, transparent 1px), radial-gradient(circle at 92% -4%, var(--accent-soft), transparent 24%), radial-gradient(circle at 8% 45%, rgba(56, 143, 151, .14), transparent 24%), linear-gradient(135deg, rgba(255,255,255,.025) 25%, transparent 25%); background-size: 28px 28px, auto, auto, 5px 5px; transition: color .3s, background .3s; }
.theme-light { color: #20252a; background-color: #eef1ed; background-image: radial-gradient(rgba(32,37,42,.07) 1px, transparent 1px), radial-gradient(circle at 92% -4%, var(--accent-soft), transparent 24%), linear-gradient(135deg, rgba(255,255,255,.7) 25%, transparent 25%); }
.theme-light .topbar, .theme-light .toolbar { border-color: rgba(32,37,42,.14); }.theme-light .date-label, .theme-light .subtitle, .theme-light .filter-label, .theme-light .progress { color: #687278; }.theme-light .search-box input { color: #20252a; }.theme-light .filters button { color: #687278; }.theme-light .filters button.active { color: #20252a; background: rgba(32,37,42,.1); }.theme-light .column { border-color: rgba(32,37,42,.12); background: linear-gradient(145deg, rgba(255,255,255,.9), rgba(222,228,222,.82)); box-shadow: 0 22px 45px rgba(46,57,49,.1), inset 0 1px rgba(255,255,255,.8); }.theme-light .task-card { border-color: rgba(32,37,42,.13); color: #20252a; background: linear-gradient(145deg, #fff, #e1e7e1); box-shadow: 0 13px 0 rgba(46,57,49,.08), 0 18px 25px rgba(46,57,49,.12), inset 0 1px rgba(255,255,255,.9); }.theme-light .task-card p, .theme-light .tag { color: #687278; }.theme-light .tag { background: rgba(32,37,42,.08); }.theme-light .task-form { border-color: rgba(32,37,42,.12); background: rgba(255,255,255,.72); }.theme-light .task-form input, .theme-light .task-form select { color: #20252a; background: #fff; border-color: rgba(32,37,42,.14); }.theme-light .theme-toggle { color: #20252a; border-color: rgba(32,37,42,.18); }
.topbar { height: 76px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid rgba(255,255,255,.1); }
.brand { display: flex; align-items: center; gap: 10px; font-family: 'Space Grotesk', sans-serif; font-size: 18px; font-weight: 700; letter-spacing: -.4px; }
.brand-mark { display: grid; place-items: center; width: 27px; height: 27px; color: #fff; background: var(--accent); border-radius: 8px; font-size: 21px; line-height: 1; box-shadow: 0 0 20px var(--accent-soft); }.topbar-actions { display: flex; align-items: center; gap: 20px; color: #8e969d; font-size: 13px; }.avatar { border: 1px solid rgba(255,255,255,.2); width: 34px; height: 34px; border-radius: 50%; color: #fff; background: #334e68; font-size: 12px; font-weight: 700; box-shadow: 0 0 0 5px rgba(51,78,104,.12); }.theme-toggle { width: 32px; height: 32px; border: 1px solid rgba(255,255,255,.2); border-radius: 50%; color: #f3f0e9; background: transparent; font-size: 18px; transition: transform .2s, background .2s; }.theme-toggle:hover { transform: rotate(15deg); background: var(--accent-soft); }
.workspace-heading { display: flex; justify-content: space-between; align-items: end; gap: 24px; padding: 62px 0 38px; }.eyebrow { margin: 0 0 14px; color: #f07b62; font-size: 12px; font-weight: 700; letter-spacing: 1.5px; text-transform: uppercase; }.workspace-heading h1 { margin: 0; max-width: 680px; font-family: 'Space Grotesk', sans-serif; font-size: clamp(34px, 5vw, 58px); letter-spacing: -2.8px; line-height: .98; text-shadow: 0 3px 0 rgba(255,255,255,.04); }.subtitle { margin: 18px 0 0; color: #929ba1; font-size: 16px; }.primary-button { display: inline-flex; align-items: center; gap: 8px; padding: 13px 17px; border: 1px solid rgba(255,255,255,.16); border-radius: 7px; color: #fff; background: linear-gradient(135deg, #ef735a, #c94e3b); font-weight: 700; box-shadow: 0 9px 24px rgba(201,78,59,.22), inset 0 1px rgba(255,255,255,.25); transition: transform .2s, background .2s, box-shadow .2s; }.primary-button:hover { background: linear-gradient(135deg, #fa836c, #d95b46); transform: translateY(-3px); box-shadow: 0 13px 30px rgba(201,78,59,.3), inset 0 1px rgba(255,255,255,.3); }.primary-button span { font-size: 21px; line-height: 12px; }
.task-form { display: flex; gap: 10px; flex-wrap: wrap; padding: 16px; margin-bottom: 28px; border: 1px solid rgba(255,255,255,.12); border-radius: 10px; background: rgba(35,39,43,.82); box-shadow: 0 18px 40px rgba(0,0,0,.18), inset 0 1px rgba(255,255,255,.06); }.task-form input, .task-form select { flex: 1 1 160px; min-width: 0; padding: 12px; border: 1px solid rgba(255,255,255,.12); border-radius: 6px; color: #f3f0e9; background: #1c2024; outline-color: #e85e46; }.task-form .primary-button { flex: 0 0 auto; }
.toolbar { display: flex; align-items: center; gap: 24px; padding: 14px 0; border-top: 1px solid rgba(255,255,255,.1); border-bottom: 1px solid rgba(255,255,255,.1); }.search-box { display: flex; align-items: center; gap: 9px; flex: 1; max-width: 300px; color: #8c969b; }.search-box input { width: 100%; border: 0; outline: 0; color: #f3f0e9; background: transparent; }.search-box span { font-size: 24px; transform: rotate(-20deg); }.filters { display: flex; align-items: center; gap: 5px; }.filter-label { margin-right: 5px; color: #8e969d; font-size: 12px; }.filters button { padding: 7px 10px; border: 0; border-radius: 5px; color: #8e969d; background: transparent; font-size: 12px; }.filters button.active { color: #fff; background: rgba(255,255,255,.12); font-weight: 700; }.accent-picker { display: flex; align-items: center; gap: 8px; color: #8e969d; font-size: 12px; }.accent-picker input { width: 27px; height: 27px; padding: 0; border: 0; border-radius: 50%; background: transparent; cursor: pointer; }.progress { margin-left: auto; color: #8e969d; font-size: 12px; }
.board { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; padding-top: 34px; perspective: 1200px; }.column { min-height: 400px; padding: 16px; border: 1px solid rgba(255,255,255,.1); border-radius: 12px; background: linear-gradient(145deg, rgba(40,45,49,.84), rgba(27,30,33,.76)); box-shadow: 0 22px 45px rgba(0,0,0,.2), inset 0 1px rgba(255,255,255,.06); transition: border-color .25s, transform .25s, box-shadow .25s; }.column.is-drop-target { border-color: rgba(240,123,98,.75); transform: translateY(-5px) rotateX(1deg); box-shadow: 0 28px 55px rgba(232,94,70,.16), inset 0 1px rgba(255,255,255,.14); }.column-heading { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; }.column-heading > div { display: flex; align-items: center; gap: 9px; }.column-heading h2 { margin: 0; font-family: 'Space Grotesk', sans-serif; font-size: 15px; }.status-dot { width: 9px; height: 9px; border-radius: 50%; box-shadow: 0 0 12px currentColor; }.coral { color: #e85e46; background: #e85e46; }.gold { color: #d9a441; background: #d9a441; }.mint { color: #4a9d7b; background: #4a9d7b; }.count { display: grid; place-items: center; width: 23px; height: 23px; border-radius: 5px; color: #a2abb0; background: rgba(255,255,255,.09); font-size: 11px; }.task-list { display: flex; flex-direction: column; gap: 12px; min-height: 350px; padding: 2px; }.task-card { --rotate-x: 0deg; --rotate-y: 0deg; position: relative; padding: 17px; border: 1px solid rgba(255,255,255,.13); border-radius: 8px; color: #e9e6de; background: linear-gradient(145deg, rgba(66,72,77,.9), rgba(33,37,40,.96)); box-shadow: 0 13px 0 rgba(8,10,11,.2), 0 18px 25px rgba(0,0,0,.22), inset 0 1px rgba(255,255,255,.13); transform: perspective(750px) rotateX(var(--rotate-x)) rotateY(var(--rotate-y)); transform-style: preserve-3d; animation: card-in .45s both; cursor: grab; transition: border-color .2s, transform .18s ease-out, box-shadow .2s; }.task-card::before { position: absolute; inset: 0; border-radius: inherit; background: linear-gradient(110deg, rgba(255,255,255,.16), transparent 27%, transparent 72%, rgba(255,255,255,.04)); content: ''; pointer-events: none; }.task-card:hover { border-color: rgba(255,255,255,.34); box-shadow: 0 13px 0 rgba(8,10,11,.2), 0 25px 35px rgba(0,0,0,.3), inset 0 1px rgba(255,255,255,.2); }.task-card:active { cursor: grabbing; }.task-card.is-dragging { opacity: .38; transform: scale(.97) rotateZ(2deg); }.task-card-top { position: relative; display: flex; justify-content: space-between; align-items: center; z-index: 1; }.priority { padding: 4px 7px; border-radius: 4px; font-size: 10px; font-weight: 700; }.priority.alta { color: #ffb1a1; background: rgba(232,94,70,.2); }.priority.média { color: #f1cc7a; background: rgba(217,164,65,.18); }.priority.baixa { color: #8bd5b4; background: rgba(74,157,123,.18); }.delete-button { border: 0; color: #929ba1; background: transparent; font-size: 20px; line-height: 14px; }.delete-button:hover { color: #ff9d8c; }.task-card h3 { position: relative; margin: 15px 0 7px; font-family: 'Space Grotesk', sans-serif; font-size: 15px; z-index: 1; }.task-card p { position: relative; min-height: 35px; margin: 0 0 15px; color: #aab1b3; font-size: 12px; line-height: 1.5; z-index: 1; }.tag { position: relative; display: inline-block; padding: 5px 8px; border-radius: 4px; color: #b9c0c0; background: rgba(255,255,255,.09); font-size: 10px; z-index: 1; }.empty-state { display: grid; place-items: center; min-height: 100px; border: 1px dashed rgba(255,255,255,.18); border-radius: 8px; color: #747e84; font-size: 12px; }
@keyframes card-in { from { opacity: 0; transform: perspective(750px) translateY(12px) rotateX(-4deg); } to { opacity: 1; transform: perspective(750px) rotateX(0) rotateY(0); } }
.task-card { touch-action: none; }.feedback { position: fixed; right: 24px; bottom: 24px; z-index: 10; margin: 0; padding: 12px 16px; border: 1px solid var(--accent); border-radius: 7px; color: #fff; background: #252b2f; box-shadow: 0 12px 30px rgba(0,0,0,.25); font-size: 13px; }.modal-backdrop { position: fixed; inset: 0; display: grid; place-items: center; z-index: 20; padding: 20px; background: rgba(9,12,14,.7); backdrop-filter: blur(8px); }.task-modal { position: relative; width: min(100%, 480px); padding: 30px; border: 1px solid var(--accent); border-radius: 12px; color: #f3f0e9; background: #252b2f; box-shadow: 0 25px 80px rgba(0,0,0,.38); }.theme-light .task-modal { color: #20252a; background: #f8faf7; }.modal-close { position: absolute; top: 12px; right: 14px; border: 0; color: inherit; background: transparent; font-size: 25px; }.task-modal h2 { margin: 18px 0 10px; font-family: 'Space Grotesk', sans-serif; }.task-modal p { color: #aab1b3; line-height: 1.6; }.theme-light .task-modal p { color: #687278; }.task-modal dl { display: flex; gap: 28px; margin: 26px 0 0; }.task-modal dt { color: #8e969d; font-size: 11px; }.task-modal dd { margin: 5px 0 0; font-weight: 700; }.skeleton-column { animation: pulse 1.2s ease-in-out infinite alternate; }.skeleton-line, .skeleton-card { border-radius: 6px; background: rgba(255,255,255,.12); }.skeleton-line { width: 38%; height: 18px; margin-bottom: 28px; }.skeleton-card { height: 105px; margin-bottom: 12px; }.modal-enter-active, .modal-leave-active, .toast-enter-active, .toast-leave-active { transition: opacity .2s, transform .2s; }.modal-enter-from, .modal-leave-to { opacity: 0; }.modal-enter-from .task-modal, .modal-leave-to .task-modal { transform: translateY(12px) scale(.98); }.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateY(10px); }
@keyframes pulse { from { opacity: .55; } to { opacity: 1; } }
@media (max-width: 760px) { .app-shell { padding: 0 20px 36px; }.date-label { display: none; }.workspace-heading { align-items: start; flex-direction: column; padding: 42px 0 30px; }.workspace-heading h1 { letter-spacing: -1.8px; }.toolbar { align-items: stretch; flex-direction: column; gap: 12px; }.search-box { max-width: none; }.accent-picker { justify-content: space-between; }.progress { margin-left: 0; }.board { grid-template-columns: 1fr; gap: 28px; }.column { min-height: auto; }.task-list { min-height: 90px; }.feedback { right: 20px; bottom: 18px; left: 20px; text-align: center; }.task-modal { padding: 25px; } }
</style>
