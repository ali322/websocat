<template>
  <a-layout style="min-height:100vh;">
    <a-layout-header class="flex align-middle">
      <h1 class="text-white">Websocat</h1>
    </a-layout-header>
    <a-layout-content>
      <div class="px-20 py-12">
        <a-input v-model:value="address" placeholder="websocket url" style="width: 820px;" addon-before="Address">
          <template #addonAfter>
            <SendOutlined @click="connect" />
          </template>
        </a-input>
        <div class="py-4 text-gray-500">server status <span class="pl-12"
            :class="{ 'text-green-500': status === 'open', 'text-red-500': status === 'closed' }">{{ status }}</span>
        </div>
        <div class="pb-8">
          <a-textarea v-model:value="message" placeholder="message content" :rows="6"></a-textarea>
        </div>
        <div class="flex items-center">
          <a-button type="primary" @click="send">发送</a-button>
          <a-button class="ml-4" @click="format">格式化</a-button>
          <a-button type="danger" class="ml-4" @click="purge">清空消息</a-button>
        </div>
        <div class="py-4">
          <div v-for="(item, i) in records" :key="i" class="flex bg-white shadow-sm rounded px-4 py-2 mb-2">
            <div class="w-5">
              <ArrowUpOutlined v-if="item.createdBy === 'me'" />
              <ArrowDownOutlined v-else />
            </div>
            <div class="flex-1">{{ item.data }}</div>
          </div>
        </div>
      </div>
    </a-layout-content>
  </a-layout>
</template>
<script lang="ts" setup>
import { SendOutlined, ArrowUpOutlined, ArrowDownOutlined } from '@ant-design/icons-vue'
import { ref, Ref, onUnmounted, onMounted } from 'vue'

const address = ref('')
const message = ref('')
const records: Ref<Array<Record>> = ref([])
const status = ref('')
const urlHistory: Ref<Array<string>> = ref([])

let ws: WebSocket

interface Record {
  createdBy: string,
  data: string
}

const connect = () => {
  urlHistory.value.push(address.value)
  window.localStorage.setItem('websocat-url-history', JSON.stringify(urlHistory.value))
  ws = new WebSocket(address.value)
  ws.onmessage = (evt: MessageEvent) => {
    records.value.push({
      createdBy: 'server', data: evt.data
    })
  }
  ws.onopen = () => {
    status.value = 'open'
  }
  ws.onclose = () => {
    status.value = 'closed'
  }
}

const state = () => {
  if (!ws) {
    return 'empty'
  } else {
    switch (ws.readyState) {
      case 0:
        return 'connecting'
      case 1:
        return 'open'
      case 2:
        return 'closing'
      case 3:
        return 'closed'
    }
  }
}

const format = () => {
  const data = JSON.parse(message.value)
  message.value = JSON.stringify(data, null, 2)
}

const purge = () => {
  records.value = []
}

const send = () => {
  records.value.push({
    createdBy: 'me', data: message.value
  })
  ws.send(message.value)
}

onMounted(() => {
  let ret = window.localStorage.getItem('websocat-url-history')
  if (ret !== null) {
    urlHistory.value = JSON.parse(ret)
  }
})
</script>