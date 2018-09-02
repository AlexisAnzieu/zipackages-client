<template>
        <Upload></Upload>
</template>

<script>
import Upload from "~/pages/Upload.vue";
import axios from "axios";

export default {
  head() {
    return {
      title: "pack2zip"
    };
  },
  components: {
    Upload
  }
};
</script>

<style>
html {
  text-align: center;
}
</style>
