<script setup lang='ts'>
import type { Ref } from 'vue'
import { computed, defineAsyncComponent, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRoute } from 'vue-router'
import { storeToRefs } from 'pinia'
import type { MessageReactive } from 'naive-ui'
import { NAutoComplete, NButton, NInput, NSelect, NSpace, NSpin, useDialog, useMessage,NModal,NCard,NLayout,NSwitch } from 'naive-ui'
import html2canvas from 'html2canvas'
import { Message } from './components'
import { useScroll } from './hooks/useScroll'
import { useChat } from './hooks/useChat'
import HeaderComponent from './components/Header/index.vue'
import { HoverButton, SvgIcon } from '@/components/common'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { useAuthStore, useChatStore, usePromptStore, useUserStore } from '@/store'
import { fetchChatAPIProcess, fetchChatResponseoHistory, fetchChatStopResponding, fetchUpdateUserChatModel,fetchUpdatePlugin,queryUserFunction } from '@/api'
import { t } from '@/locales'
import { debounce } from '@/utils/functions/debounce'
import IconPrompt from '@/icons/Prompt.vue'
import { UserConfig } from '@/components/common/Setting/model'
import type { CHATMODEL, IMAGEMODE } from '@/components/common/Setting/model'
import eventMitt, { BACK_PLAN_CONFIG_READONLY } from '@/events'

const Prompt = defineAsyncComponent(() => import('@/components/common/Setting/Prompt.vue'))

let controller = new AbortController()
let lastChatInfo: any = {}

const openLongReply = import.meta.env.VITE_GLOB_OPEN_LONG_REPLY === 'true'

const route = useRoute()
const dialog = useDialog()
const ms = useMessage()
const authStore = useAuthStore()
const userStore = useUserStore()
const chatStore = useChatStore()

const { isMobile } = useBasicLayout()
const { addChat, updateChat, updateChatSome, getChatByUuidAndIndex } = useChat()
const { scrollRef, scrollToBottom, scrollToBottomIfAtBottom, scrollTo } = useScroll()

const { uuid } = route.params as { uuid: string }

const currentChatHistory = computed(() => chatStore.getChatHistoryByCurrentActive)
const usingContext = computed(() => currentChatHistory?.value?.usingContext ?? true)
const dataSources = computed(() => chatStore.getChatByUuid(+uuid))
const conversationList = computed(() => dataSources.value.filter(item => (!item.inversion && !!item.conversationOptions)))

const prompt = ref<string>('')
const firstLoading = ref<boolean>(false)
const loading = ref<boolean>(false)
const inputRef = ref<Ref | null>(null)
const showPrompt = ref(false)
const imageUrl = ref<any>('')
const zipImageUrl = ref<any>('')
const imageType = ref<any>('')
let imageAction = ''
let changeTaskId = ''
let imgResultStatus = ''

let loadingms: MessageReactive
let allmsg: MessageReactive
let prevScrollTop: number

// 添加PromptStore
const promptStore = usePromptStore()

// 使用storeToRefs，保证store修改后，联想部分能够重新渲染
const { promptList: promptTemplate } = storeToRefs<any>(promptStore)

// 未知原因刷新页面，loading 状态不会重置，手动重置
dataSources.value.forEach((item, index) => {
  if (item.loading)
    updateChatSome(+uuid, index, { loading: false })
})

function handleSubmit() {
  onConversation()
}

function compress(base64String: string, w: number, quality: number) {
  const newImage = new Image()
  let imgWidth, imgHeight
  const promise = new Promise(resolve => (newImage.onload = resolve))
  newImage.src = base64String
  promise.then(() => {
    imgWidth = newImage.width
    imgHeight = newImage.height
    const canvas = document.createElement('canvas')
    // eslint-disable-next-line no-console
    console.log(canvas)
    const ctx: any = canvas.getContext('2d')
    if (Math.max(imgWidth, imgHeight) > w) {
      if (imgWidth > imgHeight) {
        canvas.width = w
        canvas.height = (w * imgHeight) / imgWidth
      }
      else {
        canvas.height = w
        canvas.width = (w * imgWidth) / imgHeight
      }
    }
    else {
      canvas.width = imgWidth
      canvas.height = imgHeight
    }
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    ctx.drawImage(newImage, 0, 0, canvas.width, canvas.height)
    // var base64 = canvas.toDataURL(getMimeType(base64String), quality);
    const base64 = canvas.toDataURL('image/jpeg', quality)
    // eslint-disable-next-line no-console
    console.log(base64)
    zipImageUrl.value = base64
  })
}

