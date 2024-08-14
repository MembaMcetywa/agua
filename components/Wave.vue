<template>
    <div ref="sceneContainer" class="scene-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref(null);
const isMorphingToSecondShape = ref(true);
let scene, camera, renderer, plane, geometry1, geometry2, lerpAmount = 0;

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
    //shapes
    geometry1 = new THREE.PlaneGeometry(150, 150, 50, 50);
    geometry2 = new THREE.PlaneGeometry(150, 150, 50, 50);

    const positionAttribute2 = geometry2.attributes.position;
    for (let i = 0; i < positionAttribute2.count; i++) {
        const x = positionAttribute2.getX(i);
        const y = positionAttribute2.getY(i);
        const z = Math.sin(x * 0.2) * 5 + Math.sin(y * 0.2) * 5;
        positionAttribute2.setZ(i, z);
    }

    plane = new THREE.Mesh(geometry1, new THREE.MeshStandardMaterial({
        map: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementMap: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementScale: 10,
        metalness: 0.5,
        roughness: 0.5,
    }));

    scene.add(plane);

    // Lighting
    const pointLight = new THREE.PointLight(0xffffff, 1);
    pointLight.position.set(50, 50, 50);
    scene.add(pointLight);

    const ambientLight = new THREE.AmbientLight(0x404040, 2);
    scene.add(ambientLight);

    // Resize event
    window.addEventListener('resize', onWindowResize, false);

    
    setInterval(() => {
        isMorphingToSecondShape.value = !isMorphingToSecondShape.value;
        lerpAmount = 0;
    }, 5000);
};

const animate = () => {
    requestAnimationFrame(animate);

    const positionAttribute1 = geometry1.attributes.position;
    const positionAttribute2 = geometry2.attributes.position;
    const positionAttributeCurrent = plane.geometry.attributes.position;

    for (let i = 0; i < positionAttributeCurrent.count; i++) {
        const z1 = positionAttribute1.getZ(i);
        const z2 = positionAttribute2.getZ(i);
        const zCurrent = THREE.MathUtils.lerp(z1, z2, lerpAmount);

        positionAttributeCurrent.setZ(i, zCurrent);
    }

    lerpAmount += 0.01;
    if (lerpAmount >= 1) lerpAmount = 1; 

    positionAttributeCurrent.needsUpdate = true;

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
