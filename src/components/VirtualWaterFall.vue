<template>
    <div class="virtual-waterfall-container" ref="containerRef" @scroll="handleScroll">
        <div class="virtual-waterfall-list" :style="listStyle">
            <div class="virtual-waterfall-item" 
                v-for="{item,style} in renderList" 
                :style="style" 
                :key="item.id">
                <slot name="item" :item="item"></slot>
            </div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { debounce, rafThrottle } from '@/utils/tools';
import type { ICardItem, IColumnQueue, IItemRect, IRenderItem, IVirtualWaterFallProps } from '@/utils/type';
import { computed, onMounted, onUnmounted, reactive, ref, watch, type CSSProperties } from 'vue';

const props = defineProps<IVirtualWaterFallProps>()

const resizeObserver = new ResizeObserver(() => {
    handleResize()
})
defineSlots<{
    item(props: {item:ICardItem}) : any
}>()
const containerRef = ref<HTMLDivElement | null>(null)
//卡片数据源状态
const dataState = reactive({
        loading:false,
        isFinish:false,
        currentPage:1,
        list:[] as ICardItem[] 
    })
//与虚拟列表有关的状态    
const scrollState = reactive({
    viewHeight:0,
    viewWidth:0,
    start:0
})
const end = computed(() => scrollState.viewHeight + scrollState.start)

const queueState = reactive({
    queue:Array(props.column).fill(0).map<IColumnQueue>(() => ({list:[],height:0})),
    len:0//储存当前视图上展示所有卡片数量
})

//计算最小最大高度列和索引
const computedHeight = computed(() => {
    let minIndex = 0
    let minHeight = Infinity
    let maxHeight = -Infinity

    queueState.queue.forEach(({height},index) =>{
         if (height < minHeight) {
      minHeight = height;
      minIndex = index;
    }
    if (height > maxHeight) {
      maxHeight = height;
    }
    })
    return {
        minIndex,
        minHeight,
        maxHeight
    }
})
const listStyle = computed(() => ({height: `${computedHeight.value.maxHeight}px`} as CSSProperties))

//确认视图范围，计算renderlist
//现在状况queueState是一个二维数组，我们需要得到一个一维数组
const cardList = computed(() => queueState.queue.reduce<IRenderItem[]>((pre, { list }) => pre.concat(list), []))
//进行截取操作
const renderList = computed(() => cardList.value.filter((i) => i.h + i.y > scrollState.start && i.y < end.value))

watch(
    () => props.column,
    () => {
        handleResize()
    }
)
//将数据源dataState转化为queueState
const addInQueue = (size:number) => {
    for(let i = 0;i < size ;i++){
        const minIndex = computedHeight.value.minIndex
        const currentColumn = queueState.queue[minIndex]
        const before = currentColumn.list[currentColumn.list.length - 1] || null
        const dataItem = dataState.list[queueState.len]
        const item = generatorItem(dataItem,before,minIndex)
        currentColumn.list.push(item)
        currentColumn.height += item.h
        queueState.len++
    }
}
//计算卡片的信息
const generatorItem = (item:ICardItem,before:IRenderItem | null,index:number):IRenderItem => {
     const rect = itemSizeInfo.value.get(item.id)
     const width = rect!.width
     const height = rect!.height
     let y = 0
     if(before){
        y = before.y + before.h + props.gap
     }
     return {
        item,
        y,
        h:height,
        style:{
            width:`${width}px`,
            height:`${height}px`,
            transform:`translate3d(${index === 0 ? 0 : (width + props.gap) * index}px,${y}px,0)`
        }
     }
}

const itemSizeInfo = computed(() =>
  dataState.list.reduce<Map<ICardItem["id"], IItemRect>>((pre, current) => {
    const itemWidth = Math.floor((scrollState.viewWidth - (props.column - 1) * props.gap) / props.column);
    pre.set(current.id, {
      width: itemWidth,
      height: Math.floor((itemWidth * current.height) / current.width),
    });
    return pre;
  }, new Map())
);

//初始化操作
const initScrollState = () => {
    scrollState.viewWidth = containerRef.value!.clientWidth
    scrollState.viewHeight = containerRef.value!.clientHeight
    scrollState.start = containerRef.value!.scrollTop
}

const init = async () => {
    initScrollState()
    resizeObserver.observe(containerRef.value!)
    const len = await loadDataList()
    len && addInQueue(len)
}

onMounted(() => {
    init()
})
onUnmounted(() => {
    resizeObserver.unobserve(containerRef.value!)
})

const loadDataList = async () => {
    if (dataState.isFinish) return;
    dataState.loading = true;
    const list = await props.request(dataState.currentPage++, props.pageSize);
    if (!list.length) {
        dataState.isFinish = true;
        return;
    }
    dataState.list.push(...list);
    dataState.loading = false;
    return list.length;
}
const handleScroll = rafThrottle(() => {
    const {scrollTop , clientHeight} = containerRef.value!
    scrollState.start = scrollTop
    if(scrollTop + clientHeight > computedHeight.value.minHeight){
        !dataState.loading && 
        loadDataList().then((len) => {
            len && addInQueue(len)
        })
    }
})

//响应式实现
const reComputedQueue = () => {
    //清空所有queueState的所有卡片数据
    queueState.queue = new Array(props.column).fill(0).map<IColumnQueue>(() => ({list:[],height:0}))
    queueState.len = 0
    //根据当前数据源长度重新计算
    addInQueue(dataState.list.length)
}
const handleResize = debounce(() => {
    initScrollState()
    reComputedQueue()
})
</script>

<style scoped lang="scss">
.virtual-waterfall{
    &-container{
        width: 100%;
        height: 100%;
        overflow-y: scroll;
        overflow-x: hidden;
    }
    &-list{
        position: relative;
        width: 100%;
    }
    &-item{
        position: absolute;
        top: 0;
        left:0;
        box-sizing: border-box;
    }
}
</style>