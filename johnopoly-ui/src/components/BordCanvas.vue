<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import Konva from 'konva'

function darken(color: string, amount: number): string {
  if (!color || !color.startsWith('#')) return color

  // Remove '#' and expand shorthand (e.g. #f00 → #ff0000)
  let hex = color.replace('#', '')
  if (hex.length === 3) {
    hex = hex
      .split('')
      .map((c) => c + c)
      .join('')
  }

  const num = parseInt(hex, 16)
  const r = (num >> 16) & 0xff
  const g = (num >> 8) & 0xff
  const b = num & 0xff

  const factor = 1 - amount
  const newR = Math.round(r * factor)
  const newG = Math.round(g * factor)
  const newB = Math.round(b * factor)

  // Convert back to hex
  const toHex = (n: number) => n.toString(16).padStart(2, '0')
  const darkHex = `#${toHex(newR)}${toHex(newG)}${toHex(newB)}`

  console.log(darkHex)
  return darkHex
}

interface Token {
  id: string
  x: number
  y: number
  color: string
  type: 'token'
}

interface House {
  id: string
  x: number
  y: number
  type: 'house'
}

interface Hotel {
  id: string
  x: number
  y: number
  type: 'hotel'
}

const stageSize = ref({
  width: window.innerWidth,
  height: window.innerHeight - 60,
})

const boardSize = ref({
  size: 0,
  offsetX: 0,
  offsetY: 0,
})

const scale = ref(1)
const stagePos = ref({ x: 0, y: 0 })
const isPanning = ref(false)
const lastPointerPos = ref({ x: 0, y: 0 })
const tokens = ref<Token[]>([])
const houses = ref<House[]>([])
const hotels = ref<Hotel[]>([])
const dragItemId = ref<string | null>(null)
const showClearModal = ref(false)
const selectedColor = ref('#ff0000')
const boardImage = ref<HTMLImageElement | null>(null)

const tokenColors = ['#ff0000', '#0000ff', '#00ff00', '#ffff00', '#ff00ff', '#00ffff']

// Hardcoded board image path - replace with your actual image path
const BOARD_IMAGE_PATH = '/monopoly-board.png'

const handleResize = () => {
  stageSize.value = {
    width: window.innerWidth,
    height: window.innerHeight - 60,
  }
  calculateBoardSize()
}

const calculateBoardSize = () => {
  const availableWidth = stageSize.value.width
  const availableHeight = stageSize.value.height

  // Make the board a square that fits within the available space
  const size = Math.min(availableWidth, availableHeight) - 40 // 40px padding

  boardSize.value = {
    size,
    offsetX: (availableWidth - size) / 2,
    offsetY: (availableHeight - size) / 2,
  }
}

const loadBoardImage = () => {
  const img = new Image()
  img.onload = () => {
    boardImage.value = img
    calculateBoardSize()
  }
  img.src = BOARD_IMAGE_PATH
}

const addToken = () => {
  tokens.value.push({
    id: `token-${Date.now()}`,
    x: boardSize.value.offsetX + boardSize.value.size / 2,
    y: boardSize.value.offsetY + boardSize.value.size / 2,
    color: selectedColor.value,
    type: 'token',
  })
}

const addHouse = () => {
  houses.value.push({
    id: `house-${Date.now()}`,
    x: boardSize.value.offsetX + boardSize.value.size / 2,
    y: boardSize.value.offsetY + boardSize.value.size / 2,
    type: 'house',
  })
}

const addHotel = () => {
  hotels.value.push({
    id: `hotel-${Date.now()}`,
    x: boardSize.value.offsetX + boardSize.value.size / 2,
    y: boardSize.value.offsetY + boardSize.value.size / 2,
    type: 'hotel',
  })
}

const handleDragStart = (e: Konva.KonvaEventObject<DragEvent>) => {
  const node = e.target
  if (!node) return
  dragItemId.value = node.id()
}

