<script setup lang="ts">
import { ref, onMounted, onUnmounted, computed } from 'vue';
import { useRouter } from 'vue-router';
import { Cartesian3, Math as CesiumMath, Color } from 'cesium';
import spaceBgImage from '@/assets/fond.png';
import Spacecraft from '@/components/Spacecraft.vue';
import PlanetTerritories from '@/components/PlanetTerritories.vue';
import type { Planet, PlanetLevel, Game } from '@/types';

const router = useRouter();

interface ClickPosition {
  x: number;
  y: number;
}

interface CameraPosition {
  x: number;
  y: number;
}

type ViewState = 'global' | 'traveling' | 'zooming' | 'planet';

// Configuration
const mapWidth = ref<number>(0);
const mapHeight = ref<number>(0);
const currentScale = ref<number>(1);
const cameraPos = ref<CameraPosition>({ x: 0, y: 0 });
const lastClick = ref<ClickPosition>({ x: 0, y: 0 });
const currentViewState = ref<ViewState>('global');
const selectedPlanet = ref<Planet | null>(null);
const selectedTerritory = ref<PlanetLevel | null>(null);
const spacecraftPosition = ref({ x: 0, y: 0 });
const targetPlanet = ref<Planet | null>(null);
const isTraveling = ref(false);
const isZooming = ref(false);
const zoomLevel = ref(1);
const lastPlanetVisited = ref<Planet | null>(null);
const notification = ref<{ message: string; visible: boolean }>({ message: '', visible: false });

// Position initiale du vaisseau (au centre de la Terre)
const initialSpacecraftPosition = { 
  x: 364 + 77, // Centre de la Terre (x + size/2) - position mise à jour selon l'image
  y: 428 + 77   // Centre de la Terre (y + size/2) - position mise à jour selon l'image
};

// Liste des planètes avec leurs niveaux
// Positions ajustées pour correspondre exactement à l'image de fond
const planets = ref<Planet[]>([
  { 
    id: 0, 
    name: 'Soleil', 
    x: 568,  // Centre de l'image (soleil au centre)
    y: 247,   // Centre vertical
    size: 400, // Grand soleil au centre
    color: 'rgba(255, 100, 0, 0.1)',
    levels: [
      { 
        id: 1, 
        name: 'Photosphère', 
        description: 'Surface visible du Soleil', 
        depth: 0, 
        color: 'rgba(255, 200, 0, 0.3)',
        x: 20, y: 20, width: 25, height: 25,
        games: [
          { name: 'Jeu Solaire 1', description: 'Explorez la surface du soleil', route: '/solar-game-1', icon: '☀️' }
        ]
      },
      { 
        id: 2, 
        name: 'Chromosphère', 
        description: 'Couche atmosphérique solaire', 
        depth: 1, 
        color: 'rgba(255, 150, 0, 0.3)',
        x: 55, y: 20, width: 25, height: 25
      },
      { 
        id: 3, 
        name: 'Zone de convection', 
        description: 'Zone de mouvement convectif', 
        depth: 2, 
        color: 'rgba(255, 100, 0, 0.3)',
        x: 20, y: 55, width: 25, height: 25
      },
      { 
        id: 4, 
        name: 'Noyau', 
        description: 'Cœur du Soleil', 
        depth: 3, 
        color: 'rgba(255, 50, 0, 0.3)',
        x: 55, y: 55, width: 25, height: 25
      }
    ]
  },
  { id: 1, name: 'Mercure', x: 222, y: 276, size: 80, color: 'rgba(180, 180, 180, 0.3)' },  // Haut-gauche (sous la comète)
  { id: 2, name: 'Vénus', x: 1070, y: 463, size: 100, color: 'rgba(220, 180, 50, 0.3)' },    // Haut-gauche (au-dessus de la Terre)
  { 
    id: 3, 
    name: 'Terre', 
    x: 364,  // Milieu-gauche (au-dessus de Jupiter)
    y: 428,   // Position verticale
    size: 154, 
    color: 'rgba(0, 100, 255, 0.3)',
    levels: [
      { 
        id: 1, 
        name: 'Surface', 
        description: 'Croûte terrestre et océans', 
        depth: 0, 
        color: 'rgba(0, 150, 255, 0.3)',
        x: 20, y: 20, width: 25, height: 25,
        games: [
          { name: 'Exploration Terrestre', description: 'Explorez la surface de la Terre', route: '/earth-surface', icon: '🌍' },
          { name: 'Océans', description: 'Plongez dans les océans', route: '/oceans', icon: '🌊' }
        ]
      },
      { 
        id: 2, 
        name: 'Crust', 
        description: 'Couche crustale solide', 
        depth: 1, 
        color: 'rgba(100, 100, 150, 0.3)',
        x: 55, y: 20, width: 25, height: 25
      },
      { 
        id: 3, 
        name: 'Manteau', 
        description: 'Manteau supérieur et inférieur', 
        depth: 2, 
        color: 'rgba(150, 50, 50, 0.3)',
        x: 20, y: 55, width: 25, height: 25
      },
      { 
        id: 4, 
        name: 'Noyau', 
        description: 'Noyau planétaire', 
        depth: 3, 
        color: 'rgba(200, 100, 0, 0.3)',
        x: 55, y: 55, width: 25, height: 25
      }
    ]
  },
  { id: 4, name: 'Mars', x: 215, y: 555, size: 135, color: 'rgba(255, 50, 0, 0.3)' },        // Milieu-gauche (sous la Terre)
  { id: 5, name: 'Jupiter', x: 456, y: 645, size: 200, color: 'rgba(200, 150, 100, 0.3)' },  // Bas-gauche (grande, rayée)
  { id: 6, name: 'Saturne', x: 1012, y: 185, size: 180, color: 'rgba(240, 220, 150, 0.3)' }, // Gauche (avec anneaux)
  { id: 7, name: 'Uranus', x: 1030, y: 697, size: 110, color: 'rgba(100, 200, 255, 0.3)' },  // Gauche (bleu clair)
  { id: 8, name: 'Neptune', x: 1247, y: 529, size: 100, color: 'rgba(0, 0, 150, 0.3)' },     // Gauche (bleu profond)
  { id: 9, name: 'Pluton', x: 1011, y: 102, size: 50, color: 'rgba(200, 200, 200, 0.3)' }   // Gauche (haut)
]);

