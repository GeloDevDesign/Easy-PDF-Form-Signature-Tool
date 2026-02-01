<template>
  <div class="bg-dark min-vh-100 text-white pb-5 position-relative overflow-hidden d-flex flex-column font-sans app-container">
    
    <div class="animation-layer">
      <div class="cute-file walker"><div class="face">📄</div></div>
      <div class="fight-scene">
        <div class="cute-file fighter fighter-1">📝</div>
        <div class="vs-spark">💥</div>
        <div class="cute-file fighter fighter-2">📉</div>
      </div>
    </div>

    <nav class="navbar navbar-expand-lg navbar-dark bg-secondary mb-4 shadow-lg sticky-top" style="backdrop-filter: blur(10px); background: rgba(50, 50, 50, 0.95) !important;">
      <div class="container-fluid px-3">
        <div class="d-flex justify-content-between w-100 align-items-center mb-2" v-if="pdfFile">
             <span class="navbar-brand fw-bold d-flex align-items-center m-0" style="font-size: 1rem;">
              <span class="me-2">✨</span> PDF Editor
            </span>
             <button @click="downloadPdf" class="btn btn-sm btn-success fw-bold px-3 shadow-sm d-md-none">
              Save
            </button>
        </div>
        <span class="navbar-brand fw-bold d-none d-md-flex align-items-center" v-else>
          <span class="me-2">✨</span> Vue PDF Editor
        </span>
        
        <div class="d-flex flex-wrap gap-2 align-items-center justify-content-center w-100" v-if="pdfFile">
          
          <button 
            @click="toggleTextMode" 
            :class="['btn btn-sm d-flex align-items-center gap-2', isTextMode ? 'btn-info text-dark fw-bold' : 'btn-outline-light']">
            <span>{{ isTextMode ? 'Tap to Type' : 'Add Text' }}</span>
          </button>
          
          <div class="d-flex align-items-center gap-1 bg-dark rounded px-2 py-1 border border-secondary" v-if="isTextMode || selectedTextIndex !== null">
            <input 
              type="number" 
              v-model.number="currentFontSize" 
              min="8" max="72" 
              class="form-control form-control-sm bg-transparent text-white border-0 p-0 text-center" 
              style="width: 35px;"
              @input="updateSelectedTextStyle"
            />
            <div class="vr bg-secondary mx-1"></div>
            <button @click="toggleBold" class="btn btn-sm p-0 text-white" :class="{ 'text-info': currentFontWeight === 'bold' }">B</button>
            <div class="vr bg-secondary mx-1"></div>
            <button @click="toggleItalic" class="btn btn-sm p-0 text-white" :class="{ 'text-info': currentFontStyle === 'italic' }">I</button>
          </div>
          
          <div class="btn-group">
            <button @click="showSignaturePad = true" class="btn btn-sm btn-warning">Sign</button>
            <button @click="showSavedSignatures = true" class="btn btn-sm btn-outline-warning">
              Saved ({{ savedSignatures.length }})
            </button>
          </div>
          
          <button @click="downloadPdf" class="btn btn-sm btn-success fw-bold px-4 shadow-sm d-none d-md-block">
            Download PDF
          </button>
        </div>
      </div>
    </nav>

    <div 
      ref="mainContainer"
      class="container-fluid d-flex flex-column align-items-center position-relative p-0" 
      :class="{ 'justify-content-center flex-grow-1': !pdfFile }"
      style="z-index: 10;">
      
      <div v-if="!pdfFile" class="upload-box card bg-dark border-secondary text-center p-4 shadow-lg mx-3" style="max-width: 500px;">
        <div class="card-body">
          <div class="mb-3 display-1">📄</div>
          <h4 class="card-title text-white mb-2">Upload Document</h4>
          <p class="card-text text-secondary mb-4 small">Tap below to select a PDF</p>
          <label class="btn btn-primary btn-lg px-5 rounded-pill shadow-sm w-100">
            Choose File
            <input type="file" accept="application/pdf" @change="handleFileUpload" hidden />
          </label>
        </div>
      </div>

      <div v-else class="pdf-wrapper shadow-lg" @click.self="deselectAll">
        <canvas ref="pdfCanvas" class="d-block mx-auto"></canvas>

        <div 
          class="position-absolute top-0 start-0 w-100 h-100" 
          :style="{ cursor: isTextMode ? 'text' : 'default' }"
          @click="handleCanvasClick"
          v-if="isTextMode">
        </div>

        <div 
          v-for="(item, index) in textElements" 
          :key="'text-' + index"
          class="element-wrapper position-absolute"
          :class="{ 'is-selected': selectedTextIndex === index }"
          :style="{ 
            left: item.x + 'px', 
            top: item.y + 'px',
            zIndex: selectedTextIndex === index ? 1000 : 1,
            touchAction: 'none'
          }"
          @mousedown="startDragText($event, index)"
          @touchstart="startDragText($event, index)"
          @click.stop="selectText(index)">
          
          <div v-if="selectedTextIndex === index" class="floating-actions">
            <span class="drag-handle">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 9l-3 3 3 3M9 5l3-3 3 3M19 9l3 3-3 3M15 19l-3 3-3-3M9 9h6v6H9z"/></svg>
            </span>
            <button @click.stop="removeText(index)" class="action-btn delete">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
            </button>
          </div>

          <input 
            v-model="item.text"
            :ref="el => { if (index === textElements.length - 1) el?.focus() }"
            class="bare-input"
            :style="{ 
              fontSize: item.fontSize + 'px',
              fontWeight: item.fontWeight,
              fontStyle: item.fontStyle,
              minWidth: '50px'
            }"
            placeholder="Type..."
            @focus="selectText(index)"
          />
        </div>

        <div 
          v-if="signatureImage" 
          class="element-wrapper position-absolute"
          :class="{ 'is-selected': selectedSignature }"
          :style="{ 
            left: sigPos.x + 'px', 
            top: sigPos.y + 'px', 
            width: sigSize.width + 'px',
            height: sigSize.height + 'px',
            zIndex: selectedSignature ? 1000 : 1,
            touchAction: 'none'
          }"
          @mousedown="startDragSig"
          @touchstart="startDragSig"
          @click.stop="selectSignature">
          
          <div v-if="selectedSignature" class="floating-actions">
            <span class="drag-handle">
               <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 9l-3 3 3 3M9 5l3-3 3 3M19 9l3 3-3 3M15 19l-3 3-3-3M9 9h6v6H9z"/></svg>
            </span>
            <button @click.stop="saveCurrentSigToStorage" class="action-btn save">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"></path><polyline points="17 21 17 13 7 13 7 21"></polyline><polyline points="7 3 7 8 15 8"></polyline></svg>
            </button>
            <button @click.stop="removeSignature" class="action-btn delete">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
            </button>
          </div>

          <img :src="signatureImage" class="w-100 h-100" style="pointer-events: none; user-select: none;" />
          
          <template v-if="selectedSignature">
            <div class="resize-handle se" @mousedown.stop="startResize($event, 'se')" @touchstart.stop="startResize($event, 'se')"></div>
            <div class="resize-handle ne" @mousedown.stop="startResize($event, 'ne')" @touchstart.stop="startResize($event, 'ne')"></div>
            <div class="resize-handle sw" @mousedown.stop="startResize($event, 'sw')" @touchstart.stop="startResize($event, 'sw')"></div>
            <div class="resize-handle nw" @mousedown.stop="startResize($event, 'nw')" @touchstart.stop="startResize($event, 'nw')"></div>
          </template>
        </div>

      </div>
    </div>

    <div v-if="showSignaturePad" class="modal-overlay">
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 95%; max-width: 500px;">
        <div class="card-header border-secondary d-flex justify-content-between align-items-center">
          <h5 class="mb-0">✍️ Draw Signature</h5>
          <button @click="closeSignaturePad" class="btn-close btn-close-white"></button>
        </div>
        <div class="card-body bg-white text-center p-0 position-relative">
          <canvas 
            ref="sigCanvas" 
            width="498" height="200" 
            class="signature-pad-canvas w-100"
            style="touch-action: none;"
            @mousedown="startDrawing" @mousemove="draw" @mouseup="stopDrawing" @mouseleave="stopDrawing"
            @touchstart="startDrawing" @touchmove="draw" @touchend="stopDrawing">
          </canvas>
        </div>
        <div class="card-footer border-secondary d-flex justify-content-between">
          <button @click="clearSignature" class="btn btn-outline-light btn-sm">Clear</button>
          <button @click="saveSignature" class="btn btn-primary btn-sm px-4">Use & Save</button>
        </div>
      </div>
    </div>

    <div v-if="showSavedSignatures" class="modal-overlay" @click.self="showSavedSignatures = false">
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 90%; max-width: 400px; max-height: 80vh;">
        <div class="card-header border-secondary d-flex justify-content-between align-items-center">
          <h5 class="mb-0">📂 My Signatures</h5>
          <button @click="showSavedSignatures = false" class="btn-close btn-close-white"></button>
        </div>
        <div class="card-body overflow-auto p-3">
          <div v-if="savedSignatures.length === 0" class="text-center text-muted py-4">
            <p>No saved signatures yet.</p>
          </div>
          <div class="d-grid gap-3">
            <div 
              v-for="(sig, idx) in savedSignatures" 
              :key="idx" 
              class="saved-sig-item bg-white rounded p-2 position-relative">
              <img 
                :src="sig" 
                class="img-fluid" 
                style="height: 60px; object-fit: contain; cursor: pointer; display: block; margin: 0 auto;"
                @click="useSavedSignature(sig)"
              />
              <button 
                @click.stop="deleteSavedSignature(idx)" 
                class="btn btn-danger btn-sm position-absolute top-0 end-0 m-1 p-0 rounded-circle d-flex align-items-center justify-content-center shadow-sm"
                style="width: 24px; height: 24px; font-size: 14px;">
                ×
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, watch, onMounted } from 'vue';
import { PDFDocument, rgb, StandardFonts } from 'pdf-lib';
import * as pdfjsLib from 'pdfjs-dist';

