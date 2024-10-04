<template>
    <div class="container">
      <h2>Upload de Arquivo MP3</h2>
      <input type="file" @change="handleFileUpload" accept=".mp3">
      <p v-if="selectedFile">Arquivo selecionado: {{ selectedFile.name }}</p>
      <button @click="processFile" :disabled="!selectedFile || isProcessing">
        {{ isProcessing ? 'Processando...' : 'Processar Arquivo' }}
      </button>
      <div class="controls">
        <button @click="playAudio" :disabled="!audioBuffer">Play</button>
        <button @click="pauseAudio" :disabled="!isPlaying">Pause</button>
        <button @click="stopAudio" :disabled="!audioBuffer">Stop</button>
      </div>
      <canvas ref="canvas" :width="canvasWidth" :height="canvasHeight"></canvas>
      <input type="range" min="0" :max="audioDuration" step="0.01" v-model="currentTime" @input="seekAudio" class="progress-bar">
      <!-- Elemento de áudio oculto -->
      <audio ref="audioElement" :src="audioSrc"></audio>
    </div>
  </template>
  
  <script>
  // Importar Meyda
  import Meyda from 'meyda'
  
  export default {
    name: 'FileUpload',
    data() {
      return {
        selectedFile: null,
        isProcessing: false,
        beatData: [], // Armazena informações sobre cada batida
        audioBuffer: null,
        audioContext: null,
        isPlaying: false,
        canvasWidth: 600,
        canvasHeight: 400,
        circleRadius: 50, // Raio base para a elipse
        distortionX: 0, // Distorção horizontal (eixo X)
        distortionY: 0, // Distorção vertical (eixo Y)
        circleColor: '#f00',
        currentTime: 0,
        audioDuration: 0,
        animationFrameRequest: null,
        audioSrc: null,
        updateInterval: null,
        nextBeatIndex: 0,
      }
    },
    methods: {
      handleFileUpload(event) {
        this.selectedFile = event.target.files[0]
        this.beatData = []
        this.audioBuffer = null
        this.isPlaying = false
        this.currentTime = 0
        this.audioDuration = 0
        this.circleColor = '#f00'
        this.distortionX = 0
        this.distortionY = 0
        this.nextBeatIndex = 0
  
        // Revogar URL anterior se existir
        if (this.audioSrc) {
          URL.revokeObjectURL(this.audioSrc)
        }
        this.audioSrc = URL.createObjectURL(this.selectedFile)
        
        this.drawEllipse()
      },
      async processFile() {
        if (!this.selectedFile) {
          alert('Por favor, selecione um arquivo MP3 primeiro.')
          return
        }
  
        this.isProcessing = true
  
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)()
        const fileReader = new FileReader()
  
        fileReader.onload = async (e) => {
          const arrayBuffer = e.target.result
          this.audioBuffer = await this.audioContext.decodeAudioData(arrayBuffer)
  
          // Definir a duração do áudio
          this.audioDuration = this.audioBuffer.duration
  
          // Detecção de batidas com análise de frequência
          const sampleRate = this.audioBuffer.sampleRate
          const channelData = this.audioBuffer.getChannelData(0)
          const bufferLength = channelData.length
  
          // Parâmetros para detecção de batidas
          const blockSize = 1024
          const energyHistory = []
          const hopSize = 512 // Sobreposição de 50%
  
          for (let i = 0; i < bufferLength - blockSize; i += hopSize) {
            const block = channelData.slice(i, i + blockSize)
  
            // Usar Meyda para extrair recursos
            const features = Meyda.extract(['rms', 'spectralCentroid'], block, {
              sampleRate: sampleRate,
              bufferSize: blockSize,
            })
  
            const energy = features.rms // Root Mean Square como medida de energia
            const spectralCentroid = features.spectralCentroid // Centroide espectral
  
            // Manter histórico de energias para comparação
            if (energyHistory.length < 43) {
              energyHistory.push(energy)
              continue
            }
  
            // Calcular energia média
            const localEnergy = energyHistory.reduce((a, b) => a + b) / energyHistory.length
  
            // Definir um limiar
            const threshold = localEnergy * 1.5
  
            if (energy > threshold) {
              // Salvar dados da batida
              const time = i / sampleRate
              const beatInfo = {
                time: time,
                frequency: spectralCentroid,
                intensity: energy,
              }
              this.beatData.push(beatInfo)
            }
  
            // Atualizar o histórico
            energyHistory.shift()
            energyHistory.push(energy)
          }
  
          this.isProcessing = false
  
          this.$nextTick(() => {
            this.drawEllipse()
          })
        }
  
        fileReader.readAsArrayBuffer(this.selectedFile)
      },
      playAudio() {
        if (!this.audioSrc) {
          alert('Por favor, processe um arquivo primeiro.')
          return
        }
  
        const audioElement = this.$refs.audioElement
  
        audioElement.play()
          .then(() => {
            this.isPlaying = true
            this.nextBeatIndex = this.beatData.findIndex(beat => beat.time >= this.currentTime)
            if (this.nextBeatIndex === -1) {
              this.nextBeatIndex = this.beatData.length
            }
            this.animate()
            this.updateInterval = setInterval(() => {
              this.currentTime = audioElement.currentTime
            }, 100)
          })
          .catch(error => {
            console.error('Erro ao reproduzir o áudio:', error)
          })
      },
      pauseAudio() {
        const audioElement = this.$refs.audioElement
        audioElement.pause()
        this.isPlaying = false
  
        // Parar a animação
        cancelAnimationFrame(this.animationFrameRequest)
  
        // Parar de atualizar currentTime
        clearInterval(this.updateInterval)
      },
      stopAudio() {
        const audioElement = this.$refs.audioElement
        audioElement.pause()
        audioElement.currentTime = 0
        this.currentTime = 0
        this.isPlaying = false
        this.distortionX = 0
        this.distortionY = 0
        this.nextBeatIndex = 0
        this.drawEllipse()
  
        // Parar a animação
        cancelAnimationFrame(this.animationFrameRequest)
  
        // Parar de atualizar currentTime
        clearInterval(this.updateInterval)
      },
      seekAudio() {
        const audioElement = this.$refs.audioElement
        audioElement.currentTime = this.currentTime
  
        // Sincronizar nextBeatIndex com currentTime
        this.nextBeatIndex = this.beatData.findIndex(beat => beat.time >= this.currentTime)
        if (this.nextBeatIndex === -1) {
          this.nextBeatIndex = this.beatData.length
        }
  
        // Se estiver tocando, garantir que a animação continue
        if (this.isPlaying) {
          cancelAnimationFrame(this.animationFrameRequest)
          this.animate()
        } else {
          this.drawEllipse()
        }
      },
      animate() {
        if (!this.isPlaying) return
  
        // Verificar se chegamos ao fim da lista de batidas
        if (this.nextBeatIndex < this.beatData.length) {
          const beat = this.beatData[this.nextBeatIndex]
  
          if (this.currentTime >= beat.time) {
            // Batida ocorreu, ajustar distorção
            const frequency = beat.frequency
            const intensity = beat.intensity
  
            // Definir limiar para graves e agudos (ajuste conforme necessário)
            const frequencyThreshold = 5000 // Em Hz
  
            // Multiplicador para controlar a intensidade da distorção
            const distortionMultiplier = 50
  
            if (frequency < frequencyThreshold) {
              // Batida grave, distorção vertical
              this.distortionY = intensity * distortionMultiplier
            } else {
              // Batida aguda, distorção horizontal
              this.distortionX = intensity * distortionMultiplier
            }
  
            // Mudar a cor
            this.circleColor = this.getRandomColor()
            this.nextBeatIndex++
          }
        }
  
        // Gradualmente reduzir a distorção
        if (this.distortionX > 0) {
          this.distortionX *= 0.9
        } else {
          this.distortionX = 0
        }
  
        if (this.distortionY > 0) {
          this.distortionY *= 0.9
        } else {
          this.distortionY = 0
        }
  
        // Redesenhar a elipse com distorção
        this.drawEllipse()
  
        // Continuar a animação
        this.animationFrameRequest = requestAnimationFrame(() => this.animate())
      },
      drawEllipse() {
        const canvas = this.$refs.canvas
        const ctx = canvas.getContext('2d')
  
        // Limpar canvas
        ctx.clearRect(0, 0, canvas.width, canvas.height)
  
        // Desenhar fundo preto
        ctx.fillStyle = '#000'
        ctx.fillRect(0, 0, canvas.width, canvas.height)
  
        // Configurar estilos da elipse
        ctx.fillStyle = this.circleColor
  
        // Definir o raio base
        const baseRadius = this.circleRadius
  
        // Calcular os raios horizontal e vertical com distorção
        const radiusX = baseRadius + this.distortionX
        const radiusY = baseRadius + this.distortionY
  
        // Garantir que os raios não sejam negativos e não excedam um valor máximo
        const minRadius = 10
        const maxRadius = baseRadius * 3
        const finalRadiusX = Math.max(minRadius, Math.min(maxRadius, radiusX))
        const finalRadiusY = Math.max(minRadius, Math.min(maxRadius, radiusY))
  
        // Desenhar elipse
        ctx.beginPath()
        ctx.ellipse(canvas.width / 2, canvas.height / 2, finalRadiusX, finalRadiusY, 0, 0, 2 * Math.PI)
        ctx.fill()
      },
      getRandomColor() {
        const letters = '0123456789ABCDEF'
        let color = '#'
        for (let i = 0; i < 6; i++) {
          color += letters[Math.floor(Math.random() * 16)]
        }
        return color
      },
    },
    beforeUnmount() {
      // Limpar intervalos e animações ao destruir o componente
      clearInterval(this.updateInterval)
      cancelAnimationFrame(this.animationFrameRequest)
  
      // Revogar a URL do Blob para liberar memória
      if (this.audioSrc) {
        URL.revokeObjectURL(this.audioSrc)
      }
    },
  }
  </script>
  
  <style scoped>
  .container {
    max-width: 800px;
    margin: 0 auto;
    text-align: center;
    font-family: Arial, sans-serif;
  }
  
  h2 {
    color: #333;
  }
  
  input[type="file"] {
    margin: 10px 0;
  }
  
  .controls {
    margin: 10px 0;
  }
  
  button {
    margin: 5px;
    padding: 10px 20px;
    font-size: 16px;
  }
  
  button:disabled {
    opacity: 0.5;
  }
  
  canvas {
    margin-top: 20px;
    border: 2px solid #ccc;
  }
  
  .progress-bar {
    width: 100%;
    margin-top: 10px;
  }
  </style>
  