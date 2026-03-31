<template>
  <div class="message" :style="{ backgroundColor: messageColor }">
    <div class="message-header">
      <div class="message-info">
        <span class="message-index">#{{ message.id }}</span>
        <strong>{{ message.author }}</strong>
      </div>
      <span class="message-time">{{ message.timestamp }}</span>
    </div>
    <div class="message-content">
      {{ message.text }}
    </div>
    <div v-if="message.imageHeight" class="message-image-wrapper">
      <div
        class="message-image"
        :style="{ height: (message.imageHeight + expandHeight) + 'px', backgroundColor: imageColor }"
      >
        <div class="image-placeholder">
          📷 {{ message.imageHeight + expandHeight }}px
        </div>
      </div>
    </div>
    <div v-if="expandHeight > 0" class="message-extra">
      <div class="extra-content" :style="{ height: expandHeight + 'px', backgroundColor: 'rgba(33, 150, 243, 0.1)' }">
        Extra content loaded...
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'MessageComp',
  props: {
    message: {
      type: Object,
      required: true,
    }
  },
  data() {
    return {
      expandHeight: 0,
    };
  },
  computed: {
    messageColor() {
      // Alternate colors based on author
      return this.message.author === 'You' ? '#e3f2fd' : '#f5f5f5';
    },
    imageColor() {
      const colors = ['#ffcdd2', '#c8e6c9', '#bbdefb', '#fff9c4', '#f0f4c3', '#ffe0b2'];
      // Generate consistent color based on message id
      return colors[this.message.id % colors.length];
    }
  },
  mounted() {
    // Measure initial height and emit it
    this.$nextTick(() => {
      this.measureAndEmit();
    });
    const random = Math.floor(Math.random() * 2);
    console.log({random, id: this.message.id})
    if (random) {
        this.expandTimer = setTimeout(() => {
            this.expandHeight = Math.floor(Math.random() * (200 - 100 + 1)) + 200;
            // Measure new height and emit it
            this.$nextTick(() => {
                this.measureAndEmit();
            });
        }, 200);
    }
    // After 500ms, increase height by 200-300px to test dynamic height changes
   
  },
  methods: {
    measureAndEmit() {
      // Get the actual rendered height of this component
      const el = this.$el;
      if (el) {
        const actualHeight = el.offsetHeight;
        // Emit the actual measured height
        this.$emit('height-change', {
          id: this.message.id,
          actualHeight: actualHeight
        });
      }
    }
  },
  beforeDestroy() {
    if (this.expandTimer) {
      clearTimeout(this.expandTimer);
    }
  }
};
</script>

<style scoped>
.message {
  padding: 12px 16px;
  border-radius: 8px;
  border-left: 4px solid #2196f3;
}

.vs-item {
  padding: 8px 0; /* thay margin */
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
  font-size: 14px;
}

.message-info {
  display: flex;
  gap: 8px;
  align-items: center;
}

.message-index {
  color: #999;
  font-size: 12px;
  font-weight: 500;
  padding: 2px 6px;
  background: rgba(0, 0, 0, 0.05);
  border-radius: 3px;
  min-width: 35px;
  text-align: center;
}

.message-header strong {
  color: #1976d2;
  font-weight: 600;
}

.message-time {
  color: #999;
  font-size: 12px;
}

.message-content {
  color: #333;
  line-height: 1.5;
  font-size: 14px;
  margin-bottom: 8px;
}

.message-image-wrapper {
  margin-top: 8px;
}

.message-image {
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
  overflow: hidden;
}

.image-placeholder {
  font-size: 12px;
  color: #666;
  font-weight: 500;
}

.message-extra {
  margin-top: 8px;
}

.extra-content {
  border-radius: 4px;
  padding: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  color: #2196f3;
  font-weight: 500;
  animation: slideIn 0.3s ease-in-out;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>
