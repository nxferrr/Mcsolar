<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import Spacecraft from './Spacecraft.vue';
import spaceBgImage from '@/assets/fond.png';
import type { Planet, PlanetLevel } from '@/types';

interface Props {
  planet: Planet;
  spacecraftPosition: { x: number; y: number };
  mapWidth: number;
  mapHeight: number;
  planetCenterX: number;
  planetCenterY: number;
}

const props = defineProps<Props>();

const emit = defineEmits<{
  returnToSpace: [];
}>();

const zoomLevel = ref(1);
const isZooming = ref(true);
const selectedLevel = ref<PlanetLevel | null>(null);

// Niveaux par défaut si la planète n'en a pas
const defaultLevels: PlanetLevel[] = [
  { id: 1, name: 'Surface', description: 'Niveau de surface de la planète', depth: 0, color: props.planet.color },
  { id: 2, name: 'Crust', description: 'Couche crustale', depth: 1, color: props.planet.color.replace('0.3', '0.5') },
  { id: 3, name: 'Mantle', description: 'Manteau planétaire', depth: 2, color: props.planet.color.replace('0.3', '0.7') },
  { id: 4, name: 'Core', description: 'Noyau planétaire', depth: 3, color: props.planet.color.replace('0.3', '0.9') }
];

const planetLevels = computed(() => {
  return props.planet.levels || defaultLevels;
});

// Animation de zoom progressif (style Quantum Break)
onMounted(() => {
  // Zoom progressif sur 2 secondes
  const zoomDuration = 2000;
  const startTime = Date.now();
  const startZoom = 1;
  const endZoom = 20; // Zoom très important pour entrer "dans" la planète

  const animateZoom = () => {
    const elapsed = Date.now() - startTime;
    const progress = Math.min(elapsed / zoomDuration, 1);
    
    // Courbe d'easing pour un effet plus fluide
    const easeProgress = 1 - Math.pow(1 - progress, 3);
    zoomLevel.value = startZoom + (endZoom - startZoom) * easeProgress;

    if (progress < 1) {
      requestAnimationFrame(animateZoom);
    } else {
      isZooming.value = false;
    }
  };

  requestAnimationFrame(animateZoom);
});

const handleSpacecraftClick = () => {
  emit('returnToSpace');
};

const selectLevel = (level: PlanetLevel) => {
  selectedLevel.value = level;
};
</script>

<template>
  <div class="planet-view">
    <!-- Vue zoomée sur la carte (pendant le zoom) -->
    <div 
      v-if="isZooming"
      class="zoomed-universe"
    >
      <div 
        class="zoomed-map"
        :style="{
          width: `${mapWidth}px`,
          height: `${mapHeight}px`,
          backgroundImage: `url(${spaceBgImage})`,
          transform: `translate(calc(50vw - ${planetCenterX}px), calc(50vh - ${planetCenterY}px)) scale(${zoomLevel})`,
          transformOrigin: '0 0'
        }"
      ></div>
      <div class="zoom-overlay">
        <div class="zoom-message">Entrée dans {{ planet.name }}...</div>
      </div>
    </div>

    <!-- Vue des niveaux de la planète (après le zoom) -->
    <div v-else class="planet-levels-view">
      <div class="planet-header">
        <h1 class="planet-title">{{ planet.name }}</h1>
        <p class="planet-subtitle">Sélectionnez un niveau à explorer</p>
      </div>

      <div class="levels-container">
        <div 
          v-for="level in planetLevels" 
          :key="level.id"
          class="level-card"
          :class="{ 'selected': selectedLevel?.id === level.id }"
          :style="{ 
            backgroundColor: level.color || planet.color,
            '--depth': level.depth
          }"
          @click="selectLevel(level)"
        >
          <div class="level-depth-indicator">
            Profondeur: {{ level.depth }}
          </div>
          <h2 class="level-name">{{ level.name }}</h2>
          <p class="level-description">{{ level.description }}</p>
        </div>
      </div>

      <!-- Détails du niveau sélectionné -->
      <div v-if="selectedLevel" class="level-details">
        <h3>{{ selectedLevel.name }}</h3>
        <p>{{ selectedLevel.description }}</p>
        <div class="level-visualization" :style="{ backgroundColor: selectedLevel.color || planet.color }">
          <div class="depth-layers">
            <div 
              v-for="l in planetLevels.filter(l => ((selectedLevel?.depth ?? -1) >= l.depth))"
              :key="l.id"
              class="depth-layer"
              :style="{ backgroundColor: l.color || planet.color }"
            ></div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Vaisseau pour retourner -->
    <div class="spacecraft-container" @click="handleSpacecraftClick">
      <Spacecraft 
        :x="spacecraftPosition.x" 
        :y="spacecraftPosition.y"
        :angle="180"
        :scale="1.5"
      />
      <div class="click-hint">Cliquez pour retourner dans l'espace</div>
    </div>
  </div>
