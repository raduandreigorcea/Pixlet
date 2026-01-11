<script setup lang="ts">
import { ref, computed, onMounted } from 'vue';
import PencilIcon from './assets/pencil-line.svg';
import EraserIcon from './assets/eraser.svg';
import FillIcon from './assets/paint-bucket.svg';
import EyedropperIcon from './assets/pipette.svg';
import ClearIcon from './assets/circle-x.svg';
import ResizeIcon from './assets/ruler.svg';
import SaveIcon from './assets/save.svg';

// Grid settings
const gridWidth = ref(32);
const gridHeight = ref(32);

// Fixed canvas size (like modern pixel art editors)
const CANVAS_SIZE = 640; // Fixed 640x640px canvas

// Zoom state
const zoomLevel = ref(1);
const MIN_ZOOM = 0.25;
const MAX_ZOOM = 4;

// Calculate cell size based on grid dimensions
const pixelSize = computed(() => {
  // The canvas is always CANVAS_SIZE x CANVAS_SIZE
  // Cell size adjusts to fit the grid within this fixed space
  const cellWidth = CANVAS_SIZE / gridWidth.value;
  const cellHeight = CANVAS_SIZE / gridHeight.value;
  
  // Use the smaller dimension to keep cells square
  return Math.min(cellWidth, cellHeight);
});

// Grid data - 2D array of colors
const grid = ref<string[][]>([]);
const canvas = ref<HTMLCanvasElement | null>(null);
const ctx = ref<CanvasRenderingContext2D | null>(null);

// Drawing state
const currentColor = ref('#000000');
const currentTool = ref<'pencil' | 'eraser' | 'fill' | 'eyedropper'>('pencil');
const isDrawing = ref(false);
const lastDrawnPos = ref<{ row: number; col: number } | null>(null);
const brushSize = ref(1);
const hoveredCell = ref<{ row: number; col: number } | null>(null);

// History state for undo/redo
const history = ref<string[][][]>([]);
const historyIndex = ref(-1);
const MAX_HISTORY = 50;

// Dialog state
const dialogVisible = ref(false);
const dialogType = ref<'confirm' | 'prompt' | 'alert'>('alert');
const dialogTitle = ref('');
const dialogMessage = ref('');
const dialogInput = ref('');
const dialogCallback = ref<((value?: string | boolean) => void) | null>(null);

const showDialog = (type: 'confirm' | 'prompt' | 'alert', title: string, message: string, callback: (value?: string | boolean) => void) => {
  dialogType.value = type;
  dialogTitle.value = title;
  dialogMessage.value = message;
  dialogInput.value = '';
  dialogCallback.value = callback;
  dialogVisible.value = true;
};

const closeDialog = () => {
  dialogVisible.value = false;
  dialogCallback.value = null;
};

const confirmDialog = () => {
  if (dialogCallback.value) {
    if (dialogType.value === 'prompt') {
      dialogCallback.value(dialogInput.value);
    } else if (dialogType.value === 'confirm') {
      dialogCallback.value(true);
    } else {
      dialogCallback.value();
    }
  }
  closeDialog();
};

const cancelDialog = () => {
  if (dialogCallback.value && dialogType.value === 'confirm') {
    dialogCallback.value(false);
  }
  closeDialog();
};

// Color history - track recently used colors
const colorHistory = ref<string[]>(['#000000', '#FFFFFF']);

// Save current state to history
const saveState = () => {
  // Remove any states after current index (when doing new action after undo)
  history.value = history.value.slice(0, historyIndex.value + 1);
  
  // Deep copy the grid
  const stateCopy = grid.value.map(row => [...row]);
  history.value.push(stateCopy);
  
  // Limit history size
  if (history.value.length > MAX_HISTORY) {
    history.value.shift();
  } else {
    historyIndex.value++;
  }
};

// Undo action
const undo = () => {
  if (historyIndex.value > 0) {
    historyIndex.value--;
    grid.value = history.value[historyIndex.value].map(row => [...row]);
    renderCanvas();
  }
};

