<template>
  <!-- 1. 加载层 -->
  <div id="loading" v-if="isLoading">
    <div class="spinner"></div>
    <div class="loading-text">{{ loadingText }}</div>
  </div>

  <!-- 2. 全局通知 (右下角) -->
  <transition name="slide-fade">
    <div v-if="notification.show" :class="['notification', notification.type]">
      {{ notification.message }}
    </div>
  </transition>

  <!-- 3. 右上角控制面板 (使用 @click.stop 彻底拦截冒泡，防止触发画布点击) -->
  <div class="controls" @click.stop>
    <button class="control-btn" @click="handleBtnToggleRotation" :class="{ active: !isAutoRotating }">
      <span>{{ isAutoRotating ? '⏸️' : '▶️' }}</span>
      <span>{{ isAutoRotating ? '暂停旋转' : '恢复旋转' }}</span>
    </button>
    <button class="control-btn" @click="handleBtnRefresh" :disabled="isRefreshing">
      <span :class="{ spinning: isRefreshing }">🔄</span>
      <span>{{ isRefreshing ? '刷新中...' : '刷新数据' }}</span>
    </button>
    <button class="control-btn" @click="handleBtnReset">
      <span>🔍</span>
      <span>重置视角</span>
    </button>
    <button class="control-btn" @click="toggleInfoPanel">
      <span>ℹ️</span>
      <span>{{ showInfoPanel ? '隐藏信息' : '显示信息' }}</span>
    </button>
  </div>

  <!-- 4. 左下角信息面板 (拦截冒泡) -->
  <div class="info-panel" @click.stop v-show="showInfoPanel">
    <h3><span>🌍</span> Three.js 3D地球监测</h3>
    <ul>
      <li>实时显示全球 4.0 级以上地震</li>
      <li>数据源：USGS Earthquake Catalog</li>
      <li>时间范围：过去 30 天</li>
      <li>鼠标拖拽旋转，滚轮缩放</li>
      <li>点击地震点暂停并查看详情</li>
    </ul>

    <div id="earthquake-count">
      <strong>当前显示：</strong>
      <span class="count-number">{{ earthquakeCount }}</span> 个地震点
    </div>

    <div class="legend">
      <div class="legend-item">
        <div class="legend-color" style="background: #3399ff;"></div>
        <span>M4.0 - 4.9</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #66ccff;"></div>
        <span>M4.5 - 4.9</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #ffcc00;"></div>
        <span>M5.0 - 5.9</span>
      </div>
      <div class="legend-item">
        <div class="legend-color" style="background: #ff6600;"></div>
        <span>M6.0 - 6.9</span>
      </div>
      <div class="legend-item" style="grid-column: span 2;">
        <div class="legend-color" style="background: #ff0000; box-shadow: 0 0 5px red;"></div>
        <span>M7.0+ (重大地震)</span>
      </div>
    </div>
  </div>

  <!-- 5. Tooltip -->
  <div 
    id="tooltip" 
    v-show="tooltip.visible" 
    :style="{ left: tooltip.x + 'px', top: tooltip.y + 'px' }"
    v-html="tooltip.content"
  ></div>

  <!-- 6. Three.js 画布 (独立的点击事件) -->
  <div ref="canvasContainer" id="canvas-container" @click="onCanvasClick"></div>
</template>

<script setup>
import { ref, reactive, onMounted, onBeforeUnmount } from 'vue';
import * as THREE from 'three';
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

// --- 状态管理 ---
const canvasContainer = ref(null);
const isLoading = ref(true);
const loadingText = ref('正在初始化 3D 地球...');
const isRefreshing = ref(false);
const isAutoRotating = ref(true);
const manualPaused = ref(false);
const showInfoPanel = ref(true);
const earthquakeCount = ref(0);
const tooltip = reactive({ visible: false, x: 0, y: 0, content: '' });
const notification = reactive({ show: false, message: '', type: 'success' });

