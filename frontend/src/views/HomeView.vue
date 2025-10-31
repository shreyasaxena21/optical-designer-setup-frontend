<template>
  <div class="designer-layout">
    <aside class="component-palette">
      <h2>Optical Components</h2>
      <div class="component-list">
        <div class="palette-item" draggable="true" @dragstart="handleDragStart($event, 'Source', laserSourceImage)">
          <img :src="laserSourceImage" alt="Laser Source" />
          <span>Laser Source</span>
        </div>
        
        <div class="palette-item" draggable="true" @dragstart="handleDragStart($event, 'Mirror', planarMirrorImage)">
          <img :src="planarMirrorImage" alt="Planar Mirror" />
          <span>Planar Mirror</span>
        </div>

        <div class="palette-item" draggable="true" @dragstart="handleDragStart($event, 'Lens', convexLensImage)">
          <img :src="convexLensImage" alt="Convex Lens" />
          <span>Convex Lens</span>
        </div>
        
        <div class="palette-item" draggable="true" @dragstart="handleDragStart($event, 'Beam Splitter', beamSplitterImage)">
          <img :src="beamSplitterImage" alt="Beam Splitter" />
          <span>Beam Splitter</span>
        </div>

        <div class="palette-item" draggable="true" @dragstart="handleDragStart($event, 'Photo Detector', photoDetectorImage)">
          <img :src="photoDetectorImage" alt="Photo Detector" />
          <span>Photo Detector</span>
        </div>
      </div>
    </aside>

    <main 
      class="grid-area" 
      @dragover.prevent="handleDragOver" 
      @drop="handleDrop"
    >
      <div v-for="comp in components" :key="comp.id" 
        class="grid-component" 
        :class="{ 'selected': selectedComponentId === comp.id }" 
        :data-id="comp.id" 
        :style="{ 
          left: `${comp.x}px`, 
          top: `${comp.y}px`, 
          transform: `rotate(${comp.angle}deg)`,
          zIndex: selectedComponentId === comp.id ? 10 : 9 
        }"
        @mousedown="startDragComponent(comp.id, $event)"
        @click.stop="selectComponent(comp.id)"
      >
        <img v-if="comp.imageUrl" :src="comp.imageUrl" :alt="comp.type" class="component-image" />
        <div v-else class="component-placeholder">{{ comp.type.charAt(0) }}</div>
        <div class="rotation-handle" @mousedown.stop="startRotateComponent(comp.id, $event)"></div>
      </div>

      <svg class="ray-visualization" v-if="simulationResults && simulationResults.results && simulationResults.results.rays_out">
        <line 
          v-for="(ray, index) in simulationResults.results.rays_out" 
          :key="index"
          :x1="ray.x1" 
          :y1="ray.y1" 
          :x2="ray.x2" 
          :y2="ray.y2" 
          :stroke="ray.color || 'yellow'" 
          stroke-width="2" 
          stroke-linecap="round"
        />
      </svg>
    </main>

    <aside class="controls-panel">
      <h2>Properties & Controls</h2>
      
      <div v-if="selectedComponent">
        <h3>{{ selectedComponent.type }} Properties</h3>
        <label>ID: {{ selectedComponent.id }}</label><br/>
        <label>X: <input type="number" v-model.number="selectedComponent.x" @input="selectedComponent.x = $event.target.valueAsNumber" /></label><br/>
        <label>Y: <input type="number" v-model.number="selectedComponent.y" @input="selectedComponent.y = $event.target.valueAsNumber" /></label><br/>
        <label>Angle: <input type="number" v-model.number="selectedComponent.angle" @input="selectedComponent.angle = $event.target.valueAsNumber" /></label><br/>
        
        <div v-if="selectedComponent.type === 'Source'">
          <label>Wavelength: <input type="number" v-model.number="selectedComponent.properties.wavelength" /></label>
          <label>Spread (deg): <input type="number" v-model.number="selectedComponent.properties.spread_deg" /></label>
        </div>
        <div v-if="selectedComponent.type === 'Mirror'">
          <label>Reflectivity: <input type="number" step="0.01" min="0" max="1" v-model.number="selectedComponent.properties.reflectivity" /></label>
        </div>
        <div v-if="selectedComponent.type === 'Lens'">
          <label>Focal Length: <input type="number" v-model.number="selectedComponent.properties.focalLength" /></label>
          <label>Diameter: <input type="number" v-model.number="selectedComponent.properties.diameter" /></label>
        </div>
        <div v-if="selectedComponent.type === 'Beam Splitter'">
          <label>Split Ratio: <input type="number" step="0.01" min="0" max="1" v-model.number="selectedComponent.properties.splitRatio" /></label>
        </div>
        <div v-if="selectedComponent.type === 'Photo Detector'">
          <label>Sensitivity: <input type="number" step="0.01" min="0" max="1" v-model.number="selectedComponent.properties.sensitivity" /></label>
        </div>

        <button @click="deleteSelectedComponent">Delete Component</button>
        <hr />
      </div>

      <h3>Global Controls</h3>
      <button @click="handleDownloadJSON">Download JSON</button>
      <button @click="handleRunSimulation">Run Simulation</button>
      
      <div v-if="simulationResults" class="simulation-results">
        <h3>Simulation Results:</h3>
        <p>Approx Path Length: 
          <strong>{{ simulationResults.results?.approx_path_length_mm || 'N/A' }} mm</strong>
        </p>
        <div v-if="simulationResults.results && simulationResults.results.detectors">
            <p v-for="(value, key) in simulationResults.results.detectors" :key="key">
              Detector {{ key.split('-')[1] }}: <strong>{{ value.toFixed(3) }}</strong> Intensity
            </p>
        </div>
      </div>
        
    </aside>
  </div>
