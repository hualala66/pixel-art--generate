<template>
  <div class="studio-shell">
    <aside class="tool-panel" aria-label="Pixel generator controls">
      <header class="brand-bar">
        <div class="brand-mark" aria-hidden="true">
          <Sparkles :size="22" />
        </div>
        <div>
          <p class="eyebrow">Pixel Workbench</p>
          <h1>Pixel Art Generator</h1>
        </div>
      </header>

      <section class="panel-section">
        <div class="section-heading">
          <FileImage :size="18" />
          <span>Image Input</span>
        </div>

        <input
          ref="fileInput"
          class="file-input"
          type="file"
          accept="image/*"
          @change="handleImageUpload"
        />
        <button
          type="button"
          class="upload-dropzone"
          :class="{ 'is-dragging': isDragging, 'has-file': imageLoaded }"
          @click="openFileDialog"
          @dragover.prevent="isDragging = true"
          @dragleave.prevent="isDragging = false"
          @drop.prevent="handleDrop"
        >
          <img
            v-if="imageLoaded && uploadedPreviewSrc"
            class="upload-preview"
            :src="uploadedPreviewSrc"
            :alt="`${fileName} original preview`"
          />
          <UploadCloud v-else :size="30" />
          <span class="upload-title">{{ imageLoaded ? fileName : 'Drop or upload' }}</span>
          <span class="upload-copy">
            {{ imageLoaded ? imageMetaLabel : 'PNG, JPG, WebP supported' }}
          </span>
        </button>
      </section>

      <section class="panel-section">
        <div class="section-heading">
          <WandSparkles :size="18" />
          <span>Local AI Cutout</span>
        </div>

        <label class="switch-row" :class="{ 'is-disabled': !imageLoaded || isProcessingBg }">
          <input
            type="checkbox"
            v-model="settings.removeBackground"
            :disabled="!imageLoaded || isProcessingBg"
            @change="processImage"
          />
          <span class="switch-track" aria-hidden="true"></span>
          <span>
            <strong>{{ settings.removeBackground ? 'Enabled' : 'Disabled' }}</strong>
            <small>{{ isProcessingBg ? 'Loading model and cutting out' : 'Transparent pixels are skipped' }}</small>
          </span>
        </label>
      </section>

      <section class="panel-section controls-stack" :class="{ 'is-locked': !imageLoaded }">
        <div class="section-heading">
          <SlidersHorizontal :size="18" />
          <span>Pixel Settings</span>
        </div>

        <label class="range-control">
          <span>
            <strong>Tiles X</strong>
            <em>{{ settings.tilesX }} tiles</em>
          </span>
          <input
            type="range"
            min="10"
            max="150"
            v-model.number="settings.tilesX"
            :disabled="!imageLoaded"
            @input="renderCanvas"
          />
        </label>

        <label class="range-control">
          <span>
            <strong>Contrast Threshold</strong>
            <em>{{ settings.contrast > 0 ? '+' : '' }}{{ settings.contrast }}</em>
          </span>
          <input
            type="range"
            min="-100"
            max="100"
            v-model.number="settings.contrast"
            :disabled="!imageLoaded"
            @input="renderCanvas"
          />
        </label>

        <div class="shape-control" role="group" aria-label="Pixel shape">
          <span class="shape-label">
            <Shapes :size="16" />
            Pixel Shape
          </span>
          <div class="segmented">
            <button
              v-for="option in shapeOptions"
              :key="option.value"
              type="button"
              :disabled="!imageLoaded"
              :class="{ active: settings.shape === option.value }"
              @click="setShape(option.value)"
            >
              <component :is="option.icon" :size="16" />
              {{ option.label }}
            </button>
          </div>
        </div>

        <div class="option-control" role="group" aria-label="Export background">
          <span class="shape-label">
            <Download :size="16" />
            Export Background
          </span>
          <div class="segmented">
            <button
              v-for="option in exportBackgroundOptions"
              :key="option.value"
              type="button"
              :disabled="!imageLoaded"
              :class="{ active: settings.exportBackground === option.value }"
              @click="setExportBackground(option.value)"
            >
              {{ option.label }}
            </button>
          </div>
        </div>
      </section>

      <div v-if="errorMessage" class="status-card is-error" role="alert">
        <AlertTriangle :size="18" />
        <span>{{ errorMessage }}</span>
      </div>

      <div v-else class="status-card">
        <component :is="statusIcon" :size="18" :class="{ spin: isProcessingBg || isImageLoading }" />
        <span>{{ statusText }}</span>
      </div>

      <button
        type="button"
        class="export-btn"
        :disabled="!canExport"
        @click="downloadImage"
      >
        <component :is="exportIcon" :size="19" :class="{ spin: isProcessingBg || isImageLoading }" />
        {{ exportLabel }}
      </button>
    </aside>

    <main class="preview-stage" aria-label="Pixel preview">
      <div class="stage-header">
        <div>
          <p class="eyebrow">Live Preview</p>
          <h2>{{ imageLoaded ? fileName : 'Waiting for image' }}</h2>
        </div>
        <div class="stage-readout" aria-live="polite">
          {{ readoutText }}
        </div>
      </div>

      <div
        class="canvas-frame"
        :class="{ 'is-empty': !imageLoaded }"
        :style="{ '--tile-count': settings.tilesX }"
      >
        <div class="ruler ruler-x" aria-hidden="true"></div>
        <div class="ruler ruler-y" aria-hidden="true"></div>
        <canvas ref="mainCanvas" width="600" height="600"></canvas>
        <div v-if="!imageLoaded" class="placeholder">
          <ImagePlus :size="42" />
          <strong>Drop an image into the control panel</strong>
          <span>The pixel preview will update in real time.</span>
        </div>
        <div v-if="isProcessingBg || isImageLoading" class="processing-overlay">
          <LoaderCircle :size="28" class="spin" />
          <span>{{ isImageLoading ? 'Loading image' : 'Processing locally' }}</span>
        </div>
      </div>
    </main>
  </div>
