<!doctype html>
<html lang="de">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Bike-GTA (Prototype)</title>
<style>
  :root { --hud-bg: rgba(0,0,0,0.5); --hud-color: #fff; }
  body { margin:0; font-family:system-ui,Arial; height:200vh; /* hohe Seite: scrollen möglich */ }
  #gameCanvas { position:fixed; left:0; top:0; width:100vw; height:100vh; display:block; z-index:0; }
  #overlay { position:fixed; left:12px; top:12px; z-index:5; background:var(--hud-bg); color:var(--hud-color); padding:10px; border-radius:8px; max-width:320px; }
  #hud { position:fixed; right:12px; top:12px; z-index:5; background:var(--hud-bg); color:var(--hud-color); padding:10px;border-radius:8px;text-align:right; }
  #message { position:fixed; left:50%; bottom:18px; transform:translateX(-50%); z-index:5; background:var(--hud-bg); padding:10px 14px;border-radius:6px;color:var(--hud-color); display:none; }
  #minimap { position:fixed; right:12px; bottom:12px; width:160px; height:160px; background:rgba(255,255,255,0.06); border-radius:8px; z-index:5; }
  #controlsHint { font-size:13px; opacity:0.9; }
  #pageContent { position:relative; margin-top:100vh; background:#fafafa; padding:30px; z-index:1; }
  button { cursor:pointer; }
  .stat { font-weight:700; }
  #lockBtn { margin-top:8px; display:inline-block; padding:6px 10px; border-radius:6px; border:0; background:#2b8cff;color:white; }
</style>
</head>
<body>

<canvas id="gameCanvas"></canvas>

<div id="overlay">
  <div><strong>Bike-GTA — Prototype</strong></div>
  <div id="controlsHint">
    W/A/S/D: laufen/fahren • Maus: schauen • E: interagieren (Fahrrad stehlen / aufsteigen) • Shift: sprint • Space: Handbremse
  </div>
  <div style="margin-top:8px;">
    <button id="lockBtn">Maus sperren (Play)</button>
    <button id="spawnBtn">Mehr Fahrräder</button>
  </div>
</div>

<div id="hud">
  Geld: <span id="money">0</span> $<br>
  Gestohlene Fahrräder: <span id="stolen">0</span><br>
  Status: <span id="status">zu Fuß</span>
</div>

<div id="minimap"><canvas id="minimapCanvas" width="160" height="160"></canvas></div>
<div id="message"></div>

<!-- Seite weiter unten: Missions / Einstellungen (scrollbar demonstriert) -->
<div id="pageContent">
  <h2>Spiel-Info & Einstellungen</h2>
  <p>Die Seite ist bewusst länger gemacht, damit du nach unten scrollen kannst ohne dass das Spiel-Zeichenfeld verschwindet. Scrollen funktioniert normal solange die Maus *nicht* gesperrt ist. Mit dem Button <em>Maus sperren (Play)</em> aktivierst du PointerLock und das Spiel bekommt Maus-Eingabe.</p>
  <h3>Missionen</h3>
  <ul>
    <li>Stehle 5 Fahrräder und liefere sie zur Sammelstelle (Belohnung: 300$)</li>
    <li>Suche spezielle blaue Fahrräder (höherer Wert)</li>
  </ul>
  <h3>Hinweise</h3>
  <p>Dies ist ein Prototype. Wenn du möchtest, kann ich Multiplayer-Stuff, bessere Fahrphysik, Sounds, NPCs, Mission-Log und Map-Objekte hinzufügen.</p>
</div>

<!-- Three.js -->
<script src="https://cdn.jsdelivr.net/npm/three@0.155.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.155.0/examples/js/controls/PointerLockControls.js"></script>

<script>
// ---------- Globals ----------
const canvas = document.getElementById('gameCanvas');
const renderer = new THREE.WebGLRenderer({canvas, antialias:true});
renderer.setSize(window.innerWidth, window.innerHeight);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb);

const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.set(0,1.8,8);

const controls = new THREE.PointerLockControls(camera, document.body);
const lockBtn = document.getElementById('lockBtn');

let worldSize = 120;
let bikes = [];
let npcs = [];
let player = { money:0, stolen:0, mounted:null };
let move = { forward:false, back:false, left:false, right:false, sprint:false, brake:false };
let velocity = new THREE.Vector3();