</template>

<script setup>
import { ref, computed, nextTick, watch, onBeforeUnmount } from 'vue';

// --- IMAGE IMPORTS ---
import laserSourceImage from '@/assets/laser-source.png';
import planarMirrorImage from '@/assets/planar-mirror.png';
import convexLensImage from '@/assets/convex-lens.png';
import beamSplitterImage from '@/assets/beam-splitter.png';
import photoDetectorImage from '@/assets/photo-detector.png';


// --- REF & STATE INITIALIZATION ---
const components = ref([]);
const selectedComponentId = ref(null);
const simulationResults = ref(null); 

// Drag/Rotate State
let isDragging = false;
let isRotating = false;
let draggingComponentId = null;
let startAngle = 0;
const gridAreaOffset = ref({ left: 0, top: 0 }); // Stores grid position for drag fix

// Constants
const BACKEND_URL = "http://localhost:5000/simulate";
const GRID_SIZE = 20;


// --- COMPUTED PROPERTIES ---

const selectedComponent = computed(() => 
  components.value.find(comp => comp.id === selectedComponentId.value)
);

// Property to track component layout changes for dynamic simulation
const simulationInputData = computed(() => {
    // We only need a simple string representation of the component positions/angles/props
    // to trigger the watcher.
    const dataToSimulate = components.value.map(({ id, x, y, angle, properties }) => ({
        id, x, y, angle, properties: JSON.stringify(properties) // Stringify properties for simpler watch
    }));
    return JSON.stringify(dataToSimulate);
});


// --- DRAG-AND-DROP FROM PALETTE TO GRID ---
const generateUniqueId = (type) => `${type.toLowerCase()}-${Date.now()}`;

const handleDragStart = (event, type, imageUrlVariable) => {
    event.dataTransfer.setData('componentType', type);
    event.dataTransfer.setData('componentImageUrl', imageUrlVariable); 
    event.dataTransfer.effectAllowed = 'copy';
};

const handleDragOver = (event) => {
    event.preventDefault();
    event.dataTransfer.dropEffect = 'copy';
};

const handleDrop = (event) => {
    event.preventDefault();
    const type = event.dataTransfer.getData('componentType');
    const imageUrl = event.dataTransfer.getData('componentImageUrl');

    if (!type) return;

    // Get position relative to the grid-area
    const gridArea = event.currentTarget.getBoundingClientRect();
    let x = event.clientX - gridArea.left;
    let y = event.clientY - gridArea.top;

    // Snap to grid
    x = Math.round(x / GRID_SIZE) * GRID_SIZE;
    y = Math.round(y / GRID_SIZE) * GRID_SIZE;

    const newComponent = {
        id: generateUniqueId(type),
        type: type,
        x: x,
        y: y,
        angle: 0,
        imageUrl: imageUrl,
        properties: {}
    };

    // Set default properties based on type
    if (type === 'Source') {
        newComponent.properties = { wavelength: 632, intensity: 1.0, spread_deg: 0 };
    } else if (type === 'Mirror') {
        newComponent.properties = { reflectivity: 0.95, diameter: 50 };
    } else if (type === 'Lens') {
        newComponent.properties = { focalLength: 100, diameter: 50, transmission: 0.98 };
    } else if (type === 'Beam Splitter') {
        newComponent.properties = { splitRatio: 0.5, diameter: 50 };
    } else if (type === 'Photo Detector') {
        newComponent.properties = { sensitivity: 1.0, diameter: 50 };
    }
    
    components.value.push(newComponent);
    selectedComponentId.value = newComponent.id;
    handleRunSimulation(); // Run simulation on new component drop
};

