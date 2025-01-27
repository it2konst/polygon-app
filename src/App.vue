<template>
  <div class="main container">
    <h1>Polygon application</h1>
    <div class="controls">
      <button @click="createPolygons">Создать</button>
      <button @click="savePolygons">Сохранить</button>
      <button @click="clearPolygons">Сбросить</button>
    </div>
    <div class="zones">
      <div class="buffer-zone">
        <h3 draggable="false">Буферная зона</h3>
        <svg class="svg-container" @drop="onDrop('buffer')" @dragover.prevent>
          <polygon
            v-for="polygon in bufferPolygons"
            :key="polygon.id"
            :points="polygon.points"
            :fill="polygon.color"
            @mousedown="startDrag(polygon, 'buffer')"
            @mouseup="stopDrag"
            @dblclick="deletePolygon(polygon, 'buffer')"
          />
        </svg>
      </div>
      <div
        class="work-zone"
        @wheel="scaleWorkZone"
        @drop="onDrop('work')"
        @dragover.prevent
        @mousemove="updateMousePosition"
        @mousedown="startPan"
        @mouseup="stopPan"
        @mouseleave="stopPan"
      >
        <h3 draggable="false">Рабочая зона</h3>
        <div class="scale">
          <span>Масштаб: {{ scale.toFixed(2) }}</span>
          <span>X: {{ mouseX }}</span>
          <span>Y: {{ mouseY }}</span>
        </div>
        <svg
          class="svg-container"
          :style="{
            transform: `scale(${scale}) translate(${panX}px, ${panY}px)`,
          }"
        >
          <polygon
            v-for="polygon in workPolygons"
            :key="polygon.id"
            :points="polygon.points"
            :fill="polygon.color"
            @mousedown="startDrag(polygon, 'work')"
            @mouseup="stopDrag"
            @dblclick="deletePolygon(polygon, 'work')"
          />
        </svg>
        <grid-lines></grid-lines>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref, onMounted } from "vue";
import GridLines from "./components/GridLines.vue";
import { useScale } from "./composables/useScale";

// Состояние приложения
const bufferPolygons = reactive([]);
const workPolygons = reactive([]);
const { scale } = useScale();
const mouseX = ref(0);
const mouseY = ref(0);
const panX = ref(0);
const panY = ref(0);
let isPanning = false;
let draggedPolygon = null;
let draggedFrom = null;

// Загрузка сохраненных данных
onMounted(() => {
  const savedPolygons = localStorage.getItem("workPolygons");
  if (savedPolygons) {
    workPolygons.push(...JSON.parse(savedPolygons));
  }
});

// Создание случайных полигонов
const createPolygons = () => {
  const count = Math.floor(Math.random() * 16) + 5; // От 5 до 20 полигонов
  for (let i = 0; i < count; i++) {
    const id = Date.now() + i;
    const color = `hsl(${Math.random() * 360}, 70%, 50%)`;
    const vertices = Math.floor(Math.random() * 8) + 3; // От 3 до 10 вершин
    const points = generatePolygonPoints(vertices);
    bufferPolygons.push({ id, color, points });
  }
};

// Генерация точек для полигона
const generatePolygonPoints = (vertices) => {
  const points = [];
  const radius = 50;
  const centerX = Math.random() * 200; // Случайное расположение по X
  const centerY = Math.random() * 200; // Случайное расположение по Y
  for (let i = 0; i < vertices; i++) {
    const angle = (i / vertices) * 2 * Math.PI;
    const x = centerX + radius * Math.cos(angle);
    const y = centerY + radius * Math.sin(angle);
    points.push(`${x},${y}`);
  }
  return points.join(" ");
};

// Сохранение полигонов в localStorage
const savePolygons = () => {
  localStorage.setItem("workPolygons", JSON.stringify(workPolygons));
};

// Очистка данных
const clearPolygons = () => {
  bufferPolygons.length = 0;
  workPolygons.length = 0;
  localStorage.removeItem("workPolygons");
};

// Масштабирование рабочей зоны
const scaleWorkZone = (event) => {
  event.preventDefault();
  scale.value += event.deltaY * -0.001;
  scale.value = Math.min(Math.max(0.5, scale.value), 2);
};

// Перетаскивание
const startDrag = (polygon, from) => {
  if (draggedPolygon) return; // Если уже перетаскивается полигон, игнорируем
  draggedPolygon = polygon;
  draggedFrom = from;
};

const stopDrag = () => {
  draggedPolygon = null;
  draggedFrom = null;
};

const onDrop = (to) => {
  if (draggedPolygon && draggedFrom !== to) {
    // Удаляем полигон из исходной зоны
    if (draggedFrom === "buffer") {
      const index = bufferPolygons.findIndex((p) => p.id === draggedPolygon.id);
      if (index !== -1) {
        bufferPolygons.splice(index, 1);
      }
    } else {
      const index = workPolygons.findIndex((p) => p.id === draggedPolygon.id);
      if (index !== -1) {
        workPolygons.splice(index, 1);
      }
    }

    // Добавляем полигон в целевую зону
    if (to === "buffer") {
      bufferPolygons.push(draggedPolygon);
    } else {
      workPolygons.push(draggedPolygon);
    }
  }
  stopDrag();
};

// Панорамирование
const startPan = () => {
  isPanning = true;
};

const stopPan = () => {
  isPanning = false;
};

const updateMousePosition = (event) => {
  if (isPanning) {
    panX.value += event.movementX;
    panY.value += event.movementY;
  }
  mouseX.value = event.clientX;
  mouseY.value = event.clientY;
};

// Метод для удаления полигона
const deletePolygon = (polygon, zone) => {
  if (zone === "buffer") {
    const index = bufferPolygons.findIndex((p) => p.id === polygon.id);
    if (index !== -1) {
      bufferPolygons.splice(index, 1);
    }
  } else if (zone === "work") {
    const index = workPolygons.findIndex((p) => p.id === polygon.id);
    if (index !== -1) {
      workPolygons.splice(index, 1);
    }
  }
};
</script>

<style>
.controls {
  padding: 1rem;
  display: flex;
  /* flex-wrap: wrap; */
  align-items: center;
  justify-content: center;
  /* justify-content: flex-start; */
  gap: 1rem;
}

.zones {
  display: flex;
  flex-direction: column;
  max-width: 100%;
  width: 100%;
}

.buffer-zone,
.work-zone {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;

  margin: 0 auto;
  max-width: 100%;
  width: 100%;
  height: 300px;

  border: 1px solid #ddd;
  overflow: hidden;
}

.svg-container {
  width: 100%;
  height: 100%;
}

.scale {
  position: absolute;
  top: 0;
  right: 0;
  background: rgba(255, 255, 255, 0.8);
  padding: 0.5rem;
  border-bottom-left-radius: 0.5rem;
}

polygon {
  cursor: move;

  &:hover {
    stroke: #000;
    stroke-width: 2px;
    /* fill: #ccc; */
  }
}
</style>