// --- Configuration ---
pdfjsLib.GlobalWorkerOptions.workerSrc = `https://unpkg.com/pdfjs-dist@${pdfjsLib.version}/build/pdf.worker.min.mjs`;

// --- State ---
const mainContainer = ref(null);
const pdfFile = ref(null);
const pdfBytes = ref(null);
const pdfCanvas = ref(null);
const isTextMode = ref(false);
const textElements = ref([]);
const scale = ref(1.5);

// Typography
const currentFontSize = ref(18);
const currentFontWeight = ref('normal');
const currentFontStyle = ref('normal');

// Selection
const selectedTextIndex = ref(null);
const isDraggingText = ref(false);

// Signature
const showSignaturePad = ref(false);
const showSavedSignatures = ref(false);
const savedSignatures = ref([]);
const sigCanvas = ref(null);
const signatureImage = ref(null); 
const sigPos = ref({ x: 50, y: 50 });
const sigSize = ref({ width: 150, height: 75 });
const selectedSignature = ref(false);
const isDraggingSig = ref(false);
const isResizingSig = ref(false);

// Drawing State
let isDrawing = false;
let points = [];
let canvasRect = null;
let ctx = null;
let startPos = { x: 0, y: 0 };
let startSize = { width: 0, height: 0 };
let resizeDirection = '';

