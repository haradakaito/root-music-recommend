<script setup="ts">
// ページタイトルの設定
useHead({
  titleTemplate: (title) => {
    return title
      ? `${title} | No MUSIC No Travel`
      : 'No Music No Travel'
  },
})
</script>

<template>
  <div>
    <NuxtLayout>
      <NuxtPage />
    </NuxtLayout>
  </div>
</template>

<style>
body {
  margin: 0;
}
</style>