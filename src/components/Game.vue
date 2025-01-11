<script lang="ts">
import ConfettiExplosion from "vue-confetti-explosion";
import { Camera, Languages, Volume2, VolumeOff, Plus, Minus } from 'lucide-vue-next';
import { ref, computed, onMounted } from "vue";
import sound from "../assets/audioblocks-realistic-whoosh-6-fight-kung-fu-ninja-fight-kung-fu-ninja_rt2gZXzUAPL_NWM.mp3"
// Define the structure of the messages object
type Messages = {
  [lang: string]: {
    title: string;
    howToPlay: string;
    winMessage: string;
    restart: string;
    difficulty: string; // Add this line
  };
};

const getGridSizeFromURL = (): number => {
  const params = new URLSearchParams(window.location.search);
  const size = parseInt(params.get("size") || "3", 10); // Default to 3 if size is not provided
  return Math.min(Math.max(size, 3), 30); // Ensure size is between 3 and 10
};

export default {

  components: {
    Camera,
    Languages,
    Volume2,
    VolumeOff,
    Plus,
    Minus,
    ConfettiExplosion
  },
  setup() {
    const gridSize = ref(getGridSizeFromURL()); // Get grid size from URL or default to 3; // Default grid size (4x4)
    const totalTiles = computed(() => gridSize.value * gridSize.value); // Total tiles (gridSize²)

    // Reactive state for the tiles (1 to totalTiles - 1 + null for the empty space)
    const tiles = ref<(number | null)[]>([]);
    const language = ref(localStorage.getItem("language") || "en"); // Persist language in localStorage
    const isMuted = ref(localStorage.getItem("isMuted") === "true"); // Retrieve mute state


    const audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
    let audioBuffer: AudioBuffer | null = null;

    // Preload the sound effect into an AudioBuffer
    const loadSound = async () => {
      const response = await fetch(
        sound
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
      localStorage.setItem("isMuted", String(isMuted.value)); // Persist mute state
    };

    // Translations for both languages
    // Translations for both languages
    const messages: Messages = {
      en: {
        title: "Number Klotski Puzzle Game",
        howToPlay: "Click any tile adjacent to the empty space to move it. Arrange the tiles in ascending order, with the empty space ending in the bottom-right corner.",
        winMessage: "You Win!",
        restart: "Restart",
        difficulty: "Difficulty", // Add this line
      },
      zh: {
        title: "數字華容道遊戲",
        winMessage: "你贏了！",
        restart: "重新開始",
        howToPlay: "點擊空格旁邊的方塊移動它。將方塊按升序排列，空格需要最終位於右下角。",
        difficulty: "難度", // Add this line
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


    // Toggle between English and Chinese
    const toggleLanguage = (): void => {
      const newLanguage = language.value === 'en' ? 'zh' : 'en';
      setLanguage(newLanguage); // Update the language state
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

    const increaseSize = (): void => {
      const newSize = Math.min(gridSize.value + 1, 30); // Cap the size at 30
      setGridSize(newSize);
    };

    const decreaseSize = (): void => {
      const newSize = Math.max(gridSize.value - 1, 3); // Ensure size doesn't go below 3
      setGridSize(newSize);
    };

    const setGridSize = (size: number): void => {
      gridSize.value = size;
      updateURL(size); // Update the URL with the new size
      initializeGrid(); // Reinitialize the grid with the new size
    };

    const updateURL = (size: number): void => {
      const params = new URLSearchParams(window.location.search);
      params.set("size", String(size));
      window.history.replaceState({}, '', `${window.location.pathname}?${params}`);
    };

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
      toggleLanguage,
      isMuted, // Expose mute state
      toggleMute, // Expose mute toggle function
      increaseSize,
      decreaseSize
    };
  },
};
</script>

<template>
  <div v-if="isGameWon" class="win-effect">
    <ConfettiExplosion :particleCount="200" :force="0.3" />
  </div>
  <div v-confetti="{ particleCount: 200, force: 0.3 }" />
  <div class="game-container">


    <h1>{{ t("title") }}</h1>

    <div class="menu-row">
      <div class="language-toggle">
        <button class="language-toggle-button" @click="toggleLanguage" aria-label="Toggle Language">
          <Languages color="white" :size="32" class="language-icon" />
        </button>
      </div>
      <div class="language-toggle">
        <button class="language-toggle-button" @click="toggleMute" aria-label="Toggle Language">
          <template v-if="isMuted">
            <VolumeOff color="white" :size="32" class="language-icon" />
          </template>
          <template v-else>
            <Volume2 color="white" :size="32" class="language-icon" />
          </template>

        </button>
      </div>



    </div>

    <div v-if="isGameWon" class="win-message">🎉 {{ t("winMessage") }} 🎉</div>

    <div class="grid-container" :style="gridStyle">
      <div v-for="(tile, index) in tiles" :key="index" class="grid-item" :class="{ empty: tile === null }"
        @click="handleTileClick(index)">
        {{ tile }}
      </div>
    </div>

    <p class="instructions">{{ t('howToPlay') }}</p> <!-- Updated instructions -->

    <div class="bottom-menu-group">
      <div class="bottom-menu-group-item">
        <button class="restart-button" @click="restartGame">
          {{ t("restart") }}
        </button>
      </div>
      <div class="bottom-menu-group-item">
        <span class="box-title" @click="restartGame">
          {{ t("difficulty") }}
        </span>
        <span class="size-control">
          {{ gridSize }}
        </span>
        <div class="size-control-buttons">
          <button class="size-control-button" @click="increaseSize" aria-label="Toggle Language">
            <Plus color="white" :size="32" class="language-icon" />

          </button>

          <button class="size-control-button" @click="decreaseSize" aria-label="Toggle Language">
            <Minus color="white" :size="32" class="language-icon" />

          </button>
        </div>

      </div>

    </div>

    <div>

    </div>

    <p>@deven.tw</p>

  </div>
</template>

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
  display: flex;
  /* Align buttons in a row */
  justify-content: center;
  /* Center the button */
}

.language-toggle-button {
  padding: 10px;
  /* Add some padding for the button */
  border: none;
  border-radius: 50%;
  /* Make the button circular */
  background-color: #1a1a1a;
  /* Background color for the button */
  cursor: pointer;
  /* Change cursor to pointer */
  transition: background-color 0.3s;
  /* Smooth transition for hover effect */
}

.language-toggle-button:hover {
  background-color: #1a1a1a50;
  /* Hover effect */
}

.language-icon {
  display: block;
  /* Ensures the icon is centered */
}

/* Game container */
.game-container {
  max-width: 480px;
  padding: 1rem;
  text-align: center;
  background: #00000050;
  width: 100%;
  display: flex;
  flex-direction: column;
  width: 80%;
  border-radius: 1rem;
  margin: 0.5rem;
  max-height: 90vh;
  z-index: 1;
}

/* Win message */
.win-message {
  font-size: 1.5rem;
  color: #28a745;
  margin-bottom: 20px;
}


.win-effect {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  overflow: hidden;
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
  -webkit-user-select: none;
  /* Chrome/Safari */
  -moz-user-select: none;
  /* Firefox */
  -ms-user-select: none;
  /* IE10+ */
}

.grid-item.empty {
  background-color: #484D51FF;
  border: 2px dashed #595959;
  cursor: default;
}

.grid-item:not(.empty):hover {
  background-color: #BEBBBBFF;
  animation: scaleDown 0.3s ease-out forwards;
}

@keyframes scaleDown {
  0% {
    transform: scale(1);
  }

  100% {
    transform: scale(0.95);
  }
}

.box-title {
  color: #1a1a1a;
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

.menu-row {
  display: flex;
  justify-content: center;
  align-items: center;
  margin-bottom: 0.5rem;
  gap: 0.5rem;
}

.size-control {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
  padding: 0.25rem 3.5rem;
  border: 1px solid #ccc;
  border-radius: 0.8rem;
  font-size: 1.1rem;
  font-weight: bold;
  color: #000;
}

.size-control-buttons {
  display: flex;
  gap: 0.5rem;
}

.size-control-button {
  /* margin: 10px 0; */
  /* padding: 10px 20px; */
  font-size: 1rem;
  font-weight: bold;
  color: #fff;
  background-color: #1a1a1a;
  border: none;
  border-radius: 0.8rem;
  cursor: pointer;
}

.size-control-button:hover {
  background-color: #1a1a1a50;
}


.bottom-menu-group {
  justify-content: space-around;
  align-items: center;
  margin-top: 1rem;

  display: grid;
  grid-template-columns: repeat(2, 1fr);
  padding: 1rem;
  gap: 1rem;
  /* Create 2 equal columns */
}

.bottom-menu-group-item {
  background-color: #e5e5e5;
  padding: 0.5rem;
  aspect-ratio: 1;
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  align-items: center;
  border-radius: 1rem;
  gap: 0.5rem;
  box-shadow: 2px 2px 10px rgba(0, 0, 0, 0.3);
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