const handleDragEnd = (e: Konva.KonvaEventObject<DragEvent>) => {
  dragItemId.value = null
  const node = e.target
  if (!node) return

  const id = node.id()
  const newX = node.x()
  const newY = node.y()

  // Update token position
  const tokenIndex = tokens.value.findIndex((t) => t.id === id)
  if (tokenIndex !== -1) {
    const currentToken = tokens.value[tokenIndex]
    if (currentToken === undefined) {
      return
    }
    currentToken.x = newX
    currentToken.y = newY
    return
  }

  // Update house position
  const houseIndex = houses.value.findIndex((h) => h.id === id)
  if (houseIndex !== -1) {
    const currentHouse = houses.value[houseIndex]
    if (currentHouse === undefined) {
      return
    }
    currentHouse.x = newX
    currentHouse.y = newY
    return
  }

  // Update hotel position
  const hotelIndex = hotels.value.findIndex((h) => h.id === id)
  if (hotelIndex !== -1) {
    const currentHotel = hotels.value[hotelIndex]
    if (currentHotel === undefined) {
      return
    }
    currentHotel.x = newX
    currentHotel.y = newY
  }
}

const handleWheel = (e: Konva.KonvaEventObject<WheelEvent>) => {
  e.evt.preventDefault()

  const stage = e.target.getStage()
  if (!stage) return

  const oldScale = scale.value
  const pointer = stage.getPointerPosition()
  if (!pointer) return

  const mousePointTo = {
    x: (pointer.x - stagePos.value.x) / oldScale,
    y: (pointer.y - stagePos.value.y) / oldScale,
  }

  // Zoom in/out
  const scaleBy = 1.1
  const newScale = e.evt.deltaY < 0 ? oldScale * scaleBy : oldScale / scaleBy

  // Limit zoom range
  scale.value = Math.max(0.5, Math.min(3, newScale))

  const newPos = {
    x: pointer.x - mousePointTo.x * scale.value,
    y: pointer.y - mousePointTo.y * scale.value,
  }

  stagePos.value = newPos
}

const handleMouseDown = (e: Konva.KonvaEventObject<MouseEvent>) => {
  // Only start panning if clicking on the stage or board image (not on draggable items)
  const target = e.target
  const clickedOnEmpty = target === target.getStage()
  const clickedOnBoard = target.getClassName() === 'Image' && !target.draggable()

  if (clickedOnEmpty || clickedOnBoard) {
    isPanning.value = true
    const stage = target.getStage()
    const pointer = stage?.getPointerPosition()
    if (pointer) {
      lastPointerPos.value = pointer
    }
  }
}

const clearBoard = () => {
  tokens.value = []
  houses.value = []
  hotels.value = []
  showClearModal.value = false
}

const handleMouseMove = (e: Konva.KonvaEventObject<MouseEvent>) => {
  if (!isPanning.value) return

  const stage = e.target.getStage()
  const pointer = stage?.getPointerPosition()
  if (!pointer) return

  const dx = pointer.x - lastPointerPos.value.x
  const dy = pointer.y - lastPointerPos.value.y

  stagePos.value = {
    x: stagePos.value.x + dx,
    y: stagePos.value.y + dy,
  }

  lastPointerPos.value = pointer
}

const handleMouseUp = () => {
  isPanning.value = false
}

const resetView = () => {
  scale.value = 1
  stagePos.value = { x: 0, y: 0 }
}

onMounted(() => {
  window.addEventListener('resize', handleResize)
  loadBoardImage()
})
</script>

