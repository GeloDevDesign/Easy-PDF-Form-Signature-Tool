<template>
  <div class="bg-dark min-vh-100 text-white pb-5">
    
    <nav class="navbar navbar-dark bg-secondary mb-4 shadow-sm">
      <div class="container d-flex justify-content-between align-items-center">
        <span class="navbar-brand mb-0 h1 fw-bold">Vue PDF Editor</span>
        
        <div class="d-flex gap-2 align-items-center" v-if="pdfFile">
          <button 
            @click="toggleTextMode" 
            :class="['btn', isTextMode ? 'btn-info' : 'btn-outline-light']">
            {{ isTextMode ? 'Click PDF to Type' : 'Add Text' }}
          </button>
          
          <div class="d-flex align-items-center gap-2" v-if="isTextMode || selectedTextIndex !== null">
            <label class="text-white small mb-0">Size:</label>
            <input 
              type="number" 
              v-model.number="currentFontSize" 
              min="8" 
              max="72" 
              class="form-control form-control-sm" 
              style="width: 60px;"
              @input="updateSelectedTextStyle"
            />
            
            <label class="text-white small mb-0 ms-2">Weight:</label>
            <select 
              v-model="currentFontWeight" 
              class="form-select form-select-sm" 
              style="width: 100px;"
              @change="updateSelectedTextStyle">
              <option value="normal">Normal</option>
              <option value="bold">Bold</option>
              <option value="lighter">Light</option>
            </select>
            
            <label class="text-white small mb-0 ms-2">Style:</label>
            <select 
              v-model="currentFontStyle" 
              class="form-select form-select-sm" 
              style="width: 100px;"
              @change="updateSelectedTextStyle">
              <option value="normal">Normal</option>
              <option value="italic">Italic</option>
            </select>
          </div>
          
          <button @click="showSignaturePad = true" class="btn btn-warning">
            Add Signature
          </button>
          
          <button @click="downloadPdf" class="btn btn-success">
            Download PDF
          </button>
        </div>
      </div>
    </nav>

    <div class="container d-flex flex-column align-items-center">
      
      <div v-if="!pdfFile" class="card bg-dark border-secondary text-center p-5" style="width: 100%; max-width: 600px; border-style: dashed !important;">
        <div class="card-body">
          <h4 class="card-title text-muted mb-3">Upload Document</h4>
          <p class="card-text text-secondary mb-4">Select a PDF file to begin editing</p>
          <input type="file" accept="application/pdf" @change="handleFileUpload" class="form-control" />
        </div>
      </div>

      <div v-else class="position-relative shadow-lg border border-secondary" style="background-color: #525659;">
        <canvas ref="pdfCanvas" class="d-block"></canvas>

        <div 
          class="position-absolute top-0 start-0 w-100 h-100" 
          :style="{ cursor: isTextMode ? 'crosshair' : 'default' }"
          @click="handleCanvasClick"
          v-if="isTextMode">
        </div>

        <div 
          v-for="(item, index) in textElements" 
          :key="'text-' + index"
          class="position-absolute"
          :style="{ 
            left: item.x + 'px', 
            top: item.y + 'px',
            zIndex: selectedTextIndex === index ? 1000 : 1,
            cursor: isDraggingText && selectedTextIndex === index ? 'grabbing' : 'move'
          }"
          @mousedown="startDragText($event, index)"
          @click.stop="selectText(index)">
          
          <div class="d-flex align-items-center">
            <input 
              v-model="item.text"
              :ref="el => { if (index === textElements.length - 1) el?.focus() }"
              class="form-control form-control-sm border-0 bg-transparent text-dark"
              :style="{ 
                width: '200px', 
                fontSize: item.fontSize + 'px',
                fontWeight: item.fontWeight,
                fontStyle: item.fontStyle,
                padding: '2px', 
                boxShadow: selectedTextIndex === index ? '0 0 0 2px #0d6efd' : 'none',
                backgroundColor: selectedTextIndex === index ? 'rgba(13, 110, 253, 0.1)' : 'transparent',
                cursor: 'move',
                pointerEvents: 'auto'
              }"
              placeholder="Type here..."
              @focus="selectText(index)"
              @blur="deselectText"
            />
            <button 
              @click.stop="removeText(index)" 
              class="btn btn-sm btn-danger rounded-circle ms-1 py-0 px-1" 
              style="font-size: 10px; width: 20px; height: 20px;">
              ×
            </button>
          </div>
        </div>

        <div 
          v-if="signatureImage" 
          class="position-absolute"
          :class="{ 'border border-primary border-2': selectedSignature }"
          :style="{ 
            left: sigPos.x + 'px', 
            top: sigPos.y + 'px', 
            width: sigSize.width + 'px',
            height: sigSize.height + 'px',
            cursor: isDraggingSig ? 'grabbing' : 'grab',
            zIndex: selectedSignature ? 1000 : 1
          }"
          @mousedown="startDragSig"
          @click.stop="selectSignature">
          
          <img 
            :src="signatureImage" 
            class="w-100 h-100"
            style="pointer-events: none; user-select: none;"
          />
          
          <!-- Resize handles -->
          <template v-if="selectedSignature">
            <div 
              class="position-absolute bg-white border border-2 border-primary rounded-circle"
              style="width: 12px; height: 12px; bottom: -6px; right: -6px; cursor: se-resize; z-index: 1001;"
              @mousedown.stop="startResize($event, 'se')"
            ></div>
            <div 
              class="position-absolute bg-white border border-2 border-primary rounded-circle"
              style="width: 12px; height: 12px; top: -6px; right: -6px; cursor: ne-resize; z-index: 1001;"
              @mousedown.stop="startResize($event, 'ne')"
            ></div>
            <div 
              class="position-absolute bg-white border border-2 border-primary rounded-circle"
              style="width: 12px; height: 12px; bottom: -6px; left: -6px; cursor: sw-resize; z-index: 1001;"
              @mousedown.stop="startResize($event, 'sw')"
            ></div>
            <div 
              class="position-absolute bg-white border border-2 border-primary rounded-circle"
              style="width: 12px; height: 12px; top: -6px; left: -6px; cursor: nw-resize; z-index: 1001;"
              @mousedown.stop="startResize($event, 'nw')"
            ></div>
            
            <button 
              @click.stop="removeSignature" 
              class="btn btn-sm btn-danger rounded-circle position-absolute" 
              style="top: -10px; right: -10px; font-size: 10px; width: 20px; height: 20px; padding: 0;">
              ×
            </button>
          </template>
        </div>

      </div>

    </div>

    <div v-if="showSignaturePad" class="position-fixed top-0 start-0 w-100 h-100 d-flex justify-content-center align-items-center" style="background: rgba(0,0,0,0.8); z-index: 1050;">
      <div class="card bg-dark text-white border-secondary" style="width: 500px;">
        <div class="card-header border-secondary">
          <h5 class="mb-0">Draw Signature</h5>
        </div>
        <div class="card-body bg-white text-center p-0">
          <canvas 
            ref="sigCanvas" 
            width="498" 
            height="200" 
            style="cursor: crosshair;" 
            @mousedown="startDrawing" 
            @mousemove="draw" 
            @mouseup="stopDrawing" 
            @mouseleave="stopDrawing"
            @touchstart="startDrawing"
            @touchmove="draw"
            @touchend="stopDrawing">
          </canvas>
        </div>
        <div class="card-footer border-secondary d-flex justify-content-end gap-2">
          <button @click="clearSignature" class="btn btn-outline-light btn-sm">Clear</button>
          <button @click="closeSignaturePad" class="btn btn-secondary btn-sm">Cancel</button>
          <button @click="saveSignature" class="btn btn-primary btn-sm">Use Signature</button>
        </div>
      </div>
    </div>

  </div>
