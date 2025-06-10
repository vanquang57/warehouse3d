<template>
  <div id="demoOne">
    <div class="controls">
      <h2>Warehouse Visualization</h2>
      <div>
        <label>Color by: </label>
        <select v-model="colorBy" @change="updateVisualization">
          <option value="none">None</option>
          <option value="cases_damaged">Cases Damaged</option>
          <option value="weekly_mvmt">Weekly Movement</option>
          <option value="item_cost">Item Cost</option>
        </select>
      </div>
      <div>
        <button @click="resetCamera">Reset View</button>
        <button @click="toggleGrid">{{ showGrid ? 'Hide Grid' : 'Show Grid' }}</button>
      </div>
      <div>
        <div v-if="startSlot">
          <strong>Start:</strong> {{ startSlot.location }}
        </div>
        <div v-if="endSlot">
          <strong>End:</strong> {{ endSlot.location }}
        </div>
        <div v-if="currentPathLength">
          <strong>Distance:</strong> {{ currentPathLength }} steps
        </div>
      </div>
    </div>
    <div id="container" ref="container"></div>
    <div class="tooltip" ref="tooltip"></div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import { Line2 } from 'three/examples/jsm/lines/Line2.js';
import { LineMaterial } from 'three/examples/jsm/lines/LineMaterial.js';
import { LineGeometry } from 'three/examples/jsm/lines/LineGeometry.js';

import * as d3 from 'd3'
import layoutData from './warehouse-layout.json'
import inventoryData from './warehouse-inventory.json'