// UI
const moneyEl = document.getElementById('money');
const stolenEl = document.getElementById('stolen');
const statusEl = document.getElementById('status');
const messageEl = document.getElementById('message');
const spawnBtn = document.getElementById('spawnBtn');

// Minimap
const miniCanvas = document.getElementById('minimapCanvas');
const mctx = miniCanvas.getContext('2d');

// ---------- Scene ----------
const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.9);
hemi.position.set(0,50,0); scene.add(hemi);
const dir = new THREE.DirectionalLight(0xffffff, 0.6); dir.position.set(-10,20,10); scene.add(dir);

// Boden
const groundMat = new THREE.MeshStandardMaterial({color:0x556b2f});
const ground = new THREE.Mesh(new THREE.PlaneGeometry(worldSize*2, worldSize*2), groundMat);
ground.rotation.x = -Math.PI/2; scene.add(ground);

// Gebäude / Hindernisse (einfach)
const boxGeo = new THREE.BoxGeometry(6,4,6);
for(let i=0;i<35;i++){
  const b = new THREE.Mesh(boxGeo, new THREE.MeshStandardMaterial({color:0x8b8b8b}));
  b.position.set((Math.random()-0.5)*(worldSize-20),2,(Math.random()-0.5)*(worldSize-20));
  b.rotation.y = Math.random()*Math.PI;
  scene.add(b);
}

// Spawn initial bikes
for(let i=0;i<18;i++){
  spawnBike(new THREE.Vector3((Math.random()-0.5)* (worldSize-10), 0.05, (Math.random()-0.5)*(worldSize-10)));
}

// simple NPCs (moving boxes)
for(let i=0;i<6;i++){
  const n = new THREE.Mesh(new THREE.BoxGeometry(1.5,1,3), new THREE.MeshStandardMaterial({color:0xff5555}));
  n.position.set((Math.random()-0.5)*(worldSize-10),0.5,(Math.random()-0.5)*(worldSize-10));
  n.userData = {dir: Math.random()*Math.PI*2, speed: 1 + Math.random()*1.5};
  scene.add(n); npcs.push(n);
}

// ---------- Functions ----------
function spawnBike(pos){
  const g = new THREE.Group();
  // frame
  const frame = new THREE.Mesh(new THREE.BoxGeometry(1.2,0.12,0.12), new THREE.MeshStandardMaterial({color:0x222222}));
  frame.position.set(0,0.55,0); frame.rotation.z = 0.12; g.add(frame);
  // wheels
  const wheelGeo = new THREE.TorusGeometry(0.42,0.09,8,16);
  const w1 = new THREE.Mesh(wheelGeo, new THREE.MeshStandardMaterial({color:0x000000}));
  w1.rotation.x = Math.PI/2; w1.position.set(0.65,0.22,0);
  const w2 = w1.clone(); w2.position.set(-0.65,0.22,0); g.add(w1); g.add(w2);
  // seat
  const seat = new THREE.Mesh(new THREE.BoxGeometry(0.32,0.06,0.22), new THREE.MeshStandardMaterial({color:0x331a00}));
  seat.position.set(0.0,0.78,0); g.add(seat);
  // handlebars
  const bar = new THREE.Mesh(new THREE.BoxGeometry(0.56,0.06,0.06), new THREE.MeshStandardMaterial({color:0x333333}));
  bar.position.set(0.95,0.82,0); g.add(bar);

  g.position.copy(pos);
  g.userData = {stolen:false, mounted:false, value: 30 + Math.floor(Math.random()*80)}; // value varies
  scene.add(g);
  bikes.push(g);
  return g;
}

function worldClamp(vec){
  vec.x = THREE.MathUtils.clamp(vec.x, -worldSize, worldSize);
  vec.z = THREE.MathUtils.clamp(vec.z, -worldSize, worldSize);
}

function showMessage(t, time=240){
  messageEl.style.display='block';
  messageEl.textContent = t;
  if(window._msgTimeout) clearTimeout(window._msgTimeout);
  window._msgTimeout = setTimeout(()=>{ messageEl.style.display='none'; }, time*4); // rough ms
}

function updateHUD(){
  moneyEl.textContent = player.money;
  stolenEl.textContent = player.stolen;
  statusEl.textContent = player.mounted ? 'auf Fahrrad' : 'zu Fuß';
}

