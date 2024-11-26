<script setup name="ScrollPanel">
import { ref, reactive, onMounted } from "vue"
const slot = useSlots()

const MIN_WIDTH = 200
const layout = ref([])

const mainRef = ref(null)

// 面板尺寸配置参数
const resizeConfig = reactive({
  last: {
    pos: 0, // 上一次鼠标位置
    index: 0 // 上一个调整宽度的resizer的序号
  }
})

onMounted(() => {
  // 初始化layout参数
  Object.keys(slot).forEach(() => {
    layout.value.push({
      left: 0,
      width: 0
    })
  })
  nextTick(() => {
    if (mainRef.value) {
      // 监听外层容器domresize，更新layout中每个panel的width和left
      const resizeObserver = new ResizeObserver((entries) => {
        for (const entry of entries) {
          const width = entry.contentRect.width
          initPanelSize(width)
        }
      })
      resizeObserver.observe(mainRef.value)
    }
  })
})

const initPanelSize = (parentWidth) => {
  const width = parentWidth / layout.value.length
  layout.value.forEach((item, index) => {
    item.left = index * width
    item.width = width
  })
}

const isMoving = ref(false)
const handleMousedown = (e) => {
  isMoving.value = true
  const index = e.target.getAttribute("data-index")
  if (index !== null) {
    // 禁止文字选择
    document.onselectstart = () => false
    // 禁止元素拖拽
    document.ondragstart = () => false
    resizeConfig.last.pos = e.clientX
    resizeConfig.last.index = parseInt(index)
    document.addEventListener("mousemove", handleMousemove)
    document.addEventListener("mouseup", handleMouseup)
  }
}

const handleMousemove = (e) => {
  // 当前resizer的序号
  const index = resizeConfig.last.index
  let offsetX = e.clientX - resizeConfig.last.pos
  // 找到左边可以调整宽度的panel
  let leftIndex = index
  while (leftIndex >= 0 && layout.value[leftIndex].width <= MIN_WIDTH) {
    leftIndex--
  }
  let rightIndex = index + 1
  while (rightIndex < layout.value.length && layout.value[rightIndex].width <= MIN_WIDTH) {
    rightIndex++
  }
  // 当左侧panel全部已经最小化则暂停
  if (leftIndex < 0 && offsetX < 0) {
    return
  }
  // 当右侧panel全部已经最小化则暂停
  if (rightIndex >= layout.value.length && offsetX > 0) {
    return
  }
  if (offsetX < 0) {
    // resizer左侧panel宽度
    const newLeftWidth = layout.value[leftIndex].width + offsetX
    if (newLeftWidth <= MIN_WIDTH) {
      offsetX = MIN_WIDTH - layout.value[leftIndex].width
      layout.value[leftIndex].width = MIN_WIDTH
    } else {
      layout.value[leftIndex].width = newLeftWidth
    }
    // 修改被折叠的中间panel（最小宽度）的位置，要放在修改offset的语句后面
    for (let i = index; i > leftIndex; i--) {
      layout.value[i].left += offsetX
    }
    layout.value[index + 1].width -= offsetX
    layout.value[index + 1].left += offsetX
    resizeConfig.last.pos = e.clientX
  } else if (offsetX > 0) {
    // resizer右侧panel宽度
    const newRightWidth = layout.value[rightIndex].width - offsetX
    if (newRightWidth <= MIN_WIDTH) {
      // 要和上面反过来，因为这边的offset要是>0的
      offsetX = layout.value[rightIndex].width - MIN_WIDTH
      layout.value[rightIndex].width = MIN_WIDTH
    } else {
      layout.value[rightIndex].width = newRightWidth
    }
    // 修改被折叠的中间panel（最小宽度）的位置
    for (let i = index + 1; i <= rightIndex; i++) {
      layout.value[i].left += offsetX
    }
    layout.value[index].width += offsetX
    resizeConfig.last.pos = e.clientX
  }
}

const handleMouseup = () => {
  isMoving.value = false
  // 允许文字选择
  document.onselectstart = null
  // 允许元素拖拽
  document.ondragstart = null
  document.removeEventListener("mousemove", handleMousemove)
  document.removeEventListener("mousemove", handleMouseup)
}
</script>

<template>
  <div class="main" ref="mainRef" @mousedown="handleMousedown">
    <template :key="index" v-for="(item, index) in layout">
      <div
        class="main-item"
        :style="{
          width: `${item.width}px`,
          left: `${item.left}px`
        }"
      >
        <div class="opacity w-[100%] h-[100%] absolute top-0 left-0" v-if="isMoving" />
        <slot :name="index" />
      </div>
      <div v-if="index !== 0" class="resizer" :style="{ left: `${item.left}px` }" :data-index="index - 1" />
    </template>
  </div>
</template>

<style scoped>
.main {
  height: 100%;
  width: 100%;
  position: relative;
}

.main-item {
  background-color: #131a24;
  height: 100%;
  position: absolute;
  /* 很重要 */
  overflow: hidden;
  color: #fff;
}

.resizer {
  position: absolute;
  background-color: #d3d3d3;
  height: 100%;
  width: 1px;
  z-index: 1;
  cursor: ew-resize;
}

.resizer::after {
  content: "";
  position: absolute;
  left: 50%;
  top: 0;
  height: 100%;
  width: 100%;
  transform: translateX(-50%);
  min-width: 12px !important;
}
</style>
