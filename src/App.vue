<template>
  <div id="app">
    <div class="chat-container">
      <div class="chat-header">
        <h1>Virtual Scroll Chat Demo</h1>
        <div class="controls">
          <button @click="addMessage">Add Message</button>
          <button @click="clearMessages">Clear All</button>
          <div class="jump-to-input">
            <input 
              v-model.number="jumpMessageId" 
              type="number" 
              placeholder="Enter message ID to jump"
              @keyup.enter="handleJumpToMessage"
            />
            <button @click="handleJumpToMessage">Jump to ID</button>
          </div>
          <span class="message-count">{{ messages.length }} messages</span>
        </div>
      </div>
      <VirtualScroll
        ref="virtualScroll"
        :items="messages"
        itemKey="id"
        :buffer="40"
        class="chat-list"
        @scroll-top="onScrollTop"
        @scroll-bottom="onScrollBottom"
      >
        <template slot-scope="{ item }">
          <Message :message="item" />
        </template>
      </VirtualScroll>
    </div>
  </div>
</template>

<script>
import VirtualScroll from './components/VirtualScroll.vue'
import Message from './components/Message.vue'

const AUTHORS = ['You', 'Alice', 'Bob', 'Charlie', 'Diana']

const SAMPLE_TEXTS = [
  'Hey, how are you?',
  'I am doing great, thanks for asking!',
  'This is a short message.',
  'This is a much longer message that will take up more space and test the dynamic height calculation of the virtual scroll component. It has multiple lines and should wrap properly.',
  'Another short one.',
  'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.',
  'Quick reply',
  'How about we meet tomorrow? I have some time in the afternoon.',
  'That sounds great! What time works best for you?',
  'Let me check my schedule and get back to you.',
  'Sure, no problem. Take your time!'
]

export default {
  name: 'App',
  components: {
    VirtualScroll,
    Message
  },
  data() {
    return {
      messages: [],
      messageId: 0,
      jumpMessageId: null
    }
  },
  mounted() {
    // Generate initial messages
    this.generateInitialMessages()
  },
  methods: {
    generateInitialMessages() {
      for (let i = 0; i < 50; i++) {
        this.messages.push(this.createMessage())
      }
    },
    createMessage() {
      const author = AUTHORS[Math.floor(Math.random() * AUTHORS.length)]
      const text = SAMPLE_TEXTS[Math.floor(Math.random() * SAMPLE_TEXTS.length)]
      const now = new Date()
      const timestamp = `${now.getHours()}:${String(now.getMinutes()).padStart(2, '0')}`

      // Randomly add image (30% chance)
      const hasImage = Math.random() < 0.3
      const imageHeight = hasImage ? Math.floor(Math.random() * (500 - 300 + 1)) + 300 : null

      return {
        id: this.messageId++,
        author,
        text,
        timestamp,
        imageHeight
      }
    },
    addMessage() {
      this.messages.push(this.createMessage())
      // VirtualScroll will auto-scroll to bottom if near bottom
    },
    clearMessages() {
      this.messages = []
      this.messageId = 0
    },
    onScrollBottom() {
      // Simulate auto-loading new messages when scrolling to bottom
      setTimeout(() => {
        for (let i = 0; i < 20; i++) {
          this.messages.push(this.createMessage())
        }
      }, 500);
    },
    onScrollTop() {
      // Prevent multiple concurrent loads
      if (this.$refs.virtualScroll.isLoading) return;

      // Show loading indicator
      this.$refs.virtualScroll.setLoading(true);

      // Simulate network delay
      setTimeout(() => {
        // Load 50 more messages at the top
        const newMessages = [];
        for (let i = 0; i < 50; i++) {
          newMessages.push(this.createMessage());
        }
        // Prepend to beginning
        this.messages.unshift(...newMessages);

        // Notify VirtualScroll to adjust scroll position
        this.$nextTick(() => {
          this.$refs.virtualScroll.prependItems(50);
          this.$refs.virtualScroll.setLoading(false);
        });
      }, 800);
    },
    handleJumpToMessage() {
      if (this.jumpMessageId === null || this.jumpMessageId === '') {
        alert('Please enter a message ID');
        return;
      }
      this.$refs.virtualScroll.jumpToMessageById(this.jumpMessageId);
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.chat-container {
  display: flex;
  flex-direction: column;
  height: 100%;
  background: #fff;
}

.chat-header {
  padding: 16px;
  border-bottom: 1px solid #e0e0e0;
  background: #f5f5f5;
}

.chat-header h1 {
  margin: 0 0 12px 0;
  font-size: 20px;
  color: #333;
}

.controls {
  display: flex;
  gap: 8px;
  align-items: center;
}

.controls button {
  padding: 8px 16px;
  background: #2196f3;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: background 0.2s;
}

.controls button:hover {
  background: #1976d2;
}

.jump-to-input {
  display: flex;
  gap: 6px;
  align-items: center;
}

.jump-to-input input {
  padding: 6px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
  width: 180px;
}

.jump-to-input input:focus {
  outline: none;
  border-color: #2196f3;
  box-shadow: 0 0 0 2px rgba(33, 150, 243, 0.1);
}

.message-count {
  font-size: 12px;
  color: #666;
  margin-left: auto;
}

.chat-list {
  height: 500px;
  overflow: hidden;
  background: #fafafa;
  padding: 0 16px;
}
</style>
