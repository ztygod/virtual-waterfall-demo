# virtual-waterfall-demo

"**有限的 DOM 来加载无限的数据**" <br />在实际情况中，我们可能需要面对大量数据的瀑布流，这时我们请求加载的时间就会变得非常长。这时我们就可以用到虚拟列表来优化加载，只加载可视区域内的瀑布流表。
<a name="qXh6S"></a>

# 定高

我们首先来看看定高情况的实现，注意这里的定高指的是每个图片从后端返回的高度与渲染到浏览器的高度固定，不包含有文本的情况。<br />**明确目标，我们需要将在视窗范围内的图片显示出来，而不在的则不显示。**那我们如何做到呢，首先我们需要定义一些类型接口，方便我们来描述卡片得到我们想要的属性。
<a name="WEcUP"></a>

## 定义类型接口

```typescript
import type { CSSProperties } from "vue";

export interface IVirtualWaterFallProps{
    gap:number,//卡片间隔
    column:number,//列数
    pageSize:number,//单次请求的数据量
    request : (page:number,pageSize:number) => Promise<ICardItem[]>
}

export interface ICardItem {
  id: number | string;
  width: number;
  height: number;
  [key: string]: any;
}

export interface IColumnQueue{
    list: IRenderItem[],
    height:number
}

export interface IRenderItem {
  item: ICardItem; // 数据源
  y: number; // 卡片距离列表顶部的距离
  h: number; // 卡片自身高度
  style: CSSProperties; // 用于渲染视图上的样式（宽、高、偏移量）
}
export interface IItemRect{
  width:number,
  height:number
}
```

IVirtualWaterFallProps用于接收父组件传递的数据。<br />ICardItem则是我们最原始的卡片单项。<br />IColumnQueue是一个二维数组，对应的是每一列的渲染项。<br />IRenderItem在布局中最为关键，他牵扯到每个卡片在页面中的宽高与偏移量，在模板中我们遍历渲染。<br />IItemRect是用来记录我们卡片的宽高，作为一个中间量。
<a name="MLiZg"></a>

## 处理数据源