</template>

<script setup>
import { ref, watch } from 'vue';
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
const currentFontSize = ref(18);
const currentFontWeight = ref('normal');
const currentFontStyle = ref('normal');
const selectedTextIndex = ref(null);
const isDraggingText = ref(false);

// Signature State
const showSignaturePad = ref(false);
const sigCanvas = ref(null);
const signatureImage = ref(null); 
const sigPos = ref({ x: 50, y: 50 });
const sigSize = ref({ width: 200, height: 100 });
const selectedSignature = ref(false);
const isDraggingSig = ref(false);
const isResizingSig = ref(false);

let isDrawing = false;
let ctx = null;
let startPos = { x: 0, y: 0 };
let startSize = { width: 0, height: 0 };
let resizeDirection = '';

// --- 1. File Upload & Rendering ---
// --- 1. File Upload & Rendering ---
const handleFileUpload = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const arrayBuffer = await file.arrayBuffer();
  pdfBytes.value = arrayBuffer;
  pdfFile.value = file;

  // Create a copy (.slice(0)) of the buffer for the renderer.
  // This prevents PDF.js from "detaching" the original arrayBuffer,
  // keeping pdfBytes.value valid for the download function later.
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
  if (isTextMode.value) {
    selectedSignature.value = false;
    selectedTextIndex.value = null;
  }
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
  currentFontSize.value = textElements.value[index].fontSize;
  currentFontWeight.value = textElements.value[index].fontWeight;
  currentFontStyle.value = textElements.value[index].fontStyle;
};

