<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Bike GTA - Minimal 3D Prototype</title>
  <style>
    html,body{height:100%;margin:0;overflow:hidden;font-family:Arial,Helvetica,sans-serif}
    #overlay{
      position:fixed;left:10px;top:10px;color:#fff;z-index:5;background:rgba(0,0,0,0.4);padding:12px;border-radius:8px;
    }
    #hud{position:fixed;right:10px;top:10px;color:#fff;z-index:5;background:rgba(0,0,0,0.4);padding:12px;border-radius:8px;text-align:right}
    #instructions{width:300px}
    canvas{display:block}
    #message{position:fixed;left:50%;bottom:20px;transform:translateX(-50%);color:#fff;background:rgba(0,0,0,0.6);padding:8px 12px;border-radius:6px;z-index:5}
  </style>
</head>
<body>
<div id="overlay">
  <div id="instructions">
    <strong>Bike GTA — Prototype</strong>
    <p>W: Gas / S: Bremse / A/D: Lenken<br>E: Interagieren (Fahrrad stehlen / aufsteigen / absteigen)<br>Shift: Sprint / Leertaste: Handbremse<br>Mouse: Blick
    </p>
  </div>
</div>
<div id="hud">
  <div>Geld: <span id="money">0</span> $</div>
  <div>Gestohlene Fahrräder: <span id="stolen">0</span></div>
</div>
<div id="message">Klicke ins Fenster, um die Maus zu sperren.</div>

<!-- Three.js CDN -->
<script src="https://cdn.jsdelivr.net/npm/three@0.155.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.155.0/examples/js/controls/PointerLockControls.js"></script>
<script>
// Minimal open-world-like prototype using Three.js
// Keine externen Assets — einfache Formen

let scene, camera, renderer, controls;
let player = {height:1.8, money:0, stolen:0, mounted:null};
let move = {forward:false,back:false,left:false,right:false,sprint:false,brake:false};
let bikes = [];
let raycaster = new THREE.Raycaster();
let mouse = new THREE.Vector2();
let messageTimer = 0;

init();
animate();

function init(){
  scene = new THREE.Scene();
  scene.background = new THREE.Color(0x87ceeb);

  camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
  camera.position.set(0, player.height, 5);

  renderer = new THREE.WebGLRenderer({antialias:true});
  renderer.setSize(window.innerWidth, window.innerHeight);
  document.body.appendChild(renderer.domElement);

  // Light
  const hemi = new THREE.HemisphereLight(0xffffff, 0x444444, 0.8);
  hemi.position.set(0,50,0);
  scene.add(hemi);
  const dir = new THREE.DirectionalLight(0xffffff,0.6);
  dir.position.set(-10,20,10);
  scene.add(dir);

  // Ground (small world)
  const ground = new THREE.Mesh(
    new THREE.PlaneGeometry(200,200,10,10),
    new THREE.MeshStandardMaterial({color:0x556b2f})
  );
  ground.rotation.x = -Math.PI/2;
  scene.add(ground);

  // Simple city blocks / obstacles
  const boxGeo = new THREE.BoxGeometry(4,2,4);
  for(let i=0;i<40;i++){
    const mat = new THREE.MeshStandardMaterial({color:0x8b8b8b});
    const m = new THREE.Mesh(boxGeo, mat);
    m.position.set((Math.random()-0.5)*90,1,(Math.random()-0.5)*90);
    m.rotation.y = Math.random()*Math.PI;
    scene.add(m);
  }

  // Spawn bikes
  for(let i=0;i<25;i++) spawnBike(new THREE.Vector3((Math.random()-0.5)*80,0.1,(Math.random()-0.5)*80));

  // Controls
  controls = new THREE.PointerLockControls(camera, document.body);
  const message = document.getElementById('message');
  document.addEventListener('click', ()=>{ controls.lock(); message.style.display='none'; });
  controls.addEventListener('lock', ()=>{ /* locked */ });
  controls.addEventListener('unlock', ()=>{ message.style.display='block'; });

  // Input
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);
  window.addEventListener('resize', onWindowResize);
  window.addEventListener('keypress', onKeyPress);
}

function spawnBike(pos){
  const g = new THREE.Group();
  // frame
  const frame = new THREE.Mesh(new THREE.BoxGeometry(1.2,0.1,0.1), new THREE.MeshStandardMaterial({color:0x222222}));
  frame.position.set(0,0.5,0);
  frame.rotation.z = 0.2;
  g.add(frame);
  // wheels
  const wheelGeo = new THREE.TorusGeometry(0.4,0.08,8,16);
  const w1 = new THREE.Mesh(wheelGeo, new THREE.MeshStandardMaterial({color:0x000000}));
  w1.rotation.x = Math.PI/2; w1.position.set(0.6,0.2,0);
  const w2 = w1.clone(); w2.position.set(-0.6,0.2,0);
  g.add(w1); g.add(w2);
  // seat
  const seat = new THREE.Mesh(new THREE.BoxGeometry(0.3,0.05,0.2), new THREE.MeshStandardMaterial({color:0x331a00}));
  seat.position.set(0.0,0.7,0);
  g.add(seat);
  // simple handlebars
  const bar = new THREE.Mesh(new THREE.BoxGeometry(0.6,0.05,0.05), new THREE.MeshStandardMaterial({color:0x333333}));
  bar.position.set(0.9,0.8,0);
  g.add(bar);

  g.position.copy(pos);
  g.userData = {stolen:false, mounted:false};
  scene.add(g);
  bikes.push(g);
}

