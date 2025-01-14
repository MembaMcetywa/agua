<template>
    <div>
        <div ref="sceneContainer" class="scene-container"></div>
        <div v-if="!audioContext" class="start-message">
            Click anywhere to start visualising sound
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';

const sceneContainer = ref < HTMLElement | null > (null);

let scene1: THREE.Scene;
let scene2: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let plane1: THREE.Mesh;
let plane2: THREE.Mesh;
let waveInterferenceMesh: THREE.Mesh | undefined;
let waterMesh: THREE.Mesh | undefined;
let particleSystem: THREE.Points | undefined;

// Scenes switching
let currentScene = 1;
let transitioning = false;
let transitionSpeed = 0.02;
let blendFactor = 0;

// Audio
let audioContext: AudioContext | undefined;
let analyser: AnalyserNode | undefined;
let bufferLength = 0;
let dataArray: Uint8Array | undefined;

// Smoothed values
let smoothedBass = 0;
let smoothedMid = 0;
let smoothedTreble = 0;

let rafId: number | null = null;

const ripples: THREE.Mesh[] = [];

const createRipple = (x: number, y: number, intensity: number) => {
    const rippleGeometry = new THREE.CircleGeometry(1, 32);
    const rippleMaterial = new THREE.MeshBasicMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.2,
    });
    const ripple = new THREE.Mesh(rippleGeometry, rippleMaterial);
    ripple.position.set(x, y, 0.1);
    scene1.add(ripple);

    ripples.push(ripple);
};

const updateRipples = () => {
    for (let i = ripples.length - 1; i >= 0; i--) {
        const ripple = ripples[i];
        ripple.scale.x += 0.05;
        ripple.scale.y += 0.05;

        const mat = ripple.material as THREE.MeshBasicMaterial;
        mat.opacity -= 0.01;

        if (mat.opacity <= 0) {
            scene1.remove(ripple);
            ripples.splice(i, 1);
        }
    }
};

const createWaterEffect = () => {
    const waterGeometry = new THREE.PlaneGeometry(200, 200, 50, 50);
    const waterMaterial = new THREE.MeshPhongMaterial({
        color: 0x0077be,
        wireframe: false,
        side: THREE.DoubleSide,
        transparent: true,
        opacity: 0.6,
    });
    waterMesh = new THREE.Mesh(waterGeometry, waterMaterial);
    waterMesh.rotation.x = -Math.PI / 2;
    scene1.add(waterMesh);
};

const updateWaterEffect = (time: number, audioData: Uint8Array) => {
    if (!waterMesh || !waterMesh.geometry) return;

    const positions = (waterMesh.geometry.attributes.position as THREE.BufferAttribute);
    for (let i = 0; i < positions.count; i++) {
        const x = positions.getX(i);
        const y = positions.getY(i);
        const amp = 5;
        const freq = 0.05;

        const z = amp * Math.sin(freq * (x + time)) * Math.cos(freq * (y + time));
        positions.setZ(i, z + audioData[i % audioData.length] * 0.1);
    }
    positions.needsUpdate = true;

 
    const r = Math.sin(time * 0.5) * 0.5 + 0.5;
    const g = Math.sin(time * 0.3) * 0.5 + 0.5;
    const b = Math.sin(time * 0.7) * 0.5 + 0.5;
    (waterMesh.material as THREE.MeshPhongMaterial).color.setRGB(r, g, b);
};

const createWaveInterference = () => {
    const waveGeometry = new THREE.PlaneGeometry(200, 200, 100, 100);
    const waveMaterial = new THREE.MeshPhongMaterial({
        color: 0x0077be,
        wireframe: true,
    });
    waveInterferenceMesh = new THREE.Mesh(waveGeometry, waveMaterial);
    scene1.add(waveInterferenceMesh);
};

const updateWaveInterference = (time: number, audioData: Uint8Array) => {
    if (!waveInterferenceMesh || !waveInterferenceMesh.geometry) return;

    const positions = waveInterferenceMesh.geometry.attributes.position as THREE.BufferAttribute;
    for (let i = 0; i < positions.count; i++) {
        const x = positions.getX(i);
        const y = positions.getY(i);
        const amp = 5;
        const freq = 0.05;
        const z = amp * Math.sin(freq * (x + time)) * Math.cos(freq * (y + time));
        positions.setZ(i, z + audioData[i % audioData.length] * 0.05);
    }
    positions.needsUpdate = true;
};

const updateParticles = (audioData: Uint8Array) => {
    if (!particleSystem || !particleSystem.geometry) return;

    const positions = particleSystem.geometry.attributes.position.array as Float32Array;
    for (let i = 0; i < positions.length; i += 3) {
        positions[i + 1] += (audioData[i % audioData.length] * 0.1) - 0.05;
        if (positions[i + 1] > 100) {
            positions[i + 1] = -100;
        }
    }
    particleSystem.geometry.attributes.position.needsUpdate = true;
};

const triggerRippleIfNeeded = (bass: number) => {
    if (bass > 150) {
        createRipple(
            Math.random() * 200 - 100,
            Math.random() * 200 - 100,
            bass
        );
    }
};

