<script setup lang="ts">
import { computed, ref, watch } from 'vue'

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

watch(tasks, (value) => localStorage.setItem('vue-kanban-tasks', JSON.stringify(value)), { deep: true })

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
  if (!newTask.value.title.trim()) return
  tasks.value.unshift({ id: Date.now(), title: newTask.value.title.trim(), description: newTask.value.description.trim() || 'Sem descrição', priority: newTask.value.priority, tag: newTask.value.tag.trim() || 'Geral', column: 'backlog' })
  newTask.value = { title: '', description: '', priority: 'Média', tag: 'Geral' }
  isFormOpen.value = false
}

function deleteTask(id: number) {
  tasks.value = tasks.value.filter((task) => task.id !== id)
}

function moveTask(column: ColumnId) {
  const task = tasks.value.find((item) => item.id === draggedTaskId.value)
  if (task) task.column = column
  draggedTaskId.value = null
  dragOverColumn.value = null
}

function startDragging(id: number) {
  draggedTaskId.value = id
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
  <main class="app-shell">
    <header class="topbar">
      <div class="brand"><span class="brand-mark">+</span><span>Taskroom</span></div>
      <div class="topbar-actions"><span class="date-label">Terça-feira, 19 de agosto</span><button class="avatar" aria-label="Perfil">GT</button></div>
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
      <span class="progress">{{ completedCount }} de {{ tasks.length }} concluídas</span>
    </section>

    <section class="board">
      <article v-for="column in columns" :key="column.id" :class="['column', { 'is-drop-target': dragOverColumn === column.id }]" @dragover.prevent="dragOverColumn = column.id" @dragleave="dragOverColumn = null" @drop="moveTask(column.id)">
        <div class="column-heading"><div><span :class="['status-dot', column.color]"></span><h2>{{ column.title }}</h2></div><span class="count">{{ tasksForColumn(column.id).length }}</span></div>
        <div class="task-list">
          <div v-for="task in tasksForColumn(column.id)" :key="task.id" :class="['task-card', { 'is-dragging': draggedTaskId === task.id }]" draggable="true" @dragstart="startDragging(task.id)" @dragend="draggedTaskId = null; dragOverColumn = null" @mousemove="updateCardTilt" @mouseleave="resetCardTilt">
            <div class="task-card-top"><span :class="['priority', task.priority.toLowerCase()]">{{ task.priority }}</span><button class="delete-button" aria-label="Excluir tarefa" @click="deleteTask(task.id)">×</button></div>
            <h3>{{ task.title }}</h3><p>{{ task.description }}</p><span class="tag">{{ task.tag }}</span>
          </div>
          <div v-if="tasksForColumn(column.id).length === 0" class="empty-state">Solte uma tarefa aqui</div>
        </div>
      </article>
    </section>
  </main>
</template>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=Space+Grotesk:wght@500;600;700&display=swap');

:root { color: #252525; background: #17191c; font-family: 'DM Sans', sans-serif; font-synthesis: none; }
* { box-sizing: border-box; }
body { margin: 0; min-width: 320px; background: #17191c; }
button, input, select { font: inherit; }
button { cursor: pointer; }
.app-shell { min-height: 100vh; padding: 0 6vw 56px; color: #f3f0e9; background: radial-gradient(circle at 92% -4%, rgba(232, 94, 70, .22), transparent 24%), radial-gradient(circle at 8% 45%, rgba(56, 143, 151, .14), transparent 24%), #17191c; background-image: radial-gradient(rgba(255,255,255,.045) 1px, transparent 1px), radial-gradient(circle at 92% -4%, rgba(232, 94, 70, .22), transparent 24%), radial-gradient(circle at 8% 45%, rgba(56, 143, 151, .14), transparent 24%), linear-gradient(135deg, rgba(255,255,255,.025) 25%, transparent 25%); background-size: 28px 28px, auto, auto, 5px 5px; }
.topbar { height: 76px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid rgba(255,255,255,.1); }
.brand { display: flex; align-items: center; gap: 10px; font-family: 'Space Grotesk', sans-serif; font-size: 18px; font-weight: 700; letter-spacing: -.4px; }
.brand-mark { display: grid; place-items: center; width: 27px; height: 27px; color: #fff; background: #e85e46; border-radius: 8px; font-size: 21px; line-height: 1; box-shadow: 0 0 20px rgba(232,94,70,.35); }.topbar-actions { display: flex; align-items: center; gap: 20px; color: #8e969d; font-size: 13px; }.avatar { border: 1px solid rgba(255,255,255,.2); width: 34px; height: 34px; border-radius: 50%; color: #fff; background: #334e68; font-size: 12px; font-weight: 700; box-shadow: 0 0 0 5px rgba(51,78,104,.12); }
.workspace-heading { display: flex; justify-content: space-between; align-items: end; gap: 24px; padding: 62px 0 38px; }.eyebrow { margin: 0 0 14px; color: #f07b62; font-size: 12px; font-weight: 700; letter-spacing: 1.5px; text-transform: uppercase; }.workspace-heading h1 { margin: 0; max-width: 680px; font-family: 'Space Grotesk', sans-serif; font-size: clamp(34px, 5vw, 58px); letter-spacing: -2.8px; line-height: .98; text-shadow: 0 3px 0 rgba(255,255,255,.04); }.subtitle { margin: 18px 0 0; color: #929ba1; font-size: 16px; }.primary-button { display: inline-flex; align-items: center; gap: 8px; padding: 13px 17px; border: 1px solid rgba(255,255,255,.16); border-radius: 7px; color: #fff; background: linear-gradient(135deg, #ef735a, #c94e3b); font-weight: 700; box-shadow: 0 9px 24px rgba(201,78,59,.22), inset 0 1px rgba(255,255,255,.25); transition: transform .2s, background .2s, box-shadow .2s; }.primary-button:hover { background: linear-gradient(135deg, #fa836c, #d95b46); transform: translateY(-3px); box-shadow: 0 13px 30px rgba(201,78,59,.3), inset 0 1px rgba(255,255,255,.3); }.primary-button span { font-size: 21px; line-height: 12px; }
.task-form { display: flex; gap: 10px; flex-wrap: wrap; padding: 16px; margin-bottom: 28px; border: 1px solid rgba(255,255,255,.12); border-radius: 10px; background: rgba(35,39,43,.82); box-shadow: 0 18px 40px rgba(0,0,0,.18), inset 0 1px rgba(255,255,255,.06); }.task-form input, .task-form select { flex: 1 1 160px; min-width: 0; padding: 12px; border: 1px solid rgba(255,255,255,.12); border-radius: 6px; color: #f3f0e9; background: #1c2024; outline-color: #e85e46; }.task-form .primary-button { flex: 0 0 auto; }
.toolbar { display: flex; align-items: center; gap: 24px; padding: 14px 0; border-top: 1px solid rgba(255,255,255,.1); border-bottom: 1px solid rgba(255,255,255,.1); }.search-box { display: flex; align-items: center; gap: 9px; flex: 1; max-width: 300px; color: #8c969b; }.search-box input { width: 100%; border: 0; outline: 0; color: #f3f0e9; background: transparent; }.search-box span { font-size: 24px; transform: rotate(-20deg); }.filters { display: flex; align-items: center; gap: 5px; }.filter-label { margin-right: 5px; color: #8e969d; font-size: 12px; }.filters button { padding: 7px 10px; border: 0; border-radius: 5px; color: #8e969d; background: transparent; font-size: 12px; }.filters button.active { color: #fff; background: rgba(255,255,255,.12); font-weight: 700; }.progress { margin-left: auto; color: #8e969d; font-size: 12px; }
.board { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; padding-top: 34px; perspective: 1200px; }.column { min-height: 400px; padding: 16px; border: 1px solid rgba(255,255,255,.1); border-radius: 12px; background: linear-gradient(145deg, rgba(40,45,49,.84), rgba(27,30,33,.76)); box-shadow: 0 22px 45px rgba(0,0,0,.2), inset 0 1px rgba(255,255,255,.06); transition: border-color .25s, transform .25s, box-shadow .25s; }.column.is-drop-target { border-color: rgba(240,123,98,.75); transform: translateY(-5px) rotateX(1deg); box-shadow: 0 28px 55px rgba(232,94,70,.16), inset 0 1px rgba(255,255,255,.14); }.column-heading { display: flex; justify-content: space-between; align-items: center; margin-bottom: 14px; }.column-heading > div { display: flex; align-items: center; gap: 9px; }.column-heading h2 { margin: 0; font-family: 'Space Grotesk', sans-serif; font-size: 15px; }.status-dot { width: 9px; height: 9px; border-radius: 50%; box-shadow: 0 0 12px currentColor; }.coral { color: #e85e46; background: #e85e46; }.gold { color: #d9a441; background: #d9a441; }.mint { color: #4a9d7b; background: #4a9d7b; }.count { display: grid; place-items: center; width: 23px; height: 23px; border-radius: 5px; color: #a2abb0; background: rgba(255,255,255,.09); font-size: 11px; }.task-list { display: flex; flex-direction: column; gap: 12px; min-height: 350px; padding: 2px; }.task-card { --rotate-x: 0deg; --rotate-y: 0deg; position: relative; padding: 17px; border: 1px solid rgba(255,255,255,.13); border-radius: 8px; color: #e9e6de; background: linear-gradient(145deg, rgba(66,72,77,.9), rgba(33,37,40,.96)); box-shadow: 0 13px 0 rgba(8,10,11,.2), 0 18px 25px rgba(0,0,0,.22), inset 0 1px rgba(255,255,255,.13); transform: perspective(750px) rotateX(var(--rotate-x)) rotateY(var(--rotate-y)); transform-style: preserve-3d; animation: card-in .45s both; cursor: grab; transition: border-color .2s, transform .18s ease-out, box-shadow .2s; }.task-card::before { position: absolute; inset: 0; border-radius: inherit; background: linear-gradient(110deg, rgba(255,255,255,.16), transparent 27%, transparent 72%, rgba(255,255,255,.04)); content: ''; pointer-events: none; }.task-card:hover { border-color: rgba(255,255,255,.34); box-shadow: 0 13px 0 rgba(8,10,11,.2), 0 25px 35px rgba(0,0,0,.3), inset 0 1px rgba(255,255,255,.2); }.task-card:active { cursor: grabbing; }.task-card.is-dragging { opacity: .38; transform: scale(.97) rotateZ(2deg); }.task-card-top { position: relative; display: flex; justify-content: space-between; align-items: center; z-index: 1; }.priority { padding: 4px 7px; border-radius: 4px; font-size: 10px; font-weight: 700; }.priority.alta { color: #ffb1a1; background: rgba(232,94,70,.2); }.priority.média { color: #f1cc7a; background: rgba(217,164,65,.18); }.priority.baixa { color: #8bd5b4; background: rgba(74,157,123,.18); }.delete-button { border: 0; color: #929ba1; background: transparent; font-size: 20px; line-height: 14px; }.delete-button:hover { color: #ff9d8c; }.task-card h3 { position: relative; margin: 15px 0 7px; font-family: 'Space Grotesk', sans-serif; font-size: 15px; z-index: 1; }.task-card p { position: relative; min-height: 35px; margin: 0 0 15px; color: #aab1b3; font-size: 12px; line-height: 1.5; z-index: 1; }.tag { position: relative; display: inline-block; padding: 5px 8px; border-radius: 4px; color: #b9c0c0; background: rgba(255,255,255,.09); font-size: 10px; z-index: 1; }.empty-state { display: grid; place-items: center; min-height: 100px; border: 1px dashed rgba(255,255,255,.18); border-radius: 8px; color: #747e84; font-size: 12px; }
@keyframes card-in { from { opacity: 0; transform: perspective(750px) translateY(12px) rotateX(-4deg); } to { opacity: 1; transform: perspective(750px) rotateX(0) rotateY(0); } }
@media (max-width: 760px) { .app-shell { padding: 0 20px 36px; }.date-label { display: none; }.workspace-heading { align-items: start; flex-direction: column; padding: 42px 0 30px; }.workspace-heading h1 { letter-spacing: -1.8px; }.toolbar { align-items: stretch; flex-direction: column; gap: 12px; }.search-box { max-width: none; }.progress { margin-left: 0; }.board { grid-template-columns: 1fr; gap: 28px; }.column { min-height: auto; }.task-list { min-height: 90px; } }
</style>
