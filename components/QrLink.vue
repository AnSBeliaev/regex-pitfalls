<script setup>
import { onMounted, ref, watch } from 'vue'
import QRCode from 'qrcode'

const props = defineProps({
  url: { type: String, required: true },
  size: { type: Number, default: 200 },
  label: { type: String, default: '' },
})

const src = ref('')

async function render() {
  src.value = await QRCode.toDataURL(props.url, {
    width: props.size,
    margin: 1,
    color: {
      dark: '#32353C',
      light: '#FFFFFF',
    },
  })
}

onMounted(render)
watch(() => [props.url, props.size], render)
</script>

<template>
  <a :href="url" class="flex flex-col items-center gap-3 no-underline! border-none!">
    <img
      :src="src"
      :width="size"
      :height="size"
      class="rounded bg-white p-2"
      alt=""
    >
    <span v-if="label" class="text-sm text-[#F7DF1E]">{{ label }}</span>
  </a>
</template>
