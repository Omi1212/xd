<template>
  <v-container>
    <v-alert type="error" v-if="noFrontCamera">You don't seem to have a front camera on your device</v-alert>
    <v-alert type="error" v-if="noRearCamera">You don't seem to have a rear camera on your device</v-alert>

    <v-card class="camera-container">
      <qrcode-stream
        :paused="paused"
        :torch="torchActive"
        :constraints="{ deviceId: facingMode }"
        @detect="onDetect"
        @error="onError"
        @camera-on="onCameraOn"
      >
        <v-alert type="success" v-if="validationSuccess && paused">This is a valid code</v-alert>
        <v-alert type="error" v-if="validationFailure && paused">This is NOT a valid code!</v-alert>
        <v-alert type="info" v-if="validationPending && paused">Long validation in progress...</v-alert>

        <div class="top-buttons">
          <v-btn @click="switchCamera" icon>
            <v-icon>mdi-camera-switch</v-icon>
          </v-btn>
          <v-btn @click="toggleTorch" :disabled="torchNotSupported" icon>
            <v-icon>mdi-flashlight</v-icon>
          </v-btn>
        </div>

        <div class="qr-frame"></div>

        <v-btn @click="showMyQR" class="upload-button">Mi QR</v-btn>

        <div class="bottom-left-button">
          <v-btn @click="pasteFromClipboard" class="bottom-button">Paste</v-btn>
        </div>
        <div class="bottom-right-button">
          <qrcode-capture ref="qrcodeCapture" @detect="onDetect" />
        </div>
      </qrcode-stream>

      <v-btn @click="close" class="close-button" icon>
        <v-icon>mdi-close</v-icon>
      </v-btn>
    </v-card>
  </v-container>
</template>

<script>
import { QrcodeStream, QrcodeCapture } from 'vue-qrcode-reader'
import { shallowRef } from 'vue'

const PTypes = {
  BOLT11: 'BOLT11',
  LnAddress: 'LnAddress',
  LNURLP: 'LNURLP'
}

const regexs = shallowRef([
  { name: PTypes.BOLT11, regex: /^(lnbc|lntb|lnsb|lnbcrt)([0-9]{1,}[munp]?)?(1)([02-9ac-hj-np-z]{1,}){6,}$/i },
  { name: PTypes.LnAddress, regex: /^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$/ },
  { name: PTypes.LNURLP, regex: /^(lnurl1){1}[0-9ac-hj-np-z]+$/i }
])

export default {
  components: { QrcodeStream, QrcodeCapture },

  data() {
    return {
      isValid: undefined,
      paused: false,
      result: null,
      facingMode: 'environment',
      noRearCamera: false,
      noFrontCamera: false,
      torchActive: false,
      torchNotSupported: false,
      videoInputDevices: [],
      currentDeviceIndex: 0
    }
  },

  computed: {
    validationPending() {
      return this.isValid === undefined && this.paused
    },
    validationSuccess() {
      return this.isValid === true
    },
    validationFailure() {
      return this.isValid === false
    }
  },

  mounted() {
    this.getVideoInputDevices()
  },

  methods: {
    async getVideoInputDevices() {
      const devices = await navigator.mediaDevices.enumerateDevices()
      this.videoInputDevices = devices.filter(device => device.kind === 'videoinput')
    },
    onError(error) {
      if (error.name === 'OverconstrainedError') {
        this.facingMode === 'environment' ? this.noRearCamera = true : this.noFrontCamera = true
      }
      console.error(error)
    },
    async onDetect([firstDetectedCode]) {
      this.result = firstDetectedCode.rawValue
      this.paused = true
      console.log(this.result)
      this.validateResult()
      await this.timeout(2000)
      this.paused = false
    },
    validateResult() {
      this.isValid = regexs.value.some(({ regex }) => regex.test(this.result))
    },
    timeout(ms) {
      return new Promise(resolve => setTimeout(resolve, ms))
    },
    switchCamera() {
      if (this.videoInputDevices.length > 0) {
        this.currentDeviceIndex = (this.currentDeviceIndex + 1) % this.videoInputDevices.length
        this.facingMode = this.videoInputDevices[this.currentDeviceIndex].deviceId
        console.log(`Switched to camera: ${this.videoInputDevices[this.currentDeviceIndex].label}`)
      }
    },
    onCameraOn(capabilities) {
      console.log(capabilities)
      this.torchNotSupported = !capabilities.torch
    },
    async pasteFromClipboard() {
      try {
        const text = await navigator.clipboard.readText()
        this.result = text
        console.log(this.result)
        this.validateResult()
        this.paused = true
        await this.timeout(2000)
        this.paused = false
      } catch (err) {
        console.error('Failed to read clipboard contents: ', err)
      }
    },
    close() {
      console.log('Close button clicked')
    },
    showMyQR() {
      console.log('Mi QR button clicked')
    },
    toggleTorch() {
      this.torchActive = !this.torchActive
    }
  }
}
</script>

<style scoped>
.camera-container {
  width: 100%;
  max-width: 400px;
  height: 740px;
  margin: 0 auto;
  position: relative;
  overflow: hidden;
  border: 1px solid #ccc;
  border-radius: 10px;
}

.qr-frame {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 60%;
  padding-top: 60%;
  transform: translate(-50%, -50%);
  border: 4px solid rgba(255, 255, 255, 0.8);
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  z-index: 10;
}

.top-buttons {
  position: absolute;
  top: 10px;
  left: 10px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  z-index: 20;
}

.bottom-left-button {
  position: absolute;
  bottom: 10px;
  left: 10px;
  z-index: 20;
}

.bottom-right-button {
  position: absolute;
  bottom: 10px;
  right: 10px;
  z-index: 20;
}

.bottom-button {
  margin: 5px 0;
}

.upload-button {
  position: absolute;
  top: calc(50% + 20%);
  left: 50%;
  transform: translate(-50%, 0);
  z-index: 20;
}

.close-button {
  position: absolute;
  top: 10px;
  right: 10px;
  background-color: red;
  color: white;
  border-radius: 50%;
  z-index: 30;
}

.custom-upload-button {
  background-color: #4CAF50; /* Green background */
  color: white; /* White text */
  border-radius: 5px; /* Rounded corners */
  padding: 10px 20px; /* Padding */
  font-size: 16px; /* Font size */
  text-transform: uppercase; /* Uppercase text */
}

.custom-upload-button:hover {
  background-color: #45a049; /* Darker green on hover */
}
</style>