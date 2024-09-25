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

const opNames = ["setLineWidth", "setStrokeRGBColor", "constructPath", "setFillRGBColor", "stroke", "fill", "fillStroke"]

const opTextGroups = ["beginText", "dependency", "setFont", "setTextMatrix", "showText", "endText"]

function convertOpsToThree(ops) {
  // console.log("ops", ops);
  let context = {};
  const threeObjs = [];
  let points = null;
  ops.forEach((op,opIndex) => {
    const opName = op.at(0);
    const opArgs = op.at(1);
    if (opName === opNames.at(0)) {
      context.lineWidth = opArgs[0];
    }
    if (opName === opNames.at(1)) {
      context.color = opArgs;
    }
    const path = new THREE.Path();
    if (opName === opNames.at(2)) {
      const innerOps = opArgs.at(0);
      const innerOsArgs = opArgs.at(1);
      if (innerOps.at(0) === "rectangle") {
        const startX = innerOsArgs.at(0);
        const startY = innerOsArgs.at(1);
        const height = innerOsArgs.at(2);
        const width = innerOsArgs.at(3);
        // 一个是从上到下，一个是从下到上
        path.moveTo(startX, startY);
        path.lineTo(startX - width, startY);
        path.lineTo(startX - width, startY - height);
        path.lineTo(startX, startY - height);
        path.closePath();
        points = path.getPoints();
        context.type = "rectangle";
      } else {
        let startIndex = 0
        innerOps.forEach((o, i) => {
          let v1 = innerOsArgs[startIndex * 2];
          let v2 = innerOsArgs[startIndex * 2 + 1];
          if (o === "moveTo") {
            path.moveTo(v1, v2);
          }
          if (o === "lineTo") {
            path.lineTo(v1, v2);
          }
          if (o === "curveTo") {
            const v3 = innerOsArgs[startIndex * 2 + 2];
            const v4 = innerOsArgs[startIndex * 2 + 3];
            const v5 = innerOsArgs[startIndex * 2 + 4];
            const v6 = innerOsArgs[startIndex * 2 + 5];
            path.bezierCurveTo(v1, v2,v3,v4,v5,v6);
            startIndex = startIndex + 2;
          }
          if (o === "closePath") {
            path.closePath();
            context.type = "shape";
          }
          startIndex++;
        });
        points = path.getPoints();
      }
    }
    if (opName === opNames.at(3)) {
      context.background = opArgs;
    }
    if ([opNames.at(4), opNames.at(5), opNames.at(6)].includes(opName)) {
      if (opName === "stroke") {
        const materialLine = new THREE.LineBasicMaterial({
          linewidth: context.lineWidth,
        });
        if (context.color) {
          materialLine.color = new THREE.Color(...context.color);
        }
        const line = new THREE.Line(
          new THREE.BufferGeometry().setFromPoints(points),
          materialLine
        );
        threeObjs.push(line);
      } else if (opName === "fill") {
        const geometry = new THREE.ShapeGeometry(new THREE.Shape(points));
        if (!context.background) {
          context.background = context.color?? [255, 255, 255]
        }
        // 创建一个基础材质
        var material = new THREE.MeshBasicMaterial({
          color: new THREE.Color(...context.background),
        }); // 设置颜色为红色
        // 创建一个Mesh对象，将多边形几何体和材质传入
        var mesh = new THREE.Mesh(geometry, material);
        
        threeObjs.push(mesh);
      } else if (opName === "fillStroke") {
        var geometry = new THREE.ShapeGeometry(new THREE.Shape(points));
        const background = context.background ??context.color?? [255, 255, 255];
        // 创建一个基础材质
        var material = new THREE.MeshBasicMaterial({
          color: new THREE.Color(...background),
        }); // 设置颜色为红色
        // 创建一个Mesh对象，将多边形几何体和材质传入
        var mesh = new THREE.Mesh(geometry, material);
        threeObjs.push(mesh);
        if (context.color) {
          const materialLine = new THREE.LineBasicMaterial({
            linewidth: context.lineWidth,
            color: new THREE.Color(...context.color),
          });
          const line = new THREE.Line(
            new THREE.BufferGeometry().setFromPoints(points),
            materialLine
          );
          threeObjs.push(line);
        }
      }
      // if (context.type === 'rectangle') {
      //   const geometry = new THREE.ShapeGeometry(new THREE.Shape(points));
      //   const background = context.background??[255,255,255]
      //   // 创建一个基础材质
      //   var material = new THREE.MeshBasicMaterial({ color: new THREE.Color(...background)}); // 设置颜色为红色
      //   // 创建一个Mesh对象，将多边形几何体和材质传入
      //   var mesh = new THREE.Mesh(geometry, material);
      //   threeObjs.push(mesh);
      //   if (context.color) {
      //     const materialLine = new THREE.LineBasicMaterial({
      //       linewidth: context.lineWidth,
      //       color: new THREE.Color(...context.color),
      //     });
      //     const line = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points), materialLine);
      //     threeObjs.push(line);
      //   }
      // } else if (context.type === 'shape'){
      //   var geometry = new THREE.ShapeGeometry(new THREE.Shape(points));
      //   const background = context.background??context.color??[255,255,255]
      //   // 创建一个基础材质
      //   var material = new THREE.MeshBasicMaterial({ color: new THREE.Color(...background)}); // 设置颜色为红色
      //   // 创建一个Mesh对象，将多边形几何体和材质传入
      //   var mesh = new THREE.Mesh(geometry, material);
      //   threeObjs.push(mesh);
      //   if (context.color) {
      //     const materialLine = new THREE.LineBasicMaterial({
      //       linewidth: context.lineWidth,
      //       color: new THREE.Color(...context.color),
      //     });
      //     const line = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points), materialLine);
      //     threeObjs.push(line);
      //   }
      // } else {
      //   const materialLine = new THREE.LineBasicMaterial({
      //       linewidth: context.lineWidth,
      //       color: new THREE.Color(...context.color),
      //     });
      //     const line = new THREE.Line(new THREE.BufferGeometry().setFromPoints(points), materialLine);
      //     threeObjs.push(line);
      // }
    }
    // if(opTextGroups.includes(opName)) {
    //   console.log(opName, opArgs,op)
    // }
    // if (!opNames.includes(opName) && !opTextGroups.includes(opName)) {
    //   console.log("pre", ops[opIndex - 1])
    //   console.log(opName)
    //   console.log("next", ops[opIndex+ 1])
    // }
    
  });
  console.log('ops',ops)
  return threeObjs;
}

