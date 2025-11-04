# newok
just do it
[20251101AR01.html](https://github.com/user-attachments/files/23322407/20251101AR01.html)
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="utf-8">
<title>WebAR 多阶段安装说明书</title>
<meta name="viewport" content="width=device-width,initial-scale=1,user-scalable=no">
<style>
body{margin:0;overflow:hidden;font-family:Arial}
#info{position:absolute;top:10px;left:10px;color:#fff;background:#0006;padding:8px;border-radius:6px;z-index:10}
#ctrl{position:absolute;bottom:20px;left:50%;transform:translateX(-50%);z-index:10}
button{height:40px;font-size:18px;margin:0 5px}
</style>
</head>
<body>
<div id="info">请依次点击屏幕 3 次（①原点 ②1m方向 ③水平面）</div>
<div id="ctrl" style="display:none">
  <button id="prev">← 上一步</button>
  <button id="next">下一步 →</button>
</div>

<!-- 三大件 -->
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/loaders/GLTFLoader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.160.0/examples/js/controls/OrbitControls.js"></script>

<script>
/* ========== 0. 全局变量 ========== */
let camera, scene, renderer, controls;
let reticle=[], hitTestSource, hitTestSourceRequested=false;
let stage=0, clickCount=0;
let models=[];          // 3 个阶段模型
const origin=new THREE.Vector3();
const xAxis=new THREE.Vector3();
const zAxis=new THREE.Vector3();

/* ========== 1. 初始化 Three.js ========== */
function initTHREE(){
  scene=new THREE.Scene();
  camera=new THREE.PerspectiveCamera(70,innerWidth/innerHeight,0.01,100);
  camera.position.set(0,1.6,0);

  renderer=new THREE.WebGLRenderer({antialias:true,alpha:true});
  renderer.setSize(innerWidth,innerHeight);
  renderer.xr.enabled=true;
  document.body.appendChild(renderer.domElement);

  const light=new THREE.HemisphereLight(0xffffff,0x444444,1);
  scene.add(light);

  window.addEventListener('resize',()=>{
    camera.aspect=innerWidth/innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(innerWidth,innerHeight);
  });
}

/* ========== 2. WebXR 会话 ========== */
async function startXR(){
  if(!navigator.xr){
    alert('当前浏览器不支持 WebXR');
    return;
  }
  const session=await navigator.xr.requestSession('immersive-ar',{
    requiredFeatures:['hit-test','dom-overlay'],
    domOverlay:{root:document.body}
  });
  await renderer.xr.setSession(session);
  session.requestReferenceSpace('local').then(refSpace=>{
    session.requestHitTestSource({space:refSpace}).then(source=>{
      hitTestSource=source;
    });
  });
  renderer.setAnimationLoop(render);
}

/* ========== 3. 加载 3 个阶段模型 ========== */
const loader=new THREE.GLTFLoader();
const names=['step1.glb','step2.glb','step3.glb'];
let loaded=0;
names.forEach((url,i)=>{
  loader.load(url,gltf=>{
    models[i]=gltf.scene;
    models[i].visible=false;
    scene.add(models[i]);
    if(++loaded===3) readyToPlace();
  },undefined,err=>alert('模型加载失败：'+url+' '+err));
});

/* ========== 4. 用户点 3 次 → 建立坐标系 ========== */
function readyToPlace(){
  const info=document.getElementById('info');
  renderer.domElement.addEventListener('click',async e=>{
    if(clickCount>=3) return;
    if(!hitTestSource){ alert('hit-test 还未准备好');return; }
    const session=renderer.xr.getSession();
    const refSpace=await session.requestReferenceSpace('viewer');
    const hit=await session.requestHitTest(e.clientX,e.clientY,refSpace);
    if(hit.length){
      const p=hit[0].getPose(await session.requestReferenceSpace('local')).transform.position;
      reticle.push(new THREE.Vector3(p.x,p.y,p.z));
      clickCount++;
      if(clickCount===1){
        origin.copy(reticle[0]);
        info.innerText='② 在距离原点约 1 米处点击';
      }else if(clickCount===2){
        xAxis.subVectors(reticle[1],origin).normalize();
        info.innerText='③ 在水平面另一点点击';
      }else if(clickCount===3){
        const tmp=new THREE.Vector3().subVectors(reticle[2],origin).normalize();
        zAxis.crossVectors(xAxis,tmp).normalize();   // 水平面法向量
        showUI();
      }
    }
  });
}

/* ========== 5. 显示 UI 并默认展示阶段 0 ========== */
function showUI(){
  document.getElementById('info').style.display='none';
  document.getElementById('ctrl').style.display='flex';
  setStage(0);
}
function setStage(n){
  models.forEach((m,i)=>m.visible=i===n);
  stage=n;
}
document.getElementById('prev').onclick=()=>setStage((stage+2)%3);
document.getElementById('next').onclick=()=>setStage((stage+1)%3);

/* ========== 6. 渲染循环 ========== */
function render(){
  renderer.render(scene,camera);
}

/* ========== 7. 启动 ========== */
initTHREE();
startXR();
</script>
</body>
</html>