onMounted(() => {
  const stored = localStorage.getItem('my_signatures');
  if (stored) savedSignatures.value = JSON.parse(stored);
});

// --- HELPER: Get Client Coordinates (Touch/Mouse) ---
const getClientPos = (e) => {
  return {
    x: e.clientX || e.touches?.[0]?.clientX,
    y: e.clientY || e.touches?.[0]?.clientY
  };
};

// --- 1. File Upload & Responsive Rendering ---
const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;
  const arrayBuffer = await file.arrayBuffer();
  pdfBytes.value = arrayBuffer;
  pdfFile.value = file;
  renderPdf(arrayBuffer.slice(0));
};

const renderPdf = async (buffer) => {
  const loadingTask = pdfjsLib.getDocument({ data: buffer });
  const pdf = await loadingTask.promise;
  const page = await pdf.getPage(1);
  const viewport = page.getViewport({ scale: 1.5 }); // Base viewport

  // Responsive Scaling Logic
  const screenWidth = window.innerWidth;
  let finalScale = 1.5;
  
  // If mobile, scale to fit width (minus padding)
  if (screenWidth < 800) {
    const desiredWidth = screenWidth - 20; 
    const baseViewport = page.getViewport({ scale: 1.0 });
    finalScale = desiredWidth / baseViewport.width;
  }
  
  scale.value = finalScale;
  const finalViewport = page.getViewport({ scale: finalScale });

  const canvas = pdfCanvas.value;
  const context = canvas.getContext('2d');
  canvas.height = finalViewport.height;
  canvas.width = finalViewport.width;
  
  await page.render({ canvasContext: context, viewport: finalViewport }).promise;
};

