<template>
  <div ref="diceContainer" @click="moveRightToLeft" />
</template>

<script setup>
import { onMounted, onUnmounted, ref } from 'vue'
import Matter from 'matter-js'

const { Engine, Render, Runner, World, Bodies, Body, Events } = Matter;
const diceContainer = ref(null)

const infoSprites = {
  center: {
    1: 0,
    2: 1,
    3: 2,
    4: 3,
    5: 5,
    6: 6,
    7: 7,
    8: 8,
    9: 10,
    10: 11,
    11: 12,
    12: 13,
    13: 15,
    14: 16,
    15: 17,
    16: 18,
    17: 20,
    18: 21,
    19: 22,
    20: 23
  },
  '19-1': {
    pre: [9, 3],
    from: 25,
    to: 28,
    next: [7, 13],
  },
  '1-13': {
    pre: [19, 7],
    from: 30,
    to: 33,
    next: [11, 5],
  },
  '1-7': {
    pre: [19, 13],
    from: 35,
    to: 38,
    next: [15, 17],
  },
  '2-12': {
    pre: [18, 20],
    from: 40,
    to: 43,
    next: [15, 10],
  },
  '18-2': {
    pre: [5, 4],
    from: 45,
    to: 48,
    next: [12, 20],
  },
  '2-20': {
    pre: [18, 12],
    from: 50,
    to: 53,
    next: [8, 14],
  },
  '16-3': {
    pre: [8, 6],
    from: 55,
    to: 58,
    next: [17, 19],
  },
  '3-17': {
    pre: [19, 16],
    from: 60,
    to: 63,
    next: [7, 10],
  },
  '3-19': {
    pre: [17, 16],
    from: 65,
    to: 68,
    next: [1, 9],
  },
  '11-4': {
    pre: [13, 9],
    from: 70,
    to: 73,
    next: [18, 14],
  },
  '4-14': {
    pre: [18, 11],
    from: 75,
    to: 78,
    next: [20, 6],
  },
  '4-18': {
    pre: [14, 11],
    from: 80,
    to: 83,
    next: [5, 2],
  },
  '5-13': {
    pre: [15, 18],
    from: 85,
    to: 88,
    next: [1, 11],
  },
  '5-15': {
    pre: [13, 18],
    from: 90,
    to: 93,
    next: [7, 12],
  },
  '5-18': {
    pre: [13, 15],
    from: 95,
    to: 98,
    next: [4, 2],
  },
  '6-9': {
    pre: [14, 16],
    from: 100,
    to: 103,
    next: [11, 19],
  },
  '6-14': {
    pre: [9, 16],
    from: 105,
    to: 108,
    next: [4, 20],
  },
  '16-6': {
    pre: [8, 3],
    from: 110,
    to: 113,
    next: [14, 9],
  },
  '7-15': {
    pre: [17, 1],
    from: 115,
    to: 118,
    next: [5, 12],
  },
  '17-7': {
    pre: [10, 3],
    from: 120,
    to: 123,
    next: [1, 15],
  },
  '10-8': {
    pre: [17, 12],
    from: 125,
    to: 128,
    next: [20, 16],
  },
  '8-16': {
    pre: [20, 10],
    from: 130,
    to: 133,
    next: [6, 3],
  },
  '8-20': {
    pre: [10, 16],
    from: 135,
    to: 138,
    next: [2, 14],
  },
  '11-9': {
    pre: [13, 4],
    from: 140,
    to: 143,
    next: [19, 6],
  },
  '9-19': {
    pre: [11, 6],
    from: 145,
    to: 148,
    next: [1, 3],
  },
  '12-10': {
    pre: [15, 2],
    from: 150,
    to: 153,
    next: [17, 8],
  },
  '10-17': {
    pre: [12, 8],
    from: 155,
    to: 158,
    next: [7, 3],
  },
  '11-13': {
    pre: [9, 4],
    from: 160,
    to: 163,
    next: [1, 5],
  },
  '15-12': {
    pre: [7, 5],
    from: 165,
    to: 168,
    next: [10, 2],
  },
  '14-20': {
    pre: [4, 6],
    from: 170,
    to: 173,
    next: [2, 8],
  },
  '20-8': {
    pre: [2, 14],
    from: 175,
    to: 178,
    next: [10, 16],
  }
}