// --- Three.js 变量 ---
let scene, camera, renderer, controls;
let earthGroup, quakeGroup, flyLineGroup;
let raycaster, mouse;
let animationId, intervalId;

const EARTH_RADIUS = 100;
const USGS_API = 'https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_month.geojson';

// --- 字典与工具 ---
const DICTIONARY = [
  { k: 'Papua New Guinea', v: '巴布亚新几内亚' },
  { k: 'New Zealand', v: '新西兰' },
  { k: 'New Caledonia', v: '新喀里多尼亚' },
  { k: 'Solomon Islands', v: '所罗门群岛' },
  { k: 'Philippines', v: '菲律宾' },
  { k: 'Indonesia', v: '印度尼西亚' },
  { k: 'South Sandwich Islands', v: '南桑威奇群岛' },
  { k: 'Easter Island', v: '复活节岛' },
  { k: 'Kuril Islands', v: '千岛群岛' },
  { k: 'Aleutian Islands', v: '阿留申群岛' },
  { k: 'Mid-Atlantic Ridge', v: '大西洋中脊' },
  { k: 'California', v: '加利福尼亚' },
  { k: 'Alaska', v: '阿拉斯加' },
  { k: 'Japan', v: '日本' },
  { k: 'Chile', v: '智利' },
  { k: 'Mexico', v: '墨西哥' },
  { k: 'Peru', v: '秘鲁' },
  { k: 'Fiji', v: '斐济' },
  { k: 'Tonga', v: '汤加' },
  { k: 'Russia', v: '俄罗斯' },
  { k: 'China', v: '中国' },
  { k: 'Taiwan', v: '中国台湾' },
  { k: 'Argentina', v: '阿根廷' },
  { k: 'Turkey', v: '土耳其' },
  { k: 'Iran', v: '伊朗' },
  { k: 'India', v: '印度' },
  { k: 'Iceland', v: '冰岛' },
  { k: 'Italy', v: '意大利' },
  { k: 'Afghanistan', v: '阿富汗' },
  { k: '\\bSouth of\\b', v: '以南' }, { k: '\\bNorth of\\b', v: '以北' },
  { k: '\\bEast of\\b', v: '以东' }, { k: '\\bWest of\\b', v: '以西' },
  { k: '\\bSSE\\b', v: '东南偏南' }, { k: '\\bSSW\\b', v: '西南偏南' },
  { k: '\\bNNE\\b', v: '东北偏北' }, { k: '\\bNNW\\b', v: '西北偏北' },
  { k: '\\bENE\\b', v: '东北偏东' }, { k: '\\bWNW\\b', v: '西北偏西' },
  { k: '\\bESE\\b', v: '东南偏东' }, { k: '\\bWSW\\b', v: '西南偏西' },
  { k: '\\bSE\\b', v: '东南' }, { k: '\\bSW\\b', v: '西南' },
  { k: '\\bNE\\b', v: '东北' }, { k: '\\bNW\\b', v: '西北' },
  { k: '\\bE\\b', v: '东' }, { k: '\\bW\\b', v: '西' },
  { k: '\\bN\\b', v: '北' }, { k: '\\bS\\b', v: '南' },
  { k: 'region', v: '地区' },
  { k: 'Island', v: '岛' },
  { k: 'Sea', v: '海' },
  { k: 'Coast', v: '海岸' },
  { k: 'Central', v: '中部' },
  { k: 'Southern', v: '南部' },
  { k: 'Northern', v: '北部' },
  { k: 'Eastern', v: '东部' },
  { k: 'Western', v: '西部' },
  { k: 'km', v: '千米' },
  { k: ' of ', v: ' 位于 ' }
];

function showNotification(msg, type = 'success') {
  notification.message = msg;
  notification.type = type;
  notification.show = true;
  setTimeout(() => { notification.show = false; }, 3000);
}

