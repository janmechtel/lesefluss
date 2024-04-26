<template>
  <div class="text-part">
    {{ text }}
  </div>
</template>

<script lang="ts">
import { defineComponent, PropType } from 'vue';

export default defineComponent({
  props: {
    text: {
      type: String as PropType<string>,
      required: true,
    },
  },
});
</script>

<style scoped>
.text-part {
  /* Add styles for text part here if needed */
}
</style>
