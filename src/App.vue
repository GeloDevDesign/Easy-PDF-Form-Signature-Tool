<template>
  <div class="bg-dark min-vh-100 text-white pb-5 position-relative d-flex flex-column font-sans app-container">
    
    <nav class="navbar navbar-expand-lg sticky-top p-3" style="background: transparent; pointer-events: none;">
      <div class="container-fluid shadow-lg rounded-pill px-4 py-2 d-flex align-items-center justify-content-between" 
           style="background: rgba(30, 30, 30, 0.95); backdrop-filter: blur(10px); border: 1px solid #444; pointer-events: auto; max-width: 1200px; margin: 0 auto;">
        
        <div class="d-flex align-items-center d-none d-md-flex">
          <div class="bg-primary rounded-circle d-flex align-items-center justify-content-center me-2" style="width: 32px; height: 32px;">
            <span>✨</span>
          </div>
          <span class="fw-bold tracking-tight">Easy PDF</span>
        </div>

        <div class="d-flex align-items-center gap-3 mx-auto">
          
          <div class="d-flex align-items-center bg-dark rounded-pill p-1 border border-secondary">
            <button 
              @click="toggleTextMode" 
              class="btn btn-sm rounded-pill px-3 transition-all"
              :class="isTextMode ? 'btn-light fw-bold' : 'btn-dark text-secondary hover-white'">
              <span v-if="!isTextMode">T</span>
              <span v-else>Tap PDF</span>
            </button>
            
            <div v-if="isTextMode || selectedTextIndex !== null" class="d-flex align-items-center px-2 animate-fade-in">
              <div class="vr bg-secondary mx-2" style="height: 15px;"></div>
              <button @click="adjustFontSize(-2)" class="btn btn-link text-white p-0 text-decoration-none hover-scale">➖</button>
              <span class="mx-2 small fw-mono text-muted" style="min-width: 20px; text-align: center;">{{ currentFontSize }}</span>
              <button @click="adjustFontSize(2)" class="btn btn-link text-white p-0 text-decoration-none hover-scale">➕</button>
              
              <div class="vr bg-secondary mx-2" style="height: 15px;"></div>
              <button @click="toggleBold" class="btn btn-sm p-0 fw-bold hover-scale" :class="currentFontWeight === 'bold' ? 'text-info' : 'text-secondary'">B</button>
              <span class="mx-1"></span>
              <button @click="toggleItalic" class="btn btn-sm p-0 fst-italic hover-scale" :class="currentFontStyle === 'italic' ? 'text-info' : 'text-secondary'">I</button>
            </div>
          </div>

          <div class="d-flex gap-2">
            <button 
              @click="openSignaturePad" 
              class="btn btn-sm btn-warning rounded-pill px-3 d-flex align-items-center gap-2 shadow-sm hover-lift">
              <span>✍️</span>
              <span class="d-none d-sm-inline fw-bold">Sign</span>
            </button>
            
            <button 
              @click="showSavedSignatures = true" 
              class="btn btn-sm btn-outline-secondary rounded-pill px-3 d-flex align-items-center gap-2 hover-white"
              title="Open Saved Signatures">
              <span>📂</span>
              <span class="d-none d-sm-inline">Library</span>
              <span class="badge bg-dark border border-secondary rounded-circle ms-1" style="font-size: 10px;">{{ savedSignatures.length }}</span>
            </button>
          </div>

        </div>

        <div>
          <button 
            @click="downloadPdf" 
            class="btn btn-success rounded-pill px-4 fw-bold shadow-lg hover-lift d-flex align-items-center gap-2"
            :disabled="!pdfFile">
            <span>💾</span>
            <span class="d-none d-md-inline">Download</span>
          </button>
        </div>

      </div>
    </nav>

    <div 
      ref="mainContainer"
      class="container-fluid d-flex flex-column align-items-center position-relative p-0" 
      :class="{ 'justify-content-center flex-grow-1': !pdfFile }"
      style="z-index: 10;">
      
      <div v-if="!pdfFile" class="upload-box card bg-dark border-secondary text-center p-5 shadow-lg mx-3" style="max-width: 500px; border-radius: 20px;">
        <div class="card-body">
          <div class="mb-4 p-3 bg-secondary bg-opacity-10 rounded-circle d-inline-block">
            <span class="display-4">📄</span>
          </div>
          <h3 class="card-title text-white mb-2 fw-bold">Upload PDF</h3>
          <p class="card-text text-secondary mb-4">Tap the button below to start editing</p>
          <label class="btn btn-primary btn-lg px-5 rounded-pill shadow-lg hover-lift">
            Select Document
            <input type="file" accept="application/pdf" @change="handleFileUpload" hidden />
          </label>
        </div>
      </div>

      <div v-else class="pdf-wrapper shadow-2xl" @click.self="deselectAll">
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
            touchAction: 'none',
            maxWidth: '90%'
          }"
          @mousedown="startDragText($event, index)"
          @touchstart="startDragText($event, index)"
          @click.stop="selectText(index)">
          
          <div class="dynamic-input-container">
             <span class="text-measurer" :style="{
                fontSize: item.fontSize + 'px',
                fontWeight: item.fontWeight,
                fontStyle: item.fontStyle,
             }">{{ item.text || 'Type...' }}</span>
             
             <input 
              v-model="item.text"
              :ref="el => { if (index === textElements.length - 1) el?.focus() }"
              class="bare-input"
              :style="{ 
                fontSize: item.fontSize + 'px',
                fontWeight: item.fontWeight,
                fontStyle: item.fontStyle,
              }"
              placeholder="Type..."
              @focus="selectText(index)"
            />
          </div>
          
          <div v-if="selectedTextIndex === index" class="position-absolute start-50 translate-middle-x" style="top: -35px;">
            <button @click.stop="removeText(index)" class="btn btn-dark btn-sm rounded-pill px-3 shadow-sm border border-secondary text-danger">
              Delete
            </button>
          </div>

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
          
          <div v-if="selectedSignature" class="floating-toolbar" @mousedown.stop @touchstart.stop>
            <button @click.stop="removeSignature" class="btn btn-sm btn-danger py-1 px-3 rounded-pill shadow-sm">Delete</button>
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
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 95%; max-width: 500px; border-radius: 15px;">
        <div class="card-header border-secondary d-flex justify-content-between align-items-center py-3">
          <h5 class="mb-0 fw-bold">✍️ Draw Signature</h5>
          <button @click="closeSignaturePad" class="btn-close btn-close-white"></button>
        </div>
        
        <div class="card-body bg-white text-center p-0 position-relative overflow-hidden" style="height: 250px;" ref="sigPadContainer">
          <canvas 
            ref="sigCanvas" 
            class="d-block w-100 h-100"
            style="touch-action: none; cursor: crosshair;"
            @mousedown="startDrawing" @mousemove="draw" @mouseup="stopDrawing" @mouseleave="stopDrawing"
            @touchstart="startDrawing" @touchmove="draw" @touchend="stopDrawing">
          </canvas>
          <div class="position-absolute bottom-0 start-0 w-100 border-top text-muted small py-1" style="background: rgba(240,240,240,0.8); pointer-events: none;">
            Sign inside the box
          </div>
        </div>
        
        <div class="card-footer border-secondary d-flex justify-content-between py-3">
          <button @click="clearSignature" class="btn btn-outline-light rounded-pill px-4">Clear</button>
          <button @click="saveSignature" class="btn btn-primary rounded-pill px-5 shadow-sm">Use Signature</button>
        </div>
      </div>
    </div>

    <div v-if="showSavedSignatures" class="modal-overlay" @click.self="showSavedSignatures = false">
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 90%; max-width: 400px; max-height: 80vh; border-radius: 15px;">
        <div class="card-header border-secondary d-flex justify-content-between align-items-center py-3">
          <h5 class="mb-0 fw-bold">📂 Saved Signatures</h5>
          <button @click="showSavedSignatures = false" class="btn-close btn-close-white"></button>
        </div>
        <div class="card-body overflow-auto p-3">
          <div v-if="savedSignatures.length === 0" class="text-center text-muted py-5">
            <div class="mb-2" style="font-size: 2rem; opacity: 0.5;">📭</div>
            <p>No saved signatures yet.</p>
          </div>
          <div class="d-grid gap-3">
            <div 
              v-for="(sig, idx) in savedSignatures" 
              :key="idx" 
              class="saved-sig-item bg-white rounded p-3 position-relative shadow-sm transition-all"
              @click="useSavedSignature(sig)">
              <img 
                :src="sig.data" 
                class="img-fluid" 
                style="height: 60px; object-fit: contain; display: block; margin: 0 auto;"
              />
              <div class="position-absolute top-0 start-0 w-100 h-100 d-flex align-items-center justify-content-center opacity-0 hover-opacity-100" style="background: rgba(13, 110, 253, 0.1); cursor: pointer;">
                <span class="badge bg-primary">Tap to use</span>
              </div>
              <button 
                @click.stop="deleteSavedSignature(idx)" 
                class="btn btn-danger btn-sm position-absolute top-0 end-0 m-2 p-0 rounded-circle d-flex align-items-center justify-content-center shadow-sm"
                style="width: 24px; height: 24px; font-size: 14px; z-index: 10;">
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
import { ref, nextTick, onMounted } from 'vue';
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
const sigPadContainer = ref(null);
const signatureImage = ref(null); 
const sigPos = ref({ x: 50, y: 50 });
const sigSize = ref({ width: 150, height: 75 });
const selectedSignature = ref(false);
const isDraggingSig = ref(false);
const isResizingSig = ref(false);

