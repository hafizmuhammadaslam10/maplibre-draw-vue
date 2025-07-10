<template>
  <div ref="mapContainer" class="map-container">
    <!-- 🟢 Drawing Toggle Button -->
    <button @click="toggleDrawingMode" class="draw-toggle">
      {{ drawingEnabled ? "Disable" : "Enable" }} Drawing
    </button>

    <!-- 🎛 Brush Controls (only visible when drawing is enabled) -->
    <div v-if="drawingEnabled" class="draw-tools">
      <label>
        Color:
        <select v-model="selectedColor">
          <option value="orange">Orange</option>
          <option value="blue">Blue</option>
          <option value="green">Green</option>
          <option value="black">Black</option>
        </select>
      </label>

      <label>
        Size:
        <input type="range" min="1" max="20" v-model="brushSize" />
        <span>{{ brushSize }}px</span>
      </label>
      <div class="undo-redo-buttons">
        <button @click="undoLastLine" :disabled="lines.length === 0">Undo</button>
        <button @click="redoLastUndo" :disabled="undoStack.length === 0">Redo</button>
      </div>
      <button @click="toggleEraser">
        {{ isEraser ? "Switch to Pen" : "Switch to Eraser" }}
      </button>
      <button @click="clearAllLines">
        Clear All
      </button>
      <button @click="saveDrawingAsImage">
        Save Drawing
      </button>

    </div>


    <!-- 🖌️ Drawing Canvas Overlay -->
    <div class="canvas-wrapper">
      <v-stage :config="{ width: width, height: height }" @mousedown="startDrawing" @mousemove="draw"
        @mouseup="endDrawing">
        <v-layer ref="layerRef" />
      </v-stage>
    </div>
  </div>
</template>

<script setup lang="ts">

import { ref, onMounted } from "vue";
import maplibregl from "maplibre-gl";
import Konva from "konva";
import { useWindowSize } from "@vueuse/core";

const lines = ref<Konva.Line[]>([]);
const undoStack = ref<Konva.Line[]>([]);


// Responsive canvas size
const { width, height } = useWindowSize();

// DOM references
const mapContainer = ref<HTMLDivElement | null>(null);
const isEraser = ref(false);
const layerRef = ref();

// Drawing state
const drawingEnabled = ref(false);
const isDrawing = ref(false);
let currentLine: Konva.Line | null = null;

// Brush settings
const selectedColor = ref("orange");
const brushSize = ref(5);

// Toggle drawing mode
function toggleDrawingMode() {
  drawingEnabled.value = !drawingEnabled.value;
}

function toggleEraser() {
  isEraser.value = !isEraser.value;
}

// Start drawing
function startDrawing(e: any) {
  if (!drawingEnabled.value) return;
  isDrawing.value = true;

  const stage = e.target.getStage();
  const pos = stage.getPointerPosition();
  const layer = layerRef.value?.getNode();
  if (!layer || !pos) return;

  currentLine = new Konva.Line({
    stroke: selectedColor.value,
    strokeWidth: isEraser.value ? brushSize.value * 2 : brushSize.value,
    globalCompositeOperation: isEraser.value ? "destination-out" : "source-over",
    lineCap: "round",
    tension: isEraser.value ? 0 : 0.5,
    bezier: !isEraser.value,
    points: [pos.x, pos.y],
  });

  layer.add(currentLine);
  if (!isEraser.value) {
    lines.value.push(currentLine);
  } else {
    undoStack.value = []; // clear redo stack on erase
  }

}

// Continue drawing
function draw(e: any) {
  if (!drawingEnabled.value || !isDrawing.value || !currentLine) return;

  const stage = e.target.getStage();
  const pos = stage.getPointerPosition();
  if (!pos) return;

  const newPoints = currentLine.points().concat([pos.x, pos.y]);
  currentLine.points(newPoints);

  requestAnimationFrame(() => {
    currentLine?.getLayer()?.batchDraw();
  });
}

// End drawing
function endDrawing() {
  isDrawing.value = false;
  currentLine = null;
}

function undoLastLine() {
  if (lines.value.length === 0) return;
  const lastLine = lines.value.pop();
  if (lastLine) {
    lastLine.hide(); // hide instead of destroy
    undoStack.value.push(lastLine);
    layerRef.value?.getNode().draw();
  }
}

function redoLastUndo() {
  if (undoStack.value.length === 0) return;
  const line = undoStack.value.pop();
  if (line) {
    line.show();
    lines.value.push(line);
    layerRef.value?.getNode().draw();
  }
}

function clearAllLines() {
  const confirmClear = confirm("Are you sure you want to clear all drawings?");
  if (!confirmClear) return;

  const layer = layerRef.value?.getNode();
  if (!layer) return;

  layer.destroyChildren(); // remove all lines from canvas
  layer.draw();

  lines.value = [];
  undoStack.value = [];
}

function saveDrawingAsImage() {
  const stage = layerRef.value?.getNode()?.getStage();
  if (!stage) return;

  const dataURL = stage.toDataURL({
    pixelRatio: 2, // higher quality
  });

  const link = document.createElement("a");
  link.download = "drawing.png";
  link.href = dataURL;
  link.click();
}



// Initialize MapLibre on mount
onMounted(() => {
  new maplibregl.Map({
    container: mapContainer.value!,
    style: "https://demotiles.maplibre.org/style.json",
    center: [0, 0],
    zoom: 2,
  });
  window.addEventListener("keydown", (e) => {
    if (e.ctrlKey && e.key === "z") {
      e.preventDefault();
      if (drawingEnabled.value) undoLastLine();
    }
  });
});
</script>

<style scoped>
.map-container {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
}

/* Ensure MapLibre map stays below Konva */
.map-container>div:first-child {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 0;
}

/* Canvas overlay (Konva) */
.canvas-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 10;
  width: 100%;
  height: 100%;
  pointer-events: auto;
}

.canvas-wrapper canvas {
  width: 100% !important;
  height: 100% !important;
  pointer-events: auto;
}

/* Toggle drawing button */
.draw-toggle {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 20;
  background-color: white;
  border: 1px solid #ccc;
  padding: 8px 14px;
  border-radius: 6px;
  font-weight: bold;
  cursor: pointer;
  user-select: none;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
}

/* Brush controls visible when drawing */
.draw-tools {
  position: absolute;
  top: 60px;
  left: 10px;
  z-index: 20;
  background: white;
  padding: 10px 14px;
  border-radius: 6px;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  display: flex;
  gap: 12px;
  align-items: center;
  user-select: none;
}

.draw-tools label {
  display: flex;
  align-items: center;
  font-size: 14px;
  gap: 6px;
}

.undo-redo-buttons {
  display: flex;
  gap: 10px;
  margin-top: 8px;
}

.undo-redo-buttons button {
  padding: 6px 10px;
  font-size: 13px;
  border-radius: 4px;
  border: 1px solid #ccc;
  cursor: pointer;
  background: #f7f7f7;
}

.undo-redo-buttons button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.draw-tools button {
  padding: 6px 10px;
  font-size: 13px;
  border-radius: 4px;
  border: 1px solid #ccc;
  background: #f7f7f7;
  cursor: pointer;
}

.draw-tools button:hover {
  background-color: #eee;
}

.draw-tools button:last-child {
  background-color: #ff4d4d;
  color: white;
  border: none;
}

.draw-tools button:last-child:hover {
  background-color: #e60000;
}

.draw-tools button {
  /* already styled previously */
}

.draw-tools button:hover {
  background-color: #e0e0e0;
}
</style>
