<template>
  <main class="page">
    <h1>Pixel Editor</h1>

    <section class="layout">
      <!-- Left: tools -->
      <aside class="panel">
        <h2>Tools</h2>

        <label class="row">
          Grid:
          <select v-model.number="gridSize">
            <option :value="16">16 × 16</option>
            <option :value="32">32 × 32</option>
            <option :value="50">50 × 50</option>
            <option :value="64">64 × 64</option>
          </select>
        </label>

        <label class="row">
          Cell size:
          <input type="range" min="8" max="30" v-model.number="cellSize" />
          <span>{{ cellSize }}px</span>
        </label>

        <label class="row">
          Mode:
          <select v-model="mode">
            <option value="draw">Draw</option>
            <option value="erase">Erase</option>
          </select>
        </label>

        <button class="btn" @click="clearGrid" style="background-color: #FFFBA8; color: black;">Clear</button>
        <button class="btn" @click="randomFill" style="background-color: #FFFBA8; color: black;">Random</button>

        <hr />

        <h2>Palette</h2>
        <div class="palette">
          <button
            v-for="(c, idx) in palette"
            :key="c"
            class="swatch"
            :style="{ background: c, outline: idx === selectedColor ? '3px solid #333' : '1px solid #f7f5f5ff' }"
            @click="selectedColor = idx"
            :title="c"
          />
        </div>

        <label class="row">
          Add color:
          <input type="color" v-model="newColor" />
          <button class="btn" @click="addColor" style="background-color: #FFFBA8; color: black;">Add</button>
        </label>

        <hr />

        <h2>Export</h2>
        <button class="btn" @click="exportJSON" style="background-color: #ef1f14ff; color: white;">Download JSON</button>
        <button class="btn" @click="exportPNG" style="background-color: #ef1f14ff; color: white;">Download PNG</button>
      </aside>

      <!-- Right: canvas -->
      <section class="canvasWrap">
        <canvas
          ref="canvasRef"
          :width="gridSize * cellSize"
          :height="gridSize * cellSize"
          class="canvas"
          @mousedown="onMouseDown"
          @mousemove="onMouseMove"
          @mouseup="onMouseUp"
          @mouseleave="onMouseUp"
        />
        <p class="hint">Click + drag to paint. ({{ gridSize }}×{{ gridSize }})</p>
      </section>
    </section>
  </main>
</template>

<script setup>
import { computed, nextTick, onMounted, ref, watch } from "vue";

const canvasRef = ref(null);

const gridSize = ref(32);
const cellSize = ref(16);

const palette = ref([
  "#FFFFFF", // 0 background
  "#111827", // 1 near-black
  "#FF6FB1", // 2 pink
  "#7CDBD5", // 3 mint
  "#FFD166", // 4 warm yellow
]);

const selectedColor = ref(1);
const newColor = ref("#A78BFA"); // purple

// grid stores palette indexes (0..palette.length-1)
const grid = ref([]);

// painting state
const isPainting = ref(false);
const mode = ref("draw"); // draw | erase

const totalCells = computed(() => gridSize.value * gridSize.value);

function initGrid() {
  grid.value = new Array(totalCells.value).fill(0);
}

function indexFromXY(x, y) {
  return y * gridSize.value + x;
}

function getCellFromMouseEvent(evt) {
  const canvas = canvasRef.value;
  const rect = canvas.getBoundingClientRect();
  const px = evt.clientX - rect.left;
  const py = evt.clientY - rect.top;

  const x = Math.floor(px / cellSize.value);
  const y = Math.floor(py / cellSize.value);

  if (x < 0 || y < 0 || x >= gridSize.value || y >= gridSize.value) return null;
  return { x, y };
}

function paintCell(x, y) {
  const idx = indexFromXY(x, y);
  const colorIndex = mode.value === "erase" ? 0 : selectedColor.value;
  if (grid.value[idx] === colorIndex) return;

  grid.value[idx] = colorIndex;
  drawCell(x, y); // incremental draw for speed
}