// Initialiser la position du vaisseau
onMounted(() => {
  spacecraftPosition.value = { ...initialSpacecraftPosition };
});

// Fonctions
const initMapDimensions = (): void => {
  const img = new Image();
  img.src = spaceBgImage;
  img.onload = () => {
    mapWidth.value = img.naturalWidth;
    mapHeight.value = img.naturalHeight;
    fitToScreen();
  };
};

const fitToScreen = (): void => {
  if (mapWidth.value === 0) return;

  const screenW = window.innerWidth;
  const screenH = window.innerHeight;

  const scaleX = screenW / mapWidth.value;
  const scaleY = screenH / mapHeight.value;
  
  const newScale = Math.max(scaleX, scaleY); 

  currentScale.value = newScale;

  cameraPos.value.x = (screenW - (mapWidth.value * newScale)) / 2;
  cameraPos.value.y = (screenH - (mapHeight.value * newScale)) / 2;
};

const calculateAngle = (x1: number, y1: number, x2: number, y2: number): number => {
  const angle = Math.atan2(y2 - y1, x2 - x1) * (180 / Math.PI);
  return angle;
};

const showNotification = (message: string): void => {
  notification.value = { message, visible: true };
  setTimeout(() => {
    notification.value.visible = false;
  }, 3000); // Disparaît après 3 secondes
};

