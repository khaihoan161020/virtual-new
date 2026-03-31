<template>
  <div class="vs-container" ref="container" @scroll="onScroll">
    <!-- Loading indicator at top -->
    <div v-if="isLoading" class="vs-loading">
      <div class="loading-spinner">Loading more messages...</div>
    </div>

    <!-- Scroll to bottom button -->
    <div v-if="showScrollToBottom" class="vs-scroll-to-bottom">
      <button @click="onClickScrollToBottom" class="scroll-btn">
        ↓ Scroll to Bottom
      </button>
    </div>

    <div class="vs-spacer" :style="{ height: totalHeight + 'px' }"></div>

    <div
      class="vs-translate"
      :style="{ transform: `translateY(${computedOffsetY}px)` }"
    >
      <div
        v-for="node in visibleNodes"
        :key="node.key"
        class="vs-item"
        :data-key="node.key"
        :ref="setItemRef"
        @height-change="onItemHeightChange"
      >
        <slot :item="node.item" :index="node.index" />
      </div>
    </div>
  </div>
</template>

<script>
const ESTIMATED_HEIGHT = 60;

export default {
  name: 'VirtualScroll',

  props: {
    items: Array,
    itemKey: { type: String, default: 'id' },
    buffer: { type: Number, default: 50 }
  },

  data() {
    return {
      heights: {},
      measured: new Set(),

      startIndex: 0,
      endIndex: 0,

      scrollTop: 0,
      viewportHeight: 0,

      resizeObserver: null,
      itemRefs: new Map(),

      isReady: false,
      stickyBottom: true,
      showScrollToBottom: false,
      resizeTicking: false,
      scrollTicking: false,
      isLoading: false,
      scrollToBottomTimer: null
    };
  },

  computed: {
    // Calculate total height of all items
    totalHeight() {
      let sum = 0;
      for (let i = 0; i < this.items.length; i++) {
        const key = this.getKey(this.items[i]);
        sum += this.heights[key] || ESTIMATED_HEIGHT;
      }
      return sum;
    },

    // Get visible items with buffer
    visibleNodes() {
      if (!this.items.length) return [];

      const bufferedStart = Math.max(0, this.startIndex - this.buffer);
      const bufferedEnd = Math.min(this.items.length, this.endIndex + this.buffer);

      let top = 0;
      let nodes = [];

      // Calculate offset from top
      for (let i = 0; i < bufferedStart; i++) {
        const key = this.getKey(this.items[i]);
        top += this.heights[key] || ESTIMATED_HEIGHT;
      }

      // Render visible items
      for (let i = bufferedStart; i < bufferedEnd; i++) {
        const item = this.items[i];
        const key = this.getKey(item);
        const height = this.heights[key] || ESTIMATED_HEIGHT;

        nodes.push({
          item,
          index: i,
          key,
          top
        });

        top += height;
      }

      return nodes;
    },

    // Calculate offsetY separately to avoid side effects in computed property
    computedOffsetY() {
      const nodes = this.visibleNodes;
      return nodes.length > 0 ? nodes[0].top : 0;
    }
  },

  mounted() {
    const el = this.$refs.container;
    this.viewportHeight = el.clientHeight;

    this.setupResizeObserver();
    this.initMeasure();
  },

  watch: {
    items: {
      handler(newVal, oldVal) {
        this.handleItemsChange(newVal, oldVal);
      },
      deep: false
    }
  },

  methods: {
    getKey(item) {
      return item[this.itemKey];
    },

    // ===== INIT MEASURE =====
    async initMeasure() {
      // Start showing last items (chat bottom)
      this.startIndex = Math.max(0, this.items.length - 20);
      this.endIndex = this.items.length;

      await this.$nextTick();

      this.measureVisible();

      await this.$nextTick();

      this.scrollToBottom(true);

      this.isReady = true;
    },

    measureVisible() {
      this.itemRefs.forEach((el, key) => {
        const h = el.offsetHeight;
        if (h > 0 && this.heights[key] !== h) {
          this.heights[key] = h;
          this.measured.add(key);
        }
      });
    },

    // ===== ResizeObserver =====
    setupResizeObserver() {
      this.resizeObserver = new ResizeObserver((entries) => {
        if (this.resizeTicking) return;
        this.resizeTicking = true;

        requestAnimationFrame(() => {
          let changed = false;

          entries.forEach((entry) => {
            const key = entry.target.dataset.key;
            const h = entry.contentRect.height;

            if (h > 0 && this.heights[key] !== h) {
              this.heights[key] = h;
              changed = true;
            }
          });

          // If heights changed and we're near bottom, auto-scroll
          if (changed && this.stickyBottom) {
            this.scrollToBottom();
          }

          this.resizeTicking = false;
        });
      });
    },

    setItemRef(el) {
      if (!el) return;

      const key = el.dataset.key;
      this.itemRefs.set(key, el);

      if (this.resizeObserver && this.isReady) {
        this.resizeObserver.observe(el);
      }
    },

    // Handle height change event from child component
    onItemHeightChange(payload) {
      const { id, actualHeight } = payload;
      const key = id;

      // Update height cache with actual measured height
      if (actualHeight && actualHeight > 0) {
        const oldHeight = this.heights[key];
        this.heights[key] = actualHeight;

        // Only trigger scroll if height actually changed
        if (oldHeight !== actualHeight && this.stickyBottom) {
          this.scrollToBottom();
        }
      }
    },

    // ===== SCROLL =====
    onScroll() {
      if (this.scrollTicking) return;
      this.scrollTicking = true;

      requestAnimationFrame(() => {
        const el = this.$refs.container;
        this.scrollTop = el.scrollTop;

        this.stickyBottom = this.isNearBottom();
        
        // Show scroll to bottom button if not near bottom
        this.showScrollToBottom = !this.stickyBottom;

        this.calculateVisibleRange();

        // Check if near top and emit event to load more
        this.checkAndEmitNearTop();

        // Check if near bottom and emit event to load more
        this.checkAndEmitNearBottom();

        this.scrollTicking = false;
      });
    },

    // Check if user scrolled near top, emit event to load more items
    checkAndEmitNearTop() {
      // Trigger load more when user is within top 10-20 items offset
      const itemThreshold = 10;
      
      if (this.startIndex <= itemThreshold && this.startIndex > 0) {
        // Emit event to load more
        this.$emit('scroll-top', {
          topIndex: this.startIndex,
          message: 'Load more items above'
        });
      }
    },

    // Check if user scrolled near bottom, emit event to load more items
    checkAndEmitNearBottom() {
      // Trigger load more when user is within bottom 20 items offset
      const itemThreshold = 20;
      
      if (this.endIndex >= this.items.length - itemThreshold && this.endIndex < this.items.length) {
        // Emit event to load more
        // this.$emit('scroll-bottom', {
        //   bottomIndex: this.endIndex,
        //   message: 'Load more items below'
        // });
      }
    },

    // Calculate which items should be visible based on scroll position
    calculateVisibleRange() {
      if (!this.items.length) return;

      const scrollTop = this.scrollTop;
      const scrollBottom = scrollTop + this.viewportHeight;

      let offset = 0;
      let startFound = false;

      for (let i = 0; i < this.items.length; i++) {
        const key = this.getKey(this.items[i]);
        const h = this.heights[key] || ESTIMATED_HEIGHT;
        const itemBottom = offset + h;

        if (!startFound && itemBottom > scrollTop) {
          this.startIndex = i;
          startFound = true;
        }

        if (startFound && offset >= scrollBottom) {
          this.endIndex = i;
          return;
        }

        offset += h;
      }

      // Fallback: if we reached the end
      if (startFound) {
        this.endIndex = this.items.length;
      }
    },

    isNearBottom() {
      const el = this.$refs.container;
      const distance = el.scrollHeight - (el.scrollTop + el.clientHeight);
      return distance < 200;
    },

    // Handle scroll to bottom button click
    onClickScrollToBottom() {
      // Clear any pending scroll operations
      if (this.scrollToBottomTimer) {
        clearTimeout(this.scrollToBottomTimer);
      }

      this.stickyBottom = true;
      this.showScrollToBottom = false;
      const el = this.$refs.container;
      
      // Scroll to bottom multiple times to ensure it works
      // This helps when content is still loading or rendering
      const scrollToBottomMultipleTimes = (times = 0) => {
        if (times >= 3) return; // Stop after 3 attempts
        
        this.$nextTick(() => {
          this.scrollToBottomTimer = setTimeout(() => {
            el.scrollTop = el.scrollHeight;
            // Try again after a delay
            scrollToBottomMultipleTimes(times + 1);
          }, 100);
        });
      };

      // Start scrolling to bottom
      scrollToBottomMultipleTimes(0);
    },

    scrollToBottom(force = false) {
      const el = this.$refs.container;

      // Always scroll if force is true, otherwise check stickyBottom
      if (!force && !this.stickyBottom) return;

      this.$nextTick(() => {
        // Use setTimeout to ensure DOM is fully updated
        setTimeout(() => {
          el.scrollTop = el.scrollHeight;
        }, 50);
      });
    },

    // Maintain scroll position when items are prepended
    // Keep viewing the same item (e.g., if viewing #12, stay on #12 after load more)
    prependItems(newItemsCount) {
      const el = this.$refs.container;
      
      // Step 1: Record current scroll position
      const scrollTopBefore = el.scrollTop;
      
      this.$nextTick(() => {
        setTimeout(() => {
          // Step 2: Measure all newly prepended items (indices 0 to newItemsCount-1)
          let totalPrependedHeight = 0;
          const allElements = Array.from(el.querySelectorAll('[data-key]'));
          
          for (let i = 0; i < newItemsCount; i++) {
            const key = this.getKey(this.items[i]);
            const itemEl = allElements.find(el => el.dataset.key == key);
            
            if (itemEl && itemEl.offsetHeight > 0) {
              // Store measured height
              this.heights[key] = itemEl.offsetHeight;
              totalPrependedHeight += itemEl.offsetHeight;
            } else {
              // Use estimated height if not measured
              totalPrependedHeight += ESTIMATED_HEIGHT;
            }
          }
          
          // Step 3: Keep same item in view
          // Item #12 was at scrollTop 500px
          // After adding 50 items above (3000px total), it's now at 500 + 3000 = 3500px
          // So we scroll to that new position
          const newScrollTop = scrollTopBefore + totalPrependedHeight;
          el.scrollTop = newScrollTop;
        }, 100);
      });
    },

    // Jump to a specific message by ID
    jumpToMessageById(messageId) {
      // Find the index of the message with the given ID
      const index = this.items.findIndex(item => this.getKey(item) === messageId);
      
      if (index === -1) {
        console.warn(`Message with ID ${messageId} not found`);
        return;
      }

      const el = this.$refs.container;
      
      // First, force render the target item and surrounding items to measure heights
      this.startIndex = Math.max(0, index - 5);
      this.endIndex = Math.min(this.items.length, index + 5);
      
      // Function to calculate and perform jump with current heights
      const performJump = () => {
        // Recalculate scroll position with current heights each time
        let scrollTop = 0;
        for (let i = 0; i < index; i++) {
          const key = this.getKey(this.items[i]);
          scrollTop += this.heights[key] || ESTIMATED_HEIGHT;
        }
        
        el.scrollTop = scrollTop;
        this.stickyBottom = false;
        this.calculateVisibleRange();
      };

      // Wait for DOM to render target items, then measure and jump
      this.$nextTick(() => {
        // Measure all visible items
        this.measureVisible();
        
        performJump();
        
        requestAnimationFrame(() => {
          performJump();
          requestAnimationFrame(() => {
            performJump();
          });
        });
      });
    },

    // ===== DATA CHANGE =====
    handleItemsChange(newVal, oldVal) {
      if (!oldVal || oldVal.length === 0) {
        // Initial load
        this.initMeasure();
        return;
      }

      const oldLength = oldVal.length;
      const newLength = newVal.length;

      if (newLength > oldLength) {
        // Items appended (new messages)
        this.$nextTick(() => {
          this.measureVisible();

          if (this.stickyBottom) {
            this.scrollToBottom();
          }
        });
      } else if (newLength < oldLength) {
        // Items removed
        this.calculateVisibleRange();
      } else {
        // Items replaced
        this.$nextTick(() => {
          this.measureVisible();
        });
      }
    },

    // Public method to control loading state
    setLoading(loading) {
      this.isLoading = loading;
    }
  },

  beforeDestroy() {
    if (this.resizeObserver) {
      this.resizeObserver.disconnect();
    }
    if (this.scrollToBottomTimer) {
      clearTimeout(this.scrollToBottomTimer);
    }
    this.itemRefs.clear();
  }
};
</script>

