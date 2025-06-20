<template>
  <div ref="container" class="grid-container"></div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';

export default {
  name: 'DemoThree',
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      gridSize: { x:  500, y: 500, z: 3 }, // Giảm kích thước để demo (đổi lại 500x500x3 khi triển khai thực tế)
      cellSize: 50,
      instances: [],
      currentLOD: 'low',
      frameId: null
    };
  },
  computed: {
    totalCells() {
      return this.gridSize.x * this.gridSize.y * this.gridSize.z;
    }
  },
  mounted() {
    this.initThreeJS();
    this.createOptimizedGrid();
    this.animate();
    window.addEventListener('resize', this.handleResize);
  },
  beforeDestroy() {
    this.cleanup();
  },
  methods: {
    initThreeJS() {
      // Scene
      this.scene = new THREE.Scene();
      this.scene.background = new THREE.Color(0xf0f0f0);

      // Camera
      const container = this.$refs.container;
      const width = container.clientWidth;
      const height = container.clientHeight;
      
      this.camera = new THREE.PerspectiveCamera(60, width / height, 1, 100000);
      this.camera.position.set(500, 500, 500);
      this.camera.lookAt(0, 0, 0);

      // Renderer
      this.renderer = new THREE.WebGLRenderer({
        antialias: false,
        logarithmicDepthBuffer: false,
      })
      
      this.renderer.shadowMap.enabled = false;
      this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
      this.renderer.setSize(width, height);
      container.appendChild(this.renderer.domElement);

      // Controls
      this.controls = new OrbitControls(this.camera, this.renderer.domElement);
      this.controls.enableDamping = true;
      this.controls.dampingFactor = 0.05;

      // Lighting
      const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
      this.scene.add(ambientLight);

      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(1, 1, 1).normalize();
      this.scene.add(directionalLight);
    },
    
    createGeometries() {
      return {
        high: new THREE.BoxGeometry(this.cellSize - 4, this.cellSize - 4, this.cellSize - 4, 8, 8, 8),
        medium: new THREE.BoxGeometry(this.cellSize - 4, this.cellSize - 4, this.cellSize - 4, 3, 3, 3),
        low: new THREE.BoxGeometry(this.cellSize - 4, this.cellSize - 4, this.cellSize - 4, 1, 1, 1)
      };
    },
    
    createOptimizedGrid() {
      const geometries = this.createGeometries();
      const material = new THREE.MeshPhongMaterial({
        color: 0x3498db,
        transparent: true,
        opacity: 0.8,
        shininess: 30
      });
           const materialMedium = new THREE.MeshPhongMaterial({
        color: 0xaa98db,
        transparent: true,
        opacity: 0.8,
        shininess: 30
      });
           const materialLow = new THREE.MeshPhongMaterial({
        color: 0xff98db,
        transparent: true,
        opacity: 0.8,
        shininess: 30
      });

      // Tạo InstancedMesh cho từng mức LOD
      this.instances = {
        high: new THREE.InstancedMesh(geometries.high, material, this.totalCells),
        medium: new THREE.InstancedMesh(geometries.medium, materialMedium, this.totalCells),
        low: new THREE.InstancedMesh(geometries.low, materialLow, this.totalCells)
      };

      // Cấu hình ban đầu
      Object.values(this.instances).forEach(instance => {
        instance.visible = false;
        instance.count = this.totalCells;
        instance.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
        this.scene.add(instance);
      });

      // Thiết lập vị trí cho tất cả instances
      this.setupInstancePositions();
      
      // Bật LOD mặc định
      this.instances.low.visible = true;
      this.currentLOD = 'low';
    },
    
    setupInstancePositions() {
      const dummy = new THREE.Object3D();
      const halfX = (this.gridSize.x * this.cellSize) / 2;
      const halfY = (this.gridSize.y * this.cellSize) / 2;
      
      let index = 0;
      for (let x = 0; x < this.gridSize.x; x++) {
        for (let y = 0; y < this.gridSize.y; y++) {
          for (let z = 0; z < this.gridSize.z; z++) {
            dummy.position.set(
              x * this.cellSize - halfX,
              z * this.cellSize,
              y * this.cellSize - halfY
            );
            dummy.updateMatrix();
            
            // Áp dụng cùng vị trí cho tất cả mức LOD
            this.instances.high.setMatrixAt(index, dummy.matrix);
            this.instances.medium.setMatrixAt(index, dummy.matrix);
            this.instances.low.setMatrixAt(index, dummy.matrix);
            index++;
          }
        }
      }
      
      // Cập nhật buffer
      this.instances.high.instanceMatrix.needsUpdate = true;
      this.instances.medium.instanceMatrix.needsUpdate = true;
      this.instances.low.instanceMatrix.needsUpdate = true;
    },
    
    updateLOD() {
      const distance = this.camera.position.distanceTo(new THREE.Vector3(0, 0, 0));
      let newLOD = 'low';
      
      if (distance < 300) {
        newLOD = 'high';
      } else if (distance < 600) {
        newLOD = 'medium';
      }
      
      if (newLOD !== this.currentLOD) {
        this.instances[this.currentLOD].visible = false;
        this.instances[newLOD].visible = true;
        this.currentLOD = newLOD;
      }
    },
    
    animate() {
      this.frameId = requestAnimationFrame(this.animate);
      this.updateLOD();
      this.controls.update();
      this.renderer.render(this.scene, this.camera);
    },
    
    handleResize() {
      const container = this.$refs.container;
      const width = container.clientWidth;
      const height = container.clientHeight;
      
      this.camera.aspect = width / height;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(width, height);
    },
    
    cleanup() {
      window.removeEventListener('resize', this.handleResize);
      cancelAnimationFrame(this.frameId);
      
      // Dọn dẹp tài nguyên
      if (this.renderer) {
        this.renderer.dispose();
        this.renderer.forceContextLoss();
        this.renderer.domElement = null;
      }
      
      // Hủy các instances
      Object.values(this.instances).forEach(instance => {
        instance.geometry.dispose();
        instance.material.dispose();
        this.scene.remove(instance);
      });
      
      if (this.$refs.container && this.renderer?.domElement) {
        this.$refs.container.removeChild(this.renderer.domElement);
      }
    }
  }
};
</script>

<style scoped>
.grid-container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  position: relative;
}
</style>