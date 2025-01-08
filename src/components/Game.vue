<template>
  <div class="game-container">

    <h1>{{ t("title") }}</h1>
    <div v-if="isGameWon" class="win-message">🎉 {{ t("winMessage") }} 🎉</div>
    <!-- Language toggle -->
    <div class="language-toggle">
      <button
        v-for="lang in Object.keys(messages)"
        :key="lang"
        :class="{ active: language === lang }"
        @click="setLanguage(lang)"
      >
        {{ lang.toUpperCase() }}
      </button>
    </div>


    <!-- Mute toggle button -->
    <button class="mute-toggle" @click="toggleMute">
      {{ isMuted ? "🔇 Mute" : "🔊 Unmute" }}
    </button>


    <div class="grid-container" :style="gridStyle">
      <div
        v-for="(tile, index) in tiles"
        :key="index"
        class="grid-item"
        :class="{ empty: tile === null }"
        @click="handleTileClick(index)"
      >
        {{ tile }}
      </div>
    </div>
    <p class="instructions">{{ t('howToPlay') }}</p> <!-- Updated instructions -->
    <button class="restart-button" @click="restartGame">
      {{ t("restart") }}
    </button>
    <p>@deven.tw</p>
  </div>
</template>
<script lang="ts">
import { ref, computed, onMounted } from "vue";
import ConfettiExplosion from "vue-confetti-explosion";

// Define the structure of the messages object
type Messages = {
    [lang: string]: {
      title: string;
      howToPlay: string;
      winMessage: string;
      restart: string;
    };
  };
  
  const getGridSizeFromURL = (): number => {
  const params = new URLSearchParams(window.location.search);
  const size = parseInt(params.get("size") || "3", 10); // Default to 3 if size is not provided
  return Math.min(Math.max(size, 3), 30); // Ensure size is between 3 and 10
};

export default {

  
  setup() {
    const gridSize = ref(getGridSizeFromURL()); // Get grid size from URL or default to 3; // Default grid size (4x4)
    const totalTiles = computed(() => gridSize.value * gridSize.value); // Total tiles (gridSize²)

    // Reactive state for the tiles (1 to totalTiles - 1 + null for the empty space)
    const tiles = ref<(number | null)[]>([]);
    const language = ref(localStorage.getItem("language") || "en"); // Persist language in localStorage
    const isMuted = ref(false); // Mute state

    const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    let audioBuffer: AudioBuffer | null = null;

    // Preload the sound effect into an AudioBuffer
    const loadSound = async () => {
      const response = await fetch(
        "/src/assets/audioblocks-realistic-whoosh-6-fight-kung-fu-ninja-fight-kung-fu-ninja_rt2gZXzUAPL_NWM.mp3"
      );
      const arrayBuffer = await response.arrayBuffer();
      audioBuffer = await audioContext.decodeAudioData(arrayBuffer);
    };

       // Play the preloaded sound
  const playMoveSound = (): void => {
    if (isMuted.value || !audioBuffer) return; // If the sound is not loaded yet, do nothing
      const source = audioContext.createBufferSource();
      source.buffer = audioBuffer;
      source.connect(audioContext.destination);
      source.start(0); 
    };


    const toggleMute = (): void => {
      isMuted.value = !isMuted.value;
    };

    // Translations for both languages
    // Translations for both languages
    const messages: Messages = {
      en: {
        title: "Number Klotski Puzzle Game",
   howToPlay: "Click any tile adjacent to the empty space to move it. Arrange the tiles in ascending order, with the empty space ending in the bottom-right corner.",
        winMessage: "You Win!",
        restart: "Restart",
      },
      zh: {
        title: "數字華容道遊戲",
        winMessage: "你贏了！",
        restart: "重新開始",
        howToPlay: "點擊空格旁邊的方塊移動它。將方塊按升序排列，空格需要最終位於右下角。",
      },
    };

    // Function to get the translated text
    const t = (key: keyof Messages["en"]): string =>
      messages[language.value][key];

    // Set the selected language and persist it
    const setLanguage = (lang: string): void => {
      language.value = lang;
      localStorage.setItem("language", lang);
    };

    // Initialize the grid with shuffled and solvable tiles
    const initializeGrid = (): void => {
      const numbers: (number | null)[] = Array.from(
        { length: totalTiles.value - 1 },
        (_, i) => i + 1
      ); // Numbers 1 to totalTiles - 1
      numbers.push(null); // Add the empty space (null)

      do {
        shuffleArray(numbers); // Shuffle the numbers
      } while (!isSolvable(numbers)); // Ensure the puzzle is solvable

      tiles.value = numbers;
    };

    // Shuffle an array
    const shuffleArray = (array: (number | null)[]): void => {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
    };

    // Check if the puzzle is solvable
    const isSolvable = (numbers: (number | null)[]): boolean => {
      let inversions = 0;
      const tilesWithoutNull = numbers.filter(
        (num): num is number => num !== null
      ); // Exclude null
      for (let i = 0; i < tilesWithoutNull.length; i++) {
        for (let j = i + 1; j < tilesWithoutNull.length; j++) {
          if (tilesWithoutNull[i] > tilesWithoutNull[j]) inversions++;
        }
      }

      const emptyRow = Math.floor(numbers.indexOf(null) / gridSize.value);
      if (gridSize.value % 2 === 0) {
        // Even grid
        return (inversions + emptyRow) % 2 === 0;
      } else {
        // Odd grid
        return inversions % 2 === 0;
      }
    };

    // Check if two tiles are adjacent
    const checkAdjacency = (index1: number, index2: number): boolean => {
      const row1 = Math.floor(index1 / gridSize.value);
      const col1 = index1 % gridSize.value;
      const row2 = Math.floor(index2 / gridSize.value);
      const col2 = index2 % gridSize.value;

      return (
        (row1 === row2 && Math.abs(col1 - col2) === 1) || // Horizontal adjacency
        (col1 === col2 && Math.abs(row1 - row2) === 1) // Vertical adjacency
      );
    };

    // Swap two tiles
    const swapTiles = (index1: number, index2: number): void => {
      const temp = tiles.value[index1];
      tiles.value[index1] = tiles.value[index2];
      tiles.value[index2] = temp;
    };

    // Trigger haptic feedback
    const triggerHapticFeedback = (): void => {
      if (navigator.vibrate) {
        navigator.vibrate(50); // Vibrate for 50ms
      }
    };

    // Handle tile click
    const handleTileClick = (index: number): void => {
      const emptyIndex = tiles.value.indexOf(null); // Find the empty space
      if (checkAdjacency(index, emptyIndex)) {
        swapTiles(index, emptyIndex);
        playMoveSound();
        triggerHapticFeedback(); // Add haptic feedback on valid move
       // Reset playback time and play the sound
      }
    };

    // Restart the game
    const restartGame = (): void => {
      initializeGrid();
      triggerHapticFeedback(); // Add haptic feedback on restart
    };

    // Check if the player has won
    const isGameWon = computed((): boolean => {
      // Check if the current grid is in ascending order with null at the end
      return tiles.value.every(
        (tile, index) => tile === index + 1 || tile === null
      );
    });

    // Dynamically calculate CSS grid style based on gridSize
    const gridStyle = computed(() => ({
      display: "grid",
      gridTemplateColumns: `repeat(${gridSize.value}, 1fr)`,
      gap: "5px",
    }));

    // Initialize the grid on mount
    onMounted(() => {
      initializeGrid();
      loadSound(); // Preload the sound on mount
    });

    return {
      gridSize,
      tiles,
      isGameWon,
      handleTileClick,
      restartGame,
      gridStyle,
      t,
      setLanguage,
      messages,
      language,
      isMuted, // Expose mute state
      toggleMute, // Expose mute toggle function
    };
  },
};
</script>

