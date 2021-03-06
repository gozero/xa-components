<template>
  <div class="xa-refresh-control" :style="refreshStyle" :class="refreshClass">
    <svg v-show="!refreshing && draging" viewBox="0 0 24 24" class="xa-refresh-svg-icon" :style="circularStyle">
      <path d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"/>
    </svg>
    <circular v-show="refreshing" :size="24" :color="circularColor" :border-width="2"></circular>
  </div>
</template>

<script>
import Drag from '../../utils/drag'
import * as domUtil from '../../utils/dom'
import circular from '../circular'
const LENGTH = 130 // 下拉最大长度
const INITY = -68  // 初始化Y轴位置
const VALID = 130  // 有效下拉距离
export default {
  components: { circular },
  props: {
    refreshing: {
      type: Boolean,
      default: false
    },
    color: {
      type: [String, Array],
      default: 'red'
    }
  },
  data () {
    return {
      y: 0,
      draging: false,
      state: 'pending',
      trigger: null,
      number: 0,
      timer: 0,
      circularColor: this.color[0] || this.color || ''
    }
  },
  computed: {
    initColor () {
      return this.isColorArray ? this.color[0] : this.color
    },
    isColorArray () {
      return Array.isArray
        ? Array.isArray(this.color)
        : this.color instanceof Array
    },
    refreshStyle () {
      let style = {color: this.initColor}
      if (!this.refreshing && this.draging) {
        let translate3d = 'translate3d(0, ' + (this.y + INITY) + 'px, 0) '
        style['-webkit-transform'] = style['transform'] = translate3d
      }
      return style
    },
    circularStyle () {
      let style = {}
      if (!this.refreshing && this.draging) {
        let percentage = this.y / LENGTH
        let rotate = 'rotate(' + (360 * percentage) + 'deg)'
        let opacity = this.y / Math.abs(INITY)
        style['-webkit-transform'] = style['transform'] = rotate
        style['opacity'] = opacity
      }
      return style
    },
    refreshClass () {
      let classNames = []
      switch (this.state) {
        case 'pending':
          break
        case 'ready':
          classNames.push('xa-refresh-control-noshow')
          break
        case 'dragStart':
          classNames.push('xa-refresh-control-hide')
          break
        case 'dragAnimate':
          classNames.push('xa-refresh-control-animate')
          classNames.push('xa-refresh-control-hide')
          break
        case 'refreshAnimate':
          classNames.push('xa-refresh-control-animate')
          classNames.push('xa-refresh-control-noshow')
          break
      }
      if (this.refreshing) classNames.push('xa-refresh-control-refreshing')
      return classNames
    }
  },
  mounted () {
    this.trigger = this.$el.parentNode
    this.bindDrag()
  },
  methods: {
    clearState () {
      this.state = 'ready'
      this.draging = false
      this.y = 0
    },
    getScrollEventTarget (element) {
      let currentNode = element
      while (currentNode && currentNode.tagName !== 'HTML' &&
        currentNode.tagName !== 'BODY' && currentNode.nodeType === 1) {
        let overflowY = document.defaultView.getComputedStyle(currentNode).overflowY
        if (overflowY === 'scroll' || overflowY === 'auto') {
          return currentNode
        }
        currentNode = currentNode.parentNode
      }
      return window
    },
    getScrollTop (element) {
      if (element === window) {
        return Math.max(window.pageYOffset || 0, document.documentElement.scrollTop)
      } else {
        return element.scrollTop
      }
    },
    bindDrag () {
      if (!this.trigger) return
      const drager = this.drager = new Drag(this.trigger)
      this.state = 'ready'
      drager.start(() => {
        if (this.refreshing) return
        this.state = 'dragStart'
        const scrollTop = this.getScrollTop(this.getScrollEventTarget(this.$el))
        if (scrollTop === 0) this.draging = true
      }).drag((pos, event) => {
        const scrollTop = this.getScrollTop(this.getScrollEventTarget(this.$el))
        if (pos.y < 5 || this.refreshing || scrollTop !== 0) return // 消除误差

        if (scrollTop === 0 && !this.draging) {
          this.draging = true
          drager.reset(event)
        }

        if (this.draging && pos.y > 0) {
          event.preventDefault()
          event.stopPropagation()
        }

        this.y = pos.y / 2
        if (this.y < 0) this.y = 1
        if (this.y > LENGTH) this.y = LENGTH
      }).end((pos, event) => {
        if (!pos.y || pos.y < 5) {
          this.clearState()
          return // 消除误差
        }
        let canRefresh = pos.y > VALID && this.draging
        this.state = 'dragAnimate'
        if (canRefresh) {
          this.draging = false
          this.$emit('refresh')
        } else {
          this.y = 0
          domUtil.transitionEnd(this.$el, this.clearState.bind(this))
        }
      })
    },
    unbindDrag () {
      if (!this.drager) return
      this.drager.destory()
      this.drager = null
    },
    autoChangeColor () {
      if (!this.isColorArray) return
      this.timer = setInterval(() => {
        this.number++
        this.circularColor = this.color[this.number % this.color.length]
      }, 1666)
    },
    clearAutoChangeColor () {
      if (!this.isColorArray) return
      clearInterval(this.timer)
      this.number = 0
      this.circularColor = this.initColor
    }
  },
  watch: {
    refreshing (val) {
      if (!val) {
        this.clearAutoChangeColor()
        domUtil.transitionEnd(this.$el, this.clearState.bind(this))
      } else {
        this.autoChangeColor()
        this.state = 'refreshAnimate'
      }
    },
    trigger (trigger, oldTrigger) {
      if (trigger === oldTrigger) return
      this.unbindDrag()
      this.bindDrag()
    }
  },
  beforeDestroy () {
    this.clearAutoChangeColor()
    this.unbindDrag()
  }
}
</script>

<style scoped>
.xa-refresh-control{
  display: flex;
  margin: 0 auto;
  width: 40px;
  height: 40px;
  color: red;
  align-items: center;
  justify-content: center;
  background-color: #FFF;
  border-radius: 50%;
  box-shadow: rgba(0, 0, 0, 0.117647) 0px 1px 6px, rgba(0, 0, 0, 0.117647) 0px 1px 4px;
  position: absolute;
  left: 50%;
  margin-left: -18px;
  margin-top: 24px;
  z-index: 90;
}

.xa-refresh-control .icon {
    display: inline-block;
    vertical-align: middle;
  }

.xa-refresh-svg-icon {
  display: inline-block;
  width: 28px;
  height: 28px;
  fill: currentColor;
  user-select: none;
}

.xa-refresh-control-animate{
   transition: all 0.45s ease;
}

.xa-refresh-control-hide{
  opacity: 1;
  transform: translate3d(0, -68px, 0);
}

.xa-refresh-control-noshow{
  opacity: 0;
  transform: scale(0.01);
}

.xa-refresh-control-refreshing {
  transform: scale(1);
  opacity: 1;
}

</style>