async function onConversation() {
  let message = prompt.value
  const image = zipImageUrl.value
  // eslint-disable-next-line no-console
  console.log(userStore.userInfo.config.imageMode)
  if (loading.value)
    return

  if (!message || message.trim() === '')
    return

  controller = new AbortController()

  const chatUuid = Date.now()
  addChat(
    +uuid,
    {
      uuid: chatUuid,
      dateTime: new Date().toLocaleString(),
      text: message,
      inversion: true,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: null },
      imageBase64: image,
      changeTaskId,
      imageResultStatus: imgResultStatus,
      imageAction,
    },
  )
  scrollToBottom()

  loading.value = true
  prompt.value = ''

  let options: Chat.ConversationRequest = {}
  const lastContext = conversationList.value[conversationList.value.length - 1]?.conversationOptions

  if (lastContext && usingContext.value)
    options = { ...lastContext }

  addChat(
    +uuid,
    {
      uuid: chatUuid,
      dateTime: new Date().toLocaleString(),
      text: '',
      loading: true,
      inversion: false,
      error: false,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
      imageBase64: '',
      changeTaskId,
      imageResultStatus: imgResultStatus,
      imageAction,
    },
  )
  scrollToBottom()

  try {
    let lastText = ''
    const fetchChatAPIOnce = async () => {
      await fetchChatAPIProcess<Chat.ConversationResponse>({
        taskId: changeTaskId,
        imgAction: imageAction,
        imageType: userStore.userInfo.config.imageMode || '',
        imageBase64: image,
        roomId: +uuid,
        uuid: chatUuid,
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            lastChatInfo = data
            imgResultStatus = data.imgResultStatus || null
            if (imgResultStatus === 'change')
              changeTaskId = data.taskId || null
            const imageActionReturn = data.imgOperation
            const usage = (data.detail && data.detail.usage)
              ? {
                  completion_tokens: data.detail.usage.completion_tokens || null,
                  prompt_tokens: data.detail.usage.prompt_tokens || null,
                  total_tokens: data.detail.usage.total_tokens || null,
                  estimated: data.detail.usage.estimated || null,
                }
              : undefined
            updateChat(
              +uuid,
              dataSources.value.length - 1,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (data.text ?? ''),
                inversion: false,
                error: false,
                loading: true,
                conversationOptions: { conversationId: data.conversationId, parentMessageId: data.id },
                requestOptions: { prompt: message, options: { ...options } },
                usage,
                imageBase64: image,
                changeTaskId,
                imageResultStatus: imgResultStatus,
                imageAction: imageActionReturn,
              },
            )
            imageAction = ''
            if (openLongReply && data.detail && data.detail.choices.length > 0 && data.detail.choices[0].finish_reason === 'length') {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }

            scrollToBottomIfAtBottom()
          }
          catch (error) {
            console.log('error')
          }
        },
      })
      updateChatSome(+uuid, dataSources.value.length - 1, { loading: false })
    }

    await fetchChatAPIOnce()
  }
  catch (error: any) {
    const errorMessage = error?.message ?? t('common.wrong')

    if (error.message === 'canceled') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          loading: false,
        },
      )
      scrollToBottomIfAtBottom()
      return
    }

    const currentChat = getChatByUuidAndIndex(+uuid, dataSources.value.length - 1)

    if (currentChat?.text && currentChat.text !== '') {
      updateChatSome(
        +uuid,
        dataSources.value.length - 1,
        {
          text: `${currentChat.text}\n[${errorMessage}]`,
          error: false,
          loading: false,
        },
      )
      return
    }

    updateChat(
      +uuid,
      dataSources.value.length - 1,
      {
        dateTime: new Date().toLocaleString(),
        text: errorMessage,
        inversion: false,
        error: true,
        loading: false,
        conversationOptions: null,
        requestOptions: { prompt: message, options: { ...options } },
        imageBase64: image,
        changeTaskId,
        imageResultStatus: imgResultStatus,
        imageAction,
      },
    )
    scrollToBottomIfAtBottom()
  }
  finally {
    loading.value = false
  }
}

