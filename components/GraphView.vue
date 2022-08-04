<template lang="pug">
div
  .graph(ref="graphView")
</template>

<script setup lang="ts">
import { WebGLRenderer } from "three";
const props = defineProps<{ formula: string }>();

const graphView = ref<HTMLDivElement>();

const o: { [key: string]: any } = new Proxy(
  {},
  {
    set(obj, prop, val) {
      if (prop in obj && typeof obj[prop].dispose === "function") {
        obj[prop].dispose();
      }
      obj[prop] = val;
      return true;
    },
    deleteProperty(obj, prop) {
      if (prop in obj) {
        delete obj[prop];
      }
      return true;
    },
  }
);
const finalize = () => {
  for (const key in o) {
    if (typeof o[key].dispose === "function") {
      o[key].dispose();
    }
  }
};
onMounted(() => {
  o.renderer = new WebGLRenderer();
});
onUnmounted(() => {
  finalize();
});
</script>
