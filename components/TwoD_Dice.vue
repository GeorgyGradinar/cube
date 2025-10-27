<template>
  <div ref="diceContainer" @click="moveRightToLeft" />
</template>

<script setup>
import { onMounted, ref } from 'vue'
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
  console.log(`start - ${ start }`)
  console.log(`end - ${ end }`)
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

function convertNumbersToKeys(path) {
  const result = [];
  for (let i = 0; i < path.length - 1; i++) {
    const key = `${ path[i] }-${ path[i + 1] }`;
    result.push(key);
  }
  return result;
}

const scale = 0.2;
const frameCols = 5;
const frameRows = 2;
const originalFrameWidth = 1412 / 4;
const originalFrameHeight = 12707 / 36;
const frameWidth = originalFrameWidth * scale;
const frameHeight = originalFrameHeight * scale;
const options = { isStatic: true };
const color = '#fdfdfd';

let isRolling = ref(false)
let resultFrame = ref(150)
let dice
let currentFrame = 0
let engine
let render
let sprite

let debounceTimeout = ref(null);
let lastXPosition = ref(0);
let lastSpeed = ref(0);

const graph = buildGraph(infoSprites);

let numbersPath

let activePathSprites = ref([]);

onMounted(() => {
  uploadSprite();
  numbersPath = findShortestPath(17, 18, graph);
  console.log(convertNumbersToKeys(numbersPath))
  foundActivePath();

  engine = Engine.create();
  engine.world.gravity.y = 0;
  const currentWidth = window.innerWidth - 50;
  const ground = Bodies.rectangle(currentWidth / 2, 400, currentWidth, 10, options);
  const leftWall = Bodies.rectangle(0, 200, 10, 400, options);
  const rightWall = Bodies.rectangle(currentWidth, 200, 10, 400, options);
  const topWall = Bodies.rectangle(currentWidth / 2, 0, currentWidth, 10, options);

  render = Render.create({
    element: diceContainer.value,
    engine,
    options: {
      width: currentWidth,
      height: 400,
      wireframes: false,
      background: color
    }
  });

  getDiceBody(currentWidth);

  World.add(engine.world, [dice, ground, leftWall, rightWall, topWall]);
  const direction = (Math.random() * (80 - 30) + 30) * -1;
  const power = direction < -10 ? (Math.random() * (60 - 30) + 30) * -1 : Math.floor(Math.random() * 10);

  Matter.Body.setVelocity(dice, {
    x: -15, // move left (negative x)
    y: 5, // optional upward force
  });
  // Matter.Body.setAngularVelocity(dice, 0);
  // dice.angle = 0; // или другое нужное значение
  // dice.inertia = Infinity;
  Events.on(render, 'afterRender', () => handleAfterRender());

  Render.run(render);
  setTimeout(() => Runner.run(Runner.create(), engine), 1000);

  Matter.Events.on(engine, 'afterUpdate', handleMoveUpdate);

  setTimeout(() => {
    resultFrame.value = infoSprites.center[15];
    isRolling.value = true
    stopMoving();
  }, 4500)
})

function foundActivePath() {
  console.log(numbersPath)
  for (let mainIndex = 0; mainIndex < numbersPath.length - 1; mainIndex++) {

    console.log(numbersPath[mainIndex])
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

    console.log(JSON.parse(JSON.stringify(activePathSprites.value)))
    console.log('-------------')
  }

  // resultFrame.value = activePathSprites.value[0]
  // activePathSprites.value.shift()
}

function handleAfterRender() {

  const ctx = render.context;
  const { position, angle } = dice;

  const col = currentFrame % frameCols;
  const row = Math.floor(currentFrame / frameCols);
  ctx.save();
  ctx.translate(position.x, position.y);
  ctx.rotate(angle);
  ctx.fillStyle = color; // or any yellow color
  ctx.fillRect(-frameWidth / 2 - 1, -frameHeight / 2 - 1, frameWidth + 2, frameHeight + 2);
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
  ctx.beginPath();
  ctx.restore();
}

function handleMoveUpdate() {
  const speed = Math.hypot(dice.velocity.x, dice.velocity.y);
  const angular = Math.abs(dice.angularVelocity);
  updateSprite();


  // console.log(isRolling.value)
  // console.log(speed)
  // console.log(angular)
  // console.log(dice.velocity.x)

  // if (lastXPosition.value > dice.velocity.x) {
  //   console.log('left')
  // } else {
  //   console.log('right')
  // }
  currentFrame = resultFrame.value; // now show the final frame
  if (speed > 0.1) {
    // currentFrame = resultFrame.value; // now show the final frame
  } else {
    // currentFrame = 99;
    Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
  }
  // isRolling.value = false;
}

function updateSprite() {
  if (!window.lastUpdateSpriteTime) window.lastUpdateSpriteTime = Date.now();
  const now = Date.now();

  const speed = Math.hypot(dice.velocity.x, dice.velocity.y);

  // Выбираем интервал в зависимости от скорости
  let interval = speed < 10 ? speed * 100 : 10;
  if (+speed < 5) {
    interval = 95 - (+speed * 10)
  } else if (+speed < 10) {
    interval = 95 - (+speed * 5)
  } else {
    interval = 1
  }

  if (isRolling.value) return;
  if (now - window.lastUpdateSpriteTime >= 1) {
    // resultFrame.value = Math.floor(Math.random() * 10);
    if (lastSpeed.value > 1 && +speed.toFixed(2) < 0.3) {
      // resultFrame.value = 189
    } else {
      const last = activePathSprites.value.length === 1 ? numbersPath[numbersPath.length - 1] : null;

      if (lastXPosition.value > dice.velocity.x) {
        // resultFrame.value = resultFrame.value - 1;
        resultFrame.value = activePathSprites.value[activePathSprites.value.length - 1]
        activePathSprites.value.pop()
      } else {
        resultFrame.value = activePathSprites.value[0]
        activePathSprites.value.shift()
        // resultFrame.value = resultFrame.value + 1;
      }

      console.log(resultFrame.value)
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
  isRolling.value = true;
  const power = Math.floor(Math.random() * 30) * -1;
  Matter.Body.setVelocity(dice, {
    x: 20,
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
  dice = Bodies.rectangle(currentWidth - 100, 200, frameWidth, frameHeight, {
    restitution: 0.8,
    friction: 0.3,
    density: 0.7,
    frictionAir: air || 0,
    // inertia: Infinity, // <--- вот это ключевое!
  });
}

function uploadSprite() {
  sprite = new Image();
  sprite.src = '/images/dices-new.png';
}

function debounceTime(callbackFn, time) {
  if (debounceTimeout) clearTimeout(debounceTimeout);

  debounceTimeout = setTimeout(() => {
    callbackFn();
  }, time ? time : 200);
}

onUnmounted(() => {
  Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
})
</script>
