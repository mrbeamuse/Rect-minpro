<script setup lang="ts">
const handler = (event: ErrorEvent) => {
  // 阻止向上抛出控制台报错
  event.preventDefault();
  // 处理错误
  console.log(event);
  // HandleEvents.handleError(event);
};

window.addEventListener('error', (event) => handler(event), true);

function throwError() {
  setTimeout(() => {
    throw new Error('error');
  }, 1000);
}

throwError();
</script>

<template>
  <div></div>
</template>