// --- DRAGGING LOGIC (FIXED) ---
const startDragComponent = (id, event) => {
    isDragging = true;
    draggingComponentId = id;
    selectedComponentId.value = id;
    const comp = components.value.find(c => c.id === id);
    if (!comp) return;

    // FIX 1: Capture grid offset by finding the grid-area *parent*
    // event.currentTarget is the .grid-component
    const gridArea = event.currentTarget.parentElement.getBoundingClientRect();
    gridAreaOffset.value.left = gridArea.left;
    gridAreaOffset.value.top = gridArea.top;
    
    // Calculate offset from mouse to component's top-left
    const compRect = event.currentTarget.getBoundingClientRect();
    comp.offsetX = event.clientX - compRect.left;
    comp.offsetY = event.clientY - compRect.top;

    window.addEventListener('mousemove', dragComponentMove);
    window.addEventListener('mouseup', stopDragComponent);
};

const dragComponentMove = (event) => {
    if (!isDragging || !draggingComponentId) return;
    const comp = components.value.find(c => c.id === draggingComponentId);
    if (!comp) return;

    // FIX 2: Use stored grid offset
    let newX = event.clientX - gridAreaOffset.value.left - comp.offsetX; 
    let newY = event.clientY - gridAreaOffset.value.top - comp.offsetY;

    // Snap to grid
    newX = Math.round(newX / GRID_SIZE) * GRID_SIZE;
    newY = Math.round(newY / GRID_SIZE) * GRID_SIZE;

    comp.x = newX;
    comp.y = newY;
};

const stopDragComponent = () => {
    isDragging = false;
    draggingComponentId = null;
    window.removeEventListener('mousemove', dragComponentMove);
    window.removeEventListener('mouseup', stopDragComponent);
    // CRITICAL: Trigger simulation after drag is complete
    handleRunSimulation(); 
};

// --- ROTATION LOGIC ---
const startRotateComponent = (id, event) => {
    isRotating = true;
    draggingComponentId = id;
    selectedComponentId.value = id;
    const comp = components.value.find(c => c.id === id);
    if (!comp) return;

    const compRect = event.currentTarget.closest('.grid-component').getBoundingClientRect();
    const centerX = compRect.left + compRect.width / 2;
    const centerY = compRect.top + compRect.height / 2;

    const dx = event.clientX - centerX;
    const dy = event.clientY - centerY;
    startAngle = Math.atan2(dy, dx) * (180 / Math.PI);

    window.addEventListener('mousemove', rotateComponentMove);
    window.addEventListener('mouseup', stopRotateComponent);
};

const rotateComponentMove = (event) => {
    if (!isRotating || !draggingComponentId) return;
    const comp = components.value.find(c => c.id === draggingComponentId);
    if (!comp) return;

    const compElement = document.querySelector(`.grid-component[data-id="${comp.id}"]`); 
    if (!compElement) return;

    const compRect = compElement.getBoundingClientRect();
    const centerX = compRect.left + compRect.width / 2;
    const centerY = compRect.top + compRect.height / 2;

    const dx = event.clientX - centerX;
    const dy = event.clientY - centerY;
    const currentAngle = Math.atan2(dy, dx) * (180 / Math.PI);

    // Update angle and normalize to 0-360
    comp.angle = (comp.angle + (currentAngle - startAngle));
    comp.angle = (comp.angle % 360 + 360) % 360; // Keep angle positive
    startAngle = currentAngle;
};

const stopRotateComponent = () => {
    isRotating = false;
    draggingComponentId = null;
    window.removeEventListener('mousemove', rotateComponentMove);
    window.removeEventListener('mouseup', stopRotateComponent);
    handleRunSimulation(); // Trigger simulation after rotation is complete
};


// --- GENERAL UI LOGIC ---
const selectComponent = (id) => {
    selectedComponentId.value = id;
};

const deleteSelectedComponent = () => {
    if (selectedComponentId.value) {
        components.value = components.value.filter(comp => comp.id !== selectedComponentId.value);
        selectedComponentId.value = null;
        handleRunSimulation(); // Run simulation after delete
    }
};

// --- CORE: DYNAMIC SIMULATION & BACKEND CONNECTION ---
const handleRunSimulation = async () => {
    try {
        const dataToSend = components.value.map(({ id, type, x, y, angle, properties }) => ({
            id, type, x, y, angle, properties
        }));

        const response = await fetch(BACKEND_URL, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(dataToSend),
        });

        if (!response.ok) {
            throw new Error(`HTTP error! Status: ${response.status}`);
        }

        const data = await response.json();
        // data will be { "status": "success", "results": { "rays_out": [...], ... } }
        simulationResults.value = data; 
        
    } catch (error) {
        console.error("Error running simulation:", error);
        // Fallback or display error message to user
        simulationResults.value = null;
    }
};

