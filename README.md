const SAVE_KEY = 'ghostline_ops_save_v1';
const WORLD_HALF = 45;
const MATCH_LABEL = 'Ghostline Ops';

const QUALITY_MAP = {
  '720p': { pixelRatio: 0.9, cameraFov: 78, label: '720p' },
  '1080p': { pixelRatio: 1.15, cameraFov: 72, label: '1080p' },
  '1440p': { pixelRatio: 1.45, cameraFov: 68, label: '1440p' },
};

const MISSIONS = [
  {
    id: 1,
    name: 'Dustline Breach',
    type: 'kill',
    target: 6,
    description: 'Neutralize six hostile operators in the abandoned train yard.',
    instruction: 'Clear the trash zone. Keep moving and do not get pinned.',
  },
  {
    id: 2,
    name: 'Signal Scramble',
    type: 'kill',
    target: 9,
    description: 'Take down the jamming squad guarding the relay tower.',
    instruction: 'Break the blockade and secure the radio tower approach.',
  },
  {
    id: 3,
    name: 'Final Push',
    type: 'kill',
    target: 12,
    description: 'Sweep the convoy route and eliminate the remaining assault force.',
    instruction: 'This is the final operational push. Finish the corridor.',
  },
];

const WEAPONS = [
  {
    id: 'rifle',
    name: 'Rifle',
    damage: 17,
    fireRate: 0.15,
    magSize: 30,
    reserve: 120,
    spread: 0.03,
    color: 0x86d1ff,
  },
  {
    id: 'smg',
    name: 'SMG',
    damage: 12,
    fireRate: 0.09,
    magSize: 35,
    reserve: 120,
    spread: 0.06,
    color: 0xa9ffb0,
  },
  {
    id: 'burst',
    name: 'Burst',
    damage: 25,
    fireRate: 0.28,
    magSize: 18,
    reserve: 72,
    spread: 0.022,
    color: 0xffda8a,
  },
];

const ui = {
  menu: document.getElementById('menu'),
  hud: document.getElementById('hud'),
  saveStatus: document.getElementById('saveStatus'),
  missionName: document.getElementById('missionName'),
  healthValue: document.getElementById('healthValue'),
  ammoValue: document.getElementById('ammoValue'),
  objectiveText: document.getElementById('objectiveText'),
  weaponName: document.getElementById('weaponName'),
  statusText: document.getElementById('statusText'),
  message: document.getElementById('message'),
  startBtn: document.getElementById('startBtn'),
  continueBtn: document.getElementById('continueBtn'),
  saveBtn: document.getElementById('saveBtn'),
  qualitySelect: document.getElementById('qualitySelect'),
};

const state = {
  started: false,
  playing: false,
  missionIndex: 0,
  missionKills: 0,
  totalKills: 0,
  selectedWeapon: 0,
  statusText: 'Awaiting orders',
  pointerLocked: false,
  quality: '720p',
  currentMessageTimer: 0,
  lastShotAt: 0,
  currentAmmo: WEAPONS[0].magSize,
  reserveAmmo: WEAPONS[0].reserve,
  reloadTimer: 0,
  canShoot: true,
  score: 0,
  flashlight: null,
};

const player = {
  position: new THREE.Vector3(0, 1.7, 20),
  velocity: new THREE.Vector3(0, 0, 0),
  yaw: 0,
  pitch: 0,
  health: 100,
  speed: 9.8,
};

const input = {
  forward: false,
  backward: false,
  left: false,
  right: false,
  sprint: false,
};

const world = {
  scene: null,
  camera: null,
  renderer: null,
  buildings: [],
  enemies: [],
  bullets: [],
  projectiles: [],
  missionMarker: null,
  gunMesh: null,
  muzzleFlash: null,
};

let lastTimestamp = 0;

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function randRange(min, max) {
  return min + Math.random() * (max - min);
}

function getCurrentMission() {
  return MISSIONS[state.missionIndex] || MISSIONS[MISSIONS.length - 1];
}

function getCurrentWeapon() {
  return WEAPONS[state.selectedWeapon];
}

function showMessage(text, duration = 1500) {
  ui.message.textContent = text;
  ui.message.classList.remove('hidden');
  state.currentMessageTimer = duration;
}

function hideMessage() {
  ui.message.classList.add('hidden');
}

function setStatus(text) {
  state.statusText = text;
  ui.statusText.textContent = text;
}

