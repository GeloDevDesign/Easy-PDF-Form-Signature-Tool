<template>
  <div class="bg-dark min-vh-100 text-white pb-5 position-relative overflow-hidden d-flex flex-column font-sans">
    
    <div class="animation-layer">
      <div class="cute-file walker"><div class="face">📄</div></div>
      <div class="fight-scene">
        <div class="cute-file fighter fighter-1">📝</div>
        <div class="vs-spark">💥</div>
        <div class="cute-file fighter fighter-2">📉</div>
      </div>
    </div>

    <nav class="navbar navbar-expand-lg navbar-dark bg-secondary mb-4 shadow-lg" style="z-index: 20; backdrop-filter: blur(10px); background: rgba(50, 50, 50, 0.9) !important;">
      <div class="container">
        <span class="navbar-brand fw-bold d-flex align-items-center">
          <span class="me-2">✨</span> Vue PDF Editor
        </span>
        
        <div class="d-flex gap-2 align-items-center" v-if="pdfFile">
          <button 
            @click="toggleTextMode" 
            :class="['btn btn-sm d-flex align-items-center gap-2', isTextMode ? 'btn-info text-dark fw-bold' : 'btn-outline-light']">
            <span>{{ isTextMode ? 'Click PDF to Type' : 'Add Text' }}</span>
          </button>
          
          <div class="d-flex align-items-center gap-2 bg-dark rounded-pill px-3 py-1 border border-secondary" v-if="isTextMode || selectedTextIndex !== null">
            <input 
              type="number" 
              v-model.number="currentFontSize" 
              min="8" max="72" 
              class="form-control form-control-sm bg-transparent text-white border-0 p-0 text-center" 
              style="width: 40px;"
              @input="updateSelectedTextStyle"
              title="Font Size"
            />
            <div class="vr bg-secondary mx-1"></div>
            <select v-model="currentFontWeight" class="form-select form-select-sm bg-transparent text-white border-0 py-0" style="width: 80px;" @change="updateSelectedTextStyle">
              <option value="normal" class="text-dark">Normal</option>
              <option value="bold" class="text-dark">Bold</option>
              <option value="lighter" class="text-dark">Light</option>
            </select>
            <div class="vr bg-secondary mx-1"></div>
            <select v-model="currentFontStyle" class="form-select form-select-sm bg-transparent text-white border-0 py-0" style="width: 80px;" @change="updateSelectedTextStyle">
              <option value="normal" class="text-dark">Normal</option>
              <option value="italic" class="text-dark">Italic</option>
            </select>
          </div>
          
          <div class="btn-group">
            <button @click="showSignaturePad = true" class="btn btn-sm btn-warning">New Signature</button>
            <button @click="showSavedSignatures = true" class="btn btn-sm btn-outline-warning">
              📂 Saved ({{ savedSignatures.length }})
            </button>
          </div>
          
          <button @click="downloadPdf" class="btn btn-sm btn-success fw-bold px-4 shadow-sm">
            Download
          </button>
        </div>
      </div>
    </nav>

    <div 
      class="container d-flex flex-column align-items-center position-relative" 
      :class="{ 'justify-content-center flex-grow-1': !pdfFile }"
      style="z-index: 10;">
      
      <div v-if="!pdfFile" class="upload-box card bg-dark border-secondary text-center p-5 shadow-lg">
        <div class="card-body">
          <div class="mb-4 display-1">📄</div>
          <h4 class="card-title text-white mb-3">Upload Document</h4>
          <p class="card-text text-secondary mb-4">Select a PDF file to begin editing</p>
          <label class="btn btn-primary btn-lg px-5 rounded-pill shadow-sm">
            Choose File
            <input type="file" accept="application/pdf" @change="handleFileUpload" hidden />
          </label>
        </div>
      </div>

      <div v-else class="pdf-container position-relative shadow-lg" @click.self="deselectAll">
        <canvas ref="pdfCanvas" class="d-block"></canvas>

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
          }"
          @mousedown="startDragText($event, index)"
          @click.stop="selectText(index)">
          
          <div v-if="selectedTextIndex === index" class="floating-actions">
            <span class="drag-handle">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 9l-3 3 3 3M9 5l3-3 3 3M19 9l3 3-3 3M15 19l-3 3-3-3M9 9h6v6H9z"/></svg>
            </span>
            <button @click.stop="removeText(index)" class="action-btn delete" title="Remove">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
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
            zIndex: selectedSignature ? 1000 : 1
          }"
          @mousedown="startDragSig"
          @click.stop="selectSignature">
          
          <div v-if="selectedSignature" class="floating-actions">
            <span class="drag-handle">
               <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M5 9l-3 3 3 3M9 5l3-3 3 3M19 9l3 3-3 3M15 19l-3 3-3-3M9 9h6v6H9z"/></svg>
            </span>
            <button @click.stop="saveCurrentSigToStorage" class="action-btn save" title="Save to Library">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"></path><polyline points="17 21 17 13 7 13 7 21"></polyline><polyline points="7 3 7 8 15 8"></polyline></svg>
            </button>
            <button @click.stop="removeSignature" class="action-btn delete" title="Remove">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
            </button>
          </div>

          <img :src="signatureImage" class="w-100 h-100" style="pointer-events: none; user-select: none;" />
          
          <template v-if="selectedSignature">
            <div class="resize-handle se" @mousedown.stop="startResize($event, 'se')"></div>
            <div class="resize-handle ne" @mousedown.stop="startResize($event, 'ne')"></div>
            <div class="resize-handle sw" @mousedown.stop="startResize($event, 'sw')"></div>
            <div class="resize-handle nw" @mousedown.stop="startResize($event, 'nw')"></div>
          </template>
        </div>

      </div>
    </div>

    <div v-if="showSignaturePad" class="modal-overlay">
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 500px;">
        <div class="card-header border-secondary d-flex justify-content-between align-items-center">
          <h5 class="mb-0">✍️ Draw Signature</h5>
          <button @click="closeSignaturePad" class="btn-close btn-close-white"></button>
        </div>
        <div class="card-body bg-white text-center p-0 position-relative">
          <canvas 
            ref="sigCanvas" 
            width="498" height="200" 
            class="signature-pad-canvas"
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
      <div class="card bg-dark text-white border-secondary shadow-lg" style="width: 400px; max-height: 80vh;">
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
              class="saved-sig-item bg-white rounded p-2 position-relative group-hover">
              <img 
                :src="sig" 
                class="img-fluid" 
                style="height: 60px; object-fit: contain; cursor: pointer;"
                @click="useSavedSignature(sig)"
              />
              <button 
                @click.stop="deleteSavedSignature(idx)" 
                class="btn btn-danger btn-sm position-absolute top-0 end-0 m-1 p-0 rounded-circle d-flex align-items-center justify-content-center"
                style="width: 20px; height: 20px; font-size: 12px;">
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

