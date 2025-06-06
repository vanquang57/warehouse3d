<template>
  <div class="container">
    <div class="scene-container" ref="sceneContainer"></div>
    <div class="table-container">
      <table class="inventory-table">
        <thead>
          <tr>
            <th>
              <input
                type="checkbox"
                v-model="checkAll"
                @change="toggleCheckAll"
                :indeterminate="indeterminate"
              />
            </th>
            <th>LOCATION</th>
            <th>ITEM NO</th>
            <th>DESCRIPTION</th>
            <th>COLOR</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, idx) in warehouseData" :key="idx">
            <td>
              <input
                type="checkbox"
                v-model="selectedItems[idx]"
                @change="updateCheckAllState"
              />
            </td>
            <td>{{ item.LOCATION }}</td>
            <td>{{ item['\\sITEM NO'] }}</td>
            <td>{{ item['ITEM DESCRIPTION'] }}</td>
            <td>{{ item['COLOR_CASES DAMAGED'] }}</td>
          </tr>
        </tbody>
      </table>
    </div>
    <div class="tooltip" ref="tooltip" v-show="tooltipVisible" :style="tooltipStyle">
      {{ tooltipText }}
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref, computed, watch } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls'

import warehouseData from '@/assets/inventory_converted.json'
import layoutData from '@/assets/layout.json'

// Checkbox logic
const checkAll = ref(true)
const selectedItems = ref(new Array(warehouseData.length).fill(true))

const indeterminate = computed(() => {
  const anyChecked = selectedItems.value.some(v => v)
  const allChecked = selectedItems.value.every(v => v)
  return anyChecked && !allChecked
})

const toggleCheckAll = () => {
  const newVal = checkAll.value
  selectedItems.value = selectedItems.value.map(() => newVal)
}

const updateCheckAllState = () => {
  checkAll.value = selectedItems.value.every(v => v)
}

// Tooltip logic
const tooltip = ref(null)
const tooltipText = ref('')
const tooltipVisible = ref(false)
const tooltipStyle = ref({ top: '0px', left: '0px' })

// Three.js logic
const sceneContainer = ref(null)
const objectsInScene = ref([])