function updateHUD() {
  const mission = getCurrentMission();
  ui.missionName.textContent = `${mission.id}. ${mission.name}`;
  ui.healthValue.textContent = Math.max(0, Math.ceil(player.health));
  ui.ammoValue.textContent = `${state.currentAmmo} / ${state.reserveAmmo}`;
  ui.objectiveText.textContent = `${mission.description} (${state.missionKills}/${mission.target})`;
  ui.weaponName.textContent = getCurrentWeapon().name;
  ui.statusText.textContent = state.statusText;
}

function applyQualityPreset(name) {
  state.quality = name;
  const config = QUALITY_MAP[name];
  if (world.camera) {
    world.camera.fov = config.cameraFov;
    world.camera.updateProjectionMatrix();
  }
  if (world.renderer) {
    world.renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, config.pixelRatio));
    world.renderer.setSize(window.innerWidth, window.innerHeight, false);
  }
  showMessage(`Quality set to ${config.label}`, 1000);
}

function buildEnvironment() {
  if (!world.scene) return;

  const ambient = new THREE.AmbientLight(0x9bc1ff, 0.8);
  world.scene.add(ambient);

  const directional = new THREE.DirectionalLight(0xb9d5ff, 0.5);
  directional.position.set(30, 40, 20);
  world.scene.add(directional);

  const ground = new THREE.Mesh(
    new THREE.BoxGeometry(120, 1, 120),
    new THREE.MeshStandardMaterial({ color: 0x2a3a3a, roughness: 0.95 })
  );
  ground.position.set(0, -0.6, 0);
  world.scene.add(ground);

  const runway = new THREE.Mesh(
    new THREE.BoxGeometry(14, 1, 40),
    new THREE.MeshStandardMaterial({ color: 0x555d60, roughness: 0.9 })
  );
  runway.position.set(0, -0.2, -10);
  world.scene.add(runway);

  const pathBorder = new THREE.Mesh(
    new THREE.BoxGeometry(2, 1, 60),
    new THREE.MeshStandardMaterial({ color: 0x4d5d67, roughness: 0.8 })
  );
  pathBorder.position.set(-30, 0.2, 0);
  world.scene.add(pathBorder);

  const pathBorder2 = pathBorder.clone();
  pathBorder2.position.x = 30;
  world.scene.add(pathBorder2);

  const buildingMaterial = new THREE.MeshStandardMaterial({
    color: 0x6b8390,
    roughness: 0.85,
    metalness: 0.1,
  });

  const buildingPositions = [
    [-18, 0, -12],
    [18, 0, -22],
    [-28, 0, 18],
    [26, 0, 18],
    [-5, 0, 24],
    [10, 0, 12],
    [-20, 0, 34],
    [20, 0, 34],
  ];

  for (const pos of buildingPositions) {
    const b = new THREE.Mesh(new THREE.BoxGeometry(8, 8, 8), buildingMaterial);
    b.position.set(pos[0], 4, pos[2]);
    world.scene.add(b);
    world.buildings.push(b);
  }

  const objectiveZone = new THREE.Mesh(
    new THREE.CylinderGeometry(2.5, 2.5, 0.4, 16),
    new THREE.MeshStandardMaterial({ color: 0x77d8ff, emissive: 0x0c6087, emissiveIntensity: 0.55 })
  );
  objectiveZone.position.set(0, 0.3, -35);
  world.missionMarker = objectiveZone;
  world.scene.add(objectiveZone);

  const crateMaterial = new THREE.MeshStandardMaterial({ color: 0x876f5a, roughness: 1 });
  for (let i = 0; i < 14; i++) {
    const crate = new THREE.Mesh(new THREE.BoxGeometry(2.2, 2.2, 2.2), crateMaterial);
    crate.position.set(randRange(-22, 22), 1.1, randRange(-30, 18));
    world.scene.add(crate);
    world.buildings.push(crate);
  }
}

