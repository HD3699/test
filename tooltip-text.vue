<template>
  <div class="text-tooltip">
    <el-tooltip effect="dark" ref="aaa" :disabled="isShowTooltip" :content="content" placement="top-start" popper-class="text-tooltip">
      <div class="over-flow" :style="cssStyle" @mouseover="onMouseOver(refName)">
        <span :ref="refName">{{ content || '-' }}</span>
      </div>
    </el-tooltip>
  </div>
</template>

<script>
export default {
  props: {
    // 显示的文字内容
    content: {
      type: String
    },
    refName: {
      type: String
    },
    width: {
      type: Number
    },
    maxWidth: {
      type: Number
    },
    row: {
      type: Number
    }
  },
  computed: {
    cssStyle () {
      if (this.maxWidth) return {maxWidth: this.maxWidth + 'px'}
      if (this.width) return {width: this.width + 'px'}
      if (this.row) return `overflow: hidden;text-overflow: ellipsis;display: -webkit-box;-webkit-line-clamp: ${this.row};-webkit-box-orient: vertical;`
    }
  },
  data () {
    return {
      tooltipWidth: '100%',
      isShowTooltip: true
    }
  },
  methods: {
    onMouseOver (refName) {
      // 通过设置文本行数进行是否展示tooltip
      if (this.row) {
        this.isShowTooltip = this.$refs[refName].parentNode.offsetHeight > this.$refs[refName].offsetHeight
      } else if (this.width || this.maxWidth) {
        // 通过设置文本宽度进行是否展示tooltip
        this.$refs[refName].style.whiteSpace = 'nowrap'
        this.isShowTooltip = this.$refs[refName].parentNode.offsetWidth >= this.$refs[refName].offsetWidth
      }
      if (!this.isShowTooltip) {
        this.$nextTick(() => {
          let tipDom = document.querySelectorAll('.text-tooltip.el-tooltip__popper')[document.querySelectorAll('.text-tooltip.el-tooltip__popper').length - 1]
          if (tipDom) {
            tipDom.style.maxWidth = this.$refs[refName].parentNode.offsetWidth + 'px'
          }
        })
      }
    }
  }
}
</script>

<style lang="less">
.text-tooltip.el-tooltip__popper {
  max-height: calc(~'100vh - 100px');
  // overflow: auto;
}
</style>
<style lang="less" scoped>
.over-flow {
  overflow: hidden;
  text-overflow: ellipsis;
}
</style>
