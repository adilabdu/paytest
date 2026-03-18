<!-- @fallthroughAttributes true -->
<!-- @strictTemplates true -->

<script setup lang="ts">
import type { HTMLAttributes } from 'vue'
import { cn } from '@/lib/utils'

const props = defineProps<{
  label?: string
  disabled?: boolean
  class?: HTMLAttributes['class']
}>()
</script>

<template>
  <optgroup
    :label="props.label"
    :disabled="props.disabled"
    :class="cn('bg-popover text-popover-foreground', props.class)"
  >
    <slot />
  </optgroup>
</template>
