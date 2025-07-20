<template>
  <div class="flex items-center justify-center max-w-xl mx-auto my-2 p-2">
    <div v-if="grid" class="w-full grid select-none touch-none border border-neutral-700/90"
      :style="{ gridTemplateColumns: `repeat(${gridSize}, 1fr)`, gridTemplateRows: `repeat(${gridSize}, 1fr)` }">
      <template v-for="y in gridSize" :key="`y${y}`">
        <template v-for="x in gridSize" :key="`x${x}`">
          <div
            class="cell bg-gray-200 aspect-square flex items-center justify-center cursor-pointer"
            :style="{ 
              backgroundColor: colorPalette[grid[y - 1][x - 1].zone % colorPalette.length],
              borderRight: x == gridSize || grid[y - 1][x - 1].zone !== grid[y - 1][x].zone ? '1px solid rgb(64 64 64 / 0.9)' : '1px solid #ffffff30',
              borderBottom: y == gridSize || grid[y - 1][x - 1].zone !== grid[y][x - 1].zone ? '1px solid rgb(64 64 64 / 0.9)' : '1px solid #ffffff30',
              borderLeft: x == 1 || grid[y - 1][x - 1].zone !== grid[y - 1][x - 2].zone ? '1px solid rgb(64 64 64 / 0.9)' : '1px solid #ffffff30',
              borderTop: y == 1 || grid[y - 1][x - 1].zone !== grid[y - 2][x - 1].zone ? '1px solid rgb(64 64 64 / 0.9)' : '1px solid #ffffff30'
            }"
            @pointerenter="onCellClick(x - 1, y - 1)"
            @pointerdown="($event?.target as Element)?.releasePointerCapture?.($event.pointerId); if(touch) $event.preventDefault(); mousedown = true"
            @mousedown="onCellClick(x - 1, y - 1)">
            <Icon v-if="grid[y - 1][x - 1].content === 'queen'" name="tabler:chess-queen-filled" class="size-2/3 pointer-events-none text-neutral-700/90"></Icon>
            <Icon v-else-if="grid[y - 1][x - 1].content?.includes('marker')" name="tabler:point-filled" class="size-1/3 pointer-events-none text-neutral-700/90"></Icon>
          </div>
        </template>
      </template>
    </div>
  </div>
</template>

<script lang="ts" setup>
import Rand, { PRNG } from 'rand-seed'
import { primaryInput } from 'detect-it'

const route = useRoute()
const level = route.query.level
const size = parseInt(route.query.size as string) || 8

if (!level)
  navigateTo(`/?level=${Math.random().toString(36).substring(2, 15)}`)

const rand = new Rand(`${level}`, PRNG.xoshiro128ss)
function randInt(min: number, max: number): number {
  return Math.floor(rand.next() * (max - min + 1)) + min
}

const touch = ref(false)
const mousedown = ref(false)
const dragAction = ref<'queen' | 'marker' | 'remove' | null>(null)
watch(mousedown, (value) => {
  if (!value) dragAction.value = null
})

const gridSize = size < 4 ? 4 : size > 16 ? 16 : size

const colorPalette = [
  '#bba3e2', '#ffc992', '#96beff', '#b3dfa0', '#dfdfdf', '#ff7b60',
  '#e6f388', '#b9b29e', '#dfa0bf', '#a3d2d8', '#62efea', '#ff93f3',
  '#8acc6d', '#729aec', '#c387e0', '#ffe04b'
]

interface Cell {
  x: number
  y: number
  zone: number
  isQueen: boolean
  content: 'queen' | 'auto-marker' | 'user-marker' | null
}

function putMarkersForQueen(x: number, y: number) {
  if (!grid.value) return

  for (let i = 0; i < grid.value.length; i++)
    if (!grid.value[i][x].content)
      grid.value[i][x].content = 'auto-marker'
  for (let i = 0; i < grid.value[y].length; i++)
    if (!grid.value[y][i].content)
      grid.value[y][i].content = 'auto-marker'

  if (x + 1 < gridSize && y + 1 < gridSize && !grid.value[y + 1][x + 1].content)
    grid.value[y + 1][x + 1].content = 'auto-marker'
  if (x - 1 >= 0 && y - 1 >= 0 && !grid.value[y - 1][x - 1].content)
    grid.value[y - 1][x - 1].content = 'auto-marker'
  if (x + 1 < gridSize && y - 1 >= 0 && !grid.value[y - 1][x + 1].content)
    grid.value[y - 1][x + 1].content = 'auto-marker'
  if (x - 1 >= 0 && y + 1 < gridSize && !grid.value[y + 1][x - 1].content)
    grid.value[y + 1][x - 1].content = 'auto-marker'

  grid.value[y][x].content = 'queen'
}

