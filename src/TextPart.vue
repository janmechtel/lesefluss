<script lang="ts">

// Define an interface or type for  the fetchAccessToken function
interface FetchAccessToken {
  (): Promise<string>; // Defines a function that returns a Promise<string>
}

import { defineComponent, inject, type PropType } from 'vue';
import { type TextPartInterface } from './types/TextPartInterface';

export default defineComponent({
  props: {
    part: {
      type: Object as PropType<TextPartInterface>,
      required: true,
    },
  },
  emits: ['partEnded'],
  setup() {
    const fetchAccessToken = inject('fetchAccessToken') as FetchAccessToken;
    return {
      fetchAccessToken,
    };
  },
  data() {
    return {
      audioBlob: null as Blob | null,
      audioChunks: [] as Blob[],
      audioElement: null as HTMLAudioElement | null,
      isRecorded: false,
      isRecording: false,
      mediaRecorder: null as MediaRecorder | null,
      recordedAudio: null as HTMLAudioElement | null,
      transcribedText: null as string | null,
      showSuccess: false, // to control the visibility of the success animation
    };
  },
  mounted() {
    this.createAudioElement();
  },
  methods: {
    showSuccessAnimation() {
      this.showSuccess = true;
      setTimeout(() => {
        this.showSuccess = false; // Hide the animation after 2 seconds
      }, 2000);
    },

    createAudioElement() {
      if (this.part.audio) {
        this.audioElement = new Audio(this.part.audio);
        this.audioElement.onended = () => this.$emit('partEnded');
      }
    },

    startPart() {
      if (this.part.readback) {
        this.startRecording();
      } else {
        this.audioElement?.play();
      }
    },

    async startRecording() {
      if (!this.isRecording) {
        this.audioChunks = [];
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.mediaRecorder = new MediaRecorder(stream);
        this.mediaRecorder.ondataavailable = (event) => {
          console.log("Recording data...");
          console.log(event.data);
          this.audioChunks.push(event.data);
        };
        this.mediaRecorder.onstop = async () => {
          console.log("Stopped Recording")
          this.processRecordedAudioChunks();
          this.processAudioReadback();
        }
        this.mediaRecorder.start();
        setTimeout(() => this.stopRecording(), 3000);
        this.isRecording = true;
        this.isRecorded = false;
      }
    },

    stopRecording() {
      if (this.isRecording && this.mediaRecorder) {
        this.mediaRecorder.stop();
        this.mediaRecorder.stream.getTracks().forEach(track => track.stop());
        this.isRecording = false;
        this.isRecorded = true;
      }
    },

    processRecordedAudioChunks() {
      if (this.audioChunks.length > 0) {
        const audioBlob = new Blob(this.audioChunks, { type: 'audio/opus' });
        this.audioBlob = audioBlob;
        this.audioChunks = [];
        this.recordedAudio = new Audio(URL.createObjectURL(audioBlob));
        console.log(this.audioBlob);
        console.log(this.recordedAudio);
      }
    },

    playRecordedAudio() {
      if (this.recordedAudio) {
        this.recordedAudio.play();
      }
    },

    blobToBase64(blob: Blob) {
      return new Promise<string | ArrayBuffer | null>((resolve, _) => {
        const reader = new FileReader();
        reader.onloadend = () => resolve(reader.result);
        reader.readAsDataURL(blob);
      });
    },
    async processAudioReadback() {
      if (this.audioBlob) {
        this.transcribedText = await this.sendAudioToSpeechAPI();
        console.log(`transcribed text: ${this.transcribedText}`);
        if (this.transcribedText === this.part.text.trim()) {
          this.showSuccessAnimation(); // Trigger success animation instead of logging to console
        } else {
          console.log("FAIL!");
        }
        this.$emit('partEnded');
      }
    },

    async sendAudioToSpeechAPI(): Promise<string> {
      const config = {
        encoding: 'WEBM_OPUS',
        sampleRateHertz: 16000,
        languageCode: 'de-DE',
        enableAutomaticPunctuation: true,
      };
      try {
        const accessToken = await this.fetchAccessToken(); // Fetch the access token
        const base64data = await this.blobToBase64(this.audioBlob as Blob) as string;
        const content = base64data.substr(base64data.indexOf(',') + 1);
        const response = await fetch('https://speech.googleapis.com/v1p1beta1/speech:recognize', {
          method: 'POST',
          body: JSON.stringify({
            config: config,
            audio: {
              content: content,
            },
          }),
          headers: {
            'Authorization': `Bearer ${accessToken}`,
          },
        });
        console.log("Sending audio to Speech API...");
        const json_response = await response.json();
        return json_response.results
          .map((result: { alternatives: { transcript: any; }[]; }) => result.alternatives[0].transcript).join(' ').trim();

      } catch (error) {
        console.error('Error sending audio to Speech API:', error);
        return ""
      }
    }
  }
});
</script>

<template>
  <div class="text-part">
    {{ part.text }}
    <div v-if="showSuccess" class="success-animation">
      Success!
    </div>
  </div>
  <div v-if="part.readback" class="text-part-readback">
    <div v-if="isRecording" class="recording-indicator">
      Recording <span class="dot">•</span>
    </div>
    <button v-if="isRecording" @click="stopRecording">Stop Recording</button>
  </div>
  <hr />
</template>

<style scoped>
.success-animation {
  animation: successBlink 2s infinite;
  color: green;
  font-size: 20px;
  font-weight: bold;
}

@keyframes successBlink {
  0%, 100% {
    opacity: 1;
  }
  50% {
    opacity: 0;
  }
}

.recording-indicator .dot {
  animation: blink 1s infinite;
  color: red;
}
</style>




