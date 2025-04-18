<script setup>
import { ref, onMounted, onUnmounted, computed, nextTick, watch } from "vue";
import { useTerminalStore } from "../stores/terminal";
import { availableCommands } from "../data/commands";

const terminalStore = useTerminalStore();
const inputRef = ref(null);

const promptText = computed(() => "visitor@portfolio:~$ ");

const handleKeyDown = (e) => {
  if (e.key === "Enter") {
    e.preventDefault();
    const command = terminalStore.currentCommand.trim();
    terminalStore.handleCommand(command);
    // Emit a custom event that Terminal.vue can listen for
    window.dispatchEvent(new CustomEvent("command-executed"));
  } else if (e.key === "ArrowUp") {
    e.preventDefault();
    terminalStore.navigateHistory("up");

    // After getting a command from history, move cursor to the end
    setTimeout(() => {
      if (inputRef.value) {
        inputRef.value.selectionStart = inputRef.value.value.length;
        inputRef.value.selectionEnd = inputRef.value.value.length;
        terminalStore.updateCursorPosition(inputRef.value.value.length);
      }
    }, 0);
  } else if (e.key === "ArrowDown") {
    e.preventDefault();
    terminalStore.navigateHistory("down");

    // After getting a command from history, move cursor to the end
    setTimeout(() => {
      if (inputRef.value) {
        inputRef.value.selectionStart = inputRef.value.value.length;
        inputRef.value.selectionEnd = inputRef.value.value.length;
        terminalStore.updateCursorPosition(inputRef.value.value.length);
      }
    }, 0);
  } else if (e.key === "ArrowLeft") {
    // Let the default behavior happen first
    nextTick(() => {
      if (inputRef.value) {
        terminalStore.updateCursorPosition(inputRef.value.selectionStart);
      }
    });
  } else if (e.key === "ArrowRight") {
    // Let the default behavior happen first
    nextTick(() => {
      if (inputRef.value) {
        terminalStore.updateCursorPosition(inputRef.value.selectionStart);
      }
    });
  } else if (e.key === "c" && e.ctrlKey) {
    e.preventDefault();
    terminalStore.currentCommand = "";
    terminalStore.cursorPosition = 0;
  } else if (e.key === "r" && e.ctrlKey) {
    e.preventDefault();
    // Start history search - open search mode or search current command
    terminalStore.startHistorySearch();
  } else if (e.key === "l" && e.ctrlKey) {
    e.preventDefault();
    // Clear the screen, same as the clear command
    terminalStore.executeCommand("clear");
  } else if (e.key === "Tab") {
    e.preventDefault();
    terminalStore.handleTabCompletion();
  }
};

const focusInput = () => {
  if (inputRef.value) {
    inputRef.value.focus();
  }
  terminalStore.startCursorBlink();
};

const handleBlur = () => {
  terminalStore.stopCursorBlink();
};

defineExpose({ focusInput });

// Monitor focus changes
watch(
  () => inputRef.value?.matches(":focus"),
  (isFocused) => {
    if (isFocused) {
      terminalStore.startCursorBlink();
    } else {
      terminalStore.stopCursorBlink();
    }
  },
  { immediate: true }
);

onMounted(() => {
  terminalStore.setFocusFunction(focusInput);

  // Focus whenever currentCommand changes
  watch(
    () => terminalStore.currentCommand,
    () => {
      // If the cursor position isn't manually set, default to end of input
      if (
        inputRef.value &&
        inputRef.value.selectionStart === inputRef.value.selectionEnd
      ) {
        nextTick(() => {
          if (inputRef.value) {
            const cursorPos = inputRef.value.selectionStart || 0;
            terminalStore.updateCursorPosition(cursorPos);
          }
        });
      }
    }
  );
});

// Safety cleanup
onUnmounted(() => {
  if (terminalStore.cursorInterval) {
    clearInterval(terminalStore.cursorInterval);
  }
});
</script>

<template>
  <div class="command-prompt">
    <div class="prompt-container">
      <span class="prompt-text">visitor@portfolio:~</span>
      <span class="dollar-sign">$&nbsp;</span>
    </div>
    <div class="input-container">
      <input
        ref="inputRef"
        type="text"
        v-model="terminalStore.currentCommand"
        @keydown="handleKeyDown"
        @focus="focusInput"
        @blur="handleBlur"
        @click="
          terminalStore.updateCursorPosition(inputRef?.selectionStart || 0)
        "
        @input="
          terminalStore.updateCursorPosition(inputRef?.selectionStart || 0)
        "
        @selectionchange="
          terminalStore.updateCursorPosition(inputRef?.selectionStart || 0)
        "
        spellcheck="false"
        autocomplete="off"
        autocorrect="off"
        class="command-input"
      />
      <span
        class="cursor-block"
        v-show="terminalStore.cursorVisible"
        :style="{
          left: terminalStore.cursorPosition + 'ch',
        }"
      ></span>
    </div>
  </div>
</template>

<style>
.command-prompt {
  display: flex;
  margin-top: 10px;
  flex-wrap: wrap;
}

.prompt-container {
  display: flex;
  position: relative;
}

.prompt-text {
  color: #50fa7b;
  user-select: none;
  white-space: nowrap;
}

.dollar-sign {
  color: #f8f8f2;
  user-select: none;
  white-space: nowrap;
  margin-right: 4px;
}

.input-container {
  position: relative;
  flex: 1;
  display: flex;
}

.command-input {
  flex: 1;
  background: transparent;
  border: none;
  color: #f8f8f2;
  font-family: "JetBrains Mono", monospace;
  font-size: 14px;
  outline: none;
  padding: 0;
  caret-color: transparent;
}

.cursor-block {
  position: absolute;
  width: 10px;
  height: 18px;
  background-color: #f8f8f2;
  top: 0;
  animation: blink 1s infinite;
}

@keyframes blink {
  0%,
  49% {
    opacity: 1;
  }
  50%,
  100% {
    opacity: 0;
  }
}
</style>