async function onRegenerate(index: number) {
  if (loading.value)
    return

  controller = new AbortController()

  const { requestOptions } = dataSources.value[index]
  const imageBase64 = dataSources.value[index].imageBase64
  // eslint-disable-next-line no-console
  console.log(requestOptions)
  let responseCount = dataSources.value[index].responseCount || 1
  responseCount++

  let message = requestOptions?.prompt ?? ''

  let options: Chat.ConversationRequest = {}

  if (requestOptions.options)
    options = { ...requestOptions.options }

  loading.value = true
  const chatUuid = dataSources.value[index].uuid
  const changeTaskId = dataSources.value[index].changeTaskId
  const imageAction = dataSources.value[index].imageAction
  updateChat(
    +uuid,
    index,
    {
      dateTime: new Date().toLocaleString(),
      text: '',
      inversion: false,
      responseCount,
      error: false,
      loading: true,
      conversationOptions: null,
      requestOptions: { prompt: message, options: { ...options } },
      imageBase64,
      changeTaskId: '',
      imageResultStatus: '',
      imageAction: '',
    },
  )

  try {
    let lastText = ''
    const fetchChatAPIOnce = async () => {
      await fetchChatAPIProcess<Chat.ConversationResponse>({
        taskId: changeTaskId,
        imgAction: imageAction,
        imageType: userStore.userInfo.config.imageMode || '',
        imageBase64,
        roomId: +uuid,
        uuid: chatUuid || Date.now(),
        regenerate: true,
        prompt: message,
        options,
        signal: controller.signal,
        onDownloadProgress: ({ event }) => {
          const xhr = event.target
          const { responseText } = xhr
          // Always process the final line
          const lastIndex = responseText.lastIndexOf('\n', responseText.length - 2)
          let chunk = responseText
          if (lastIndex !== -1)
            chunk = responseText.substring(lastIndex)
          try {
            const data = JSON.parse(chunk)
            lastChatInfo = data
            const imgResultStatus = data.imgResultStatus || null
            let changeTaskId
            if (imgResultStatus === 'change')
              changeTaskId = data.taskId || null
            const imageAction = data.imgOperation || null
            const usage = (data.detail && data.detail.usage)
              ? {
                  completion_tokens: data.detail.usage.completion_tokens || null,
                  prompt_tokens: data.detail.usage.prompt_tokens || null,
                  total_tokens: data.detail.usage.total_tokens || null,
                  estimated: data.detail.usage.estimated || null,
                }
              : undefined
            updateChat(
              +uuid,
              index,
              {
                dateTime: new Date().toLocaleString(),
                text: lastText + (data.text ?? ''),
                inversion: false,
                responseCount,
                error: false,
                loading: true,
                conversationOptions: { conversationId: data.conversationId, parentMessageId: data.id },
                requestOptions: { prompt: message, options: { ...options } },
                usage,
                imageBase64,
                changeTaskId,
                imageResultStatus: imgResultStatus,
                imageAction,
              },
            )

            if (openLongReply && data.detail && data.detail.choices.length > 0 && data.detail.choices[0].finish_reason === 'length') {
              options.parentMessageId = data.id
              lastText = data.text
              message = ''
              return fetchChatAPIOnce()
            }
          }
          catch (error) {
            //
          }
        },
      })
      updateChatSome(+uuid, index, { loading: false })
    }
    await fetchChatAPIOnce()
  }
  catch (error: any) {
    if (error.message === 'canceled') {
      updateChatSome(
        +uuid,
        index,
        {
          loading: false,
        },
      )
      return
    }

    const errorMessage = error?.message ?? t('common.wrong')

    updateChat(
      +uuid,
      index,
      {
        dateTime: new Date().toLocaleString(),
        text: errorMessage,
        inversion: false,
        responseCount,
        error: true,
        loading: false,
        conversationOptions: null,
        requestOptions: { prompt: message, options: { ...options } },
        imageBase64: '',
        changeTaskId: '',
        imageResultStatus: '',
        imageAction: '',
      },
    )
  }
  finally {
    loading.value = false
  }
}

