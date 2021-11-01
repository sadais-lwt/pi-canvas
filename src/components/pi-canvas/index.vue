<template>
  <div ref="root" :class="[customClass, 'wrapper']" :style="[customStyle]">
    <canvas
      ref="canvas"
      class="content"
      :id="canvasId"
      :canvas-id="canvasId"
      :width="canvasw"
      :height="canvash"
      @touchstart="start"
      @touchmove="move"
      @touchcancel="end"
      @touchend="end"
      @mousedown="start"
      @mousemove="move"
      @mouseup="end"
    ></canvas>
    <canvas
      ref="cropCanvas"
      class="copy"
      :id="cropId"
      :canvas-id="cropId"
      :width="cropw"
      :height="croph"
    ></canvas>
  </div>
</template>
<script>
export default {
  name: 'PiCanvas',
  props: {
    // 自定义class
    customClass: {
      type: String,
      // `''`
      default: '',
    },
    // 自定义style
    customStyle: {
      type: Object,
      // `{}`
      default: () => ({}),
    },
    // 是否最小化空白区域
    minimal: {
      type: Boolean,
      // `false`
      default: false,
    },
    // 当minimal:true时，指定pandding
    padding: {
      type: Object,
      // `{ top: 8, right: 8, bottom: 8, left: 8 }`
      default: () => ({ top: 8, right: 8, bottom: 8, left: 8 }),
    },
    // 画布背景色
    bgColor: {
      type: String,
      // `#ffffff`
      default: '#ffffff',
    },
    // 画笔颜色
    color: {
      type: String,
      // `#333333`
      default: '#333333',
    },
    // 线条宽度
    lineWidth: {
      type: Number,
      // `1`
      default: 1,
    },
  },
  data() {
    return {
      isBegin: false, // 标记是否先触发了touchstart or mousedown
      canvasId: '', // 内容画布的ID
      cropId: '', // 裁剪画布的ID
      ctx: null, // 内容画布的上下文
      canvasw: 0, // 内容画布的宽度
      canvash: 0, // 内容画布的高度
      cropw: 0, // 裁剪画布的宽度
      croph: 0, // 裁剪画布的高度
    }
  },
  created() {
    this.$nextTick(() => {
      this.initCanvas()
    })
  },
  methods: {
    generateId() {
      // 产生16位随机的字母、数字混合的随机ID
      const tmps = '0123456789'
        .split('')
        .concat('abcdefghijklmnopqrstuvwxyz'.split(''))
        .concat('ABCDEFGHIJKLMNOPQRSTUVWXYZ'.split(''))
      let n = tmps.length
      while (n--) {
        const idx = Math.floor(Math.random() * n)
        const tmp = tmps[idx]
        tmps[idx] = tmps[n]
        tmps[n] = tmp
      }
      return tmps.join('').substr(0, 16)
    },
    initCanvas() {
      this.canvasId = this.generateId()
      this.cropId = this.generateId()
      if (!window.uni) {
        // 非uni环境
        const root = this.$refs.root
        const style = window.getComputedStyle(root)
        this.canvasw = parseInt(style.width, 10)
        this.canvash = parseInt(style.height, 10)
        const canvas = this.$refs.canvas
        this.ctx = canvas.getContext('2d')
        this.clear()
      } else {
        // uni环境：无法操作dom 只能通过它提供的API获取布局信息
        const that = this
        const query = uni.createSelectorQuery().in(this)
        query
          .select('.wrapper')
          .boundingClientRect((res) => {
            that.canvasw = res.width
            that.canvash = res.height
            that.ctx = uni.createCanvasContext(that.canvasId, that)
            that.clear()
          })
          .exec()
      }
    },
    /**
     * @vuese
     * 清空画布内容
     */
    clear() {
      this.$nextTick(() => {
        this.initRange()
        this.initPen()
        this.ctx.clearRect(0, 0, this.canvasw, this.canvash)
        this.ctx.fillRect(0, 0, this.canvasw, this.canvash)
      })
    },
    initPen() {
      // 初始化画笔
      this.ctx.fillStyle = this.bgColor
      this.ctx.strokeStyle = this.color
      this.ctx.lineWidth = this.lineWidth
    },
    initRange() {
      // 记录画布上左上角的点 s: start
      this.s = { x: Number.MAX_VALUE, y: Number.MAX_VALUE }
      // 记录画布右下角的点 e: end
      this.e = { x: 0, y: 0 }
    },
    /**
     * @vuese
     * 裁剪画布内容
     */
    crop() {
      if (this.s.x === Number.MAX_VALUE) {
        // 标明画布无内容 不进行裁剪操作
        return
      }
      if (this.minimal) {
        // 需要最小化空白区域，计算裁剪区域
        let x = this.s.x - (this.padding.left || 0)
        let y = this.s.y - (this.padding.top || 0)
        x = x < 0 ? 0 : x
        y = y < 0 ? 0 : y

        let x1 = this.e.x + (this.padding.right || 0)
        let y1 = this.e.y + (this.padding.bottom || 0)
        x1 = x1 > this.canvasw ? this.canvasw : x1
        y1 = y1 > this.canvash ? this.canvash : y1

        // 获取裁剪区域的RGBA像素点数据
        const imageData = this.ctx.getImageData(x, y, x1 - x, y1 - y)
        this.emitCrop(imageData, x1 - x, y1 - y)
      } else {
        // 直接获取整张画布区域的像素点数据
        const imageData = this.ctx.getImageData(
          0,
          0,
          this.canvasw,
          this.canvash
        )
        this.emitCrop(imageData, this.canvasw, this.canvash)
      }
    },
    async getBase64(imageData, w, h) {
      // 原理：将imageData放入裁剪画布中，然后即可生成base64
      if (window.uni) {
        // uni环境下 按uni提供的API进行操作
        const that = this
        return new Promise((resolve) => {
          uni.canvasPutImageData(
            {
              canvasId: that.cropId,
              data: imageData,
              x: 0,
              y: 0,
              success: () => {
                uni.canvasToTempFilePath({
                  canvasId: that.cropId,
                  x: 0,
                  y: 0,
                  width: w,
                  height: h,
                  success: (res) => {
                    resolve(res.tempFilePath)
                  },
                  fail: () => {
                    resolve('')
                  },
                })
              },
            },
            this
          )
        })
      } else {
        // 否则 按常规尝试进行
        const canvas = this.$refs.cropCanvas
        ctx = canvas.getContext('2d')
        ctx.putImageData(imageData, 0, 0)
        const base64 = canvas.toDataURL()
        return base64
      }
    },
    emitCrop(imageData, w, h) {
      this.cropw = w
      this.croph = h
      this.$nextTick(async () => {
        // $nextTick的目的是为了让裁剪画布的宽高生效后 再进行操作
        const base64 = await this.getBase64(imageData, w, h)

        const results = base64.match(/data:(.*?);.*?,(.*)/)
        const mineType = results[1]
        const suffix = mineType.split('/')[1]
        const binstr = window.atob(results[2])
        let n = binstr.length
        const bytes = new Uint8Array(n)
        while (n--) {
          bytes[n] = binstr.charCodeAt(n)
        }
        const blob = new Blob([bytes], { type: mineType })
        // export
        // arg `{base64, blob, width, height, style}`
        this.$emit('export', {
          base64,
          blob,
          width: w,
          height: h,
          style: { width: `${w}px`, height: `${h}px` },
        })
      })
    },
    /**
     * 随着画笔移动，不断调用此方法更新：包裹内容的矩形的左上角&右下角的坐标
     */
    updateRange(x, y) {
      if (x < this.s.x) {
        this.s.x = x
      } else if (x > this.e.x) {
        this.e.x = x
      }
      if (y < this.s.y) {
        this.s.y = y
      } else if (y > this.e.y) {
        this.e.y = y
      }
    },
    start(e) {
      console.log('start=========', e)
      // 适配移动端的写法: e.touches移动端才有
      const touches = (e && e.touches) || []
      const { offsetX: x, offsetY: y } = touches[0] || e
      this.ctx.save()

      // 为了清空之前残留的路径
      this.ctx.beginPath()
      this.ctx.moveTo(x, y)
      this.ctx.stroke()
      this.isBegin = true
      this.updateRange(x, y)
    },
    move(e) {
      if (!this.isBegin) {
        return
      }
      console.log('move======', e)
      const touches = (e && e.touches) || []
      const { offsetX: x, offsetY: y } = touches[0] || e

      this.ctx.lineTo(x, y)
      this.ctx.stroke()
      this.updateRange(x, y)
    },
    end(e) {
      console.log('end======', e)
      this.move(e)
      this.isBegin = false
      this.ctx.restore()
    },
  },
}
</script>

<style lang="scss" scoped>
.wrapper {
  position: relative;
  width: 100%;
  height: 100%;
  border: 1px solid #aaaaaa;
}

.content {
  position: absolute;
  top: 0;
  left: 0;
}

.copy {
  position: absolute;
  top: -50000px;
  left: -50000px;
}
</style>
