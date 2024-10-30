<template>
  <div class="container">
    <h2 class="title">Gazo</h2>
    <div class="controls">
      <input type="file" @change="uploadImage" accept="image/*" class="upload-input" />
      <button @click="applyGrayscale" class="button">Grayscale</button>
      <button @click="addRectangle" class="button">Add Rectangle</button>
    </div>
    <canvas
      ref="canvas"
      :width="width"
      :height="height"
      @mousedown="onMouseDown"
      @mousemove="onMouseMove"
      @mouseup="onMouseUp"
      @mouseleave="onMouseLeave"
      class="canvas"
    ></canvas>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const canvas = ref(null);
const ctx = ref(null);
const width = 800;
const height = 600;

const shapes = ref([]); 
const image = ref(null); 
let imagePosition = ref({ x: 0, y: 0 });
let isDragging = ref(false);
let isResizing = ref(false);
let currentShape = ref(null);
let offset = { x: 0, y: 0 };
let resizeHandleSize = 10;

onMounted(() => {
  if (canvas.value) {
    ctx.value = canvas.value.getContext('2d');
  }
});

const uploadImage = (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const reader = new FileReader();
  reader.onload = () => {
    image.value = new Image();
    image.value.src = reader.result;
    image.value.onload = () => {
      drawShapes();
    };
  };
  reader.readAsDataURL(file);
};

const applyGrayscale = () => {
  const imgData = ctx.value.getImageData(0, 0, width, height);
  const data = imgData.data;

  for (let i = 0; i < data.length; i += 4) {
    const avg = (data[i] + data[i + 1] + data[i + 2]) / 3;
    data[i] = avg;
    data[i + 1] = avg;
    data[i + 2] = avg;
  }
  ctx.value.putImageData(imgData, 0, 0);
};

const addRectangle = () => {
  const rect = { x: 50, y: 50, width: 100, height: 100, color: 'red' };
  shapes.value.push(rect);
  drawShapes();
};

const drawShapes = () => {
  ctx.value.clearRect(0, 0, width, height);

  if (image.value) {
    ctx.value.drawImage(image.value, imagePosition.value.x, imagePosition.value.y);
  }

  shapes.value.forEach(shape => {
    ctx.value.fillStyle = shape.color;
    ctx.value.fillRect(shape.x, shape.y, shape.width, shape.height);
    
    if (shape === currentShape.value) {
      ctx.value.strokeStyle = 'blue';
      ctx.value.lineWidth = 2;
      ctx.value.strokeRect(shape.x, shape.y, shape.width, shape.height);
    }
    
    drawResizeHandles(shape);
  });
};

const drawResizeHandles = (shape) => {
  ctx.value.fillStyle = 'blue';
  ctx.value.fillRect(shape.x + shape.width - resizeHandleSize / 2, shape.y + shape.height - resizeHandleSize / 2, resizeHandleSize, resizeHandleSize);
};

const onMouseDown = (event) => {
  const mousePos = getMousePos(event);
  
  if (isMouseInsideImage(mousePos)) {
    isDragging.value = true;
    offset.x = mousePos.x - imagePosition.value.x;
    offset.y = mousePos.y - imagePosition.value.y;
    return;
  }

  shapes.value.forEach(shape => {
    if (isMouseInsideRect(mousePos, shape)) {
      isDragging.value = true;
      currentShape.value = shape;
      offset.x = mousePos.x - shape.x;
      offset.y = mousePos.y - shape.y;
    } else if (isMouseInsideResizeHandle(mousePos, shape)) {
      isResizing.value = true;
      currentShape.value = shape;
    }
  });
};

const onMouseMove = (event) => {
  const mousePos = getMousePos(event);

  if (isDragging.value) {
    if (currentShape.value) {
      currentShape.value.x = mousePos.x - offset.x;
      currentShape.value.y = mousePos.y - offset.y;
    } else if (image.value) {
      imagePosition.value.x = mousePos.x - offset.x;
      imagePosition.value.y = mousePos.y - offset.y;
    }
    drawShapes();
  } else if (isResizing.value && currentShape.value) {
    const newWidth = mousePos.x - currentShape.value.x;
    const newHeight = mousePos.y - currentShape.value.y;
    
    if (newWidth > 0 && newHeight > 0) {
      currentShape.value.width = newWidth;
      currentShape.value.height = newHeight;
    }
    drawShapes();
  }
};

const onMouseUp = () => {
  isDragging.value = false;
  isResizing.value = false;
  currentShape.value = null;
};

const onMouseLeave = () => {
  isDragging.value = false;
  isResizing.value = false;
  currentShape.value = null;
};

const getMousePos = (event) => {
  const rect = canvas.value.getBoundingClientRect();
  return {
    x: event.clientX - rect.left,
    y: event.clientY - rect.top,
  };
};

const isMouseInsideRect = (mousePos, rect) => {
  return (
    mousePos.x >= rect.x &&
    mousePos.x <= rect.x + rect.width &&
    mousePos.y >= rect.y &&
    mousePos.y <= rect.y + rect.height
  );
};

const isMouseInsideResizeHandle = (mousePos, rect) => {
  return (
    mousePos.x >= rect.x + rect.width - resizeHandleSize &&
    mousePos.x <= rect.x + rect.width &&
    mousePos.y >= rect.y + rect.height - resizeHandleSize &&
    mousePos.y <= rect.y + rect.height
  );
};

const isMouseInsideImage = (mousePos) => {
  return (
    image.value &&
    mousePos.x >= imagePosition.value.x &&
    mousePos.x <= imagePosition.value.x + image.value.width &&
    mousePos.y >= imagePosition.value.y &&
    mousePos.y <= imagePosition.value.y + image.value.height
  );
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  background-color: #f9f9f9;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
}

.title {
  font-size: 24px;
  margin-bottom: 20px;
}

.controls {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.upload-input {
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 5px;
}

.button {
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px 15px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.button:hover {
  background-color: #0056b3;
}

.canvas {
  border: 1px solid #ccc;
  background-color: white;
  cursor: crosshair;
}
</style>
  