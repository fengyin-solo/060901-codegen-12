<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useRoom } from '@/composables/useRoom'
import { useExpire } from '@/composables/useExpire'
import type { Memory, Topic } from '@/types'
import { TOPIC_COLORS, TOPIC_EMOJIS } from '@/types'

const route = useRoute()
const router = useRouter()
const { loadRoom, currentRoom, loadRooms, addMemory, removeMemory } = useRoom()
const { isRoomExpired } = useExpire()

const roomId = computed(() => route.params.id as string)

const showAddMemory = ref(false)
const photoPreview = ref('')
const photoData = ref('')
const caption = ref('')
const authorName = ref('')
const selectedTopicId = ref('')
const selectedTopicContent = ref('')

const flippedTopics = computed(() =>
  (currentRoom.value?.topics.filter((t: Topic) => t.isFlipped) || []) as Topic[]
)

const sortedMemories = computed(() => {
  if (!currentRoom.value?.memories) return []
  return [...currentRoom.value.memories].sort(
    (a: Memory, b: Memory) => new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime()
  )
})

const timelineGroups = computed(() => {
  const groups: { date: string; label: string; memories: Memory[] }[] = []
  const map = new Map<string, Memory[]>()

  for (const m of sortedMemories.value) {
    const d = new Date(m.createdAt)
    const key = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(d.getDate()).padStart(2, '0')}`
    if (!map.has(key)) {
      map.set(key, [])
    }
    map.get(key)!.push(m)
  }

  for (const [date, memories] of map) {
    const d = new Date(date)
    const label = d.toLocaleDateString('zh-CN', { month: 'long', day: 'numeric', weekday: 'short' })
    groups.push({ date, label, memories })
  }

  return groups
})

onMounted(() => {
  loadRooms()
  const success = loadRoom(roomId.value)
  if (!success) {
    router.push('/')
    return
  }

  if (currentRoom.value && isRoomExpired(currentRoom.value.expiresAt)) {
    alert('房间已过期')
    router.push('/')
    return
  }

  if (currentRoom.value?.members.length) {
    authorName.value = currentRoom.value.members[0].name
  }
})

const handlePhotoUpload = (event: Event) => {
  const input = event.target as HTMLInputElement
  if (!input.files || !input.files[0]) return

  const file = input.files[0]
  if (!file.type.startsWith('image/')) {
    alert('请选择图片文件')
    return
  }

  const reader = new FileReader()
  reader.onload = (e) => {
    const result = e.target?.result as string
    photoData.value = result
    photoPreview.value = result
  }
  reader.readAsDataURL(file)
}

const selectTopic = (topic: Topic) => {
  if (selectedTopicId.value === topic.id) {
    selectedTopicId.value = ''
    selectedTopicContent.value = ''
  } else {
    selectedTopicId.value = topic.id
    selectedTopicContent.value = topic.content
  }
}

const handleAddMemory = () => {
  if (!photoData.value || !currentRoom.value) return

  addMemory(
    roomId.value,
    photoData.value,
    authorName.value || '匿名',
    selectedTopicId.value || undefined,
    selectedTopicContent.value || undefined,
    caption.value
  )

  photoData.value = ''
  photoPreview.value = ''
  caption.value = ''
  selectedTopicId.value = ''
  selectedTopicContent.value = ''
  showAddMemory.value = false
}

const handleDeleteMemory = (memoryId: string) => {
  if (confirm('确定要删除这条记忆吗？')) {
    removeMemory(roomId.value, memoryId)
  }
}

const goBack = () => {
  router.push(`/room/${roomId.value}`)
}

const formatTime = (dateStr: string) => {
  const d = new Date(dateStr)
  return d.toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
}

const formatFullTime = (dateStr: string) => {
  const d = new Date(dateStr)
  return d.toLocaleDateString('zh-CN', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  })
}
</script>

<template>
  <div
    v-if="currentRoom"
    class="memory-wall min-h-screen bg-gradient-to-br from-amber-50 via-rose-50 to-purple-50"
  >
    <div class="max-w-4xl mx-auto px-4 py-6">
      <div class="flex items-center justify-between mb-6">
        <button
          class="flex items-center gap-2 text-gray-600 hover:text-gray-800 transition-colors"
          @click="goBack"
        >
          <span>←</span>
          <span>返回房间</span>
        </button>

        <h1 class="text-xl font-bold text-gray-800 flex items-center gap-2">
          <span>📷</span> 记忆墙
        </h1>

        <button
          class="px-4 py-2 bg-gradient-to-r from-amber-500 to-rose-500 text-white rounded-full text-sm font-medium hover:opacity-90 transition-opacity"
          @click="showAddMemory = true"
        >
          + 添加记忆
        </button>
      </div>

      <div class="bg-white rounded-3xl p-5 shadow-lg mb-6">
        <div class="flex items-center gap-3">
          <div class="text-4xl">🎒</div>
          <div>
            <h2 class="font-bold text-gray-800 text-lg">{{ currentRoom.name }}</h2>
            <p class="text-sm text-gray-500">
              共 {{ currentRoom.memories?.length || 0 }} 条记忆 ·
              {{ currentRoom.members.length }} 位成员
            </p>
          </div>
        </div>
      </div>

      <div v-if="sortedMemories.length === 0" class="text-center py-20 bg-white rounded-3xl shadow-md">
        <div class="text-7xl mb-4">🌟</div>
        <p class="text-gray-500 text-lg mb-2">记忆墙还是空白的</p>
        <p class="text-gray-400 text-sm mb-6">拍下现场照片，配上聊过的话题，留住今晚的气氛</p>
        <button
          class="px-6 py-3 bg-gradient-to-r from-amber-500 to-rose-500 text-white rounded-xl font-medium hover:opacity-90 transition-opacity"
          @click="showAddMemory = true"
        >
          📷 记录第一条记忆
        </button>
      </div>

      <div v-else class="relative">
        <div class="absolute left-6 top-0 bottom-0 w-0.5 bg-gradient-to-b from-amber-300 via-rose-300 to-purple-300"></div>

        <div
          v-for="group in timelineGroups"
          :key="group.date"
          class="mb-8"
        >
          <div class="relative flex items-center gap-3 mb-4 ml-1">
            <div class="w-10 h-10 rounded-full bg-gradient-to-br from-amber-400 to-rose-400 flex items-center justify-center text-white text-sm font-bold shadow-md z-10">
              {{ new Date(group.date).getDate() }}
            </div>
            <span class="text-sm font-medium text-gray-600">{{ group.label }}</span>
          </div>

          <div class="space-y-4 ml-5 pl-6">
            <div
              v-for="memory in group.memories"
              :key="memory.id"
              class="relative bg-white rounded-2xl shadow-md hover:shadow-lg transition-all overflow-hidden group"
            >
              <div class="absolute left-0 top-0 bottom-0 w-1 rounded-l-2xl"
                :style="{ backgroundColor: memory.topicId ? (TOPIC_COLORS[currentRoom.topics.find((t: Topic) => t.id === memory.topicId)?.type] || '#9D4EDD') : '#D1D5DB' }"
              ></div>

              <div class="p-4 pl-5">
                <div class="flex items-start gap-4">
                  <div class="flex-1 min-w-0">
                    <div class="flex items-center gap-2 mb-2">
                      <span class="text-xs text-gray-400">{{ formatTime(memory.createdAt) }}</span>
                      <span class="text-xs text-gray-400">·</span>
                      <span class="text-xs text-gray-500">{{ memory.author }}</span>
                    </div>

                    <div
                      v-if="memory.topicContent"
                      class="mb-3 px-3 py-2 rounded-xl text-sm"
                      :style="{
                        backgroundColor: (TOPIC_COLORS[currentRoom.topics.find((t: Topic) => t.id === memory.topicId)?.type] || '#9D4EDD') + '15',
                        borderLeft: `3px solid ${TOPIC_COLORS[currentRoom.topics.find((t: Topic) => t.id === memory.topicId)?.type] || '#9D4EDD'}`
                      }"
                    >
                      <span class="text-xs font-medium" :style="{ color: TOPIC_COLORS[currentRoom.topics.find((t: Topic) => t.id === memory.topicId)?.type] || '#9D4EDD' }">
                        {{ TOPIC_EMOJIS[currentRoom.topics.find((t: Topic) => t.id === memory.topicId)?.type] || '💬' }}
                        聊过的话题
                      </span>
                      <p class="mt-1 text-gray-700 font-medium">{{ memory.topicContent }}</p>
                    </div>

                    <p v-if="memory.caption" class="text-gray-600 text-sm mb-2">
                      {{ memory.caption }}
                    </p>
                  </div>

                  <div
                    v-if="memory.photo"
                    class="w-28 h-28 rounded-xl overflow-hidden flex-shrink-0 shadow-sm"
                  >
                    <img
                      :src="memory.photo"
                      :alt="memory.caption || '聚会照片'"
                      class="w-full h-full object-cover"
                    />
                  </div>
                </div>

                <div class="flex justify-end mt-1">
                  <button
                    class="opacity-0 group-hover:opacity-100 transition-opacity text-gray-400 hover:text-red-500 text-xs p-1"
                    @click="handleDeleteMemory(memory.id)"
                  >
                    删除
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div
      v-if="showAddMemory"
      class="fixed inset-0 bg-black/50 flex items-center justify-center z-50 p-4"
      @click.self="showAddMemory = false"
    >
      <div class="bg-white rounded-3xl p-6 w-full max-w-lg shadow-2xl max-h-[90vh] overflow-y-auto">
        <h2 class="text-xl font-bold text-gray-800 mb-4 text-center">
          📷 记录这一刻
        </h2>

        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-2">
            现场照片 *
          </label>
          <div
            v-if="!photoPreview"
            class="border-2 border-dashed border-gray-300 rounded-2xl p-8 text-center hover:border-amber-400 transition-colors cursor-pointer"
            @click="($refs.fileInput as HTMLInputElement)?.click()"
          >
            <div class="text-4xl mb-2">📸</div>
            <p class="text-gray-500 text-sm">点击上传照片</p>
            <p class="text-gray-400 text-xs mt-1">支持 JPG、PNG 格式</p>
          </div>
          <div v-else class="relative">
            <img
              :src="photoPreview"
              alt="预览"
              class="w-full max-h-64 object-cover rounded-2xl"
            />
            <button
              class="absolute top-2 right-2 w-8 h-8 bg-black/50 text-white rounded-full flex items-center justify-center hover:bg-black/70 transition-colors"
              @click="photoPreview = ''; photoData = ''"
            >
              ✕
            </button>
          </div>
          <input
            ref="fileInput"
            type="file"
            accept="image/*"
            class="hidden"
            @change="handlePhotoUpload"
          />
        </div>

        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-2">
            配对话题（可选）
          </label>
          <div v-if="flippedTopics.length > 0" class="max-h-40 overflow-y-auto space-y-2">
            <button
              v-for="topic in flippedTopics"
              :key="topic.id"
              class="w-full text-left p-3 rounded-xl border-2 transition-all"
              :class="{
                'border-amber-400 bg-amber-50': selectedTopicId === topic.id,
                'border-gray-200 hover:border-gray-300': selectedTopicId !== topic.id
              }"
              @click="selectTopic(topic)"
            >
              <div class="flex items-center gap-2">
                <span
                  class="px-1.5 py-0.5 rounded text-xs font-medium text-white"
                  :style="{ backgroundColor: topic.color }"
                >
                  {{ TOPIC_EMOJIS[topic.type] }}
                </span>
                <span class="text-sm text-gray-700 line-clamp-1">{{ topic.content }}</span>
              </div>
            </button>
          </div>
          <p v-else class="text-sm text-gray-400 py-3 text-center">
            还没有聊过的话题，先去翻牌吧 🎴
          </p>
        </div>

        <div class="mb-4">
          <label class="block text-sm font-medium text-gray-700 mb-2">
            心情备注
          </label>
          <textarea
            v-model="caption"
            placeholder="这一刻的气氛怎么样？"
            rows="2"
            class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-amber-500 focus:ring-2 focus:ring-amber-200 outline-none transition-all resize-none"
          ></textarea>
        </div>

        <div class="mb-6">
          <label class="block text-sm font-medium text-gray-700 mb-2">
            你的名字
          </label>
          <input
            v-model="authorName"
            type="text"
            placeholder="输入你的名字"
            class="w-full px-4 py-3 rounded-xl border border-gray-200 focus:border-amber-500 focus:ring-2 focus:ring-amber-200 outline-none transition-all"
          />
        </div>

        <div class="flex gap-3">
          <button
            class="flex-1 px-6 py-3 bg-gray-100 text-gray-600 rounded-xl font-medium hover:bg-gray-200 transition-colors"
            @click="showAddMemory = false"
          >
            取消
          </button>
          <button
            class="flex-1 px-6 py-3 bg-gradient-to-r from-amber-500 to-rose-500 text-white rounded-xl font-medium hover:opacity-90 transition-opacity disabled:opacity-50"
            :disabled="!photoData || !authorName.trim()"
            @click="handleAddMemory"
          >
            收藏记忆 ✨
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
