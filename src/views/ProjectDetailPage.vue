<script setup>
import { computed, onMounted, reactive, ref, watch } from 'vue'
import { storeToRefs } from 'pinia'
import { useRouter } from 'vue-router'
import { useTitle } from '@vueuse/core'
import AppButton from '@/components/AppButton.vue'
import FormField from '@/components/FormField.vue'
import TaskItem from '@/components/TaskItem.vue'
import { useProjectsStore } from '@/stores/projects'
import { useTasksStore } from '@/stores/tasks'
import { useAsync } from '@/composables/useAsync'
import { TASK_STATUSES, TASK_STATUS_LABELS } from '@/lib/constants'

const props = defineProps({
  id: { type: Number, required: true },
})

const router = useRouter()
const projectsStore = useProjectsStore()
const tasksStore = useTasksStore()
const { list: tasks, loading: tasksLoading } = storeToRefs(tasksStore)

const project = ref(null)
const loadingProject = ref(false)
const loadError = ref(null)

useTitle(() => (project.value ? `${project.value.name} · Projects` : 'Project'))

const editingProject = ref(false)
const projectForm = reactive({ name: '', description: '' })
const { loading: projectLoading, fieldErrors: projectFieldErrors, run: runProject } = useAsync()

const newTask = reactive({ title: '', description: '', status: 'todo' })
const {
  loading: newTaskLoading,
  error: newTaskError,
  fieldErrors: newTaskFieldErrors,
  run: runNewTask,
} = useAsync()

const grouped = computed(() => {
  const groups = Object.fromEntries(TASK_STATUSES.map((s) => [s, []]))
  for (const task of tasks.value) {
    if (groups[task.status]) groups[task.status].push(task)
  }
  return groups
})

const stats = computed(() => {
  const total = tasks.value.length
  const done = tasks.value.filter((t) => t.status === 'done').length
  const pct = total === 0 ? 0 : Math.round((done / total) * 100)
  return { total, done, pct }
})

async function loadProject() {
  loadingProject.value = true
  loadError.value = null
  try {
    project.value = await projectsStore.getById(props.id)
  } catch (err) {
    loadError.value = err
  } finally {
    loadingProject.value = false
  }
}

function startEdit() {
  projectForm.name = project.value.name
  projectForm.description = project.value.description ?? ''
  editingProject.value = true
}

async function saveProject() {
  try {
    const updated = await runProject(() => projectsStore.update(props.id, { ...projectForm }))
    project.value = updated
    editingProject.value = false
  } catch {
    // shown in form
  }
}

async function deleteProject() {
  if (!window.confirm(`Delete project "${project.value.name}" and all its tasks?`)) return
  await projectsStore.remove(props.id)
  router.replace('/projects')
}

async function addTask() {
  try {
    await runNewTask(() => tasksStore.create(props.id, { ...newTask }))
    newTask.title = ''
    newTask.description = ''
    newTask.status = 'todo'
  } catch {
    // shown in form
  }
}

async function onTaskStatus(task, status) {
  await tasksStore.update(task.id, { status })
}

async function onTaskDelete(task) {
  if (!window.confirm(`Delete task "${task.title}"?`)) return
  await tasksStore.remove(task.id)
}

onMounted(async () => {
  await loadProject()
  if (project.value) await tasksStore.fetchByProject(props.id).catch(() => {})
})

watch(
  () => props.id,
  async (newId, oldId) => {
    if (newId === oldId) return
    tasksStore.reset()
    await loadProject()
    if (project.value) await tasksStore.fetchByProject(newId).catch(() => {})
  },
)
</script>

