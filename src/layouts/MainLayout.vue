<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title>
          Quasar PDF Viewer
        </q-toolbar-title>

        <!-- Add the file input button -->
        <q-btn
          flat
          dense
          round
          icon="upload"
          aria-label="Upload PDF"
          @click="triggerFileInput"
        />
        <input
          type="file"
          ref="fileInput"
          accept="application/pdf"
          @change="handleFileUpload"
          style="display: none;"
        />
        
        <!-- Add input box -->
        <q-input
          filled
          v-model="inputText"
          label="Enter text"
          class="q-ml-md"
          maxlength="100"
          style="background-color: #f0f0f0;"
        />

        <!-- Add "Click here" button -->
        <q-btn
          flat
          dense
          label="Click here"
          class="q-ml-md"
          @click="handleClickHere"
        />
      </q-toolbar>
    </q-header>

    <!-- Display the uploaded PDF -->
    <q-page-container>
      <q-page class="q-pa-none">
        <div v-if="pdfUrl" class="pdf-container">
          <canvas ref="pdfCanvas" class="pdf-canvas"></canvas>
        </div>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script>
import * as pdfjsLib from 'pdfjs-dist';
import 'pdfjs-dist/web/pdf_viewer.css';

export default {
  data() {
    return {
      pdfUrl: null,
      inputText: ''
    };
  },
  methods: {
    triggerFileInput() {
      this.$refs.fileInput.click();
    },
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file && file.type === 'application/pdf') {
        this.pdfUrl = URL.createObjectURL(file);
        this.renderPDF();
      } else {
        alert('Please upload a valid PDF file.');
      }
    },
    async renderPDF() {
      try {
        const loadingTask = pdfjsLib.getDocument(this.pdfUrl);
        const pdf = await loadingTask.promise;
        const page = await pdf.getPage(1);
        const viewport = page.getViewport({ scale: 1.5 });
        const canvas = this.$refs.pdfCanvas;
        const context = canvas.getContext('2d');
        canvas.height = viewport.height;
        canvas.width = viewport.width;

        const renderContext = {
          canvasContext: context,
          viewport: viewport
        };
        await page.render(renderContext).promise;
      } catch (error) {
        console.error('Error rendering PDF:', error);
      }
    },
    handleClickHere() {
      if (this.inputText) {
        this.highlightText(this.inputText);
      }
    },
    async highlightText(text) {
      try {
        const loadingTask = pdfjsLib.getDocument(this.pdfUrl);
        const pdf = await loadingTask.promise;
        const page = await pdf.getPage(1);
        const textContent = await page.getTextContent();
        const canvas = this.$refs.pdfCanvas;
        const context = canvas.getContext('2d');

        textContent.items.forEach(item => {
          if (item.str.includes(text)) {
            const { transform, width, height } = item;
            const [x, y] = transform.slice(4, 6);
            context.fillStyle = 'yellow';
            context.fillRect(x, canvas.height - y, width, height);
          }
        });
      } catch (error) {
        console.error('Error highlighting text:', error);
      }
    }
  }
};
</script>

<style>
.pdf-container {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
}

.pdf-canvas {
  width: 100%;
  height: 100%;
  border: none;
}
</style>