const travelToPlanet = (planet: Planet): void => {
  if (currentViewState.value === 'traveling' || currentViewState.value === 'zooming') return;
  
  // Vérifier si on essaie de voyager vers la même planète
  if (selectedPlanet.value && selectedPlanet.value.id === planet.id) {
    showNotification('Vous êtes déjà sur cette planète !');
    return;
  }
  
  // Si on est déjà sur une planète, dézoomer d'abord
  if (currentViewState.value === 'planet' && selectedPlanet.value) {
    // Dézoomer rapidement
    const currentZoom = currentScale.value;
    const dezoomDuration = 500;
    const startTime = Date.now();
    
    const dezoom = () => {
      const elapsed = Date.now() - startTime;
      const progress = Math.min(elapsed / dezoomDuration, 1);
      const easeProgress = 1 - Math.pow(1 - progress, 2);
      
      currentScale.value = currentZoom - (currentZoom - 1) * easeProgress;
      
      // Ajuster la caméra
      const screenW = window.innerWidth;
      const screenH = window.innerHeight;
      const planetCenterX = selectedPlanet.value!.x + (selectedPlanet.value!.size / 2);
      const planetCenterY = selectedPlanet.value!.y + (selectedPlanet.value!.size / 2);
      
      // Calculer la position de la caméra pour centrer la planète
      // L'univers est centré, donc on doit compenser avec le centre de l'univers
      cameraPos.value.x = (mapWidth.value / 2 - planetCenterX) * currentScale.value;
      cameraPos.value.y = (mapHeight.value / 2 - planetCenterY) * currentScale.value;
      
      if (progress < 1) {
        requestAnimationFrame(dezoom);
      } else {
        // Une fois dézoomé, commencer le voyage
        startTravelToPlanet(planet);
      }
    };
    
    requestAnimationFrame(dezoom);
    selectedTerritory.value = null;
  } else {
    // Voyage direct depuis la vue globale
    startTravelToPlanet(planet);
  }
};

const startTravelToPlanet = (planet: Planet): void => {
  targetPlanet.value = planet;
  currentViewState.value = 'traveling';
  isTraveling.value = true;

  // Calculer la position cible (centre de la planète)
  const targetX = planet.x + (planet.size / 2);
  const targetY = planet.y + (planet.size / 2);

  // Position de départ du vaisseau (planète actuelle ou position initiale)
  const startX = lastPlanetVisited.value 
    ? lastPlanetVisited.value.x + (lastPlanetVisited.value.size / 2)
    : spacecraftPosition.value.x;
  const startY = lastPlanetVisited.value
    ? lastPlanetVisited.value.y + (lastPlanetVisited.value.size / 2)
    : spacecraftPosition.value.y;

  // Animer la caméra vers la planète
  const screenW = window.innerWidth;
  const screenH = window.innerHeight;
  
  currentScale.value = 1;
  // Calculer la position de la caméra pour centrer la planète
  // L'univers est centré, donc on doit compenser avec le centre de l'univers
  cameraPos.value.x = (mapWidth.value / 2 - targetX) * currentScale.value;
  cameraPos.value.y = (mapHeight.value / 2 - targetY) * currentScale.value;

  // Positionner le vaisseau à la position de départ
  spacecraftPosition.value = {
    x: startX,
    y: startY
  };

  // Animer le vaisseau vers la planète
  setTimeout(() => {
    spacecraftPosition.value = {
      x: targetX,
      y: targetY
    };
  }, 50);

  // Après l'animation (3 secondes), commencer le zoom
  setTimeout(() => {
    isTraveling.value = false;
    selectedPlanet.value = planet;
    lastPlanetVisited.value = planet;
    currentViewState.value = 'zooming';
    isZooming.value = true;
    zoomLevel.value = 1;
    
    // Animation de zoom progressif
    const zoomDuration = 2000;
    const startTime = Date.now();
    const startZoom = 1;
    // Calculer le zoom pour que la planète remplisse environ 60-70% de l'écran
    const screenW = window.innerWidth;
    const screenH = window.innerHeight;
    const planetSize = planet.size;
    const scaleX = (screenW * 0.6) / planetSize;
    const scaleY = (screenH * 0.6) / planetSize;
    const endZoom = Math.min(scaleX, scaleY, 5); // Limiter à 5x max pour voir toute la planète
    
    const animateZoom = () => {
      const elapsed = Date.now() - startTime;
      const progress = Math.min(elapsed / zoomDuration, 1);
      
      // Courbe d'easing pour un effet plus fluide
      const easeProgress = 1 - Math.pow(1 - progress, 3);
      zoomLevel.value = startZoom + (endZoom - startZoom) * easeProgress;
      
      // Ajuster la caméra pour rester centrée sur la planète
      const screenW = window.innerWidth;
      const screenH = window.innerHeight;
      const planetCenterX = targetX;
      const planetCenterY = targetY;
      
      // Calculer la position de la caméra pour centrer la planète
      // L'univers est centré, donc on doit compenser avec le centre de l'univers
      cameraPos.value.x = (mapWidth.value / 2 - planetCenterX) * zoomLevel.value;
      cameraPos.value.y = (mapHeight.value / 2 - planetCenterY) * zoomLevel.value;
      currentScale.value = zoomLevel.value;
      
      if (progress < 1) {
        requestAnimationFrame(animateZoom);
      } else {
        isZooming.value = false;
        currentViewState.value = 'planet';
      }
    };
    
    requestAnimationFrame(animateZoom);
  }, 3000);
};

