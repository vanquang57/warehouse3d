<template>
  <!-- Phần template giữ nguyên -->
  <div class="warehouse-grid-container" ref="container">
    <div class="controls">
      <button @click="zoomIn">Zoom In (+)</button>
      <button @click="zoomOut">Zoom Out (-)</button>
      <button @click="resetView">Reset View</button>
      <button @click="toggleFullscreen">Fullscreen</button>
      <span>Zoom: {{ (zoomLevel * 100).toFixed(0) }}%</span>
    </div>
    <div 
      class="canvas-wrapper"
      @mousedown="startPan"
      @mousemove="pan"
      @mouseup="endPan"
      @mouseleave="endPan"
      @wheel.prevent="handleWheel"
    >
      <canvas ref="canvas"></canvas>
    </div>
  </div>
</template>

<script>
export default {
  name: 'WarehouseGrid',
  props: {
    layoutData: {
      type: Array,
      default: () => []
    },
    pathData: {
      type: Array,
      default: () => []
    },
    warehouseX: {
      type: Number,
      default: 100
    },
    warehouseY: {
      type: Number,
      default: 100
    }
  },
  data() {
    return {
      // Các data khác giữ nguyên
      canvas: null,
      ctx: null,
      container: null,
      cellSize: 50,
      zoomLevel: 1,
      minZoom: 0.02,
      maxZoom: 5,
      offsetX: 0,
      offsetY: 0,
      isPanning: false,
      lastPanX: 0,
      lastPanY: 0,
      animationFrame: null,
      needsRedraw: true,
      panTimeout: null,
      lastRedrawTime: 0,
      isFullscreen: false
    };
  },
    mounted() {
    this.canvas = this.$refs.canvas;
    this.ctx = this.canvas.getContext('2d');
    this.container = this.$refs.container;
    
    this.resizeCanvas();
    window.addEventListener('resize', this.resizeCanvas);
    document.addEventListener('fullscreenchange', this.handleFullscreenChange);
    
    // Fit to viewport on initial load
    this.$nextTick(() => {
      this.fitToViewport();
    });
    
    this.render();
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resizeCanvas);
    document.removeEventListener('fullscreenchange', this.handleFullscreenChange);
    if (this.animationFrame) {
      cancelAnimationFrame(this.animationFrame);
    }
    if (this.panTimeout) {
      clearTimeout(this.panTimeout);
    }
  },
  computed: {
    // Thêm computed property để chuyển đổi tọa độ Y
    convertedLayoutData() {
      return this.layoutData.map(item => ({
        ...item,
        y: this.warehouseY - item.y + 1 // Chuyển từ hệ dưới trái sang trên trái
      }));
    },
    convertedPathData() {
      return this.pathData.map(path => 
        path.map(([x, y]) => [x, this.warehouseY - y + 1])
      );
    },
    flatPathCells() {
      const cells = new Set();
      this.convertedPathData.forEach(path => {
        path.forEach(([x, y]) => {
          cells.add(`${x},${y}`);
        });
      });
      return cells;
    },
    // Các computed properties khác giữ nguyên
    scaledCellSize() {
      return this.getEffectiveCellSize();
    },
    visibleGridStartX() {
      return Math.max(0, Math.floor(-this.offsetX / this.scaledCellSize));
    },
    visibleGridEndX() {
      return Math.min(
        this.warehouseX,
        Math.ceil((this.canvas.width - this.offsetX) / this.scaledCellSize)
      );
    },
    visibleGridStartY() {
      return Math.max(0, Math.floor(-this.offsetY / this.scaledCellSize));
    },
    visibleGridEndY() {
      return Math.min(
        this.warehouseY,
        Math.ceil((this.canvas.height - this.offsetY) / this.scaledCellSize)
      );
    },
    needsGridGrouping() {
      const warehouseLarge = this.warehouseX > 300 || this.warehouseY > 300;
      const zoomedOut = this.scaledCellSize < 10;
      return warehouseLarge && zoomedOut;
    },
    needsShelvingSimplification() {
      const warehouseLarge = this.warehouseX > 300 || this.warehouseY > 300;
      const zoomedOut = this.scaledCellSize < 5;
      return warehouseLarge && zoomedOut;
    }
  },
  methods: {

     fitToViewport() {
      if (!this.canvas || !this.container) return;
      
      const containerWidth = this.container.clientWidth;
      const containerHeight = this.container.clientHeight;
      
      // Calculate required zoom to fit warehouse
      const zoomX = containerWidth / (this.warehouseX * this.cellSize);
      const zoomY = containerHeight / (this.warehouseY * this.cellSize);
      
      this.zoomLevel = Math.min(zoomX, zoomY, 1); // Don't zoom in more than 100%
      this.offsetX = (containerWidth - this.warehouseX * this.scaledCellSize) / 2;
      this.offsetY = (containerHeight - this.warehouseY * this.scaledCellSize) / 2;
      
      this.needsRedraw = true;
    },
    toggleFullscreen() {
      if (!document.fullscreenElement) {
        this.container.requestFullscreen().catch(err => {
          console.error(`Error attempting to enable fullscreen: ${err.message}`);
        });
      } else {
        document.exitFullscreen();
      }
    },
    handleFullscreenChange() {
      this.isFullscreen = !!document.fullscreenElement;
      this.resizeCanvas();
      if (!this.isFullscreen) {
        this.fitToViewport();
      }
    },
    getEffectiveCellSize() {
     return this.cellSize * this.zoomLevel;
      return Math.max(2, baseSize); // Minimum 2px to prevent rendering issues
    },
    getGridStep() {
      if (!this.needsGridGrouping) return 1;

      if (this.scaledCellSize < 2) return 5;
      if (this.scaledCellSize < 3) return 4;
      if (this.scaledCellSize < 5) return 3;
      if (this.scaledCellSize < 10) return 2;
      if (this.scaledCellSize < 20) return 1;
      return 1;
    },
    resizeCanvas() {
      if (!this.container) return;
      
      const width = this.container.clientWidth;
      const height = this.container.clientHeight;
      
      this.canvas.width = width;
      this.canvas.height = height;
      this.needsRedraw = true;
    },
    render() {
      if (this.needsRedraw) {
        this.drawGrid();
        this.needsRedraw = false;
      }
      this.animationFrame = requestAnimationFrame(this.render);
    },
    drawGrid() {
      if (!this.ctx) return;
      
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
      
      const cellSize = this.scaledCellSize;
      const gridStep = this.getGridStep();
      const startX = this.visibleGridStartX;
      const endX = this.visibleGridEndX;
      const startY = this.visibleGridStartY;
      const endY = this.visibleGridEndY;
      
      // Draw path cells first
      this.drawPathCells(startX, endX, startY, endY, cellSize);
      
      // Draw shelving units
      if (this.needsShelvingSimplification) {
        this.drawSimplifiedShelving(startX, endX, startY, endY, cellSize);
      } else {
        this.drawDetailedShelving(startX, endX, startY, endY, cellSize);
      }
      
      // Draw grid lines
      this.drawGridLines(startX, endX, startY, endY, cellSize, gridStep);
      
      // Draw border and axis labels
      this.drawBorderAndLabels(startX, endX, startY, endY, cellSize);
      
      // Draw "No" in top-left corner
      this.drawNoLabel(cellSize);
    },
    drawNoLabel(cellSize) {
      if (this.visibleGridStartX <= 0 && this.visibleGridStartY <= 0) {
        this.ctx.fillStyle = '#ffff00';
        this.ctx.fillRect(this.offsetX, this.offsetY, cellSize, cellSize);
        
        this.ctx.fillStyle = '#000';
        this.ctx.font = `${Math.max(13, Math.min(14, cellSize / 3))}px Arial`;
        this.ctx.textAlign = 'center';
        this.ctx.textBaseline = 'middle';
        this.ctx.fillText(
          'No',
          this.offsetX + cellSize / 2,
          this.offsetY + cellSize / 2
        );
      }
    },
    // Cập nhật các phương thức vẽ để sử dụng dữ liệu đã chuyển đổi
    drawPathCells(startX, endX, startY, endY, cellSize) {
      this.ctx.fillStyle = 'rgba(0, 255, 0, 0.3)';
      
      for (let x = startX; x <= endX; x++) {
        for (let y = startY; y <= endY; y++) {
          if (this.flatPathCells.has(`${x},${y}`)) {
            this.ctx.fillRect(
              this.offsetX + x * cellSize,
              this.offsetY + y * cellSize,
              cellSize,
              cellSize
            );
          }
        }
      }
    },
    drawBorderAndLabels(startX, endX, startY, endY, cellSize) {
      // Draw border
      this.ctx.strokeStyle = '#a0a0a0';
      this.ctx.lineWidth = 2;
      this.ctx.strokeRect(
        this.offsetX,
        this.offsetY,
        this.warehouseX * cellSize,
        this.warehouseY * cellSize
      );
      
      // Draw axis labels
      this.drawAxisLabels(startX, endX, startY, endY, cellSize);
    },
    drawGridLines(startX, endX, startY, endY, cellSize, gridStep) {
      this.ctx.strokeStyle = '#e0e0e0';
      this.ctx.lineWidth = 1;
      
      // Vertical lines
      for (let x = startX; x <= endX; x += gridStep) {
        const screenX = this.offsetX + x * cellSize;
        this.ctx.beginPath();
        this.ctx.moveTo(screenX, this.offsetY + startY * cellSize);
        this.ctx.lineTo(screenX, this.offsetY + endY * cellSize);
        this.ctx.stroke();
      }
      
      // Horizontal lines
      for (let y = startY; y <= endY; y += gridStep) {
        const screenY = this.offsetY + y * cellSize;
        this.ctx.beginPath();
        this.ctx.moveTo(this.offsetX + startX * cellSize, screenY);
        this.ctx.lineTo(this.offsetX + endX * cellSize, screenY);
        this.ctx.stroke();
      }
    },
    drawDetailedShelving(startX, endX, startY, endY, cellSize) {
      this.ctx.fillStyle = 'rgba(255, 165, 0, 0.5)';
      
      this.convertedLayoutData.forEach(item => {
        const x = item.x;
        const y = item.y;
        
        if (x >= startX && x <= endX && y >= startY && y <= endY) {
          this.ctx.fillRect(
            this.offsetX + x * cellSize,
            this.offsetY + y * cellSize,
            cellSize,
            cellSize
          );
          
          if (cellSize > 15 && item.aisle) {
            this.ctx.fillStyle = '#000';
            this.ctx.font = `${Math.max(8, Math.min(12, cellSize / 3))}px Arial`;
            this.ctx.textAlign = 'center';
            this.ctx.textBaseline = 'middle';
            this.ctx.fillText(
              item.aisle,
              this.offsetX + x * cellSize + cellSize / 2,
              this.offsetY + y * cellSize + cellSize / 2
            );
            this.ctx.fillStyle = 'rgba(255, 165, 0, 0.5)';
          }
        }
      });
    },
    drawSimplifiedShelving(startX, endX, startY, endY, cellSize) {
      const clusterSize = 4;
      const clusters = new Map();
      
      this.convertedLayoutData.forEach(item => {
        const x = item.x;
        const y = item.y;
        
        if (x >= startX && x <= endX && y >= startY && y <= endY) {
          const clusterX = Math.floor(x / clusterSize) * clusterSize;
          const clusterY = Math.floor(y / clusterSize) * clusterSize;
          const key = `${clusterX},${clusterY}`;
          
          clusters.set(key, { x: clusterX, y: clusterY });
        }
      });
      this.ctx.fillStyle = 'rgba(255, 165, 0, 0.8)';
      this.ctx.lineWidth = 1;
      
      clusters.forEach(cluster => {
        this.ctx.fillRect(
          this.offsetX + cluster.x * cellSize,
          this.offsetY + cluster.y * cellSize,
          clusterSize * cellSize,
          clusterSize * cellSize
        );
      });
    },
    drawAxisLabels(startX, endX, startY, endY, cellSize) {
      const showLabels = cellSize > 8;
      // Top axis (y = 0) - vẫn giữ nguyên vì đã chuyển đổi tọa độ
      if (startY <= 0 && endY >= 0) {
        this.ctx.fillStyle = '#ffff00';
        for (let x = startX; x <= endX; x++) {
          const screenX = this.offsetX + x * cellSize;
          this.ctx.fillRect(screenX, this.offsetY, cellSize, cellSize);
          
          if (showLabels) {
            this.ctx.fillStyle = '#000';
            this.ctx.font = `${Math.max(8, Math.min(12, cellSize / 3))}px Arial`;
            this.ctx.textAlign = 'center';
            this.ctx.textBaseline = 'middle';
            this.ctx.fillText(
              (x + 1).toString(), // Hiển thị số thứ tự từ 1
              screenX + cellSize / 2,
              this.offsetY + cellSize / 2
            );
            this.ctx.fillStyle = '#ffff00';
          }
        }
      }
      
      // Left axis (x = 0) - hiển thị số thứ tự từ dưới lên
      if (startX <= 0 && endX >= 0) {
        this.ctx.fillStyle = '#ffff00';
        for (let y = startY; y <= endY; y++) {
          const screenY = this.offsetY + y * cellSize;
          this.ctx.fillRect(this.offsetX, screenY, cellSize, cellSize);
          
          if (showLabels) {
            this.ctx.fillStyle = '#000';
            this.ctx.font = `${Math.max(8, Math.min(12, cellSize / 3))}px Arial`;
            this.ctx.textAlign = 'center';
            this.ctx.textBaseline = 'middle';
            this.ctx.fillText(
              (this.warehouseY - y).toString(), // Hiển thị số thứ tự từ dưới lên
              this.offsetX + cellSize / 2,
              screenY + cellSize / 2
            );
            this.ctx.fillStyle = '#ffff00';
          }
        }
      }
    },
    // Các phương thức khác giữ nguyên
    // ...
    startPan(e) {
      this.isPanning = true;
      this.lastPanX = e.clientX;
      this.lastPanY = e.clientY;
      this.container.querySelector('.canvas-wrapper').style.cursor = 'grabbing';
    },
    pan(e) {
      if (!this.isPanning) return;
      
      const dx = e.clientX - this.lastPanX;
      const dy = e.clientY - this.lastPanY;
      
      this.offsetX += dx;
      this.offsetY += dy;
      
      this.lastPanX = e.clientX;
      this.lastPanY = e.clientY;
      
      // Throttle redraws during pan
      const now = performance.now();
      if (now - this.lastRedrawTime > 16) { // ~60fps
        this.needsRedraw = true;
        this.lastRedrawTime = now;
      } else {
        clearTimeout(this.panTimeout);
        this.panTimeout = setTimeout(() => {
          this.needsRedraw = true;
        }, 16);
      }
    },
    endPan() {
      this.isPanning = false;
      this.container.querySelector('.canvas-wrapper').style.cursor = 'grab';
    },
    handleWheel(e) {
      const zoomIntensity = 0.1;
      const mouseX = e.clientX - this.container.getBoundingClientRect().left;
      const mouseY = e.clientY - this.container.getBoundingClientRect().top;
      
      const gridX = (mouseX - this.offsetX) / this.scaledCellSize;
      const gridY = (mouseY - this.offsetY) / this.scaledCellSize;
      
      const delta = e.deltaY < 0 ? 1 + zoomIntensity : 1 - zoomIntensity;
      const newZoom = Math.min(this.maxZoom, Math.max(this.minZoom, this.zoomLevel * delta));
      
      if (newZoom !== this.zoomLevel) {
        this.zoomLevel = newZoom;
        this.offsetX = mouseX - gridX * this.scaledCellSize;
        this.offsetY = mouseY - gridY * this.scaledCellSize;
        this.needsRedraw = true;
      }
    },
    zoomIn() {
      this.zoomTo(this.zoomLevel * 1.2);
    },
    zoomOut() {
      this.zoomTo(this.zoomLevel / 1.2);
    },
    zoomTo(newZoom) {
      const containerRect = this.container.getBoundingClientRect();
      const centerX = containerRect.width / 2;
      const centerY = containerRect.height / 2;
      
      const gridX = (centerX - this.offsetX) / this.scaledCellSize;
      const gridY = (centerY - this.offsetY) / this.scaledCellSize;
      
      newZoom = Math.min(this.maxZoom, Math.max(this.minZoom, newZoom));
      
      if (newZoom !== this.zoomLevel) {
        this.zoomLevel = newZoom;
        this.offsetX = centerX - gridX * this.scaledCellSize;
        this.offsetY = centerY - gridY * this.scaledCellSize;
        this.needsRedraw = true;
      }
    },
    resetView() {
      this.fitToViewport();
    }
  }
};
</script>

<style scoped>
/* Phần style giữ nguyên */
.warehouse-grid-container {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100%;
}

.controls {
  padding: 10px;
  background: #f0f0f0;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.controls button {
  padding: 5px 10px;
  cursor: pointer;
  min-width: 80px;
}

.canvas-wrapper {
  flex: 1;
  overflow: hidden;
  position: relative;
  cursor: grab;
  background: #f8f8f8;
  will-change: transform;
}

.canvas-wrapper:active {
  cursor: grabbing;
}

canvas {
  display: block;
  background: white;
  image-rendering: -moz-crisp-edges;
  image-rendering: -webkit-optimize-contrast;
  image-rendering: pixelated;
  image-rendering: optimize-contrast;
}
</style>