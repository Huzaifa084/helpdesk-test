<template>
  <Button label="Refresh" theme="blue" :loading="loading">
    <template #icon>
      <RefreshIcon class="h-4 w-4" />
    </template>
  </Button>
</template>

<script setup>
defineProps({
  loading: {
    type: Boolean,
    default: false,
  },
});
</script>
