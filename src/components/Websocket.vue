<template>
  <a-layout style="min-height:100vh;">
    <a-layout-header class="flex align-middle">
      <h1 class="text-white">Websocat</h1>
    </a-layout-header>
    <a-layout-content>
      <div class="px-20 py-12">
        <a-auto-complete v-model:value="address" placeholder="websocket url" style="width: 820px;"
          :options="addressSuggestions" @search="handleAddressSearch" @select="handleAddressSelect">
        </a-auto-complete>
        <a-button v-if="status == 'open'" type="danger" @click="disconnect">
          <template #icon>
            <CloseOutlined />
          </template>
        </a-button>
        <a-button v-else type="primary" @click="connect">
          <template #icon>
            <SendOutlined />
          </template>
        </a-button>
        <div class="py-4 text-gray-500">server status <span class="pl-12"
            :class="{ 'text-green-500': status === 'open', 'text-red-500': status === 'closed' }">{{ status }}</span>
        </div>
        <div class="pb-8">
          <a-auto-complete v-model:value="message" placeholder="message content" :options="messageSuggestions" @search="handleMessageSearch" @select="handleMessageSelect">
            <a-textarea :rows="6" style="width:800px"></a-textarea>
          </a-auto-complete>
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
import { SendOutlined, CloseOutlined, ArrowUpOutlined, ArrowDownOutlined } from '@ant-design/icons-vue'
import { ref, Ref, onUnmounted, onMounted } from 'vue'

const address = ref('')
const addressSuggestions: Ref<Array<any>> = ref([])
const message = ref('')
const messageSuggestions: Ref<Array<any>> = ref([])
const records: Ref<Array<Record>> = ref([])
const status = ref('')
const urlHistory: Ref<Array<string>> = ref([])
const messageHistory: Ref<Array<string>> = ref([])

let ws: WebSocket

interface Record {
  createdBy: string,
  data: string
}

const disconnect = () => {
  if (status.value === 'open') {
    ws.close()
  }
}

const connect = () => {
  if (!address.value.startsWith("ws")) {
    return
  }
  if (!urlHistory.value.includes(address.value)) {
    urlHistory.value.push(address.value)
    window.localStorage.setItem('websocat-url-history', JSON.stringify(urlHistory.value))
  }
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
  window.localStorage.setItem('websocat-url-history', JSON.stringify([]))
  window.localStorage.setItem('websocat-message-history', JSON.stringify([]))
}

const send = () => {
  if (message.value === '') {
    return
  }
  if (!ws) {
    return
  }
  if (!messageHistory.value.includes(message.value)) {
    messageHistory.value.push(message.value)
    window.localStorage.setItem('websocat-message-history', JSON.stringify(messageHistory.value))
  }
  records.value.push({
    createdBy: 'me', data: message.value
  })
  ws.send(message.value)
}

const handleAddressSearch = (key: string) => {
  if (!key) {
    addressSuggestions.value = []
    return
  }
  addressSuggestions.value = urlHistory.value.filter((v: string) => {
    return v.includes(key)
  }).map((v: string) => ({ value: v, label: v }))
}

const handleAddressSelect = (selected: string) => {
  console.log('selected', selected)
  address.value = selected
}

const handleMessageSearch = (key: string) => {
  if (!key) {
    messageSuggestions.value = []
    return
  }
  messageSuggestions.value = messageHistory.value.filter((v: string) => {
    return v.includes(key)
  }).map((v: string) => ({ value: v, label: v }))
}

const handleMessageSelect = (selected: string) => {
  console.log('selected', selected)
  message.value = selected
}

onMounted(() => {
  let urls = window.localStorage.getItem('websocat-url-history')
  if (urls !== null) {
    urlHistory.value = JSON.parse(urls)
  }
  let messages = window.localStorage.getItem('websocat-message-history')
  if (messages !== null) {
    messageHistory.value = JSON.parse(messages)
  }
})
</script>