// Redo action
const redo = () => {
  if (historyIndex.value < history.value.length - 1) {
    historyIndex.value++;
    grid.value = history.value[historyIndex.value].map(row => [...row]);
    renderCanvas();
  }
};

// Initialize grid
const initializeGrid = () => {
  grid.value = Array(gridHeight.value)
    .fill(null)
    .map(() => Array(gridWidth.value).fill('#FFFFFF'));
  renderCanvas();
  
  // Reset history and save initial state
  history.value = [];
  historyIndex.value = -1;
  saveState();
};

// Render grid to canvas
const renderCanvas = () => {
  if (!ctx.value || !canvas.value) return;
  
  canvas.value.width = CANVAS_SIZE;
  canvas.value.height = CANVAS_SIZE;
  
  for (let row = 0; row < gridHeight.value; row++) {
    for (let col = 0; col < gridWidth.value; col++) {
      ctx.value.fillStyle = grid.value[row][col];
      ctx.value.fillRect(
        col * pixelSize.value,
        row * pixelSize.value,
        pixelSize.value,
        pixelSize.value
      );
    }
  }
  
  drawGridLines();
};

// Draw grid lines overlay
const drawGridLines = () => {
  if (!ctx.value) return;
  
  ctx.value.strokeStyle = 'rgba(0, 0, 0, 0.1)';
  ctx.value.lineWidth = 1;
  
  // Vertical lines
  for (let i = 0; i <= gridWidth.value; i++) {
    ctx.value.beginPath();
    ctx.value.moveTo(i * pixelSize.value + 0.5, 0);
    ctx.value.lineTo(i * pixelSize.value + 0.5, gridHeight.value * pixelSize.value);
    ctx.value.stroke();
  }
  
  // Horizontal lines
  for (let i = 0; i <= gridHeight.value; i++) {
    ctx.value.beginPath();
    ctx.value.moveTo(0, i * pixelSize.value + 0.5);
    ctx.value.lineTo(gridWidth.value * pixelSize.value, i * pixelSize.value + 0.5);
    ctx.value.stroke();
  }
};

// Render hover preview
const renderHoverPreview = () => {
  if (!ctx.value || !hoveredCell.value || isDrawing.value) return;
  
  // First, re-render the canvas to clear previous preview
  renderCanvas();
  
  const { row, col } = hoveredCell.value;
  const offset = Math.floor(brushSize.value / 2);
  
  // Determine preview color based on tool
  let previewColor = currentColor.value;
  if (currentTool.value === 'eraser') {
    previewColor = '#FFFFFF';
  } else if (currentTool.value === 'fill' || currentTool.value === 'eyedropper') {
    // No brush preview for fill and eyedropper
    return;
  }
  
  // Convert hex to rgba with transparency
  const r = parseInt(previewColor.slice(1, 3), 16);
  const g = parseInt(previewColor.slice(3, 5), 16);
  const b = parseInt(previewColor.slice(5, 7), 16);
  ctx.value.fillStyle = `rgba(${r}, ${g}, ${b}, 0.5)`;
  
  // Draw preview for brush size
  for (let r = -offset; r < brushSize.value - offset; r++) {
    for (let c = -offset; c < brushSize.value - offset; c++) {
      const newRow = row + r;
      const newCol = col + c;
      if (newRow >= 0 && newRow < gridHeight.value && newCol >= 0 && newCol < gridWidth.value) {
        ctx.value.fillRect(
          newCol * pixelSize.value,
          newRow * pixelSize.value,
          pixelSize.value,
          pixelSize.value
        );
      }
    }
  }
  
  // Redraw grid lines on top of preview
  drawGridLines();
};

// Add color to history when used
const addToColorHistory = (color: string) => {
  // Remove color if it already exists in history
  const filtered = colorHistory.value.filter(c => c !== color);
  // Add to beginning
  colorHistory.value = [color, ...filtered].slice(0, 20); // Keep last 20 colors
};

