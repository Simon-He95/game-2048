<script setup lang="ts">
import { onMounted, ref } from 'vue'

const size = 4
const board = ref<number[][]>([])
const tiles = ref<{ id: number, value: number, x: number, y: number, moving: boolean, toX: number, toY: number }[]>([])
let tileId = 1
const score = ref(0)
const gameOver = ref(false)
const win = ref(false)
const mode = ref<'normal' | 'oneTwo'>('normal')

function initBoard() {
  board.value = Array.from({ length: size }, () => Array(size).fill(0))
  score.value = 0
  gameOver.value = false
  win.value = false
  addRandomTile()
  addRandomTile()
  syncTiles()
}

function addRandomTile() {
  const empty: [number, number][] = []
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (board.value[i][j] === 0)
        empty.push([i, j])
    }
  }
  if (empty.length === 0)
    return
  const [i, j] = empty[Math.floor(Math.random() * empty.length)]
  tileId++ // 每次新格子递增
  if (mode.value === 'normal') {
    board.value[i][j] = Math.random() < 0.9 ? 2 : 4
  }
  else {
    board.value[i][j] = Math.random() < 0.9 ? 1 : 2
  }
}

function syncTiles(withAnim = false) {
  const arr: typeof tiles.value = []
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (board.value[i][j]) {
        arr.push({
          id: `${i}-${j}-${board.value[i][j]}-${tileId}`, // 保证唯一
          value: board.value[i][j],
          x: j,
          y: i,
          moving: false,
          toX: j,
          toY: i,
        })
      }
    }
  }
  if (withAnim) {
    arr.forEach((tile) => {
      const old = tiles.value.find(t => t.value === tile.value && t.x === tile.x && t.y === tile.y)
      if (old) {
        tile.x = old.x
        tile.y = old.y
      }
      tile.moving = true
    })
    tiles.value = arr
    requestAnimationFrame(() => {
      tiles.value.forEach((tile) => {
        tile.x = tile.toX
        tile.y = tile.toY
        tile.moving = false
      })
    })
  }
  else {
    tiles.value = arr
  }
}

function move(dir: 'left' | 'right' | 'up' | 'down') {
  if (gameOver.value || win.value)
    return
  let moved = false
  function slide(row: number[]) {
    const arr = row.filter(x => x)
    const res: number[] = []
    let i = 0
    while (i < arr.length) {
      if (i < arr.length - 1 && arr[i] === arr[i + 1]) {
        let merged
        if (mode.value === 'normal') {
          merged = arr[i] * 2
        }
        else {
          merged = arr[i] + 1
        }
        res.push(merged)
        score.value += merged
        if (merged === 2048)
          win.value = true
        i += 2
      }
      else {
        res.push(arr[i])
        i += 1
      }
    }
    while (res.length < size) res.push(0)
    return res
  }
  if (dir === 'left') {
    for (let i = 0; i < size; i++) {
      const old = [...board.value[i]]
      board.value[i] = slide(board.value[i])
      if (old.join() !== board.value[i].join())
        moved = true
    }
  }
  if (dir === 'right') {
    for (let i = 0; i < size; i++) {
      const old = [...board.value[i]]
      board.value[i] = slide(board.value[i].slice().reverse()).reverse()
      if (old.join() !== board.value[i].join())
        moved = true
    }
  }
  if (dir === 'up') {
    for (let j = 0; j < size; j++) {
      const col = board.value.map(row => row[j])
      const old = [...col]
      const newCol = slide(col)
      for (let i = 0; i < size; i++) board.value[i][j] = newCol[i]
      if (old.join() !== newCol.join())
        moved = true
    }
  }
  if (dir === 'down') {
    for (let j = 0; j < size; j++) {
      const col = board.value.map(row => row[j])
      const old = [...col]
      const newCol = slide(col.reverse()).reverse()
      for (let i = 0; i < size; i++) board.value[i][j] = newCol[i]
      if (old.join() !== newCol.join())
        moved = true
    }
  }
  if (moved) {
    addRandomTile()
    syncTiles(true) // 触发动画
    setTimeout(() => {
      syncTiles() // 动画结束后同步最终位置
      if (isGameOver())
        gameOver.value = true
    }, 350) // 动画时长与上面一致
  }
}

function isGameOver() {
  for (let i = 0; i < size; i++) {
    for (let j = 0; j < size; j++) {
      if (board.value[i][j] === 0)
        return false
      if (i < size - 1 && board.value[i][j] === board.value[i + 1][j])
        return false
      if (j < size - 1 && board.value[i][j] === board.value[i][j + 1])
        return false
    }
  }
  return true
}