// --- 2. Text Logic ---
const toggleTextMode = () => {
  isTextMode.value = !isTextMode.value;
  if (isTextMode.value) deselectAll();
};

const toggleBold = () => {
  currentFontWeight.value = currentFontWeight.value === 'bold' ? 'normal' : 'bold';
  updateSelectedTextStyle();
};

const toggleItalic = () => {
  currentFontStyle.value = currentFontStyle.value === 'italic' ? 'normal' : 'italic';
  updateSelectedTextStyle();
};

const handleCanvasClick = (e) => {
  if (!isTextMode.value) return;
  const rect = pdfCanvas.value.getBoundingClientRect();
  const clientPos = getClientPos(e);
  
  textElements.value.push({ 
    x: clientPos.x - rect.left, 
    y: clientPos.y - rect.top, 
    text: '',
    fontSize: currentFontSize.value,
    fontWeight: currentFontWeight.value,
    fontStyle: currentFontStyle.value
  });
  selectedTextIndex.value = textElements.value.length - 1;
  isTextMode.value = false;
};

const selectText = (index) => {
  selectedTextIndex.value = index;
  selectedSignature.value = false;
  const item = textElements.value[index];
  currentFontSize.value = item.fontSize;
  currentFontWeight.value = item.fontWeight;
  currentFontStyle.value = item.fontStyle;
};

const updateSelectedTextStyle = () => {
  if (selectedTextIndex.value !== null) {
    const item = textElements.value[selectedTextIndex.value];
    item.fontSize = currentFontSize.value;
    item.fontWeight = currentFontWeight.value;
    item.fontStyle = currentFontStyle.value;
  }
};

const startDragText = (e, index) => {
  // Prevent default only if it's touch to stop scrolling
  if (e.type === 'touchstart') e.preventDefault();
  
  isDraggingText.value = true;
  selectedTextIndex.value = index;
  selectedSignature.value = false;
  
  const pos = getClientPos(e);
  const startX = pos.x;
  const startY = pos.y;
  const startLeft = textElements.value[index].x;
  const startTop = textElements.value[index].y;

  const onMove = (moveEvent) => {
    if (moveEvent.type === 'touchmove') moveEvent.preventDefault();
    const movePos = getClientPos(moveEvent);
    textElements.value[index].x = startLeft + (movePos.x - startX);
    textElements.value[index].y = startTop + (movePos.y - startY);
  };
  
  const onEnd = () => {
    isDraggingText.value = false;
    window.removeEventListener('mousemove', onMove);
    window.removeEventListener('mouseup', onEnd);
    window.removeEventListener('touchmove', onMove);
    window.removeEventListener('touchend', onEnd);
  };

  window.addEventListener('mousemove', onMove);
  window.addEventListener('mouseup', onEnd);
  window.addEventListener('touchmove', onMove, { passive: false });
  window.addEventListener('touchend', onEnd);
};

const removeText = (index) => {
  textElements.value.splice(index, 1);
  selectedTextIndex.value = null;
};

// --- 3. Optimized Signature Logic (Smooth & Touch) ---
const startDrawing = (e) => {
  e.preventDefault(); // Stop scrolling
  isDrawing = true;
  points = [];
  
  const canvas = sigCanvas.value;
  ctx = canvas.getContext('2d');
  
  // DPI Scaling for sharpness on mobile
  const dpr = window.devicePixelRatio || 1;
  if (canvas.width !== canvas.offsetWidth * dpr) {
    canvas.width = canvas.offsetWidth * dpr;
    canvas.height = canvas.offsetHeight * dpr;
    ctx.scale(dpr, dpr);
  }

  ctx.lineWidth = 3;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.strokeStyle = '#000';
  ctx.shadowBlur = 1;
  ctx.shadowColor = '#000';

  canvasRect = canvas.getBoundingClientRect();
  
  const { x, y } = getDrawPos(e);
  points.push({ x, y });
  
  ctx.beginPath();
  ctx.moveTo(x, y);
};