function translateLocation(text) {
  if (!text) return '未知地点';
  let res = text;
  for (const item of DICTIONARY) {
    try {
      const reg = new RegExp(item.k, 'gi');
      res = res.replace(reg, item.v);
    } catch (e) {
      res = res.split(item.k).join(item.v);
    }
  }
  return res.replace(/\s+/g, ' ').trim();
}

function formatTime(timeStr) {
  const date = new Date(timeStr);
  return date.toLocaleString('zh-CN', {
    year: 'numeric', month: '2-digit', day: '2-digit',
    hour: '2-digit', minute: '2-digit', second: '2-digit'
  });
}

onMounted(() => {
  initThree();
  fetchData();
  intervalId = setInterval(fetchData, 5 * 60 * 1000);
  window.addEventListener('resize', onWindowResize);
  window.addEventListener('mousemove', onMouseMove);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId);
  clearInterval(intervalId);
  window.removeEventListener('resize', onWindowResize);
  window.removeEventListener('mousemove', onMouseMove);
  if (renderer) renderer.dispose();
});

// --- Three.js 初始化 ---
function initThree() {
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x010103);

  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 2000);
  camera.position.set(200, 100, 250);

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: false });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.shadowMap.enabled = true;
  renderer.outputColorSpace = THREE.SRGBColorSpace; 
  canvasContainer.value.appendChild(renderer.domElement);

  controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.05;
  controls.autoRotate = isAutoRotating.value;
  controls.autoRotateSpeed = 0.8;
  controls.minDistance = 120;
  controls.maxDistance = 600;

  // 监听用户拖拽，更新Vue状态
  controls.addEventListener('start', () => {
    if (isAutoRotating.value) {
      isAutoRotating.value = false;
      showNotification('已暂停自动旋转', 'warning');
    }
  });

  const ambientLight = new THREE.AmbientLight(0xffffff, 0.4);
  scene.add(ambientLight);
  const mainLight = new THREE.DirectionalLight(0xffffff, 1.2);
  mainLight.position.set(500, 300, 500);
  scene.add(mainLight);
  const backLight = new THREE.DirectionalLight(0x445566, 0.8);
  backLight.position.set(-500, -300, -500);
  scene.add(backLight);

  earthGroup = new THREE.Group();
  scene.add(earthGroup);
  quakeGroup = new THREE.Group();
  earthGroup.add(quakeGroup);
  flyLineGroup = new THREE.Group();
  earthGroup.add(flyLineGroup);

  raycaster = new THREE.Raycaster();
  mouse = new THREE.Vector2();

  createEarth();
  createStars();
  animate();
}

function createEarth() {
  const geometry = new THREE.SphereGeometry(EARTH_RADIUS, 64, 64);
  const textureLoader = new THREE.TextureLoader();
  
  textureLoader.load('/earth.jpg', (texture) => {
    texture.colorSpace = THREE.SRGBColorSpace; 
    const material = new THREE.MeshPhongMaterial({
      map: texture,
      specular: new THREE.Color(0x333333),
      shininess: 5,
      transparent: false,
      opacity: 1.0,
      depthWrite: true,
      depthTest: true
    });
    const earth = new THREE.Mesh(geometry, material);
    earthGroup.add(earth);
    isLoading.value = false;
  }, undefined, (err) => {
    console.warn("贴图加载失败", err);
    const material = new THREE.MeshPhongMaterial({ 
      color: 0x1a365d, 
      transparent: false, 
      opacity: 1.0 
    });
    const earth = new THREE.Mesh(geometry, material);
    earthGroup.add(earth);
    const wireframe = new THREE.LineSegments(new THREE.WireframeGeometry(geometry));
    wireframe.material.color = new THREE.Color(0x44aaff);
    wireframe.material.opacity = 0.3;
    wireframe.material.transparent = true;
    earthGroup.add(wireframe);
    isLoading.value = false;
  });

  const atmosMat = new THREE.MeshPhongMaterial({
    color: 0x0099ff, 
    transparent: true, 
    opacity: 0.15,
    side: THREE.BackSide, 
    blending: THREE.AdditiveBlending,
    depthWrite: false
  });
  const atmosphere = new THREE.Mesh(new THREE.SphereGeometry(EARTH_RADIUS * 1.02, 64, 64), atmosMat);
  earthGroup.add(atmosphere);
}

