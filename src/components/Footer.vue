<!-- src/components/Footer.vue -->
<template>
  <footer class="footer">
    <div class="footer-content">
      <div class="contact-info">
        <p>Contact: matt@mwschultz.com</p>
      </div>
    </div>
  </footer>
</template>

<script>
export default {
  name: 'FooterComponent'
}
</script>

<style scoped>
.footer {
  background-color: #f8f8f8;
  padding: 20px;
  text-align: center;
}
.contact-info {
  font-size: 14px;
  color: #333;
}
</style>