const deselectText = () => {
  setTimeout(() => {
    if (selectedTextIndex.value !== null && !isTextMode.value) {
      selectedTextIndex.value = null;
    }
  }, 200);
};

const updateSelectedTextStyle = () => {
  if (selectedTextIndex.value !== null) {
    textElements.value[selectedTextIndex.value].fontSize = currentFontSize.value;
    textElements.value[selectedTextIndex.value].fontWeight = currentFontWeight.value;
    textElements.value[selectedTextIndex.value].fontStyle = currentFontStyle.value;
  }
};

const startDragText = (e, index) => {
  e.preventDefault();
  
  // Allow dragging from anywhere in the container
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
  if (selectedTextIndex.value === index) {
    selectedTextIndex.value = null;
  }
};

// --- 3. Signature Logic ---
const startDrawing = (e) => {
  e.preventDefault();
  isDrawing = true;
  ctx = sigCanvas.value.getContext('2d');
  ctx.lineWidth = 2;
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
  const tempCtx = canvas.getContext('2d');
  const imageData = tempCtx.getImageData(0, 0, canvas.width, canvas.height);
  const data = imageData.data;
  
  // Check if signature is empty
  let hasDrawing = false;
  for (let i = 3; i < data.length; i += 4) {
    if (data[i] > 0) {
      hasDrawing = true;
      break;
    }
  }
  
  if (!hasDrawing) {
    alert('Please draw a signature first');
    return;
  }
  
  signatureImage.value = canvas.toDataURL('image/png');
  showSignaturePad.value = false;
  selectedSignature.value = true;
  selectedTextIndex.value = null;
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

// Drag signature
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

// Resize signature
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
    
    switch (resizeDirection) {
      case 'se':
        sigSize.value.width = Math.max(50, startSize.width + deltaX);
        sigSize.value.height = sigSize.value.width / aspectRatio;
        break;
      case 'sw':
        sigSize.value.width = Math.max(50, startSize.width - deltaX);
        sigSize.value.height = sigSize.value.width / aspectRatio;
        sigPos.value.x = startPosX + (startSize.width - sigSize.value.width);
        break;
      case 'ne':
        sigSize.value.width = Math.max(50, startSize.width + deltaX);
        sigSize.value.height = sigSize.value.width / aspectRatio;
        sigPos.value.y = startPosY + (startSize.height - sigSize.value.height);
        break;
      case 'nw':
        sigSize.value.width = Math.max(50, startSize.width - deltaX);
        sigSize.value.height = sigSize.value.width / aspectRatio;
        sigPos.value.x = startPosX + (startSize.width - sigSize.value.width);
        sigPos.value.y = startPosY + (startSize.height - sigSize.value.height);
        break;
    }
  };
  
  const onMouseUp = () => {
    isResizingSig.value = false;
    window.removeEventListener('mousemove', onMouseMove);
    window.removeEventListener('mouseup', onMouseUp);
  };

  window.addEventListener('mousemove', onMouseMove);
  window.addEventListener('mouseup', onMouseUp);
};

// Deselect when clicking canvas
watch(isTextMode, (newVal) => {
  if (newVal) {
    selectedSignature.value = false;
  }
});

// --- 4. Save/Download Logic ---
const downloadPdf = async () => {
  if (!pdfBytes.value) return;
  
  try {
    // Create a copy of the ArrayBuffer to avoid detachment issues
    const pdfBytesCopy = pdfBytes.value.slice(0);
    const pdfDoc = await PDFDocument.load(pdfBytesCopy);
    const pages = pdfDoc.getPages();
    const firstPage = pages[0];
    const { height } = firstPage.getSize();

    // Draw Text - only if there are text elements with content
    for (const item of textElements.value) {
      if (item.text && item.text.trim()) {
        // Select appropriate font based on weight and style
        let font;
        if (item.fontWeight === 'bold' && item.fontStyle === 'italic') {
          font = await pdfDoc.embedFont(StandardFonts.HelveticaBoldOblique);
        } else if (item.fontWeight === 'bold') {
          font = await pdfDoc.embedFont(StandardFonts.HelveticaBold);
        } else if (item.fontStyle === 'italic') {
          font = await pdfDoc.embedFont(StandardFonts.HelveticaOblique);
        } else {
          font = await pdfDoc.embedFont(StandardFonts.Helvetica);
        }
        
        firstPage.drawText(item.text, {
          x: item.x / scale.value,
          y: height - (item.y / scale.value) - (item.fontSize / scale.value),
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
    
    // Cleanup
    setTimeout(() => URL.revokeObjectURL(link.href), 100);
  } catch (error) {
    console.error('Error saving PDF:', error);
    alert('Error saving PDF: ' + error.message);
  }
};
</script>