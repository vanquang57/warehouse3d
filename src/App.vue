<template>
  <div id="app">
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
    </div>
    <div id="container" ref="container"></div>
    <div class="tooltip" ref="tooltip"></div>
  </div>
</template>

<script>
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'
import * as d3 from 'd3'
import layoutData from './warehouse-layout.json'
import inventoryData from './warehouse-inventory.json'

export default {
  name: 'App',
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      slots: [],
      colorBy: 'none',
      showGrid: true,
      gridHelper: null,
      raycaster: new THREE.Raycaster(),
      mouse: new THREE.Vector2(),
      intersectedObject: null,
      warehouseCenter: new THREE.Vector3(),
      animationFrameId: null,
      warehouseGroup: null,
      slotsGroup: null,
      warehouseOffset: null,
      // Performance optimization variables
      lastRenderTime: 0,
      frameRate: 30,
      frameInterval: 1000 / 30,
      colorCache: {},
      materialCache: {},
      geometryCache: {},
      areaConfigs: [
        {
          name: 'A',
          color: 0xff9999,  // Màu đỏ nhạt
          position: { x: 1100, z: 1950 }, // Vị trí trung tâm
          size: { width: 3000, depth: 4500 }, // Kích thước
          rotation: 0 // Góc xoay (radian)
        },
        {
          name: 'B',
          color: 0x99ff99,  // Màu xanh lá nhạt
          position: { x: 4200, z: 3100 },
          size: { width: 3000, depth: 2200 },
          rotation: 0
        },
        {
          name: 'Staging',
          color: 0x9999ff,  // Màu xanh dương nhạt
          position: { x: 4200, z: 800 },
          size: { width: 3000, depth: 2200 },
          rotation: 0 // Xoay 45 độ
        }
      ],
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
    this.$refs.container.addEventListener('mousemove', this.throttle(this.onMouseMove, 50))
  },
  beforeDestroy() {
    this.cleanUp()
  },
  methods: {
    // Throttle function to limit how often a function can be called
    throttle(func, limit) {
      let lastFunc;
      let lastRan;
      return function() {
        const context = this;
        const args = arguments;
        if (!lastRan) {
          func.apply(context, args);
          lastRan = Date.now();
        } else {
          clearTimeout(lastFunc);
          lastFunc = setTimeout(function() {
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
      this.warehouseOffset = new THREE.Vector3(
        -boundingBox.min.x - size.x/2,
        -boundingBox.min.y,
        -boundingBox.min.z - size.z/2
      )
      
      this.warehouseCenter = new THREE.Vector3(0, size.y/2, 0)
    },
    
    initThreeJS() {
      // Create scene
      this.scene = new THREE.Scene()
      this.scene.background = new THREE.Color(0xf0f0f0)

      // Create warehouse group
      this.warehouseGroup = new THREE.Group()
      this.scene.add(this.warehouseGroup)

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
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)) // Limit pixel ratio for performance
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

      // Thêm trục tọa độ
      this.axesHelper = new THREE.AxesHelper(1000) // Kích thước 1000 đơn vị
      this.axesHelper.position.copy(this.warehouseCenter)
      this.scene.add(this.axesHelper)

      // Tùy chỉnh màu sắc các trục nếu cần
      this.axesHelper.setColors(
        new THREE.Color(0xff0000), // Trục X - Đỏ
        new THREE.Color(0x00ff00), // Trục Y - Xanh lá
        new THREE.Color(0x0000ff)  // Trục Z - Xanh dương
      )
    },
    
    createWarehouse(layoutData, inventoryData) {
      // Clear existing slots efficiently
      if (this.slotsGroup) {
        this.warehouseGroup.remove(this.slotsGroup)
        this.disposeGroup(this.slotsGroup)
      }
      this.slots = []
      
      // Create new slots group
      this.slotsGroup = new THREE.Group()
      this.slotsGroup.position.copy(this.warehouseOffset)
      
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
      

      
      const edgeMaterial = new THREE.LineBasicMaterial({ 
        color: 0x000000,
        linewidth: 1
      })
      
      // Create geometry cache to reuse geometries of the same dimensions
      const getGeometry = (width, height, depth) => {
        const key = `${width}_${height}_${depth}`
        if (!this.geometryCache[key]) {
          this.geometryCache[key] = new THREE.BoxGeometry(width, height, depth)
        }
        return this.geometryCache[key]
      }
      
      // Create material cache to reuse materials of the same color
      const getMaterial = (color) => {
        const colorKey = color ? color.getHex() : 'default'
        if (!this.materialCache[colorKey]) {
          this.materialCache[colorKey] = new THREE.MeshPhongMaterial({
            color: color || 0xeeeeee,
            transparent: true,
            opacity: 0.9
          })
        }
        return this.materialCache[colorKey]
      }
      
      // Create each slot with optimized geometry and material reuse
      layoutData.forEach(slotData => {
        const width = +slotData.WIDTH
        const depth = +slotData.DEPTH
        const height = +slotData.HEIGHT
        const x = +slotData.X
        const y = +slotData.Z
        const z = +slotData.Y
        
        // Get inventory data
        const inventory = inventoryLookup.get(slotData.LOCATION)
        let color = null
        
        if (inventory) {
          // Apply color based on selection
          if (this.colorBy === 'cases_damaged' && inventory["CASES DAMAGED"]) {
            const damageValue = +inventory["CASES DAMAGED"]
            const colorKey = `damage_${damageValue}`
            if (!this.colorCache[colorKey]) {
              this.colorCache[colorKey] = new THREE.Color(damagedColorScale(damageValue))
            }
            color = this.colorCache[colorKey]
          } else if (this.colorBy === 'weekly_mvmt' && inventory["WEEKLY MVMT"]) {
            const mvmtValue = +inventory["WEEKLY MVMT"]
            const colorKey = `mvmt_${mvmtValue}`
            if (!this.colorCache[colorKey]) {
              this.colorCache[colorKey] = new THREE.Color(movementColorScale(mvmtValue))
            }
            color = this.colorCache[colorKey]
          } else if (this.colorBy === 'item_cost' && inventory["ITEM COST"]) {
            const costValue = +inventory["ITEM COST"]
            const colorKey = `cost_${costValue}`
            if (!this.colorCache[colorKey]) {
              this.colorCache[colorKey] = new THREE.Color(costColorScale(costValue))
            }
            color = this.colorCache[colorKey]
          } else if (inventory["COLOR_CASES DAMAGED"]) {
            color = new THREE.Color(inventory["COLOR_CASES DAMAGED"])
          }
        }
        
        // Create mesh with cached geometry and material
        const geometry = getGeometry(width, height, depth)
        const material = getMaterial(color)
        const mesh = new THREE.Mesh(geometry, material)
        mesh.position.set(x, y + height/2, z)
        mesh.userData = {
          location: slotData.LOCATION,
          inventory: inventory
        }
        
        // Add edges (only if we're not dealing with too many objects)
        if (layoutData.length < 5000) {
          const edges = new THREE.LineSegments(
            new THREE.EdgesGeometry(geometry),
            edgeMaterial
          )
          edges.position.copy(mesh.position)
          this.slotsGroup.add(edges)
        }
        
        // Add to slots group
        this.slotsGroup.add(mesh)
        
        this.slots.push({
          mesh: mesh,
          data: slotData,
          inventory: inventory
        })
      })
      
      // Add slots group to warehouse group
      this.warehouseGroup.add(this.slotsGroup)
      
      // Position warehouse group at center
      this.warehouseGroup.position.copy(this.warehouseCenter)

      // Create floor planes for each AREA
      this.createAreaFloors(layoutData)
    },

    createAreaFloors() {
      // Xóa các sàn cũ nếu có
      if (this.floorPlanes) {
        this.floorPlanes.forEach(plane => {
          this.warehouseGroup.remove(plane)
          plane.geometry.dispose()
          plane.material.dispose()
        })
      }
      this.floorPlanes = [];

      // Tính toán sức chứa các khu vực
      const inventoryLookup = new Map()
      inventoryData.forEach(item => inventoryLookup.set(item.LOCATION, item));
      const areaCapacity = this.calculateAreaCapacity(layoutData, inventoryLookup);

      // Tạo sàn cho từng khu vực theo cấu hình
      this.areaConfigs.forEach(area => {
        const geometry = new THREE.PlaneGeometry(area.size.width, area.size.depth)
        const material = new THREE.MeshStandardMaterial({
          color: area.color,
          side: THREE.DoubleSide,
          transparent: true,
          opacity: 0.3,
          metalness: 0.1,
          roughness: 0.7
        })

        const floor = new THREE.Mesh(geometry, material)
        floor.rotation.x = -Math.PI / 2 // Xoay nằm ngang
        floor.rotation.z = area.rotation // Xoay theo cấu hình
        floor.position.set(
          area.position.x,
          -5, // Đặt thấp hơn các khe chứa
          area.position.z
        )
        
        // Áp dụng offset của kho
        floor.position.add(this.warehouseOffset)
        this.warehouseGroup.add(floor)
        this.floorPlanes.push(floor)

        // Thêm nhãn với thông tin sức chứa
        this.addAreaLabel(area, floor.position, areaCapacity[area.name])
      })
    },

    addAreaLabel(area, position, capacity) {
      const canvas = document.createElement('canvas')
      canvas.width = 400
      canvas.height = 120
      const context = canvas.getContext('2d')
      
      // Clear canvas với nền trong suốt
      context.clearRect(0, 0, canvas.width, canvas.height)
      
      // Tính phần trăm sức chứa
      const usagePercent = capacity ? Math.round((capacity.usedSlots / capacity.totalSlots) * 100) : 0
      const usageText = `${usagePercent}% (${capacity.usedSlots}/${capacity.totalSlots})`
      
      // Vẽ text với viền trắng (stroke)
      context.font = 'Bold 46px Arial'
      context.textAlign = 'center'
      
      context.strokeStyle = '#000000'
      context.lineWidth = 6
      context.strokeText(`Area ${area.name}`, canvas.width/2, 35)
      context.fillStyle = '#ffffff'
      context.fillText(`Area ${area.name}`, canvas.width/2, 35)
      
      // Phần trăm sức chứa với viền trắng
      context.font = '36px Arial'
      context.strokeText(usageText, canvas.width/2, 70)
      context.fillText(usageText, canvas.width/2, 70)
      
      // Thanh tiến trình
      const barWidth = 300
      const barHeight = 20
      const barX = (canvas.width - barWidth) / 2
      const barY = 80
      
      // Nền thanh
      context.fillStyle = '#eeeeee'
      context.fillRect(barX, barY, barWidth, barHeight)
      
      // Phần đã sử dụng
      context.fillStyle = `#${area.color.toString(16).padStart(6, '0')}`
      context.fillRect(barX, barY, barWidth * (usagePercent / 100), barHeight)
      
      // Viền thanh
      context.strokeStyle = '#000000'
      context.lineWidth = 1
      context.strokeRect(barX, barY, barWidth, barHeight)

      const texture = new THREE.CanvasTexture(canvas)
      const material = new THREE.SpriteMaterial({ 
        map: texture,
        transparent: true,
        opacity: 1
      })
      
      const sprite = new THREE.Sprite(material)
      sprite.scale.set(600, 180, 1)
      sprite.position.copy(position)
      sprite.position.y = 550
      
      this.warehouseGroup.add(sprite)
      this.floorPlanes.push(sprite)
    },
    
    updateVisualization() {
      // Clear color cache when visualization changes
      this.colorCache = {}
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
      
      const intersects = this.raycaster.intersectObjects(
        this.slots.map(slot => slot.mesh),
        false // Don't check recursively for better performance
      )
      
      const tooltip = this.$refs.tooltip
      if (intersects.length > 0) {
        const object = intersects[0].object
        
        if (this.intersectedObject !== object) {
          this.intersectedObject = object
          
          if (object.userData.inventory) {
            const inv = object.userData.inventory
            tooltip.innerHTML = `
              <strong>${object.userData.location}</strong><br>
              Item: ${inv["ITEM DESCRIPTION"]}<br>
              Cases Damaged: ${inv["CASES DAMAGED"]}<br>
              Weekly Movement: ${inv["WEEKLY MVMT"]}<br>
              Item Cost: $${inv["ITEM COST"]}
            `
          } else {
            tooltip.innerHTML = `
              <strong>${object.userData.location}</strong><br>
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
      
      // Implement frame rate limiting
      const now = performance.now()
      const delta = now - this.lastRenderTime
      
      if (delta >= this.frameInterval) {
        this.controls.update()
        this.renderer.render(this.scene, this.camera)
        this.lastRenderTime = now - (delta % this.frameInterval)
      }
    },
    
    disposeGroup(group) {
      if (!group) return
      
      group.traverse(child => {
        if (child.geometry) child.geometry.dispose()
        if (child.material) {
          if (Array.isArray(child.material)) {
            child.material.forEach(mat => mat.dispose())
          } else {
            child.material.dispose()
          }
        }
      })
    },
    
    cleanUp() {
      cancelAnimationFrame(this.animationFrameId)
      window.removeEventListener('resize', this.throttle(this.onWindowResize, 100))
      
      if (this.$refs.container) {
        this.$refs.container.removeEventListener('mousemove', this.throttle(this.onMouseMove, 50))
      }
      
      // Dispose of all Three.js objects
      this.disposeGroup(this.slotsGroup)
      this.disposeGroup(this.warehouseGroup)
      
      if (this.gridHelper) {
        this.scene.remove(this.gridHelper)
        this.gridHelper.dispose()
      }
      
      if (this.renderer) {
        this.renderer.dispose()
      }
      
      // Clear caches
      this.colorCache = {}
      this.materialCache = {}
      this.geometryCache = {}

      // Xóa trục tọa độ khi dọn dẹp
      if (this.axesHelper) {
        this.scene.remove(this.axesHelper)
        this.axesHelper.dispose()
      }
    },

    calculateAreaCapacity(layoutData, inventoryLookup) {
      // Tính toán sức chứa từng khu vực
      const areaStats = {}
      layoutData.forEach(slot => {
        if (!areaStats[slot.AREA]) {
          areaStats[slot.AREA] = {
            totalSlots: 0,
            usedSlots: 0
          }
        }
        areaStats[slot.AREA].totalSlots++
        if (inventoryLookup.get(slot.LOCATION)) {
          areaStats[slot.AREA].usedSlots++
        }
      })
      return areaStats
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
#app { 
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
.controls select, .controls button {
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