<template>
  <q-layout view="lHh Lpr lFf">
    <q-header elevated>
      <q-toolbar>
        <q-toolbar-title> Quasar PDF Viewer </q-toolbar-title>
        <!-- Add the file input button -->
        <q-btn flat dense round icon="upload" aria-label="Upload PDF" @click="triggerFileInput" />
        <input
          type="file"
          ref="fileInput"
          accept="application/pdf"
          @change="handleFileUpload"
          style="display: none"
        />
        <!-- Add input box -->
        <q-input
          filled
          v-model="inputText"
          label="Enter text"
          class="q-ml-md"
          maxlength="100"
          style="background-color: #f0f0f0"
        />
        <!-- Add "Click here" button -->
        <q-btn flat dense label="Click here" class="q-ml-md" @click="handleClickHere" />
      </q-toolbar>
    </q-header>
    <!-- Display the uploaded PDF -->
    <q-page-container>
      <q-page class="q-pa-none">
        <div v-if="pdfUrl" class="pdf-container" ref="pdfContainer">
          <q-inner-loading :showing="loading">
            <q-spinner-dots size="50px" color="primary" />
          </q-inner-loading>
          <canvas ref="pdfCanvas" class="pdf-canvas"></canvas>
        </div>
        <div v-else class="text-center q-pa-md">
          <div class="text-h6">No PDF uploaded</div>
          <div class="text-subtitle1">Click the upload button to select a PDF file</div>
        </div>
      </q-page>
    </q-page-container>
  </q-layout>
</template>
<script>
import * as pdfjsLib from 'pdfjs-dist'
import 'pdfjs-dist/web/pdf_viewer.css'
// Set worker source for PDF.js
pdfjsLib.GlobalWorkerOptions.workerSrc =
  'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.10.377/pdf.worker.min.js'
export default {
  data() {
    return {
      pdfUrl: null,
      inputText: '',
      loading: false,
    }
  },
  methods: {
    triggerFileInput() {
      this.$refs.fileInput.click()
    },
    handleFileUpload(event) {
      const file = event.target.files[0]
      if (!file) return
      if (file.type !== 'application/pdf') {
        alert('Please upload a valid PDF file.')
        return
      }
      // Clean up previous URL
      if (this.pdfUrl) {
        URL.revokeObjectURL(this.pdfUrl)
        this.pdfUrl = null
      }
      // Create new URL and render
      const reader = new FileReader()
      reader.onload = (e) => {
        const arrayBuffer = e.target.result
        const blob = new Blob([arrayBuffer], { type: 'application/pdf' })
        this.pdfUrl = URL.createObjectURL(blob)
        this.$nextTick(() => {
          this.renderPDF()
        })
      }
      reader.readAsArrayBuffer(file)
    },
    async renderPDF() {
      try {
        const loadingTask = pdfjsLib.getDocument(this.pdfUrl)
        this.loading = true
        const pdf = await loadingTask.promise
        const totalPages = pdf.numPages

        // Clear the container before rendering new pages
        if (this.$refs.pdfContainer) {
          this.$refs.pdfContainer.innerHTML = ''
        } else {
          console.error('pdfContainer ref is not defined')
          return
        }

        for (let pageNum = 1; pageNum <= totalPages; pageNum++) {
          const page = await pdf.getPage(pageNum)
          const canvas = document.createElement('canvas')
          canvas.classList.add('pdf-canvas')
          this.$refs.pdfContainer.appendChild(canvas)

          const context = canvas.getContext('2d')
          const viewport = page.getViewport({ scale: 1.0 })
          const desiredWidth = canvas.parentElement.clientWidth - 40
          const scale = desiredWidth / viewport.width
          const scaledViewport = page.getViewport({ scale })

          canvas.width = scaledViewport.width
          canvas.height = scaledViewport.height

          const renderContext = {
            canvasContext: context,
            viewport: scaledViewport,
            enableWebGL: true,
          }
          await page.render(renderContext).promise
        }
        this.loading = false
      } catch (error) {
        console.error('Error loading PDF:', error)
        this.loading = false
        this.pdfUrl = null
        alert('Error loading PDF. Please try again.')
      }
    },
    handleClickHere() {
      if (this.inputText) {
        this.highlightText(this.inputText)
      }
    },
    async highlightText(text) {
      await this.renderPDF()
      try {
        const loadingTask = pdfjsLib.getDocument(this.pdfUrl)
        const pdf = await loadingTask.promise
        const totalPages = pdf.numPages

        for (let pageNum = 1; pageNum <= totalPages; pageNum++) {
          const page = await pdf.getPage(pageNum)
          const textContent = await page.getTextContent()
          console.log('textContent:', textContent)
          
          const canvas = this.$refs.pdfContainer.querySelectorAll('.pdf-canvas')[pageNum - 1]
          const context = canvas.getContext('2d')
          
          textContent.items.forEach((item) => {
            if (item.str.toLowerCase().includes(text.toLowerCase())) {
              const { transform, width, height } = item
              const x = transform[4]
              const y = transform[5]
              var xlength = 0
              const index = item.str.toLowerCase().indexOf(text.toLowerCase());
              if(index == 0) xlength = 0;
              else xlength = item.str.substring(0, index).length;
              const viewport = page.getViewport({ scale: 2.0 })
              const scale = canvas.width / viewport.width
              const adjustedX = x * scale
              const adjustedY = (viewport.height - y) * scale
              const adjustedWidth = width * scale * (text.length) / item.str.length;
              const adjustedHeight = height * scale
              context.fillStyle = 'rgba(255, 255, 0, 0.5)'
              context.fillRect(adjustedX + width * scale * xlength / item.str.length, adjustedY - adjustedHeight, adjustedWidth, adjustedHeight)
            }
          })
        }
      } catch (error) {
        console.error('Error highlighting text:', error)
      }
    },
  },
}
</script>
<style>
.pdf-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}
.pdf-canvas {
  max-width: 100%;
  height: auto;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  background-color: white;
  margin: 20px;
}
</style>