function drawGrid() {
  const canvas = canvasRef.value;
  const ctx = canvas.getContext("2d");

  // background
  ctx.clearRect(0, 0, canvas.width, canvas.height);

  // draw cells
  for (let y = 0; y < gridSize.value; y++) {
    for (let x = 0; x < gridSize.value; x++) {
      drawCell(x, y, ctx);
    }
  }

  // grid lines (optional, light)
  ctx.globalAlpha = 0.15;
  ctx.strokeStyle = "#000";
  for (let i = 0; i <= gridSize.value; i++) {
    // vertical
    ctx.beginPath();
    ctx.moveTo(i * cellSize.value + 0.5, 0);
    ctx.lineTo(i * cellSize.value + 0.5, gridSize.value * cellSize.value);
    ctx.stroke();
    // horizontal
    ctx.beginPath();
    ctx.moveTo(0, i * cellSize.value + 0.5);
    ctx.lineTo(gridSize.value * cellSize.value, i * cellSize.value + 0.5);
    ctx.stroke();
  }
  ctx.globalAlpha = 1;
}

function drawCell(x, y, providedCtx) {
  const canvas = canvasRef.value;
  const ctx = providedCtx ?? canvas.getContext("2d");

  const idx = indexFromXY(x, y);
  const color = palette.value[grid.value[idx]] ?? "#FFFFFF";

  ctx.fillStyle = color;
  ctx.fillRect(
    x * cellSize.value,
    y * cellSize.value,
    cellSize.value,
    cellSize.value
  );
}

function onMouseDown(evt) {
  isPainting.value = true;
  const cell = getCellFromMouseEvent(evt);
  if (cell) paintCell(cell.x, cell.y);
}

function onMouseMove(evt) {
  if (!isPainting.value) return;
  const cell = getCellFromMouseEvent(evt);
  if (cell) paintCell(cell.x, cell.y);
}

function onMouseUp() {
  isPainting.value = false;
}

function clearGrid() {
  initGrid();
  drawGrid();
}

function randomFill() {
  for (let i = 0; i < grid.value.length; i++) {
    grid.value[i] = Math.floor(Math.random() * palette.value.length);
  }
  drawGrid();
}

function addColor() {
  const c = newColor.value.toUpperCase();
  if (!palette.value.includes(c)) palette.value.push(c);
}

function downloadFile(filename, content, mime) {
  const blob = new Blob([content], { type: mime });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}

function exportJSON() {
  const data = {
    width: gridSize.value,
    height: gridSize.value,
    cellSize: cellSize.value,
    palette: palette.value,
    grid: grid.value, // palette indexes
  };
  downloadFile("pattern.json", JSON.stringify(data, null, 2), "application/json");
}

function exportPNG() {
  const canvas = canvasRef.value;
  const url = canvas.toDataURL("image/png");
  const a = document.createElement("a");
  a.href = url;
  a.download = "pattern.png";
  a.click();
}

// re-init when gridSize changes
watch(gridSize, async () => {
  initGrid();
  await nextTick();
  drawGrid();
});

// redraw when cellSize changes (canvas size changes)
watch(cellSize, async () => {
  await nextTick();
  drawGrid();
});

// redraw if palette changes (colors changed)
watch(palette, () => drawGrid(), { deep: true });

onMounted(() => {
  initGrid();
  drawGrid();
});
</script>

<style scoped>
.page {
  font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
  padding: 20px;
}
.layout {
  display: grid;
  grid-template-columns: 320px 1fr;
  gap: 16px;
  align-items: start;
}
.panel {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  padding: 14px;
}
.row {
  display: flex;
  gap: 10px;
  align-items: center;
  margin: 10px 0;
  flex-wrap: wrap;
}
.btn {
  width: 100%;
  margin: 6px 0;
  padding: 10px 12px;
  border-radius: 10px;
  border: 1px solid #d1d5db;
  background: #fff;
  cursor: pointer;
}
.btn:hover {
  background: #f9fafb;
}
.palette {
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 8px;
  margin: 10px 0;
}
.swatch {
  width: 100%;
  aspect-ratio: 1 / 1;
  border-radius: 10px;
  border: 1px solid #ccc;
  cursor: pointer;
}
.canvasWrap {
  display: grid;
  gap: 10px;
  justify-items: start;
}
.canvas {
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  background: #fff;
  image-rendering: pixelated;
}
.hint {
  opacity: 0.7;
  margin: 0;
}
</style>
