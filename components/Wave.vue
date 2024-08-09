<template>
    <div ref="sceneContainer" class="scene-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref(null);
let scene, camera, renderer, plane, geometry;

const init = () => {
    // Scene
    scene = new THREE.Scene();

    // Camera
    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.z = 50;

    // Renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    sceneContainer.value.appendChild(renderer.domElement);

    // Geometry
    geometry = new THREE.PlaneGeometry(150, 150, 50, 50);

    // Material
    const material = new THREE.MeshBasicMaterial({ color: 0x45728d, wireframe: true });

    // Mesh
    plane = new THREE.Mesh(geometry, material);
    scene.add(plane);

    // Resize event
    window.addEventListener('resize', onWindowResize, false);
};

const animate = () => {
    requestAnimationFrame(animate);

    // Animate vertices
    const time = Date.now() * 0.001;
    const positionAttribute = geometry.attributes.position;

    for (let i = 0; i < positionAttribute.count; i++) {
        const x = positionAttribute.getX(i);
        const y = positionAttribute.getY(i);
        const z = Math.sin(x * 0.5 + time) * 2 + Math.sin(y * 0.5 + time) * 2;
        positionAttribute.setZ(i, z);
    }

    positionAttribute.needsUpdate = true;
    renderer.render(scene, camera);
};

const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
    init();
    animate();
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
});
</script>

<style scoped>
.scene-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
}
</style>