// Interaction: try to steal / mount nearest bike in front of camera
const raycaster = new THREE.Raycaster();
function tryInteract(){
  const origin = camera.getWorldPosition(new THREE.Vector3());
  const dir = new THREE.Vector3(); camera.getWorldDirection(dir);
  raycaster.set(origin, dir);
  // flatten list of bike meshes
  const meshList = bikes.flatMap(b => b.children);
  const intersects = raycaster.intersectObjects(meshList, true);
  if(intersects.length > 0){
    // find parent group
    let obj = intersects[0].object;
    while(obj && !bikes.includes(obj)) obj = obj.parent;
    if(!obj) return;
    const dist = intersects[0].distance;
    if(dist > 3.2){ showMessage('Zu weit weg'); return; }
    // If not stolen -> steal it
    if(!obj.userData.stolen){
      obj.userData.stolen = true;
      player.money += obj.userData.value;
      player.stolen += 1;
      updateHUD();
      showMessage(`Fahrrad gestohlen! +${obj.userData.value} $`);
    } else {
      showMessage('Dieses Fahrrad wurde schon genommen');
    }
    // mount
    if(!obj.userData.mounted){
      player.mounted = obj;
      obj.userData.mounted = true;
      showMessage('Aufgestiegen');
      // attach camera behind bike when mounted (handled in loop)
    }
  } else {
    showMessage('Kein Fahrrad in Sicht');
  }
}

// Dismount function
function dismount(){
  if(player.mounted){
    player.mounted.userData.mounted = false;
    player.mounted = null;
    showMessage('Abgestiegen');
    updateHUD();
  }
}

// Input
window.addEventListener('keydown', (e)=>{
  if(e.code === 'KeyW') move.forward = true;
  if(e.code === 'KeyS') move.back = true;
  if(e.code === 'KeyA') move.left = true;
  if(e.code === 'KeyD') move.right = true;
  if(e.code === 'ShiftLeft') move.sprint = true;
  if(e.code === 'Space') move.brake = true;
  if(e.code === 'KeyE') tryInteract();
  if(e.code === 'KeyX') dismount();
});
window.addEventListener('keyup', (e)=>{
  if(e.code === 'KeyW') move.forward = false;
  if(e.code === 'KeyS') move.back = false;
  if(e.code === 'KeyA') move.left = false;
  if(e.code === 'KeyD') move.right = false;
  if(e.code === 'ShiftLeft') move.sprint = false;
  if(e.code === 'Space') move.brake = false;
});

// Lock controls via button
lockBtn.addEventListener('click', ()=>{
  if(document.pointerLockElement === null){
    controls.lock();
  } else {
    controls.unlock();
  }
});
controls.addEventListener('lock', ()=>{ lockBtn.textContent = 'Maus entsperren'; showMessage('Maus gesperrt — Steuerung aktiv'); });
controls.addEventListener('unlock', ()=>{ lockBtn.textContent = 'Maus sperren (Play)'; showMessage('Maus entsperrt — Seite scrollbar'); });

// spawn button
spawnBtn.addEventListener('click', ()=>{
  for(let i=0;i<6;i++){
    spawnBike(new THREE.Vector3((Math.random()-0.5)*(worldSize-10),0.05,(Math.random()-0.5)*(worldSize-10)));
  }
  showMessage('Neue Fahrräder gespawnt');
});

// Resize handling
window.addEventListener('resize', ()=>{
  renderer.setSize(window.innerWidth, window.innerHeight);
  camera.aspect = window.innerWidth/window.innerHeight; camera.updateProjectionMatrix();
});