const props = defineProps({
  endNumber: Number,
})
const { endNumber } = toRefs(props);

// Build bidirectional graph
function buildGraph(info) {
  const graph = {};
  for (const key in info) {
    if (key === "center") continue;
    const [a, b] = key.split("-").map(Number);
    if (!graph[a]) graph[a] = [];
    if (!graph[b]) graph[b] = [];
    graph[a].push(b);
    graph[b].push(a);
  }
  return graph;
}

function findShortestPath(start, end, graph) {
  const queue = [[start]];
  const visited = new Set([start]);

  while (queue.length) {
    const path = queue.shift();
    const node = path[path.length - 1];

    if (node === end) return path;

    for (const neighbor of graph[node] || []) {
      if (!visited.has(neighbor)) {
        visited.add(neighbor);
        queue.push([...path, neighbor]);
      }
    }
  }

  return null;
}

const scale = 0.2;
const frameCols = 5;
const originalFrameWidth = 1412 / 4;
const originalFrameHeight = 12708 / 36;
const frameWidth = originalFrameWidth * scale;
const frameHeight = originalFrameHeight * scale;
const options = { isStatic: true };
const graph = buildGraph(infoSprites);

let isRolling = ref(false)
let resultFrame = ref(150)
let dice
let currentFrame = 0
let engine
let render
let sprite

let lastXPosition = ref(0);
let lastSpeed = ref(0);

let branches = [];
let isBranchActive = false;
let branchDuration = ref(400); // 2.5 seconds in milliseconds
let startHue = 220;

// Dice shake animation variables
let isShaking = false;
let shakeStartTime = 0;
let shakeDuration = 500; // 1 second in milliseconds
let maxShakeIntensity = 20;
let shakeOffset = { x: 0, y: 0 };
let originalDicePosition = { x: 0, y: 0 };
let numbersPath
let activePathSprites = ref([]);

onMounted(() => {
  const is4K = window.innerWidth >= 3840 || window.innerHeight >= 2160;
  const isMobile = Math.min(window.innerWidth, window.innerHeight) <= 768;
  // FHD оставляем текущие параметры, для 4K короче, для мобильных ещё короче
  branchDuration.value = isMobile ? 1000 : (is4K ? 400 : 400);
  // Ждём полной загрузки спрайта, затем инициализируем движок и анимации
  uploadSprite().then(() => initAfterSpriteLoaded());
})

function initAfterSpriteLoaded() {
  numbersPath = findShortestPath(17, 18, graph);
  foundActivePath();

  engine = Engine.create();
  engine.world.gravity.y = 0;
  const currentWidth = window.innerWidth - 50;

  // Make walls much thicker and taller to prevent escape
  const wallThickness = 50; // Increased from 10 to 50
  const wallHeight = 600; // Increased from 400 to 600
  const containerHeight = 400;

  const whiteWallRender = { render: { fillStyle: '#ffffff', strokeStyle: '#ffffff' } };
  const ground = Bodies.rectangle(
    currentWidth / 2,
    containerHeight + wallThickness / 2,
    currentWidth + wallThickness * 2,
    wallThickness,
    { ...options, ...whiteWallRender }
  );
  const leftWall = Bodies.rectangle(
    -wallThickness / 2,
    containerHeight / 2,
    wallThickness,
    wallHeight,
    { ...options, ...whiteWallRender }
  );
  const rightWall = Bodies.rectangle(
    currentWidth + wallThickness / 2,
    containerHeight / 2,
    wallThickness,
    wallHeight,
    { ...options, ...whiteWallRender }
  );
  const topWall = Bodies.rectangle(
    currentWidth / 2,
    -wallThickness / 2,
    currentWidth + wallThickness * 2,
    wallThickness,
    { ...options, ...whiteWallRender }
  );

  render = Render.create({
    element: diceContainer.value,
    engine,
    options: {
      width: currentWidth,
      height: 400,
      wireframes: false,
    }
  });

  // Повышаем чёткость: рендерим в разрешении устройства (HiDPI)
  const dpr = window.devicePixelRatio || 1;
  if (Render.setPixelRatio) {
    Render.setPixelRatio(render, dpr);
  } else {
    render.options.pixelRatio = dpr;
    const canvas = render.canvas;
    canvas.width = render.options.width * dpr;
    canvas.height = render.options.height * dpr;
    canvas.style.width = render.options.width + 'px';
    canvas.style.height = render.options.height + 'px';
  }
  // Настройки сглаживания при масштабировании изображений
  render.context.imageSmoothingEnabled = true;
  console.log('imageSmoothingQuality' in render.context)
  if ('imageSmoothingQuality' in render.context) {
    render.context.imageSmoothingQuality = 'high';
  }

  getDiceBody(currentWidth);

  World.add(engine.world, [dice, ground, leftWall, rightWall, topWall]);
  const direction = (Math.random() * (40 - 30) + 30) * -1;
  const power = direction < -10 ? (Math.random() * (40 - 30) + 30) * -1 : Math.floor(Math.random() * 10);
  console.log(direction)
  Matter.Body.setVelocity(dice, {
    x: direction, // move left (negative x)
    y: power, // optional upward force
  });
  Events.on(render, 'afterRender', () => handleAfterRender());

  Render.run(render);
  setTimeout(() => Runner.run(Runner.create(), engine), 1000);

  Matter.Events.on(engine, 'afterUpdate', handleMoveUpdate);

  setTimeout(() => {
    resultFrame.value = infoSprites.center[endNumber.value];
    isRolling.value = true
    stopMoving();
  }, 4500)

  setTimeout(() => {
    createBranchAnimation();
  }, 4200)
}

