<script lang="ts" setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 组件属性
const props = defineProps({
  duration: {
    type: Number,
    default: 10 // 默认10秒
  },
  reminderTime: {
    type: Number,
    default: 2 // 默认剩余2秒提醒
  },
  title: {
    type: String,
    default: '倒计时'
  }
});

// 倒计时状态
const isRunning = ref(false);
const timeLeft = ref(props.duration);
const timerInterval = ref<number | null>(null);
const showReminder = ref(false);
const reminderMessage = ref('');
const hasReminder = ref(false);

// 格式化时间显示
const formatTime = (seconds: number): string => {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = seconds % 60;
  return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
};

// 播放提醒声音
const playReminderSound = (isLong = false) => {
  try {
    if (isLong) {
      // 长提示：播放三遍短提示音
      const playBeep = (times: number, interval: number) => {
        for (let i = 0; i < times; i++) {
          setTimeout(() => {
            const audio = new Audio('/short-beep.mp3');
            audio.volume = 0.5;
            audio.play().catch(error => {
              console.error('无法播放提示音:', error);
            });
          }, i * interval);
        }
      };
      playBeep(3, 1000); // 播放三遍，间隔1秒
    } else {
      // 短提示：播放一遍短提示音
      const audio = new Audio('/short-beep.mp3');
      audio.volume = 0.5;
      audio.play().catch(error => {
        console.error('无法播放提示音:', error);
      });
    }
  } catch (error) {
    console.error('无法播放声音:', error);
  }
};

// 显示提醒
const showNotification = (message: string, isLong = false) => {
  reminderMessage.value = message;
  showReminder.value = true;
  
  // 播放提醒声音
  playReminderSound(isLong);
  
  // 长提醒持续5秒，短提醒持续2秒
  setTimeout(() => {
    showReminder.value = false;
  }, isLong ? 5000 : 2000);
};

// 开始/暂停倒计时
const toggleTimer = () => {
  if (isRunning.value) {
    // 暂停
    if (timerInterval.value) {
      clearInterval(timerInterval.value);
      timerInterval.value = null;
    }
  } else {
    // 开始
    timerInterval.value = window.setInterval(() => {
      if (timeLeft.value > 0) {
        timeLeft.value--;
        
        // 提醒
        if (timeLeft.value === props.reminderTime && !hasReminder.value) {
          showNotification(`剩余${props.reminderTime}秒！`, false);
          hasReminder.value = true;
        }
      } else {
        // 倒计时结束
        if (timerInterval.value) {
          clearInterval(timerInterval.value);
          timerInterval.value = null;
        }
        isRunning.value = false;
        showNotification('倒计时完成！', true);
      }
    }, 1000);
  }
  isRunning.value = !isRunning.value;
};

// 重置倒计时
const resetTimer = () => {
  if (timerInterval.value) {
    clearInterval(timerInterval.value);
    timerInterval.value = null;
  }
  isRunning.value = false;
  timeLeft.value = props.duration;
  hasReminder.value = false;
  showReminder.value = false;
};

// 屏幕常亮管理
let wakeLock: any = null;

// 请求屏幕常亮
const requestWakeLock = async () => {
  try {
    // 检查是否支持 Wake Lock API
    if ('wakeLock' in navigator) {
      wakeLock = await (navigator as any).wakeLock.request('screen');
      console.log('屏幕常亮已启用');
    } else {
      console.log('设备不支持 Wake Lock API');
    }
  } catch (error) {
    console.error('无法启用屏幕常亮:', error);
  }
};

// 释放屏幕常亮
const releaseWakeLock = () => {
  if (wakeLock) {
    wakeLock.release();
    wakeLock = null;
    console.log('屏幕常亮已释放');
  }
};

// 监听页面可见性变化
const handleVisibilityChange = () => {
  if (document.visibilityState === 'visible' && isRunning.value) {
    requestWakeLock();
  }
};

// 开始倒计时时启用屏幕常亮
const startTimerWithWakeLock = () => {
  toggleTimer();
  if (isRunning.value) {
    requestWakeLock();
  }
};

// 组件挂载时设置事件监听
onMounted(() => {
  document.addEventListener('visibilitychange', handleVisibilityChange);
});

// 组件卸载时清理
onUnmounted(() => {
  if (timerInterval.value) {
    clearInterval(timerInterval.value);
  }
  releaseWakeLock();
  document.removeEventListener('visibilitychange', handleVisibilityChange);
});
</script>

