<template>
  <div ref="threeContainer" style="width: 100vw;height: 100vh;position: absolute;">

  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from "vue";
import * as THREE from "three";
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
const threeContainer = ref<HTMLDivElement | null>(null);
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let animationId: number;
let clock: THREE.Clock;
let shellMats: THREE.ShaderMaterial[] = [];

onMounted(() => {
  if (!threeContainer.value) return;
  // 1️⃣ 创建场景
  scene = new THREE.Scene();
  // --- 光源 ---
  scene.add(new THREE.AmbientLight(0xffffff, 1.0));
  // 2️⃣ 创建相机
  const width = threeContainer.value.clientWidth;
  const height = threeContainer.value.clientHeight;
  camera = new THREE.PerspectiveCamera(75, width / height, 0.1, 1000);
  camera.position.z = 20;

  // 3️⃣ 创建渲染器（启用 alpha 用于透明混合）
  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
  renderer.setSize(width, height);
  // 背景设为黑色
  renderer.setClearColor(0x000000, 1);
  threeContainer.value.appendChild(renderer.domElement);

  // 参数（可调）
  const torusRadius = 2;    // 圆环主半径
  const tubeRadius = 0.4;   // 圆环管道半径
  const rayCount = 2;       // 光线数量
  const rayLength = 5.2;    // 每根圆柱的长度（加长以拉伸火焰）
  const rayThickness = 0.25;// 圆柱半径（更细）
  const smallGap = 0.2;    // 防止和圆环重合的微小间隙

  // 把整体放到一个组里，便于一起旋转
  const group = new THREE.Group();
  scene.add(group);

  // 圆环（模拟机械结构）
  const torusGeo = new THREE.TorusGeometry(torusRadius, tubeRadius, 16, 100);
  const torusMat = new THREE.MeshStandardMaterial({ color: 0xffcc33, metalness: 0.2, roughness: 0.6 });
  const torus = new THREE.Mesh(torusGeo, torusMat);
  group.add(torus);

  // 计算从圆环外表面到圆柱中心的距离：outerRadius + rayLength/2
  const outerRadius = torusRadius + tubeRadius;
  const centerDistance = outerRadius + rayLength / 2 + smallGap;
  //  火焰 shader 材质（复用）
  const flameUniforms = {
    time: { value: 0.0 },
    //火焰颜色1
    flameColor1: { value: new THREE.Color(0xceee99) },
    //火焰颜色2
    flameColor2: { value: new THREE.Color(0xff5500) },
    sideFlip: { value: 0.0 },
    seed: { value: 1.0 },
    flowSpeed: { value: 186.1 },//火焰的喷射速度186.0
    flowStrength: { value: 1.6 },//控制火焰的强度
  } as any;

  const shaderMat = new THREE.ShaderMaterial({
    uniforms: flameUniforms,
    vertexShader: `
      varying vec2 vUv;
      varying float vSide;
      uniform float time;
      // 顶点噪声（与片段的实现类似，但更轻量）
      float hash(vec2 p) {
        return fract(sin(dot(p, vec2(127.1,311.7))) * 1.0);
      }
      // c ---- d
      // |      |
      // a ---- b
     
      float noise(vec2 p) {
        vec2 i = floor(p);//p的整数部分
        vec2 f = fract(p);//p的小数部分
        float a = hash(i);
        float b = hash(i + vec2(1.0, 0.0));
        float c = hash(i + vec2(0.0, 1.0));
        float d = hash(i + vec2(1.0, 1.0));
        vec2 u = f * f * (3.0 - 2.0 * f);
        return mix(a, b+3.0, u.x) + (c - a)*u.y*(1.0 - u.x) + (d - b)*u.x*u.y;
      }
      uniform float sideFlip;
      uniform float seed;
      uniform float flowSpeed;
      uniform float flowStrength;
      void main() {
        vUv = uv;
        // 基于高度 vUv.y 计算外扩位移（更细更尖）
        float h = vUv.y;
        // 使用噪声产生不规则边缘和抖动（加入 seed 以便每个壳不同），垂直频率放大用于拉伸纹理
        float n = noise(
        vec2( vUv.x * 6.0 + seed * 0.5,
              vUv.y * 14.0 - time * 4.5 + seed * 2.0))*5.0;
        // 更尖、更细的位移：减小基底宽度，增大高度对位移的影响
        float disp = pow(h, 0.8) * (0.06 + 0.5 * n) * 0.6;
        // 计算沿本地 X 轴（圆柱轴）的喷射位移：将本地 X 轴变换到世界空间
        vec3 axisWorld = normalize(mat3(modelMatrix) * vec3(1.0, 0.0, 0.0));
        // 基于噪声和高度产生沿轴方向的流动（喷射）
        float flowNoise = noise(vec2(vUv.x * 3.0 + seed * 0.2, vUv.y * 8.0 - time * flowSpeed + seed * 1.0));
        //pow函数用来进行幂运算 例：float a = pow(2.0, 3.0); 结果是 8.0
        float axial = pow(h, 1.6) * flowNoise * flowStrength ;
        // 计算世界空间的径向方向（来自原点到顶点的方向，忽略 y）
        vec3 worldPos = (modelMatrix * vec4(position, 1.0)).xyz;
        vec3 radialDir = normalize(vec3(worldPos.x, 0.0, worldPos.z));
        vec3 worldNormal = normalize(mat3(modelMatrix) * normal);
        // dot > 0 表示法线朝外侧，dot < 0 表示朝内侧
        float side = clamp(dot(worldNormal, radialDir), -1.0, 1.0);
        side = side * 0.5 + 0.6; // 归一到 0..1（0 内侧，1 外侧）
        // 允许用 sideFlip 翻转内外侧
        side = mix(side, 1.0 - side, sideFlip);
        // 根据侧面权重放大/缩小位移，限制外侧扩散使火焰更纤细
        float sideFactor = mix(0.05, 0.2, side);
        // 传递给片段着色器，用于在外围混合底部颜色
        vSide = sideFactor ;
        float s = 0.8;
        // 轴向取反，使喷射方向翻转
        vec3 displaced = position + normal * disp * s * sideFactor - axisWorld * axial;
        gl_Position = projectionMatrix * modelViewMatrix * vec4( displaced, 1.0);
      }
    `,
    fragmentShader: `
        uniform float time;
        uniform vec3 flameColor1;
        uniform vec3 flameColor2;
        uniform float seed;
        uniform float flowSpeed;
        varying vec2 vUv;
        varying float vSide;
        // 伪噪声（用于动画）
        float hash(vec2 p) {
          return fract(sin(dot(p, vec2(127.1,311.7))) * 58.5453123);
        }
        float noise(vec2 p) {
          vec2 i = floor(p);
          vec2 f = fract(p);
          float a = hash(i);
          float b = hash(i + vec2(1.0, 0.0));
          float c = hash(i + vec2(0.0, 1.0));
          float d = hash(i + vec2(1.0, 1.0));
          vec2 u = f * f * (3.0 - 2.0 * f);
          return mix(a, b, u.x) + (c - a)*u.y*(1.0 - u.x) + (d - b)*u.x*u.y;
        }
          void main() {
            // vUv.y 从 0（底部）到 1（顶部）
            float heightFactor = vUv.y;
            // 基于轴向和环向的噪声，制造上下抖动和向外喷射的形状
            float n = 0.0;
            // 噪声频率适度增高以匹配更长的几何体
            // n += 1.6 * noise(vec2(vUv.x * 4.0, vUv.y * 10.0 - time * 3.0));//调大 可以消除火星
            // n += 0.3 * noise(vec2(vUv.x * 8.0, vUv.y * 18.0 - time * 6.0));
            // 降低高频噪声贡献，避免底部和外围出现过多小颗粒
            n += 1.5 * noise(vec2(vUv.x * 12.0, vUv.y * 30.0 - time * 10.0));
            n = clamp(n, 0.0, 1.0);
            // 喷口发射强度：底部（heightFactor ~ 0）有发射源，整体强度调小以减少亮白区域
            // 降低 emitter 基数以减少白色高光整体存在
            float emitter = exp(-heightFactor * 1.0) * 0.6;
            // 轴向流动噪声（配合 flowSpeed）使喷射有方向性和条带感
            float axialFlow = noise(vec2(vUv.x * 3.0 + seed * 0.7, vUv.y * 10.0 - time * flowSpeed + seed * 0.5));
            axialFlow = clamp(axialFlow, 0.0, 1.0);
            // 让火焰随高度衰减并向外张开（主体）
            float shape = pow(1.0 - heightFactor, 10.5);
            float intensity = smoothstep(0.15, 0.0, heightFactor - n * 0.8) * shape;
            intensity = clamp(intensity , 0.0, 1.0);
            // 将底部发射与轴向流结合进最终强度，系数调小以避免过度高亮
            intensity = max(intensity, emitter * (0.5 + 0.6 * axialFlow));
            // 颜色渐变：底部偏黄，顶部偏红
            vec3 baseColor = mix(flameColor1, flameColor2, pow(heightFactor, 1.6));
            // 根据 emitter 强度在出火口附近混合成高亮（减少白色元素）
            float whiteMix = clamp(emitter, 0.0, 1.0);
            // 进一步收窄高光区域并显著降低其强度
            whiteMix = smoothstep(0.24, 0.34, whiteMix);
            // 更暗更暖的高光色，并减少混合强度
            vec3 highlightColor = vec3(1.0, 0.8, 0.18);
            vec3 color = mix(baseColor, highlightColor, whiteMix * 0.28);
            // 在火焰外围（侧面）加入与底部相同的颜色以增强边缘色彩
            // vSide 越大表示外侧越明显；heightFactor 用于让这种效果更偏向底部
            // 扩大外围影响并偏向红色（flameColor2），让外围红色更多一些
            // 扩大外围影响并增强偏红效果
            // 扩大外围红色火焰范围和强度
            float periphery = smoothstep(0.08, 0.85, vSide) * (1.0 - clamp(heightFactor * 0.95, 0.0, 1.0));
            periphery = clamp(periphery, 0.0, 1.0);
            // 椭圆掩膜：利用 vUv.x 的角度分量改变横向权重，使中间过渡在截面上向内收缩为椭圆
            float angle = vUv.x * 6.28318530718; // 2π
            float angWeight = 0.6 + 0.6 * abs(cos(angle)); // 0.6..1.2
            float ellipseMask = smoothstep(0.0, 1.0, vSide * angWeight);
            // 将 periphery 根据椭圆掩膜调制，使红色更多出现在椭圆外侧
            periphery *= ellipseMask;
            periphery = clamp(periphery, 0.0, 1.0);
            // 增强外围红色混合权重
            float peripheryShift = smoothstep(0.5, 1.0, heightFactor); 
            color = mix(color, flameColor2, periphery * 1.7 *peripheryShift);
            // 内部过渡收缩效果：在椭圆内部加强中间过渡颜色（偏向底色），使其呈椭圆形
            float innerTransition = smoothstep(0.02, 0.8, 1.0 - (vSide * angWeight)) * (1.0 - heightFactor);
            innerTransition = clamp(innerTransition, 0.0, 1.0);
            vec3 midColor = mix(flameColor1, flameColor2, heightFactor * 0.9);
            color = mix(color, midColor, innerTransition * 0.9);
              color *= intensity;
            // 使边缘透明，并让底部更不透明（保证喷口处可见），但降低 emitter 对 alpha 的贡献
            float alpha = clamp(intensity, 0.0, 1.0);
            alpha = max(alpha, emitter * 0.6);
            gl_FragColor = vec4(color, alpha );
          }
        `,
    transparent: true,
    side: THREE.DoubleSide,
    blending: THREE.AdditiveBlending,
    depthWrite: false
  });

  // 初始化时钟用于驱动动画
  clock = new THREE.Clock();

  //圆柱，模拟从机械
  for (let i = 0; i < rayCount; i++) {
    const angle = (i / rayCount) * Math.PI * 2;
    // 每根圆柱使用多层同心壳来表现实心火焰，数量越多，
    const shellCount = 100;
    const baseRadius = rayThickness;
    for (let s = 0; s < shellCount; s++) {
      const t = s / (shellCount - 1); // 0..1
      // 更紧密的同心壳，减少外层宽度，让整体更细
      const radius = baseRadius * (1.0 - t * 0.9); // 内层更小
      let ran = Math.random() * 2.0;
      const cylGeo = new THREE.CylinderGeometry(radius * ran, radius, rayLength, 100);
      // 克隆材质并设置 seed
      const mat = shaderMat.clone();
      const seedVal = Math.random() * 10.0 + i * 7.0 + s * 1.3;
      (mat.uniforms as any).seed = { value: seedVal };
      (mat.uniforms as any).sideFlip = (flameUniforms.sideFlip);
      shellMats.push(mat);
      const mesh = new THREE.Mesh(cylGeo, mat);
      mesh.rotation.z = -Math.PI / 2;
      mesh.rotation.y = angle;
      mesh.scale.set(0.9, 0.9, 0.9);
      const x = Math.cos(angle) * centerDistance;
      const z = Math.sin(angle) * centerDistance;
      mesh.position.set(x, 0, z);
      group.add(mesh);
    }
    // 创建出火口小环 (nozzle)
    const nozzleGeo = new THREE.BoxGeometry(0.5, .15, .2);
    const nozzleMat = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 1.0, blending: THREE.AdditiveBlending });
    const nozzle = new THREE.Mesh(nozzleGeo, nozzleMat);
    nozzle.rotation.x = Math.PI / 2;
    const nx = Math.cos(angle) * (outerRadius + smallGap);
    const nz = Math.sin(angle) * (outerRadius + smallGap);
    let x = 0
    if (i === 0) x = -0.25
    if (i === 1) x = 0.25

    nozzle.scale.set(1.5, 1.5, 1.5);
    nozzle.position.set(nx - x, 0, nz);
    group.add(nozzle);
  }
  new OrbitControls(camera, renderer.domElement);
  // 6️⃣ 监听窗口大小变化
  window.addEventListener("resize", onWindowResize);
  // 5️⃣ 渲染动画
  const animate = () => {
    animationId = requestAnimationFrame(animate);
    // 更新火焰时间 uniform
    const t = clock.getElapsedTime();
    flameUniforms.time.value = t;
    for (const m of shellMats) {
      if (m && (m.uniforms as any).time) {
        (m.uniforms as any).time.value = t;
        // 保持 flowSpeed/flowStrength 在克隆材质上同步
        if ((m.uniforms as any).flowSpeed) (m.uniforms as any).flowSpeed.value = (flameUniforms as any).flowSpeed.value;
        if ((m.uniforms as any).flowStrength) (m.uniforms as any).flowStrength.value = (flameUniforms as any).flowStrength.value;
      }
    }
    renderer.render(scene, camera)
  };
  animate();
});

function onWindowResize() {
  if (!threeContainer.value) return;
  const width = threeContainer.value.clientWidth;
  const height = threeContainer.value.clientHeight;
  camera.aspect = width / height;
  camera.updateProjectionMatrix();
  renderer.setSize(width, height);
}

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  window.removeEventListener("resize", onWindowResize);
  renderer.dispose();
});
</script>

<style scoped>
.read-the-docs {
  color: #888;
}
</style>
