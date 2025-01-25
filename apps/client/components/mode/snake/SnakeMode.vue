<template>
  <div class="relative h-full w-full">
    <div class="mb-4 text-center">
      <p class="text-xl font-bold mb-2">{{ courseStore.currentStatement?.chinese }}</p>
      <p class="text-lg text-gray-600">{{ courseStore.currentStatement?.english }}</p>
    </div>
    <canvas
      ref="gameCanvas"
      class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 border border-gray-300 dark:border-gray-600"
      :width="canvasWidth"
      :height="canvasHeight"
    ></canvas>

    <div
      v-if="!isGameStarted"
      class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-center"
    >
      <p class="mb-4 text-xl">{{ courseStore.currentStatement?.chinese }}</p>
      <p class="mb-4 text-lg text-gray-500">使用方向键或 WASD 控制蛇的移动</p>
      <button
        class="btn btn-primary"
        @click="startGame"
      >
        开始游戏
      </button>
    </div>

    <div
      v-if="isGameOver"
      class="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-center"
    >
      <p class="mb-4 text-xl text-red-500">游戏结束</p>
      <button
        class="btn btn-primary"
        @click="restartGame"
      >
        重新开始
      </button>
    </div>

    <MainPrevAndNextBtn />
  </div>
</template>

<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";

import { useCourseStore } from "~/store/course";
import { useGameMode } from "~/composables/main/game";
import { useShortcutKeyMode } from "~/composables/user/shortcutKey";

const courseStore = useCourseStore();
const { shortcutKeys } = useShortcutKeyMode();
const { showQuestion } = useGameMode();

function usePrevAndNextQuestion(prevKey: string, nextKey: string) {
  function goToNextQuestion() {
    courseStore.toNextStatement();
    showQuestion();
    if (isGameStarted.value) {
      clearInterval(gameLoop!);
      startGame();
    }
  }

  function goToPreviousQuestion() {
    courseStore.toPreviousStatement();
    showQuestion();
    if (isGameStarted.value) {
      clearInterval(gameLoop!);
      startGame();
    }
  }

  return { goToNextQuestion, goToPreviousQuestion };
}

const { goToNextQuestion, goToPreviousQuestion } = usePrevAndNextQuestion(
  shortcutKeys.value.previous,
  shortcutKeys.value.skip
);
const gameCanvas = ref<HTMLCanvasElement | null>(null);
const canvasWidth = 800;
const canvasHeight = 600;
const gridSize = 25;

const isGameStarted = ref(false);
const isGameOver = ref(false);

interface SnakeSegment {
  x: number;
  y: number;
}

interface Letter {
  x: number;
  y: number;
  char: string;
}

let snake: SnakeSegment[] = [];
let direction = { x: 1, y: 0 };
let letters: Letter[] = [];
let targetWord = "";
let collectedLetters = "";
let gameLoop: number | null = null;
let errorMessageTimer: number | null = null;
let errorMessage: string = "";

function initGame() {
  // 初始化蛇的位置
  snake = [
    { x: 5, y: 10 },
    { x: 4, y: 10 },
    { x: 3, y: 10 },
  ];

  // 获取目标单词并打乱字母，过滤掉空格
  targetWord = (courseStore.currentStatement?.english || "").replace(/\s+/g, "");
  const shuffledLetters = targetWord.split("").sort(() => Math.random() - 0.5);

  // 随机放置字母
  letters = shuffledLetters.map((char) => ({
    x: Math.floor(Math.random() * (canvasWidth / gridSize)),
    y: Math.floor(Math.random() * (canvasHeight / gridSize)),
    char,
  }));

  collectedLetters = "";
  direction = { x: 1, y: 0 };
  isGameOver.value = false;
}

function startGame() {
  isGameStarted.value = true;
  initGame();
  gameLoop = setInterval(updateGame, 200);
}

function restartGame() {
  isGameOver.value = false;
  startGame();
}