const draw = (e) => {
  if (!isDrawing) return;
  e.preventDefault(); 
  
  const { x, y } = getDrawPos(e);
  points.push({ x, y });

  // Quadratic Curve Smoothing
  if (points.length > 2) {
    const lastPoint = points[points.length - 2];
    const p1 = points[points.length - 3];
    const midPoint = {
      x: (lastPoint.x + p1.x) / 2,
      y: (lastPoint.y + p1.y) / 2
    };

    ctx.beginPath();
    ctx.moveTo(midPoint.x, midPoint.y);
    ctx.quadraticCurveTo(lastPoint.x, lastPoint.y, x, y);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(x, y);
  }
};

const stopDrawing = () => { 
  if (!isDrawing) return;
  isDrawing = false;
  if (points.length > 0) ctx.stroke();
  points = [];
};

const getDrawPos = (e) => {
  const pos = getClientPos(e);
  return {
    x: pos.x - canvasRect.left,
    y: pos.y - canvasRect.top
  };
};

const clearSignature = () => {
  const canvas = sigCanvas.value;
  const context = canvas.getContext('2d');
  context.save();
  context.setTransform(1, 0, 0, 1, 0, 0);
  context.clearRect(0, 0, canvas.width, canvas.height);
  context.restore();
};

const closeSignaturePad = () => {
  showSignaturePad.value = false;
  clearSignature();
};

const saveSignature = () => {
  const canvas = sigCanvas.value;
  // Simple empty check
  const pixelBuffer = new Uint32Array(
    canvas.getContext('2d').getImageData(0, 0, canvas.width, canvas.height).data.buffer
  );
  if (!pixelBuffer.some(color => color !== 0)) {
     alert('Please draw a signature first');
     return;
  }
  
  const dataUrl = canvas.toDataURL('image/png');
  savedSignatures.value.push(dataUrl);
  localStorage.setItem('my_signatures', JSON.stringify(savedSignatures.value));
  
  useSavedSignature(dataUrl);
  closeSignaturePad();
};

const useSavedSignature = (dataUrl) => {
  signatureImage.value = dataUrl;
  selectedSignature.value = true;
  selectedTextIndex.value = null;
  showSavedSignatures.value = false;
  // Reset pos centered on current view usually, but fixed for now
  sigPos.value = { x: 50, y: 50 };
  sigSize.value = { width: 150, height: 75 };
};

const saveCurrentSigToStorage = () => {
  if (signatureImage.value && !savedSignatures.value.includes(signatureImage.value)) {
    savedSignatures.value.push(signatureImage.value);
    localStorage.setItem('my_signatures', JSON.stringify(savedSignatures.value));
  }
};

const deleteSavedSignature = (index) => {
  savedSignatures.value.splice(index, 1);
  localStorage.setItem('my_signatures', JSON.stringify(savedSignatures.value));
};

const selectSignature = () => {
  selectedSignature.value = true;
  selectedTextIndex.value = null;
  isTextMode.value = false;
};

const removeSignature = () => {
  signatureImage.value = null;
  selectedSignature.value = false;
};

// Drag Signature (Touch Enabled)
const startDragSig = (e) => {
  if (isResizingSig.value) return;
  if (e.type === 'touchstart') e.preventDefault();
  
  isDraggingSig.value = true;
  selectedSignature.value = true;
  selectedTextIndex.value = null;
  
  const pos = getClientPos(e);
  const startX = pos.x;
  const startY = pos.y;
  const startLeft = sigPos.value.x;
  const startTop = sigPos.value.y;

  const onMove = (moveEvent) => {
    if (moveEvent.type === 'touchmove') moveEvent.preventDefault();
    const movePos = getClientPos(moveEvent);
    sigPos.value.x = startLeft + (movePos.x - startX);
    sigPos.value.y = startTop + (movePos.y - startY);
  };
  
  const onEnd = () => {
    isDraggingSig.value = false;
    window.removeEventListener('mousemove', onMove);
    window.removeEventListener('mouseup', onEnd);
    window.removeEventListener('touchmove', onMove);
    window.removeEventListener('touchend', onEnd);
  };

  window.addEventListener('mousemove', onMove);
  window.addEventListener('mouseup', onEnd);
  window.addEventListener('touchmove', onMove, { passive: false });
  window.addEventListener('touchend', onEnd);
};