<style scoped>
.vs-container {
  overflow-y: auto;
  height: 500px;
  position: relative;
}

.vs-loading {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 999;
  background: white;
  padding: 20px 40px;
  border-radius: 8px;
  box-shadow: 0 2px 12px rgba(0, 0, 0, 0.15);
}

.loading-spinner {
  font-size: 14px;
  color: #2196f3;
  font-weight: 500;
  text-align: center;
  animation: pulse 1.5s ease-in-out infinite;
}

.vs-scroll-to-bottom {
  position: fixed;
  bottom: 30px;
  right: 30px;
  z-index: 100;
  animation: slideUp 0.3s ease-out;
}

.scroll-btn {
  background: #2196f3;
  color: white;
  border: none;
  padding: 12px 20px;
  border-radius: 24px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  box-shadow: 0 4px 12px rgba(33, 150, 243, 0.3);
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  gap: 6px;
}

.scroll-btn:hover {
  background: #1976d2;
  box-shadow: 0 6px 16px rgba(33, 150, 243, 0.4);
  transform: translateY(-2px);
}

.scroll-btn:active {
  transform: translateY(0);
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.vs-spacer {
  width: 100%;
}

.vs-translate {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  will-change: transform;
}

.vs-item {
  width: 100%;
  contain: layout paint style;
}
</style>