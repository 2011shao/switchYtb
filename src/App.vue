
<template>
  <main>
    <switchYtb class="p-all-5" />
  </main>
</template>
<script setup>
import switchYtb from "./components/switchYtb.vue";
</script>

<style>
@import "@/assets/superStyle/superCss.css";
@import "@/assets/superStyle/selfCss.css";

main {
  /* padding: 103px; */
}
.labelCss {
  width: 70px;
  text-align: center;
  flex-shrink: 0;
}
h4 {
  font-size: calc(1.275rem + 0.3vw);
  margin-bottom: 1rem;
}
code {
  font-size: 0.875em;
  color: #d63384;
  word-wrap: break-word;
}
</style>
