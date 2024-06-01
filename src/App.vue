<template>
  <div class="app">
    <div class="container" ref="containerRef">
      <VirtualWaterFall
        :request="getData"
        :gap="15"
        :page-size="15"
        :column="column">
        <template #item="{item}">
          <div
          class="test-item"
          :style="{
            background:item.bgColor
          }"></div>
        </template>
      </VirtualWaterFall>
    </div>
  </div>
</template>

<script setup lang="ts">
import VirtualWaterFall from './components/VirtualWaterFall.vue';
import type { ICardItem } from './utils/type';
import list from './config/index'
import { onMounted, onUnmounted, ref } from 'vue';

const containerRef = ref<HTMLDivElement | null>(null)
const column = ref(6)
const containerObserver = new ResizeObserver((entries) => {
  changeColumn(entries[0].target.clientWidth)
})

const changeColumn = (width: number) => {
  if (width > 960) {
    column.value = 5;
  } else if (width >= 690 && width < 960) {
    column.value = 4;
  } else if (width >= 500 && width < 690) {
    column.value = 3;
  } else {
    column.value = 2;
  }
}

onMounted(() => {
  containerRef.value && containerObserver.observe(containerRef.value)
})

onUnmounted(() => {
  containerRef.value && containerObserver.unobserve(containerRef.value)
})
const getData = (page:number,pageSize:number) => {
  return new Promise<ICardItem[]>((reslove) => {
    setTimeout(() => {
      reslove(list.slice((page - 1) * pageSize,(page - 1) * pageSize + pageSize))
    },100)
  })
}
</script>

<style scoped lang="scss">
.app{
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100vw;
  height: 100vh;
  .container{
    width: 90vw;
    height: 90vh;
    border: 1px solid red
  }
  .test-item{
    width: 100%;
    height: 100%;
    box-sizing: border-box;
    border-radius: 10px;
    animation: MoveAnimate 0.25s
  }
  @keyframes MoveAnimate {
  from {
    opacity: 0;
    transform: translateY(200px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
}
</style>