async function onResponseHistory(index: number, historyIndex: number) {
  const chat = (await fetchChatResponseoHistory(+uuid, dataSources.value[index].uuid || Date.now(), historyIndex)).data
  updateChat(
    +uuid,
    index,
    {
      dateTime: chat.dateTime,
      text: chat.text,
      inversion: false,
      responseCount: chat.responseCount,
      error: true,
      loading: false,
      conversationOptions: chat.conversationOptions,
      requestOptions: { prompt: chat.requestOptions.prompt, options: { ...chat.requestOptions.options } },
      usage: chat.usage,
      imageBase64: zipImageUrl,
      changeTaskId: chat.taskId,
      imageResultStatus: chat.imgResultStatus,
      imageAction: chat.imgOperation,
    },
  )
}

function handleExport() {
  if (loading.value)
    return

  const d = dialog.warning({
    title: t('chat.exportImage'),
    content: t('chat.exportImageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: async () => {
      try {
        d.loading = true
        const ele = document.getElementById('image-wrapper')
        const canvas = await html2canvas(ele as HTMLDivElement, {
          useCORS: true,
        })
        const imgUrl = canvas.toDataURL('image/png')
        const tempLink = document.createElement('a')
        tempLink.style.display = 'none'
        tempLink.href = imgUrl
        tempLink.setAttribute('download', 'chat-shot.png')
        if (typeof tempLink.download === 'undefined')
          tempLink.setAttribute('target', '_blank')

        document.body.appendChild(tempLink)
        tempLink.click()
        document.body.removeChild(tempLink)
        window.URL.revokeObjectURL(imgUrl)
        d.loading = false
        ms.success(t('chat.exportSuccess'))
        Promise.resolve()
      }
      catch (error: any) {
        ms.error(t('chat.exportFailed'))
      }
      finally {
        d.loading = false
      }
    },
  })
}

function handleDelete(index: number) {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.deleteMessage'),
    content: t('chat.deleteMessageConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.deleteChatByUuid(+uuid, index)
    },
  })
}

function handleClear() {
  if (loading.value)
    return

  dialog.warning({
    title: t('chat.clearChat'),
    content: t('chat.clearChatConfirm'),
    positiveText: t('common.yes'),
    negativeText: t('common.no'),
    onPositiveClick: () => {
      chatStore.clearChatByUuid(+uuid)
    },
  })
}

function handleEnter(event: KeyboardEvent) {
  if (!isMobile.value) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
  else {
    if (event.key === 'Enter' && event.ctrlKey) {
      event.preventDefault()
      handleSubmit()
    }
  }
}

async function handleStop() {
  if (loading.value) {
    controller.abort()
    loading.value = false
    await fetchChatStopResponding(lastChatInfo.text, lastChatInfo.id, lastChatInfo.conversationId)
  }
}

async function loadMoreMessage(event: any) {
  const chatIndex = chatStore.chat.findIndex(d => d.uuid === +uuid)
  if (chatIndex <= -1 || chatStore.chat[chatIndex].data.length <= 0)
    return

  const scrollPosition = event.target.scrollHeight - event.target.scrollTop

  const lastId = chatStore.chat[chatIndex].data[0].uuid
  await chatStore.syncChat({ uuid: +uuid } as Chat.History, lastId, () => {
    loadingms && loadingms.destroy()
    nextTick(() => scrollTo(event.target.scrollHeight - scrollPosition))
  }, () => {
    loadingms = ms.loading(
      '加载中...', {
        duration: 0,
      },
    )
  }, () => {
    allmsg && allmsg.destroy()
    allmsg = ms.warning('没有更多了', {
      duration: 1000,
    })
  })
}

const handleLoadMoreMessage = debounce(loadMoreMessage, 300)
const handleSyncChat
  = debounce(() => {
    // 直接刷 极小概率不请求
    chatStore.syncChat({ uuid: Number(uuid) } as Chat.History, undefined, () => {
      firstLoading.value = false
      const scrollRef = document.querySelector('#scrollRef')
      if (scrollRef)
        nextTick(() => scrollRef.scrollTop = scrollRef.scrollHeight)
      if (inputRef.value && !isMobile.value)
        inputRef.value?.focus()
    })
  }, 200)

async function handleScroll(event: any) {
  const scrollTop = event.target.scrollTop
  if (scrollTop < 50 && (scrollTop < prevScrollTop || prevScrollTop === undefined))
    handleLoadMoreMessage(event)
  prevScrollTop = scrollTop
}