<template>
  <div class="timer-container">
    <h1>{{ title }}</h1>
    
    <div class="timer-display">
      {{ formatTime(timeLeft) }}
    </div>
    
    <div class="button-group">
      <button 
        class="control-button" 
        :class="{ running: isRunning }"
        @click="startTimerWithWakeLock"
      >
        {{ isRunning ? '暂停' : '开始' }}
      </button>
      <button 
        class="control-button reset"
        @click="resetTimer"
      >
        重置
      </button>
    </div>
    
    <!-- 提醒弹窗 -->
    <div v-if="showReminder" class="reminder">
      {{ reminderMessage }}
    </div>
  </div>
</template>

<style scoped>
.timer-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 100%;
  box-sizing: border-box;
  background-color: #f5f5f5;
  padding: 20px;
}

h1 {
  font-size: 2.5rem;
  color: #333;
  margin-bottom: 40px;
  font-weight: 600;
}

.timer-display {
  font-size: 5rem;
  font-weight: bold;
  color: #2c3e50;
  background-color: white;
  padding: 30px 50px;
  border-radius: 15px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
  margin-bottom: 40px;
  font-family: 'Courier New', monospace;
  min-width: 350px;
  text-align: center;
}

.button-group {
  display: flex;
  gap: 20px;
  margin-bottom: 30px;
}

.control-button {
  padding: 15px 30px;
  font-size: 1.2rem;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.control-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.control-button.running {
  background-color: #e74c3c;
  color: white;
}

.control-button.running:hover {
  background-color: #c0392b;
}

.control-button:not(.running) {
  background-color: #3498db;
  color: white;
}

.control-button:not(.running):hover {
  background-color: #2980b9;
}

.control-button.reset {
  background-color: #95a5a6;
  color: white;
}

.control-button.reset:hover {
  background-color: #7f8c8d;
}

.reminder {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 40px 60px;
  border-radius: 15px;
  font-size: 2rem;
  font-weight: bold;
  z-index: 1000;
  animation: pulse 1s infinite;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3);
}

@keyframes pulse {
  0% {
    transform: translate(-50%, -50%) scale(1);
  }
  50% {
    transform: translate(-50%, -50%) scale(1.05);
  }
  100% {
    transform: translate(-50%, -50%) scale(1);
  }
}

/* 响应式设计 */
/* 平板设备 */
@media (max-width: 1024px) {
  h1 {
    font-size: 2.2rem;
  }
  
  .timer-display {
    font-size: 4rem;
    padding: 25px 40px;
    min-width: 320px;
  }
  
  .button-group {
    gap: 15px;
  }
  
  .control-button {
    padding: 14px 28px;
    font-size: 1.1rem;
  }
  
  .reminder {
    padding: 35px 50px;
    font-size: 1.8rem;
  }
}

/* 手机设备 - 横向 */
@media (max-width: 768px) {
  h1 {
    font-size: 1.8rem;
  }
  
  .timer-display {
    font-size: 3rem;
    padding: 20px 30px;
    min-width: 280px;
  }
  
  .button-group {
    flex-direction: column;
    gap: 10px;
  }
  
  .control-button {
    padding: 12px 24px;
    font-size: 1rem;
  }
  
  .reminder {
    padding: 30px 40px;
    font-size: 1.5rem;
  }
}

/* 手机设备 - 纵向小屏幕 */
@media (max-width: 480px) {
  h1 {
    font-size: 1.5rem;
    margin-bottom: 30px;
  }
  
  .timer-display {
    font-size: 2.5rem;
    padding: 15px 25px;
    min-width: 220px;
    margin-bottom: 30px;
  }
  
  .button-group {
    flex-direction: column;
    gap: 8px;
    width: 100%;
    max-width: 200px;
  }
  
  .control-button {
    padding: 10px 20px;
    font-size: 0.9rem;
    width: 100%;
  }
  
  .reminder {
    padding: 20px 30px;
    font-size: 1.2rem;
    text-align: center;
  }
}

/* 确保在触摸设备上有良好的点击体验 */
@media (hover: none) and (pointer: coarse) {
  .control-button {
    min-height: 48px;
    touch-action: manipulation;
  }
  
  /* 增大触摸目标 */
  .button-group {
    gap: 16px;
  }
}
</style>