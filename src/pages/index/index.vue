<template>
  <view class="content">
    <view class="tool">
      <button class="item" @click="handleClear">清除</button>
      <button class="item" @click="handleCrop">导出图片</button>
      <checkbox class="item" v-model="minimal" />是否最小化空白区域
    </view>
    <pi-canvas
      ref="picanvas"
      :minimal="minimal === 'true'"
      style="width: 100%; height: 200rpx"
      @export="handleExport"
    />
    <image :src="src" :style="[style, { marginTop: '32rpx' }]" />
  </view>
</template>

<script>
import PiCanvas from '@/components/pi-canvas'
export default {
  components: {
    PiCanvas,
  },
  data() {
    return {
      src: '',
      style: {},
      minimal: 'false',
    }
  },
  onLoad() {},
  methods: {
    handleClear() {
      this.$refs.picanvas.clear()
    },
    handleCrop(res) {
      this.$refs.picanvas.crop()
    },
    handleExport(res) {
      this.src = res.base64
      this.style = res.style
    },
  },
}
</script>

<style lang="scss" scoped>
.tool {
  height: 64px;

  display: flex;
  align-items: center;
  .item {
    margin-left: 8px;
  }
}
.content {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.logo {
  height: 200rpx;
  width: 200rpx;
  margin: 200rpx auto 50rpx auto;
}

.text-area {
  display: flex;
  justify-content: center;
}

.title {
  font-size: 36rpx;
  color: #8f8f94;
}
</style>