// Drawing functions
const startDrawing = (row: number, col: number) => {
  isDrawing.value = true;
  hoveredCell.value = null; // Clear hover preview when drawing starts
  lastDrawnPos.value = { row, col };
  drawPixel(row, col);
};

const stopDrawing = () => {
  if (isDrawing.value) {
    saveState(); // Save state after drawing
  }
  isDrawing.value = false;
  lastDrawnPos.value = null;
};

// Bresenham's line algorithm to draw line between two points
const drawLine = (row0: number, col0: number, row1: number, col1: number) => {
  const dx = Math.abs(col1 - col0);
  const dy = Math.abs(row1 - row0);
  const sx = col0 < col1 ? 1 : -1;
  const sy = row0 < row1 ? 1 : -1;
  let err = dx - dy;

  let currentRow = row0;
  let currentCol = col0;

  while (true) {
    drawPixel(currentRow, currentCol);

    if (currentRow === row1 && currentCol === col1) break;

    const e2 = 2 * err;
    if (e2 > -dy) {
      err -= dy;
      currentCol += sx;
    }
    if (e2 < dx) {
      err += dx;
      currentRow += sy;
    }
  }
};

const drawPixel = (row: number, col: number) => {
  if (row < 0 || row >= gridHeight.value || col < 0 || col >= gridWidth.value) return;

  if (currentTool.value === 'pencil') {
    const color = currentColor.value;
    addToColorHistory(color);
    
    // Draw with brush size
    const offset = Math.floor(brushSize.value / 2);
    for (let r = -offset; r < brushSize.value - offset; r++) {
      for (let c = -offset; c < brushSize.value - offset; c++) {
        const newRow = row + r;
        const newCol = col + c;
        if (newRow >= 0 && newRow < gridHeight.value && newCol >= 0 && newCol < gridWidth.value) {
          grid.value[newRow][newCol] = color;
          renderSinglePixel(newRow, newCol, color);
        }
      }
    }
  } else if (currentTool.value === 'eraser') {
    // Erase with brush size
    const offset = Math.floor(brushSize.value / 2);
    for (let r = -offset; r < brushSize.value - offset; r++) {
      for (let c = -offset; c < brushSize.value - offset; c++) {
        const newRow = row + r;
        const newCol = col + c;
        if (newRow >= 0 && newRow < gridHeight.value && newCol >= 0 && newCol < gridWidth.value) {
          grid.value[newRow][newCol] = '#FFFFFF';
          renderSinglePixel(newRow, newCol, '#FFFFFF');
        }
      }
    }
  } else if (currentTool.value === 'fill') {
    floodFill(row, col, grid.value[row][col], currentColor.value);
    addToColorHistory(currentColor.value);
    renderCanvas();
    saveState(); // Save state after fill
  } else if (currentTool.value === 'eyedropper') {
    currentColor.value = grid.value[row][col];
    currentTool.value = 'pencil';
  }
};

// Render single pixel for performance
const renderSinglePixel = (row: number, col: number, color: string) => {
  if (!ctx.value) return;
  
  ctx.value.fillStyle = color;
  ctx.value.fillRect(
    col * pixelSize.value,
    row * pixelSize.value,
    pixelSize.value,
    pixelSize.value
  );
  
  // Draw grid lines around this pixel only
  ctx.value.strokeStyle = 'rgba(0, 0, 0, 0.1)';
  ctx.value.lineWidth = 1;
  
  // Top
  ctx.value.beginPath();
  ctx.value.moveTo(col * pixelSize.value + 0.5, row * pixelSize.value + 0.5);
  ctx.value.lineTo((col + 1) * pixelSize.value + 0.5, row * pixelSize.value + 0.5);
  ctx.value.stroke();
  
  // Right
  ctx.value.beginPath();
  ctx.value.moveTo((col + 1) * pixelSize.value + 0.5, row * pixelSize.value + 0.5);
  ctx.value.lineTo((col + 1) * pixelSize.value + 0.5, (row + 1) * pixelSize.value + 0.5);
  ctx.value.stroke();
  
  // Bottom
  ctx.value.beginPath();
  ctx.value.moveTo(col * pixelSize.value + 0.5, (row + 1) * pixelSize.value + 0.5);
  ctx.value.lineTo((col + 1) * pixelSize.value + 0.5, (row + 1) * pixelSize.value + 0.5);
  ctx.value.stroke();
  
  // Left
  ctx.value.beginPath();
  ctx.value.moveTo(col * pixelSize.value + 0.5, row * pixelSize.value + 0.5);
  ctx.value.lineTo(col * pixelSize.value + 0.5, (row + 1) * pixelSize.value + 0.5);
  ctx.value.stroke();
};