<template>
  <div class="monopoly-container">
    <!-- Toolbar -->
    <div class="toolbar">
      <div class="color-selector">
        <span class="label">Speler kleur:</span>
        <div class="color-buttons">
          <button
            v-for="color in tokenColors"
            :key="color"
            @click="selectedColor = color"
            :class="['color-btn', { active: selectedColor === color }]"
            :style="{ backgroundColor: color }"
          />
        </div>
      </div>

      <button @click="addToken" class="btn btn-success">
        <span class="icon">👤</span>
        Speler toevoegen
      </button>

      <button @click="addHouse" class="btn btn-warning">
        <span class="icon">🏠</span>
        Huisje bouwen
      </button>

      <button @click="addHotel" class="btn btn-danger">
        <span class="icon">🏨</span>
        Hotelletje bouwen
      </button>

      <div class="spacer"></div>

      <button @click="resetView" class="btn btn-info">
        <span class="icon">🔄</span>
        Reset View
      </button>

      <button @click="showClearModal = true" class="btn btn-clear">
        <span class="icon">🗑️</span>
        Stoppuh
      </button>
    </div>

    <!-- Canvas -->
    <v-stage
      :config="{
        ...stageSize,
        scaleX: scale,
        scaleY: scale,
        x: stagePos.x,
        y: stagePos.y,
      }"
      @dragstart="handleDragStart"
      @dragend="handleDragEnd"
      @wheel="handleWheel"
      @mousedown="handleMouseDown"
      @mousemove="handleMouseMove"
      @mouseup="handleMouseUp"
      @mouseleave="handleMouseUp"
      class="stage"
    >
      >
      <v-layer>
        <!-- Board Background -->
        <v-image
          v-if="boardImage"
          :config="{
            image: boardImage,
            x: boardSize.offsetX,
            y: boardSize.offsetY,
            width: boardSize.size,
            height: boardSize.size,
          }"
        />

        <!-- Tokens -->
        <v-circle
          v-for="token in tokens"
          :key="token.id"
          :config="{
            id: token.id,
            x: token.x,
            y: token.y,
            radius: 15,
            fillRadialGradientStartRadius: 0,
            fillRadialGradientEndRadius: 15,
            fillRadialGradientStartPointX: -5,
            fillRadialGradientStartPointY: -5,
            fillRadialGradientColorStops: [
              0,
              '#ffffff',
              0.2,
              token.color,
              0.9,
              token.color,
              1,
              darken(token.color, 0.4),
            ],
            draggable: true,
            shadowColor: 'black',
            shadowBlur: 5,
            shadowOpacity: 0.6,
            shadowOffsetX: dragItemId === token.id ? 8 : 3,
            shadowOffsetY: dragItemId === token.id ? 8 : 3,
            scaleX: dragItemId === token.id ? 1.2 : 1,
            scaleY: dragItemId === token.id ? 1.2 : 1,
          }"
        />

        <!-- Houses -->
        <v-group
          v-for="house in houses"
          :key="house.id"
          :config="{
            id: house.id,
            x: house.x,
            y: house.y,
            draggable: true,
            scaleX: dragItemId === house.id ? 1.2 : 1,
            scaleY: dragItemId === house.id ? 1.2 : 1,
          }"
        >
          <!-- House body -->
          <v-rect
            :config="{
              x: -15,
              y: -10,
              width: 25,
              height: 20,
              fill: '#00aa00',
              stroke: '#006b00',
              strokeWidth: 0.5,
            }"
          />
          <!-- Windows -->
          <v-rect
            :config="{ x: -12, y: -8, width: 6, height: 6, fill: '#006b00', cornerRadius: 1 }"
          />
          <v-rect
            :config="{ x: 0, y: -8, width: 6, height: 6, fill: '#006b00', cornerRadius: 1 }"
          />
          <v-rect
            :config="{ x: -12, y: 2, width: 6, height: 6, fill: '#006b00', cornerRadius: 1 }"
          />
          <v-rect :config="{ x: 0, y: 2, width: 6, height: 6, fill: '#006b00', cornerRadius: 1 }" />
          <!-- Roof -->
          <v-regular-polygon
            :config="{
              x: -2.5,
              y: -12,
              sides: 3,
              radius: 20,
              scaleY: 0.25,
              fill: '#00aa00',
              stroke: '#006b00',
              strokeWidth: 1.5,
            }"
          />
        </v-group>

        <!-- Hotels -->
        <v-group
          v-for="hotel in hotels"
          :key="hotel.id"
          :config="{
            id: hotel.id,
            x: hotel.x,
            y: hotel.y,
            draggable: true,
            scaleX: dragItemId === hotel.id ? 1.2 : 1,
            scaleY: dragItemId === hotel.id ? 1.2 : 1,
          }"
        >
          <!-- Hotel body -->
          <v-rect
            :config="{
              x: -20,
              y: -15,
              width: 30,
              height: 30,
              fill: '#cc0000',
              stroke: '#A60000',
              strokeWidth: 1.0,
            }"
          />
          <!-- Windows -->
          <v-rect
            :config="{ x: -15, y: -10, width: 8, height: 8, fill: '#A60000', cornerRadius: 1 }"
          />
          <v-rect
            :config="{ x: -3, y: -10, width: 8, height: 8, fill: '#A60000', cornerRadius: 1 }"
          />
          <v-rect
            :config="{ x: -15, y: 2, width: 8, height: 8, fill: '#A60000', cornerRadius: 1 }"
          />
          <v-rect
            :config="{ x: -3, y: 2, width: 8, height: 8, fill: '#A60000', cornerRadius: 1 }"
          />
          <!-- Roof -->
          <v-regular-polygon
            :config="{
              x: -5,
              y: -19,
              radius: 25,
              sides: 3,
              scaleY: 0.3,
              fill: '#cc0000',
              stroke: '#A60000',
              strokeWidth: 1.5,
            }"
          />
        </v-group>
      </v-layer>
    </v-stage>

    <!-- Clear Confirmation Modal -->
    <div v-if="showClearModal" class="modal-overlay" @click="showClearModal = false">
      <div class="modal" @click.stop>
        <h2>Stoppuh??</h2>
        <p>Weet juh zeker dat juh wil stoppuh met Johnopoly??</p>
        <div class="modal-actions">
          <button @click="showClearModal = false" class="btn btn-secondary">Neej</button>
          <button @click="clearBoard" class="btn btn-danger">Ja egt</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.monopoly-container {
  width: 100vw;
  height: 100vh;
  background: #1a1a1a;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  font-family: 'Futura', sans-serif;
}