我们的图片信息是从后端返回的，后端的信息大概长这样<br />![屏幕截图 2024-06-02 203700.png](https://cdn.nlark.com/yuque/0/2024/png/40660095/1717332026507-8d8efaa9-360a-44e8-8c20-52aa27c5bc6b.png#averageHue=%232c2f36&clientId=uec87f7d5-f5c4-4&from=ui&id=uce9dc855&originHeight=1598&originWidth=2555&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=514578&status=done&style=none&taskId=u0dbd10aa-d9b2-4442-9ead-71c113fd315&title=)<br />很显然，这样的数据是不方便我们前端处理的，因此我们对数据进行一些加工，加工成ICardItem类型：

```typescript
import data1 from './data1.json'
import data2 from './data2.json'
import type { ICardItem } from '@/utils/type';

const colorArr = ["#409eff", "#67c23a", "#e6a23c", "#f56c6c", "#909399"]

const list1: ICardItem[] = data1.data.items.map((i) => ({
  id: i.id,
  width: i.note_card.cover.width,
  height: i.note_card.cover.height,
  title: i.note_card.display_title,
  author: i.note_card.user.nickname,
}));
```

我们再将数据添加我们的前端组件的数据中，方便进一步处理

```typescript
const dataState = reactive({
        loading:false,
        isFinish:false,
        currentPage:1,
        list:[] as ICardItem[] 
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
```

<a name="Ce1bR"></a>

## 将dataState转换为renderList

现在我们已经有了初始数据源，现在我们要做的就是将初始数据源转换为要渲染在列表上的item，renderList是存储在queueState二维数组中的最基本项。

```typescript
const queueState = reactive({
    queue:Array(props.column).fill(0).map<IColumnQueue>(() => ({list:[],height:0})),
    len:0//储存当前视图上展示所有卡片数量
})
```

这时我们需要先将二维数组转化为一维数组，通过一维数组中的信息将新的ICardItem变为IRenderItem，添加到数组中。<br />那我们如何转化呢，这就是我们瀑布流与虚拟列表结合的一个关键了。<br />首先还是按照我们之前单独实现虚拟列表的方法，在最短列的下面添加新元素。<br />但是新元素要求是IRenderItem类型的，我们有的只是 ICardItem，这就涉及到我们对样式的计算了。

1. 宽高的计算与普通虚拟列表一致

```typescript
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
```

2. 利用宽高与同一列之前一项的y与h，我们可以得到当前项的y

![屏幕截图 2024-06-02 210052.png](https://cdn.nlark.com/yuque/0/2024/png/40660095/1717333241878-5ebec91d-3b63-4a07-9c5c-d22aad2c8a5c.png#averageHue=%23f3f8f7&clientId=uec87f7d5-f5c4-4&from=ui&id=u99166da9&originHeight=581&originWidth=1125&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=286684&status=done&style=none&taskId=uee9a3fef-4ee1-4fc9-a993-291248dc122&title=)

```typescript
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
```

3. 利用addInQueue，我们就可以得到每一个ICardItem对应的IRenderItem。
   <a name="ohdQ7"></a>

## 截取可视区域的数据项

我们得到了每个数据源对应的IRenderItem，接下来需要做的就是在他们在可视区域的时候显示他们。我们如何判断他们处于可视区域呢，我们可以根据IRenderItem中的y与h来确定：<br />![屏幕截图 2024-06-02 210620.png](https://cdn.nlark.com/yuque/0/2024/png/40660095/1717333578624-4f8f976b-c746-40a1-a50e-ab5cf9fc044d.png#averageHue=%23f3f8f7&clientId=uec87f7d5-f5c4-4&from=ui&height=352&id=ub0995a6c&originHeight=703&originWidth=1042&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=292782&status=done&style=none&taskId=uddd47054-efd2-4744-aee7-3a72cfc7268&title=&width=522)

![屏幕截图 2024-06-02 210627.png](https://cdn.nlark.com/yuque/0/2024/png/40660095/1717333589753-b3a7b2b3-fdd1-44f6-bfb5-d7adaa950dce.png#averageHue=%23f5f9fa&clientId=uec87f7d5-f5c4-4&from=ui&height=461&id=u4b5d9471&originHeight=944&originWidth=1069&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=331970&status=done&style=none&taskId=u291438c8-5183-4698-8ca7-30cb67584fc&title=&width=522)<br />由此我们可以知道，当y + h  > start，同时 y < end 时，显示在视窗内。由此得到我们的renderList

```typescript
//确认视图范围，计算renderlist
//现在状况queueState是一个二维数组，我们需要得到一个一维数组
const cardList = 
  computed(() => queueState.queue.reduce<IRenderItem[]>((pre, { list }) => pre.concat(list), []))
//进行截取操作
const renderList = 
  computed(() => cardList.value.filter((i) => i.h + i.y > scrollState.start && i.y < end.value))
```

在视图上进行渲染

```vue
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
```

<a name="XojIb"></a>

## 响应式实现

这里指的响应式指的是整个容器的列数随视窗口的大小来变化，在这里我们用到还是ResizeObserver来监视窗口尺寸的变化。<br />这个响应式需要父子组件共同来实现。

1. 首先当父组件的窗口大小发送变化时，我们执行绑定给esizeObserver的回调函数，改变列数。

```typescript
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
```

2. 在子组件中我们监听列数的变化与窗口大小的变化，在他变化时执行回调函数handleResize，handleResize主要清空所有queueState的所有卡片数据的信息，同时根据当前数据源长度重新计算queueState的信息

```typescript
const resizeObserver = new ResizeObserver(() => {
    handleResize()
})
watch(
    () => props.column,
    () => {
        handleResize()
    }
)

//初始化操作
const initScrollState = () => {
    scrollState.viewWidth = containerRef.value!.clientWidth
    scrollState.viewHeight = containerRef.value!.clientHeight
    scrollState.start = containerRef.value!.scrollTop
}
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


const init = async () => {
    initScrollState()
    resizeObserver.observe(containerRef.value!)
    const len = await loadDataList()
    len && addInQueue(len)
}

onMounted(() => {
    init()
})
```