// CORE: WATCHER to automatically run the simulation on component properties input changes
watch(simulationInputData, (newVal, oldVal) => {
    if (newVal !== oldVal) {
        if (window.simulationTimeout) {
            clearTimeout(window.simulationTimeout);
        }
        window.simulationTimeout = setTimeout(() => {
            // Only run if not dragging/rotating (which have their own trigger on mouseup/stop)
            if (!isDragging && !isRotating) {
               handleRunSimulation();
            }
        }, 300); 
    }
}, { deep: false });

// --- UTILITY ---
const handleDownloadJSON = () => {
    const dataToExport = components.value.map(({ id, type, x, y, angle, properties }) => ({
        id, type, x, y, angle, properties
    }));
    const dataStr = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(dataToExport, null, 2));
    const downloadAnchorNode = document.createElement('a');
    
    const now = new Date().toISOString().replace(/[:.]/g, '-');
    downloadAnchorNode.setAttribute("href", dataStr);
    downloadAnchorNode.setAttribute("download", `setup_${now}.json`);
    
    document.body.appendChild(downloadAnchorNode);
    nextTick(() => { 
        downloadAnchorNode.click();
        downloadAnchorNode.remove();
    });
};

onBeforeUnmount(() => {
    // Clean up global listeners to prevent memory leaks/errors
    window.removeEventListener('mousemove', dragComponentMove);
    window.removeEventListener('mouseup', stopDragComponent);
    window.removeEventListener('mousemove', rotateComponentMove);
    window.removeEventListener('mouseup', stopRotateComponent);
    if (window.simulationTimeout) {
         clearTimeout(window.simulationTimeout);
    }
});

// Run initial simulation if components are already present (e.g., from local storage or initial load)
nextTick(() => {
    if (components.value.length > 0) {
        handleRunSimulation();
    }
});

</script>

<style scoped>
/* (The existing CSS styles go here) */
.designer-layout {
  display: flex;
  height: 100vh;
  font-family: Arial, sans-serif;
  background-color: #141414;
}
.component-palette {
  width: 200px;
  background-color: #030303;
  border-right: 1px solid #444;
  padding: 15px;
  overflow-y: auto;
  color: white;
}
.component-palette h2 {
  color: #fff;
  border-bottom: 1px solid #444;
  padding-bottom: 10px;
  margin-bottom: 15px;
}
.palette-item {
  display: flex;
  align-items: center;
  padding: 10px;
  border: 1px solid #333;
  border-radius: 8px;
  background-color: #222;
  cursor: grab;
  color: #fff;
  transition: background-color 0.2s;
  margin-bottom: 10px;
}
.palette-item:hover {
  background-color: #35495e;
}
.palette-item img {
  width: 32px;
  height: 32px;
  margin-right: 10px;
  object-fit: contain;
}
.grid-area {
  flex-grow: 1;
  position: relative;
  background-color: #0a0a0a; 
  background-image: 
    linear-gradient(to right, #333 1px, transparent 1px),
    linear-gradient(to bottom, #333 1px, transparent 1px);
  background-size: 20px 20px;
  cursor: grab;
  overflow: hidden;
}
.grid-component {
  position: absolute;
  width: 50px;
  height: 50px;
  display: flex;
  justify-content: center;
  align-items: center;
  border: 2px solid transparent; 
  border-radius: 8px;
  cursor: grab;
  user-select: none;
  transition: border-color 0.1s, z-index 0s;
}
.grid-component.selected {
  border-color: #1890ff;
  box-shadow: 0 0 10px rgba(24, 144, 255, 0.5);
}
.component-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  pointer-events: none;
}
.rotation-handle {
  position: absolute;
  width: 15px;
  height: 15px;
  background-color: #1890ff;
  border-radius: 50%;
  border: 1px solid white;
  bottom: -5px; 
  right: -5px;
  cursor: ew-resize;
  z-index: 101;
}
.ray-visualization {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
}
.controls-panel {
  width: 280px;
  background-color: #0d0d0d;
  border-left: 1px solid #444;
  padding: 15px;
  overflow-y: auto;
  color: white;
}
.controls-panel h2, .controls-panel h3 {
  color: #fff;
}
.controls-panel label {
  display: block;
  margin-bottom: 8px;
  font-size: 0.9em;
  color: #ccc;
}
.controls-panel input[type="number"] {
  width: calc(100% - 10px);
  padding: 6px;
  border: 1px solid #555;
  background-color: #333;
  color: white;
  border-radius: 4px;
  margin-top: 4px;
}
.controls-panel button {
  width: 100%;
  padding: 10px 15px;
  margin-bottom: 10px;
  background-color: #1890ff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background-color 0.2s;
}
.controls-panel button:hover {
  background-color: #40a9ff;
}
.simulation-results {
  background-color: #1a1a1a;
  border: 1px solid #444;
  border-radius: 4px;
  padding: 10px;
  margin-top: 20px;
}
.simulation-results p {
  margin: 5px 0;
  font-size: 0.9em;
  color: #ccc;
}
</style>