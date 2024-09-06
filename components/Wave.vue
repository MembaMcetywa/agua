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
const isMicrophoneAccessGranted = ref(false);
const isAudioInitialized = ref(false);

let scene1, scene2, camera, renderer, plane1, plane2;
let currentScene = 1;

let audioContext, analyser, dataArray, bufferLength;
let smoothedBass = 0, smoothedMid = 0, smoothedTreble = 0;

let blendFactor = 0;
let transitioning = false;
let transitionSpeed = 0.02;

let water, waterMesh, particleSystem, waveInterferenceMesh; 

const createRipple = (x, y, intensity) => {
    const rippleGeometry = new THREE.CircleGeometry(1, 32);
    const rippleMaterial = new THREE.MeshBasicMaterial({
        color: 0xffffff,
        transparent: true,
        opacity: 0.2,
    });
    const ripple = new THREE.Mesh(rippleGeometry, rippleMaterial);
    ripple.position.set(x, y, 0.1);
    scene1.add(ripple);

    const rippleAnimate = () => {
        ripple.scale.x += 0.05;
        ripple.scale.y += 0.05;
        ripple.material.opacity -= 0.01;
        if (ripple.material.opacity <= 0) {
            scene1.remove(ripple);
        } else {
            requestAnimationFrame(animate);
        }
    };
    rippleAnimate();
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

const updateWaterEffect = (time, audioData) => {
    if (!waterMesh || !waterMesh.geometry) {
        console.error('waterMesh or its geometry is undefined');
        return;
    }
    const positions = waterMesh.geometry.attributes.position;
    for (let i = 0; i < positions.count; i++) {
        const x = positions.getX(i);
        const y = positions.getY(i);
        const amp = 5;
        const freq = 0.05;
        const z = amp * Math.sin(freq * (x + time)) * Math.cos(freq * (y + time));
        positions.setZ(i, z + audioData[i % audioData.length] * 0.1);
    }
    positions.needsUpdate = true;

    // Update water color based on audio
    const r = Math.sin(time * 0.5) * 0.5 + 0.5;
    const g = Math.sin(time * 0.3) * 0.5 + 0.5;
    const b = Math.sin(time * 0.7) * 0.5 + 0.5;
    waterMesh.material.color.setRGB(r, g, b);

};

const createWaveInterference = () => {
    const waveGeometry = new THREE.PlaneGeometry(200, 200, 100, 100);
    const waveMaterial = new THREE.MeshPhongMaterial({
        color: 0x0077be,
        wireframe: true,
    });
    const waveMesh = new THREE.Mesh(waveGeometry, waveMaterial);
    scene1.add(waveMesh);

    return (time, audioData) => {
        if (!waveMesh.geometry.attributes.position) {
            console.error("waveMesh geometry position attribute is missing.");
            return;
        }

        const positions = waveMesh.geometry.attributes.position;
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
};


// const createParticleSystem = () => {
//     const particleCount = 100000;
//     const particles = new THREE.BufferGeometry();
//     const positions = new Float32Array(particleCount * 3);

//     for (let i = 0; i < particleCount * 3; i++) {
//         positions[i] = (Math.random() - 0.5) * 200;
//     }

//     particles.setAttribute('position', new THREE.BufferAttribute(positions, 3));

//     // Load the disc texture from the Three.js examples
//     const textureLoader = new THREE.TextureLoader();
//     const texture = textureLoader.load('https://threejs.org/examples/textures/sprites/disc.png');

//     const particleMaterial = new THREE.PointsMaterial({
//         color: 0x00ffff,
//         size: 0.5,
//         transparent: true,
//         opacity: 0.5,
//         map: texture,
//         depthWrite: false // Prevent particles from being occluded by each other.
//     });

//     const particleSystem = new THREE.Points(particles, particleMaterial);
//     scene1.add(particleSystem);

//     return (audioData) => {
//         const positions = particles.attributes.position.array;
//         for (let i = 0; i < particleCount; i++) {
//             const i3 = i * 3;
//             positions[i3 + 1] += (audioData[i % audioData.length] * 0.1) - 0.05;
//             if (positions[i3 + 1] > 100) positions[i3 + 1] = -100;
//         }
//         particles.attributes.position.needsUpdate = true;
//     };
// };


// const updateWaterColor = (bassLevel, midLevel, trebleLevel) => {
//     const r = Math.floor(bassLevel);
//     const g = Math.floor(midLevel);
//     const b = Math.floor(trebleLevel);
//     if (!water || !water.material) {
//         console.error('water or its material is undefined');
//         return;
//     }
//     water.material.uniforms.waterColor.value.setRGB(r / 255, g / 255, b / 255);
// };

const setupAudioAnalyser = async () => {
    if (audioContext.state === 'suspended') {
        await audioContext.resume();
    }

    if(!analyser) {
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

const updateRipples = (bass) => {
    if (bass > 150) {
        createRipple(Math.random() * 200 - 100, Math.random() * 200 - 100, bass);
    }
};

const updateWaveInterference = (time, audioData) => {
    if (!waveInterferenceMesh || !waveInterferenceMesh.geometry) {
        console.error('waveInterferenceMesh or its geometry is undefined');
        return;
    }
    const positions = waveInterferenceMesh.geometry.attributes.position;
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

const updateParticles = (audioData) => {
    if (!particleSystem || !particleSystem.geometry) {
        console.error('particleSystem or its geometry is undefined');
        return;
    }
    const positions = particleSystem.geometry.attributes.position.array;
    for (let i = 0; i < positions.length; i += 3) {
        positions[i + 1] += (audioData[i % audioData.length] * 0.1) - 0.05;
        if (positions[i + 1] > 100) positions[i + 1] = -100;
    }
    particleSystem.geometry.attributes.position.needsUpdate = true;
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
        displacementScale: 20,
        metalness: 0.5,
        roughness: 0.5,
    });
    plane2 = new THREE.Mesh(geometry2, material2);
    scene2.add(plane2);

    // Lighting
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
    camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.z = 50;

    // Renderer
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    sceneContainer.value.appendChild(renderer.domElement);

    createWaveInterference();
   // createParticleSystem();
    createWaterEffect();
    
    window.addEventListener('resize', onWindowResize, false);
};

const easeInOutQuad = (t) => {
    return t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t;
};


const animate = () => {
    requestAnimationFrame(animate);

    const { bass, mid, treble } = calculateAudioLevels();

    updateRipples(bass);
    if (waveInterferenceMesh) updateWaveInterference(performance.now() * 0.001, dataArray);
    if (particleSystem) updateParticles(dataArray);
    if (waterMesh) updateWaterEffect(performance.now() * 0.001, dataArray);
    if (water) updateWaterColor(bass, mid, treble);

    const positionAttribute1 = plane1.geometry.attributes.position;
    const positionAttribute2 = plane2.geometry.attributes.position;

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

    renderer.setScissor(0, 0, window.innerWidth * (1 - easedBlend), window.innerHeight);
    renderer.render(scene1, camera);

    renderer.setScissor(window.innerWidth * (1 - easedBlend), 0, window.innerWidth * easedBlend, window.innerHeight);
    renderer.render(scene2, camera);

    renderer.setScissorTest(false);

    if (currentScene === 1 && bass > 120 && !transitioning) {
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
        setupAudioAnalyser();
        animate()
    }
};

const handleUserInteraction = () => {
    initAudioContext();
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
    cancelAnimationFrame(animate)
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