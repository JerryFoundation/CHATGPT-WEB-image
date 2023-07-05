<script lang="ts" setup>
import { computed, onMounted, onUnmounted, onUpdated, ref } from 'vue'
import MarkdownIt from 'markdown-it'
import mdKatex from '@traptitech/markdown-it-katex'
import mila from 'markdown-it-link-attributes'
import hljs from 'highlight.js'
import { NButton } from 'naive-ui'
import { useBasicLayout } from '@/hooks/useBasicLayout'
import { t } from '@/locales'
import { copyToClip } from '@/utils/copy'
import eventMitt, { BACK_PLAN_CONFIG_READONLY } from '@/events'
import { ImgChange } from '@/components/common/Setting/model'

const props = defineProps<Props>()

const UList = ref([
  {
    value: 'U1',
    name: '放大1',
  },
  {
    value: 'U2',
    name: '放大2',
  },
  {
    value: 'U3',
    name: '放大3',
  },
  {
    value: 'U4',
    name: '放大4',
  },
])

const VList = ref([
  {
    value: 'V1',
    name: '变换1',
  },
  {
    value: 'V2',
    name: '变换2',
  },
  {
    value: 'V3',
    name: '变换3',
  },
  {
    value: 'V4',
    name: '变换4',
  },
])

interface Props {
  inversion?: boolean
  error?: boolean
  text?: string
  loading?: boolean
  asRawText?: boolean
  genImageBase64?: string
  roomId?: number
  taskId?: string
  imageResultStatus?: string
}

const { isMobile } = useBasicLayout()

const textRef = ref<HTMLElement>()

const mdi = new MarkdownIt({
  html: false,
  linkify: true,
  highlight(code, language) {
    const validLang = !!(language && hljs.getLanguage(language))
    if (validLang) {
      const lang = language ?? ''
      return highlightBlock(hljs.highlight(code, { language: lang }).value, lang)
    }
    return highlightBlock(hljs.highlightAuto(code).value, '')
  },
})

mdi.use(mila, { attrs: { target: '_blank', rel: 'noopener' } })
mdi.use(mdKatex, { blockClass: 'katexmath-block rounded-md p-[10px]', errorColor: ' #cc0000' })

const wrapClass = computed(() => {
  return [
    'text-wrap',
    'min-w-[20px]',
    'rounded-md',
    isMobile.value ? 'p-2' : 'px-3 py-2',
    props.inversion ? 'bg-[#d2f9d1]' : 'bg-[#f4f6f8]',
    props.inversion ? 'dark:bg-[#a1dc95]' : 'dark:bg-[#1e1e20]',
    props.inversion ? 'message-request' : 'message-reply',
    { 'text-red-500': props.error },
  ]
})

const text = computed(() => {
  const value = props.text ?? ''
  if (!props.asRawText)
    return mdi.render(value)
  return value
})

const genImageBase64 = computed(() => {
  const value = props.genImageBase64 ?? ''
  if (value && props.inversion)
    return value
  return ''
})

function highlightBlock(str: string, lang?: string) {
  return `<pre class="code-block-wrapper"><div class="code-block-header"><span class="code-block-header__lang">${lang}</span><span class="code-block-header__copy">${t('chat.copyCode')}</span></div><code class="hljs code-block-body ${lang}">${str}</code></pre>`
}

function addCopyEvents() {
  if (textRef.value) {
    const copyBtn = textRef.value.querySelectorAll('.code-block-header__copy')
    copyBtn.forEach((btn) => {
      btn.addEventListener('click', () => {
        const code = btn.parentElement?.nextElementSibling?.textContent
        if (code) {
          copyToClip(code).then(() => {
            btn.textContent = '复制成功'
            setTimeout(() => {
              btn.textContent = '复制代码'
            }, 1000)
          })
        }
      })
    })
  }
}
function removeCopyEvents() {
  if (textRef.value) {
    const copyBtn = textRef.value.querySelectorAll('.code-block-header__copy')
    copyBtn.forEach((btn) => {
      btn.removeEventListener('click', () => { })
    })
  }
}

function changeImg(action: string, actionName: string) {
  eventMitt.emit(
    BACK_PLAN_CONFIG_READONLY,
    new ImgChange(action, props.taskId as string, actionName, props.roomId as number),
  )
}

onMounted(() => {
  addCopyEvents()
})
onUpdated(() => {
  addCopyEvents()
})
onUnmounted(() => {
  removeCopyEvents()
})
</script>

<template>
  <div class="text-black" :class="wrapClass">
    <div>
      <div ref="textRef" class="leading-relaxed break-words">
        <div v-if="!inversion" class="flex items-end">
          <div v-if="!asRawText" class="w-full markdown-body" v-html="text" />
          <div v-else class="w-full whitespace-pre-wrap" v-text="text" />
          <span v-if="loading" class="dark:text-white w-[4px] h-[20px] block animate-blink" />
        </div>
        <div v-else class="whitespace-pre-wrap" v-text="text" />
        <img v-if="genImageBase64" :src="genImageBase64" style="width: 400px;">
      </div>
    </div>
    <div v-if="!inversion && imageResultStatus !== null && imageResultStatus === 'change'">
      <div
        style="margin-top: 20px;"
      >
        <NButton
          v-for="(item, index) in UList"
          :key="index"
          type="tertiary"
          size="medium"
          class="custom-button"
          ghost
          strong
          @click="changeImg(item.value, item.name)"
        >
          {{ item.name }}
        </NButton>
      </div>
      <div
        style="margin-top: 20px;"
      >
        <NButton
          v-for="(item, index) in VList"
          :key="index"
          type="tertiary"
          size="medium"
          class="custom-button"
          ghost
          strong
          @click="changeImg(item.value, item.name)"
        >
          {{ item.name }}
        </NButton>
      </div>
    </div>
  </div>
</template>

<style lang="less">
  /* 样式属性 */
  @media (max-width: 767px) {
  .custom-button {
    margin-right: 5px;
    /* 其他手机样式 */
    color:#ff69b4;
  }
}

/* 电脑设备样式 */
@media (min-width: 768px) {
  .custom-button {
    margin-right: 30px;
    color:rgb(73, 214, 167)
    /* 其他电脑样式 */
  }
}

@import url(./style.less);
</style>
