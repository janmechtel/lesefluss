<script lang="ts">
import PictureUpload from './PictureUpload.vue';
import TextPart from './TextPart.vue';
import { type TextPartInterface } from './types/TextPartInterface';
import { defineComponent, ref, provide } from 'vue';

interface SynthesizeSpeechResponse {
  audioContent: string; // Base64-encoded string of audio content
}

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
  mounted() {
    if (this.OCRText) {
      this.handleTextChange(this.OCRText);
    }
  },
  setup() {
    const accessToken = ref("");
    const fetchAccessToken = async () => {
      if (accessToken.value) {
        return accessToken.value;
      }
      const response = await fetch('https://europe-central2-leseapp-416115.cloudfunctions.net/myCloudFunction');
      if (!response.ok) {
        throw new Error('Failed to fetch access token');
      }
      const data = await response.json();
      accessToken.value = data.accessToken;
      return data.accessToken;
    };

    provide('fetchAccessToken', fetchAccessToken);
    return {
      fetchAccessToken,
    };
  },
  methods: {
    handlePartEnded(index: number, _?: Event) {
      console.log(`Finished part ${index}`);
      index++;
      
      if (index < this.parts.length) {
        console.log(`Now starting part ${index}`);
        (this.$refs[`textPart${index}`] as any)[0].startPart();
      } else {
        this.reset();
      }
    },
    reset() {
      this.transcribedText=null;
      this.OCRText="";
    },


    async synthesizeTextToSpeech(text: string) {
      const accessToken = await this.fetchAccessToken();

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
      let index = 0;
      for (const part of this.splitTextAtMiddleWord(this.OCRText)) {
        const readback = index === 1; // Enable readback for the second part
        this.parts.push({
          text: part,
          audio: await this.synthesizeTextToSpeech(part),
          readback: readback
        });
        index++;
      }
    },

    async playFirstPart() {
      if (this.parts.length == 0) {
        await this.setupReadTextWithUserParts()
      }
      this.handlePartEnded(-1); // start play the first because part -1 ended
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
      <TextPart v-for="(part, index) in parts" :key="index" :part="part"
        @partEnded="(event) => handlePartEnded(index, event)" :ref="`textPart${index}`" />
      <button @click="playFirstPart">Nochmal vorlesen</button>
    </div>
  </main>
</template>