// --- State Management ---
const pdfFile = ref(null);
const pdfBytes = ref(null);
const pdfCanvas = ref(null);
const isTextMode = ref(false);
const textElements = ref([]);
const scale = ref(1.5);

// Typography Defaults
const currentFontSize = ref(18);
const currentFontWeight = ref('normal');
const currentFontStyle = ref('normal');

// Selection State
const selectedTextIndex = ref(null);
const isDraggingText = ref(false);

// Signature State
const showSignaturePad = ref(false);
const showSavedSignatures = ref(false);
const savedSignatures = ref([]);
const sigCanvas = ref(null);
const signatureImage = ref(null); 
const sigPos = ref({ x: 100, y: 100 });
const sigSize = ref({ width: 200, height: 100 });
const selectedSignature = ref(false);
const isDraggingSig = ref(false);
const isResizingSig = ref(false);

// Canvas Contexts
let isDrawing = false;
let ctx = null;
let startPos = { x: 0, y: 0 };
let startSize = { width: 0, height: 0 };
let resizeDirection = '';

// --- Lifecycle ---
onMounted(() => {
  const stored = localStorage.getItem('my_signatures');
  if (stored) {
    savedSignatures.value = JSON.parse(stored);
  }
});

// --- 1. File Upload & Rendering ---
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
  const viewport = page.getViewport({ scale: scale.value });
  const canvas = pdfCanvas.value;
  const context = canvas.getContext('2d');
  canvas.height = viewport.height;
  canvas.width = viewport.width;
  await page.render({ canvasContext: context, viewport: viewport }).promise;
};

// --- 2. Text Logic ---
const toggleTextMode = () => {
  isTextMode.value = !isTextMode.value;
  if (isTextMode.value) deselectAll();
};