</template>

<script setup>
import { computed, reactive, ref } from 'vue';
import { removeBackground } from '@imgly/background-removal';
import {
  AlertTriangle,
  CheckCircle2,
  Circle,
  Download,
  FileImage,
  ImagePlus,
  LoaderCircle,
  Shapes,
  SlidersHorizontal,
  Sparkles,
  Square,
  UploadCloud,
  WandSparkles
} from 'lucide-vue-next';

const CANVAS_SIZE = 600;

const mainCanvas = ref(null);
const fileInput = ref(null);
const imageLoaded = ref(false);
const isImageLoading = ref(false);
const isProcessingBg = ref(false);
const isDragging = ref(false);
const errorMessage = ref('');
const fileName = ref('');
const uploadedPreviewSrc = ref('');
const imageMeta = reactive({
  width: 0,
  height: 0,
  size: 0
});
const lastExportedAt = ref(null);

const settings = reactive({
  removeBackground: false,
  tilesX: 64,
  contrast: 0,
  shape: 'square',
  exportBackground: 'transparent'
});

const shapeOptions = [
  { value: 'square', label: 'Square', icon: Square },
  { value: 'circle', label: 'Dot', icon: Circle }
];

const exportBackgroundOptions = [
  { value: 'transparent', label: 'Transparent' },
  { value: 'preview', label: 'Preview BG' }
];

let originalImage = null;
let workingImage = null;
let workingObjectUrl = '';
let processVersion = 0;
let exportTimer = null;

const imageMetaLabel = computed(() => {
  const dimensions = imageMeta.width && imageMeta.height
    ? `${imageMeta.width} x ${imageMeta.height}`
    : 'Image loaded';
  const size = imageMeta.size ? ` / ${formatFileSize(imageMeta.size)}` : '';
  return `${dimensions}${size}`;
});

const canExport = computed(() => imageLoaded.value && !isImageLoading.value && !isProcessingBg.value);

const exportLabel = computed(() => {
  if (isImageLoading.value || isProcessingBg.value) return 'Processing';
  if (!imageLoaded.value) return 'Upload an image first';
  if (lastExportedAt.value) return 'PNG exported';
  return 'Export PNG';
});

const exportIcon = computed(() => {
  if (isImageLoading.value || isProcessingBg.value) return LoaderCircle;
  if (lastExportedAt.value) return CheckCircle2;
  return Download;
});

const statusText = computed(() => {
  if (isImageLoading.value) return 'Reading image file';
  if (isProcessingBg.value) return 'AI cutout is running locally';
  if (!imageLoaded.value) return 'Waiting for upload';
  if (lastExportedAt.value) return `Exported at ${formatTime(lastExportedAt.value)}`;
  return 'Parameter changes update the preview instantly';
});

