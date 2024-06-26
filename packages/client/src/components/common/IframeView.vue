<script lang="ts">
import { useDevToolsColorMode } from '@vue/devtools-ui'

const iframeCacheMap = new Map<string, HTMLIFrameElement>()
</script>

<script setup lang="ts">
const props = withDefaults(defineProps<{
  src: string
  inline?: boolean
}>(), {
  inline: false,
})

const { colorMode } = useDevToolsColorMode()
const anchor = ref<HTMLDivElement>()
const key = computed(() => props.src)
const iframeEl = ref<HTMLIFrameElement>()
const box = reactive(useElementBounding(anchor))
const iframeLoaded = ref(false)

onMounted(() => {
  if (iframeCacheMap.get(key.value)) {
    iframeEl.value = iframeCacheMap.get(key.value)!
    iframeEl.value.style.visibility = 'visible'
    iframeLoaded.value = true
  }
  else {
    iframeEl.value = document.createElement('iframe')
    iframeCacheMap.set(key.value, iframeEl.value)
    iframeEl.value.src = props.src
    // CORS
    try {
      iframeEl.value.style.opacity = '0.01'
      iframeEl.value.onload = () => {
        syncColorMode()
        iframeEl.value!.style.opacity = '1'
        iframeLoaded.value = true
      }
    }
    catch (e) {
      iframeEl.value.style.opacity = '1'
    }

    document.body.appendChild(iframeEl.value)
    nextTick(updateIframeBox)
  }

  // should force update the iframe visible on conflict(inline mode is unmounted after global mode is mounted)
  const timer = setTimeout(resolveConflictVisible, 100)
  setTimeout(syncColorMode, 100)

  onUnmounted(() => {
    clearTimeout(timer)
  })
})

watchEffect(updateIframeBox)
watchEffect(syncColorMode)

onUnmounted(() => {
  if (iframeEl.value)
    iframeEl.value.style.visibility = 'hidden'
})

function resolveConflictVisible() {
  if (!iframeEl.value)
    return
  iframeEl.value.style.visibility = 'visible'
}

function syncColorMode() {
  if (!iframeEl.value || !iframeEl.value.contentWindow)
    return
  try {
    const html = iframeEl.value.contentWindow.document.querySelector('html')
    html?.classList.toggle('dark', colorMode.value === 'dark')
    html?.classList.toggle('light', colorMode.value === 'dark')
  }
  catch (e) {
  }
}

function updateIframeBox() {
  if (!iframeEl.value)
    return
  Object.assign(iframeEl.value.style, {
    position: 'fixed',
    left: `${box.left}px`,
    top: `${box.top}px`,
    width: `${box.width}px`,
    height: `${props.inline ? box.height - box.top : box.height}px`,
    outline: 'none',
  })
}
</script>

<template>
  <div ref="anchor" h-full w-full>
    <div v-if="!iframeLoaded" absolute inset-0 flex items-center justify-center>
      <i class="mdi:loading animate-spin text-3xl" />
    </div>
  </div>
</template>
