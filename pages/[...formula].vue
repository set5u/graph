<template lang="pug">
div
  GraphView(:formula="formula")
</template>

<script setup lang="ts">
const param = useRoute().params.formula;
const formula = typeof param === "string" ? param : param.join("/");
</script>