function createStars() {
  const geometry = new THREE.BufferGeometry();
  const vertices = [];
  for (let i = 0; i < 4000; i++) {
    vertices.push((Math.random() - 0.5) * 2500, (Math.random() - 0.5) * 2500, (Math.random() - 0.5) * 2500);
  }
  geometry.setAttribute('position', new THREE.Float32BufferAttribute(vertices, 3));
  const material = new THREE.PointsMaterial({ color: 0x888888, size: 1.2, sizeAttenuation: false });
  const stars = new THREE.Points(geometry, material);
  scene.add(stars);
}

function latLonToVector3(lat, lon, radius) {
  const phi = (90 - lat) * (Math.PI / 180);
  const theta = (lon + 180) * (Math.PI / 180);
  const x = -(radius * Math.sin(phi) * Math.cos(theta));
  const z = (radius * Math.sin(phi) * Math.sin(theta));
  const y = (radius * Math.cos(phi));
  return new THREE.Vector3(x, y, z);
}

function createCircleTexture() {
  const canvas = document.createElement('canvas');
  canvas.width = 64; canvas.height = 64;
  const ctx = canvas.getContext('2d');
  const gradient = ctx.createRadialGradient(32, 32, 0, 32, 32, 32);
  gradient.addColorStop(0, 'rgba(255, 255, 255, 1)');
  gradient.addColorStop(0.3, 'rgba(255, 255, 255, 0.9)'); 
  gradient.addColorStop(0.5, 'rgba(255, 255, 255, 0.3)');
  gradient.addColorStop(1, 'rgba(0, 0, 0, 0)');
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, 64, 64);
  return new THREE.CanvasTexture(canvas);
}
const circleTexture = createCircleTexture();

async function fetchData() {
  isRefreshing.value = true;
  loadingText.value = "正在同步全球数据...";
  try {
    const res = await fetch(USGS_API);
    const data = await res.json();
    localStorage.setItem('quake_data', JSON.stringify(data));
    renderData(data);
    showNotification(`数据更新成功！共 ${earthquakeCount.value} 个地震点`);
  } catch (e) {
    const cached = localStorage.getItem('quake_data');
    if (cached) {
      renderData(JSON.parse(cached));
      showNotification('网络异常，已加载本地缓存', 'warning');
    } else {
      showNotification('数据获取失败', 'error');
    }
  } finally {
    isRefreshing.value = false;
    isLoading.value = false;
  }
}

function renderData(data) {
  quakeGroup.clear();
  let count = 0;

  data.features.forEach(f => {
    const { coordinates } = f.geometry;
    const { mag, time, place } = f.properties;
    
    if (mag < 4.0) return;
    count++;

    const pos = latLonToVector3(coordinates[1], coordinates[0], EARTH_RADIUS);
    
    let color = 0x3399ff;
    let size = 1.0;
    if (mag >= 7.0) { color = 0xff0000; size = 2.0; }
    else if (mag >= 6.0) { color = 0xff6600; size = 1.5; }
    else if (mag >= 5.0) { color = 0xffcc00; size = 1.2; }
    else if (mag >= 4.5) { color = 0x66ccff; size = 1.1; }

    const material = new THREE.SpriteMaterial({
      map: circleTexture,
      color: color,
      transparent: true,
      opacity: 1.0,
      depthTest: true,
      depthWrite: false
    });
    
    const sprite = new THREE.Sprite(material);
    sprite.position.copy(pos.clone().multiplyScalar(1.02)); 
    const scale = (mag - 3) * 1.5 * size;
    sprite.scale.set(scale, scale, 1);
    
    sprite.userData = { 
      place: translateLocation(place), 
      mag, 
      depth: coordinates[2], 
      time: formatTime(time) 
    };
    
    quakeGroup.add(sprite);
  });
  
  earthquakeCount.value = count;
}

