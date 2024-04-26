<script lang="ts">
// Removed unused import formToJSON
import PictureUpload from './PictureUpload.vue';
import TextPart from './TextPart.vue';
import { TextPartInterface } from './types/TextPartInterface';
import { defineComponent, ref, provide } from 'vue';

// Removed the textPart class as we will use the TextPartInterface

export default defineComponent({
  components: {
    PictureUpload,
    TextPart
  },
  data() {
    return {
      parts: [] as TextPartInterface[],
      OCRText: "25 460 0 Ich brauche einen Piratenthron, damit ich mich auf diesem Schiff wie zuhause fühlen kann!",
      transcribedText: null,
    }
  },
  setup() {
    const accessToken = ref("");
    const fetchAccessToken = async () => {
      // Check if accessToken already exists and return it
      if (accessToken.value) {
        return accessToken.value;
      }
      const response = await fetch('https://europe-central2-leseapp-416115.cloudfunctions.net/myCloudFunction');
      if (!response.ok) {
        throw new Error('Failed to fetch access token');
      }
      const data = await response.json();
      // console.log(data);
      accessToken.value = data.accessToken; // Store the fetched token
      return data.accessToken;
    };

    provide('fetchAccessToken', fetchAccessToken);
    return {
      fetchAccessToken,
    };
  },
  methods: {
    async synthesizeTextToSpeech(text: string) {
      const accessToken = await this.fetchAccessToken(); // Fetch the access token

      const requestOptions: RequestInit = {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          input: { text: text },
          voice: { languageCode: 'de-DE', ssmlGender: 'NEUTRAL' },
          audioConfig: { audioEncoding: 'MP3' },
        })
      };
      const response = await fetch('https://texttospeech.googleapis.com/v1/text:synthesize', requestOptions);
      if (!response.ok) {
        throw new Error('Failed to synthesize text');
      }
      const data = await response.json();
      const data: SynthesizeSpeechResponse = await response.json();
      const audioContent = data.audioContent;
      return `data:audio/mp3;base64,${audioContent}`;
    },

    splitTextAtMiddleWord(text: string) {
      const words = text.split(' ');
      const middleIndex = Math.floor(words.length / 2);
      const middleWord = words[middleIndex];
      const firstHalf = words.slice(0, middleIndex).join(' ');
      const secondHalf = words.slice(middleIndex + 1).join(' ');
      return [firstHalf, middleWord, secondHalf];
    },

    async setupReadTextWithUserParts() {
      for (const part of this.splitTextAtMiddleWord(this.OCRText)) {
        console.log(part);
        this.parts.push({
          text: part,
          audio: await this.synthesizeTextToSpeech(part),
          readback: false // Set the initial readback state
        });
      }

      // Removed event listeners for audio as we will handle this in the TextPart component
    },

    async playFirstPart() {
      if (this.parts.length == 0) {
        await this.setupReadTextWithUserParts()
      }

      this.parts[0]["audio"].play();
    },

    async handleTextChange(newValue: string) {
      this.OCRText = newValue;
      await this.setupReadTextWithUserParts();
      this.playFirstPart();
    },
  }
});


</script>

<template>
  <main>
    <h1>LeseApp</h1>
    <div v-if="!OCRText">
      <PictureUpload @ocrTextChanged="handleTextChange" />
    </div>
    <div v-if="OCRText">
      <div id="fullOCRText">{{ OCRText }}</div>
      <button @click="playFirstPart">Nochmal vorlesen</button>
      <TextPart v-for="(part, index) in parts" :key="index" :text="part.text" :readback="part.readback" />
    </div>


  </main>
</template>