onMounted(() => {
  document.body.appendChild(renderer.domElement);
  // const loadingTask = pdfjs.getDocument("http://localhost:5173/Rectangles.pdf");
  // const loadingTask = pdfjs.getDocument("http://localhost:5173/Triangles.pdf");
  // const loadingTask = pdfjs.getDocument("http://localhost:5173/Lines.pdf");
  const loadingTask = pdfjs.getDocument("http://localhost:5173/demo1.pdf");
  // const loadingTask = pdfjs.getDocument("http://localhost:5173/Circles.pdf");
  loadingTask.promise.then((pdf) => {
    pdf.getPage(1).then(async (page) => {
      const ops = await page.getOperatorList();
      const OPSReverse = {};
      for (const key in pdfjs.OPS) {
        OPSReverse[pdfjs.OPS[key]] = key;
      }
      // console.log(OPSReverse);
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
      console.log(groups)
      const demoG = []
      let mergeGeometry = new THREE.BufferGeometry()
      groups.forEach((g) => {
        convertOpsToThree(g).forEach((obj,index) => {
          // meshes[i].updateMatrix();
          // mergeGeometry.merge(obj.geometry, obj.matrix, index)
          // demoG.push(obj)
          scene.add(obj);
        });
      });
      scene.add(mergeGeometry)
      camera.position.z = 1000;
      camera.position.y = 500;
      camera.position.x = 500;
      renderer.render(scene, camera);
    });
  });
});
const handleLeft = () => {
  camera.position.x += 1000;
  renderer.render(scene, camera);
};
const handleUp = () => {
  camera.position.y -= 1000;
  renderer.render(scene, camera);
};
const handleDown = () => {
  camera.position.y += 1000;
  renderer.render(scene, camera);
};
const handleRight = () => {
  camera.position.x -= 1000;
  renderer.render(scene, camera);
};
const handleZoomIn = () => {
  let nextLevel = camera.position.z - 50;
  if (nextLevel < 0) {
    nextLevel = camera.position.z / 5;
  }
  camera.position.z = nextLevel;
  renderer.render(scene, camera);
};
const handleZoomOut = () => {
  camera.position.z += 50;
  renderer.render(scene, camera);
};
// 监听鼠标滚轮事件
    function onMouseWheel(event) {
      // 检测滚轮方向
      const delta = event.deltaY > 0 ? 1 : -1;

      // 调整相机位置以实现放大缩小
      camera.position.z += delta * 200;
      renderer.render(scene, camera);
    }

onMounted(() => {
    window.addEventListener('wheel', onMouseWheel);
})

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