export default {
  name: 'DemoOne',
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      slots: layoutData,
      colorBy: 'none',
      showGrid: true,
      gridHelper: null,
      raycaster: new THREE.Raycaster(),
      mouse: new THREE.Vector2(),
      intersectedObject: null,
      warehouseCenter: new THREE.Vector3(),
      animationFrameId: null,
      warehouseGroup: null,
      instancesGroup: null,
      warehouseOffset: 0,
      lastRenderTime: 0,
      frameRate: 120,
      frameInterval: 1000 / 30,
      colorCache: {},
      materialCache: {},
      geometryCache: {},
      instanceMeshes: {}, // Store our InstancedMesh objects
      instanceData: [], // Store data for each instance
      gridSize: 24,
      walkableGrid: {},
      startPoint: null,
      endPoint: null,
      clickMarkerMaterial: new THREE.MeshBasicMaterial({ color: 0x00ff00 }),
      startSlot: null,
      endSlot: null,
      currentPathLength: null
    }
  },
  destroyed() {
    this.cleanUp()
  },
  mounted() {
    this.initThreeJS()
    this.calculateWarehouseCenter(layoutData)
    this.createWarehouse(layoutData, inventoryData)
    this.animate()
    window.addEventListener('resize', this.throttle(this.onWindowResize, 100))
    this.$refs.container.addEventListener('mousemove', this.throttle(this.onMouseMove, 50));
    this.testPathfinding();
    this.renderer.autoClear = false;
  },
  beforeDestroy() {
    this.cleanUp()
  },
  methods: {
    throttle(func, limit) {
      let lastFunc;
      let lastRan;
      return function () {
        const context = this;
        const args = arguments;
        if (!lastRan) {
          func.apply(context, args);
          lastRan = Date.now();
        } else {
          clearTimeout(lastFunc);
          lastFunc = setTimeout(function () {
            if ((Date.now() - lastRan) >= limit) {
              func.apply(context, args);
              lastRan = Date.now();
            }
          }, limit - (Date.now() - lastRan));
        }
      };
    },

    calculateWarehouseCenter(layoutData) {
      const positions = layoutData.map(slot => new THREE.Vector3(+slot.X, +slot.Z, +slot.Y))
      const boundingBox = new THREE.Box3().setFromPoints(positions)
      const size = new THREE.Vector3()
      boundingBox.getSize(size)
      this.warehouseOffset = new THREE.Vector3(0, 0, 0)
      this.warehouseCenter = new THREE.Vector3(0, 0, 0)
    },

    initThreeJS() {
      // Create scene
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0xf0f0f0)

      // Create warehouse group
      this.warehouseGroup = new THREE.Group()
      this.scene.add(this.warehouseGroup)

      // Create instances group
      this.instancesGroup = new THREE.Group()
      this.warehouseGroup.add(this.instancesGroup)

      // Create camera with optimized settings
      this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 10000)
      this.camera.position.set(0, 1500, 2000)
      this.camera.lookAt(0, 0, 0)

      // Create renderer with performance options
      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        powerPreference: "high-performance",
        logarithmicDepthBuffer: true
      })
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
      this.renderer.setSize(window.innerWidth, window.innerHeight)
      this.$refs.container.appendChild(this.renderer.domElement)

      // Add optimized controls
      this.controls = new OrbitControls(this.camera, this.renderer.domElement)
      this.controls.enableDamping = true
      this.controls.dampingFactor = 0.25
      this.controls.screenSpacePanning = false
      this.controls.target.copy(this.warehouseCenter)

      // Optimized lighting
      const ambientLight = new THREE.AmbientLight(0xffffff, 0.7)
      this.scene.add(ambientLight)

      const directionalLight = new THREE.DirectionalLight(0xffffff, 1.2)
      directionalLight.position.set(1, 1, 1).normalize()
      this.scene.add(directionalLight)

      // Add grid helper
      this.gridHelper = new THREE.GridHelper(11000, 10, 0x888888, 0x888888)
      this.gridHelper.position.copy(this.warehouseCenter)
      this.scene.add(this.gridHelper)

      // Add axes helper
      this.axesHelper = new THREE.AxesHelper(1000)
      this.axesHelper.position.copy(this.warehouseCenter)
      this.scene.add(this.axesHelper)
      this.axesHelper.setColors(
        new THREE.Color(0xff0000),
        new THREE.Color(0x00ff00),
        new THREE.Color(0x0000ff))
    },

    createWarehouse(layoutData, inventoryData) {
      // Clear existing instances
      this.instancesGroup.clear()
      this.instanceMeshes = {}
      this.instanceData = []

      // Prepare color scales
      const damagedColorScale = d3.scaleLinear()
        .domain([0, 8])
        .range(["#00ff00", "#ff0000"])

      const movementColorScale = d3.scaleSequential(d3.interpolateRdYlGn)
        .domain(d3.extent(inventoryData, d => +d["WEEKLY MVMT"]))

      const costColorScale = d3.scaleSequential(d3.interpolateBlues)
        .domain(d3.extent(inventoryData, d => +d["ITEM COST"]))

      // Create inventory lookup map for faster access
      const inventoryLookup = new Map()
      inventoryData.forEach(item => {
        inventoryLookup.set(item.LOCATION, item)
      })

      // Group slots by dimensions for instancing
      const slotsByDimensions = {}

      layoutData.forEach((slotData, index) => {
        const width = +slotData.WIDTH
        const depth = +slotData.DEPTH
        const height = +slotData.HEIGHT
        const key = `${width}_${height}_${depth}`

        if (!slotsByDimensions[key]) {
          slotsByDimensions[key] = {
            width,
            height,
            depth,
            slots: []
          }
        }

        slotsByDimensions[key].slots.push({
          slotData,
          index
        })
      })

      // Create an InstancedMesh for each unique dimension group
      Object.keys(slotsByDimensions).forEach(key => {
        const group = slotsByDimensions[key]
        const count = group.slots.length

        // Create geometry if not cached
        if (!this.geometryCache[key]) {
          this.geometryCache[key] = new THREE.BoxGeometry(group.width, group.height, group.depth)
        }

        // Create material
        const material = new THREE.MeshPhongMaterial({
          color: 0xeeeeee,
          transparent: true,
          opacity: 0.9
        })

        // Create instanced mesh
        const instancedMesh = new THREE.InstancedMesh(this.geometryCache[key], material, count)
        instancedMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage)
        instancedMesh.count = count
        instancedMesh.userData.key = key

        // Store reference
        this.instanceMeshes[key] = instancedMesh
        this.instancesGroup.add(instancedMesh)

        // Create dummy object for raycasting
        const dummy = new THREE.Object3D()

        // Process each slot in this group
        group.slots.forEach((slot, instanceId) => {
          const slotData = slot.slotData
          const x = +slotData.X
          const y = +slotData.Z
          const z = +slotData.Y

          // Set position and rotation
          dummy.position.set(x, y + group.height / 2, z)
          dummy.updateMatrix()

          // Apply the matrix to the instance
          instancedMesh.setMatrixAt(instanceId, dummy.matrix)

          // Get inventory data
          const inventory = inventoryLookup.get(slotData.LOCATION)
          let color = new THREE.Color(0xeeeeee)

          if (inventory) {
            // Apply color based on selection
            if (this.colorBy === 'cases_damaged' && inventory["CASES DAMAGED"]) {
              const damageValue = +inventory["CASES DAMAGED"]
              color = new THREE.Color(damagedColorScale(damageValue))
            } else if (this.colorBy === 'weekly_mvmt' && inventory["WEEKLY MVMT"]) {
              const mvmtValue = +inventory["WEEKLY MVMT"]
              color = new THREE.Color(movementColorScale(mvmtValue))
            } else if (this.colorBy === 'item_cost' && inventory["ITEM COST"]) {
              const costValue = +inventory["ITEM COST"]
              color = new THREE.Color(costColorScale(costValue))
            } else if (inventory["COLOR_CASES DAMAGED"]) {
              color = new THREE.Color(inventory["COLOR_CASES DAMAGED"])
            }
          }

          // Set color for this instance
          instancedMesh.setColorAt(instanceId, color)

          // Store instance data for raycasting and tooltips
          this.instanceData.push({
            instanceId,
            instancedMesh,
            location: slotData.LOCATION,
            inventory,
            position: new THREE.Vector3(x, y + group.height / 2, z)
          })
        })

        // Update the instance
        instancedMesh.instanceMatrix.needsUpdate = true
        if (instancedMesh.instanceColor) {
          instancedMesh.instanceColor.needsUpdate = true
        }
      })

      // Position warehouse group at center
      this.instancesGroup.position.copy(this.warehouseOffset)
      this.warehouseGroup.position.copy(this.warehouseCenter)
    },

    updateVisualization() {
      // Recreate warehouse with new coloring
      this.createWarehouse(layoutData, inventoryData)
    },

    resetCamera() {
      if (!this.camera || !this.controls) return

      this.camera.position.set(0, 1500, 2000)
      this.camera.lookAt(0, 0, 0)

      if (this.controls) {
        this.controls.target.set(0, 0, 0)
        this.controls.update()
      }
    },

    toggleGrid() {
      this.showGrid = !this.showGrid
      this.gridHelper.visible = this.showGrid
    },

    onMouseMove(event) {
      if (!this.$refs.container) return

      this.mouse.x = (event.clientX / this.$refs.container.clientWidth) * 2 - 1
      this.mouse.y = -(event.clientY / this.$refs.container.clientHeight) * 2 + 1

      this.raycaster.setFromCamera(this.mouse, this.camera)

      // Find intersected instances
      const intersects = this.raycaster.intersectObjects(this.instancesGroup.children)

      const tooltip = this.$refs.tooltip
      if (intersects.length > 0) {
        const intersect = intersects[0]
        const instancedMesh = intersect.object
        const instanceId = intersect.instanceId

        // Find the instance data
        const instanceInfo = this.instanceData.find(data =>
          data.instancedMesh === instancedMesh && data.instanceId === instanceId
        )

        if (instanceInfo && this.intersectedObject !== instanceInfo) {
          this.intersectedObject = instanceInfo

          if (instanceInfo.inventory) {
            const inv = instanceInfo.inventory
            tooltip.innerHTML = `
              <strong>${instanceInfo.location}</strong><br>
              Item: ${inv["ITEM DESCRIPTION"]}<br>
              Cases Damaged: ${inv["CASES DAMAGED"]}<br>
              Weekly Movement: ${inv["WEEKLY MVMT"]}<br>
              Item Cost: $${inv["ITEM COST"]}
            `
          } else {
            tooltip.innerHTML = `
              <strong>${instanceInfo.location}</strong><br>
              Empty slot
            `
          }
        }

        tooltip.style.display = 'block'
        tooltip.style.left = `${event.clientX + 10}px`
        tooltip.style.top = `${event.clientY + 10}px`
      } else {
        tooltip.style.display = 'none'
        this.intersectedObject = null
      }
    },

    onWindowResize() {
      if (!this.camera || !this.renderer) return

      this.camera.aspect = window.innerWidth / window.innerHeight
      this.camera.updateProjectionMatrix()
      this.renderer.setSize(window.innerWidth, window.innerHeight)
    },

    animate() {
      this.animationFrameId = requestAnimationFrame(this.animate)

      const now = performance.now()
      const delta = now - this.lastRenderTime

      if (delta >= this.frameInterval) {
        this.controls.update()
        this.renderer.render(this.scene, this.camera)
        this.lastRenderTime = now - (delta % this.frameInterval)
      }
    },

    cleanUp() {
      cancelAnimationFrame(this.animationFrameId)
      window.removeEventListener('resize', this.throttle(this.onWindowResize, 100))

      if (this.$refs.container) {
        this.$refs.container.removeEventListener('mousemove', this.throttle(this.onMouseMove, 50))
      }

      // Dispose of all Three.js objects
      Object.values(this.instanceMeshes).forEach(mesh => {
        this.instancesGroup.remove(mesh)
        mesh.dispose()
      })

      if (this.gridHelper) {
        this.scene.remove(this.gridHelper)
        this.gridHelper.dispose()
      }

      if (this.renderer) {
        this.renderer.dispose()
      }

      if (this.axesHelper) {
        this.scene.remove(this.axesHelper)
        this.axesHelper.dispose()
      }

      if (this.combinedWalls) {
        this.combinedWalls.forEach(wall => {
          this.warehouseGroup.remove(wall)
          wall.geometry.dispose()
          wall.material.dispose()
        })
      }

      // Clear caches
      this.colorCache = {}
      this.materialCache = {}
      this.geometryCache = {}
    },

    buildWalkableGrid(buffer = 0) {
      const blocked = new Set();
      for (const slot of this.slots) {
        const x0 = slot.X - slot.WIDTH / 2 - buffer;
        const x1 = slot.X + slot.WIDTH / 2 + buffer;
        const y0 = slot.Y - slot.DEPTH / 2 - buffer;
        const y1 = slot.Y + slot.DEPTH / 2 + buffer;

        const gx0 = Math.floor(x0 / this.gridSize);
        const gx1 = Math.floor(x1 / this.gridSize);
        const gy0 = Math.floor(y0 / this.gridSize);
        const gy1 = Math.floor(y1 / this.gridSize);

        for (let gx = gx0; gx <= gx1; gx++) {
          for (let gy = gy0; gy <= gy1; gy++) {
            blocked.add(`${gx},${gy}`);
          }
        }
      }

      this.walkableGrid = {};
      const xs = this.slots.map(s => s.X);
      const ys = this.slots.map(s => s.Y);
      const maxGx = Math.ceil(Math.max(...xs) / this.gridSize);
      const maxGy = Math.ceil(Math.max(...ys) / this.gridSize);

      for (let gx = 0; gx <= maxGx; gx++) {
        for (let gy = 0; gy <= maxGy; gy++) {
          const key = `${gx},${gy}`;
          this.walkableGrid[key] = !blocked.has(key);
        }
      }
    },

    findNearestWalkable(pos, maxRadius = 10) {
      let closest = null;
      let minDist = Infinity;

      for (let dx = -maxRadius; dx <= maxRadius; dx++) {
        for (let dy = -maxRadius; dy <= maxRadius; dy++) {
          const nx = pos.x + dx;
          const ny = pos.y + dy;
          const key = `${nx},${ny}`;
          if (this.walkableGrid[key]) {
            const dist = Math.abs(dx) + Math.abs(dy); // dùng Manhattan cho nhanh
            if (dist < minDist) {
              minDist = dist;
              closest = { x: nx, y: ny };
            }
          }
        }
      }
      return closest;
    },

    findPathDijkstra(start, end) {
      const queue = [{ pos: start, dist: 0, path: [start] }];
      const visited = new Set();
      const key = ({ x, y }) => `${x},${y}`;

      while (queue.length > 0) {
        queue.sort((a, b) => a.dist - b.dist);
        const { pos, dist, path } = queue.shift();
        const k = key(pos);
        if (visited.has(k)) continue;
        visited.add(k);
        if (k === key(end)) return path;

        const directions = [
          { x: 1, y: 0 }, { x: -1, y: 0 },
          { x: 0, y: 1 }, { x: 0, y: -1 }
        ];

        for (const dir of directions) {
          const nx = pos.x + dir.x;
          const ny = pos.y + dir.y;
          const nkey = `${nx},${ny}`;
          if (this.walkableGrid[nkey] && !visited.has(nkey)) {
            queue.push({ pos: { x: nx, y: ny }, dist: dist + 1, path: [...path, { x: nx, y: ny }] });
          }
        }
      }
      return null;
    },

    gridToWorld({ x, y }) {
      return new THREE.Vector3(x * this.gridSize, 0, y * this.gridSize);
    },

    worldToGrid(vector) {
      return {
        x: Math.floor(vector.x / this.gridSize),
        y: Math.floor(vector.z / this.gridSize),
      };
    },

