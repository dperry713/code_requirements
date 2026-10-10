# Three.js Interactive 3D Development
### Production-Grade Guide with Complete Advanced Examples

**Target**: Three.js r186+ (WebGLRenderer + WebGPURenderer / TSL)  
**Style**: Modern ES modules, import maps, production-ready patterns, performance-first  
**Focus**: Interactive 3D designs — scenes that respond to user input, scroll, pointer, and real-time data

---

## Table of Contents

1. [Modern Project Setup](#1-modern-project-setup)
2. [Core Scene Architecture](#2-core-scene-architecture)
3. [Asset Loading & Optimization](#3-asset-loading--optimization)
4. [Materials, Lighting & Physically Based Rendering](#4-materials-lighting--physically-based-rendering)
5. [Instancing & Batched Rendering](#5-instancing--batched-rendering)
6. [Custom Shaders & TSL (Three.js Shading Language)](#6-custom-shaders--tsl-threejs-shading-language)
7. [Post-Processing Pipelines](#7-post-processing-pipelines)
8. [Interactive Systems (Pointer, Raycasting, GPU Picking)](#8-interactive-systems-pointer-raycasting-gpu-picking)
9. [Animation, Scroll-Driven & Timeline Systems](#9-animation-scroll-driven--timeline-systems)
10. [Performance, Memory & Production Hardening](#10-performance-memory--production-hardening)
11. [WebGPU, Compute Shaders & Advanced Effects](#11-webgpu-compute-shaders--advanced-effects)
12. [Gaussian Splatting & Emerging Techniques](#12-gaussian-splatting--emerging-techniques)
13. [Architecture Patterns for Large Interactive Apps](#13-architecture-patterns-for-large-interactive-apps)
14. [Final Recommendations & Checklist](#14-final-recommendations--checklist)

---

## 1. Modern Project Setup

### 1.1 Recommended Stack (2026)

- **Module system**: ES modules + import maps (or Vite / esbuild)
- **Three.js**: `three@0.186.0` or newer
- **Addons**: Always import from `three/addons/`
- **Renderer choice**:
  - `WebGLRenderer` — maximum compatibility, mature ecosystem
  - `WebGPURenderer` — preferred for new interactive experiences that need compute, TSL, or high particle counts

### 1.2 Minimal Production HTML + Import Map

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Interactive Three.js Scene</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { width: 100%; height: 100%; overflow: hidden; background: #0a0a0a; }
    canvas { display: block; width: 100%; height: 100%; touch-action: none; }
    #ui { position: absolute; top: 16px; left: 16px; color: #fff; font-family: system-ui; pointer-events: none; }
  </style>
  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.186.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.186.0/examples/jsm/"
    }
  }
  </script>
</head>
<body>
  <div id="ui">Interactive 3D — production template</div>
  <script type="module" src="./main.js"></script>
</body>
</html>
```

### 1.3 Core Bootstrap (main.js)

```js
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { RGBELoader } from 'three/addons/loaders/RGBELoader.js';
import Stats from 'three/addons/libs/stats.module.js';

class App {
  constructor() {
    this.clock = new THREE.Clock();
    this.container = document.body;
    this.initRenderer();
    this.initScene();
    this.initCamera();
    this.initControls();
    this.initLights();
    this.initHelpers();
    this.onResize = this.onResize.bind(this);
    window.addEventListener('resize', this.onResize);
    this.animate = this.animate.bind(this);
    this.animate();
  }

  initRenderer() {
    this.renderer = new THREE.WebGLRenderer({
      antialias: true,
      powerPreference: 'high-performance',
      alpha: false,
    });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    this.renderer.setSize(window.innerWidth, window.innerHeight);
    this.renderer.outputColorSpace = THREE.SRGBColorSpace;
    this.renderer.toneMapping = THREE.ACESFilmicToneMapping;
    this.renderer.toneMappingExposure = 1.0;
    this.renderer.shadowMap.enabled = true;
    this.renderer.shadowMap.type = THREE.PCFShadowMap; // PCFSoftShadowMap removed in r186
    this.container.appendChild(this.renderer.domElement);
  }

  initScene() {
    this.scene = new THREE.Scene();
    this.scene.background = new THREE.Color(0x0a0a0a);
    this.scene.fog = new THREE.FogExp2(0x0a0a0a, 0.02);
  }

  initCamera() {
    this.camera = new THREE.PerspectiveCamera(
      50,
      window.innerWidth / window.innerHeight,
      0.1,
      200
    );
    this.camera.position.set(4, 3, 8);
  }

  initControls() {
    this.controls = new OrbitControls(this.camera, this.renderer.domElement);
    this.controls.enableDamping = true;
    this.controls.dampingFactor = 0.05;
    this.controls.target.set(0, 1, 0);
    this.controls.update();
  }

  initLights() {
    const ambient = new THREE.AmbientLight(0xffffff, 0.15);
    this.scene.add(ambient);

    const dir = new THREE.DirectionalLight(0xffffff, 2.5);
    dir.position.set(5, 10, 7);
    dir.castShadow = true;
    dir.shadow.mapSize.set(2048, 2048);
    dir.shadow.camera.near = 0.5;
    dir.shadow.camera.far = 50;
    dir.shadow.bias = -0.0001;
    this.scene.add(dir);
    this.dirLight = dir;
  }

  initHelpers() {
    this.stats = new Stats();
    this.stats.dom.style.position = 'absolute';
    this.stats.dom.style.top = '0px';
    this.stats.dom.style.right = '0px';
    document.body.appendChild(this.stats.dom);
  }

  onResize() {
    const w = window.innerWidth;
    const h = window.innerHeight;
    this.camera.aspect = w / h;
    this.camera.updateProjectionMatrix();
    this.renderer.setSize(w, h);
  }

  animate() {
    requestAnimationFrame(this.animate);
    const dt = this.clock.getDelta();
    this.controls.update();
    this.stats.update();
    this.renderer.render(this.scene, this.camera);
  }

  dispose() {
    window.removeEventListener('resize', this.onResize);
    this.controls.dispose();
    this.renderer.dispose();
    // Traverse and dispose geometries/materials/textures (see §10)
  }
}

new App();
```

---

## 2. Core Scene Architecture

### 2.1 Scene Graph Best Practices

- Keep the scene graph shallow where possible.
- Group related objects under a single `THREE.Group` for transforms and disposal.
- Prefer `Object3D` hierarchy over manual matrix updates unless you need raw performance.
- Always call `matrixAutoUpdate = false` on static objects after the first update.

```js
const staticGroup = new THREE.Group();
staticGroup.matrixAutoUpdate = false;
staticGroup.updateMatrix();
scene.add(staticGroup);
```

### 2.2 Camera Management for Interactive Experiences

```js
// Smooth camera transitions (production pattern)
import { gsap } from 'gsap'; // or pure lerp

function focusOnObject(object, camera, controls, duration = 1.2) {
  const box = new THREE.Box3().setFromObject(object);
  const center = box.getCenter(new THREE.Vector3());
  const size = box.getSize(new THREE.Vector3());
  const maxDim = Math.max(size.x, size.y, size.z);
  const fov = camera.fov * (Math.PI / 180);
  const distance = (maxDim / 2) / Math.tan(fov / 2) * 1.4;

  const direction = new THREE.Vector3()
    .subVectors(camera.position, controls.target)
    .normalize();
  const newPos = center.clone().add(direction.multiplyScalar(distance));

  gsap.to(camera.position, {
    x: newPos.x, y: newPos.y, z: newPos.z,
    duration,
    ease: 'power2.inOut',
    onUpdate: () => controls.update(),
  });
  gsap.to(controls.target, {
    x: center.x, y: center.y, z: center.z,
    duration,
    ease: 'power2.inOut',
    onUpdate: () => controls.update(),
  });
}
```

---

## 3. Asset Loading & Optimization

### 3.1 Production Loading Pipeline

Always use compressed formats:

| Asset Type | Format | Loader | Notes |
|------------|--------|--------|-------|
| Meshes | glTF / GLB + Draco | `GLTFLoader` + `DRACOLoader` | Prefer binary GLB |
| Textures | KTX2 (Basis Universal) | `KTX2Loader` | GPU-compressed, far smaller |
| HDR Environments | HDR / EXR | `RGBELoader` / `EXRLoader` | Use for IBL |
| Audio | compressed | `AudioLoader` | Optional spatial audio |

### 3.2 Complete Draco + KTX2 Loader Setup

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';
import { KTX2Loader } from 'three/addons/loaders/KTX2Loader.js';
import { MeshoptDecoder } from 'three/addons/libs/meshopt_decoder.module.js';

function createProductionLoader(renderer) {
  const dracoLoader = new DRACOLoader();
  dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.7/');
  dracoLoader.setDecoderConfig({ type: 'js' }); // or 'wasm'

  const ktx2Loader = new KTX2Loader()
    .setTranscoderPath('https://cdn.jsdelivr.net/npm/three@0.186.0/examples/jsm/libs/basis/')
    .detectSupport(renderer);

  const loader = new GLTFLoader();
  loader.setDRACOLoader(dracoLoader);
  loader.setKTX2Loader(ktx2Loader);
  loader.setMeshoptDecoder(MeshoptDecoder);

  return loader;
}

// Usage
const loader = createProductionLoader(renderer);
loader.load('/models/product.glb', (gltf) => {
  const model = gltf.scene;
  model.traverse((child) => {
    if (child.isMesh) {
      child.castShadow = true;
      child.receiveShadow = true;
      // Optional: convert materials to MeshPhysicalMaterial if needed
    }
  });
  scene.add(model);
}, undefined, (err) => console.error('Load failed', err));
```

### 3.3 Progressive Loading & LOD

```js
import { LOD } from 'three';

const lod = new THREE.LOD();
lod.addLevel(highDetailMesh, 0);
lod.addLevel(mediumDetailMesh, 15);
lod.addLevel(lowDetailMesh, 40);
scene.add(lod);
```

For large scenes, combine with `BatchedMesh` or instancing (see §5).

---

## 4. Materials, Lighting & Physically Based Rendering

### 4.1 Production Material Stack

Prefer `MeshPhysicalMaterial` for realistic surfaces (clearcoat, transmission, sheen, iridescence, anisotropy).

```js
const bodyMaterial = new THREE.MeshPhysicalMaterial({
  color: 0x1a1a2e,
  metalness: 0.9,
  roughness: 0.15,
  clearcoat: 1.0,
  clearcoatRoughness: 0.05,
  reflectivity: 0.9,
  envMapIntensity: 1.2,
  // r186+ retroreflective option available on MeshPhysicalMaterial
});
```

### 4.2 Environment Lighting (IBL)

```js
new RGBELoader().load('/hdr/studio_small_09_1k.hdr', (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.environment = texture;
  scene.background = texture; // or a solid color / gradient
  scene.backgroundBlurriness = 0.4; // soft background
});
```

### 4.3 Modern Directional + Cascaded Shadows (SunLight addon in r186)

```js
// For outdoor interactive scenes, prefer the new SunLight when available
// Classic approach still works:
dirLight.shadow.camera.left = -20;
dirLight.shadow.camera.right = 20;
dirLight.shadow.camera.top = 20;
dirLight.shadow.camera.bottom = -20;
dirLight.shadow.bias = -0.0002;
dirLight.shadow.normalBias = 0.02;
```

---

## 5. Instancing & Batched Rendering

### 5.1 InstancedMesh — Thousands of Identical Objects

```js
const geometry = new THREE.IcosahedronGeometry(0.15, 2);
const material = new THREE.MeshStandardMaterial({
  color: 0x88ccff,
  metalness: 0.4,
  roughness: 0.3,
});

const count = 5000;
const mesh = new THREE.InstancedMesh(geometry, material, count);
mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage); // if updating every frame
mesh.castShadow = true;
mesh.receiveShadow = true;

const dummy = new THREE.Object3D();
const color = new THREE.Color();

for (let i = 0; i < count; i++) {
  dummy.position.set(
    (Math.random() - 0.5) * 40,
    Math.random() * 8,
    (Math.random() - 0.5) * 40
  );
  dummy.rotation.set(Math.random() * Math.PI, Math.random() * Math.PI, 0);
  dummy.scale.setScalar(0.6 + Math.random() * 0.8);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);

  color.setHSL(Math.random(), 0.7, 0.5);
  mesh.setColorAt(i, color);
}

mesh.instanceMatrix.needsUpdate = true;
if (mesh.instanceColor) mesh.instanceColor.needsUpdate = true;
scene.add(mesh);

// Animation example — update a subset every frame
function updateInstances(time) {
  for (let i = 0; i < count; i += 10) { // update every 10th for perf
    mesh.getMatrixAt(i, dummy.matrix);
    dummy.position.y += Math.sin(time + i) * 0.01;
    dummy.updateMatrix();
    mesh.setMatrixAt(i, dummy.matrix);
  }
  mesh.instanceMatrix.needsUpdate = true;
}
```

### 5.2 BatchedMesh — Different Geometries, Same Material (r159+)

```js
import { BatchedMesh } from 'three';

const maxGeometryCount = 100;
const maxVertexCount = 50_000;
const maxIndexCount = 100_000;

const batched = new BatchedMesh(
  maxGeometryCount,
  maxVertexCount,
  maxIndexCount,
  material
);

const geoIds = [];
geoIds.push(batched.addGeometry(boxGeometry));
geoIds.push(batched.addGeometry(sphereGeometry));
geoIds.push(batched.addGeometry(cylinderGeometry));

// Add instances
const id1 = batched.addInstance(geoIds[0]);
batched.setMatrixAt(id1, matrix1);
const id2 = batched.addInstance(geoIds[1]);
batched.setMatrixAt(id2, matrix2);

scene.add(batched);
```

BatchedMesh is ideal for product configurators, UI panels, or architectural pieces that share a material but have unique shapes.

---

## 6. Custom Shaders & TSL (Three.js Shading Language)

### 6.1 Classic onBeforeCompile (still valid for WebGL)

```js
const material = new THREE.MeshStandardMaterial({ color: 0x3388ff });
material.onBeforeCompile = (shader) => {
  shader.uniforms.uTime = { value: 0 };
  shader.vertexShader = `
    uniform float uTime;
    ${shader.vertexShader}
  `.replace(
    '#include <begin_vertex>',
    `
    #include <begin_vertex>
    transformed.y += sin(position.x * 3.0 + uTime) * 0.15;
    `
  );
  material.userData.shader = shader;
};

// In animate loop
if (material.userData.shader) {
  material.userData.shader.uniforms.uTime.value = clock.elapsedTime;
}
```

### 6.2 TSL (Preferred for WebGPU / modern node materials)

```js
import * as THREE from 'three/webgpu';
import { texture, uv, time, sin, positionLocal, vec3, float } from 'three/tsl';

const material = new THREE.MeshStandardNodeMaterial();
material.colorNode = texture(myTexture, uv());
material.positionNode = positionLocal.add(
  vec3(0, sin(positionLocal.x.mul(3).add(time)).mul(0.15), 0)
);

// For compute or advanced nodes see §11
```

TSL benefits:
- Works across WebGL and WebGPU backends
- No string injection / fragile `onBeforeCompile`
- Full type safety and composability via nodes

---

## 7. Post-Processing Pipelines

### 7.1 Modern EffectComposer (WebGL)

```js
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
import { SMAAPass } from 'three/addons/postprocessing/SMAAPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));

const bloom = new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  0.6,  // strength
  0.4,  // radius
  0.85  // threshold
);
composer.addPass(bloom);

const smaa = new SMAAPass(
  window.innerWidth * renderer.getPixelRatio(),
  window.innerHeight * renderer.getPixelRatio()
);
composer.addPass(smaa);

composer.addPass(new OutputPass());

// In animate
composer.render();
```

### 7.2 Selective Bloom (common interactive pattern)

Render emissive objects to a separate layer, bloom only that layer, then composite. Use `layers` and two render passes or a custom selective bloom pass.

### 7.3 WebGPU / TSL Post-Processing

```js
import { pass, bloom } from 'three/tsl';
import { PostProcessing } from 'three/webgpu';

const scenePass = pass(scene, camera);
const colorNode = scenePass.getTextureNode('output');
const bloomNode = bloom(colorNode, 0.5, 0.3, 0.8);

const postProcessing = new PostProcessing(renderer);
postProcessing.outputNode = colorNode.add(bloomNode);
```

---

## 8. Interactive Systems (Pointer, Raycasting, GPU Picking)

### 8.1 High-Quality Raycasting with BVH (production)

```js
import { MeshBVH, acceleratedRaycast } from 'three-mesh-bvh';

THREE.Mesh.prototype.raycast = acceleratedRaycast;

// After loading a complex mesh
mesh.geometry.boundsTree = new MeshBVH(mesh.geometry);

const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function onPointerMove(event) {
  pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
  pointer.y = -(event.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(pointer, camera);
  const hits = raycaster.intersectObject(mesh, false);
  if (hits.length) {
    // highlight, show tooltip, etc.
  }
}
```

### 8.2 GPU Picking (for thousands of instances)

Render the scene to a picking render target with unique colors per instance, then read the pixel under the cursor. Extremely fast for dense interactive scenes.

```js
// Simplified GPU picking pattern
const pickingTexture = new THREE.WebGLRenderTarget(1, 1);
const pixelBuffer = new Uint8Array(4);

function pick(x, y) {
  // Render picking scene (objects with unique id colors) into 1×1 target at mouse position
  renderer.setRenderTarget(pickingTexture);
  renderer.render(pickingScene, camera);
  renderer.readRenderTargetPixels(pickingTexture, 0, 0, 1, 1, pixelBuffer);
  renderer.setRenderTarget(null);
  const id = (pixelBuffer[0] << 16) | (pixelBuffer[1] << 8) | pixelBuffer[2];
  return id;
}
```

### 8.3 Interactive Hover + Click with InstancedMesh

```js
const mouse = new THREE.Vector2();
const raycaster = new THREE.Raycaster();

window.addEventListener('pointermove', (e) => {
  mouse.x = (e.clientX / innerWidth) * 2 - 1;
  mouse.y = -(e.clientY / innerHeight) * 2 + 1;
});

function checkIntersection() {
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObject(instancedMesh);
  if (intersects.length > 0) {
    const instanceId = intersects[0].instanceId;
    // change color of that instance, play sound, open panel, etc.
    instancedMesh.setColorAt(instanceId, highlightColor);
    instancedMesh.instanceColor.needsUpdate = true;
  }
}
```

---

## 9. Animation, Scroll-Driven & Timeline Systems

### 9.1 Scroll-Driven Camera & Object Transforms (GSAP + ScrollTrigger)

```js
import gsap from 'gsap';
import { ScrollTrigger } from 'gsap/ScrollTrigger';
gsap.registerPlugin(ScrollTrigger);

const tl = gsap.timeline({
  scrollTrigger: {
    trigger: '#scroll-container',
    start: 'top top',
    end: 'bottom bottom',
    scrub: 1.2,
  },
});

tl.to(camera.position, { x: 2, y: 4, z: 6, ease: 'none' }, 0)
  .to(controls.target, { x: 0, y: 1.5, z: 0, ease: 'none' }, 0)
  .to(model.rotation, { y: Math.PI * 2, ease: 'none' }, 0);
```

### 9.2 Animation Mixer for GLTF Clips

```js
const mixer = new THREE.AnimationMixer(model);
const clips = gltf.animations;
const action = mixer.clipAction(clips[0]);
action.play();

// In animate
const delta = clock.getDelta();
mixer.update(delta);
```

### 9.3 Custom Vertex Animation via Shader / TSL

See §6 — drive vertex displacement with `time` or a scroll progress uniform for liquid, cloth, or morphing interactive surfaces.

---

## 10. Performance, Memory & Production Hardening

### 10.1 Golden Rules (2026)

- Keep draw calls ideally < 100, hard limit ~500 on mid-range mobile.
- Prefer InstancedMesh / BatchedMesh / merged static geometry.
- Always dispose geometries, materials, textures, and render targets.
- Cap pixel ratio at 2.
- Use KTX2 + Draco + Meshopt.
- Avoid creating objects inside the render loop.
- Profile with Spector.js / Chrome Performance + WebGL Inspector.

### 10.2 Safe Dispose Pattern

```js
function disposeObject(obj) {
  if (obj.geometry) obj.geometry.dispose();
  if (obj.material) {
    if (Array.isArray(obj.material)) {
      obj.material.forEach((m) => disposeMaterial(m));
    } else {
      disposeMaterial(obj.material);
    }
  }
}

function disposeMaterial(mat) {
  for (const key of Object.keys(mat)) {
    const value = mat[key];
    if (value && value.isTexture) value.dispose();
  }
  mat.dispose();
}

function clearScene(scene) {
  scene.traverse(disposeObject);
  while (scene.children.length) scene.remove(scene.children[0]);
}
```

### 10.3 Adaptive Quality

```js
let pixelRatio = Math.min(devicePixelRatio, 2);
renderer.setPixelRatio(pixelRatio);

// On low FPS, drop quality
function adaptiveQuality(fps) {
  if (fps < 40 && pixelRatio > 1) {
    pixelRatio = 1;
    renderer.setPixelRatio(1);
    // optionally reduce shadow map size or disable bloom
  }
}
```

---

## 11. WebGPU, Compute Shaders & Advanced Effects

### 11.1 WebGPURenderer Bootstrap

```js
import * as THREE from 'three/webgpu';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init();
renderer.setPixelRatio(Math.min(devicePixelRatio, 2));
renderer.setSize(innerWidth, innerHeight);
document.body.appendChild(renderer.domElement);
```

### 11.2 Simple Compute Shader (particle positions)

```js
import { storage, instanceIndex, float, vec3, time, Fn } from 'three/tsl';

const count = 100_000;
const positionBuffer = new THREE.StorageInstancedBufferAttribute(new Float32Array(count * 3), 3);

const computeNode = Fn(() => {
  const pos = storage(positionBuffer, 'vec3', count).element(instanceIndex);
  const t = time;
  pos.x = float(instanceIndex).mul(0.01).sin().mul(5);
  pos.y = t.sin().mul(2);
  pos.z = float(instanceIndex).mul(0.01).cos().mul(5);
})().compute(count);

// In init
await renderer.computeAsync(computeNode);

// In animate (if dynamic)
renderer.compute(computeNode);
```

### 11.3 GPGPU Particles + Instanced Rendering

Combine compute for simulation with `InstancedMesh` or `Sprite` for rendering. This is the standard path for 50k–1M interactive particles in 2026.

---

## 12. Gaussian Splatting & Emerging Techniques

r186 introduced first-class Gaussian Splatting support via a TSL-based renderer (WebGPU preferred).

```js
// Conceptual usage (check latest examples for exact API)
import { GaussianSplatLoader } from 'three/addons/loaders/GaussianSplatLoader.js'; // or equivalent in r186+

const loader = new GaussianSplatLoader();
const splat = await loader.loadAsync('/splats/scene.splat');
scene.add(splat);
```

Use cases for interactive 3D:
- Photoreal product captures
- Location-based experiences
- Rapid prototyping of complex environments without traditional mesh authoring

Also watch:
- VXGI / voxel cone tracing nodes for indirect lighting
- SunLight with cascaded shadows
- Retroreflective materials

---

## 13. Architecture Patterns for Large Interactive Apps

### 13.1 Recommended Structure

```
src/
  core/
    App.js              // main loop, renderer, resize
    ResourceManager.js  // loading + dispose
    EventBus.js
  scenes/
    ProductScene.js
    ConfiguratorScene.js
  systems/
    InteractionSystem.js
    AnimationSystem.js
    PostProcessingSystem.js
  materials/
    custom/
  utils/
    math.js
    dispose.js
```

### 13.2 Scene Manager with Proper Cleanup

```js
class SceneManager {
  constructor(app) {
    this.app = app;
    this.current = null;
  }

  async set(sceneClass, ...args) {
    if (this.current) {
      await this.current.dispose();
      this.app.scene.clear();
    }
    this.current = new sceneClass(this.app, ...args);
    await this.current.init();
  }
}
```

### 13.3 React Three Fiber / Threlte / TresJS

For highly interactive UI-driven experiences, consider:

- **React Three Fiber** + `@react-three/drei` + `@react-three/postprocessing`
- Keep the same performance rules: instancing, dispose, KTX2, etc.
- Use `useFrame` sparingly and with priority when needed.

---

## 14. Final Recommendations & Checklist

### Production Checklist

- [ ] Import maps or bundler configured for `three` + `three/addons/`
- [ ] Pixel ratio capped, adaptive quality implemented
- [ ] All heavy assets use Draco / Meshopt / KTX2
- [ ] Draw calls profiled and kept low (InstancedMesh / BatchedMesh / merge)
- [ ] Proper dispose on scene change / unmount
- [ ] Shadows use `PCFShadowMap` (r186+)
- [ ] Tone mapping + color space set correctly (`SRGBColorSpace` + ACES)
- [ ] Raycasting accelerated with BVH for complex meshes
- [ ] Post-processing budgeted (especially on mobile)
- [ ] WebGPU path tested when targeting modern browsers
- [ ] Accessibility: reduced-motion media query respected, keyboard navigation considered
- [ ] Core Web Vitals measured (LCP, INP) with 3D content

### Key Principles

1. **Measure first** — never optimize blindly.
2. **Prefer data-driven GPU work** (instancing, compute, textures) over CPU loops.
3. **Dispose everything** you create.
4. **Design for the lowest device** you claim to support, then enhance.
5. **Keep interactivity feeling immediate** — aim for < 16 ms frame times on target hardware.

### Further Learning (Official & High-Signal)

- Official examples: https://threejs.org/examples/
- Migration guides: https://github.com/mrdoob/three.js/wiki/Migration-Guide
- TSL / WebGPU examples (filter by “webgpu”)
- three-mesh-bvh for production raycasting
- glTF Transform CLI for asset pipelines

---

*Document version: 1.0 — written for Three.js r186 (September 2026) and forward-compatible patterns.  
Last reviewed: October 2026.*
