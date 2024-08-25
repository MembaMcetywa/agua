<template>
    <div>
        <div ref="sceneContainer" class="scene-container"></div>
        <div v-if="!audioContext" class="start-message">
            Click anywhere to start visualising sound
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref(null);
let scene1, scene2, camera, renderer, plane1, plane2;
let currentScene = 1;

let audioContext, analyser, dataArray, bufferLength;
let smoothedBass = 0, smoothedMid = 0, smoothedTreble = 0;

const setupAudioAnalyzer = async () => {
    if (audioContext.state === 'suspended') {
        await audioContext.resume();
    }

    audioContext = new (window.AudioContext || window.webkitAudioContext)();
    analyser = audioContext.createAnalyser();
    analyser.fftSize = 2048;  // Increased for better frequency resolution
    bufferLength = analyser.frequencyBinCount;
    dataArray = new Uint8Array(bufferLength);

    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const source = audioContext.createMediaStreamSource(stream);
        source.connect(analyser);
        animate();
    } catch (err) {
        console.error('Error accessing microphone:', err);
        // Fallback: use demo audio file
        // useDemoAudio();
    }
};

const useDemoAudio = async () => {
    const audio = new Audio('/path/to/demo/audio.mp3');
    const source = audioContext.createMediaElementSource(audio);
    source.connect(analyser);
    source.connect(audioContext.destination);
    audio.play();
};

const calculateAudioLevels = () => {
    analyser.getByteFrequencyData(dataArray);


    const bassEnd = Math.floor(bufferLength * 0.1);
    const midEnd = Math.floor(bufferLength * 0.5);

    let bassSum = 0, midSum = 0, trebleSum = 0;

    for (let i = 0; i < bufferLength; i++) {
        if (i < bassEnd) bassSum += dataArray[i];
        else if (i < midEnd) midSum += dataArray[i];
        else trebleSum += dataArray[i];
    }

    const bassLevel = bassSum / bassEnd;
    const midLevel = midSum / (midEnd - bassEnd);
    const trebleLevel = trebleSum / (bufferLength - midEnd);

    // Apply smoothing
    smoothedBass = smoothedBass * 0.8 + bassLevel * 0.2;
    smoothedMid = smoothedMid * 0.8 + midLevel * 0.2;
    smoothedTreble = smoothedTreble * 0.8 + trebleLevel * 0.2;

    return { bass: smoothedBass, mid: smoothedMid, treble: smoothedTreble };
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

    // Scene 2
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

    const { bass, mid, treble } = calculateAudioLevels();

    const positionAttribute1 = plane1.geometry.attributes.position;
    const positionAttribute2 = plane2.geometry.attributes.position;

    for (let i = 0; i < positionAttribute1.count; i++) {
        const x = positionAttribute1.getX(i);
        const y = positionAttribute1.getY(i);

        // Use different frequency ranges for different areas of the plane
        let z;
        if (Math.abs(x) < 50 && Math.abs(y) < 50) {
            z = bass * 0.2;
        } else if (Math.abs(x) < 100 && Math.abs(y) < 100) {
            z = mid * 0.15;
        } else {
            z = treble * 0.1;
        }

        positionAttribute1.setZ(i, z);
        positionAttribute2.setZ(i, z);
    }

    positionAttribute1.needsUpdate = true;
    positionAttribute2.needsUpdate = true;

    plane1.rotation.z += 0.001 + (bass / 128) * 0.05;
    plane2.rotation.z += 0.001 + (bass / 128) * 0.05;

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

    if (currentScene === 1 && bass > 100 && !transitioning) {
        transitioning = true;
        currentScene = 2;
    } else if (currentScene === 2 && bass < 50 && !transitioning) {
        transitioning = true;
        currentScene = 1;
    }
};

const onWindowResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
};

const initAudioContext = () => {
    if (!audioContext) {
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        setupAudioAnalyzer();
    }
};

const handleUserInteraction = () => {
    initAudioContext();
    // Remove the event listener after the first interaction
    document.removeEventListener('click', handleUserInteraction);
};

onMounted(() => {
    init();
    document.addEventListener('click', handleUserInteraction)
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    document.removeEventListener('click', handleUserInteraction);
    if (audioContext) {
        audioContext.close();
    }
});
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Poppins:ital,wght@0,100;0,200;0,300;0,400;0,500;0,600;0,700;0,800;0,900;1,100;1,200;1,300;1,400;1,500;1,600;1,700;1,800;1,900&display=swap");

.scene-container {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 0;
}

.start-message {
    font-family: Poppins, sans-serif;
}
</style>