const statusIcon = computed(() => {
  if (isImageLoading.value || isProcessingBg.value) return LoaderCircle;
  if (lastExportedAt.value) return CheckCircle2;
  return Sparkles;
});

const readoutText = computed(() => {
  if (!imageLoaded.value) return '600 x 600 canvas';
  return `${settings.tilesX} x ${settings.tilesX} / ${settings.shape === 'square' ? 'Square' : 'Dot'}`;
});

const openFileDialog = () => {
  fileInput.value?.click();
};

const handleImageUpload = (event) => {
  const [file] = event.target.files;
  processSelectedFile(file);
  event.target.value = '';
};

const handleDrop = (event) => {
  isDragging.value = false;
  const [file] = event.dataTransfer.files;
  processSelectedFile(file);
};

const processSelectedFile = async (file) => {
  if (!file) return;
  clearError();

  if (!file.type.startsWith('image/')) {
    errorMessage.value = 'Choose a PNG, JPG, WebP, or another image file.';
    return;
  }

  isImageLoading.value = true;
  imageLoaded.value = false;
  lastExportedAt.value = null;
  processVersion += 1;

  try {
    const dataUrl = await readFileAsDataURL(file);
    const image = await loadImage(dataUrl);
    originalImage = image;
    workingImage = image;
    uploadedPreviewSrc.value = dataUrl;
    fileName.value = file.name;
    imageMeta.width = image.naturalWidth || image.width;
    imageMeta.height = image.naturalHeight || image.height;
    imageMeta.size = file.size;
    imageLoaded.value = true;
    await processImage();
  } catch (error) {
    console.error('Image loading failed:', error);
    resetImageState();
    errorMessage.value = 'The image could not be loaded. Try another file.';
  } finally {
    isImageLoading.value = false;
  }
};

const processImage = async () => {
  if (!originalImage) return;

  const version = ++processVersion;
  clearError();
  lastExportedAt.value = null;

  if (!settings.removeBackground) {
    isProcessingBg.value = false;
    workingImage = originalImage;
    renderCanvas();
    return;
  }

  isProcessingBg.value = true;

  try {
    const imageBlob = await removeBackground(originalImage.src);
    if (version !== processVersion) return;

    revokeWorkingObjectUrl();
    workingObjectUrl = URL.createObjectURL(imageBlob);
    workingImage = await loadImage(workingObjectUrl);
    renderCanvas();
  } catch (error) {
    console.error('Background removal failed:', error);
    if (version === processVersion) {
      settings.removeBackground = false;
      workingImage = originalImage;
      errorMessage.value = 'AI cutout failed. The original preview was restored.';
      renderCanvas();
    }
  } finally {
    if (version === processVersion) {
      isProcessingBg.value = false;
    }
  }
};

const setShape = (shapeName) => {
  settings.shape = shapeName;
  lastExportedAt.value = null;
  renderCanvas();
};

const setExportBackground = (backgroundName) => {
  settings.exportBackground = backgroundName;
  lastExportedAt.value = null;
};

const renderCanvas = () => {
  if (!imageLoaded.value || !mainCanvas.value || !workingImage) return;

  const canvas = mainCanvas.value;
  const ctx = canvas.getContext('2d', { willReadFrequently: true });

  drawPixelArt(ctx, { showBackground: true, showGrid: true });
};

const drawPixelArt = (ctx, { showBackground, showGrid }) => {
  const canvas = ctx.canvas;
  canvas.width = CANVAS_SIZE;
  canvas.height = CANVAS_SIZE;

  ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
  drawImageContained(ctx, workingImage);

  const imageData = ctx.getImageData(0, 0, CANVAS_SIZE, CANVAS_SIZE);
  const data = imageData.data;

  ctx.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  const tileSize = CANVAS_SIZE / settings.tilesX;

  if (showBackground) {
    ctx.fillStyle = '#EEF4FB';
    ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
  }

  if (showGrid) {
    drawCanvasGrid(ctx, tileSize);
  }

  ctx.fillStyle = '#0B0D12';

  for (let y = 0; y < settings.tilesX; y += 1) {
    for (let x = 0; x < settings.tilesX; x += 1) {
      const px = Math.floor(x * tileSize + tileSize / 2);
      const py = Math.floor(y * tileSize + tileSize / 2);
      const index = (py * CANVAS_SIZE + px) * 4;

      const r = data[index];
      const g = data[index + 1];
      const b = data[index + 2];
      const a = data[index + 3];

      if (a < 8) continue;

      let brightness = (0.299 * r) + (0.587 * g) + (0.114 * b);
      brightness += settings.contrast;
      brightness = Math.max(0, Math.min(255, brightness));

      const sizeRatio = 1 - (brightness / 255);
      const drawSize = tileSize * sizeRatio;

      if (drawSize <= 0.5) continue;

      const drawX = x * tileSize + (tileSize - drawSize) / 2;
      const drawY = y * tileSize + (tileSize - drawSize) / 2;

      if (settings.shape === 'square') {
        ctx.fillRect(drawX, drawY, drawSize, drawSize);
      } else {
        ctx.beginPath();
        ctx.arc(
          x * tileSize + tileSize / 2,
          y * tileSize + tileSize / 2,
          drawSize / 2,
          0,
          Math.PI * 2
        );
        ctx.fill();
      }
    }
  }
};