onMounted(() => {
  const scene = new THREE.Scene()
  scene.background = new THREE.Color(0xf0f0f0)

  // 👉 Tính center từ layoutData
  const scaleFactor = 0.1



  let totalX = 0, totalY = 0, totalZ = 0
  layoutData.forEach(loc => {
    totalX += parseFloat(loc.X) * scaleFactor
    totalY += parseFloat(loc.Z || 0) * scaleFactor
    totalZ += parseFloat(loc.Y) * scaleFactor
  })

  const centerX = totalX / layoutData.length
  const centerY = totalY / layoutData.length
  const centerZ = totalZ / layoutData.length
  const center = new THREE.Vector3(centerX, centerY, centerZ)

  

  // 👁️ Camera
  const camera = new THREE.PerspectiveCamera( 75, (window.innerWidth * 0.7) / window.innerHeight, 0.1, 2000)

  // 👉 Đặt vị trí camera lùi về sau và cao lên để quan sát trung tâm
  camera.position.set(centerX + 150, centerY + 150, centerZ + 150)
  camera.lookAt(center)


  const renderer = new THREE.WebGLRenderer({ antialias: true })
  renderer.setSize(window.innerWidth * 0.7, window.innerHeight)
  sceneContainer.value.appendChild(renderer.domElement)

  const controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.target.set(centerX, centerY, centerZ)
  controls.dampingFactor = 0.05

  // Lights
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8)
  dirLight.position.set(10, 20, 10)
  scene.add(dirLight)
  scene.add(new THREE.AmbientLight(0xffffff, 0.2))

const axesHelper = new THREE.AxesHelper(50)
axesHelper.position.set(centerX, centerY, centerZ)
axesHelper.visible = false
scene.add(axesHelper)

const gridHelper = new THREE.GridHelper(400, 100, 0x888888, 0xcccccc);
gridHelper.position.set(centerX, -14, centerZ); // căn giữa X, Z
scene.add(gridHelper);

scene.background = new THREE.Color(0xf0f0f0)

  const boxGeometry = new THREE.BoxGeometry(1, 1, 1)

  // Group warehouse items by location
  const groupedItems = {}
  warehouseData.forEach(item => {
    const loc = item.LOCATION
    if (!groupedItems[loc]) groupedItems[loc] = []
    groupedItems[loc].push(item)
  })

  layoutData.forEach(loc => {
    const locationId = loc.LOCATION
    const items = groupedItems[locationId] || []

    const baseX = parseFloat(loc.X) * scaleFactor || 0
    const baseZ = parseFloat(loc.Y) * scaleFactor || 0
    const baseY = (parseFloat(loc.Z) || 0) * scaleFactor - 10  // hạ thấp trục Y xuống 10 đơn vị
    const width = parseFloat(loc.WIDTH) * scaleFactor || 1
    const height = parseFloat(loc.HEIGHT) * scaleFactor || 1
    const depth = parseFloat(loc.DEPTH) * scaleFactor || 1

    if (items.length === 0) {
      // Empty location - bán trong suốt trắng
      const material = new THREE.MeshStandardMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.2
      })
      const cube = new THREE.Mesh(boxGeometry, material)
      cube.scale.set(width, height, depth)
      cube.position.set(baseX, baseY, baseZ)
      scene.add(cube)
    } else {
      // Non-empty location - hộp bán trong suốt có màu
      items.forEach((item, index) => {
        let baseColor
        switch (item["COLOR_CASES DAMAGED"]) {
          case "red":
            baseColor = 0xff0000
            break
          case "blue":
            baseColor = 0x0000ff
            break
          default:
            baseColor = 0x00ff00
        }

        const material = new THREE.MeshStandardMaterial({
          color: baseColor,
          transparent: true,
          opacity: index === 0 ? 0.7 : 0.4
        })

        const cube = new THREE.Mesh(boxGeometry, material)
        cube.scale.set(width, height, depth)
        cube.position.set(baseX, baseY + index * height, baseZ)

        // 🖤 Viền đen (EdgesGeometry)
        const edges = new THREE.EdgesGeometry(boxGeometry)
        const lineMaterial = new THREE.LineBasicMaterial({ color: 0x999999 })
        const lineSegments = new THREE.LineSegments(edges, lineMaterial)
        cube.add(lineSegments) // Gắn vào cube để luôn đi cùng

        const globalIndex = warehouseData.findIndex(
          d => d.LOCATION === item.LOCATION && d['ITEM NO'] === item['ITEM NO']
        )

        cube.visible = selectedItems.value[globalIndex]
        scene.add(cube)

        objectsInScene.value.push({ itemIndex: globalIndex, cube })
      })
    }
  })

  // Raycaster for tooltip
  const raycaster = new THREE.Raycaster()
  const mouse = new THREE.Vector2()

  renderer.domElement.addEventListener('mousemove', event => {
    const rect = renderer.domElement.getBoundingClientRect()
    mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
    mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

    raycaster.setFromCamera(mouse, camera)
    const intersects = raycaster.intersectObjects(objectsInScene.value.map(o => o.cube))

    if (intersects.length > 0) {
      const intersect = intersects[0]
      const found = objectsInScene.value.find(o => o.cube === intersect.object)

      if (found) {
        const item = warehouseData[found.itemIndex]
        tooltipText.value = `ITEM NO: ${item['ITEM NO']}\nDESC: ${item['ITEM DESCRIPTION']}`
        tooltipVisible.value = true
        tooltipStyle.value = {
          top: `${event.clientY + 10}px`,
          left: `${event.clientX + 10}px`
        }
      }
    } else {
      tooltipVisible.value = false
    }
  })

  // Animation loop
  const animate = () => {
    requestAnimationFrame(animate)
    controls.update()
    renderer.render(scene, camera)
  }
  animate()

  // Handle resize
  window.addEventListener('resize', () => {
    camera.aspect = (window.innerWidth * 0.7) / window.innerHeight
    camera.updateProjectionMatrix()
    renderer.setSize(window.innerWidth * 0.7, window.innerHeight)
  })
})

// Watch checkboxes and toggle box visibility
watch(selectedItems, (newVal) => {
  objectsInScene.value.forEach(({ itemIndex, cube }) => {
    cube.visible = newVal[itemIndex]
  })
}, { deep: true })
</script>

<style scoped>
.container {
  display: flex;
  height: 100vh;
  position: relative;
}

.scene-container {
  width: 70%;
  height: 99%;
}

.table-container {
  width: 30%;
  padding: 20px;
  overflow-y: auto;
  background-color: #f9f9f9;
}

.inventory-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 14px;
}

.inventory-table th,
.inventory-table td {
  border: 1px solid #ccc;
  padding: 6px 8px;
  text-align: left;
}

.inventory-table th {
  background-color: #eee;
}

.inventory-table td input[type="checkbox"] {
  margin: 0;
}

.tooltip {
  position: fixed;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 6px 10px;
  border-radius: 4px;
  font-size: 13px;
  pointer-events: none;
  white-space: pre-line;
  z-index: 100;
}
</style>