<script setup>
/**
 * Day5 · Todo 待办清单 · A2 SOP 四步实战（进阶）
 *
 * 核心功能：增 / 删 / 改 / 查 + localStorage 持久化
 * - 输入框回车添加
 * - 勾选框切换完成态
 * - 双击文本进入编辑态（回车存 / Esc 取消）
 * - 删除按钮
 * - 空状态提示
 * - localStorage 自动保存 / 恢复
 *
 * 三态处理：空数据（✅ 已有）/ 正常态（✅）/ 异常态（localStorage 禁用时降级提示）
 */
import {  computed, onMounted, ref, watch } from 'vue'

// ============== 数据层 ==============
const STORAGE_KEY = 'vibe-todo-list'
const todos = ref([])            // [{ id, text, done }]
const input = ref('')            // 输入框
const editingId = ref(null)      // 正在编辑的 todo id
const editingText = ref('')      // 编辑中的临时文本
const storageDisabled = ref(false)

// localStorage 降级检测（隐私模式 / 禁用 cookie 时）
function canUseLocalStorage() {
  try {
    localStorage.setItem('__test__', '1')
    localStorage.removeItem('__test__')
    return true
  } catch {
    return false
  }
}

function loadTodos() {
  if (!canUseLocalStorage()) {
    storageDisabled.value = true
    // 用内存 mock 数据让页面不空
    todos.value = [
      { id: 1, text: '测试条目 1（localStorage 不可用，仅本会话有效）', done: false },
    ]
    return
  }
  const raw = localStorage.getItem(STORAGE_KEY)
  todos.value = raw ? JSON.parse(raw) : []
}

function saveTodos() {
  if (canUseLocalStorage()) {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(todos.value))
  }
}

// watch 自动持久化（任何变更都存）
watch(todos, saveTodos, { deep: true })
const filter = ref('all')

const filteredTodos = computed(() => {
  if (filter.value === 'active') return todos.value.filter(t => !t.done)
  if (filter.value === 'done')   return todos.value.filter(t => t.done)
  return todos.value
})
// ============== CRUD 四步 ==============
function addTodo() {
  const text = input.value.trim()
  if (!text) return
  todos.value.push({
    id: Date.now(), // 用时间戳当简易唯一 id
    text,
    done: false,
  })
  input.value = ''
}

function toggle(id) {
  const t = todos.value.find((x) => x.id === id)
  if (t) t.done = !t.done
}

function remove(id) {
  todos.value = todos.value.filter((x) => x.id !== id)
}

function startEdit(todo) {
  editingId.value = todo.id
  editingText.value = todo.text
}

function saveEdit() {
  if (editingId.value === null) return
  const t = todos.value.find((x) => x.id === editingId.value)
  if (t) {
    const newText = editingText.value.trim()
    if (newText) t.text = newText
    else remove(editingId.value) // 编辑为空则删除
  }
  editingId.value = null
  editingText.value = ''
}

function cancelEdit() {
  editingId.value = null
  editingText.value = ''
}

function clearDone() {
  todos.value = todos.value.filter((x) => !x.done)
}

// ============== 计算属性 ==============
const remainingCount = () => todos.value.filter((x) => !x.done).length
const doneCount = () => todos.value.filter((x) => x.done).length

// ============== 生命周期 ==============
onMounted(loadTodos)
</script>

