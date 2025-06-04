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
  name: 'DemoOne',
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
      instancesGroup: null,
      warehouseOffset: 0,
      lastRenderTime: 0,
      frameRate: 30,
      frameInterval: 1000 / 30,
      colorCache: {},
      materialCache: {},
      geometryCache: {},
      instanceMeshes: {}, // Store our InstancedMesh objects
      instanceData: [], // Store data for each instance
      areaConfigs: [
        {
          name: 'A',
          color: 0xff9999,
          position: { x: 1100, z: 1950 },
          size: { width: 3000, depth: 4500 },
          rotation: 0
        },
        {
          name: 'B',
          color: 0x99ff99,
          position: { x: 4200, z: 3100 },
          size: { width: 3000, depth: 2200 },
          rotation: 0
        },
        {
          name: 'Staging',
          color: 0x9999ff,
          position: { x: 4200, z: 800 },
          size: { width: 3000, depth: 2200 },
          rotation: 0
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
          dummy.position.set(x, y + group.height/2, z)
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
            position: new THREE.Vector3(x, y + group.height/2, z)
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

      // Create floor planes for each AREA
      this.createAreaFloors(layoutData)
      
      // Create combined walls
      this.createCombinedAreaWalls()
    },

    createCombinedAreaWalls() {
      if (this.combinedWalls) {
        this.combinedWalls.forEach(wall => {
          this.warehouseGroup.remove(wall)
          wall.geometry.dispose()
          wall.material.dispose()
        })
      }
      this.combinedWalls = []

      const areaPositions = this.areaConfigs.map(area => {
        return [
          new THREE.Vector3(
            -5550 + area.position.x - area.size.width/2,
            0,
            -4100 + area.position.z - area.size.depth/2
          ),
          new THREE.Vector3(
            -5550 + area.position.x + area.size.width/2,
            0,
            -4100 + area.position.z + area.size.depth/2
          )
        ]
      }).flat()
      
      const boundingBox = new THREE.Box3().setFromPoints(areaPositions)
      const size = new THREE.Vector3()
      boundingBox.getSize(size)

      const wallMaterial = new THREE.MeshStandardMaterial({
        color: 0xdddddd,
        transparent: true,
        opacity: 0.1,
        side: THREE.DoubleSide,
      })

      const wallThickness = 50
      const wallHeight = 100

      const wallsConfig = [
        {
          position: [
            boundingBox.min.x - wallThickness/2,
            wallHeight/2,
            boundingBox.min.z + size.z/2
          ],
          size: [wallThickness, wallHeight, size.z + wallThickness*2]
        },
        {
          position: [
            boundingBox.max.x + wallThickness/2,
            wallHeight/2,
            boundingBox.min.z + size.z/2
          ],
          size: [wallThickness, wallHeight, size.z + wallThickness*2]
        },
        {
          position: [
            boundingBox.min.x + size.x/2,
            wallHeight/2,
            boundingBox.min.z - wallThickness/2
          ],
          size: [size.x + wallThickness*2, wallHeight, wallThickness]
        },
        {
          position: [
            boundingBox.min.x + size.x/2,
            wallHeight/2,
            boundingBox.max.z + wallThickness/2
          ],
          size: [size.x + wallThickness*2, wallHeight, wallThickness]
        }
      ]

      wallsConfig.forEach(wall => {
        const wallMesh = new THREE.Mesh(
          new THREE.BoxGeometry(...wall.size),
          wallMaterial
        )
        wallMesh.position.set(
          wall.position[0] - this.warehouseOffset.x,
          wall.position[1],
          wall.position[2] - this.warehouseOffset.z
        )
        this.warehouseGroup.add(wallMesh)
        this.combinedWalls.push(wallMesh)
      })
    },

    createAreaFloors() {
      if (this.floorPlanes) {
        this.floorPlanes.forEach(plane => {
          this.warehouseGroup.remove(plane)
          plane.geometry.dispose()
          plane.material.dispose()
        })
      }
      this.floorPlanes = [];

      const inventoryLookup = new Map()
      inventoryData.forEach(item => inventoryLookup.set(item.LOCATION, item));
      const areaCapacity = this.calculateAreaCapacity(layoutData, inventoryLookup);

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
        floor.rotation.x = -Math.PI / 2
        floor.rotation.z = area.rotation
        floor.position.set(
          area.position.x,
          -5,
          area.position.z
        )
        
        floor.position.add(this.warehouseOffset)
        this.warehouseGroup.add(floor)
        this.floorPlanes.push(floor)

        this.addAreaLabel(area, floor.position, areaCapacity[area.name])
      })
    },

    addAreaLabel(area, position, capacity) {
      const canvas = document.createElement('canvas')
      canvas.width = 400
      canvas.height = 120
      const context = canvas.getContext('2d')
      
      context.clearRect(0, 0, canvas.width, canvas.height)
      
      const usagePercent = capacity ? Math.round((capacity.usedSlots / capacity.totalSlots) * 100) : 0
      const usageText = `${usagePercent}% (${capacity.usedSlots}/${capacity.totalSlots})`
      
      context.font = 'Bold 46px Arial'
      context.textAlign = 'center'
      
      context.strokeStyle = '#000000'
      context.lineWidth = 6
      context.strokeText(`Area ${area.name}`, canvas.width/2, 35)
      context.fillStyle = '#ffffff'
      context.fillText(`Area ${area.name}`, canvas.width/2, 35)
      
      context.font = '36px Arial'
      context.strokeText(usageText, canvas.width/2, 70)
      context.fillText(usageText, canvas.width/2, 70)
      
      const barWidth = 300
      const barHeight = 20
      const barX = (canvas.width - barWidth) / 2
      const barY = 80
      
      context.fillStyle = '#eeeeee'
      context.fillRect(barX, barY, barWidth, barHeight)
      
      context.fillStyle = `#${area.color.toString(16).padStart(6, '0')}`
      context.fillRect(barX, barY, barWidth * (usagePercent / 100), barHeight)
      
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

    calculateAreaCapacity(layoutData, inventoryLookup) {
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