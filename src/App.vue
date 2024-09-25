<script setup>
import { ref, onMounted } from "vue";
import * as pdfjs from "pdfjs-dist";
pdfjs.GlobalWorkerOptions.workerSrc = "/pdf.worker.min.js";
import * as THREE from "three";

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(
  75,
  window.innerWidth / window.innerHeight,
  0.1,
  1000
);
// 创建坐标轴帮助对象，长度为100
const axesHelper = new THREE.AxesHelper(100);

// 将坐标轴添加到场景
// scene.add(axesHelper);
const renderer = new THREE.WebGLRenderer();
document.body.appendChild(renderer.domElement);
renderer.setSize(window.innerWidth, window.innerHeight);
onMounted(() => {
  document.body.appendChild(renderer.domElement);
  const loadingTask = pdfjs.getDocument("http://localhost:5173/demo1.pdf");
  loadingTask.promise.then((pdf) => {
    pdf.getPage(1).then(async (page) => {
      const ops = await page.getOperatorList();
      const OPSReverse = {};
      for (const key in pdfjs.OPS) {
        OPSReverse[pdfjs.OPS[key]] = key;
      }
      console.log(OPSReverse);
      let strokes = [];
      let lastStrock = null;
      ops.fnArray.forEach((fn, i) => {
        if (fn === pdfjs.OPS.stroke) {
          if (lastStrock?.path && lastStrock?.width) {
            strokes.push(lastStrock);
            lastStrock = {};
          } else {
            lastStrock = {};
          }
        }
        if (lastStrock && fn === pdfjs.OPS.constructPath) {
          lastStrock.path = ops.argsArray[i];
        }
        if (lastStrock && fn === pdfjs.OPS.setLineWidth) {
          lastStrock.width = ops.argsArray[i] || 5;
        }
        if (i< 100) {
          console.log(OPSReverse[fn],ops.argsArray[i]);
        }
      });
      // console.log('stroke', strokes);
      strokes.forEach((stroke) => {
        // const points = [
        //   new THREE.Vector3(stroke.path[1][0], stroke.path[1][1], 0),
        //   new THREE.Vector3(stroke.path[1][2], stroke.path[1][3], 0),
        //   new THREE.Vector3(stroke.path[2][0], stroke.path[2][1], 0),
        //   new THREE.Vector3(stroke.path[2][2], stroke.path[2][3], 0)
        // ];
        // points.push(new THREE.Vector3(0, 0, 0)); // 起点
        // points.push(new THREE.Vector3(1, 1, 1)); // 终点
        // const geometry = new THREE.BufferGeometry().setFromPoints(points);
        // // 创建线的基本材质
        // const material = new THREE.LineBasicMaterial({ color: 0x000000 }); // 蓝色
        // material.linewidth = stroke.width?.at(0) || 0;
        // // 创建线
        // const line = new THREE.Line(geometry, material);
        // // 将线添加到场景中
        // scene.add(line);

        const path = new THREE.Path();
        path.moveTo(stroke.path[1][0] / 10, stroke.path[1][1] / 10);
        path.lineTo(stroke.path[1][2] / 10, stroke.path[1][3] / 10);
        // path.moveTo(stroke.path[2][0]/10, stroke.path[2][1]/10);
        // path.lineTo(stroke.path[2][2]/10, stroke.path[2][3]/10);
        const points = path.getPoints();
        const geometry = new THREE.BufferGeometry().setFromPoints(points);
        const material = new THREE.LineBasicMaterial({
          color: 0x00ff00,
          linewidth: 10,
        });

        const line = new THREE.Line(geometry, material);
        scene.add(line);
      });
      console.log(strokes.at(0));
      // const geometry = new THREE.BoxGeometry(1, 1, 1);
      // const material = new THREE.MeshBasicMaterial({ color: 0x00ff00 });
      // const cube = new THREE.Mesh(geometry, material);
      // scene.add(cube);

      camera.position.z = 1000;
      camera.position.y = 500;
      camera.position.x = 500;

      function animate() {
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
      }
      animate();
    });
  });
});
const handleLeft = () => {
  camera.position.x += 10;
};
const handleUp = () => {
  camera.position.y -= 10;
};
const handleDown = () => {
  camera.position.y += 10;
};
const handleRight = () => {
  camera.position.x -= 10;
};
const handleZoomIn = () => {
  let nextLevel = camera.position.z - 50;
  if (nextLevel < 0) {
    nextLevel = camera.position.z / 5
  }
  camera.position.z = nextLevel;
};
const handleZoomOut = () => {
  camera.position.z += 50;
};
</script>

<template>
  <div class="bar">
    <div @click="handleLeft">左</div>
    <div @click="handleUp">上</div>
    <div @click="handleDown">下</div>
    <div @click="handleRight">右</div>
    <div @click="handleZoomIn">前进</div>
    <div @click="handleZoomOut">后退</div>
  </div>
</template>

<style scoped>
.bar {
  position: absolute;
  bottom: 0px;
  left: 16px;
}
.bar div {
  margin: 8px;
  background: white;
}
</style>