// Resize Signature (Touch Enabled)
const startResize = (e, direction) => {
  e.preventDefault(); 
  e.stopPropagation();
  isResizingSig.value = true;
  resizeDirection = direction;
  
  const pos = getClientPos(e);
  startPos.x = pos.x;
  startPos.y = pos.y;
  startSize.width = sigSize.value.width;
  startSize.height = sigSize.value.height;
  const startPosX = sigPos.value.x;
  const startPosY = sigPos.value.y;

  const onMove = (moveEvent) => {
    if (moveEvent.type === 'touchmove') moveEvent.preventDefault();
    const movePos = getClientPos(moveEvent);
    const deltaX = movePos.x - startPos.x;
    const aspectRatio = startSize.width / startSize.height;
    
    if (direction.includes('e')) sigSize.value.width = Math.max(50, startSize.width + deltaX);
    if (direction.includes('w')) {
      sigSize.value.width = Math.max(50, startSize.width - deltaX);
      sigPos.value.x = startPosX + (startSize.width - sigSize.value.width);
    }
    sigSize.value.height = sigSize.value.width / aspectRatio;
    if (direction.includes('n')) sigPos.value.y = startPosY + (startSize.height - sigSize.value.height);
  };
  
  const onEnd = () => {
    isResizingSig.value = false;
    window.removeEventListener('mousemove', onMove);
    window.removeEventListener('mouseup', onEnd);
    window.removeEventListener('touchmove', onMove);
    window.removeEventListener('touchend', onEnd);
  };

  window.addEventListener('mousemove', onMove);
  window.addEventListener('mouseup', onEnd);
  window.addEventListener('touchmove', onMove, { passive: false });
  window.addEventListener('touchend', onEnd);
};

const deselectAll = () => {
  selectedTextIndex.value = null;
  selectedSignature.value = false;
};

// --- 4. Download ---
const downloadPdf = async () => {
  if (!pdfBytes.value) return;
  try {
    const pdfBytesCopy = pdfBytes.value.slice(0);
    const pdfDoc = await PDFDocument.load(pdfBytesCopy);
    const pages = pdfDoc.getPages();
    const firstPage = pages[0];
    const { height } = firstPage.getSize();

    // Draw Text
    for (const item of textElements.value) {
      if (item.text?.trim()) {
        let font;
        if (item.fontWeight === 'bold' && item.fontStyle === 'italic') font = await pdfDoc.embedFont(StandardFonts.HelveticaBoldOblique);
        else if (item.fontWeight === 'bold') font = await pdfDoc.embedFont(StandardFonts.HelveticaBold);
        else if (item.fontStyle === 'italic') font = await pdfDoc.embedFont(StandardFonts.HelveticaOblique);
        else font = await pdfDoc.embedFont(StandardFonts.Helvetica);
        
        firstPage.drawText(item.text, {
          x: item.x / scale.value,
          y: height - (item.y / scale.value) - (item.fontSize / scale.value * 0.8),
          size: item.fontSize / scale.value,
          font: font,
          color: rgb(0, 0, 0),
        });
      }
    }

    // Draw Signature
    if (signatureImage.value) {
      const pngImage = await pdfDoc.embedPng(signatureImage.value);
      firstPage.drawImage(pngImage, {
        x: sigPos.value.x / scale.value,
        y: height - (sigPos.value.y / scale.value) - (sigSize.value.height / scale.value),
        width: sigSize.value.width / scale.value,
        height: sigSize.value.height / scale.value,
      });
    }

    const modifiedBytes = await pdfDoc.save();
    const blob = new Blob([modifiedBytes], { type: 'application/pdf' });
    const link = document.createElement('a');
    link.href = URL.createObjectURL(blob);
    link.download = 'edited_document.pdf';
    link.click();
    setTimeout(() => URL.revokeObjectURL(link.href), 100);
  } catch (error) {
    alert('Error saving PDF: ' + error.message);
  }
};
</script>