const getCanvasCoordinates = (event: MouseEvent): { row: number; col: number } | null => {
  if (!canvas.value) return null;
  
  const rect = canvas.value.getBoundingClientRect();
  const scaleX = CANVAS_SIZE / rect.width;
  const scaleY = CANVAS_SIZE / rect.height;
  
  const x = (event.clientX - rect.left) * scaleX;
  const y = (event.clientY - rect.top) * scaleY;
  
  const col = Math.floor(x / pixelSize.value);
  const row = Math.floor(y / pixelSize.value);
  
  // Clamp to grid bounds
  const clampedRow = Math.max(0, Math.min(gridHeight.value - 1, row));
  const clampedCol = Math.max(0, Math.min(gridWidth.value - 1, col));
  
  return { row: clampedRow, col: clampedCol };
};

const onCanvasMouseDown = (event: MouseEvent) => {
  event.preventDefault();
  const coords = getCanvasCoordinates(event);
  if (coords) {
    startDrawing(coords.row, coords.col);
  }
};

const onCanvasMouseMove = (event: MouseEvent) => {
  // Always try to get coordinates, even outside canvas during drag
  const coords = getCanvasCoordinates(event);
  
  // Check if left mouse button is pressed
  if (event.buttons === 1) {
    if (!isDrawing.value && (currentTool.value === 'pencil' || currentTool.value === 'eraser')) {
      // Resume drawing if button is held
      isDrawing.value = true;
    }
    
    if (isDrawing.value && coords && (currentTool.value === 'pencil' || currentTool.value === 'eraser')) {
      if (lastDrawnPos.value && (lastDrawnPos.value.row !== coords.row || lastDrawnPos.value.col !== coords.col)) {
        // Draw line from last position to current position
        drawLine(lastDrawnPos.value.row, lastDrawnPos.value.col, coords.row, coords.col);
      } else if (!lastDrawnPos.value) {
        drawPixel(coords.row, coords.col);
      }
      lastDrawnPos.value = coords;
    }
  } else {
    // Update hover preview when not drawing
    if (coords) {
      hoveredCell.value = coords;
      renderHoverPreview();
    }
  }
};

const onCanvasMouseUp = () => {
  stopDrawing();
};

const onCanvasMouseLeave = () => {
  // Keep the last position so we can draw a line when re-entering
  // Don't set to null - this allows continuous lines when dragging in and out
  
  // Clear hover preview when mouse leaves canvas
  if (hoveredCell.value) {
    hoveredCell.value = null;
    renderCanvas();
  }
};

// Flood fill algorithm
const floodFill = (row: number, col: number, targetColor: string, replacementColor: string) => {
  if (targetColor === replacementColor) return;
  if (row < 0 || row >= gridHeight.value || col < 0 || col >= gridWidth.value) return;
  if (grid.value[row][col] !== targetColor) return;

  grid.value[row][col] = replacementColor;

  floodFill(row - 1, col, targetColor, replacementColor);
  floodFill(row + 1, col, targetColor, replacementColor);
  floodFill(row, col - 1, targetColor, replacementColor);
  floodFill(row, col + 1, targetColor, replacementColor);
};

