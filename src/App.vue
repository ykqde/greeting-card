<template>
  <div
    class="scenes"
    style="
      position: fixed;
      left: 0;
      top: 0;
      z-index: 10;
      pointer-events: none;
      transition: all 1s;
    "
    :style="{
      transform: `translate3d(0,${-index * 100}vh,0)`,
    }"
  >
    <div v-for="item in scenes" style="width: 100vm; height: 100vh">
      <h1 style="padding: 100px 100px; font-size: 50px; color: #fff">
        {{ item.text }}
      </h1>
    </div>
  </div>
</template>

<script setup>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader";
import { DRACOLoader } from "three/examples/jsm/loaders/DRACOLoader";
import { RGBELoader } from "three/examples/jsm/loaders/RGBELoader";
import { Water } from "three/examples/jsm/objects/Water2";
import gsap from "gsap";
import { ref } from "vue";

// 初始化场景
const scene = new THREE.Scene();
// 初始化相机
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);

camera.position.set(0, 0, -60);

camera.updateProjectionMatrix();

// 初始化渲染器
const renderer = new THREE.WebGLRenderer({
  // 设置抗锯齿
  antialias: true,
});
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 设置色调映射
renderer.outputEncoding = THREE.sRGBEncoding;
renderer.toneMapping = THREE.ACESFilmicToneMapping;
renderer.toneMappingExposure = 0.5;
renderer.shadowMap.enabled = true;
renderer.physicallyCorrectLights = true;

// 初始化控制器
const controls = new OrbitControls(camera, renderer.domElement);
controls.target.set(10, 2, -45);
controls.enableDamping = true;

// 初始化loader
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath("/draco/");
const gltfLoader = new GLTFLoader();
gltfLoader.setDRACOLoader(dracoLoader);

// 加载环境纹理
let rgbeLoader = new RGBELoader();
rgbeLoader.load("/textures/water.hdr", (texture) => {
  texture.mapping = THREE.EquirectangularReflectionMapping;
  scene.background = texture;
  scene.environment = texture;
});

// 加载模型
gltfLoader.load("/model/scene.glb", (gltf) => {
  const model = gltf.scene;
  // console.log(model);
  model.traverse((child) => {
    if (child.name === "Plane") {
      child.visible = false;
    }
    if (child.isMesh) {
      child.castShadow = true;
      child.receiveShadow = true;
    }
  });
  scene.add(model);
});

// 创建水面
const waterGeometry = new THREE.CircleGeometry(300, 32);
const water = new Water(waterGeometry, {
  textureWidth: 1024,
  textureHeight: 1024,
  color: 0xeeeeff,
  flowDirection: new THREE.Vector2(1, 1),
  scale: 100,
});
water.rotation.x = -Math.PI / 2;
water.position.y = -3; //确定水面位置
scene.add(water);

// 添加平行光
const light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(0, 50, 0);
scene.add(light);

// 添加点光源
const pointLight = new THREE.PointLight(0xffffff, 50);
pointLight.position.set(0.1, 2.4, 0);
pointLight.castShadow = true;
scene.add(pointLight);

// 创建点光源组
const pointLightGroup = new THREE.Group();
pointLightGroup.position.set(10, 2, -45);
let radius = 3;
let pointLightArr = [];
for (let i = 0; i < 3; i++) {
  // 创建球体当灯泡
  const sphereGeometry = new THREE.SphereGeometry(0.2, 32, 32);
  const sphereMaterial = new THREE.MeshBasicMaterial({
    color: 0xffffff,
    emissive: 0xffffff,
    emissiveIntensity: 10,
  });
  const sphere = new THREE.Mesh(sphereGeometry, sphereMaterial);
  pointLightArr.push(sphere);
  sphere.position.set(
    radius * Math.cos((i * 2 * Math.PI) / 3),
    Math.cos((i * 2 * Math.PI) / 3),
    radius * Math.sin((i * 2 * Math.PI) / 3)
  );

  let pointLight = new THREE.PointLight(0xffffff, 50);
  sphere.add(pointLight);
  pointLightGroup.add(sphere);
}
scene.add(pointLightGroup);

// 使用补间函数，从0到2π，使灯泡旋转
let options = {
  angle: 0,
};
gsap.to(options, {
  angle: Math.PI * 2,
  duration: 10,
  repeat: -1,
  ease: "linear",
  onUpdate: () => {
    pointLightGroup.rotation.y = options.angle;
    pointLightArr.forEach((item, index) => {
      item.position.set(
        radius * Math.cos((index * 2 * Math.PI) / 3),
        Math.cos((index * 2 * Math.PI) / 3 + options.angle * 5),
        radius * Math.sin((index * 2 * Math.PI) / 3)
      );
    });
  },
});

function render() {
  requestAnimationFrame(render);
  renderer.render(scene, camera);
  controls.update();
}
render();

// 使用补间动画移动相机
let timeLine1 = gsap.timeline();
let timeLine2 = gsap.timeline();