<template>
  <div class="min-h-screen flex flex-col items-center px-4 py-10">
    <!-- 标题 -->
    <h1 class="text-2xl font-medium text-[#1a1a18] tracking-wide">
      Todo · 待办清单
    </h1>
    <p class="mt-1 text-xs text-[#918a7d]">
      Day 5 · CRUD + localStorage 持久化
    </p>

    <!-- 输入区 -->
    <form
      class="mt-8 flex w-full max-w-md gap-2"
      @submit.prevent="addTodo"
    >
      <input
        v-model="input"
        type="text"
        placeholder="输入待办，回车添加…"
        class="flex-1 border border-[#b8b0a2] rounded px-3 py-2 text-sm bg-white focus:outline-none focus:border-[#1a1a18]"
        maxlength="120"
      />
      <button
        type="submit"
        :disabled="!input.trim()"
        class="px-4 py-2 rounded bg-[#1a1a18] text-white text-sm hover:bg-[#45413b] transition disabled:opacity-40 disabled:cursor-not-allowed"
      >
        添加
      </button>
    </form>

    <!-- localStorage 降级提示 -->
    <p
      v-if="storageDisabled"
      class="mt-4 w-full max-w-md text-[11px] text-[#b9322b] border border-[#b9322b]/30 rounded px-3 py-2 bg-[#f5e2e0]/50"
    >
      当前浏览器禁用了 localStorage，待办仅本会话有效，刷新会丢失。
    </p>

    <!-- 列表 -->
    <ul class="mt-6 w-full max-w-md space-y-2">
      <li
        v-for="todo in filteredTodos"
        :key="todo.id"
        class="flex items-center gap-3 border border-[#d4cdbf] rounded px-3 py-2 bg-white hover:bg-[#f5f0e4]/60 transition"
      >
        <!-- 勾选框 -->
        <input
          type="checkbox"
          :checked="todo.done"
          @change="toggle(todo.id)"
          class="h-4 w-4 accent-[#1a1a18] cursor-pointer"
          aria-label="标记完成"
        />

        <!-- 文本 -->
        <template v-if="editingId === todo.id">
          <input
            v-model="editingText"
            type="text"
            class="flex-1 border border-[#1a1a18] rounded px-2 py-0.5 text-sm focus:outline-none"
            @keyup.enter="saveEdit"
            @keyup.esc="cancelEdit"
            @blur="saveEdit"
            autofocus
          />
        </template>
        <template v-else>
          <span
            class="flex-1 text-sm text-[#1a1a18] cursor-pointer select-none"
            :class="{ 'line-through text-[#918a7d]': todo.done }"
            @dblclick="startEdit(todo)"
            title="双击编辑"
          >
            {{ todo.text }}
          </span>
        </template>

        <!-- 删除 -->
        <button
          type="button"
          class="text-[11px] text-[#b9322b] hover:underline"
          @click="remove(todo.id)"
        >
          删除
        </button>
      </li>

      <!-- 空状态 -->
      <li
        v-if="todos.length === 0"
        class="border border-dashed border-[#b8b0a2] rounded px-3 py-8 text-center text-sm text-[#918a7d]"
      >
        还没有待办，在上面输入第一条吧 ✨
      </li>
    </ul>
<div class="mt-4 w-full max-w-md flex gap-2 text-xs">
  <button
    type="button"
    @click="filter = 'all'"
    :class="{
      'bg-[#1a1a18] text-white rounded px-3 py-1': filter === 'all',
      'bg-white border border-[#b8b0a2] text-[#918a7d] rounded px-3 py-1': filter !== 'all'
    }"
    class="transition"
  >
    全部
  </button>
    <button
    type="button"
    @click="filter = 'active'"
    :class="{
      'bg-[#1a1a18] text-white rounded px-3 py-1': filter === 'active',
      'bg-white border border-[#b8b0a2] text-[#918a7d] rounded px-3 py-1': filter !== 'active'
    }"
    class="transition"
  >
    未完成
  </button>

  <button
    type="button"
    @click="filter = 'done'"
    :class="{
      'bg-[#1a1a18] text-white rounded px-3 py-1': filter === 'done',
      'bg-white border border-[#b8b0a2] text-[#918a7d] rounded px-3 py-1': filter !== 'done'
    }"
    class="transition"
  >
    已完成
  </button>
</div>
    <!-- 底部统计 -->
    <div
      v-if="todos.length > 0"
      class="mt-6 w-full max-w-md flex items-center justify-between text-[11px] text-[#918a7d]"
    >
      <span>
        剩余 {{ remainingCount() }} / 已完成 {{ doneCount() }}
      </span>
      <button
        v-if="doneCount() > 0"
        type="button"
        class="hover:underline text-[#45413b]"
        @click="clearDone"
      >
        清除已完成
      </button>
    </div>
  </div>
</template>