function createGun() {
  const gunGroup = new THREE.Group();

  const body = new THREE.Mesh(
    new THREE.BoxGeometry(0.22, 0.2, 1.3),
    new THREE.MeshStandardMaterial({ color: 0x1a1c1f, metalness: 0.7, roughness: 0.35 })
  );
  body.position.set(0.34, -0.38, -0.8);
  gunGroup.add(body);

  const barrel = new THREE.Mesh(
    new THREE.CylinderGeometry(0.05, 0.05, 0.85, 12),
    new THREE.MeshStandardMaterial({ color: 0x2d2e30, metalness: 0.9, roughness: 0.3 })
  );
  barrel.rotation.x = Math.PI / 2;
  barrel.position.set(0.34, -0.32, -1.4);
  gunGroup.add(barrel);

  const handle = new THREE.Mesh(
    new THREE.BoxGeometry(0.15, 0.35, 0.22),
    new THREE.MeshStandardMaterial({ color: 0x0f1011, metalness: 0.63, roughness: 0.42 })
  );
  handle.position.set(0.34, -0.58, -0.15);
  gunGroup.add(handle);

  const muzzle = new THREE.Mesh(
    new THREE.SphereGeometry(0.08, 8, 8),
    new THREE.MeshBasicMaterial({ color: 0xfff0b5 })
  );
  muzzle.position.set(0.34, -0.34, -1.9);
  muzzle.visible = false;
  world.muzzleFlash = muzzle;
  gunGroup.add(muzzle);

  world.gunMesh = gunGroup;
  world.camera.add(gunGroup);
  world.scene.add(world.camera);
}

function createEnemy(position) {
  const group = new THREE.Group();
  const body = new THREE.Mesh(
    new THREE.BoxGeometry(1.3, 2.2, 1.1),
    new THREE.MeshStandardMaterial({ color: 0xae3d3d, roughness: 0.9 })
  );
  group.add(body);

  const head = new THREE.Mesh(
    new THREE.BoxGeometry(0.8, 0.8, 0.8),
    new THREE.MeshStandardMaterial({ color: 0xecd2ba, roughness: 0.8 })
  );
  head.position.y = 1.6;
  group.add(head);

  const gun = new THREE.Mesh(
    new THREE.BoxGeometry(0.18, 0.18, 0.8),
    new THREE.MeshStandardMaterial({ color: 0x252525, metalness: 0.8, roughness: 0.45 })
  );
  gun.position.set(0.9, 0.7, 0.2);
  gun.rotation.z = -Math.PI / 2;
  group.add(gun);

  group.position.copy(position);
  group.userData = {
    health: 100,
    alive: true,
    speed: randRange(1.5, 2.4),
    attackCooldown: randRange(1.1, 1.9),
    bob: randRange(0, Math.PI * 2),
  };

  world.scene.add(group);
  world.enemies.push(group);
}

function spawnMissionEnemies() {
  world.enemies.forEach((enemy) => world.scene.remove(enemy));
  world.enemies.length = 0;

  const mission = getCurrentMission();
  const count = mission.target + 2;
  for (let i = 0; i < count; i++) {
    const angle = (i / count) * Math.PI * 2;
    const radius = randRange(17, 38);
    const x = Math.cos(angle) * radius + randRange(-8, 8);
    const z = Math.sin(angle) * radius + randRange(-8, 8);
    createEnemy(new THREE.Vector3(x, 1.1, z));
  }

  state.missionKills = 0;
  updateHUD();
}

function resetPlayerToMissionStart() {
  player.position.set(0, 1.7, 20);
  player.yaw = 0;
  player.pitch = 0;
  player.health = 100;
  state.currentAmmo = getCurrentWeapon().magSize;
  state.reserveAmmo = getCurrentWeapon().reserve;
  state.reloadTimer = 0;
  state.canShoot = true;
  world.camera.position.copy(player.position);
  world.camera.rotation.set(0, 0, 0);
}

function startGame() {
  world.scene = new THREE.Scene();
  world.camera = new THREE.PerspectiveCamera(72, window.innerWidth / window.innerHeight, 0.1, 500);
  world.camera.position.set(player.position.x, player.position.y, player.position.z);
  world.camera.rotation.order = 'YXZ';

  world.renderer = new THREE.WebGLRenderer({ canvas: document.getElementById('gameCanvas'), antialias: false, alpha: false, powerPreference: 'low-power' });
  world.renderer.setClearColor(0x091722, 1);
  world.renderer.setSize(window.innerWidth, window.innerHeight, false);
  world.renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, QUALITY_MAP[state.quality].pixelRatio));

  buildEnvironment();
  createGun();
  resetPlayerToMissionStart();
  spawnMissionEnemies();

  state.started = true;
  state.playing = true;
  ui.menu.classList.add('hidden');
  ui.hud.classList.remove('hidden');
  document.body.requestPointerLock();
  setStatus('Mission active');
  updateHUD();
}

