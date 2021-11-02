<template>
  <view class="content">
    <view class="tool">
      <button class="item" @click="handleClear">清除</button>
      <button class="item" @click="handleCrop">导出图片</button>
      <checkbox-group @change="handleCheck">
        <checkbox
          class="item"
          :value="'true'"
          :checked="minimal"
        />是否最小化空白区域
      </checkbox-group>
    </view>
    <pi-canvas
      ref="picanvas"
      :minimal="minimal"
      style="width: 90%; height: 200rpx"
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
      minimal: false,
    }
  },

  methods: {
    handleCheck(e) {
      const val = e.detail.value[0]
      this.minimal = val === 'true'
    },
    handleClear() {
      this.$refs.picanvas.clear()
    },
    handleCrop(res) {
      this.$refs.picanvas.crop()
    },
    handleExport(res) {
      console.log(`export result`, res)
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
