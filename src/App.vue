<script setup>
import { ref, onMounted } from 'vue'

const messages = ref([])
const content = ref('')
const loading = ref(false)
const error = ref('')
const health = ref(null)

const editingId = ref(null)
const editingText = ref('')

async function load() {
  try {
    error.value = ''
    const res = await fetch('/api/messages')
    if (!res.ok) throw new Error('HTTP ' + res.status)
    messages.value = await res.json()
  } catch (e) {
    error.value = '加载失败: ' + e.message
  }
}

async function loadHealth() {
  try {
    const res = await fetch('/api/health')
    health.value = await res.json()
  } catch (e) {
    health.value = null
  }
}

async function add() {
  if (!content.value.trim()) return
  loading.value = true
  try {
    error.value = ''
    const res = await fetch('/api/messages', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content: content.value.trim() })
    })
    if (!res.ok) throw new Error('HTTP ' + res.status)
    content.value = ''
    await load()
    await loadHealth()
  } catch (e) {
    error.value = '提交失败: ' + e.message
  } finally {
    loading.value = false
  }
}

function startEdit(m) {
  editingId.value = m.id
  editingText.value = m.content
}

function cancelEdit() {
  editingId.value = null
  editingText.value = ''
}

async function saveEdit(id) {
  if (!editingText.value.trim()) return
  try {
    error.value = ''
    const res = await fetch('/api/messages/' + id, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ content: editingText.value.trim() })
    })
    if (!res.ok) throw new Error('HTTP ' + res.status)
    cancelEdit()
    await load()
  } catch (e) {
    error.value = '修改失败: ' + e.message
  }
}

async function remove(id) {
  if (!window.confirm('确定删除 #' + id + ' 吗？')) return
  try {
    error.value = ''
    const res = await fetch('/api/messages/' + id, { method: 'DELETE' })
    if (!res.ok && res.status !== 204) throw new Error('HTTP ' + res.status)
    await load()
    await loadHealth()
  } catch (e) {
    error.value = '删除失败: ' + e.message
  }
}

onMounted(async () => {
  await load()
  await loadHealth()
})
</script>

<template>
  <main class="card">
    <h1>K8s 全栈演示</h1>
    <p class="sub">Ingress → Service → Pod → MySQL 完整链路（增 / 删 / 改 / 查）</p>

    <div v-if="health" class="health">
      <span class="badge">Pod: {{ health.pod }}</span>
      <span class="badge">记录数: {{ health.count }}</span>
      <span class="badge">{{ health.status }}</span>
    </div>

    <form class="form" @submit.prevent="add">
      <input v-model="content" placeholder="输入一条消息…" maxlength="255" />
      <button :disabled="loading">{{ loading ? '提交中…' : '新增' }}</button>
    </form>

    <p v-if="error" class="error">{{ error }}</p>

    <ul class="list">
      <li v-for="m in messages" :key="m.id">
        <template v-if="editingId === m.id">
          <input
            class="edit-input"
            v-model="editingText"
            maxlength="255"
            @keyup.enter="saveEdit(m.id)"
            @keyup.esc="cancelEdit"
          />
          <div class="ops">
            <button class="mini ok" @click="saveEdit(m.id)">保存</button>
            <button class="mini" @click="cancelEdit">取消</button>
          </div>
        </template>
        <template v-else>
          <span class="id">#{{ m.id }}</span>
          <span class="txt">{{ m.content }}</span>
          <div class="ops">
            <button class="mini" @click="startEdit(m)">编辑</button>
            <button class="mini danger" @click="remove(m.id)">删除</button>
          </div>
        </template>
      </li>
      <li v-if="messages.length === 0" class="empty">暂无数据，先新增一条吧</li>
    </ul>
  </main>
</template>