drawPath(path) {
  const positions = path.flatMap(p => {
    const v = this.gridToWorld(p);
    return [v.x, v.y, v.z];
  });
  const geometry = new LineGeometry();
  geometry.setPositions(positions);
  const material = new LineMaterial({
    color: 0xff0000,
    linewidth: 5, // đơn vị: pixel (trên màn hình)
    worldUnits: false, // nếu muốn dùng đơn vị thực thì set true
    dashed: false
  });

  const line = new Line2(geometry, material);
  line.computeLineDistances();
  line.scale.set(1, 1, 1);
  this.scene.add(line);
}
,

    addClickMarker(pos) {
      const geometry = new THREE.SphereGeometry(10, 8, 8);
      const mesh = new THREE.Mesh(geometry, this.clickMarkerMaterial);
      mesh.position.copy(this.gridToWorld(pos));
      this.scene.add(mesh);
    },

    onSceneClick(event) {
      const rect = this.$refs.container.getBoundingClientRect();
      const mouse = new THREE.Vector2(
        ((event.clientX - rect.left) / rect.width) * 2 - 1,
        -((event.clientY - rect.top) / rect.height) * 2 + 1
      );
      this.raycaster.setFromCamera(mouse, this.camera);
      const intersects = this.raycaster.intersectObjects(this.instancesGroup.children, true);

      if (intersects.length > 0) {
        const point = intersects[0].point;
        const gridPos = this.worldToGrid(point);
        const intersect = intersects[0];
        if (!intersect || intersect.instanceId === undefined) return;
        const slot = this.instanceData.find(s =>
          s.instancedMesh === intersect.object &&
          s.instanceId === intersect.instanceId
        );
        if (!slot) return;
        if (!this.startPoint && slot?.inventory) {
          this.startPoint = this.findNearestWalkable(gridPos) || gridPos;
          if (slot.instancedMesh && slot.instanceId !== undefined) {

            this.startSlot = slot;
            this.endSlot = null;
            this.currentPathLength = null;

            this.addClickMarker(gridPos);
            slot.instancedMesh.setColorAt(slot.instanceId, new THREE.Color(0xffff00));
            slot.instancedMesh.instanceColor.needsUpdate = true;
          }
        } else if (!this.endPoint && this.startPoint) {
          this.endPoint = this.findNearestWalkable(gridPos) || gridPos;

          this.endSlot = slot;

          this.addClickMarker(gridPos);
          slot.instancedMesh.setColorAt(slot.instanceId, new THREE.Color(0x0055ff));
          slot.instancedMesh.instanceColor.needsUpdate = true;
          if (this.startPoint && this.endPoint) {
            const path = this.findPathDijkstra(this.startPoint, this.endPoint);
            if (path) {
              this.drawPath(path);
              this.currentPathLength = path.length;
            }
          }
        } else {
          this.startPoint = null;
          this.endPoint = null;

          this.startSlot = null;
          this.endSlot = null;
          this.currentPathLength = null;
        }
      }

    },

    setupClickListener() {
      this.groundPlane = new THREE.Mesh(
        new THREE.PlaneGeometry(10000, 10000),
        new THREE.MeshBasicMaterial({ visible: false })
      );
      this.groundPlane.rotation.x = -Math.PI / 2;
      this.scene.add(this.groundPlane);
      this.$refs.container.addEventListener('click', this.onSceneClick);
    },

    testPathfinding() {
      this.buildWalkableGrid();
      this.visualizeWalkableGrid();
      this.visualizeWalkableGridBorder();
      this.setupClickListener();
    },

    visualizeWalkableGrid() {
      const geometry = new THREE.PlaneGeometry(this.gridSize, this.gridSize);
      const material = new THREE.MeshBasicMaterial({
        color: 0x00ffcc,
        opacity: 0.4,
        transparent: true
      });

      const keys = Object.keys(this.walkableGrid).filter(key => this.walkableGrid[key]);
      const count = keys.length;
      const instancedMesh = new THREE.InstancedMesh(geometry, material, count);
      this.walkableGridMesh = instancedMesh; // <-- đặt dòng này ở đây

      const dummy = new THREE.Object3D();
      let index = 0;
      for (const key of keys) {
        const [gx, gy] = key.split(',').map(Number);
        dummy.position.set(gx * this.gridSize, 0.1, gy * this.gridSize); // 0.1 tránh z-fighting
        dummy.rotation.set(-Math.PI / 2, 0, 0);
        dummy.updateMatrix();
        instancedMesh.setMatrixAt(index++, dummy.matrix);
      }
      this.scene.add(instancedMesh);
    },
    visualizeWalkableGridBorder() {
      const half = this.gridSize / 2;
      const y = 0.5; // nâng lưới lên khỏi sàn

      const positions = [];

      const xs = this.slots.map(s => s.X);
      const ys = this.slots.map(s => s.Y);
      const maxX = Math.ceil(Math.max(...xs)) / this.gridSize;
      const maxY = Math.ceil(Math.max(...ys)) / this.gridSize;
      
      for (let gx = 0; gx < maxX; gx++) {
        for (let gy = 0; gy < maxY; gy++) {
          const x = gx * this.gridSize;
          const z = gy * this.gridSize;

          // 4 cạnh của ô
          positions.push(
            x - half, y, z - half, x + half, y, z - half,  // cạnh trên
            x + half, y, z - half, x + half, y, z + half,  // cạnh phải
            x + half, y, z + half, x - half, y, z + half,  // cạnh dưới
            x - half, y, z + half, x - half, y, z - half   // cạnh trái
          );
        }
      }

      const geometry = new THREE.BufferGeometry();
      geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));

      const material = new THREE.LineBasicMaterial({ color: 0xbcbcbc });
      const gridLines = new THREE.LineSegments(geometry, material);
      this.scene.add(gridLines);
    }
    ,
    getSlotByPosition(worldPos) {
      const tolerance = 36;

      return this.instanceData.find(slot => {
        const dx = Math.abs(slot.position.x - worldPos.x);
        const dy = Math.abs(slot.position.y - worldPos.y);
        const dz = Math.abs(slot.position.z - worldPos.z);
        return dx <= tolerance && dy <= tolerance && dz <= tolerance;
      });
    }
  }
}
</script>

<style>
body {
  margin: 0;
  overflow: hidden;
  touch-action: none;
}

#demoOne {
  width: 100vw;
  height: 100vh;
}

#container {
  width: 100%;
  height: 100%;
  outline: none;
}

.controls {
  position: absolute;
  top: 10px;
  left: 10px;
  background: rgba(255, 255, 255, 0.8);
  padding: 10px;
  border-radius: 5px;
  z-index: 100;
  user-select: none;
}

.controls select,
.controls button {
  padding: 5px;
  margin: 2px;
}

.tooltip {
  position: absolute;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 5px 10px;
  border-radius: 3px;
  pointer-events: none;
  display: none;
  font-size: 14px;
  max-width: 300px;
}
</style>