async function handleToggleUsingContext() {
  if (!currentChatHistory.value)
    return

  currentChatHistory.value.usingContext = !currentChatHistory.value.usingContext
  chatStore.setUsingContext(currentChatHistory.value.usingContext, +uuid)
  if (currentChatHistory.value.usingContext)
    ms.success(t('chat.turnOnContext'))
  else
    ms.warning(t('chat.turnOffContext'))
}

// 可优化部分
// 搜索选项计算，这里使用value作为索引项，所以当出现重复value时渲染异常(多项同时出现选中效果)
// 理想状态下其实应该是key作为索引项,但官方的renderOption会出现问题，所以就需要value反renderLabel实现
const searchOptions = computed(() => {
  if (prompt.value.startsWith('/')) {
    return promptTemplate.value.filter((item: { key: string }) => item.key.toLowerCase().includes(prompt.value.substring(1).toLowerCase())).map((obj: { value: any }) => {
      return {
        label: obj.value,
        value: obj.value,
      }
    })
  }
  else {
    return []
  }
})

// value反渲染key
const renderOption = (option: { label: string }) => {
  for (const i of promptTemplate.value) {
    if (i.value === option.label)
      return [i.key]
  }
  return []
}

const placeholder = computed(() => {
  if (isMobile.value)
    return t('chat.placeholderMobile')
  return t('chat.placeholder')
})

const buttonDisabled = computed(() => {
  return loading.value || !prompt.value || prompt.value.trim() === ''
})

const footerClass = computed(() => {
  let classes = ['p-4']
  if (isMobile.value)
    classes = ['sticky', 'left-0', 'bottom-0', 'right-0', 'p-2', 'pr-3', 'overflow-hidden']
  return classes
})

async function handleSyncChatModel(chatModel: CHATMODEL) {
  if (userStore.userInfo.config == null)
    userStore.userInfo.config = new UserConfig()
  userStore.userInfo.config.chatModel = chatModel
  userStore.recordState()
  imageType.value = null
  zipImageUrl.value = ''
  await fetchUpdateUserChatModel(chatModel)
}

async function handleSyncImageModel(imageMode: IMAGEMODE) {
  if (userStore.userInfo.config == null)
    userStore.userInfo.config = new UserConfig()
  userStore.userInfo.config.imageMode = imageMode
  userStore.recordState()
}

onMounted(() => {
  firstLoading.value = true
  handleSyncChat()

  if (authStore.token) {
    const chatModels = authStore.session?.chatModels
    if (chatModels != null && chatModels.filter(d => d.value === userStore.userInfo.config.chatModel).length <= 0)
      ms.error('你选择的模型已不存在，请重新选择 | The selected model not exists, please choose again.', { duration: 7000 })
  }

  eventMitt.on(BACK_PLAN_CONFIG_READONLY, async (cantEdit) => {
    prompt.value = `任务ID:${cantEdit.taskId}->${cantEdit.actionName}`
    imageAction = cantEdit.imgAction
    changeTaskId = cantEdit.taskId
    if (cantEdit.roomId === +uuid)
      await onConversation()
  })
})

watch(() => chatStore.active, (newVal, oldVal) => {
  handleSyncChat()
})

onUnmounted(() => {
  if (loading.value)
    controller.abort()
})

const pictureOptions: { label: string; key: IMAGEMODE; value: IMAGEMODE }[] = [
  { label: '文字描述生图', key: 'wensengtu', value: 'wensengtu' },
  { label: '图片加描述生图', key: 'tusengtu', value: 'tusengtu' },
  { label: '图片生成描述(上传后随便输入文字提交即可)', key: 'tusengwen', value: 'tusengwen' },
]

function reupload() {
  const input: any = document.createElement('input')
  input.type = 'file'
  input.onchange = () => {
    const fileData: any = input.files[0]
    if (!/\.(jpg|jpeg|png|gif|bmp|webp|svg|tiff|ico|jfif|jpe|psd|raw|heif|bat|bpg|jp2|jpx|pbm|pgf|ppm|pnm|pam|pcx|pic|xbm|xpm｜heic)$/.test(fileData.name.toLowerCase()))
      return
    const reader = new FileReader()
    reader.readAsDataURL(fileData) // 异步读取文件内容，结果用data:url的字符串形式表示

    /* 当读取操作成功完成时调用 */
    reader.onload = async (e) => {
      // eslint-disable-next-line no-console
      console.log(e) // 查看对象属性里面有个result属性，属性值，是一大串的base64格式的东西，这个就是我们要的图片
      const base64Str = reader.result // 取得数据 这里的this指向FileReader（）对象的实例reader
      // eslint-disable-next-line no-console
      console.log(base64Str, 'base64Str')
      imageUrl.value = base64Str
      compress(imageUrl.value, 800, 0.9)
      // this.returnMsg.returnFile = ''
      // this.returnMsg.returnFile = base64Str
    }
  }
  const event = new MouseEvent('click')
  input.dispatchEvent(event)
}

