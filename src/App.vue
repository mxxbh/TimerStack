<script setup lang="ts">
import Clock from "./components/clock.vue";
</script>

<template>
  <Clock :duration="7200" :reminder-time="900" title="2小时倒计时" />
</template>

<style scoped>
/* 全局样式已在组件中定义 */
</style>