// =================== 事件处理区域 ===================

// 1. 按钮事件：只负责切换状态，不涉及射线检测
function handleBtnToggleRotation() {
  isAutoRotating.value = !isAutoRotating.value;
  controls.autoRotate = isAutoRotating.value;
  manualPaused.value = !isAutoRotating.value;
  showNotification(isAutoRotating.value ? '已恢复自动旋转' : '已暂停自动旋转', 'success');
}

function handleBtnRefresh() {
  if (!isRefreshing.value) fetchData();
}

function handleBtnReset() {
  if (controls) {
    controls.reset();
    isAutoRotating.value = true;
    controls.autoRotate = true;
    manualPaused.value = false;
    showNotification('视角已重置');
  }
}

function toggleInfoPanel() {
  showInfoPanel.value = !showInfoPanel.value;
}

// 2. 画布点击事件：处理地球交互
function onCanvasClick(e) {
  mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(quakeGroup.children);

  if (intersects.length > 0) {
    // 【场景A】点击了地震点 -> 强制暂停
    isAutoRotating.value = false;
    controls.autoRotate = false;
    if (!manualPaused.value) showNotification('已暂停查看详情', 'success');
    
    // 选中动画
    const s = intersects[0].object;
    const oldScale = s.scale.x;
    s.scale.setScalar(oldScale * 1.3);
    setTimeout(() => s.scale.setScalar(oldScale), 300);
  } else {
    // 【场景B】点击了背景 -> 强制恢复旋转
    if (!isAutoRotating.value && !manualPaused.value) {
      isAutoRotating.value = true;
      controls.autoRotate = true;
      // showNotification('恢复自动旋转', 'success'); // 减少打扰，背景恢复不提示
    }
  }
}