const selectTerritory = (territory: PlanetLevel): void => {
  selectedTerritory.value = territory;
};

const returnToSpace = (): void => {
  currentViewState.value = 'global';
  lastPlanetVisited.value = selectedPlanet.value;
  selectedPlanet.value = null;
  selectedTerritory.value = null;
  targetPlanet.value = null;
  isZooming.value = false;
  zoomLevel.value = 1;
  
  // Remettre le vaisseau à la position de la planète visitée
  if (lastPlanetVisited.value) {
    spacecraftPosition.value = {
      x: lastPlanetVisited.value.x + (lastPlanetVisited.value.size / 2),
      y: lastPlanetVisited.value.y + (lastPlanetVisited.value.size / 2)
    };
  } else {
    spacecraftPosition.value = { ...initialSpacecraftPosition };
  }
  
  fitToScreen();
};

const handleGameSelect = (game: Game): void => {
  // Navigation vers le jeu sélectionné
  router.push(game.route);
};

const logClickPosition = (event: MouseEvent): void => {
  if (currentViewState.value !== 'global') return;
  
  const screenX = event.clientX;
  const screenY = event.clientY;

  const camX = cameraPos.value.x;
  const camY = cameraPos.value.y;
  const scale = currentScale.value;

  const realX = (screenX - camX) / scale;
  const realY = (screenY - camY) / scale;

  // Log dans la console pour le développement
  console.log(`[DEV] Clic position - X: ${Math.round(realX)}, Y: ${Math.round(realY)}`);
  
  lastClick.value = { x: Math.round(realX), y: Math.round(realY) };
};

// Angle du vaisseau en fonction de sa direction
const spacecraftAngle = computed(() => {
  if (!targetPlanet.value || !isTraveling.value) {
    // Pas d'angle spécifique quand on n'est pas en voyage
    return 0;
  }
  
  // Pendant le voyage, calculer l'angle vers la destination
  // On utilise la position actuelle ou la dernière planète visitée
  const startX = lastPlanetVisited.value 
    ? lastPlanetVisited.value.x + (lastPlanetVisited.value.size / 2)
    : initialSpacecraftPosition.x;
  const startY = lastPlanetVisited.value
    ? lastPlanetVisited.value.y + (lastPlanetVisited.value.size / 2)
    : initialSpacecraftPosition.y;
  
  return calculateAngle(
    startX,
    startY,
    spacecraftPosition.value.x,
    spacecraftPosition.value.y
  );
});

// Position du vaisseau dans la vue planète
const planetViewSpacecraftPosition = computed(() => {
  return {
    x: 100,
    y: window.innerHeight - 150
  };
});

// Centre de la planète sélectionnée pour le zoom
const planetCenter = computed(() => {
  if (!selectedPlanet.value) return { x: 0, y: 0 };
  return {
    x: selectedPlanet.value.x + (selectedPlanet.value.size / 2),
    y: selectedPlanet.value.y + (selectedPlanet.value.size / 2)
  };
});

onMounted(() => {
  initMapDimensions();
  window.addEventListener('resize', fitToScreen);
});

onUnmounted(() => {
  window.removeEventListener('resize', fitToScreen);
});
</script>

