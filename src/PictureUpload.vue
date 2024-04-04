<script  lang="ts">

// Define an interface or type for the fetchAccessToken function
interface FetchAccessToken {
  (): Promise<string>; // Defines a function that returns a Promise<string>
}

import axios from 'axios';
import { watch } from 'fs';
import { defineComponent, inject } from 'vue';

const apiKey = import.meta.env.VITE_GOOGLE_CLOUD_API_KEY;
console.log(apiKey)

export default defineComponent({
  setup() {
      // Inject the fetchAccessToken function
      const fetchAccessToken = inject('fetchAccessToken') as FetchAccessToken;
      return {
        fetchAccessToken,
      };
  },
  emits: ['update:modelValue'],
  data() {

    return {
      image: "",
      modelValue: "",
    };
  },
  methods: {
    
    onFileChange(e: any) {
      const files = e.target.files || e.dataTransfer.files;
      if (!files.length) return;
      this.createImage(files[0]);
    },
    createImage(file: File) {
      let reader = new FileReader();
      reader.onload = (e) => {
        this.image = e.target?.result as string;
        this.submitToGoogleCloudVision().then(() => {
          // TODO this.readTextWithUser();
        });
      };
      reader.readAsDataURL(file);
    },

    async submitToGoogleCloudVision() {
          // Fetch the access token
          const accessToken = await this.fetchAccessToken(); // Fetch the access token
          
          const url = `https://vision.googleapis.com/v1/images:annotate`;
          const body = {
            requests: [
              {
                image: {
                  content: this.image.split(',')[1]
                },
                features: [
                  {
                    type: "TEXT_DETECTION"
                  }
                ]
              }
            ]
          };
    

      try {
        const response = await axios.post(url, body,
          {
            headers: {
              'Authorization': `Bearer ${accessToken}`,
            }
          });
        console.log(response.data);
        this.modelValue = response.data.responses[0].fullTextAnnotation.text;
        this.$emit('update:modelValue', this.modelValue);
        console.log(this.modelValue);
        // Handle the response as needed
      } catch (error) {
        console.error(error);
      }
    },
  }
});
</script>
<template>
  <div v-if="!modelValue">
    <p>Bitte Bild aufnehmen</p>
    <input type="file" accept="image/*" @change="onFileChange" capture="environment"/>
    <img :src="image" />
  </div>
</template>

<style scoped>
  img {
    max-width: 100%;
  }
</style>