<template>
  <svg class="icon" aria-hidden="true">
    <use v-bind:xlink:href="iconName"></use>
  </svg>
</template>

<script setup lang="ts">
import { computed } from "vue";
const props = defineProps({
  iconname: {
    type: String,
    required: true
  }
});
const iconName = computed(() => {
  return `#icon-${props.iconname}`;
});
</script>

<style scoped>
.icon {
  width: 1em;
  height: 1em;
  vertical-align: -0.15em;
  fill: currentColor;
  overflow: hidden;
}
</style>
