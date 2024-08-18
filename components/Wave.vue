<template>
    <div ref="sceneContainer" class="scene-container"></div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref(null);
let scene1, scene2, camera, renderer, plane1, plane2;
let currentScene = 1;

let audioContext, analyser, dataArray, bufferLength;
let smoothedEnergy = 0;

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
    return sum / bufferLength;
};

const init = () => {
    // Scene 1
    scene1 = new THREE.Scene();
    const geometry1 = new THREE.PlaneGeometry(150, 150, 50, 50);
    const material1 = new THREE.MeshStandardMaterial({
        map: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementMap: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementScale: 10,
        metalness: 0.5,
        roughness: 0.5,
    });
    plane1 = new THREE.Mesh(geometry1, material1);
    scene1.add(plane1);

    scene2 = new THREE.Scene();
    const geometry2 = new THREE.PlaneGeometry(150, 150, 50, 50);
    const material2 = new THREE.MeshStandardMaterial({
        map: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementMap: new THREE.TextureLoader().load('https://threejs.org/examples/textures/water.jpg'),
        displacementScale: 15,
        metalness: 0.5,
        roughness: 0.5,
    });
    plane2 = new THREE.Mesh(geometry2, material2);
    scene2.add(plane2);

    // Lighting for both scenes
    const pointLight1 = new THREE.PointLight(0xffffff, 1);
    pointLight1.position.set(50, 50, 50);
    scene1.add(pointLight1);

    const ambientLight1 = new THREE.AmbientLight(0x404040, 2);
    scene1.add(ambientLight1);

    const pointLight2 = new THREE.PointLight(0xffffff, 1);
    pointLight2.position.set(50, 50, 50);
    scene2.add(pointLight2);

    const ambientLight2 = new THREE.AmbientLight(0x404040, 2);
    scene2.add(ambientLight2);

    // Camera
    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.z = 50;

    // Renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    sceneContainer.value.appendChild(renderer.domElement);

    // Resize event
    window.addEventListener('resize', onWindowResize, false);
};

// Easing function for smooth transitions
const easeInOutQuad = (t) => {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
};

let blendFactor = 0;
let transitioning = false;
let transitionSpeed = 0.02;

const animate = () => {
    requestAnimationFrame(animate);

    // Get frequency data
    analyser.getByteFrequencyData(dataArray);

    const audioEnergy = calculateAudioEnergy();

    smoothedEnergy = smoothedEnergy * 0.8 + audioEnergy * 0.2;

    const positionAttribute1 = plane1.geometry.attributes.position;
    const positionAttribute2 = plane2.geometry.attributes.position;

    for (let i = 0; i < positionAttribute1.count; i++) {
        const frequencyIndex = Math.floor((i / positionAttribute1.count) * bufferLength);
        const z = (dataArray[frequencyIndex] * 0.1) + (smoothedEnergy * 0.1);

        positionAttribute1.setZ(i, z);
        positionAttribute2.setZ(i, z);
    }

    positionAttribute1.needsUpdate = true;
    positionAttribute2.needsUpdate = true;

    plane1.rotation.z += 0.001 + (smoothedEnergy / 128) * 0.05;
    plane2.rotation.z += 0.001 + (smoothedEnergy / 128) * 0.05;

    // scene transition
    if (transitioning) {
        blendFactor += transitionSpeed;
        if (blendFactor >= 1) {
            blendFactor = 1;
            transitioning = false;
        }
    } else {
        blendFactor -= transitionSpeed;
        if (blendFactor <= 0) {
            blendFactor = 0;
        }
    }

    const easedBlend = easeInOutQuad(blendFactor);

    // Render the blended scene
    renderer.autoClear = false;
    renderer.clear();
    renderer.setScissorTest(true);

    renderer.setScissor(0, 0, window.innerWidth * (1 - easedBlend), window.innerHeight);
    renderer.render(scene1, camera);

    renderer.setScissor(window.innerWidth * (1 - easedBlend), 0, window.innerWidth * easedBlend, window.innerHeight);
    renderer.render(scene2, camera);

    renderer.setScissorTest(false);

    if (currentScene === 1 && smoothedEnergy > 50 && !transitioning) {
        transitioning = true;
        currentScene = 2;
    } else if (currentScene === 2 && smoothedEnergy < 30 && !transitioning) {
        transitioning = true;
        currentScene = 1;
    }
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
