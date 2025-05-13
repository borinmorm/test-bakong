<template>
  <div class="home">
    <h1>Bakong QR Code Generator</h1>
    <BakongQRGenerator @qr-generated="onQRGenerated" @reset="onReset" />
  </div>
</template>

<script>
// @ is an alias to /src
import BakongQRGenerator from '@/components/BakongQRGenerator.vue';

export default {
  name: 'HomeView',
  components: {
    BakongQRGenerator
  },
  data() {
    return {
      lastGeneratedQR: null
    };
  },
  methods: {
    onQRGenerated(qrData) {
      this.lastGeneratedQR = qrData;
      console.log('QR generated in parent component:', qrData);
    },
    onReset() {
      this.lastGeneratedQR = null;
      console.log('QR reset in parent component');
    }
  }
}
</script>

<style scoped>
.home {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}

h1 {
  color: #2c3e50;
  margin-bottom: 30px;
}
</style>
