<script setup>
import { ref, onMounted } from "vue";
import * as pdfjs from "pdfjs-dist";
pdfjs.GlobalWorkerOptions.workerSrc = "/pdf.worker.min.js";
import * as THREE from "three";

const scene = new THREE.Scene();
scene.background = new THREE.Color(0xfffffff);
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

function convertOpsToThree(ops) {
  console.log("ops", ops);
  const context = {};
  const threeObjs = [];
  let points = null
  ops.forEach((op) => {
    const opName = op.at(0);
    const opArgs = op.at(1);
    if (opName === "setLineWidth") {
      context.lineWidth = opArgs[0];
    }
    if (opName === "setStrokeRGBColor") {
      context.color = opArgs;
    }
    if (opName === "constructPath") {
      const innerOps = opArgs.at(0);
      const innerOsArgs = opArgs.at(1);
      const path = new THREE.Path();
      innerOps.forEach((o, i) => {
        const v1 = innerOsArgs[i * 2];
        const v2 = innerOsArgs[i * 2 + 1];
        if (o === "moveTo") {
          path.moveTo(v1, v2);
        }
        if (o === "lineTo") {
          path.lineTo(v1, v2);
        }
        if (o === "closePath") {
          path.closePath();
          context.isShape = true
        }
      });
      points = path.getPoints();
    }
    if (opName === "setFillRGBColor") {
      context.background = opArgs;
    }
    if (opName === "fillStroke") {
      if(context.isShape) {
        var geometry = new THREE.ShapeGeometry(new THREE.Shape(points));
        const background = context.background??context.color
        // 创建一个基础材质
        var material = new THREE.MeshBasicMaterial({ color: new THREE.Color(...background)}); // 设置颜色为红色
        // 创建一个Mesh对象，将多边形几何体和材质传入
        var mesh = new THREE.Mesh(geometry, material);
        threeObjs.push(mesh);
        if (context.color) {
          const materialLine = new THREE.LineBasicMaterial({
            linewidth: context.lineWidth,
            color: new THREE.Color(...context.color),
          });
          const line = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points), materialLine);
           threeObjs.push(line);
        }
      }
    }
  });
  return threeObjs;
}

onMounted(() => {
  document.body.appendChild(renderer.domElement);
  const loadingTask = pdfjs.getDocument("http://localhost:5173/Triangles.pdf");
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
      const endIndex = ops.fnArray.length;
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
      if (crr.length > 0) {
        groups.push(crr);
      }
      groups.forEach((g) => {
        convertOpsToThree(g).forEach((obj) => {
          scene.add(obj);
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
