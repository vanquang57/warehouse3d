<template>
  <div>
    <select v-model="selectedZone" @change="filterZone" style="position: absolute; top: 10px; left: 10px; z-index: 10;">
      <option value="All">All Zones</option>
      <option v-for="zone in ['A','B','C','D','E']" :key="zone" :value="zone">Zone {{ zone }}</option>
    </select>

    <div
      v-if="tooltip.visible"
      :style="{
        position: 'absolute',
        top: tooltip.y + 'px',
        left: tooltip.x + 'px',
        background: '#ffffff',
        border: '1px solid #ccc',
        padding: '6px 10px',
        borderRadius: '4px',
        boxShadow: '0 2px 6px rgba(0,0,0,0.2)',
        pointerEvents: 'none',
        zIndex: 20
      }"
    >
      {{ tooltip.text }}
    </div>

    <div ref="sceneContainer" style="width: 100vw; height: 100vh;"></div>
  </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls';
import SpriteText from 'three-spritetext';

export default {
  name: 'App',
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      raycaster: new THREE.Raycaster(),
      mouse: new THREE.Vector2(),
      palletInfoMap: new Map(),
      palletMesh: null,
      zoneMeshes: [],
      zoneBorders: [],
      hoveredZoneIndex: null,
      selectedZone: 'All',
      tooltip: {
        visible: false,
        text: '',
        x: 0,
        y: 0,
      },
      palletZoneMap: new Map(), // instanceId → zone
      };
  },
  mounted() {
    this.initScene();
    this.createWarehouse();
    this.animate();
    window.addEventListener('click', this.onClick, false);
    window.addEventListener('mousemove', this.onMouseMove, false);
    window.addEventListener('resize', this.onWindowResize, false);
  },
  methods: {
    initScene() {
      this.scene = new THREE.Scene();
      this.scene.background = new THREE.Color(0xf0f0f0);

      this.camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 1, 2000);
      this.camera.position.set(300, 300, 300);

      this.renderer = new THREE.WebGLRenderer({ antialias: true });
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.$refs.sceneContainer.appendChild(this.renderer.domElement);

      const controls = new OrbitControls(this.camera, this.renderer.domElement);
      controls.update();

      const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
      const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
      directionalLight.position.set(100, 200, 100);
      this.scene.add(ambientLight, directionalLight);

      const gridHelper = new THREE.GridHelper(1000, 100, 0xcccccc, 0xeeeeee);
      gridHelper.position.y = 0;
      this.scene.add(gridHelper);
    },

    createWarehouse() {
      const group = new THREE.Group();

      const uprightMaterial = new THREE.MeshLambertMaterial({ color: 0x2e8b57 });
      const beamMaterial = new THREE.MeshLambertMaterial({ color: 0xff8c00 });
      const shelfMaterial = new THREE.MeshLambertMaterial({ color: 0xcccccc });

      const uprightHeight = 30;
      const uprightDepth = 0.5;
      const uprightWidth = 0.5;

      const beamLength = 20;
      const beamHeight = 0.5;
      const shelfDepth = 10;

      const rackSpacing = 25;
      const rowSpacing = 30;
      const zoneSpacing = 200;

      const palletGeometry = new THREE.BoxGeometry(8, 1, 8);
      const palletMaterial = new THREE.MeshLambertMaterial({ color: 0x00bcd4 });

      const zones = ['A', 'B', 'C', 'D', 'E'];
      const racksPerRow = 10;
      const rowsPerZone = 5;
      const shelfLevels = 3;

      const totalPallets = zones.length * rowsPerZone * racksPerRow * shelfLevels;
      this.palletMesh = new THREE.InstancedMesh(palletGeometry, palletMaterial, totalPallets);
      this.palletMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
      group.add(this.palletMesh);

      let palletIndex = 0;

      const createRack = (x, y, z, rackId) => {
        const g = new THREE.Group();

        const leftUpright = new THREE.Mesh(new THREE.BoxGeometry(uprightWidth, uprightHeight, uprightDepth), uprightMaterial);
        leftUpright.position.set(x, y + uprightHeight / 2, z - shelfDepth / 2);
        g.add(leftUpright);

        const rightUpright = new THREE.Mesh(new THREE.BoxGeometry(uprightWidth, uprightHeight, uprightDepth), uprightMaterial);
        rightUpright.position.set(x + beamLength, y + uprightHeight / 2, z - shelfDepth / 2);
        g.add(rightUpright);

        for (let i = 0; i < shelfLevels; i++) {
          const beamY = y + (i * (uprightHeight / shelfLevels)) + 2;

          const frontBeam = new THREE.Mesh(new THREE.BoxGeometry(beamLength, beamHeight, uprightDepth), beamMaterial);
          frontBeam.position.set(x + beamLength / 2, beamY, z - shelfDepth / 2);
          g.add(frontBeam);

          const backBeam = new THREE.Mesh(new THREE.BoxGeometry(beamLength, beamHeight, uprightDepth), beamMaterial);
          backBeam.position.set(x + beamLength / 2, beamY, z + shelfDepth / 2);
          g.add(backBeam);

          const shelf = new THREE.Mesh(new THREE.BoxGeometry(beamLength, 1, shelfDepth), shelfMaterial);
          shelf.position.set(x + beamLength / 2, beamY + 0.5, z);
          g.add(shelf);

          const palletMatrix = new THREE.Matrix4();
          palletMatrix.setPosition(x + beamLength / 2, beamY + 2, z);
          this.palletMesh.setMatrixAt(palletIndex, palletMatrix);
          this.palletInfoMap.set(palletIndex, `Pallet ${rackId} - tầng ${i + 1}`);
          palletIndex++;
        }

        group.add(g);
      };

      zones.forEach((zone, zi) => {
        const zoneWidth = racksPerRow * rackSpacing + 20;
        const zoneHeight = rowsPerZone * rowSpacing + 20;
        const zoneX = (racksPerRow * rackSpacing) / 2;
        const zoneZ = zi * zoneSpacing + (rowsPerZone * rowSpacing) / 2;

        const zoneColor = new THREE.Color(`hsl(${zi * 40}, 60%, 85%)`);
        const zoneMaterial = new THREE.MeshBasicMaterial({
          color: zoneColor,
          side: THREE.DoubleSide,
          transparent: true,
          opacity: 0.2
        });

        const zoneGeometry = new THREE.PlaneGeometry(zoneWidth, zoneHeight);
        const zoneMesh = new THREE.Mesh(zoneGeometry, zoneMaterial);
        zoneMesh.rotation.x = -Math.PI / 2;
        zoneMesh.position.set(zoneX, 0.01, zoneZ);
        this.scene.add(zoneMesh);
        this.zoneMeshes.push(zoneMesh);

        const label = new SpriteText(`Zone ${zone}`);
        label.position.set(zoneX, 5, zoneZ);
        label.color = '#000000';
        label.textHeight = 4;
        this.scene.add(label);

        // Khung zone
        const borderGeometry = new THREE.BufferGeometry();
        const halfW = zoneWidth / 2;
        const halfH = zoneHeight / 2;
        const vertices = new Float32Array([
          -halfW, 0.02, -halfH,  halfW, 0.02, -halfH,
           halfW, 0.02, -halfH,  halfW, 0.02,  halfH,
           halfW, 0.02,  halfH, -halfW, 0.02,  halfH,
          -halfW, 0.02,  halfH, -halfW, 0.02, -halfH,
        ]);
        borderGeometry.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
        const borderMaterial = new THREE.LineBasicMaterial({ color: 0x888888 });
        const border = new THREE.LineSegments(borderGeometry, borderMaterial);
        border.position.set(zoneX, 0, zoneZ);
        this.scene.add(border);
        this.zoneBorders.push(border);

        for (let row = 0; row < rowsPerZone; row++) {
          for (let col = 0; col < racksPerRow; col++) {
            const x = col * rackSpacing;
            const y = 0;
            const z = zi * zoneSpacing + row * rowSpacing;
            const rackId = `${zone}-${row + 1}-${col + 1}`;
            createRack(x, y, z, rackId);
          }
        }
      });

      this.scene.add(group);
    },

    onMouseMove(event) {
      const rect = this.renderer.domElement.getBoundingClientRect();
      this.mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      this.mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

      this.raycaster.setFromCamera(this.mouse, this.camera);
      const intersects = this.raycaster.intersectObjects(this.zoneMeshes);

      if (intersects.length > 0) {
        const hovered = this.zoneMeshes.indexOf(intersects[0].object);
        this.zoneMeshes.forEach((zone, i) => {
          zone.material.opacity = i === hovered ? 0.5 : 0.2;
        });
        this.hoveredZoneIndex = hovered;
      } else {
        this.zoneMeshes.forEach(zone => {
          zone.material.opacity = 0.2;
        });
        this.hoveredZoneIndex = null;
      }
    },

    onClick(event) {
      const rect = this.renderer.domElement.getBoundingClientRect();
      this.mouse.x = ((event.clientX - rect.left) / rect.width) * 2 - 1;
      this.mouse.y = -((event.clientY - rect.top) / rect.height) * 2 + 1;

      this.raycaster.setFromCamera(this.mouse, this.camera);
      const intersects = this.raycaster.intersectObject(this.palletMesh);

      if (intersects.length > 0) {
        const instanceId = intersects[0].instanceId;
        const info = this.palletInfoMap.get(instanceId);
        if (info) {
          alert(info);
        }
      }
    },

    animate() {
      requestAnimationFrame(this.animate);
      this.renderer.render(this.scene, this.camera);
    },

    onWindowResize() {
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
    }
  }
};
</script>

<style scoped>
canvas {
  display: block;
}
</style>
