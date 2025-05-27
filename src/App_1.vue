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
      warehouseGroup: null,  // Thêm group cho warehouse
      slotsGroup: null,       // Thêm group cho các slot
      warehouseOffset: null       // Thêm group cho các slot
    }
  },
  mounted() {
    this.initThreeJS()
    this.calculateWarehouseCenter(layoutData)
    this.createWarehouse(layoutData, inventoryData)
    this.animate()
    window.addEventListener('resize', this.onWindowResize)
    this.$refs.container.addEventListener('mousemove', this.onMouseMove)
  },
  beforeDestroy() {
    this.cleanUp()
  },
  methods: {
    calculateWarehouseCenter(layoutData) {
  // Tính toán bounding box để xác định kích thước tổng thể
  const positions = layoutData.map(slot => new THREE.Vector3(+slot.X, +slot.Z, +slot.Y))
  const boundingBox = new THREE.Box3().setFromPoints(positions)
  
  // Tính toán offset để dịch chuyển warehouse về trung tâm
  const size = new THREE.Vector3()
  boundingBox.getSize(size)
  this.warehouseOffset = new THREE.Vector3(
    -boundingBox.min.x - size.x/2,
    -boundingBox.min.y,
    -boundingBox.min.z - size.z/2
  )
  
  // Tính toán trung tâm warehouse (sau khi đã dịch chuyển)
  this.warehouseCenter = new THREE.Vector3(0, size.y/2, 0)
},
    
    initThreeJS() {
  // Create scene
  this.scene = new THREE.Scene()
  this.scene.background = new THREE.Color(0xf0f0f0)

  // Tạo group chính cho warehouse
  this.warehouseGroup = new THREE.Group()
  this.scene.add(this.warehouseGroup)

  // Create camera - position it based on warehouse center
  this.camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 10000)
  this.camera.position.set(0, 1500, 2000) // Vị trí camera mặc định
  this.camera.lookAt(0, 0, 0)
  // this.camera.position.x += 1000
  // this.camera.position.y += 1500
  // this.camera.position.z += 1000

  // Create renderer with performance options
  this.renderer = new THREE.WebGLRenderer({
    antialias: true,
    powerPreference: "high-performance",
    logarithmicDepthBuffer: true
  })
  this.renderer.setPixelRatio(window.devicePixelRatio)
  this.renderer.setSize(window.innerWidth, window.innerHeight)
  this.$refs.container.appendChild(this.renderer.domElement)

  // Add controls with damping for smoother movement
  this.controls = new OrbitControls(this.camera, this.renderer.domElement)
  this.controls.enableDamping = true
  this.controls.dampingFactor = 0.25
  this.controls.screenSpacePanning = false
  this.controls.target.copy(this.warehouseCenter)

  // Optimized lighting
  const ambientLight = new THREE.AmbientLight(0x404040, 0.5)
  this.scene.add(ambientLight)

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8)
  directionalLight.position.set(1, 1, 1).normalize()
  this.scene.add(directionalLight)

  // Add grid helper centered at warehouse center
  this.gridHelper = new THREE.GridHelper(5000, 100, 0x888888, 0x888888)
  this.gridHelper.position.copy(this.warehouseCenter)
  this.scene.add(this.gridHelper)
},

    
    createWarehouse(layoutData, inventoryData) {
      // Clear existing slots efficiently
      if (this.slotsGroup) {
        this.warehouseGroup.remove(this.slotsGroup)
        this.slotsGroup.children.forEach(child => {
          if (child.geometry) child.geometry.dispose()
          if (child.material) child.material.dispose()
        })
      }
      this.slots = []
      
      // Tạo group mới cho các slot
      this.slotsGroup = new THREE.Group()
      
      // Di chuyển group đến vị trí đối xứng với trung tâm
      this.slotsGroup.position.copy(this.warehouseCenter).multiplyScalar(-1)
      
      // Prepare color scales
      const damagedColorScale = d3.scaleLinear()
        .domain([0, 8])
        .range(["#00ff00", "#ff0000"])
        
      const movementColorScale = d3.scaleSequential(d3.interpolateRdYlGn)
        .domain(d3.extent(inventoryData, d => +d["WEEKLY MVMT"]))
        
      const costColorScale = d3.scaleSequential(d3.interpolateBlues)
        .domain(d3.extent(inventoryData, d => +d["ITEM COST"]))
      
      // Create inventory lookup map
      const inventoryLookup = new Map()
      inventoryData.forEach(item => {
        inventoryLookup.set(item.LOCATION, item)
      })
      
      // Create shared materials for better performance
      const defaultMaterial = new THREE.MeshPhongMaterial({
        color: 0xcccccc,
        transparent: true,
        opacity: 0.8
      })
      
      const edgeMaterial = new THREE.LineBasicMaterial({ 
        color: 0x000000,
        linewidth: 1
      })

        // Tạo group mới cho các slot
  this.slotsGroup = new THREE.Group()
  
  // Áp dụng offset để dịch chuyển toàn bộ warehouse về trung tâm
  this.slotsGroup.position.copy(this.warehouseOffset)
      
      // Create each slot
      layoutData.forEach(slotData => {
        const width = +slotData.WIDTH
        const depth = +slotData.DEPTH
        const height = +slotData.HEIGHT
        const x = +slotData.X
        const y = +slotData.Z // Three.js uses Y as up
        const z = +slotData.Y
        
        // Create geometry (consider using BufferGeometry for better performance)
        const geometry = new THREE.BoxGeometry(width, height, depth)
        
        // Get inventory data
        const inventory = inventoryLookup.get(slotData.LOCATION)
        let material = defaultMaterial.clone()
        
        if (inventory) {
          // Apply color based on selection
          let color
          if (this.colorBy === 'cases_damaged' && inventory["CASES DAMAGED"]) {
            color = new THREE.Color(damagedColorScale(+inventory["CASES DAMAGED"]))
          } else if (this.colorBy === 'weekly_mvmt' && inventory["WEEKLY MVMT"]) {
            color = new THREE.Color(movementColorScale(+inventory["WEEKLY MVMT"]))
          } else if (this.colorBy === 'item_cost' && inventory["ITEM COST"]) {
            color = new THREE.Color(costColorScale(+inventory["ITEM COST"]))
          } else if (inventory["COLOR_CASES DAMAGED"]) {
            color = new THREE.Color(inventory["COLOR_CASES DAMAGED"])
          }
          
          if (color) {
            material = new THREE.MeshPhongMaterial({
              color: color,
              transparent: true,
              opacity: 0.8
            })
          }
        }
        
        // Create mesh
        const mesh = new THREE.Mesh(geometry, material)
        mesh.position.set(x, y + height/2, z)
        mesh.userData = {
          location: slotData.LOCATION,
          inventory: inventory
        }
        
        // Add edges
        const edges = new THREE.LineSegments(
          new THREE.EdgesGeometry(geometry),
          edgeMaterial
        )
        edges.position.copy(mesh.position)
        
        // Add to slots group
        this.slotsGroup.add(mesh)
        this.slotsGroup.add(edges)
        
        this.slots.push({
          mesh: mesh,
          edges: edges,
          data: slotData,
          inventory: inventory
        })
      })
      
      // Add slots group to warehouse group
      this.warehouseGroup.add(this.slotsGroup)
      
      // Position warehouse group at center
      this.warehouseGroup.position.copy(this.warehouseCenter)
    },
    
    updateVisualization() {
      this.createWarehouse(layoutData, inventoryData)
    },
    
    resetCamera() {
  if (!this.camera || !this.controls) return
  
  // Đặt camera nhìn vào trung tâm (0,0,0 sau khi đã dịch chuyển)
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
      // Only process tooltip if mouse is over container
      if (!this.$refs.container) return
      
      this.mouse.x = (event.clientX / this.$refs.container.clientWidth) * 2 - 1
      this.mouse.y = -(event.clientY / this.$refs.container.clientHeight) * 2 + 1
      
      this.raycaster.setFromCamera(this.mouse, this.camera)
      
      const intersects = this.raycaster.intersectObjects(
        this.slots.map(slot => slot.mesh)
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
      this.controls.update()
      this.renderer.render(this.scene, this.camera)
    },
    
    cleanUp() {
      cancelAnimationFrame(this.animationFrameId)
      window.removeEventListener('resize', this.onWindowResize)
      
      if (this.$refs.container) {
        this.$refs.container.removeEventListener('mousemove', this.onMouseMove)
      }
      
      // Dispose of all Three.js objects
      if (this.slotsGroup) {
        this.slotsGroup.children.forEach(child => {
          if (child.geometry) child.geometry.dispose()
          if (child.material) child.material.dispose()
        })
      }
      
      if (this.gridHelper) {
        this.scene.remove(this.gridHelper)
        this.gridHelper.dispose()
      }
      
      if (this.renderer) {
        this.renderer.dispose()
      }
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