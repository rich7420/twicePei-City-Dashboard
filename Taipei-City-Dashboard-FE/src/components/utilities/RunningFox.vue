<!-- Running Fox Animation Component -->
<template>
  <div class="fox-container">
    <div class="fox" :style="{ left: foxPosition + 'px' }" :class="{ 'fox-reverse': isReverse }">
			<img src="./image.png" alt="Running Fox" />
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

const isReverse = ref(false);
const foxPosition = ref(0);
const foxSpeed = ref(3); // 降低移動速度從3改為1.5
const foxSize = 40; // SVG的寬度
let animationFrame;

const startFoxAnimation = () => {
  // 隨機決定狐狸的初始方向和位置
  isReverse.value = Math.random() > 0.5;
  foxPosition.value = isReverse.value ? window.innerWidth - foxSize : 0;
  foxSpeed.value = isReverse.value ? -3.0 : 3.0; // 降低移動速度
  
  // 開始移動動畫
  const moveFox = () => {
    const nextPosition = foxPosition.value + foxSpeed.value;
    
    // 檢查邊界碰撞
    if (nextPosition <= 0) {
      foxPosition.value = 0;
      foxSpeed.value = Math.abs(foxSpeed.value); // 向右
      isReverse.value = false;
    } else if (nextPosition >= window.innerWidth - foxSize) {
      foxPosition.value = window.innerWidth - foxSize;
      foxSpeed.value = -Math.abs(foxSpeed.value); // 向左
      isReverse.value = true;
    } else {
      foxPosition.value = nextPosition;
    }
    
    animationFrame = requestAnimationFrame(moveFox);
  };
  
  moveFox();
};

// 監聽視窗大小變化
const handleResize = () => {
  if (foxPosition.value > window.innerWidth - foxSize) {
    foxPosition.value = window.innerWidth - foxSize;
  }
};

onMounted(() => {
  // 立即開始狐狸動畫
  startFoxAnimation();
  window.addEventListener('resize', handleResize);
});

onBeforeUnmount(() => {
  if (animationFrame) {
    cancelAnimationFrame(animationFrame);
  }
  window.removeEventListener('resize', handleResize);
});
</script>

<style scoped>
.fox-container {
  position: fixed;
  top: 15px; /* 在navbar中間位置 */
  left: 0;
  width: 100vw;
  height: 30px;
  pointer-events: none;
  z-index: 1000; /* 確保在navbar之上但不會太高 */
  overflow: hidden;
}

.fox {
  position: absolute;
  top: 0;
  filter: drop-shadow(1px 1px 2px rgba(0, 0, 0, 0.3));
  transition: transform 0.1s ease-out;
  animation: bounce 1s ease-in-out infinite alternate;
}

.fox img {
  width: 30px;
  height: 30px;
}

.fox-reverse {
  transform: scaleX(-1);
}

/* 輕微的上下彈跳動畫 */
@keyframes bounce {
  0% {
    transform: translateY(0px);
  }
  100% {
    transform: translateY(-5px);
  }
}

.fox-reverse {
  animation: bounceReverse 1s ease-in-out infinite alternate;
}

@keyframes bounceReverse {
  0% {
    transform: translateY(0px) scaleX(-1);
  }
  100% {
    transform: translateY(-5px) scaleX(-1);
  }
}

/* 在小屏幕設備上調整狐狸大小 */
@media (max-width: 768px) {
  .fox svg {
    width: 30px;
    height: 30px;
  }
  
  .fox-container {
    top: 20px; /* 手機版navbar可能高度不同 */
  }
}

/* 當用戶懸停時狐狸會暫停並放大 */
.fox:hover {
  animation-play-state: paused;
  transform: scale(1.3) !important;
  transition: transform 0.3s ease;
}
</style>