function saveCurrentProgress() {
  const save = {
    missionIndex: state.missionIndex,
    totalKills: state.totalKills,
    selectedWeapon: state.selectedWeapon,
    quality: state.quality,
    playerHealth: player.health,
    currentAmmo: state.currentAmmo,
    reserveAmmo: state.reserveAmmo,
    weaponIndex: state.selectedWeapon,
    score: state.score,
  };

  localStorage.setItem(SAVE_KEY, JSON.stringify(save));
  ui.saveStatus.textContent = `Save created: Mission ${state.missionIndex + 1} / ${MISSIONS.length}`;
  showMessage('Progress saved', 1200);
}

function loadSavedProgress() {
  try {
    const raw = localStorage.getItem(SAVE_KEY);
    if (!raw) {
      ui.saveStatus.textContent = 'No save detected.';
      showMessage('No save found', 1200);
      return;
    }

    const save = JSON.parse(raw);
    state.missionIndex = clamp(Number(save.missionIndex) || 0, 0, MISSIONS.length - 1);
    state.totalKills = Number(save.totalKills) || 0;
    state.selectedWeapon = clamp(Number(save.selectedWeapon) || 0, 0, WEAPONS.length - 1);
    player.health = Number(save.playerHealth) || 100;
    state.currentAmmo = Number(save.currentAmmo) || WEAPONS[state.selectedWeapon].magSize;
    state.reserveAmmo = Number(save.reserveAmmo) || WEAPONS[state.selectedWeapon].reserve;
    state.score = Number(save.score) || 0;
    state.quality = save.quality || state.quality;
    applyQualityPreset(state.quality);
    ui.qualitySelect.value = state.quality;

    if (!state.started) {
      startGame();
    }

    ui.saveStatus.textContent = `Loaded save: Mission ${state.missionIndex + 1}`;
    showMessage('Save loaded', 1200);
    updateHUD();
  } catch (error) {
    console.error('Failed to load save', error);
    ui.saveStatus.textContent = 'Save data corrupted.';
  }
}

function completeMission() {
  const mission = getCurrentMission();
  state.score += 250 + state.missionKills * 20;

  if (state.missionIndex < MISSIONS.length - 1) {
    state.missionIndex += 1;
    setStatus('Mission complete');
    showMessage(`${mission.name} complete. Moving to ${MISSIONS[state.missionIndex].name}.`, 2200);
    spawnMissionEnemies();
    player.health = Math.min(100, player.health + 20);
    state.selectedWeapon = (state.selectedWeapon + 1) % WEAPONS.length;
    state.currentAmmo = WEAPONS[state.selectedWeapon].magSize;
    state.reserveAmmo = WEAPONS[state.selectedWeapon].reserve;
  } else {
    setStatus('Campaign complete');
    showMessage('Campaign complete. Operation successful.', 3000);
    state.missionIndex = MISSIONS.length - 1;
  }

  updateHUD();
  saveCurrentProgress();
}

function handleDamage(amount) {
  player.health -= amount;
  if (player.health <= 0) {
    player.health = 100;
    setStatus('You were downed');
    showMessage('Respawned at the rally point', 1800);
    player.position.set(0, 1.7, 20);
    spawnMissionEnemies();
  }
  updateHUD();
}

function updateEnemyAI(delta) {
  const mission = getCurrentMission();
  const maxEnemies = mission.target + 2;

  for (let i = world.enemies.length - 1; i >= 0; i--) {
    const enemy = world.enemies[i];
    const data = enemy.userData;

    if (!enemy || !data.alive) continue;

    const dir = new THREE.Vector3().subVectors(player.position, enemy.position);
    const distance = dir.length();
    dir.normalize();

    if (distance > 0.1) {
      const movement = distance < 11 ? 0.8 : 1.1;
      const step = delta * data.speed * movement;
      enemy.position.x += dir.x * step;
      enemy.position.z += dir.z * step;
      enemy.lookAt(player.position.x, enemy.position.y, player.position.z);
    }

    data.attackCooldown -= delta;
    if (distance < 12 && data.attackCooldown <= 0) {
      handleDamage(8 + Math.random() * 10);
      data.attackCooldown = randRange(1.0, 1.8);
    }

    const bob = Math.sin((performance.now() * 0.006) + data.bob) * 0.1;
    enemy.position.y = 1.1 + bob;
  }

  if (state.missionKills >= mission.target) {
    completeMission();
  }

  if (world.enemies.length < maxEnemies * 0.7) {
    const spawnSpot = new THREE.Vector3(randRange(-25, 25), 1.1, randRange(-35, 20));
    createEnemy(spawnSpot);
  }
}