function foundActivePath() {
  for (let mainIndex = 0; mainIndex < numbersPath.length - 1; mainIndex++) {

    const path = infoSprites[`${ numbersPath[mainIndex] }-${ numbersPath[mainIndex + 1] }`] || infoSprites[`${ numbersPath[mainIndex + 1] }-${ numbersPath[mainIndex] }`]
    activePathSprites.value.push(infoSprites.center[numbersPath[mainIndex]]);
    let index = path.from
    for (index; index <= path.to; index++) {
      activePathSprites.value.push(index);
    }


    if (infoSprites[`${ numbersPath[mainIndex] }-${ numbersPath[mainIndex + 1] }`]) {
      if (mainIndex + 1 === numbersPath.length - 1) {
        activePathSprites.value.push(infoSprites.center[numbersPath[mainIndex + 1]]);
      }
    } else {
      if (mainIndex + 1 === numbersPath.length - 1) {
        activePathSprites.value.push(infoSprites.center[numbersPath[mainIndex]]);
      }
    }
  }
}

function handleAfterRender() {

  const ctx = render.context;
  const { position, angle } = dice;

  // Убедимся, что качество сглаживания высокое при каждой перерисовке
  ctx.imageSmoothingEnabled = true;
  if ('imageSmoothingQuality' in ctx) {
    ctx.imageSmoothingQuality = 'high';
  }

  const col = currentFrame % frameCols;
  const row = Math.floor(currentFrame / frameCols);
  ctx.save();
  ctx.translate(position.x, position.y);
  ctx.rotate(angle);

  // Optional background circle (instead of square)
  const radius = Math.min(frameWidth, frameHeight) / 2;
  ctx.fillStyle = 'rgba(20, 21, 31, 1)'; // dark circle background
  ctx.arc(0, 0, radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.drawImage(
    sprite,
    col * originalFrameWidth,         // источник X
    row * originalFrameHeight,        // источник Y
    originalFrameWidth,               // ширина кадра в спрайте
    originalFrameHeight,              // высота кадра в спрайте
    -frameWidth / 2,                  // отрисовка: сместить к центру
    -frameHeight / 2,
    frameWidth,
    frameHeight
  );

  // Draw branch animation if active
  if (isBranchActive) drawBranches(ctx);
}

function handleMoveUpdate() {
  updateSprite();
  currentFrame = resultFrame.value; // now show the final frame
  if (Math.hypot(dice.velocity.x, dice.velocity.y) < 0.1) {
    Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
    // Store original position before starting shake
    originalDicePosition = { x: dice.position.x, y: dice.position.y };
    // Start shake animation when dice stops
    startShakeAnimation();
  }
}

// Branch animation functions
function rand(min, max) {
  return Math.random() * (max - min) + min;
}

function Branch(hue, x, y, angle, vel) {
  const move = 40;
  this.x = x + rand(-move, move);
  this.y = y + rand(-move, move);
  this.points = [];
  this.angle = angle != undefined ? angle : rand(0, Math.PI * 1);
  // Increase initial velocity range to make branches larger
  this.vel = vel != undefined ? vel : rand(-2, 2);
  this.spread = 0;
  this.tick = 0;
  this.hue = hue != undefined ? hue : 200;
  this.life = 1;
  this.decay = 0.005; // Увеличил скорость затухания для завершения за 1.5 секунды
  this.dead = false;

  this.points.push({
    x: this.x,
    y: this.y
  });
}

Branch.prototype.step = function (i) {
  this.life -= this.decay;
  if (this.life <= 0) {
    this.dead = true;
  }

  if (!this.dead) {
    const lastPoint = this.points[this.points.length - 1];
    this.points.push({
      x: lastPoint.x + Math.cos(this.angle) * this.vel,
      y: lastPoint.y + Math.sin(this.angle) * this.vel
    });
    this.angle += rand(-this.spread, this.spread);
    this.vel *= 0.99;
    // Increase spread so branches cover more area
    this.spread = this.vel * 0.6;
    this.tick++;
    this.hue += 0.3;
  } else {
    branches.splice(i, 1);
  }
};

Branch.prototype.draw = function (ctx, offset = { x: 0, y: 0 }) {
  if (!this.points.length || this.dead) return false;

  const length = this.points.length;
  const i = length - 1;
  const point = this.points[i];
  const lastPoint = this.points[i - Math.floor(5 + Math.random() * (100 - 5 + 1))];

  if (lastPoint) {
    const jitter = 2 + this.life * 6;
    ctx.beginPath();
    ctx.moveTo(lastPoint.x + offset.x, lastPoint.y + offset.y);
    ctx.lineTo(point.x + rand(-jitter, jitter) + offset.x, point.y + rand(-jitter, jitter) + offset.y);
    // Thicker lines for a larger visual effect
    ctx.lineWidth = 2;
    const alpha = this.life * 0.8; // Увеличил альфа с 0.075 до 0.4
    ctx.strokeStyle = `rgba(253, 181, 21, ${ alpha })`; // Changed to #fdb515 color
    ctx.stroke();
  }
};

function createBranchAnimation() {
  if (isBranchActive) return;
  isBranchActive = true;
  branches = [];

  for (let i = 0; i < 600; i++) {
    branches.push(new Branch(startHue, 0, 0));
  }

  // Stop animation after configured duration
  setTimeout(() => {
    stopBranchAnimation();
  }, branchDuration.value);
}

function drawBranches(ctx) {
  // Step animation
  let i = branches.length;
  while (i--) {
    branches[i].step(i);
  }

  // Draw branches clipped to dice circle
  ctx.save();
  const radius = Math.min(frameWidth, frameHeight) / 2;
  ctx.beginPath();
  ctx.arc(0, 0, radius, 0, Math.PI * 2);
  ctx.clip();
  ctx.globalCompositeOperation = 'lighter';
  i = branches.length;
  while (i--) {
    branches[i].draw(ctx, { x: 0, y: 0 });
  }
  ctx.restore();

  // Stop animation when no branches left
  if (branches.length === 0) {
    stopBranchAnimation();
  }
}

function stopBranchAnimation() {
  isBranchActive = false;
  branches = [];
  // branchTick = 0; // This line is no longer needed for time-based animation
}

function updateSprite() {
  if (!window.lastUpdateSpriteTime) window.lastUpdateSpriteTime = Date.now();
  const now = Date.now();

  const speed = Math.hypot(dice.velocity.x, dice.velocity.y);

  if (isRolling.value) return;
  if (now - window.lastUpdateSpriteTime >= 1) {
    // resultFrame.value = Math.floor(Math.random() * 10);
    if (lastSpeed.value > 1 && +speed.toFixed(2) > 0.3) {
      const last = activePathSprites.value.length === 1 ? numbersPath[numbersPath.length - 1] : null;

      if (lastXPosition.value > dice.velocity.x) {
        resultFrame.value = activePathSprites.value[activePathSprites.value.length - 1]
        activePathSprites.value.pop()
      } else {
        resultFrame.value = activePathSprites.value[0]
        activePathSprites.value.shift()
      }

      if (last) {
        numbersPath = findShortestPath(last, getRandomInt(20, last), graph);
        foundActivePath();
      }
    }
    lastSpeed.value = +speed.toFixed(2);
    window.lastUpdateSpriteTime = now;
  }
}

function getRandomInt(max, instead) {
  const number = Math.floor(Math.random() * max) + 1
  if (number === instead) {
    return getRandomInt(max, instead);
  } else {
    return number;
  }
}

function moveRightToLeft() {
  if (isRolling.value) return;

  // Check if dice is still moving - don't allow clicks when dice has stopped
  const speed = Math.hypot(dice.velocity.x, dice.velocity.y);
  if (speed <= 0.1) return;

  // Stop any existing branch animation
  stopBranchAnimation();

  isRolling.value = true;
  const power = Math.floor(Math.random() * 30) * -1;
  Matter.Body.setVelocity(dice, {
    x: power,
    y: Math.floor(Math.random() * 30) * -1,
  });
  Matter.Body.setAngularVelocity(dice, (Math.random() - 0.5) * 2);
}

function stopMoving() {
  if (dice) {
    Matter.Body.setVelocity(dice, { x: 0, y: 0 });
    Matter.Body.setAngularVelocity(dice, 0);
  }
  if (engine && engine.runner) {
    Runner.stop(engine.runner);
  }
  isRolling.value = false;
}

function getDiceBody(currentWidth, air) {
  const radius = Math.min(frameWidth, frameHeight) / 2;
  dice = Bodies.circle(currentWidth - 100, 200, radius, {
    restitution: 0.8,
    friction: 0.7,
    density: 0.9,
    frictionAir: air || 0,
    render: {
      visible: false,
    }
  });
}

function uploadSprite() {
  return new Promise((resolve, reject) => {
    sprite = new Image();
    sprite.onload = () => resolve(true);
    sprite.onerror = (e) => reject(e);
    sprite.src = '/images/33.png';
  });
}

// Shake animation functions
function startShakeAnimation() {
  if (isShaking) return;

  isShaking = true;
  shakeStartTime = Date.now(); // Set start time
  maxShakeIntensity = 7; // Increased from 5 to 20

  const shakeAnimation = () => {
    if (!isShaking || (Date.now() - shakeStartTime) > shakeDuration) { // Use shakeStartTime and shakeDuration
      stopShakeAnimation();
      return;
    }

    // Calculate shake intensity based on time
    const elapsedTime = Date.now() - shakeStartTime;
    const intensity = maxShakeIntensity * (1 - (elapsedTime / shakeDuration)); // Intensity decreases over time

    // Generate random shake offset with larger range
    shakeOffset.x = (Math.random() - 0.5) * intensity * 2; // Added multiplier of 2
    shakeOffset.y = (Math.random() - 0.5) * intensity * 2; // Added multiplier of 2

    // Apply shake to dice position
    Matter.Body.setPosition(dice, {
      x: originalDicePosition.x + shakeOffset.x,
      y: originalDicePosition.y + shakeOffset.y
    });

    // Continue animation
    requestAnimationFrame(shakeAnimation);
  };

  shakeAnimation();

  // Stop shake after 1 second
  setTimeout(() => {
    stopShakeAnimation();
  }, 500);
}

function stopShakeAnimation() {
  if (!isShaking) return;

  isShaking = false;
  shakeStartTime = 0; // Reset start time
  shakeDuration = 1000; // Reset duration
  maxShakeIntensity = 20; // Reset max intensity
  shakeOffset = { x: 0, y: 0 };

  // Reset dice to original position
  if (dice && originalDicePosition.x && originalDicePosition.y) {
    Matter.Body.setPosition(dice, {
      x: originalDicePosition.x,
      y: originalDicePosition.y
    });
  }
}

onUnmounted(() => {
  Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
  stopBranchAnimation();
})
</script>