function onKeyDown(e){
  switch(e.code){
    case 'KeyW': move.forward=true; break;
    case 'KeyS': move.back=true; break;
    case 'KeyA': move.left=true; break;
    case 'KeyD': move.right=true; break;
    case 'ShiftLeft': move.sprint=true; break;
    case 'Space': move.brake=true; break;
    case 'KeyE': tryInteract(); break;
  }
}
function onKeyUp(e){
  switch(e.code){
    case 'KeyW': move.forward=false; break;
    case 'KeyS': move.back=false; break;
    case 'KeyA': move.left=false; break;
    case 'KeyD': move.right=false; break;
    case 'ShiftLeft': move.sprint=false; break;
    case 'Space': move.brake=false; break;
  }
}
function onKeyPress(e){
  // placeholder
}

let velocity = new THREE.Vector3();
let direction = new THREE.Vector3();

function tryInteract(){
  // Raycast forward from camera to detect nearest bike within 2.5m
  const origin = controls.getObject().position.clone();
  const forward = new THREE.Vector3();
  camera.getWorldDirection(forward);
  raycaster.set(origin, forward);
  const intersects = raycaster.intersectObjects(bikes, true);
  if(intersects.length>0){
    // find group parent
    let g = intersects[0].object;
    while(g && !bikes.includes(g)) g = g.parent;
    if(!g) return;

    const dist = intersects[0].distance;
    if(dist>3){ showMessage('Zu weit weg'); return; }

    if(player.mounted === g){
      // already mounted -> dismount
      player.mounted = null; showMessage('Vom Fahrrad abgestiegen');
    } else if(!g.userData.mounted){
      if(!g.userData.stolen){
        // Steal
        g.userData.stolen = true;
        player.money += 50; player.stolen += 1;
        updateHUD();
        showMessage('Fahrrad gestohlen! +50 $');
      } else {
        showMessage('Dieses Fahrrad wurde bereits genommen');
      }
      // mount after stealing
      player.mounted = g;
      g.userData.mounted = true;
      showMessage('Aufgestiegen');
    }
  } else {
    showMessage('Kein Fahrrad in Sicht');
  }
}

function updateHUD(){
  document.getElementById('money').textContent = player.money;
  document.getElementById('stolen').textContent = player.stolen;
}

function showMessage(t){
  const m = document.getElementById('message');
  m.textContent = t; m.style.display='block'; messageTimer = 180; // show for 3s (60fps)
}

function onWindowResize(){
  camera.aspect = window.innerWidth/window.innerHeight; camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

function animate(){
  requestAnimationFrame(animate);

  // Movement
  const delta = 0.016; // fixed step for simplicity
  const speedBase = player.mounted? 6 : 3; // faster when on bike
  const sprintMult = move.sprint?1.6:1;
  const accel = 10 * delta * (move.forward?1: (move.back?-0.6:0));

  // rotate if mounted
  if(player.mounted){
    // simple steering
    if(move.left) player.mounted.rotation.y += 0.03 * (move.sprint?1.5:1);
    if(move.right) player.mounted.rotation.y -= 0.03 * (move.sprint?1.5:1);
    // forward vector
    const fwd = new THREE.Vector3(Math.sin(player.mounted.rotation.y),0,Math.cos(player.mounted.rotation.y));
    velocity.addScaledVector(fwd, accel);
    // braking
    if(move.brake) velocity.multiplyScalar(0.9);
    // cap speed
    const maxSpeed = speedBase * sprintMult;
    if(velocity.length()>maxSpeed) velocity.setLength(maxSpeed);
    // move bike
    player.mounted.position.addScaledVector(velocity, delta);
    // camera follows behind bike
    const camTarget = player.mounted.position.clone().add(new THREE.Vector3(0,1.2,2).applyAxisAngle(new THREE.Vector3(0,1,0), player.mounted.rotation.y));
    camera.position.lerp(camTarget, 0.2);
    camera.lookAt(player.mounted.position.clone().add(new THREE.Vector3(0,0.9,0)));
  } else {
    // on foot: use PointerLockControls getObject
    const controlObj = controls.getObject();
    direction.z = Number(move.forward) - Number(move.back);
    direction.x = Number(move.right) - Number(move.left);
    direction.normalize();
    // apply movement in camera direction
    const camDir = new THREE.Vector3(); camera.getWorldDirection(camDir); camDir.y=0; camDir.normalize();
    const camRight = new THREE.Vector3().crossVectors(new THREE.Vector3(0,1,0), camDir).normalize();
    const moveVec = new THREE.Vector3();
    moveVec.addScaledVector(camDir, direction.z);
    moveVec.addScaledVector(camRight, direction.x);
    const walkSpeed = 4 * (move.sprint?1.6:1);
    controlObj.position.addScaledVector(moveVec, walkSpeed*delta);
    camera.position.copy(controlObj.position).add(new THREE.Vector3(0, player.height, 0));
  }

  // Rotate bike wheels for visual feedback
  bikes.forEach(b=>{
    b.children.forEach(c=>{
      if(c.geometry && c.geometry.type === 'TorusGeometry') c.rotation.z += 0.2;
    });
  });

  // simple collision: keep inside world bounds
  clampToBounds();

  // update message timer
  if(messageTimer>0){ messageTimer--; if(messageTimer===0) document.getElementById('message').style.display='none'; }

  renderer.render(scene, camera);
}

function clampToBounds(){
  const bound = 95;
  if(player.mounted){
    player.mounted.position.x = THREE.MathUtils.clamp(player.mounted.position.x, -bound, bound);
    player.mounted.position.z = THREE.MathUtils.clamp(player.mounted.position.z, -bound, bound);
  } else {
    const obj = controls.getObject();
    obj.position.x = THREE.MathUtils.clamp(obj.position.x, -bound, bound);
    obj.position.z = THREE.MathUtils.clamp(obj.position.z, -bound, bound);
  }
}

</script>
</body>
</html>