function fireWeapon() {
  const weapon = getCurrentWeapon();
  const now = performance.now();
  if (state.reloadTimer > 0 || now - state.lastShotAt < weapon.fireRate * 1000) {
    return;
  }

  if (state.currentAmmo <= 0) {
    setStatus('Reload required');
    return;
  }

  state.currentAmmo -= 1;
  state.lastShotAt = now;
  world.muzzleFlash.visible = true;
  world.muzzleFlash.material.color.setHex(0xffd78a);

  const rayOrigin = world.camera.position.clone();
  const forward = new THREE.Vector3(0, 0, -1).applyQuaternion(world.camera.quaternion);
  let bestTarget = null;
  let bestScore = -Infinity;

  for (const enemy of world.enemies) {
    const toEnemy = new THREE.Vector3().subVectors(enemy.position, rayOrigin);
    const dist = toEnemy.length();
    if (dist > 46) continue;
    const dirToEnemy = toEnemy.normalize();
    const dot = forward.dot(dirToEnemy);
    if (dot < 0.84) continue;

    const score = dot * (1 / dist);
    if (score > bestScore) {
      bestScore = score;
      bestTarget = enemy;
    }
  }

  if (bestTarget) {
    const targetData = bestTarget.userData;
    targetData.health -= weapon.damage;
    if (targetData.health <= 0) {
      targetData.alive = false;
      world.scene.remove(bestTarget);
      const idx = world.enemies.indexOf(bestTarget);
      if (idx >= 0) world.enemies.splice(idx, 1);
      state.missionKills += 1;
      state.totalKills += 1;
      state.score += 50;
      setStatus('Target neutralized');
      showMessage('Hostile neutralized', 700);
    } else {
      setStatus('Target hit');
    }
  }

  setTimeout(() => {
    if (world.muzzleFlash) world.muzzleFlash.visible = false;
  }, 80);

  updateHUD();
}

function reloadWeapon() {
  const weapon = getCurrentWeapon();
  if (state.currentAmmo >= weapon.magSize || state.reserveAmmo <= 0 || state.reloadTimer > 0) {
    return;
  }

  state.reloadTimer = 1.25;
  setStatus('Reloading');
  showMessage('Reloading weapon', 700);
}

function finishReload() {
  const weapon = getCurrentWeapon();
  const needed = weapon.magSize - state.currentAmmo;
  const loaded = Math.min(needed, state.reserveAmmo);
  state.currentAmmo += loaded;
  state.reserveAmmo -= loaded;
  state.reloadTimer = 0;
  setStatus('Weapon ready');
  updateHUD();
}

function updatePlayer(delta) {
  const moveSpeed = input.sprint ? player.speed * 1.4 : player.speed;
  const moveVec = new THREE.Vector3();

  if (input.forward) moveVec.z -= 1;
  if (input.backward) moveVec.z += 1;
  if (input.left) moveVec.x -= 1;
  if (input.right) moveVec.x += 1;

  if (moveVec.lengthSq() > 0) {
    moveVec.normalize();
    const yaw = player.yaw;
    const dx = moveVec.x * Math.cos(yaw) - moveVec.z * Math.sin(yaw);
    const dz = moveVec.x * Math.sin(yaw) + moveVec.z * Math.cos(yaw);
    player.position.x += dx * moveSpeed * delta;
    player.position.z += dz * moveSpeed * delta;
  }

  player.position.x = clamp(player.position.x, -WORLD_HALF, WORLD_HALF);
  player.position.z = clamp(player.position.z, -WORLD_HALF, WORLD_HALF);

  world.camera.position.set(player.position.x, player.position.y, player.position.z);
  world.camera.rotation.y = player.yaw;
  world.camera.rotation.x = player.pitch;

  if (world.gunMesh) {
    world.gunMesh.position.set(0, 0, 0);
  }
}

function animate(timestamp) {
  const delta = Math.min(0.033, (timestamp - lastTimestamp) / 1000 || 0.016);
  lastTimestamp = timestamp;

  if (state.started && state.playing) {
    if (state.reloadTimer > 0) {
      state.reloadTimer -= delta;
      if (state.reloadTimer <= 0) finishReload();
    }

    updatePlayer(delta);
    updateEnemyAI(delta);

    if (world.missionMarker) {
      world.missionMarker.rotation.y += delta * 0.8;
    }

    if (state.currentMessageTimer > 0) {
      state.currentMessageTimer -= delta * 1000;
      if (state.currentMessageTimer <= 0) hideMessage();
    }
  }

  if (world.renderer && world.scene && world.camera) {
    world.renderer.render(world.scene, world.camera);
  }

  requestAnimationFrame(animate);
}

