<script lang="ts" setup>
import { ref, onMounted } from 'vue'

interface ListItem {
  id: string
  x: number
  y: number
  rotation: number
  scale: number
}

const stageSize = ref({
  width: window.innerWidth,
  height: window.innerHeight,
})

const list = ref<ListItem[]>([])
const dragItemId = ref<string | null>(null)

const handleDragstart = (e: DragEvent) => {
  // save drag element:
  const target = e.target as HTMLElement
  if (!target) return

  dragItemId.value = target.id

  const item = list.value.find((i) => i.id === dragItemId.value)
  if (!item) return

  const index = list.value.indexOf(item!)
  list.value.splice(index, 1)
  list.value.push(item)
}

const handleDragend = () => {
  dragItemId.value = null
}

onMounted((): void => {
  for (let n = 0; n < 30; n++) {
    list.value.push({
      id: Math.round(Math.random() * 10000).toString(),
      x: Math.random() * stageSize.value.width,
      y: Math.random() * stageSize.value.height,
      rotation: Math.random() * 180,
      scale: Math.random(),
    })
  }
})
</script>

<template>
  <v-stage ref="stage" :config="stageSize" @dragstart="handleDragstart" @dragend="handleDragend">
    <v-layer ref="layer">
      <v-star
        v-for="item in list"
        :key="item.id"
        :config="{
          x: item.x,
          y: item.y,
          rotation: item.rotation,
          id: item.id,
          numPoints: 5,
          innerRadius: 30,
          outerRadius: 50,
          fill: '#89b717',
          opacity: 0.8,
          draggable: true,
          scaleX: dragItemId === item.id ? item.scale * 1.2 : item.scale,
          scaleY: dragItemId === item.id ? item.scale * 1.2 : item.scale,
          shadowColor: 'black',
          shadowBlur: 10,
          shadowOffsetX: dragItemId === item.id ? 15 : 5,
          shadowOffsetY: dragItemId === item.id ? 15 : 5,
          shadowOpacity: 0.6,
        }"
      />
    </v-layer>
  </v-stage>
</template>

<style scoped>
v-stage {
  display: block;
  background: #f5f5f5;
}
</style>
