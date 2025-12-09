<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8" />
  <title>Chambre Timeline — Three.js</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <style>
    html,body { height:100%; margin:0; font-family:Inter,Arial,Helvetica,sans-serif; background:#f3f4f6; }
    #app { width:100%; height:100vh; overflow:hidden; position:relative; }
    canvas { display:block; }

    /* Panel info */
    .info-panel {
      position: absolute;
      right: 20px;
      top: 20px;
      width: 320px;
      max-width: calc(100% - 40px);
      background: rgba(255,255,255,0.95);
      border-radius: 8px;
      box-shadow: 0 6px 30px rgba(15,23,42,0.12);
      padding: 14px;
      display: none;
      gap:8px;
      z-index: 10;
    }
    .info-panel.show { display: flex; flex-direction:column; }
    .info-panel h3 { margin:0 0 6px 0; font-size:16px; color:#0f172a; }
    .info-panel p { margin:0 0 10px 0; color:#475569; font-size:13px; }
    .info-panel img { width:100%; height:150px; object-fit:cover; border-radius:6px; background:#eef2ff }
    .info-links { display:flex; gap:8px; justify-content:flex-end; }
    .btn { padding:6px 10px; border-radius:6px; background:#111827; color:white; font-size:13px; text-decoration:none; }

    /* Cursor hint */
    .hint {
      position:absolute; left:20px; bottom:20px;
      background: rgba(255,255,255,0.92);
      padding:8px 12px; border-radius:10px; font-size:13px; color:#374151;
      box-shadow: 0 6px 24px rgba(15,23,42,0.08);
    }

    /* tiny visual for selected milestone */
    .milestone-dot { width:10px; height:10px; border-radius:50%; display:inline-block; margin-right:6px; vertical-align:middle; }
  </style>
</head>
<body>
  <div id="app"></div>

  <div class="info-panel" id="infoPanel" aria-hidden="true">
    <h3 id="infoTitle">Titre</h3>
    <p id="infoDesc">Description courte du projet...</p>
    <img id="infoImg" alt="preview" />
    <div class="info-links">
      <a id="infoLink" class="btn" target="_blank" rel="noopener">Voir</a>
    </div>
  </div>

  <div class="hint">Pan/zoom : souris • Hover → infos • Clic → ouvrir panneau</div>

  <!-- Import map + module script -->
  <script type="importmap">
  {
    "imports": {
      "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
      "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
    }
  }
  </script>

  <script type="module">
  import * as THREE from 'three';
  import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
  // small helper - no external tween lib; we'll lerp in the tick
  const app = document.getElementById('app');
  const infoPanel = document.getElementById('infoPanel');
  const infoTitle = document.getElementById('infoTitle');
  const infoDesc = document.getElementById('infoDesc');
  const infoImg = document.getElementById('infoImg');
  const infoLink = document.getElementById('infoLink');

  // renderer
  const renderer = new THREE.WebGLRenderer({ antialias:true, alpha:false });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.outputEncoding = THREE.sRGBEncoding;
  app.appendChild(renderer.domElement);

  // scene & camera
  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0xf7f7f8);

  const camera = new THREE.PerspectiveCamera(45, window.innerWidth/window.innerHeight, 0.1, 200);
  camera.position.set(4, 2.2, 6);

  // controls
  const controls = new OrbitControls(camera, renderer.domElement);
  controls.enableDamping = true;
  controls.dampingFactor = 0.09;
  controls.minDistance = 2.5;
  controls.maxDistance = 12;
  controls.maxPolarAngle = Math.PI/2.1;

  // lights
  const hemi = new THREE.HemisphereLight(0xffffff, 0x666666, 0.7);
  scene.add(hemi);

  const dir = new THREE.DirectionalLight(0xffffff, 0.8);
  dir.position.set(5, 10, 2);
  dir.castShadow = false;
  scene.add(dir);

  // room: simple floor + back wall to frame the scene
  const floorMat = new THREE.MeshStandardMaterial({ color: 0xf8f8f9, roughness:0.9, metalness:0.0 });
  const floorGeo = new THREE.PlaneGeometry(14, 10);
  const floor = new THREE.Mesh(floorGeo, floorMat);
  floor.rotation.x = -Math.PI/2;
  floor.position.y = 0;
  scene.add(floor);

  const backWallMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness:1 });
  const backWall = new THREE.Mesh(new THREE.PlaneGeometry(14,6), backWallMat);
  backWall.position.set(0,3, -4.5);
  scene.add(backWall);

  // furniture style: low-poly blocks with wood accents
  const woodColor = 0xE4B98A;
  const gray = 0xE6E9EE;
  const dark = 0x111827;

  function makeBox(w,h,d, color){
    const geo = new THREE.BoxGeometry(w,h,d);
    const mat = new THREE.MeshStandardMaterial({ color, roughness:0.7 });
    return new THREE.Mesh(geo, mat);
  }

  // desk
  const desk = new THREE.Group();
  const deskTop = makeBox(2.4, 0.08, 0.9, 0xffffff);
  deskTop.position.y = 0.86;
  desk.add(deskTop);
  const leg1 = makeBox(0.08,0.7,0.08, woodColor); leg1.position.set(-1.15,0.45,0.38);
  const leg2 = leg1.clone(); leg2.position.x = 1.15;
  desk.add(leg1, leg2);
  desk.position.set(-1.8, 0, -1.5);
  scene.add(desk);

  // shelf
  const shelf = new THREE.Group();
  const shelfBody = makeBox(1.6, 1.6, 0.28, gray);
  shelfBody.position.y = 1.0;
  shelf.add(shelfBody);
  shelf.position.set(1.8, 0, -1.8);
  scene.add(shelf);

  // bed
  const bed = new THREE.Group();
  const bedBase = makeBox(2.2, 0.36, 1.0, 0xffffff);
  bedBase.position.y = 0.18;
  bed.add(bedBase);
  bed.position.set(0.8,0,1.6);
  scene.add(bed);

  // decorative lamp/poster as milestone candidate
  const lamp = new THREE.Group();
  const lampBase = makeBox(0.12, 0.6, 0.12, 0x0f172a);
  lampBase.position.y = 0.3;
  const lampShade = new THREE.ConeGeometry(0.18,0.2,6);
  const lampMat = new THREE.MeshStandardMaterial({ color: 0xfff1d6, roughness:0.6 });
  const lampMesh = new THREE.Mesh(lampShade, lampMat);
  lampMesh.position.y = 0.64;
  lamp.add(lampBase, lampMesh);
  lamp.position.set(-0.7, 0, -0.9);
  scene.add(lamp);

  // timeline: a thin tube running through the room
  const timelineGroup = new THREE.Group();
  // define points for the curve (across room, through furniture)
  const points = [
    new THREE.Vector3(-2.6, 1.05, -2.2),
    new THREE.Vector3(-1.8, 1.0, -1.4),
    new THREE.Vector3(-0.2, 1.0, -0.8),
    new THREE.Vector3(0.9, 1.03, -0.2),
    new THREE.Vector3(1.7, 1.0, -1.5),
    new THREE.Vector3(0.9, 1.0, 1.2)
  ];
  // create smooth curve
  const curve = new THREE.CatmullRomCurve3(points);
  const tubeGeo = new THREE.TubeGeometry(curve, 128, 0.03, 8, false);
  const tubeMat = new THREE.MeshStandardMaterial({ color: 0x9CA3AF, emissive:0x9CA3AF, emissiveIntensity: 0.18, roughness:0.6 });
  const tube = new THREE.Mesh(tubeGeo, tubeMat);
  timelineGroup.add(tube);

  scene.add(timelineGroup);

  // milestones placed along curve (map to objects)
  // We'll create small cubes with an emissive material; each has metadata
  const milestoneMat = new THREE.MeshStandardMaterial({ color: 0x111827, emissive:0x00A3FF, emissiveIntensity:0.9, roughness:0.4 });
  const milestonePositions = [
    { t: 0.12, ref: desk, title: "Projet récent — App", desc:"Développement d'une app front-end moderne.", img:"", url:"#", color:0x00A3FF },
    { t: 0.36, ref: shelf, title: "Projets antérieurs", desc:"Plusieurs projets d'UX et d'UI design.", img:"", url:"#", color:0x8B5CF6 },
    { t: 0.55, ref: lamp, title: "Certifications", desc:"Certifs et compétences clés.", img:"", url:"#", color:0x10B981 },
    { t: 0.82, ref: bed, title: "Projets perso", desc:"Expérimentations et prototypes.", img:"", url:"#", color:0xF97316 }
  ];

  const milestones = [];
  milestonePositions.forEach((m, i) => {
    const pos = curve.getPointAt(m.t);
    const cubeGeom = new THREE.BoxGeometry(0.12,0.12,0.12);
    const mat = new THREE.MeshStandardMaterial({ color: 0x111827, emissive: m.color, emissiveIntensity: 0.95, roughness:0.5 });
    const mesh = new THREE.Mesh(cubeGeom, mat);
    mesh.position.copy(pos);
    mesh.userData = { idx:i, info:m, baseScale:1.0 };
    scene.add(mesh);
    milestones.push(mesh);

    // small light to emphasize
    const pLight = new THREE.PointLight(m.color, 0.25, 1.5);
    pLight.position.copy(pos);
    scene.add(pLight);
  });

  // raycaster for hover/click
  const raycaster = new THREE.Raycaster();
  const pointer = new THREE.Vector2();
  let hovered = null;
  let selected = null;

  function onPointerMove(e){
    const rect = renderer.domElement.getBoundingClientRect();
    pointer.x = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    pointer.y = -((e.clientY - rect.top) / rect.height) * 2 + 1;
  }

  window.addEventListener('pointermove', onPointerMove);

  window.addEventListener('click', (e) => {
    if (hovered) {
      openInfo(hovered.userData.info, hovered);
      // animate camera target toward object
      initiateCameraMove(hovered.position.clone().add(new THREE.Vector3(0.0,0.6,0.8)), hovered.position.clone());
      selected = hovered;
    } else {
      closeInfo();
      selected = null;
    }
  });

  function openInfo(info, mesh) {
    infoTitle.textContent = info.title;
    infoDesc.textContent = info.desc;
    // placeholder image gradient via dataURL if none provided
    if (info.img) {
      infoImg.src = info.img;
    } else {
      // small placeholder
      infoImg.src = generatePlaceholderImg(info.color || 0xD1D5DB);
    }
    infoLink.href = info.url || '#';
    infoPanel.classList.add('show');
    infoPanel.setAttribute('aria-hidden','false');
  }
  function closeInfo(){
    infoPanel.classList.remove('show');
    infoPanel.setAttribute('aria-hidden','true');
  }

  function generatePlaceholderImg(hex) {
    const c = document.createElement('canvas');
    c.width = 800; c.height = 400;
    const ctx = c.getContext('2d');
    ctx.fillStyle = '#fafafa';
    ctx.fillRect(0,0,c.width,c.height);
    ctx.fillStyle = '#e6edf6';
    ctx.fillRect(0,0,c.width,c.height/2);
    ctx.fillStyle = '#111827';
    ctx.font = '28px sans-serif';
    ctx.fillText('Preview', 18, 60);
    return c.toDataURL();
  }

  // Camera move helper
  let camTarget = new THREE.Vector3(0,1,0);
  let camGoalPos = null;
  let camGoalTarget = null;
  let camMoveTime = 0;
  function initiateCameraMove(goalPos, goalTarget) {
    camGoalPos = goalPos.clone();
    camGoalTarget = goalTarget.clone();
    camMoveTime = 0.0001;
  }

  // animate
  const clock = new THREE.Clock();
  function animate() {
    requestAnimationFrame(animate);
    const dt = clock.getDelta();
    controls.update();

    // hover detection
    raycaster.setFromCamera(pointer, camera);
    const intersects = raycaster.intersectObjects(milestones, false);
    if (intersects.length) {
      const obj = intersects[0].object;
      if (hovered !== obj) {
        if (hovered) {
          // reset previous
          hovered.scale.setScalar(hovered.userData.baseScale);
          hovered.material.emissiveIntensity = 0.95;
        }
        hovered = obj;
        hovered.scale.setScalar(1.14);
        hovered.material.emissiveIntensity = 1.6;
        renderer.domElement.style.cursor = 'pointer';
      }
    } else {
      if (hovered) {
        hovered.scale.setScalar(hovered.userData.baseScale);
        hovered.material.emissiveIntensity = 0.95;
      }
      hovered = null;
      renderer.domElement.style.cursor = 'auto';
    }

    // subtle timeline pulsation
    const t = clock.elapsedTime;
    tube.material.emissiveIntensity = 0.18 + Math.sin(t * 1.2) * 0.03;

    // milestone tiny bob or pulse
    milestones.forEach((m,i) => {
      m.position.y += Math.sin(t*1.2 + i*0.8) * 0.0005; // extremely subtle
      // small emissive intensity dance
      m.material.emissiveIntensity = 0.85 + Math.sin(t*2 + i*0.6)*0.12;
    });

    // camera lerp if animating
    if (camGoalPos && camMoveTime < 1.0) {
      camMoveTime += dt * 2.6; // speed factor
      const ease = easeOutCubic(Math.min(camMoveTime,1));
      camera.position.lerpVectors(camera.position, camGoalPos, ease);
      camTarget.lerpVectors(camTarget, camGoalTarget, ease);
      controls.target.copy(camTarget);
    }

    renderer.render(scene, camera);
  }

  function easeOutCubic(x){ return 1 - Math.pow(1-x,3); }

  animate();

  // handle resize
  window.addEventListener('resize', () => {
    const w = window.innerWidth, h = window.innerHeight;
    renderer.setSize(w,h);
    camera.aspect = w/h;
    camera.updateProjectionMatrix();
  });

  // basic instructions: expose scene elements for easy tweaking in console
  window.__APP__ = { scene, camera, controls, milestones, tube, desk, shelf, bed, lamp };

  </script>
</body>
</html>