function onMouseMove(e) {
  mouse.x = (e.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(e.clientY / window.innerHeight) * 2 + 1;
  raycaster.setFromCamera(mouse, camera);
  const intersects = raycaster.intersectObjects(quakeGroup.children);

  if (intersects.length > 0) {
    const point = intersects[0].object;
    // 背面剔除
    const normal = point.position.clone().normalize();
    const viewDir = camera.position.clone().sub(point.position).normalize();
    
    if (normal.dot(viewDir) < 0) {
      tooltip.visible = false;
      document.body.style.cursor = 'default';
      return;
    }

    const data = point.userData;
    let colorStyle = 'color: #3399ff';
    if (data.mag >= 7) colorStyle = 'color: #ff0000; font-weight:900';
    else if (data.mag >= 6) colorStyle = 'color: #ff6600; font-weight:bold';
    else if (data.mag >= 5) colorStyle = 'color: #ffcc00';

    tooltip.visible = true;
    tooltip.x = e.clientX + 15;
    tooltip.y = e.clientY + 15;
    
    tooltip.content = `
      <div style="font-weight:bold; margin-bottom:6px; border-bottom:1px solid #444; padding-bottom:4px; max-width:220px; line-height:1.4">${data.place}</div>
      <div style="display:flex; justify-content:space-between; margin-bottom:4px">
        <span>震级:</span> <span style="${colorStyle}; font-size:15px">M${data.mag}</span>
      </div>
      <div style="display:flex; justify-content:space-between; margin-bottom:4px">
        <span>深度:</span> <span>${data.depth} 千米</span>
      </div>
      <div style="color:#aaa; font-size:12px; margin-top:6px">${data.time}</div>
    `;
    document.body.style.cursor = 'pointer';
  } else {
    tooltip.visible = false;
    document.body.style.cursor = 'default';
  }
}

function animate() {
  animationId = requestAnimationFrame(animate);
  if (controls) controls.update();
  if (renderer && scene && camera) renderer.render(scene, camera);
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}
</script>

<style scoped>
#canvas-container { width: 100%; height: 100%; display: block; }

#loading {
  position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.85); padding: 30px 50px;
  border-radius: 12px; border: 1px solid #1e88e5;
  text-align: center; color: white; z-index: 1000;
  box-shadow: 0 0 40px rgba(30, 136, 229, 0.4);
}
.spinner {
  width: 40px; height: 40px; margin: 0 auto 15px;
  border: 4px solid rgba(255,255,255,0.1); border-top-color: #1e88e5;
  border-radius: 50%; animation: spin 1s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

.controls {
  position: absolute; top: 20px; right: 20px;
  display: flex; flex-direction: column; gap: 10px; z-index: 100;
}
.control-btn {
  padding: 10px 18px;
  background: linear-gradient(135deg, #1e88e5cc, #0d47a1cc);
  color: white; border: none; border-radius: 8px;
  cursor: pointer; font-size: 14px; font-weight: 600;
  display: flex; align-items: center; gap: 8px;
  backdrop-filter: blur(4px); transition: all 0.3s;
  box-shadow: 0 4px 10px rgba(0,0,0,0.3);
  min-width: 120px;
}
.control-btn:hover { transform: translateX(-5px); background: #2196f3; }
.control-btn:disabled { opacity: 0.6; cursor: not-allowed; }
.control-btn.active { background: linear-gradient(135deg, #ff6b6b, #c0392b); }
.spinning { animation: spin 1s linear infinite; display: inline-block; }

.info-panel {
  position: absolute; bottom: 20px; left: 20px;
  background: rgba(10, 18, 30, 0.9); color: #e0e0e0;
  padding: 20px; border-radius: 12px; width: 320px;
  border-left: 4px solid #1e88e5;
  backdrop-filter: blur(10px); z-index: 100;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
  font-family: 'Segoe UI', sans-serif;
}
.info-panel h3 { margin: 0 0 12px 0; color: #4dabf7; font-size: 18px; display: flex; align-items: center; gap:8px; }
.info-panel ul { padding-left: 18px; margin: 0; font-size: 13px; line-height: 1.6; color: #ccc; }
#earthquake-count {
  margin-top: 15px; background: rgba(30, 136, 229, 0.15);
  padding: 8px; border-radius: 6px; text-align: center;
  border: 1px solid rgba(30, 136, 229, 0.3); font-size: 13px;
}
.count-number { color: #ff6b6b; font-weight: bold; font-size: 16px; margin: 0 4px; }

.legend {
  display: grid; grid-template-columns: 1fr 1fr; gap: 8px; margin-top: 15px;
  padding-top: 10px; border-top: 1px solid rgba(255,255,255,0.1);
}
.legend-item { display: flex; align-items: center; gap: 8px; font-size: 12px; color: #aaa; }
.legend-color { width: 10px; height: 10px; border-radius: 50%; }

#tooltip {
  position: absolute; background: rgba(8, 15, 25, 0.95);
  border: 1px solid #44aaff; color: #fff; padding: 12px;
  border-radius: 6px; font-size: 13px; pointer-events: none;
  box-shadow: 0 4px 20px rgba(0,0,0,0.6); z-index: 200;
  min-width: 200px;
}

.notification {
  position: absolute; bottom: 30px; right: 20px;
  padding: 12px 24px; border-radius: 8px; color: white;
  box-shadow: 0 5px 15px rgba(0,0,0,0.3); z-index: 1000;
  font-size: 14px; display: flex; align-items: center;
}
.notification.success { background: linear-gradient(135deg, #43a047, #2e7d32); }
.notification.warning { background: linear-gradient(135deg, #f39c12, #d35400); }
.notification.error { background: linear-gradient(135deg, #e53935, #c62828); }
.slide-fade-enter-active, .slide-fade-leave-active { transition: all 0.3s ease; }
.slide-fade-enter-from, .slide-fade-leave-to { transform: translateY(20px); opacity: 0; }
</style>