<style scoped>
/* General styling */
h1 {
  line-height: normal;
  font-size: 1.25rem;
  color: #ffffff;
}

/* Language toggle */
.language-toggle {
  margin-bottom: 0.5rem;
}

.language-toggle button {
  padding: 5px 10px;
  font-size: 1rem;
  font-weight: bold;
  margin: 0 5px;
  border: none;
  cursor: pointer;
  border-radius: 5px;
  background-color: #f4f4f4;
  color: #333;
}

.language-toggle button.active {
  background-color: #1a1a1a;
  color: #fff;
}

.language-toggle button:hover {
  background-color: #1a1a1a50;
  color: #fff;
}

/* Game container */
.game-container {
  /* max-width: 480px; */
  padding: 1rem;
  text-align: center;
  background: #00000050;
  width: 100%;
  display: flex;
  flex-direction: column;
  width: 80%;
  border-radius: 1rem;
  margin: 0.5rem;
  
}

/* Win message */
.win-message {
  font-size: 1.5rem;
  color: #28a745;
  margin-bottom: 20px;
}

.grid-container {
  overflow: auto;
}

/* Grid items */
.grid-item {
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #fff;
  border: 2px solid #ccc;
  border-radius: 5px;
  font-size: 1.5rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.2s;
  aspect-ratio: 1;
  color: #1a1a1a;
  user-select: none;
  -user-select: none;
  -webkit-user-select: none; /* Chrome/Safari */        
  -moz-user-select: none; /* Firefox */
  -ms-user-select: none; /* IE10+ */
}

.grid-item.empty {
  background-color: #f4f4f4;
  border: 2px dashed #bbb;
  cursor: default;
}

.grid-item:not(.empty):hover {
  background-color: #f0f0f0;
  animation: scaleUp 0.3s ease-out forwards;
}

@keyframes scaleUp {
  0% {
    transform: scale(1);
  }
  100% {
    transform: scale(1.1);
  }
}

/* Restart button */
.restart-button {
  /* margin-top: ; */
  padding: 10px 20px;
  font-size: 1rem;
  font-weight: bold;
  color: #fff;
  background-color: #1a1a1a;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.restart-button:hover {
  background-color: #1a1a1a50;
}

/* Instructions paragraph */
.instructions {
  font-size: 0.75rem;
  line-height: normal;
  color: #cccccc;
}

.mute-toggle {
  margin: 10px 0;
  padding: 10px 20px;
  font-size: 1rem;
  font-weight: bold;
  color: #fff;
  background-color: #1a1a1a;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.mute-toggle:hover {
  background-color: #1a1a1a50;
}
</style>