// ---------- Game Loop ----------
let lastTime = performance.now();
function animate(t){
  requestAnimationFrame(animate);
  const dt = Math.min(0.05, (t - lastTime)/1000);
  lastTime = t;

  // NPC simple movement
  npcs.forEach(n=>{
    n.position.x += Math.cos(n.userData.dir) * n.userData.speed * dt;
    n.position.z += Math.sin(n.userData.dir) * n.userData.speed * dt;
    // bounce at borders
    if(Math.abs(n.position.x) > worldSize-5 || Math.abs(n.position.z) > worldSize-5) n.userData.dir += Math.PI;
  });

  // Player movement: if mounted -> move bike, else move controls object
  if(player.mounted){
    // simple bike steering
    const bike = player.mounted;
    // forward/back controls
    let targetSpeed = 0;
    if(move.forward) targetSpeed = 6;
    if(move.back) targetSpeed = -1.5;
    if(move.sprint) targetSpeed *= 1.5;
    // accelerate to target
    const current = velocity.length();
    const sign = (velocity.dot(new THREE.Vector3(Math.sin(bike.rotation.y),0,Math.cos(bike.rotation.y)))>=0)?1:-1;
    const forwardVec = new THREE.Vector3(Math.sin(bike.rotation.y),0,Math.cos(bike.rotation.y));
    // apply rotation from left/right
    if(move.left) bike.rotation.y += 0.03 * (move.sprint?1.4:1);
    if(move.right) bike.rotation.y -= 0.03 * (move.sprint?1.4:1);
    // apply acceleration along forward vector
    const accel = forwardVec.clone().multiplyScalar((targetSpeed - velocity.dot(forwardVec)) * 2 * dt);
    velocity.add(accel);
    // braking
    if(move.brake) velocity.multiplyScalar(0.92);
    // cap
    const max = 8 * (move.sprint?1.4:1);
    if(velocity.length() > max) velocity.setLength(max);
    // move bike position
    bike.position.addScaledVector(velocity, dt);
    // camera follows behind and slightly above
    const behind = new THREE.Vector3(0,1.2,2).applyAxisAngle(new THREE.Vector3(0,1,0), bike.rotation.y);
    const camTarget = bike.position.clone().add(behind);
    camera.position.lerp(camTarget, 0.18);
    camera.lookAt(bike.position.clone().add(new THREE.Vector3(0,0.9,0)));
    worldClamp(bike.position);
  } else {
    // on foot: use pointerlock controls object
    const obj = controls.getObject();
    // compute movement in camera direction
    const dirZ = (move.forward?1:0) - (move.back?1:0);
    const dirX = (move.right?1:0) - (move.left?1:0);
    let camDir = new THREE.Vector3(); camera.getWorldDirection(camDir); camDir.y = 0; camDir.normalize();
    const camRight = new THREE.Vector3().crossVectors(new THREE.Vector3(0,1,0), camDir).normalize();
    const moveVec = new THREE.Vector3();
    moveVec.addScaledVector(camDir, dirZ);
    moveVec.addScaledVector(camRight, dirX);
    if(moveVec.length() > 0) moveVec.normalize();
    const speed = 4 * (move.sprint?1.6:1);
    obj.position.addScaledVector(moveVec, speed * dt);
    camera.position.copy(obj.position).add(new THREE.Vector3(0,1.8,0));
    worldClamp(obj.position);
  }

  // rotate bike wheels visually
  bikes.forEach(b => {
    b.children.forEach(c => {
      // TorusGeometry check by geometry.parameters
      if(c.geometry && c.geometry.type === 'TorusGeometry') c.rotation.z += 0.2;
    });
  });

  // Mini-map render
  updateMinimap();

  // renderer
  renderer.render(scene, camera);
}
requestAnimationFrame(animate);

// ---------- Minimap ----------
function updateMinimap(){
  mctx.clearRect(0,0,miniCanvas.width,miniCanvas.height);
  // background
  mctx.fillStyle = '#082'; mctx.fillRect(0,0,miniCanvas.width,miniCanvas.height);
  // draw border center
  const scale = miniCanvas.width / (worldSize*2);
  // player dot
  let px = (camera.position.x + worldSize) * scale;
  let pz = (camera.position.z + worldSize) * scale;
  // bikes
  bikes.forEach(b=>{
    mctx.fillStyle = b.userData.stolen ? '#888' : '#ffd700';
    const bx = (b.position.x + worldSize) * scale;
    const bz = (b.position.z + worldSize) * scale;
    mctx.fillRect(bx-2, bz-2, 4, 4);
  });
  // player
  mctx.fillStyle = '#00f';
  mctx.beginPath(); mctx.arc(px, pz, 4, 0, Math.PI*2); mctx.fill();
}

// ---------- Initial setup ----------
updateHUD();
showMessage('Klicke "Maus sperren (Play)" um mit der Maus zu spielen. Drücke X zum Absteigen.');

// Place the controls object initial position
controls.getObject().position.set(0,0,0);
scene.add(controls.getObject());

</script>
</body>
</html>
