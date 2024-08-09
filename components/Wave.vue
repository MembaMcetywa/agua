<template>
    <div ref="sceneContainer" class="scene-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref(null);
let scene, camera, renderer, plane, geometry, pointLight, ambientLight;

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

    // Load texture
    const textureLoader = new THREE.TextureLoader();
    const texture = textureLoader.load('https://threejs.org/examples/textures/water.jpg');

    // Material
    const material = new THREE.MeshStandardMaterial({
        map: texture,
        displacementMap: texture,
        displacementScale: 10,
        metalness: 0.5,
        roughness: 0.5,
        wireframe: false,
    });

    // Mesh
    plane = new THREE.Mesh(geometry, material);
    scene.add(plane);

    // Lighting
    pointLight = new THREE.PointLight(0xffffff, 1);
    pointLight.position.set(50, 50, 50);
    scene.add(pointLight);

    ambientLight = new THREE.AmbientLight(0x404040, 2); 
    scene.add(ambientLight);

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

    // Rotate the plane for a dynamic effect
    plane.rotation.z += 0.001;

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