function updateGame() {
  const ctx = gameCanvas.value?.getContext("2d");
  if (!ctx) return;

  // 移动蛇
  const head = { x: snake[0].x + direction.x, y: snake[0].y + direction.y };

  // 穿墙处理
  if (head.x < 0) {
    head.x = Math.floor(canvasWidth / gridSize) - 1;
  } else if (head.x >= canvasWidth / gridSize) {
    head.x = 0;
  }
  if (head.y < 0) {
    head.y = Math.floor(canvasHeight / gridSize) - 1;
  } else if (head.y >= canvasHeight / gridSize) {
    head.y = 0;
  }

  // 检查是否撞到自己
  if (snake.some((segment) => segment.x === head.x && segment.y === head.y)) {
    gameOver();
    return;
  }

  snake.unshift(head);

  // 检查是否吃到字母
  const letterIndex = letters.findIndex(
    (letter) => letter.x === head.x && letter.y === head.y,
  );

  if (letterIndex !== -1) {
    const letter = letters[letterIndex];
    const expectedLetter = targetWord[collectedLetters.length];

    if (letter.char === expectedLetter) {
      // 吃到正确的字母
      collectedLetters += letter.char;
      letters.splice(letterIndex, 1);

      // 检查是否完成单词
      if (collectedLetters === targetWord) {
        clearInterval(gameLoop!);
        isGameStarted.value = false;
        // 添加延迟以确保状态更新
        setTimeout(() => {
          goToNextQuestion();
        }, 500);
      }
    } else {
      // 吃到错误的字母，撤回到上一个状态
      snake.shift();
      // 设置错误提示和计时器
      errorMessage = `错误！应该吃掉字母: ${expectedLetter}`;
      if (errorMessageTimer) clearTimeout(errorMessageTimer);
      errorMessageTimer = window.setTimeout(() => {
        errorMessage = "";
        errorMessageTimer = null;
      }, 1000);
    }
  } else {
    // 没有吃到字母，移除尾部
    snake.pop();
  }

  // 绘制游戏画面
  drawGame(ctx);
}