<template>
  <div class="solar-system-page">
    <!-- Vue Univers (globale, voyage, ou zoom) -->
    <div
      v-if="mapWidth > 0"
      class="universe"
      @click="logClickPosition"
      :style="{ 
        '--x': `${cameraPos.x}px`, 
        '--y': `${cameraPos.y}px`,
        '--s': currentScale,
        '--w': `${mapWidth}px`,
        '--h': `${mapHeight}px`,
        backgroundImage: `url(${spaceBgImage})`
      }"
    >
      <!-- Vaisseau spatial -->
      <Spacecraft
        v-if="currentViewState === 'global' || currentViewState === 'traveling' || currentViewState === 'planet'"
        :x="spacecraftPosition.x"
        :y="spacecraftPosition.y"
        :angle="spacecraftAngle"
        :is-traveling="isTraveling"
        :scale="currentViewState === 'planet' ? 1 / currentScale : 1 / currentScale"
      />

      <!-- Planètes -->
      <div 
        v-for="planet in planets" 
        :key="planet.id"
        class="planet-hotspot"
        :class="{ 
          'disabled': currentViewState === 'traveling' || currentViewState === 'zooming',
          'selected': selectedPlanet?.id === planet.id,
          'hidden': currentViewState === 'planet'
        }"
        :style="{
          left: `${planet.x + planet.size / 2}px`,
          top: `${planet.y + planet.size / 2}px`,
          width: `${planet.size}px`,
          height: `${planet.size}px`,
          backgroundColor: planet.color
        }"
        @click.stop="travelToPlanet(planet)"
      >
        <span class="planet-name" :style="{ transform: `scale(${1/currentScale})` }">
          {{ planet.name }}
        </span>
      </div>

      <!-- Territoires affichés sur la planète -->
      <div 
        v-if="currentViewState === 'planet' && selectedPlanet && selectedPlanet.levels"
        class="planet-territories-overlay"
      >
        <div 
          v-for="territory in (selectedPlanet.levels || [])" 
          :key="territory.id"
          class="territory-marker"
          :style="{
            left: `${selectedPlanet.x + (selectedPlanet.size * (territory.x || 0) / 100)}px`,
            top: `${selectedPlanet.y + (selectedPlanet.size * (territory.y || 0) / 100)}px`,
            width: `${selectedPlanet.size * (territory.width || 40) / 100}px`,
            height: `${selectedPlanet.size * (territory.height || 40) / 100}px`,
            backgroundColor: territory.color || selectedPlanet.color
          }"
          @click.stop="selectTerritory(territory)"
        >
          <span class="territory-name" :style="{ transform: `scale(${1/currentScale})` }">
            {{ territory.name }}
          </span>
        </div>
      </div>
    </div>

    <!-- Boutons de navigation -->
    <button 
      v-if="currentViewState === 'planet'"
      @click="returnToSpace"
      class="back-arrow"
    >
      ←
    </button>

    <!-- Notification -->
    <div 
      v-if="notification.visible"
      class="notification"
    >
      {{ notification.message }}
    </div>

    <!-- Vue des jeux du territoire sélectionné -->
    <div 
      v-if="currentViewState === 'planet' && selectedTerritory && selectedTerritory.games"
      class="games-overlay"
    >
      <div class="games-container">
        <h2 class="territory-title">{{ selectedTerritory.name }}</h2>
        <div class="games-grid">
          <div 
            v-for="game in selectedTerritory.games" 
            :key="game.route"
            class="game-card"
            @click="handleGameSelect(game)"
          >
            <div class="game-icon">{{ game.icon || '🎮' }}</div>
            <h3 class="game-name">{{ game.name }}</h3>
            <p class="game-description">{{ game.description }}</p>
          </div>
        </div>
      </div>
    </div>

    <div v-else class="loading-screen">
      Chargement du système...
    </div>
  </div>
</template>

<style scoped>
.solar-system-page {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background-color: #000;
  position: relative;
}

.map-container {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
}

.game-world {
  position: relative;
  /* background-color: #000; Remove fixed color to let Cesium handle it */
}

.cesium-viewer-canvas {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  z-index: 0;
}

/* Ensure Cesium canvas fits */
:deep(.cesium-viewer) {
  width: 100%;
  height: 100%;
}

.planet-hotspot {
  position: absolute;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 1px solid rgba(255, 255, 255, 0.3);
  transition: all 0.3s ease;
  z-index: 40;
  width: 150px;
  height: 124px;
  min-width: 80px;
  transform: translate(-50%, -50%);
}

.planet-hotspot.disabled {
  cursor: not-allowed;
  opacity: 0.5;
  pointer-events: none;
}

