<template>
  <canvas ref="canvas"></canvas>
</template>

<script>
export default {
  name: 'WarehouseCanvasLOD',
  props: {
    layout: Array,
    path: Array,
    listItem: Array,
    warehouseX: Number,
    warehouseY: Number
  },
  data() {
    return {
      gridSize: 50,
      offsetX: 0,
      offsetY: 0,
      scale: 1,
      isDragging: false,
      startX: 0,
      startY: 0,
      ctx: null,
      miniMapImage: null,
      layoutCells: new Set(),
      pathCells: new Set(),
      itemCells: new Map(),
    };
  },
  mounted() {
    this.ctx = this.$refs.canvas.getContext('2d');
    this.resizeCanvas();
    this.generateWarehouseData();
    this.createMiniMap();
    this.zoomToFit();
    window.addEventListener('resize', this.resizeCanvas);

    const canvas = this.$refs.canvas;
    canvas.addEventListener('mousedown', this.onMouseDown);
    canvas.addEventListener('mousemove', this.onMouseMove);
    canvas.addEventListener('mouseup', this.onMouseUp);
    canvas.addEventListener('mouseleave', this.onMouseUp);
    canvas.addEventListener('wheel', this.onWheel, { passive: false });
  },
  beforeDestroy() {
    const canvas = this.$refs.canvas;
    window.removeEventListener('resize', this.resizeCanvas);

    canvas.removeEventListener('mousedown', this.onMouseDown);
    canvas.removeEventListener('mousemove', this.onMouseMove);
    canvas.removeEventListener('mouseup', this.onMouseUp);
    canvas.removeEventListener('mouseleave', this.onMouseUp);
    canvas.removeEventListener('wheel', this.onWheel);
  },
  methods: {
    resizeCanvas() {
      const canvas = this.$refs.canvas;
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
      this.zoomToFit();
    },
    generateWarehouseData() {
      this.layout.forEach(p => {
        const key = `${p.xpos},${p.ypos}`;
        this.layoutCells.add(key);
      });

      this.path.forEach(pathLine => {
        pathLine.forEach(([x, y]) => {
          this.pathCells.add(`${x},${y}`);
        });
      });

      this.listItem.forEach(item => {
        const key = `${item.xpos},${item.ypos}`;
        this.itemCells.set(key, item);
      });
    },
    createMiniMap() {
      const offscreen = document.createElement('canvas');
      offscreen.width = this.warehouseX;
      offscreen.height = this.warehouseY;
      const ctx = offscreen.getContext('2d');

      for (let x = 0; x < this.warehouseX; x++) {
        for (let y = 0; y < this.warehouseY; y++) {
          const key = `${x},${y}`;
          if (this.layoutCells.has(key)) {
            ctx.fillStyle = 'orange';
          } else if (this.pathCells.has(key)) {
            ctx.fillStyle = 'lightgreen';
          } else {
            ctx.fillStyle = '#f0f0f0';
          }
          ctx.fillRect(x, this.warehouseY - y - 1, 1, 1);
        }
      }

      this.miniMapImage = new Image();
      this.miniMapImage.src = offscreen.toDataURL();
    },
    draw() {
      const canvas = this.$refs.canvas;
      const ctx = this.ctx;
      ctx.clearRect(0, 0, canvas.width, canvas.height);

      if (this.scale < 0.2) return this.drawTileView();

      ctx.save();
      ctx.translate(this.offsetX, this.offsetY);
      ctx.scale(this.scale, this.scale);

      const startCol = Math.floor(-this.offsetX / (this.gridSize * this.scale));
      const startRow = Math.floor(-this.offsetY / (this.gridSize * this.scale));
      const visibleCols = Math.ceil(canvas.width / (this.gridSize * this.scale)) + 2;
      const visibleRows = Math.ceil(canvas.height / (this.gridSize * this.scale)) + 2;

      for (let x = startCol; x < startCol + visibleCols; x++) {
        for (let y = startRow; y < startRow + visibleRows; y++) {
          if (x < 0 || y < 0 || x >= this.warehouseX || y >= this.warehouseY) continue;

          const flipY = this.warehouseY - y - 1;
          const key = `${x},${y}`;
          if (this.layoutCells.has(key)) {
            ctx.fillStyle = 'orange';
            ctx.fillRect(x * this.gridSize, flipY * this.gridSize, this.gridSize, this.gridSize);
          } else if (this.pathCells.has(key)) {
            ctx.fillStyle = 'lightgreen';
            ctx.fillRect(x * this.gridSize, flipY * this.gridSize, this.gridSize, this.gridSize);
          }

          if (this.itemCells.has(key)) {
            ctx.fillStyle = 'blue';
            ctx.beginPath();
            ctx.arc(
              x * this.gridSize + this.gridSize / 2,
              flipY * this.gridSize + this.gridSize / 2,
              this.gridSize / 4,
              0,
              2 * Math.PI
            );
            ctx.fill();
          }

          if (this.scale > 0.3) {
            ctx.strokeStyle = '#ccc';
            ctx.strokeRect(x * this.gridSize, flipY * this.gridSize, this.gridSize, this.gridSize);
          }
        }
      }

      ctx.restore();
    },
    drawTileView() {
      if (!this.miniMapImage) return;

      const ctx = this.ctx;
      const img = this.miniMapImage;

      const scaledWidth = img.width * this.gridSize * this.scale;
      const scaledHeight = img.height * this.gridSize * this.scale;

      ctx.save();
      ctx.translate(this.offsetX, this.offsetY);
      ctx.drawImage(img, 0, 0, scaledWidth, scaledHeight);
      ctx.restore();
    },
    onMouseDown(e) {
      this.isDragging = true;
      this.startX = e.clientX - this.offsetX;
      this.startY = e.clientY - this.offsetY;
    },
    onMouseMove(e) {
      if (this.isDragging) {
        this.offsetX = e.clientX - this.startX;
        this.offsetY = e.clientY - this.startY;
        this.draw();
      }
    },
    onMouseUp() {
      this.isDragging = false;
    },
    onWheel(e) {
      e.preventDefault();
      const scaleAmount = 0.1;
      const mouseX = e.clientX;
      const mouseY = e.clientY;

      const prevScale = this.scale;
      if (e.deltaY < 0) this.scale *= (1 + scaleAmount);
      else this.scale *= (1 - scaleAmount);

      this.offsetX -= (mouseX - this.offsetX) * (this.scale / prevScale - 1);
      this.offsetY -= (mouseY - this.offsetY) * (this.scale / prevScale - 1);

      this.draw();
    },
    zoomToFit() {
      const canvas = this.$refs.canvas;
      const scaleX = canvas.width / (this.gridSize * this.warehouseX);
      const scaleY = canvas.height / (this.gridSize * this.warehouseY);
      this.scale = Math.min(scaleX, scaleY);

      const worldWidth = this.gridSize * this.warehouseX * this.scale;
      const worldHeight = this.gridSize * this.warehouseY * this.scale;

      this.offsetX = (canvas.width - worldWidth) / 2;
      this.offsetY = (canvas.height - worldHeight) / 2;

      this.draw();
    },
    zoomMax() {
      this.scale = 1;
      const canvas = this.$refs.canvas;
      const worldWidth = this.gridSize * this.warehouseX;
      const worldHeight = this.gridSize * this.warehouseY;

      this.offsetX = (canvas.width - worldWidth) / 2;
      this.offsetY = (canvas.height - worldHeight) / 2;
      this.draw();
    }
  }
};
</script>

<style scoped>
canvas {
  display: block;
  width: 100vw;
  height: 100vh;
  background: #f9f9f9;
  cursor: grab;
}
canvas:active {
  cursor: grabbing;
}
</style>