// Drawing State
let isDrawing = false;
let canvasRect = null;
let ctx = null;
let startPos = { x: 0, y: 0 };
let startSize = { width: 0, height: 0 };
let resizeDirection = '';

onMounted(() => {
  const stored = localStorage.getItem('my_signatures');
  if (stored) savedSignatures.value = JSON.parse(stored);
});

// --- HELPER: Get Client Coordinates ---
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
  const viewport = page.getViewport({ scale: 1.5 });

  // Responsive Scaling Logic
  const screenWidth = window.innerWidth;
  let finalScale = 1.5;
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

// --- 2. Text Logic (Dynamic Size) ---
const toggleTextMode = () => {
  isTextMode.value = !isTextMode.value;
  if (isTextMode.value) deselectAll();
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

const adjustFontSize = (amount) => {
  if (selectedTextIndex.value !== null) {
    const item = textElements.value[selectedTextIndex.value];
    let newSize = item.fontSize + amount;
    if (newSize < 8) newSize = 8;
    if (newSize > 72) newSize = 72;
    item.fontSize = newSize;
    currentFontSize.value = newSize;
  } else {
    let newSize = currentFontSize.value + amount;
    if (newSize < 8) newSize = 8;
    if (newSize > 72) newSize = 72;
    currentFontSize.value = newSize;
  }
};

const toggleBold = () => {
  if (selectedTextIndex.value !== null) {
    const item = textElements.value[selectedTextIndex.value];
    item.fontWeight = item.fontWeight === 'bold' ? 'normal' : 'bold';
    currentFontWeight.value = item.fontWeight;
  } else {
    currentFontWeight.value = currentFontWeight.value === 'bold' ? 'normal' : 'bold';
  }
};

const toggleItalic = () => {
  if (selectedTextIndex.value !== null) {
    const item = textElements.value[selectedTextIndex.value];
    item.fontStyle = item.fontStyle === 'italic' ? 'normal' : 'italic';
    currentFontStyle.value = item.fontStyle;
  } else {
    currentFontStyle.value = currentFontStyle.value === 'italic' ? 'normal' : 'italic';
  }
};

const startDragText = (e, index) => {
  if (e.type === 'touchstart') e.preventDefault();
  isDraggingText.value = true;
  selectedTextIndex.value = index;
  selectedSignature.value = false;
  selectText(index);
  
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

// --- 3. Optimized Snappy Signature Logic ---
const openSignaturePad = async () => {
  showSignaturePad.value = true;
  await nextTick();
  if (sigCanvas.value && sigPadContainer.value) {
    sigCanvas.value.width = sigPadContainer.value.offsetWidth;
    sigCanvas.value.height = sigPadContainer.value.offsetHeight;
    clearSignature();
  }
};

const startDrawing = (e) => {
  e.preventDefault(); 
  isDrawing = true;
  const canvas = sigCanvas.value;
  ctx = canvas.getContext('2d');
  ctx.lineWidth = 3;
  ctx.lineCap = 'round';
  ctx.lineJoin = 'round';
  ctx.strokeStyle = '#000';
  
  canvasRect = canvas.getBoundingClientRect();
  const { x, y } = getDrawPos(e);
  ctx.beginPath();
  ctx.moveTo(x, y);
};

const draw = (e) => {
  if (!isDrawing) return;
  e.preventDefault(); 
  const { x, y } = getDrawPos(e);
  ctx.lineTo(x, y);
  ctx.stroke();
};

const stopDrawing = () => { 
  if (!isDrawing) return;
  isDrawing = false;
  ctx.closePath();
};

const getDrawPos = (e) => {
  const pos = getClientPos(e);
  return {
    x: pos.x - canvasRect.left,
    y: pos.y - canvasRect.top
  };
};

const clearSignature = () => {
  if (!sigCanvas.value) return;
  const ctx = sigCanvas.value.getContext('2d');
  ctx.clearRect(0, 0, sigCanvas.value.width, sigCanvas.value.height);
};

const closeSignaturePad = () => {
  showSignaturePad.value = false;
};

const saveSignature = () => {
  const canvas = sigCanvas.value;
  const pixelBuffer = new Uint32Array(
    canvas.getContext('2d').getImageData(0, 0, canvas.width, canvas.height).data.buffer
  );
  if (!pixelBuffer.some(color => color !== 0)) {
     alert('Please draw a signature first');
     return;
  }
  
  const dataUrl = canvas.toDataURL('image/png');
  const ratio = canvas.width / canvas.height;
  
  const sigObject = { data: dataUrl, ratio: ratio };
  savedSignatures.value.push(sigObject);
  localStorage.setItem('my_signatures', JSON.stringify(savedSignatures.value));
  
  useSavedSignature(sigObject);
  closeSignaturePad();
};

const useSavedSignature = (sigObject) => {
  const imgData = typeof sigObject === 'string' ? sigObject : sigObject.data;
  const ratio = typeof sigObject === 'string' ? 2 : (sigObject.ratio || 2);
  
  signatureImage.value = imgData;
  selectedSignature.value = true;
  selectedTextIndex.value = null;
  showSavedSignatures.value = false;
  
  sigPos.value = { x: 50, y: 100 };
  const baseWidth = 200;
  sigSize.value = { 
    width: baseWidth, 
    height: baseWidth / ratio 
  };
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
    
    const ratio = startSize.width / startSize.height;
    
    if (direction.includes('e')) {
      const newWidth = Math.max(50, startSize.width + deltaX);
      sigSize.value.width = newWidth;
      sigSize.value.height = newWidth / ratio;
    }
    if (direction.includes('w')) {
      const newWidth = Math.max(50, startSize.width - deltaX);
      sigSize.value.width = newWidth;
      sigSize.value.height = newWidth / ratio;
      sigPos.value.x = startPosX + (startSize.width - newWidth);
    }
    if (direction.includes('n')) {
      sigPos.value.y = startPosY + (startSize.height - sigSize.value.height);
    }
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
.app-container { touch-action: pan-y; }

/* PDF WRAPPER */
.pdf-wrapper {
  background-color: #525659;
  border: 1px solid #444;
  position: relative;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

/* UPLOAD BOX */
.upload-box {
  transition: transform 0.2s;
  border-style: dashed !important;
  border-width: 2px;
}
.upload-box:active { transform: scale(0.98); }

/* CUSTOM UTILITIES */
.hover-scale:hover { transform: scale(1.1); transition: transform 0.2s; }
.hover-lift:hover { transform: translateY(-2px); transition: transform 0.2s; }
.hover-white:hover { color: white !important; }
.transition-all { transition: all 0.2s; }

/* ELEMENT WRAPPER */
.element-wrapper {
  display: inline-block;
  transition: border-color 0.1s;
}
.element-wrapper.is-selected {
  border: 1px dashed #0d6efd;
  background-color: rgba(13, 110, 253, 0.05);
}

/* DYNAMIC INPUT STYLING */
.dynamic-input-container {
  display: grid;
  align-items: center;
}
.dynamic-input-container::after {
  content: attr(data-value) " ";
  visibility: hidden;
  white-space: pre;
}
.text-measurer {
  grid-area: 1 / 1;
  visibility: hidden;
  white-space: pre;
  pointer-events: none;
  padding: 4px;
  min-width: 40px;
}
.bare-input {
  grid-area: 1 / 1;
  background: transparent;
  border: none;
  color: #000;
  padding: 4px;
  outline: none;
  width: 100%;
  height: 100%;
  min-width: 40px;
  overflow: hidden;
}

/* FLOATING TOOLBAR */
.floating-toolbar {
  position: absolute;
  top: -45px;
  left: 50%;
  transform: translateX(-50%);
  background: #222;
  border-radius: 8px;
  padding: 6px 10px;
  display: flex;
  gap: 5px;
  align-items: center;
  box-shadow: 0 4px 12px rgba(0,0,0,0.4);
  z-index: 2000;
  white-space: nowrap;
}
.floating-toolbar::after {
  content: ''; position: absolute; bottom: -6px; left: 50%; transform: translateX(-50%);
  border-left: 6px solid transparent; border-right: 6px solid transparent; border-top: 6px solid #222;
}

/* RESIZE HANDLES */
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
  background: rgba(0,0,0,0.85);
  display: flex; justify-content: center; align-items: center;
  z-index: 3000;
  backdrop-filter: blur(5px);
}
.saved-sig-item { border: 1px solid #dee2e6; cursor: pointer; }
.saved-sig-item:hover { border-color: #0d6efd; transform: scale(1.02); }
.hover-opacity-100:hover { opacity: 1 !important; }
</style>