// 键盘控制
function handleKey(e: KeyboardEvent) {
  if (gameOver.value)
    return
  if (['ArrowLeft', 'a', 'A'].includes(e.key))
    move('left')
  if (['ArrowRight', 'd', 'D'].includes(e.key))
    move('right')
  if (['ArrowUp', 'w', 'W'].includes(e.key))
    move('up')
  if (['ArrowDown', 's', 'S'].includes(e.key))
    move('down')
}

// 触摸滑动
let startX = 0
let startY = 0
function onTouchStart(e: TouchEvent) {
  const t = e.touches[0]
  startX = t.clientX
  startY = t.clientY
}
function onTouchEnd(e: TouchEvent) {
  if (gameOver.value)
    return
  const t = e.changedTouches[0]
  const dx = t.clientX - startX
  const dy = t.clientY - startY
  if (Math.abs(dx) > Math.abs(dy)) {
    if (dx > 30)
      move('right')
    else if (dx < -30)
      move('left')
  }
  else {
    if (dy > 30)
      move('down')
    else if (dy < -30)
      move('up')
  }
}

function getTileBg(val: number) {
  switch (val) {
    case 1: return '#eee4da'
    case 2: return '#eee4da'
    case 3: return '#ede0c8'
    case 4: return '#f2b179'
    case 8: return '#f59563'
    case 16: return '#f67c5f'
    case 32: return '#f65e3b'
    case 64: return '#edcf72'
    case 128: return '#edcc61'
    case 256: return '#edc850'
    case 512: return '#edc53f'
    case 1024: return '#edc22e'
    case 2048: return '#3cba54'
    default: return '#3c3a32'
  }
}

function getTileColor(val: number) {
  return val <= 4 ? '#776e65' : '#fff'
}

onMounted(() => {
  initBoard()
  syncTiles()
  window.addEventListener('keydown', handleKey)
})
</script>

<template>
  <div
    class="relative flex flex-col items-center justify-center min-h-screen select-none bg-[#faf8ef]"
    style="overflow: hidden;" @touchstart="onTouchStart" @touchend="onTouchEnd"
  >
    <h1 class="text-4xl font-extrabold mb-4 mt-6 text-yellow-600 drop-shadow">
      2048
    </h1>
    <div class="mb-4 text-2xl font-semibold text-gray-700">
      得分: <span class="text-orange-600">{{ score }}</span>
    </div>
    <div
      class="relative bg-[#bbada0] rounded-xl shadow-lg flex items-center justify-center" :style="{
        width: 'min(96vw, 96vh, 480px)',
        height: 'min(96vw, 96vh, 480px)',
        maxWidth: '480px',
        maxHeight: '480px',
      }"
    >
      <!-- 背景格子 -->
      <div
        v-for="i in size * size" :key="`bg${i}`" class="absolute w-[22%] h-[22%] border rounded-lg" :style="{
          left: `${((i - 1) % size) * 24 + 2}%`,
          top: `${Math.floor((i - 1) / size) * 24 + 2}%`,
          background: '#cdc1b4',
          zIndex: 0,
        }"
      />
      <!-- 动画格子 -->
      <div
        v-for="tile in tiles" :key="tile.id"
        class="absolute flex items-center justify-center rounded-lg font-extrabold border z-10 shadow-lg"
        style="transition: left 0.35s cubic-bezier(.45,1.8,.5,1), top 0.35s cubic-bezier(.45,1.8,.5,1);" :style="{
          width: '22%',
          height: '22%',
          fontSize: 'clamp(1.2rem, 5vw, 2.5rem)',
          left: `${tile.x * 24 + 2}%`,
          top: `${tile.y * 24 + 2}%`,
          background: getTileBg(tile.value),
          color: getTileColor(tile.value),
          borderColor: '#b59f89',
        }"
      >
        {{ tile.value }}
      </div>
    </div>
    <div v-if="gameOver" class="mt-6 text-red-600 font-bold text-2xl">
      游戏结束！
    </div>
    <div class="mt-6 flex flex-row gap-4">
      <button
        class="px-6 py-2 rounded-full text-lg font-bold bg-yellow-400 text-white shadow border-2 border-yellow-600 hover:scale-105 active:scale-95 transition"
        @click="initBoard"
      >
        重新开始
      </button>
      <button
        class="px-6 py-2 rounded-full text-lg font-bold bg-gradient-to-r from-yellow-400 to-orange-400 text-white border-2 border-yellow-600 shadow hover:scale-105 active:scale-95 transition"
        @click="mode = mode === 'normal' ? 'oneTwo' : 'normal'; initBoard()"
      >
        {{ mode === 'normal' ? '切换模式(1/2)'
          : '切换模式(2/4)' }}
      </button>
    </div>
    <div class="mt-6 text-base text-gray-500 text-center">
      用 <b>WASD</b> 或 <b>方向键</b> 控制，手机请滑动操作
    </div>
  </div>
</template>