.toolbar {
  background: #2d2d2d;
  padding: 12px;
  display: flex;
  align-items: center;
  gap: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
  font-family: 'Futura', sans-serif;
}

.btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  font-family: 'Futura', sans-serif;
}

.btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
}

.btn-primary {
  background: #3b82f6;
  color: white;
}

.btn-primary:hover {
  background: #2563eb;
}

.btn-success {
  background: #10b981;
  color: white;
}

.btn-success:hover {
  background: #059669;
}

.btn-warning {
  background: #f59e0b;
  color: white;
}

.btn-warning:hover {
  background: #d97706;
}

.btn-danger {
  background: #ef4444;
  color: white;
}

.btn-danger:hover {
  background: #dc2626;
}

.btn-clear {
  background: #b91c1c;
  color: white;
}

.btn-clear:hover {
  background: #991b1b;
}

.btn-info {
  background: #06b6d4;
  color: white;
}

.btn-info:hover {
  background: #0891b2;
}

.stage {
  cursor: grab;
}

.stage:active {
  cursor: grabbing;
}

.btn-secondary {
  background: #6b7280;
  color: white;
}

.btn-secondary:hover {
  background: #4b5563;
}

.icon {
  font-size: 18px;
}

.divider {
  width: 1px;
  height: 32px;
  background: #4b5563;
}

.color-selector {
  display: flex;
  align-items: center;
  gap: 8px;
}

.label {
  color: white;
  font-size: 14px;
}

.color-buttons {
  display: flex;
  gap: 8px;
}

.color-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 2px solid #4b5563;
  cursor: pointer;
  transition: all 0.2s;
}

.color-btn:hover {
  transform: scale(1.1);
}

.color-btn.active {
  border-color: white;
  transform: scale(1.15);
  box-shadow: 0 0 0 2px rgba(255, 255, 255, 0.3);
}

.spacer {
  flex: 1;
}

.stage {
  flex: 1;
  background: #f5f5f5;
}

.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: white;
  border-radius: 8px;
  padding: 24px;
  max-width: 480px;
  width: calc(100% - 32px);
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
}

.modal h2 {
  font-size: 24px;
  font-weight: bold;
  margin: 0 0 16px 0;
  color: #1f2937;
}

.modal p {
  color: #6b7280;
  margin: 0 0 24px 0;
  line-height: 1.5;
}

.modal-actions {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
}
</style>