// Grid management
const clearGrid = () => {
  showDialog('confirm', 'Clear Canvas', 'Are you sure you want to clear the entire canvas?', (confirmed) => {
    if (confirmed) {
      for (let row = 0; row < gridHeight.value; row++) {
        for (let col = 0; col < gridWidth.value; col++) {
          grid.value[row][col] = '#FFFFFF';
        }
      }
      renderCanvas();
      saveState();
    }
  });
};

const resizeGrid = () => {
  showDialog('prompt', 'Resize Canvas', `Enter grid size (8-64, current: ${gridWidth.value}x${gridHeight.value}):`, (value) => {
    if (!value) return;
    
    const newSize = parseInt(value as string);
    
    if (isNaN(newSize)) {
      showDialog('alert', 'Invalid Input', 'Please enter a valid number.', () => {});
      return;
    }
    
    if (newSize < 8 || newSize > 64) {
      showDialog('alert', 'Invalid Size', 'Grid size must be between 8 and 64.', () => {});
      return;
    }
    
    gridWidth.value = newSize;
    gridHeight.value = newSize;
    initializeGrid();
  });
};

// Export functions
const exportAsPNG = () => {
  const canvas = document.createElement('canvas');
  canvas.width = gridWidth.value;
  canvas.height = gridHeight.value;
  const ctx = canvas.getContext('2d');
  
  if (!ctx) return;

  for (let row = 0; row < gridHeight.value; row++) {
    for (let col = 0; col < gridWidth.value; col++) {
      ctx.fillStyle = grid.value[row][col];
      ctx.fillRect(col, row, 1, 1);
    }
  }

  canvas.toBlob((blob) => {
    if (!blob) return;
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'pixel-art.png';
    a.click();
    URL.revokeObjectURL(url);
  });
};

// Zoom functions
const zoomIn = () => {
  zoomLevel.value = Math.min(MAX_ZOOM, zoomLevel.value + 0.25);
};

const zoomOut = () => {
  zoomLevel.value = Math.max(MIN_ZOOM, zoomLevel.value - 0.25);
};

const resetZoom = () => {
  zoomLevel.value = 1;
};

const handleWheel = (event: WheelEvent) => {
  event.preventDefault();
  const delta = event.deltaY > 0 ? -0.1 : 0.1;
  zoomLevel.value = Math.max(MIN_ZOOM, Math.min(MAX_ZOOM, zoomLevel.value + delta));
};

onMounted(() => {
  if (canvas.value) {
    ctx.value = canvas.value.getContext('2d');
  }
  initializeGrid();
  document.addEventListener('mouseup', stopDrawing);
  
  // Keyboard shortcuts
  document.addEventListener('keydown', (event) => {
    // Ctrl+Z or Cmd+Z for undo
    if ((event.ctrlKey || event.metaKey) && event.key === 'z' && !event.shiftKey) {
      event.preventDefault();
      undo();
    }
    // Ctrl+Y or Cmd+Y or Ctrl+Shift+Z for redo
    else if ((event.ctrlKey || event.metaKey) && (event.key === 'y' || (event.key === 'z' && event.shiftKey))) {
      event.preventDefault();
      redo();
    }
  });
});
</script>

