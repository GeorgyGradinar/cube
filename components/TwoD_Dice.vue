<template>
  <div ref="diceContainer" @click="moveRightToLeft" />
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Matter from 'matter-js'

const { Engine, Render, Runner, World, Bodies, Body, Events } = Matter;
const diceContainer = ref(null)

const infoSprites = {
  1: {
    left: {
      from: 0,
      to: 4
    },
    right: {
      from: 5,
      to: 9
    },
    center: 5,
  },
  2: {
    left: {
      from: 80,
      to: 84
    },
    right: {
      from: 85,
      to: 89
    },
    center: 85,
  },
  3: {
    left: {
      from: 100,
      to: 104
    },
    right: {
      from: 105,
      to: 109
    },
    center: 105,
  },
  4: {
    left: {
      from: 70,
      to: 74
    },
    right: {
      from: 75,
      to: 79
    },
    center: 75,
  },
  5: {
    left: {
      from: 120,
      to: 124
    },
    right: {
      from: 125,
      to: 129
    },
    center: 125,
  },
  6: {
    left: {
      from: 60,
      to: 64
    },
    right: {
      from: 65,
      to: 69
    },
    center: 65,
  },
  7: {
    left: {
      from: 110,
      to: 114
    },
    right: {
      from: 115,
      to: 119
    },
    center: 115,
  },
  8: {
    left: {
      from: 50,
      to: 54
    },
    right: {
      from: 55,
      to: 59
    },
    center: 55,
  },
  9: {
    left: {
      from: 155,
      to: 159
    },
    right: {
      from: 160,
      to: 164
    },
    center: 160,
  },
  10: {
    left: {
      from: 90,
      to: 94
    },
    right: {
      from: 95,
      to: 99
    },
    center: 95,
  },
  11: {
    left: {
      from: 10,
      to: 14
    },
    right: {
      from: 15,
      to: 19
    },
    center: 15,
  },
  12: {
    left: {
      from: 40,
      to: 44
    },
    right: {
      from: 45,
      to: 49
    },
    center: 45,
  },
  13: {
    left: {
      from: 145,
      to: 149
    },
    right: {
      from: 150,
      to: 154
    },
    center: 150,
  },
  14: {
    left: {
      from: 175,
      to: 179
    },
    right: {
      from: 180,
      to: 184
    },
    center: 180,
  },
  15: {
    left: {
      from: 135,
      to: 139
    },
    right: {
      from: 140,
      to: 144
    },
    center: 140,
  },
  16: {
    left: {
      from: 165,
      to: 169
    },
    right: {
      from: 170,
      to: 174
    },
    center: 170,
  },
  17: {
    left: {
      from: 30,
      to: 34
    },
    right: {
      from: 35,
      to: 39
    },
    center: 35,
  },
  18: {
    left: {
      from: 185,
      to: 189
    },
    right: {
      from: 130,
      to: 134
    },
    center: 130,
  },
  19: {
    left: {
      from: 20,
      to: 24
    },
    right: {
      from: 25,
      to: 29
    },
    center: 25,
  },
  20: {
    left: {
      from: 190,
      to: 194
    },
    right: {
      from: 195,
      to: 199
    },
    center: 195,
  },
}

const scale = 0.2;
const frameCols = 5;
const frameRows = 2;
const originalFrameWidth = 1771 / 5;
const originalFrameHeight = 21180 / 60;
const frameWidth = originalFrameWidth * scale;
const frameHeight = originalFrameHeight * scale;
const options = { isStatic: true };
const color = '#fdfdfd';

let isRolling = ref(false)
let resultFrame = ref(0)
let dice
let currentFrame = 0
let engine
let render
let sprite

onMounted(() => {
  uploadSprite();

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
    x: direction, // move left (negative x)
    y: 1, // optional upward force
  });
  Matter.Body.setAngularVelocity(dice, 0);
  dice.angle = 0; // или другое нужное значение
  dice.inertia = Infinity;
  Events.on(render, 'afterRender', () => handleAfterRender());

  Render.run(render);
  setTimeout(() => Runner.run(Runner.create(), engine), 1000);

  Matter.Events.on(engine, 'afterUpdate', handleMoveUpdate);
})

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
  console.log('-------------------------')
  console.log(isRolling.value)
  console.log(speed)
  console.log(angular)
  if (speed > 0.1) {
    console.log(resultFrame.value)
    currentFrame = resultFrame.value; // now show the final frame
  } else {
    currentFrame = 99;
    Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
  }
  // isRolling.value = false;
}

function updateSprite() {
  if (!window.lastUpdateSpriteTime) window.lastUpdateSpriteTime = Date.now();
  const now = Date.now();
  console.log(now - window.lastUpdateSpriteTime)
  if (now - window.lastUpdateSpriteTime >= 40) {
    // resultFrame.value = Math.floor(Math.random() * 10);
    resultFrame.value = resultFrame.value + 1;
    window.lastUpdateSpriteTime = now;
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

function getDiceBody(currentWidth) {
  dice = Bodies.rectangle(currentWidth - 100, 200, frameWidth, frameHeight, {
    restitution: 0.8,
    friction: 0.3,
    density: 0.7,
    frictionAir: 0.02,
    inertia: Infinity, // <--- вот это ключевое!
  });
}

function uploadSprite() {
  sprite = new Image();
  sprite.src = '/images/dices2.png';
}

onUnmounted(() => {
  Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
})
</script>