// 定义相机移动函数
function tranlateCamera(position, target) {
  timeLine1.to(camera.position, {
    x: position.x,
    y: position.y,
    z: position.z,
    duration: 1,
    ease: "power2.out",
  });
  timeLine2.to(controls.target, {
    x: target.x,
    y: target.y,
    z: target.z,
    duration: 1,
    ease: "power2.inOut",
  });
}

let scenes = [
  {
    text: "hihihi~",
    callback: () => {
      // 执行函数切换位置
      tranlateCamera(
        new THREE.Vector3(20, 2, -60),
        new THREE.Vector3(0, 2, -60)
      );
    },
  },
  {
    text: "testtesttest~",
    callback: () => {
      // 执行函数切换位置
      tranlateCamera(
        new THREE.Vector3(0, 2, -60),
        new THREE.Vector3(15, 2, -60)
      );
    },
  },
  {
    text: "testtesttesttesttesttest~",
    callback: () => {
      // 执行函数切换位置
      tranlateCamera(
        new THREE.Vector3(15, 0, -80),
        new THREE.Vector3(15, 2, -60),
        restoreHeart()
      );
    },
  },
  {
    text: "testtesttesttesttesttesttesttesttest~",
    callback: () => {
      // 执行函数切换位置
      tranlateCamera(
        new THREE.Vector3(-15, 2, -50),
        new THREE.Vector3(0, 2, -80)
      );
    },
  },
  {
    text: "111~",
    callback: () => {
      // 执行函数切换位置
      tranlateCamera(
        new THREE.Vector3(10, 2, -80),
        new THREE.Vector3(0, 0, 80)
      );
      
    },
  },
];

let index = ref(0);
let isAnimate = false;
// 监听鼠标滚轮事件
window.addEventListener(
  "wheel",
  (e) => {
    if (isAnimate) return;
    isAnimate = true;
    if (e.deltaY > 0) {
      index.value++;
      if (index.value > scene.length - 1) {
        index.value = 0;
        restoreHeart();
      }
    }
    scenes[index.value].callback();
    setTimeout(() => {
      isAnimate = false;
    }, 1000);
  },
  false
);

// 实例化创建漫天星星
let starsInstance = new THREE.InstancedMesh(
  new THREE.SphereGeometry(0.1, 32, 32),
  new THREE.MeshStandardMaterial({
    color: 0xffffff,
    emissive: 0xffffff,
    emissiveIntensity: 10,
  }),
  100
);

// 星星随机到天上
let startArr = [];
let endArr = [];
for (let i = 0; i < 100; i++) {
  let x = Math.random() * 100 - 50;
  let y = Math.random() * 100 - 50;
  let z = Math.random() * 100 - 50;
  startArr.push(new THREE.Vector3(x, y, z));
  let matrix = new THREE.Matrix4();
  matrix.setPosition(x, y, z);
  starsInstance.setMatrixAt(i, matrix);
}
scene.add(starsInstance);

// 创建爱心路径
let heartShape = new THREE.Shape();
heartShape.moveTo(60, 60);
heartShape.bezierCurveTo(60, 60, 55, 0, 0, 0);
heartShape.bezierCurveTo(-65, 0, -65, 70, -65, 70);
heartShape.bezierCurveTo(-60, 55, -45, 112, 60, 130);
heartShape.bezierCurveTo(60, 112, 115, 55, 115, 70);
heartShape.bezierCurveTo(115, 70, 115, 0, 50, 0);
heartShape.bezierCurveTo(70, 0, 60, 60, 60, 60);

// 根据爱心路径获取点
let center = new THREE.Vector3(0, 2, -80);
for (let i = 0; i < 100; i++) {
  let point = heartShape.getPoint(i / 100);
  endArr.push(
    new THREE.Vector3(
      point.x * 0.1 + center.x,
      point.y * 0.1 + center.y,
      center.z
    )
  );
}

// 创建爱心动画
function restoreHeart() {
  let params = {
    time: 0,
  };
  gsap.to(params, {
    time: 1,
    duration: 1,
    onUpdate: () => {
      for (let i = 0; i < 100; i++) {
        let x = endArr[i].x + (startArr[i].x - endArr[i].x) * params.time;
        let y = endArr[i].y + (startArr[i].y - endArr[i].y) * params.time;
        let z = endArr[i].z + (startArr[i].z - endArr[i].z) * params.time;
        // let x = startArr[i].x + (endArr[i].x - startArr[i].x) * params.times;
        // let y = startArr[i].y + (endArr[i].y - startArr[i].y) * params.times;
        // let z = startArr[i].z + (endArr[i].z - startArr[i].z) * params.times;
        let matrix = new THREE.Matrix4();
        matrix.setPosition(x, y, z);
        starsInstance.setMatrixAt(i, matrix);
      }
      starsInstance.instanceMatrix.needsUpdate = true;
    },
  });
}
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
}
canvas {
  width: 100vw;
  height: 100vh;
  position: fixed;
  left: 0;
  top: 0;
}
</style>