function removeMarkersForQueen(x: number, y: number) {
  if (!grid.value) return

  for (let i = 0; i < grid.value.length; i++)
    if (grid.value[i][x].content === 'auto-marker')
      grid.value[i][x].content = null
  for (let i = 0; i < grid.value[y].length; i++)
    if (grid.value[y][i].content === 'auto-marker')
      grid.value[y][i].content = null

  if (x + 1 < gridSize && y + 1 < gridSize && grid.value[y + 1][x + 1].content === 'auto-marker')
    grid.value[y + 1][x + 1].content = null
  if (x - 1 >= 0 && y - 1 >= 0 && grid.value[y - 1][x - 1].content === 'auto-marker')
    grid.value[y - 1][x - 1].content = null
  if (x + 1 < gridSize && y - 1 >= 0 && grid.value[y - 1][x + 1].content === 'auto-marker')
    grid.value[y - 1][x + 1].content = null
  if (x - 1 >= 0 && y + 1 < gridSize && grid.value[y + 1][x - 1].content === 'auto-marker')
    grid.value[y + 1][x - 1].content = null
}

function onCellClick(x: number, y: number) {  
  if (!grid.value || !(mousedown.value || touch.value))
    return

  const cell = grid.value[y][x]
  if (dragAction.value) {
    if (dragAction.value === 'marker' && cell.content === null)
      cell.content = 'user-marker'
    if (dragAction.value === 'remove' && cell.content?.includes('marker'))
      cell.content = null
  } else if (!cell.content) {
    cell.content = 'user-marker'
    dragAction.value = 'marker'
  } else if (cell.content.includes('marker')) {
    cell.content = 'queen'
    dragAction.value = 'queen'
    putMarkersForQueen(x, y)
  } else {
    cell.content = null
    dragAction.value = 'remove'
    removeMarkersForQueen(x, y)
  }
}

function shuffleArray(array: any[]) {
  let currentIndex = array.length;
  while (currentIndex != 0) {
    const randomIndex = Math.floor(rand.next() * currentIndex);
    currentIndex--;
    [array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
  }
}

function generateBoard(gridSize: number): Cell[][] {
  function shuffleQueens(gridSize: number) {
    function checkIsPositionValid(pos: number, lastPos: number): boolean {
      return Math.abs(pos - lastPos) > 1
    }

    const positions: number[] = []
    const availablePositions = Array.from(Array(gridSize).keys())
    shuffleArray(availablePositions)
    for (let i = 0; i < gridSize; i++) {
      let pos = availablePositions.find(p => i === 0 || checkIsPositionValid(p, positions[i - 1]))
      if (pos === undefined)
        return shuffleQueens(gridSize) 
      positions.push(pos)
      availablePositions.splice(availablePositions.indexOf(pos), 1)
    }
    return positions
  }

  function generateZones(queensPlacement: number[]): number[][] {
    const gridSize = queensPlacement.length
    const matrix = Array.from(new Array(gridSize), _ => Array(gridSize).fill(-1))
    matrix.forEach((row, y) => row[queensPlacement[y]] = y)
    while (matrix.some(row => row.includes(-1))) {
      const x = randInt(0, gridSize - 1)
      const y = randInt(0, gridSize - 1)
      if (matrix[y][x] > -1)
        continue
      switch (randInt(0, 3)) {
        case 0: // Up
          if (y > 0 && matrix[y - 1][x] > -1) matrix[y][x] = matrix[y - 1][x]
          break
        case 1: // Down
          if (y < gridSize - 1 && matrix[y + 1][x] > -1) matrix[y][x] = matrix[y + 1][x]
          break
        case 2: // Left
          if (x > 0 && matrix[y][x - 1] > -1) matrix[y][x] = matrix[y][x - 1]
          break
        case 3: // Right
          if (x < gridSize - 1 && matrix[y][x + 1] > -1) matrix[y][x] = matrix[y][x + 1]
          break
      }
    }
    return matrix
  }

  // Initialize the board with empty cells
  const board: Cell[][] = []
  const queensPlacement = shuffleQueens(gridSize)
  const zones  = generateZones(queensPlacement)

  for (let y = 0; y < gridSize; y++) {
    board[y] = []
    for (let x = 0; x < gridSize; x++) {
      board[y][x] = {
        x,
        y,
        zone: zones[y][x],
        isQueen: queensPlacement[y] === x,
        content: null
      }
    }
  }
  return board
}

const grid = ref<Cell[][]>()
grid.value = generateBoard(gridSize)

onMounted(() => {
  touch.value = primaryInput === 'touch'
  document.addEventListener('mouseup', () => mousedown.value = false)
  document.addEventListener('pointerup', () => mousedown.value = false)
})

</script>

<style>
@media (pointer: fine) {
  .cell {
    position: relative;
  }

  .cell:hover:after {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background-color: #ffffff30;
  }
}
</style>