</template>

<style scoped>
.planet-view {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #000;
}

/* Vue zoomée style Quantum Break */
.zoomed-universe {
  width: 100vw;
  height: 100vh;
  position: relative;
  overflow: hidden;
  background: #06060c;
}

.zoomed-map {
  position: absolute;
  background-repeat: no-repeat;
  background-size: 100% 100%;
  transition: transform 0.1s linear;
  will-change: transform;
}

.zoom-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10;
  background: radial-gradient(circle, transparent 0%, rgba(0, 0, 0, 0.5) 100%);
}

.zoom-message {
  color: white;
  font-size: 2rem;
  font-weight: 600;
  text-align: center;
  padding: 30px 60px;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(10px);
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  animation: fadeInOut 2s infinite;
}

@keyframes fadeInOut {
  0%, 100% { opacity: 0.7; }
  50% { opacity: 1; }
}

/* Vue des niveaux */
.planet-levels-view {
  width: 100%;
  height: 100%;
  padding: 40px;
  overflow-y: auto;
  background: linear-gradient(135deg, #0a0a0f 0%, #1a1a2e 100%);
}

.planet-header {
  text-align: center;
  margin-bottom: 50px;
}

.planet-title {
  font-size: 3.5rem;
  font-weight: 700;
  margin-bottom: 10px;
  color: white;
  text-shadow: 2px 2px 8px rgba(0, 0, 0, 0.8);
}

.planet-subtitle {
  font-size: 1.3rem;
  color: rgba(255, 255, 255, 0.7);
}

.levels-container {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 30px;
  max-width: 1200px;
  margin: 0 auto 40px;
}

.level-card {
  background: rgba(255, 255, 255, 0.1);
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 15px;
  padding: 30px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
  overflow: hidden;
  backdrop-filter: blur(10px);
}

.level-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(135deg, transparent 0%, rgba(255, 255, 255, 0.1) 100%);
  opacity: 0;
  transition: opacity 0.3s ease;
}

.level-card:hover::before {
  opacity: 1;
}

.level-card:hover {
  transform: translateY(-10px);
  border-color: rgba(255, 255, 255, 0.5);
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.5);
}

.level-card.selected {
  border-color: rgba(74, 158, 255, 1);
  box-shadow: 0 0 30px rgba(74, 158, 255, 0.6);
  transform: translateY(-5px);
}

.level-depth-indicator {
  position: absolute;
  top: 10px;
  right: 10px;
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 5px 10px;
  border-radius: 10px;
  font-size: 0.8rem;
  font-weight: 600;
}

.level-name {
  font-size: 1.8rem;
  font-weight: 600;
  margin-bottom: 10px;
  color: white;
}

.level-description {
  color: rgba(255, 255, 255, 0.8);
  line-height: 1.6;
}

.level-details {
  max-width: 800px;
  margin: 40px auto;
  padding: 30px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 15px;
  border: 1px solid rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
}

.level-details h3 {
  font-size: 2rem;
  color: white;
  margin-bottom: 15px;
}

.level-details p {
  color: rgba(255, 255, 255, 0.8);
  font-size: 1.1rem;
  margin-bottom: 30px;
}

.level-visualization {
  width: 100%;
  height: 200px;
  border-radius: 10px;
  position: relative;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.2);
}

.depth-layers {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}

.depth-layer {
  height: calc(25% - 5px);
  margin-bottom: 5px;
  border-radius: 5px;
  transition: all 0.3s ease;
  animation: layerPulse 2s infinite;
}

.depth-layer:nth-child(1) { animation-delay: 0s; }
.depth-layer:nth-child(2) { animation-delay: 0.5s; }
.depth-layer:nth-child(3) { animation-delay: 1s; }
.depth-layer:nth-child(4) { animation-delay: 1.5s; }

@keyframes layerPulse {
  0%, 100% { opacity: 0.7; }
  50% { opacity: 1; }
}

.spacecraft-container {
  position: fixed;
  bottom: 50px;
  right: 50px;
  cursor: pointer;
  z-index: 100;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 15px;
  transition: transform 0.3s ease;
}

.spacecraft-container:hover {
  transform: scale(1.1);
}

.click-hint {
  background: rgba(0, 0, 0, 0.8);
  backdrop-filter: blur(10px);
  color: white;
  padding: 10px 20px;
  border-radius: 20px;
  font-size: 0.9rem;
  white-space: nowrap;
  border: 1px solid rgba(255, 255, 255, 0.3);
  opacity: 0;
  transition: opacity 0.3s ease;
  pointer-events: none;
}

.spacecraft-container:hover .click-hint {
  opacity: 1;
}
</style>