function onMouseMove(event) {
  if (!state.pointerLocked || !state.started || !state.playing) return;
  player.yaw -= event.movementX * 0.0022;
  player.pitch -= event.movementY * 0.0017;
  player.pitch = clamp(player.pitch, -1.35, 1.35);
}

function handlePointerLockState() {
  state.pointerLocked = document.pointerLockElement === document.body;
  if (!state.pointerLocked && state.started && state.playing) {
    setStatus('Paused');
    ui.menu.classList.remove('hidden');
    ui.hud.classList.add('hidden');
    state.playing = false;
  }
}

function handleKeyDown(event) {
  if (event.key === 'w' || event.key === 'W') input.forward = true;
  if (event.key === 's' || event.key === 'S') input.backward = true;
  if (event.key === 'a' || event.key === 'A') input.left = true;
  if (event.key === 'd' || event.key === 'D') input.right = true;
  if (event.key === 'Shift') input.sprint = true;

  if (event.key === 'r' || event.key === 'R') reloadWeapon();
  if (event.key === '1') {
    state.selectedWeapon = 0;
    setStatus('Switched to Rifle');
    state.currentAmmo = WEAPONS[0].magSize;
    state.reserveAmmo = WEAPONS[0].reserve;
    updateHUD();
  }
  if (event.key === '2') {
    state.selectedWeapon = 1;
    setStatus('Switched to SMG');
    state.currentAmmo = WEAPONS[1].magSize;
    state.reserveAmmo = WEAPONS[1].reserve;
    updateHUD();
  }
  if (event.key === '3') {
    state.selectedWeapon = 2;
    setStatus('Switched to Burst');
    state.currentAmmo = WEAPONS[2].magSize;
    state.reserveAmmo = WEAPONS[2].reserve;
    updateHUD();
  }

  if (event.key === 'F5') saveCurrentProgress();
  if (event.key === 'F9') loadSavedProgress();

  if (event.key === 'Escape') {
    if (state.started && state.playing) {
      document.exitPointerLock();
      state.playing = false;
      ui.menu.classList.remove('hidden');
      ui.hud.classList.add('hidden');
      setStatus('Paused');
    }
  }
}

function handleKeyUp(event) {
  if (event.key === 'w' || event.key === 'W') input.forward = false;
  if (event.key === 's' || event.key === 'S') input.backward = false;
  if (event.key === 'a' || event.key === 'A') input.left = false;
  if (event.key === 'd' || event.key === 'D') input.right = false;
  if (event.key === 'Shift') input.sprint = false;
}

function bindUI() {
  ui.startBtn.addEventListener('click', () => {
    state.missionIndex = 0;
    state.totalKills = 0;
    state.missionKills = 0;
    state.selectedWeapon = 0;
    player.health = 100;
    startGame();
  });

  ui.continueBtn.addEventListener('click', () => {
    loadSavedProgress();
  });

  ui.saveBtn.addEventListener('click', () => {
    saveCurrentProgress();
  });

  ui.qualitySelect.addEventListener('change', (event) => {
    applyQualityPreset(event.target.value);
  });

  window.addEventListener('resize', () => {
    if (world.camera && world.renderer) {
      world.camera.aspect = window.innerWidth / window.innerHeight;
      world.camera.updateProjectionMatrix();
      world.renderer.setSize(window.innerWidth, window.innerHeight, false);
    }
  });

  document.addEventListener('pointerlockchange', handlePointerLockState);
  document.addEventListener('mousemove', onMouseMove);
  document.addEventListener('keydown', handleKeyDown);
  document.addEventListener('keyup', handleKeyUp);
  document.addEventListener('mousedown', () => {
    if (!state.started || !state.playing) return;
    fireWeapon();
  });

  document.addEventListener('contextmenu', (event) => event.preventDefault());
}

function initialize() {
  bindUI();
  ui.qualitySelect.value = state.quality;
  applyQualityPreset('720p');
  ui.saveStatus.textContent = localStorage.getItem(SAVE_KEY) ? 'Save ready to load.' : 'No save detected.';
  updateHUD();
  requestAnimationFrame(animate);
}

initialize();
