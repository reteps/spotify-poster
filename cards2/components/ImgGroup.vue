<template>
  <div @click="$emit('hide',song.key)" :style="`max-width: ${100/cols}%; width: ${100/cols}%`">
    <div :style="`margin: ${margin}px`">
    <div v-if="showCover && song && song.cover" class="wrapper">
      <v-img :style="`background: ${song && song.color ? song.color.toHexString() : 'white'}`" class="square-img" :src="song.cover"></v-img>
    </div>
    <v-img v-if="showCode && song && song.code" :src="song.code"></v-img>
    </div>
  </div>
</template>
<script>
export default {
  props: ['song', 'showCover', 'showCode', 'cols', 'margin'],
  name: 'ImgGroup',
  data() {
    return {
    visible: true
    }
  }
}
</script>
<style scoped lang="scss">

.wrapper {
   position: relative;
   width: 100%;
   overflow: hidden;
   padding-bottom: 100%;
}
.square-img {
  position: absolute;
  width: 100%;
  height: 100%;
}
</style>
<!--
:class="visible ? '' : ''" @click="visible = !visible"
.end {
  order: 99999;
  opacity: 20%;
  position: absolute;
  top: 0;
  left: 0;
}

-->
