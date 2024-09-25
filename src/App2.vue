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
  8000
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
  const loadingTask = pdfjs.getDocument("http://localhost:5173/Circles.pdf");
  loadingTask.promise.then((pdf) => {
    pdf.getPage(1).then(async (page) => {
      const ops = await page.getOperatorList();
      const OPSReverse = {};
      for (const key in pdfjs.OPS) {
        OPSReverse[pdfjs.OPS[key]] = key;
      }
      console.log(OPSReverse);
      let groups = [];
      let crr = [];
      //ops.fnArray.length
      const endIndex = 100;
      for (let i = 0; i < endIndex; i++) {
        const fn = ops.fnArray[i];
        if (fn === 10) {
          crr = [];
          continue;
        }
        if (fn === 11) {
          groups.push(crr);
          continue;
        }
        const arg = ops.argsArray[i];
        if (Array.isArray(arg) && Array.isArray(arg.at(0))) {
          arg[0] = arg[0].map((a) => OPSReverse[a]);
        }
        crr.push([OPSReverse[fn], arg]);
      }
      console.log(groups);

      groups.forEach((g) => {
        let lastColor = null;
        const paths = []
        g.forEach((action) => {
          const opt = action.at(0);
          if (opt === "constructPath") {
            const pathVal = action.at(1);
            const pathOpts = pathVal.at(0);
            const pathOptValues = pathVal.at(1);
            const path = new THREE.Path();

            pathOpts.forEach((o, i) => {
              const v1 = pathOptValues[i * 2] / 10;
              const v2 = pathOptValues[i * 2 + 1]/10;
              if (o === "moveTo") {
                path.moveTo(v1, v2);
              }
              if (o === "lineTo") {
                path.lineTo(v1, v2);
              }
              if (o === "closePath") {
                path.lineTo(v1, v2);
                path.closePath();
              }
            });
            const points = path.getPoints();
            const geometry = new THREE.BufferGeometry().setFromPoints(points);
            const material = new THREE.LineBasicMaterial({
              linewidth: 10,
            });
            if (lastColor) {
              material.color = new THREE.Color(...lastColor);
            }
            const line = new THREE.Line(geometry, material);
            scene.add(line);
          }
          if (opt === "setFillRGBColor") {
            lastColor = action.at(1);
          }
        });
      });

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
    nextLevel = camera.position.z / 5;
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
