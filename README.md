<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8" />
  <title>Bike GTA - Testszene</title>
  <style>
    body { margin: 0; overflow: hidden; }
    canvas { display: block; }
  </style>
</head>
<body>
<script src="https://cdn.jsdelivr.net/npm/three@0.155.0/build/three.min.js"></script>
<script>
let scene = new THREE.Scene();
scene.background = new THREE.Color(0x87ceeb); // Himmelblau

// Kamera
let camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
camera.position.set(0, 2, 5);

// Renderer
let renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Licht
let light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(5,10,5);
scene.add(light);

// Boden
let ground = new THREE.Mesh(
  new THREE.PlaneGeometry(50, 50),
  new THREE.MeshStandardMaterial({color: 0x228B22})
);
ground.rotation.x = -Math.PI/2;
scene.add(ground);

// Ein einfaches Fahrrad (stark vereinfacht)
let bike = new THREE.Group();
let frame = new THREE.Mesh(new THREE.BoxGeometry(1.2,0.1,0.1), new THREE.MeshStandardMaterial({color:0x222222}));
frame.position.y = 0.5;
bike.add(frame);

// Räder
let wheelGeo = new THREE.TorusGeometry(0.4,0.08,8,16);
let w1 = new THREE.Mesh(wheelGeo, new THREE.MeshStandardMaterial({color:0x000000}));
w1.rotation.x = Math.PI/2; w1.position.set(0.6,0.2,0);
let w2 = w1.clone(); w2.position.set(-0.6,0.2,0);
bike.add(w1); bike.add(w2);

bike.position.set(0,0,0);
scene.add(bike);

// Animation
function animate(){
  requestAnimationFrame(animate);
  bike.rotation.y += 0.01; // Dreht sich langsam, damit man es sieht
  renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
