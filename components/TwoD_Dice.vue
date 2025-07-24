<template>
  <div ref="diceContainer" @click="moveRightToLeft" />
</template>

<script setup>
import { onMounted, ref } from 'vue'
import Matter from 'matter-js'

const { Engine, Render, Runner, World, Bodies, Body, Events } = Matter;
const diceContainer = ref(null)

const scale = 0.2;
const frameCols = 5;
const frameRows = 2;
const originalFrameWidth = 1765 / 5;
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
    y: 12, // optional upward force
  });
  Matter.Body.setAngularVelocity(dice, 1);
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
  if ((isRolling.value && speed > 0.1) || angular > 0.05) {
    console.log(resultFrame.value)
    currentFrame = resultFrame.value; // now show the final frame
  } else {
    currentFrame = 6;
    Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
  }
  isRolling.value = false;
}

function updateSprite() {
  if (!window.lastUpdateSpriteTime) window.lastUpdateSpriteTime = Date.now();
  const now = Date.now();
  if (now - window.lastUpdateSpriteTime >= 40) {
    resultFrame.value = Math.floor(Math.random() * 10);
    window.lastUpdateSpriteTime = now;
  }
}

function moveRightToLeft() {
  if (isRolling.value) return;
  isRolling.value = true;
  const power = Math.floor(Math.random() * 30) * -1;
  Matter.Body.setVelocity(dice, {
    x: power,
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
  });
}

function uploadSprite() {
  sprite = new Image();
  sprite.src = '/images/dicess.png';
}

onUnmounted(() => {
  Matter.Events.off(engine, 'afterUpdate', handleMoveUpdate);
})
</script>
