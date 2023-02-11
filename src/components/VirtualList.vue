<template>
  <div
    ref="list"
    :style="{ height }"
    class="infinite-list-container"
    @scroll="throttleScrollEvent"
  >
    <div
      ref="phantom"
      class="infinite-list-phantom"
    />
    <div
      ref="content"
      class="infinite-list"
    >
      <div
        v-for="item in visibleData"
        :id="item._index"
        :key="item._index"
        ref="items"
        class="infinite-list-item"
      >
        <slot
          ref="slot"
          :item="item.item"
        />
      </div>
    </div>
  </div>
</template>

<script>
import { throttle } from 'lodash'

export default {
  name: 'VirtualList',
  props: {
    // 所有列表数据
    listData: {
      type: Array,
      default: () => []
    },
    // 预估高度
    estimatedItemSize: {
      type: Number,
      required: true
    },
    // 缓冲区比例
    bufferScale: {
      type: Number,
      default: 1
    },
    // 容器高度
    height: {
      type: String,
      default: '100%'
    },
  },
  created() {
    this.initPositions()
  },
  mounted() {
    // this.$el: <div class="infinite-list-container"></div>
    this.screenHeight = this.$el.clientHeight
    this.start = 0
    this.end = this.start + this.visibleCount
  },
  updated() {
    this.$nextTick(function() {
      if (!this.$refs.items || this.$refs.items.length === 0) {
        return
      }
      this.updateItemsSize()
      // 更新列表总高度
      let height = this.positions[this.positions.length - 1].bottom
      this.$refs.phantom.style.height = height + 'px'
      // 更新真实偏移量
      this.setStartOffset()
    })
  },
  data() {
    return {
      //可视区域高度
      screenHeight: 0,
      //起始索引
      start: 0,
      //结束索引
      end: 0,
      // 列表项的信息
      positions: [],
      throttleScrollEvent: throttle(this.scrollEventHandler, 800),
    }
  },
  computed: {
    _listData() {
      return this.listData.map((item, index) => {
        return {
          _index: `_${index}`,
          item
        }
      })
    },
    // 可视区中可容纳的列表项的数量
    visibleCount() {
      return Math.ceil(this.screenHeight / this.estimatedItemSize)
    },
    // 可视区之上的缓存数据
    aboveCount() {
      return Math.min(this.start, this.bufferScale * this.visibleCount)
    },
    // 可视区之下的缓存数据
    belowCount() {
      return Math.min(this.listData.length - this.end, this.bufferScale * this.visibleCount)
    },
    // 可视区 + 可视区之上的缓存数据 + 可视区之下的缓存数据
    visibleData() {
      let start = this.start - this.aboveCount
      let end = this.end + this.belowCount
      return this._listData.slice(start, end)
    }
  },
  methods: {
    // 初始化列表项预估的位置信息
    initPositions() {
      this.positions = this.listData.map((_, index) => ({
          index,
          height: this.estimatedItemSize,
          top: index * this.estimatedItemSize,
          bottom: (index + 1) * this.estimatedItemSize
        })
      )
    },
    // 获取真实元素大小，更新列表项的高度等各项信息
    updateItemsSize() {
      const nodes = this.$refs.items
      nodes.forEach((node) => {
        const rect = node.getBoundingClientRect()
        const height = rect.height
        const index = Number(node.id.slice(1))
        const oldHeight = this.positions[index].height
        const dValue = oldHeight - height
        // 如果存在差值，则更新列表项的高度等各项信息
        if (dValue) {
          this.positions[index].bottom = this.positions[index].bottom - dValue
          this.positions[index].height = height
          for (let k = index + 1; k < this.positions.length; k++) {
            this.positions[k].top = this.positions[k - 1].bottom
            this.positions[k].bottom = this.positions[k].bottom - dValue
          }
        }
      })
    },
    // 获取当前的偏移量
    setStartOffset() {
      let startOffset
      if (this.start >= 1) {
        let size = this.positions[this.start].top -
          (this.positions[this.start - this.aboveCount] ? this.positions[this.start - this.aboveCount].top : 0)
        startOffset = this.positions[this.start - 1].bottom - size
      } else {
        startOffset = 0
      }
      // 鼠标滚轮向下滚动时，包含列表项的父元素也会向上移动，因此需要使用 transform 向下移动该元素的位置
      this.$refs.content.style.transform = `translate3d(0, ${ startOffset }px, 0)`
    },
    // 获取列表起始索引
    getStartIndex(scrollTop = 0) {
      // 查找大于 scrollTop 的某一项的 bottom
      const startIndex = this.binarySearch(this.positions, scrollTop)
      if (startIndex >= 0) {
        return startIndex
      } else {
        console.error('binary search has some errors')
      }
    },
    // 使用二分查找将时间复杂度降低为 O(logN)
    binarySearch(positions, scrollTop) {
      let start = 0
      let end = positions.length - 1

      while (start <= end) {
        const middle = Math.floor((start + end) / 2)
        if (positions[middle].bottom === scrollTop) {
          return middle + 1 //
        } else if (positions[middle].bottom < scrollTop) {
          start = middle + 1
        } else if (positions[middle].bottom > scrollTop) {
          if (middle === 0) {
            return middle
          }
          if (positions[middle - 1].bottom < scrollTop) {
            return middle
          }
          end = middle - 1
        }
      }

      return -1
    },
    // 滚动事件处理函数
    scrollEventHandler() {
      // 当前滚动位置
      let scrollTop = this.$refs.list.scrollTop
      // 此时的开始索引
      this.start = this.getStartIndex(scrollTop)
      // 此时的结束索引
      this.end = this.start + this.visibleCount
    },
  }
}
</script>

<style scoped>
.infinite-list-container {
  overflow: auto;
  position: relative;
  -webkit-overflow-scrolling: touch;
}

.infinite-list-phantom {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}

.infinite-list {
  left: 0;
  right: 0;
  top: 0;
  position: absolute;
}

.infinite-list-item {
  padding: 5px;
  color: #555;
  box-sizing: border-box;
  border-bottom: 1px solid #999;
}
</style>