.planet-hotspot.hidden {
  display: none;
}

.planet-hotspot:not(.disabled) {
  opacity: 1;
  pointer-events: all;
}

.planet-hotspot.selected {
  border: 2px solid rgba(74, 158, 255, 0.8);
  box-shadow: 0 0 20px rgba(74, 158, 255, 0.5);
  z-index: 45;
  opacity: 1 !important;
}

.planet-hotspot:hover:not(.disabled) {
  background-color: rgba(255, 255, 255, 0.15) !important;
  border-color: #fff;
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.4);
  z-index: 10;
  transform: translate(-50%, -50%) scale(1.05);
}

.planet-name {
  color: white;
  font-family: 'Segoe UI', sans-serif;
  font-weight: 600;
  text-shadow: 0 0 4px #000, 0 0 8px #000;
  pointer-events: none;
  user-select: none;
  white-space: nowrap;
  transition: transform 1.2s cubic-bezier(0.2, 0.8, 0.2, 1);
}

.planet-territories-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 30;
  pointer-events: none;
}

.territory-marker {
  position: absolute;
  border: 1px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: all;
  transition: all 0.3s ease;
  overflow: hidden;
  /* Style similaire aux planètes : transparent */
}

.territory-marker:hover {
  background-color: rgba(255, 255, 255, 0.15) !important;
  border-color: #fff;
  box-shadow: 0 0 20px rgba(255, 255, 255, 0.4);
  z-index: 35;
  transform: scale(1.05);
}

.territory-name {
  color: white;
  font-family: 'Segoe UI', sans-serif;
  font-weight: 600;
  font-size: 0.9rem;
  text-shadow: 0 0 4px #000, 0 0 8px #000;
  pointer-events: none;
  user-select: none;
  white-space: nowrap;
  text-align: center;
  padding: 5px;
}

.back-arrow {
  position: fixed;
  top: 20px;
  left: 20px;
  width: 50px;
  height: 50px;
  background: rgba(0, 0, 0, 0.6);
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  color: white;
  font-size: 24px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 300;
  transition: all 0.3s ease;
  backdrop-filter: blur(10px);
}

.back-arrow:hover {
  background: rgba(255, 255, 255, 0.2);
  border-color: rgba(255, 255, 255, 0.6);
  transform: scale(1.1);
}

.notification {
  position: fixed;
  top: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.3);
  padding: 12px 30px;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 2px;
  border-radius: 30px;
  z-index: 400;
  font-size: 14px;
  animation: slideDown 0.3s ease, fadeOut 0.3s ease 2.7s;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

@keyframes fadeOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}

.games-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.9);
  backdrop-filter: blur(20px);
  z-index: 250;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

.games-container {
  max-width: 1200px;
  width: 100%;
  text-align: center;
}

.territory-title {
  font-size: 2.5rem;
  color: white;
  margin-bottom: 40px;
  font-weight: 700;
}

.games-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 30px;
}

.game-card {
  background: rgba(255, 255, 255, 0.1);
  border: 2px solid rgba(255, 255, 255, 0.2);
  border-radius: 15px;
  padding: 30px;
  cursor: pointer;
  transition: all 0.3s ease;
  text-align: center;
  backdrop-filter: blur(10px);
}

.game-card:hover {
  transform: translateY(-10px);
  border-color: rgba(74, 158, 255, 0.8);
  box-shadow: 0 20px 40px rgba(74, 158, 255, 0.3);
}

.game-icon {
  font-size: 3rem;
  margin-bottom: 15px;
}

.game-name {
  font-size: 1.5rem;
  font-weight: 600;
  color: white;
  margin-bottom: 10px;
}

.game-description {
  color: rgba(255, 255, 255, 0.7);
  font-size: 0.9rem;
}

button {
  pointer-events: auto;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(4px);
  color: #fff;
  border: 1px solid rgba(255, 255, 255, 0.3);
  padding: 12px 30px;
  cursor: pointer;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 2px;
  border-radius: 30px;
  transition: all 0.2s;
}

button:hover {
  background: white;
  color: black;
  transform: scale(1.05);
}

.loading-screen {
  color: #fff;
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  font-family: monospace;
  font-size: 1.5rem;
}
</style>
