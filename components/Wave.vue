<template>
    <div ref="sceneContainer" class="scene-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const audio = ref(null);
const sceneContainer = ref(null);
const isMorphingToSecondShape = ref(true);
let scene, camera, renderer, plane, geometry1, geometry2, lerpAmount = 0;

let audioContext, analyser, dataArray, bufferLength;

const setupAudioAnalyzer = async () => {
    audioContext = new (window.AudioContext || window.webkitAudioContext)();
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 256;
    bufferLength = analyser.frequencyBinCount;
    dataArray = new Uint8Array(bufferLength);

    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const source = audioContext.createMediaStreamSource(stream);
        source.connect(analyser);
        animate();
    } catch (err) {
        console.error('Error accessing microphone:', err);
    }
};

const calculateAudioEnergy = () => {
    let sum = 0;
    for (let i = 0; i < bufferLength; i++) {
        sum += dataArray[i];
    }
    return sum / bufferLength; // Average amplitude across all frequencies
};

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
};

let smoothedEnergy = 0; // Initialize smoothed energy

const animate = () => {
    requestAnimationFrame(animate);

    // Get frequency data
    analyser.getByteFrequencyData(dataArray);

    // Calculate overall audio energy
    const audioEnergy = calculateAudioEnergy();

    // Smooth the audio energy to avoid abrupt changes
    smoothedEnergy = smoothedEnergy * 0.8 + audioEnergy * 0.2;

    const positionAttribute = plane.geometry.attributes.position;
    for (let i = 0; i < positionAttribute.count; i++) {
        const x = positionAttribute.getX(i);
        const y = positionAttribute.getY(i);

        const frequencyIndex = Math.floor((i / positionAttribute.count) * bufferLength);
        const z = (dataArray[frequencyIndex] * 0.1) + (smoothedEnergy * 0.1);

        positionAttribute.setZ(i, z);
    }

    positionAttribute.needsUpdate = true;

    const rotationSpeed = 0.001 + (smoothedEnergy / 128) * 0.05;
    plane.rotation.z += rotationSpeed;

    renderer.render(scene, camera);
};



const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};

onMounted(() => {
    init();
    setupAudioAnalyzer();
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    if (audioContext) {
        audioContext.close();
    }
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
