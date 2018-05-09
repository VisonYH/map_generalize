<template>
  <div class="header">
    <div class="logo">MAP GENERALIZE</div>
  </div>
</template>

<script>
export default {
  name: 'v-header',
  data () {
    return {
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="less">
@import "../../common/style/base.less";
.header{
  height: 40px;
  padding: 0 10px;
  margin-top: -40px;
  line-height: 40px;
  font-size: 24px;
  color: #ffffff;
  background: @theme-color;
}
</style>