const handleCanvasClick = (e) => {
  if (!isTextMode.value) return;
  const rect = pdfCanvas.value.getBoundingClientRect();
  textElements.value.push({ 
    x: e.clientX - rect.left, 
    y: e.clientY - rect.top, 
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
  // Only start drag if clicking the text itself or the handle, not the input text
  // However, for simplicity, we allow drag on mousedown of container
  isDraggingText.value = true;
  selectedTextIndex.value = index;
  selectedSignature.value = false;
  
  const startX = e.clientX;
  const startY = e.clientY;
  const startLeft = textElements.value[index].x;
  const startTop = textElements.value[index].y;

  const onMouseMove = (moveEvent) => {
    textElements.value[index].x = startLeft + (moveEvent.clientX - startX);
    textElements.value[index].y = startTop + (moveEvent.clientY - startY);
  };
  
  const onMouseUp = () => {
    isDraggingText.value = false;
    window.removeEventListener('mousemove', onMouseMove);
    window.removeEventListener('mouseup', onMouseUp);
  };

  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);
};

const removeText = (index) => {
  textElements.value.splice(index, 1);
  selectedTextIndex.value = null;
};

// --- 3. Signature Logic ---
const startDrawing = (e) => {
  e.preventDefault();
  isDrawing = true;
  ctx = sigCanvas.value.getContext('2d');
  ctx.lineWidth = 3;
  ctx.lineCap = 'round';
  ctx.strokeStyle = '#000';
  const rect = sigCanvas.value.getBoundingClientRect();
  const x = (e.clientX || e.touches?.[0]?.clientX) - rect.left;
  const y = (e.clientY || e.touches?.[0]?.clientY) - rect.top;
  ctx.beginPath();
  ctx.moveTo(x, y);
};

const draw = (e) => {
  if (!isDrawing) return;
  e.preventDefault();
  const rect = sigCanvas.value.getBoundingClientRect();
  const x = (e.clientX || e.touches?.[0]?.clientX) - rect.left;
  const y = (e.clientY || e.touches?.[0]?.clientY) - rect.top;
  ctx.lineTo(x, y);
  ctx.stroke();
};

const stopDrawing = () => { 
  isDrawing = false; 
  if (ctx) ctx.beginPath(); 
};

const clearSignature = () => {
  const canvas = sigCanvas.value;
  canvas.getContext('2d').clearRect(0, 0, canvas.width, canvas.height);
};

const closeSignaturePad = () => {
  showSignaturePad.value = false;
  clearSignature();
};

const saveSignature = () => {
  const canvas = sigCanvas.value;
  // Simple check for empty canvas could be added here
  const dataUrl = canvas.toDataURL('image/png');
  
  // Save to storage
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
  // Reset pos for new signature
  sigPos.value = { x: 100, y: 100 };
  sigSize.value = { width: 200, height: 100 };
};

const saveCurrentSigToStorage = () => {
  if (signatureImage.value && !savedSignatures.value.includes(signatureImage.value)) {
    savedSignatures.value.push(signatureImage.value);
    localStorage.setItem('my_signatures', JSON.stringify(savedSignatures.value));
    alert('Signature saved to library!');
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

// Drag Signature
const startDragSig = (e) => {
  if (isResizingSig.value) return;
  e.preventDefault();
  isDraggingSig.value = true;
  selectedSignature.value = true;
  selectedTextIndex.value = null;
  
  const startX = e.clientX;
  const startY = e.clientY;
  const startLeft = sigPos.value.x;
  const startTop = sigPos.value.y;

  const onMouseMove = (moveEvent) => {
    sigPos.value.x = startLeft + (moveEvent.clientX - startX);
    sigPos.value.y = startTop + (moveEvent.clientY - startY);
  };
  
  const onMouseUp = () => {
    isDraggingSig.value = false;
    window.removeEventListener('mousemove', onMouseMove);
    window.removeEventListener('mouseup', onMouseUp);
  };

  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);
};

// Resize Signature
const startResize = (e, direction) => {
  e.preventDefault();
  e.stopPropagation();
  isResizingSig.value = true;
  resizeDirection = direction;
  startPos.x = e.clientX;
  startPos.y = e.clientY;
  startSize.width = sigSize.value.width;
  startSize.height = sigSize.value.height;
  const startPosX = sigPos.value.x;
  const startPosY = sigPos.value.y;

  const onMouseMove = (moveEvent) => {
    const deltaX = moveEvent.clientX - startPos.x;
    const deltaY = moveEvent.clientY - startPos.y;
    const aspectRatio = startSize.width / startSize.height;
    
    // Simple resizing logic (preserving aspect ratio roughly)
    if (direction.includes('e')) sigSize.value.width = Math.max(50, startSize.width + deltaX);
    if (direction.includes('w')) {
      sigSize.value.width = Math.max(50, startSize.width - deltaX);
      sigPos.value.x = startPosX + (startSize.width - sigSize.value.width);
    }
    sigSize.value.height = sigSize.value.width / aspectRatio;
    
    // Vertical adjustments
    if (direction.includes('n')) sigPos.value.y = startPosY + (startSize.height - sigSize.value.height);
  };
  
  const onMouseUp = () => {
    isResizingSig.value = false;
    window.removeEventListener('mousemove', onMouseMove);
    window.removeEventListener('mouseup', onMouseUp);
  };

  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);
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
          y: height - (item.y / scale.value) - (item.fontSize / scale.value * 0.8), // Adjusted baseline
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
/* GENERAL UTILS */
.pdf-container {
  background-color: #525659;
  border: 1px solid #444;
}

/* UPLOAD BOX STYLES */
.upload-box {
  transition: transform 0.2s, box-shadow 0.2s;
  border-style: dashed !important;
  border-width: 2px;
}
.upload-box:hover {
  transform: translateY(-5px);
  box-shadow: 0 10px 20px rgba(0,0,0,0.3) !important;
  border-color: #0d6efd !important;
}

/* ANIMATIONS */
.animation-layer {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  pointer-events: none; z-index: 0; opacity: 0.3;
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
  30% { left: 80%; top: 30%; transform: rotate(10deg); }
  60% { left: 20%; top: 80%; transform: rotate(-5deg); }
  100% { left: 10%; top: 10%; transform: rotate(0deg); }
}
@keyframes fightLeft { 0% { transform: translateX(0) rotate(-10deg); } 100% { transform: translateX(20px) rotate(10deg); } }
@keyframes fightRight { 0% { transform: translateX(0) rotate(10deg); } 100% { transform: translateX(-20px) rotate(-10deg); } }
@keyframes spark { 0% { opacity: 0; scale: 1; } 50% { opacity: 1; scale: 1.5; } 100% { opacity: 0; scale: 1; } }

/* ELEMENT WRAPPERS (Text & Signature) */
.element-wrapper {
  position: absolute;
  cursor: grab;
  /* Invisible border by default to reserve space */
  border: 2px solid transparent; 
  transition: border-color 0.2s;
}

.element-wrapper:active {
  cursor: grabbing;
}

.element-wrapper.is-selected {
  border: 2px dashed #0d6efd;
  background-color: rgba(13, 110, 253, 0.05);
}

/* FLOATING ACTION BAR */
.floating-actions {
  position: absolute;
  top: -35px;
  left: 50%;
  transform: translateX(-50%);
  background: #333;
  color: white;
  border-radius: 8px;
  padding: 4px 8px;
  display: flex;
  gap: 8px;
  align-items: center;
  box-shadow: 0 4px 6px rgba(0,0,0,0.2);
  z-index: 1050;
  white-space: nowrap;
}

/* Little triangle pointing down */
.floating-actions::after {
  content: '';
  position: absolute;
  bottom: -4px;
  left: 50%;
  transform: translateX(-50%);
  border-left: 5px solid transparent;
  border-right: 5px solid transparent;
  border-top: 5px solid #333;
}

.action-btn {
  background: none;
  border: none;
  color: #fff;
  padding: 4px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.2s;
}

.action-btn:hover { background: rgba(255,255,255,0.2); }
.action-btn.delete:hover { background: #dc3545; }
.action-btn.save:hover { background: #198754; }

.drag-handle {
  cursor: grab;
  color: #aaa;
  padding: 0 4px;
  border-right: 1px solid #555;
  display: flex;
  align-items: center;
}

/* INPUT STYLING */
.bare-input {
  background: transparent;
  border: none;
  color: #000;
  padding: 4px;
  outline: none;
  width: 100%;
}

/* RESIZE HANDLES */
.resize-handle {
  position: absolute;
  width: 10px; height: 10px;
  background: white;
  border: 1px solid #0d6efd;
  border-radius: 50%;
  z-index: 1001;
}
.resize-handle.se { bottom: -5px; right: -5px; cursor: se-resize; }
.resize-handle.sw { bottom: -5px; left: -5px; cursor: sw-resize; }
.resize-handle.ne { top: -5px; right: -5px; cursor: ne-resize; }
.resize-handle.nw { top: -5px; left: -5px; cursor: nw-resize; }

/* MODAL OVERLAY */
.modal-overlay {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.6);
  display: flex; justify-content: center; align-items: center;
  z-index: 2000;
  backdrop-filter: blur(2px);
}

.signature-pad-canvas { cursor: crosshair; background: #fff; }

.saved-sig-item {
  border: 1px solid #dee2e6;
  transition: border-color 0.2s, transform 0.2s;
}
.saved-sig-item:hover {
  border-color: #0d6efd;
  transform: scale(1.02);
}
</style>