<template>
  <main class="mx-auto w-full max-w-5xl flex-1 px-5 py-8">
    <RouterLink
      to="/projects"
      class="mono text-text-3 hover:text-text mb-6 inline-flex items-center gap-1.5"
    >
      <span aria-hidden="true">←</span> Dashboard
    </RouterLink>

    <p
      v-if="loadingProject"
      class="text-text-2 text-sm"
    >
      Loading project…
    </p>

    <p
      v-else-if="loadError"
      class="border-danger/30 bg-danger-soft text-danger flex items-center gap-2 rounded-md border px-4 py-3 text-sm"
      role="alert"
      aria-live="polite"
    >
      <span aria-hidden="true">⚠</span>
      {{ loadError.message }}
    </p>

    <template v-else-if="project">
      <!-- Header card -->
      <header class="rise rise-1 card mb-6 p-6">
        <div
          v-if="!editingProject"
          class="flex flex-wrap items-start justify-between gap-4"
        >
          <div class="min-w-0 flex-1">
            <div class="mb-2 flex items-center gap-2">
              <span class="chip">#{{ project.id }}</span>
              <span class="chip">{{ stats.total }} tasks</span>
              <span
                v-if="stats.total > 0"
                class="chip chip-accent"
                >{{ stats.pct }}% done</span
              >
            </div>
            <h1 class="text-text text-2xl font-semibold tracking-tight">
              {{ project.name }}
            </h1>
            <p
              v-if="project.description"
              class="text-text-2 mt-1.5 text-sm"
            >
              {{ project.description }}
            </p>
          </div>
          <div class="flex shrink-0 gap-2">
            <AppButton
              variant="ghost"
              size="sm"
              @click="startEdit"
              >Edit</AppButton
            >
            <AppButton
              variant="danger"
              size="sm"
              @click="deleteProject"
              >Delete</AppButton
            >
          </div>
        </div>

        <form
          v-else
          class="flex flex-col gap-4"
          @submit.prevent="saveProject"
        >
          <FormField
            v-model="projectForm.name"
            label="Name"
            required
            :error="projectFieldErrors.name"
          />
          <FormField
            v-model="projectForm.description"
            label="Description"
            :error="projectFieldErrors.description"
          />
          <div class="flex justify-end gap-2">
            <AppButton
              variant="ghost"
              type="button"
              @click="editingProject = false"
              >Cancel</AppButton
            >
            <AppButton
              type="submit"
              :loading="projectLoading"
            >
              {{ projectLoading ? 'Saving…' : 'Save changes' }}
            </AppButton>
          </div>
        </form>

        <!-- Progress bar -->
        <div
          v-if="!editingProject && stats.total > 0"
          class="mt-5"
        >
          <div class="text-text-3 mb-1.5 flex items-center justify-between text-xs">
            <span>Progress</span>
            <span class="mono">{{ stats.done }}/{{ stats.total }}</span>
          </div>
          <div class="bg-surface-2 h-1.5 overflow-hidden rounded-full">
            <div
              class="bg-accent h-full rounded-full transition-[width] duration-500"
              :style="{ width: `${stats.pct}%` }"
            />
          </div>
        </div>
      </header>

      <!-- Add task -->
      <section class="rise rise-2 card mb-6 p-5">
        <h2 class="text-text mb-4 text-sm font-semibold">Add task</h2>
        <form
          class="grid gap-4 md:grid-cols-[1.5fr_2fr_auto_auto] md:items-end"
          @submit.prevent="addTask"
        >
          <FormField
            v-model="newTask.title"
            label="Title"
            required
            placeholder="What needs doing?"
            :error="newTaskFieldErrors.title"
          />
          <FormField
            v-model="newTask.description"
            label="Description"
            placeholder="Optional details"
            :error="newTaskFieldErrors.description"
          />
          <div class="flex flex-col gap-1.5">
            <label
              for="new-task-status"
              class="text-text-2 text-xs font-medium"
              >Status</label
            >
            <select
              id="new-task-status"
              v-model="newTask.status"
              class="border-border bg-surface text-text hover:border-border-strong focus:border-accent focus:ring-accent/30 h-10 rounded-md border px-3 text-sm transition-colors focus:ring-2 focus:outline-none"
            >
              <option
                v-for="s in TASK_STATUSES"
                :key="s"
                :value="s"
              >
                {{ TASK_STATUS_LABELS[s] }}
              </option>
            </select>
          </div>
          <AppButton
            type="submit"
            :loading="newTaskLoading"
          >
            {{ newTaskLoading ? 'Adding…' : 'Add task' }}
          </AppButton>
          <p
            v-if="newTaskError && Object.keys(newTaskFieldErrors).length === 0"
            class="border-danger/30 bg-danger-soft text-danger flex items-center gap-2 rounded-md border px-3 py-2 text-xs md:col-span-4"
            role="alert"
            aria-live="polite"
          >
            <span aria-hidden="true">⚠</span>
            {{ newTaskError.message }}
          </p>
        </form>
      </section>

      <!-- Tasks -->
      <section>
        <p
          v-if="tasksLoading && tasks.length === 0"
          class="text-text-2 text-sm"
        >
          Loading tasks…
        </p>
        <div
          v-else-if="tasks.length === 0"
          class="card flex flex-col items-center gap-2 p-10 text-center"
        >
          <span
            class="text-2xl"
            aria-hidden="true"
            >✦</span
          >
          <h3 class="text-text text-sm font-semibold">No tasks yet</h3>
          <p class="text-text-2 text-sm">Add your first task above to get started.</p>
        </div>
        <div
          v-else
          class="flex flex-col gap-6"
        >
          <div
            v-for="status in TASK_STATUSES"
            :key="status"
          >
            <div class="mb-2.5 flex items-center justify-between">
              <h2
                class="text-text-2 flex items-center gap-2 text-xs font-semibold tracking-wide uppercase"
              >
                {{ TASK_STATUS_LABELS[status] }}
                <span class="mono text-text-3">{{ grouped[status].length }}</span>
              </h2>
            </div>
            <ul
              v-if="grouped[status].length > 0"
              class="flex flex-col gap-2"
            >
              <TaskItem
                v-for="task in grouped[status]"
                :key="task.id"
                :task="task"
                @update-status="(s) => onTaskStatus(task, s)"
                @delete="onTaskDelete"
              />
            </ul>
            <p
              v-else
              class="border-border text-text-3 rounded-md border border-dashed px-4 py-3 text-xs"
            >
              Nothing here.
            </p>
          </div>
        </div>
      </section>
    </template>
  </main>
</template>