const drawCanvasGrid = (ctx, tileSize) => {
  ctx.save();
  ctx.strokeStyle = 'rgba(18, 18, 18, 0.12)';
  ctx.lineWidth = Math.max(0.35, Math.min(0.75, tileSize * 0.08));

  for (let position = 0; position <= CANVAS_SIZE + 0.1; position += tileSize) {
    const aligned = Math.round(position) + 0.5;
    ctx.beginPath();
    ctx.moveTo(aligned, 0);
    ctx.lineTo(aligned, CANVAS_SIZE);
    ctx.stroke();

    ctx.beginPath();
    ctx.moveTo(0, aligned);
    ctx.lineTo(CANVAS_SIZE, aligned);
    ctx.stroke();
  }

  ctx.restore();
};

const downloadImage = () => {
  if (!canExport.value || !workingImage) return;

  const exportCanvas = document.createElement('canvas');
  const exportCtx = exportCanvas.getContext('2d', { willReadFrequently: true });
  const preservePreviewBackground = settings.exportBackground === 'preview';

  drawPixelArt(exportCtx, {
    showBackground: preservePreviewBackground,
    showGrid: preservePreviewBackground
  });

  const link = document.createElement('a');
  link.download = buildExportName();
  link.href = exportCanvas.toDataURL('image/png');
  link.click();

  lastExportedAt.value = new Date();
  window.clearTimeout(exportTimer);
  exportTimer = window.setTimeout(() => {
    lastExportedAt.value = null;
  }, 2800);
};

const drawImageContained = (ctx, image) => {
  const width = image.naturalWidth || image.width;
  const height = image.naturalHeight || image.height;
  const scale = Math.min(CANVAS_SIZE / width, CANVAS_SIZE / height);
  const drawWidth = width * scale;
  const drawHeight = height * scale;
  const offsetX = (CANVAS_SIZE - drawWidth) / 2;
  const offsetY = (CANVAS_SIZE - drawHeight) / 2;

  ctx.drawImage(image, offsetX, offsetY, drawWidth, drawHeight);
};

const readFileAsDataURL = (file) => new Promise((resolve, reject) => {
  const reader = new FileReader();
  reader.onload = () => resolve(reader.result);
  reader.onerror = () => reject(reader.error);
  reader.readAsDataURL(file);
});

const loadImage = (src) => new Promise((resolve, reject) => {
  const image = new Image();
  image.onload = () => resolve(image);
  image.onerror = reject;
  image.src = src;
});

const formatFileSize = (bytes) => {
  if (bytes < 1024 * 1024) return `${Math.round(bytes / 1024)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
};

const formatTime = (date) => date.toLocaleTimeString('en-US', {
  hour: '2-digit',
  minute: '2-digit',
  second: '2-digit'
});

const buildExportName = () => {
  const baseName = fileName.value.replace(/\.[^.]+$/, '') || 'pixel-art';
  return `${baseName}-pixel.png`;
};

const clearError = () => {
  errorMessage.value = '';
};

const resetImageState = () => {
  imageLoaded.value = false;
  originalImage = null;
  workingImage = null;
  fileName.value = '';
  uploadedPreviewSrc.value = '';
  imageMeta.width = 0;
  imageMeta.height = 0;
  imageMeta.size = 0;
  revokeWorkingObjectUrl();
};

const revokeWorkingObjectUrl = () => {
  if (workingObjectUrl) {
    URL.revokeObjectURL(workingObjectUrl);
    workingObjectUrl = '';
  }
};
</script>