<template>
  <div class="app">

    <div class="container">
      <!-- Toolbar -->
      <div class="toolbar">
        <div class="tool-section">
          <h3>Tools</h3>
          <div class="tools">
            <button 
              :class="{ active: currentTool === 'pencil' }"
              @click="currentTool = 'pencil'"
              title="Pencil (Draw)">
              <img :src="PencilIcon" alt="Pencil" class="tool-icon" />
              Pencil
            </button>
            <button 
              :class="{ active: currentTool === 'eraser' }"
              @click="currentTool = 'eraser'"
              title="Eraser">
              <img :src="EraserIcon" alt="Eraser" class="tool-icon" />
              Eraser
            </button>
            <button 
              :class="{ active: currentTool === 'fill' }"
              @click="currentTool = 'fill'"
              title="Fill Bucket">
              <img :src="FillIcon" alt="Fill" class="tool-icon" />
              Fill
            </button>
            <button 
              :class="{ active: currentTool === 'eyedropper' }"
              @click="currentTool = 'eyedropper'"
              title="Color Picker">
              <img :src="EyedropperIcon" alt="Picker" class="tool-icon" />
              Picker
            </button>
          </div>
        </div>

        <div class="tool-section">
          <h3>Brush Size</h3>
          <div class="brush-sizes">
            <button 
              v-for="size in [1, 2, 3, 4, 5]" 
              :key="size"
              :class="{ active: brushSize === size }"
              @click="brushSize = size"
              class="brush-size-btn"
            >
              {{ size }}x{{ size }}
            </button>
          </div>
        </div>

        <div class="tool-section">
          <h3>Current Color</h3>
          <div class="color-picker-container">
            <input 
              type="color" 
              v-model="currentColor" 
              class="color-input"
            />
            <span class="color-value">{{ currentColor }}</span>
          </div>
        </div>

        <div class="tool-section">
          <h3>Recent Colors</h3>
          <div class="palette">
            <div 
              v-for="color in colorHistory" 
              :key="color"
              class="palette-color"
              :class="{ selected: currentColor === color }"
              :style="{ backgroundColor: color }"
              @click="currentColor = color"
              :title="color"
            ></div>
          </div>
        </div>

        <div class="tool-section">
          <h3>Actions</h3>
          <div class="actions">
            <button @click="clearGrid">
              <img :src="ClearIcon" alt="Clear" class="tool-icon" />
              Clear
            </button>
            <button @click="resizeGrid">
              <img :src="ResizeIcon" alt="Resize" class="tool-icon" />
              Resize
            </button>
            <button @click="exportAsPNG">
              <img :src="SaveIcon" alt="Export" class="tool-icon" />
              Export PNG
            </button>
          </div>
        </div>

        <div class="tool-section">
          <h3>Zoom</h3>
          <div class="zoom-controls">
            <button @click="zoomOut" :disabled="zoomLevel <= MIN_ZOOM">−</button>
            <span>{{ Math.round(zoomLevel * 100) }}%</span>
            <button @click="zoomIn" :disabled="zoomLevel >= MAX_ZOOM">+</button>
          </div>
          <button @click="resetZoom" class="reset-zoom">Reset Zoom</button>
        </div>
      </div>

      <!-- Canvas -->
      <div class="canvas-container">
        <div class="canvas-wrapper">
          <div class="canvas-zoom-wrapper" :style="{ transform: `scale(${zoomLevel})` }" @wheel="handleWheel">
            <canvas
              ref="canvas"
              class="pixel-canvas"
              :style="{
                cursor: currentTool === 'eyedropper' ? 'crosshair' : 'pointer'
              }"
              @mousedown="onCanvasMouseDown"
              @mousemove="onCanvasMouseMove"
              @mouseup="onCanvasMouseUp"
              @mouseleave="onCanvasMouseLeave"
            ></canvas>
          </div>
        </div>
        <div class="info">
          {{ gridWidth }} × {{ gridHeight }}
        </div>
      </div>
    </div>

    <!-- Custom Dialog -->
    <div v-if="dialogVisible" class="dialog-overlay" @click="cancelDialog">
      <div class="dialog-box" @click.stop>
        <h2 class="dialog-title">{{ dialogTitle }}</h2>
        <p class="dialog-message">{{ dialogMessage }}</p>
        <input 
          v-if="dialogType === 'prompt'" 
          v-model="dialogInput"
          type="text"
          class="dialog-input"
          @keyup.enter="confirmDialog"
          autofocus
        />
        <div class="dialog-buttons">
          <button v-if="dialogType !== 'alert'" @click="cancelDialog" class="dialog-btn dialog-btn-cancel">
            Cancel
          </button>
          <button @click="confirmDialog" class="dialog-btn dialog-btn-confirm">
            {{ dialogType === 'alert' ? 'OK' : 'Confirm' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.app {
  height: 100vh;
  overflow: hidden;
  background: #f6f6f6;
  padding: 5px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
  display: flex;
  flex-direction: column;
}

.container {
  display: flex;
  gap: 5px;
  width: 100%;
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

/* Toolbar */
.toolbar {
  background: white;
  border-radius: 8px;
  padding: 10px;
  min-width: 250px;
  max-width: 280px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  max-height: 100%;
  overflow-y: auto;
  overflow-x: hidden;
}

.app-title {
  font-size: 1.1rem;
  color: #667eea;
  margin: 0;
  padding: 0;
  font-weight: 700;
}

.app-title {
  font-size: 1.1rem;
  color: #667eea;
  margin: 0;
  padding: 0;
  font-weight: 700;
}

.tool-section {
  margin-bottom: 12px;
}

.tool-section:last-child {
  margin-bottom: 0;
}

.tool-section h3 {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 10px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.tools {
  display: grid;
  gap: 8px;
}

.tools button {
  padding: 12px;
  border: 2px solid #e0e0e0;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.95rem;
  transition: all 0.2s;
  text-align: left;
  color: #333;
  display: flex;
  align-items: center;
  gap: 8px;
}

.tool-icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
}

.tools button:hover {
  background: #f5f5f5;
  border-color: #667eea;
}

.tools button.active {
  background: #667eea;
  color: white;
  border-color: #667eea;
  font-weight: 600;
}

/* Color Picker */
.color-picker-container {
  display: flex;
  align-items: center;
  gap: 10px;
}

.color-input {
  width: 60px;
  height: 60px;
  border: 3px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: border-color 0.2s;
}

.color-input:hover {
  border-color: #667eea;
}

.color-value {
  font-family: 'Courier New', monospace;
  font-size: 0.9rem;
  color: #666;
  font-weight: 600;
}

/* Brush Sizes */
.brush-sizes {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 6px;
}

.brush-size-btn {
  padding: 8px;
  border: 2px solid #e0e0e0;
  background: white;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.85rem;
  transition: all 0.2s;
  color: #333;
  font-weight: 600;
}

.brush-size-btn:hover {
  background: #f5f5f5;
  border-color: #667eea;
}

.brush-size-btn.active {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

/* Palette */
.palette {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 6px;
}

.palette-color {
  width: 100%;
  aspect-ratio: 1;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
}

.palette-color:hover {
  transform: scale(1.1);
  border-color: #667eea;
}

.palette-color.selected {
  border: 3px solid #667eea;
  box-shadow: 0 0 0 2px white, 0 0 0 4px #667eea;
}

/* Actions */
.actions {
  display: grid;
  gap: 8px;
}

.actions button {
  padding: 10px;
  border: 2px solid #e0e0e0;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
  color: #333;
  display: flex;
  align-items: center;
  gap: 8px;
}

.actions button:hover {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

/* Zoom Controls */
.zoom-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  margin-bottom: 8px;
}

.zoom-controls button {
  width: 40px;
  height: 40px;
  border: 2px solid #e0e0e0;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1.5rem;
  font-weight: bold;
  transition: all 0.2s;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #333;
}

.zoom-controls button:hover:not(:disabled) {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

.zoom-controls button:disabled {
  opacity: 0.3;
  cursor: not-allowed;
}

.zoom-controls span {
  font-weight: 600;
  color: #666;
  min-width: 50px;
  text-align: center;
}

.reset-zoom {
  width: 100%;
  padding: 10px;
  border: 2px solid #e0e0e0;
  background: white;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
  color: #333;
}

.reset-zoom:hover {
  background: #667eea;
  color: white;
  border-color: #667eea;
}

/* Canvas */
.canvas-container {
  flex: 1;
  background: #f5f5f5;
  border-radius: 8px;
  padding: 5px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-width: 0;
  position: relative;
}

.canvas-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 5px;
  border-radius: 8px;
  overflow: auto;
  flex: 1;
  min-height: 0;
  width: 100%;
}

.canvas-zoom-wrapper {
  transform-origin: center center;
  transition: transform 0.1s ease-out;
  will-change: transform;
}

.pixel-canvas {
  display: block;
  background: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
  image-rendering: -moz-crisp-edges;
  image-rendering: -webkit-crisp-edges;
  image-rendering: pixelated;
  image-rendering: crisp-edges;
  -ms-interpolation-mode: nearest-neighbor;
}

.info {
  position: absolute;
  bottom: 8px;
  right: 12px;
  color: #999;
  font-size: 0.75rem;
  font-weight: 600;
  background: rgba(255, 255, 255, 0.8);
  padding: 4px 8px;
  border-radius: 4px;
  pointer-events: none;
}

/* Responsive */
@media (max-width: 1024px) {
  .container {
    flex-direction: column;
  }

  .toolbar {
    max-width: 100%;
  }
}

/* Dialog Overlay */
.dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(4px);
}

.dialog-box {
  background: white;
  border-radius: 12px;
  padding: 24px;
  min-width: 320px;
  max-width: 500px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: dialogFadeIn 0.2s ease-out;
}

@keyframes dialogFadeIn {
  from {
    opacity: 0;
    transform: scale(0.9) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: scale(1) translateY(0);
  }
}

.dialog-title {
  margin: 0 0 12px 0;
  font-size: 1.25rem;
  color: #333;
  font-weight: 600;
}

.dialog-message {
  margin: 0 0 20px 0;
  color: #666;
  line-height: 1.5;
}

.dialog-input {
  width: 100%;
  padding: 10px 12px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  margin-bottom: 20px;
  transition: border-color 0.2s;
}

.dialog-input:focus {
  outline: none;
  border-color: #667eea;
}

.dialog-buttons {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.dialog-btn {
  padding: 10px 24px;
  border: none;
  border-radius: 8px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.dialog-btn-cancel {
  background: #e0e0e0;
  color: #666;
}

.dialog-btn-cancel:hover {
  background: #d0d0d0;
}

.dialog-btn-confirm {
  background: #667eea;
  color: white;
}

.dialog-btn-confirm:hover {
  background: #5568d3;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}
</style>

<style>
body {
  margin: 0;
}

:root {
  font-family: Inter, Avenir, Helvetica, Arial, sans-serif;
  font-size: 16px;
  line-height: 24px;
  font-weight: 400;

  color: #0f0f0f;

  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  -webkit-text-size-adjust: 100%;
}

.logo {
  height: 6em;
  padding: 1.5em;
  will-change: filter;
  transition: 0.75s;
}

.logo.tauri:hover {
  filter: drop-shadow(0 0 2em #24c8db);
}

.row {
  display: flex;
  justify-content: center;
}

a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}

a:hover {
  color: #535bf2;
}

h1 {
  text-align: center;
}

input,
button {
  border-radius: 8px;
  border: 1px solid transparent;
  padding: 0.6em 1.2em;
  font-size: 1em;
  font-weight: 500;
  font-family: inherit;
  color: #0f0f0f;
  background-color: #ffffff;
  transition: border-color 0.25s;
  box-shadow: 0 2px 2px rgba(0, 0, 0, 0.2);
}

button {
  cursor: pointer;
}

button:hover {
  border-color: #396cd8;
}
button:active {
  border-color: #396cd8;
  background-color: #e8e8e8;
}

input,
button {
  outline: none;
}

#greet-input {
  margin-right: 5px;
}

@media (prefers-color-scheme: dark) {
  :root {
    color: #f6f6f6;
    background-color: #2f2f2f;
  }

  a:hover {
    color: #24c8db;
  }

  input,
  button {
    color: #ffffff;
    background-color: #0f0f0f98;
  }
  button:active {
    background-color: #0f0f0f69;
  }
}

</style>