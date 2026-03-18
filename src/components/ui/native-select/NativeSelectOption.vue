<!-- @fallthroughAttributes true -->
<!-- @strictTemplates true -->

<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'

const props = defineProps<{
  value?: string
  disabled?: boolean
  class?: HTMLAttributes['class']
}>()
</script>

<template>
  <option
    :value="props.value"
    :disabled="props.disabled"
    :class="cn('bg-popover text-popover-foreground', props.class)"
  >
    <slot />
  </option>
</template>