const showplugin = ref(false)

function handleSwitchChange(item:any){

const openSwitches = PluginList.value.filter(item => item.switch); // 筛选出开关打开的项

if (openSwitches.length > 3) {
  // 执行其他操作...
  alert('开关打开个数已超过3个')
  item.switch = false
}else{
  fetchUpdatePlugin(item.value,item.switch)
}
}

const dynamicPluginList = computed(() => {
  if (showplugin.value) {
    queryUserFunction().then((data) => {
      const functions = data.data as string[]
      // 遍历 PluginList
      PluginList.value.forEach((item) => {
        // 判断 functions 中是否存在与当前 item.name 相同的名称
        const found = functions.some((func) => func === item.value)
        console.log(found)
        if(found){
          item.switch = true
        }else{
          item.switch = false
        }
      })
    })
    // 这里返回 PluginList.value（可能是未更新的值）
    return PluginList.value
  }

  // 返回一个默认值或空数组，根据你的需求进行更改
  return []
})

const PluginList = ref([
  {
    value: 'baiduSearch',
    name: '百度搜索插件',
    switch: false,
  },
  {
    value: 'weiboHotSearch',
    name: '微博热搜插件',
    switch: false,
  },
  {
    value: 'sendMail',
    name: '发送邮件插件',
    switch: false,
  },
  {
    value: 'baiduBaikeSearch',
    name: '百度百科插件',
    switch: false,
  },
  {
    value: 'queryWeather',
    name: '天气插件',
    switch: false,
  },
  {
    value: 'getNews',
    name: '新闻插件',
    switch: false,
  },
  {
    value: 'getCurrentTime',
    name: '获取当前时间插件',
    switch: false,
  },
  {
    value: 'createImage',
    name: '生成图片',
    switch: false,
  },
])
</script>