<style scoped>
/* APP CONTAINER */
.app-container {
  touch-action: pan-y; /* Allow vertical scroll on body, but handle specifics in JS */
}

/* PDF WRAPPER */
.pdf-wrapper {
  background-color: #525659;
  border: 1px solid #444;
  position: relative;
  overflow: hidden; /* Contains the dragging elements */
}

/* UPLOAD BOX */
.upload-box {
  transition: transform 0.2s;
  border-style: dashed !important;
  border-width: 2px;
}
.upload-box:active { transform: scale(0.98); }

/* ANIMATIONS */
.animation-layer {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  pointer-events: none; z-index: 0; opacity: 0.2;
}
.cute-file { position: absolute; font-size: 3rem; user-select: none; }
.walker { animation: roam 25s linear infinite; }
.fight-scene { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 150px; height: 100px; }
.fighter { transition: transform 0.1s; }
.fighter-1 { left: 0; animation: fightLeft 1s infinite alternate; }
.fighter-2 { right: 0; animation: fightRight 1s infinite alternate; }
.vs-spark { position: absolute; top: 0; left: 50%; transform: translateX(-50%); font-size: 2rem; animation: spark 0.5s infinite; }

@keyframes roam {
  0% { left: 10%; top: 10%; transform: rotate(0deg); }
  50% { left: 80%; top: 80%; transform: rotate(10deg); }
  100% { left: 10%; top: 10%; transform: rotate(0deg); }
}
@keyframes fightLeft { 0% { transform: translateX(0) rotate(-10deg); } 100% { transform: translateX(20px) rotate(10deg); } }
@keyframes fightRight { 0% { transform: translateX(0) rotate(10deg); } 100% { transform: translateX(-20px) rotate(-10deg); } }
@keyframes spark { 0% { opacity: 0; scale: 1; } 50% { opacity: 1; scale: 1.5; } 100% { opacity: 0; scale: 1; } }

/* INTERACTIVE ELEMENTS */
.element-wrapper {
  position: absolute;
  /* border: 2px solid transparent; */
  transition: border-color 0.1s;
}

.element-wrapper.is-selected {
  border: 1px dashed #0d6efd;
  background-color: rgba(13, 110, 253, 0.05);
}

/* ACTIONS BAR */
.floating-actions {
  position: absolute;
  top: -40px;
  left: 50%;
  transform: translateX(-50%);
  background: #222;
  border-radius: 8px;
  padding: 6px 10px;
  display: flex;
  gap: 12px;
  align-items: center;
  box-shadow: 0 4px 10px rgba(0,0,0,0.3);
  z-index: 2000;
  white-space: nowrap;
}
.floating-actions::after {
  content: ''; position: absolute; bottom: -5px; left: 50%; transform: translateX(-50%);
  border-left: 6px solid transparent; border-right: 6px solid transparent; border-top: 6px solid #222;
}

.action-btn {
  background: none; border: none; color: #fff;
  padding: 5px; border-radius: 4px;
  display: flex; align-items: center; justify-content: center;
}
.action-btn.delete { color: #ff6b6b; }
.action-btn.save { color: #51cf66; }
.drag-handle { color: #aaa; padding-right: 8px; border-right: 1px solid #444; }

.bare-input {
  background: transparent; border: none; color: #000;
  padding: 4px; outline: none; width: 100%;
}

/* RESIZE HANDLES - Bigger touch targets */
.resize-handle {
  position: absolute; width: 16px; height: 16px;
  background: white; border: 2px solid #0d6efd; border-radius: 50%; z-index: 1001;
}
.resize-handle.se { bottom: -8px; right: -8px; }
.resize-handle.sw { bottom: -8px; left: -8px; }
.resize-handle.ne { top: -8px; right: -8px; }
.resize-handle.nw { top: -8px; left: -8px; }

/* MODALS */
.modal-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.8);
  display: flex; justify-content: center; align-items: center;
  z-index: 3000;
  backdrop-filter: blur(3px);
}
.signature-pad-canvas { background: #fff; border-radius: 4px; }

.saved-sig-item { border: 1px solid #dee2e6; }
.saved-sig-item:active { background-color: #f8f9fa !important; }
</style>