const calculateAudioLevels = () => {
    if (!analyser || !dataArray) return { bass: 0, mid: 0, treble: 0 };

    analyser.getByteFrequencyData(dataArray);

    const bassEnd = Math.floor(bufferLength * 0.1);
    const midEnd = Math.floor(bufferLength * 0.5);

    let bassSum = 0;
    let midSum = 0;
    let trebleSum = 0;

    for (let i = 0; i < bufferLength; i++) {
        if (i < bassEnd) bassSum += dataArray[i];
        else if (i < midEnd) midSum += dataArray[i];
        else trebleSum += dataArray[i];
    }

    const bassLevel = bassSum / bassEnd;
    const midLevel = midSum / (midEnd - bassEnd);
    const trebleLevel = trebleSum / (bufferLength - midEnd);

    smoothedBass = smoothedBass * 0.8 + bassLevel * 0.2;
    smoothedMid = smoothedMid * 0.8 + midLevel * 0.2;
    smoothedTreble = smoothedTreble * 0.8 + trebleLevel * 0.2;

    return { bass: smoothedBass, mid: smoothedMid, treble: smoothedTreble };
};

const easeInOutQuad = (t: number) => {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
};
const animate = () => {
   
    rafId = requestAnimationFrame(animate);

    const { bass, mid, treble } = calculateAudioLevels();
    if (!dataArray) return;

    triggerRippleIfNeeded(bass);

    updateRipples();

    if (waveInterferenceMesh) {
        updateWaveInterference(performance.now() * 0.001, dataArray);
    }

    if (particleSystem) {
        updateParticles(dataArray);
    }
    if (waterMesh) {
        updateWaterEffect(performance.now() * 0.001, dataArray);
    }

    // Drive planes with the bass, mid, treble
    if (plane1 && plane2) {
        const positionAttribute1 = plane1.geometry.attributes.position as THREE.BufferAttribute;
        const positionAttribute2 = plane2.geometry.attributes.position as THREE.BufferAttribute;

        for (let i = 0; i < positionAttribute1.count; i++) {
            const x = positionAttribute1.getX(i);
            const y = positionAttribute1.getY(i);

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
    }
    //cross fade transition
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

    renderer.autoClear = false;
    renderer.clear();
    renderer.setScissorTest(true);

    renderer.setScissor(
        0,
        0,
        window.innerWidth * (1 - easedBlend),
        window.innerHeight
    );
    renderer.render(scene1, camera);

    renderer.setScissor(
        window.innerWidth * (1 - easedBlend),
        0,
        window.innerWidth * easedBlend,
        window.innerHeight
    );
    renderer.render(scene2, camera);

    renderer.setScissorTest(false);

    // Decide when to switch scenes
    if (currentScene === 1 && !transitioning) {
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

const setupAudioAnalyser = async () => {
    if (!audioContext) return;

    if (audioContext.state === 'suspended') {
        await audioContext.resume();
    }

    if (!analyser) {
        analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        bufferLength = analyser.frequencyBinCount;
        dataArray = new Uint8Array(bufferLength);
    }

    try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        const source = audioContext.createMediaStreamSource(stream);
        source.connect(analyser);
    } catch (err) {
        console.error('Error accessing microphone:', err);
    }
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
        displacementScale: 20,
        metalness: 0.5,
        roughness: 0.5,
    });
    plane2 = new THREE.Mesh(geometry2, material2);
    scene2.add(plane2);

    // Lights
    const pointLight1 = new THREE.PointLight(0xffffff, 1);
    pointLight1.position.set(50, 50, 50);
    scene1.add(pointLight1);

    const ambientLight1 = new THREE.AmbientLight(0x404040, 2);
    scene1.add(ambientLight1);

    const pointLight2 = new THREE.PointLight(0xffffff, 1);
    pointLight2.position.set(70, 50, 60);
    scene2.add(pointLight2);

    const ambientLight2 = new THREE.AmbientLight(0x404040, 2);
    scene2.add(ambientLight2);

    // Camera
    camera = new THREE.PerspectiveCamera(
        75,
        window.innerWidth / window.innerHeight,
        0.1,
        1000
    );
    camera.position.z = 50;

    // Renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    if (sceneContainer.value) {
        sceneContainer.value.appendChild(renderer.domElement);
    }

   
    createWaveInterference();
    createWaterEffect();

    window.addEventListener('resize', onWindowResize, false);
};

const initAudioContext = () => {
    if (!audioContext) {
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        setupAudioAnalyser();
        animate();
    }
};
const handleUserInteraction = () => {
    initAudioContext();
    document.removeEventListener('click', handleUserInteraction);
};

onMounted(() => {
    init();
    document.addEventListener('click', handleUserInteraction);
});

onBeforeUnmount(() => {
    window.removeEventListener('resize', onWindowResize);
    document.removeEventListener('click', handleUserInteraction);

  
    if (rafId !== null) {
        cancelAnimationFrame(rafId);
    }

    if (audioContext) {
        audioContext.close();
    }
});
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@100;400;700&display=swap');

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