<template>
  <div class="flex flex-col w-full h-full">
    <HeaderComponent
      v-if="isMobile"
      :using-context="usingContext"
      :show-prompt="showPrompt"
      @export="handleExport" @toggle-using-context="handleToggleUsingContext"
      @toggle-show-prompt="showPrompt = true"
    />
    <main class="flex-1 overflow-hidden">
      <div id="scrollRef" ref="scrollRef" class="h-full overflow-hidden overflow-y-auto" @scroll="handleScroll">
        <div
          id="image-wrapper"
          class="w-full max-w-screen-xl m-auto dark:bg-[#101014]"
          :class="[isMobile ? 'p-2' : 'p-4']"
        >
          <NSpin :show="firstLoading">
            <template v-if="!dataSources.length">
              <div class="flex items-center justify-center mt-4 text-center text-neutral-300">
                <SvgIcon icon="ri:bubble-chart-fill" class="mr-2 text-3xl" />
                <span>友情提示：可用余额可在设置中查询，图片生成需要切换模型至mid-journey，然后请开始展示你的想象力^-^</span>
              </div>
            </template>
            <template v-else>
              <div>
                <Message
                  v-for="(item, index) of dataSources"
                  :key="index"
                  :date-time="item.dateTime"
                  :text="item.text"
                  :inversion="item.inversion"
                  :response-count="item.responseCount"
                  :usage="item && item.usage || undefined"
                  :task-id="item.changeTaskId"
                  :error="item.error"
                  :loading="item.loading"
                  :room-id="+uuid"
                  :gen-image-base64="item.imageBase64"
                  :image-result-status="item.imageResultStatus"
                  @regenerate="onRegenerate(index)"
                  @delete="handleDelete(index)"
                  @response-history="(ev) => onResponseHistory(index, ev)"
                />
                <div class="sticky bottom-0 left-0 flex justify-center">
                  <NButton v-if="loading" type="warning" @click="handleStop">
                    <template #icon>
                      <SvgIcon icon="ri:stop-circle-line" />
                    </template>
                    Stop Responding
                  </NButton>
                </div>
              </div>
            </template>
          </NSpin>
        </div>
      </div>
    </main>
    <footer :class="footerClass">
      <div class="w-full max-w-screen-xl m-auto">
        <NSpace vertical>
          <div class="flex items-center space-x-2">
            <HoverButton @click="handleClear">
              <span class="text-xl text-[#4f555e] dark:text-white">
                <SvgIcon icon="ri:delete-bin-line" />
              </span>
            </HoverButton>
            <HoverButton v-if="!isMobile" @click="handleExport">
              <span class="text-xl text-[#4f555e] dark:text-white">
                <SvgIcon icon="ri:download-2-line" />
              </span>
            </HoverButton>
            <HoverButton v-if="!isMobile" @click="showPrompt = true">
              <span class="text-xl text-[#4f555e] dark:text-white">
                <IconPrompt class="w-[20px] m-auto" />
              </span>
            </HoverButton>
            <HoverButton v-if="!isMobile" @click="handleToggleUsingContext">
              <span class="text-xl" :class="{ 'text-[#4b9e5f]': usingContext, 'text-[#a8071a]': !usingContext }">
                <SvgIcon icon="ri:chat-history-line" />
              </span>
            </HoverButton>
            <NSelect
              style="width: 250px"
              :value="userStore.userInfo.config.chatModel"
              :options="authStore.session?.chatModels"
              :disabled="!!authStore.session?.auth && !authStore.token"
              @update-value="(val) => handleSyncChatModel(val)"
            />
            <NSelect
              v-if="userStore?.userInfo?.config?.chatModel === 'mid-journey'"
              style="width: 250px"
              :value="userStore.userInfo.config.imageMode"
              :options="pictureOptions"
              :disabled="!!authStore.session?.auth && !authStore.token"
              @update-value="(val) => handleSyncImageModel(val)"
            />
            <div>
              <el-button
                v-if="userStore.userInfo.config.chatModel === 'mid-journey' && userStore.userInfo.config.imageMode !== 'wensengtu'"
                type="primary"
                size="mini"
                @click="reupload"
              >
                <i class="el-icon-upload el-icon--right"> 上传图片 </i>
              </el-button>
            </div>
            <NButton block @click="showplugin = true"  v-if="userStore.userInfo.config.chatModel === 'auto-gpt'" style="width: 200px">
              {{ $t('store.plugin') }}
          </NButton>
          </div>
          <div class="flex items-center justify-between space-x-2">
            <NAutoComplete v-model:value="prompt" :options="searchOptions" :render-label="renderOption">
              <template #default="{ handleInput, handleBlur, handleFocus }">
                <NInput
                  ref="inputRef"
                  v-model:value="prompt"
                  :disabled="!!authStore.session?.auth && !authStore.token"
                  type="textarea"
                  :placeholder="placeholder"
                  :autosize="{ minRows: isMobile ? 1 : 4, maxRows: isMobile ? 4 : 8 }"
                  @input="handleInput"
                  @focus="handleFocus"
                  @blur="handleBlur"
                  @keypress="handleEnter"
                />
              </template>
            </NAutoComplete>
            <NButton type="primary" :disabled="buttonDisabled" @click="handleSubmit">
              <template #icon>
                <span class="dark:text-black">
                  <SvgIcon icon="ri:send-plane-fill" />
                </span>
              </template>
            </NButton>
          </div>
        </NSpace>
      </div>
    </footer>
    <Prompt v-if="showPrompt" v-model:roomId="uuid" v-model:visible="showPrompt" />
  </div>
  <NModal v-model:show="showplugin">
    <NCard
      style="width: 450px"
      title="插件商店"
      :bordered="false"
      size="huge"
      role="dialog"
      aria-modal="true"
    >
      <div style="height: 300px;" position="absolute">
        <NLayout position="absolute" content-style="padding: 0px;">
          <NCard v-for="(item) in dynamicPluginList" :title="item.name">
            {{ item.name }}
            <NSwitch v-model:value="item.switch" @update:value="handleSwitchChange(item)"/>
          </NCard>
        </NLayout>
      </div>
    </NCard>
  </NModal>
</template>
