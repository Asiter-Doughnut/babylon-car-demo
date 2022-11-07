<script setup lang="ts">
import * as BABYLON from '@babylonjs/core';
import "@babylonjs/loaders";
import { nextTick, onMounted, ref } from 'vue';
const mycanvas = ref<HTMLCanvasElement>();
onMounted(() => {
  if (mycanvas.value) {
    const engine = new BABYLON.Engine(mycanvas.value, true, {
      preserveDrawingBuffer: true, stencil: true
    })

    const createScene = function () {

      // This creates a basic Babylon Scene object (non-mesh)
      var scene = new BABYLON.Scene(engine);

      // add skyBox
      scene.environmentTexture = BABYLON.CubeTexture.CreateFromPrefilteredData('/environment.dds', scene);
      scene.createDefaultSkybox(scene.environmentTexture);
      // Skybox
      // const skybox = BABYLON.MeshBuilder.CreateBox("skyBox", { size: 1000.0 }, scene);
      // const skyboxMaterial = new BABYLON.StandardMaterial("skyBox", scene);
      // skyboxMaterial.backFaceCulling = false;
      // skyboxMaterial.reflectionTexture = new BABYLON.CubeTexture("/skybox", scene);
      // skyboxMaterial.reflectionTexture.coordinatesMode = BABYLON.Texture.SKYBOX_MODE;
      // skyboxMaterial.diffuseColor = new BABYLON.Color3(0, 0, 0);
      // skyboxMaterial.specularColor = new BABYLON.Color3(0, 0, 0);
      // skybox.material = skyboxMaterial;

      //camera
      const camera = new BABYLON.ArcRotateCamera("Camera", Math.PI / 3, Math.PI / 3, 10, BABYLON.Vector3.Zero(), scene);
      // const camera = new BABYLON.FollowCamera("FollowCam", new BABYLON.Vector3(0, 10, -10), scene);
      camera.attachControl(true)

      // add _root_ to boody
      let carBody: BABYLON.AbstractMesh | null;
      let pivot: BABYLON.AbstractMesh | null;
      // get wheel group
      let Wheel_L: BABYLON.AbstractMesh | null;
      let Wheel_R: BABYLON.AbstractMesh | null;
      let Wheel_Back_L: BABYLON.AbstractMesh | null;
      let Wheel_Back_R: BABYLON.AbstractMesh | null;

      // This creates a light, aiming 0,1,0 - to the sky (non-mesh)
      const light = new BABYLON.HemisphericLight("light", new BABYLON.Vector3(0, 1, 0), scene);


      // Ground
      const ground = BABYLON.MeshBuilder.CreateGround("ground", { height: 500, width: 500, subdivisions: 4 }, scene);
      const groundMaterial = new BABYLON.StandardMaterial("groundMaterial", scene);
      groundMaterial.diffuseTexture = new BABYLON.Texture("https://playground.babylonjs.com/textures/wood.jpg", scene);
      (groundMaterial.diffuseTexture as any).uScale = 30;
      (groundMaterial.diffuseTexture as any).vScale = 30;
      groundMaterial.specularColor = new BABYLON.Color3(.1, .1, .1);
      ground.material = groundMaterial;

      BABYLON.SceneLoader.Append("/", "./lamborghini1.glb", scene, (scene) => {

        Wheel_L = scene.getMeshById('Wheel_L');
        Wheel_R = scene.getMeshById('Wheel_R');
        Wheel_Back_L = scene.getMeshById('Wheel_Back_L');
        Wheel_Back_R = scene.getMeshById('Wheel_Back_R');
        carBody = scene.getMeshById("__root__");

        camera.lockedTarget = carBody;
        pivot = new BABYLON.Mesh("pivot", scene); //添加车身父节点


        // pivot = BABYLON.MeshBuilder.CreateBox("box4", { size: 0.8 }, scene);
        // pivot.showBoundingBox = true; //边框大小
        pivot.position.z = carBody!.getPositionExpressedInLocalSpace().z;
        carBody!.parent = pivot;
        carBody!.position = new BABYLON.Vector3(0, 0, 0);

        /*-------------------Pivots for Front Wheels-----------------------------------*/
        const pivotFI = new BABYLON.Mesh("pivotFI", scene)
        pivotFI.parent = carBody;
        pivotFI.position = Wheel_L!.getPositionExpressedInLocalSpace();
        Wheel_L!.parent = pivotFI;
        Wheel_L!.position = new BABYLON.Vector3(0, 0, 0);

        const pivotFO = new BABYLON.Mesh("pivotFO", scene);
        pivotFO.parent = carBody;
        pivotFO.position = Wheel_R!.getPositionExpressedInLocalSpace();
        Wheel_R!.parent = pivotFO;
        Wheel_R!.position = new BABYLON.Vector3(0, 0, 0);
        /*-------------------Pivots for Front Wheels-----------------------------------*/


        /*************** 添加键盘控制 ***************/
        const keyboardMap: any = {};
        scene.actionManager = new BABYLON.ActionManager(scene);
        //添加按钮识别
        scene.actionManager.registerAction(new BABYLON.ExecuteCodeAction(BABYLON.ActionManager.OnKeyDownTrigger, (evt) => {
          keyboardMap[evt.sourceEvent.key] = evt.sourceEvent.type == "keydown";
        }))
        scene.actionManager.registerAction(new BABYLON.ExecuteCodeAction(BABYLON.ActionManager.OnKeyUpTrigger, (evt) => {
          keyboardMap[evt.sourceEvent.key] = evt.sourceEvent.type == "keydown";
        }))
        /** 结束添加 */

        /** 数据设置 */
        let theta = 0; //偏移角度
        let deltaTheta = 0; //每次偏移的角度
        let D = 0;
        let bD = 0;
        let r = 0.35; // 车轮大小 radius
        let L = 1.6;
        let A = 2.7;
        let R = 0; //旋转
        let psi, psiRI, psiRO, psiFI, psiFO; //车辆组
        let NR; //车轮旋转角度
        scene.registerAfterRender(() => {
          const F = engine.getFps();
          //速度设置
          //前进
          if ((keyboardMap["w"] || keyboardMap["W"]) && D < 30) {
            if (bD > 0) {
              bD -= 2;
              return;
            }
            else {
              bD = 0;
            }
            D += 2;
          };
          if (D > 0.15 && bD == 0) {
            D -= 0.3;
          }
          else {
            D = 0;
          }
          // 后退
          if ((keyboardMap["s"] || keyboardMap["S"]) && bD < 30) {
            if (D > 0) {
              D -= 2;
              return;
            }
            else {
              D = 0;
            }
            bD += 2;
          };
          if (bD > 0.3 && D == 0) {
            bD -= 0.3;
          }
          else {
            bD = 0;
          }

          const distance = D / F;
          const BackDistance = -bD / F;
          const psi = D / (r * F);
          const bpsi = bD / (r * F);
          if ((keyboardMap["a"] || keyboardMap["A"]) && -Math.PI / 10 < theta) {
            deltaTheta = -Math.PI / 360;
            theta += deltaTheta;
            pivotFI.rotate(BABYLON.Axis.Y, -deltaTheta, BABYLON.Space.LOCAL);
            pivotFO.rotate(BABYLON.Axis.Y, -deltaTheta, BABYLON.Space.LOCAL);

            if (Math.abs(theta) > 0.00000001) {
              console.log('L / Math.tan(theta)', L / Math.tan(theta));
              NR = A / 2 + L / Math.tan(theta);
            }
            else {
              theta = 0;
              NR = 0;
            }
            pivot!.translate(BABYLON.Axis.X, -R + NR, BABYLON.Space.LOCAL);
            carBody!.translate(BABYLON.Axis.X, - R + NR, BABYLON.Space.LOCAL);
            R = NR;
          };

          if ((keyboardMap["d"] || keyboardMap["D"]) && theta < Math.PI / 10) {
            deltaTheta = Math.PI / 360;
            theta += deltaTheta;
            pivotFI.rotate(BABYLON.Axis.Y, -deltaTheta, BABYLON.Space.LOCAL);
            pivotFO.rotate(BABYLON.Axis.Y, -deltaTheta, BABYLON.Space.LOCAL);

            if (Math.abs(theta) > 0.00000001) {
              NR = A / 2 + L / Math.tan(theta);
            }
            else {
              theta = 0;
              NR = 0;
            }
            pivot!.translate(BABYLON.Axis.X, -R + NR, BABYLON.Space.LOCAL);
            carBody!.translate(BABYLON.Axis.X, - R + NR, BABYLON.Space.LOCAL);
            R = NR;
          };
          //前进  
          if (D > 0) {
            const phi = D / (R * F)
            if (Math.abs(theta) > 0) {

              pivot!.rotate(BABYLON.Axis.Y, phi, BABYLON.Space.WORLD);
              psiRI = D / (r * F);
              psiRO = D * (R + A) / (r * F);
              psiFI = D * Math.sqrt(R * R + L * L) / (r * F);
              psiFO = D * Math.sqrt((R + A) * (R + A) + L * L) / (r * F);

              Wheel_L?.rotate(BABYLON.Axis.X, psiRI, BABYLON.Space.LOCAL);
              Wheel_R?.rotate(BABYLON.Axis.X, psiRO, BABYLON.Space.LOCAL);
              Wheel_Back_L?.rotate(BABYLON.Axis.X, psiFI, BABYLON.Space.LOCAL);
              Wheel_Back_R?.rotate(BABYLON.Axis.X, psiFO, BABYLON.Space.LOCAL);
            } else {
              pivot!.translate(BABYLON.Axis.Z, distance, BABYLON.Space.LOCAL);
              // carBody!.translate(BABYLON.Axis.Z, distance, BABYLON.Space.LOCAL);
              Wheel_L?.rotate(BABYLON.Axis.X, psi, BABYLON.Space.LOCAL);
              Wheel_R?.rotate(BABYLON.Axis.X, psi, BABYLON.Space.LOCAL);
              Wheel_Back_L?.rotate(BABYLON.Axis.X, psi, BABYLON.Space.LOCAL);
              Wheel_Back_R?.rotate(BABYLON.Axis.X, psi, BABYLON.Space.LOCAL);
            }

          }
          if (bD > 0) {
            // // carBody!.translate(BABYLON.Axis.Z, BackDistance, BABYLON.Space.LOCAL);
            // pivot!.translate(BABYLON.Axis.Z, BackDistance, BABYLON.Space.LOCAL);
            // Wheel_L?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
            // Wheel_R?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
            // Wheel_Back_L?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
            // Wheel_Back_R?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
            const phi = bD / (R * F)
            if (Math.abs(theta) > 0) {
              pivot!.rotate(BABYLON.Axis.Y, -phi, BABYLON.Space.WORLD);
              psiRI = bD / (r * F);
              psiRO = bD * (R + A) / (r * F);
              psiFI = bD * Math.sqrt(R * R + L * L) / (r * F);
              psiFO = bD * Math.sqrt((R + A) * (R + A) + L * L) / (r * F);

              Wheel_L?.rotate(BABYLON.Axis.X, psiRI, BABYLON.Space.LOCAL);
              Wheel_R?.rotate(BABYLON.Axis.X, psiRO, BABYLON.Space.LOCAL);
              Wheel_Back_L?.rotate(BABYLON.Axis.X, psiFI, BABYLON.Space.LOCAL);
              Wheel_Back_R?.rotate(BABYLON.Axis.X, psiFO, BABYLON.Space.LOCAL);
            } else {
              pivot!.translate(BABYLON.Axis.Z, BackDistance, BABYLON.Space.LOCAL);
              // carBody!.translate(BABYLON.Axis.Z, distance, BABYLON.Space.LOCAL);
              Wheel_L?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
              Wheel_R?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
              Wheel_Back_L?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
              Wheel_Back_R?.rotate(BABYLON.Axis.X, -bpsi, BABYLON.Space.LOCAL);
            }
          }
        })
      });
      return scene;
    }

    const scene = createScene();
    // run the render loop
    engine.runRenderLoop(function () {

      scene.render();

    });
  }
})
</script>

<template>
  <canvas class="mycanvas" ref="mycanvas">

  </canvas>

</template>

<style scoped>
.mycanvas {
  height: 100vh;
  width: 100vw;
  background-color: aqua;
}
</style>