function drawGame(ctx: CanvasRenderingContext2D) {
  // 清空画布
  ctx.clearRect(0, 0, canvasWidth, canvasHeight);

  // 绘制蛇
  snake.forEach((segment, index) => {
    const x = segment.x * gridSize;
    const y = segment.y * gridSize;
    const radius = gridSize / 2 - 2;

    ctx.beginPath();
    if (index === 0) {
      // 蛇头使用特殊的渐变色
      const headGradient = ctx.createRadialGradient(
        x + gridSize/2,
        y + gridSize/2,
        0,
        x + gridSize/2,
        y + gridSize/2,
        radius
      );
      headGradient.addColorStop(0, '#2dd4bf');
      headGradient.addColorStop(1, '#059669');
      ctx.fillStyle = headGradient;

      // 绘制蛇头
      ctx.arc(x + gridSize/2, y + gridSize/2, radius, 0, Math.PI * 2);
      ctx.fill();

      // 绘制蛇眼睛
      ctx.fillStyle = '#000000';
      const eyeRadius = radius / 4;
      const eyeOffset = radius / 2;

      // 根据方向调整眼睛位置
      let leftEyeX = x + gridSize/2 - eyeOffset;
      let rightEyeX = x + gridSize/2 + eyeOffset;
      let eyeY = y + gridSize/2;

      if (direction.x === 1) { // 向右
        eyeY = y + gridSize/2;
      } else if (direction.x === -1) { // 向左
        eyeY = y + gridSize/2;
      } else if (direction.y === -1) { // 向上
        leftEyeX = x + gridSize/2 - eyeOffset;
        rightEyeX = x + gridSize/2 + eyeOffset;
        eyeY = y + gridSize/2 - eyeOffset/2;
      } else if (direction.y === 1) { // 向下
        leftEyeX = x + gridSize/2 - eyeOffset;
        rightEyeX = x + gridSize/2 + eyeOffset;
        eyeY = y + gridSize/2 + eyeOffset/2;
      }

      ctx.beginPath();
      ctx.arc(leftEyeX, eyeY, eyeRadius, 0, Math.PI * 2);
      ctx.arc(rightEyeX, eyeY, eyeRadius, 0, Math.PI * 2);
      ctx.fill();
    } else {
      // 蛇身使用渐变色
      const bodyGradient = ctx.createRadialGradient(
        x + gridSize/2,
        y + gridSize/2,
        0,
        x + gridSize/2,
        y + gridSize/2,
        radius
      );
      bodyGradient.addColorStop(0, '#34d399');
      bodyGradient.addColorStop(1, '#059669');
      ctx.fillStyle = bodyGradient;
      ctx.arc(x + gridSize/2, y + gridSize/2, radius, 0, Math.PI * 2);
      ctx.fill();
    }

    // 在蛇身上显示已收集的字母
    if (index < collectedLetters.length) {
      const centerX = x + gridSize/2;
      const centerY = y + gridSize/2;
      ctx.fillStyle = "#FFFFFF";
      ctx.font = "bold 16px Arial";
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(collectedLetters[index], centerX, centerY);
    }
  });

  // 绘制网格背景
  ctx.strokeStyle = "#E5E7EB";
  for (let x = 0; x <= canvasWidth; x += gridSize) {
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, canvasHeight);
    ctx.stroke();
  }
  for (let y = 0; y <= canvasHeight; y += gridSize) {
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvasWidth, y);
    ctx.stroke();
  }

  // 绘制字母
  ctx.fillStyle = "#6366F1";
  ctx.font = "bold 20px 'Courier New'";
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  letters.forEach((letter) => {
    const centerX = letter.x * gridSize + gridSize / 2;
    const centerY = letter.y * gridSize + gridSize / 2;
    
    // 绘制字母背景
    ctx.fillStyle = "#E0E7FF";
    ctx.beginPath();
    ctx.arc(centerX, centerY, gridSize / 2 - 2, 0, Math.PI * 2);
    ctx.fill();
    
    // 绘制字母
    ctx.fillStyle = "#4F46E5";
    ctx.fillText(letter.char, centerX, centerY);
  });

  // 显示游戏信息
  ctx.textAlign = "left";
  ctx.font = "bold 18px Arial";
  ctx.fillStyle = "#1F2937";
  ctx.fillText(`已收集: ${collectedLetters}`, 20, 40);
  ctx.fillText(`目标单词: ${targetWord}`, 20, 70);
  ctx.fillText(`下一个字母: ${targetWord[collectedLetters.length]}`, 20, 100);

  // 显示错误提示（如果有）
  if (errorMessage) {
    ctx.fillStyle = "#FF0000";
    ctx.font = "bold 16px Arial";
    ctx.fillText(errorMessage, canvasWidth / 2 - 100, 30);
  }
}

function gameOver() {
  clearInterval(gameLoop!);
  isGameOver.value = true;
}

function handleKeydown(e: KeyboardEvent) {
  if (!isGameStarted.value || isGameOver.value) return;

  switch (e.key.toLowerCase()) {
    case "arrowup":
    case "w":
      if (direction.y !== 1) direction = { x: 0, y: -1 };
      break;
    case "arrowdown":
    case "s":
      if (direction.y !== -1) direction = { x: 0, y: 1 };
      break;
    case "arrowleft":
    case "a":
      if (direction.x !== 1) direction = { x: -1, y: 0 };
      break;
    case "arrowright":
    case "d":
      if (direction.x !== -1) direction = { x: 1, y: 0 };
      break;
  }
}

onMounted(() => {
  window.addEventListener("keydown", handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener("keydown", handleKeydown);
  if (gameLoop) clearInterval(gameLoop);
  if (errorMessageTimer) clearTimeout(errorMessageTimer);
});
</script>