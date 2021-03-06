<template>
  <scroll @scroll="scroll"
          :listen-scroll="listenScroll"
          :probe-type="probeType"
          :data="data"
          class="listview"
          ref="listview">
          <!--:data="data"数据是异步请求，保证数据改变后可以重新去计算scroll的高度-->
          <!--:listen-scroll="listenScroll"将this.listenScroll = true传给子组件-->
          <!--@scroll="scroll"监听子组件是否滚动-->
    <!--左边列表 start-->
    <ul>
      <li v-for="group in data" class="list-group" ref="listGroup">
        <h2 class="list-group-title">{{group.title}}</h2>
        <uL>
          <li @click="selectItem(item)" v-for="item in group.items" class="list-group-item">
          	<!--@click="selectItem(item)"监听点击事件，emit给singer组件，用作跳转到singerDetail的路由参数-->
            <img class="avatar" v-lazy="item.avatar">
            <span class="name">{{item.name}}</span>
          </li>
        </uL>
      </li>
    </ul>
    <!--左边列表 end-->
    <!--右边快速定位侧边栏 start-->
    <div class="list-shortcut" 
    	   @touchstart.stop.prevent="onShortcutTouchStart" 
    	   @touchmove.stop.prevent="onShortcutTouchMove"
         @touchend.stop>
      <ul>
        <li v-for="(item, index) in shortcutList" 
        	  :data-index="index" class="item"
            :class="{'current':currentIndex===index}">
            {{item}}
        </li>
        <!--:data-index="index"获取点击元素需要拿到下标，:class="{'current':currentIndex===index}"高亮-->
      </ul>
    </div>
    <!--右边快速定位侧边栏 end-->
    <!--fix定位的title start-->
    <div class="list-fixed" ref="fixed" v-show="fixedTitle">
      <div class="fixed-title">{{fixedTitle}} </div>
    </div>
    <!--fix定位的title end-->
    <!--loading start-->
    <div v-show="!data.length" class="loading-container">
      <loading></loading>
    </div>
    <!--loading end-->
  </scroll>
</template>

<script type="text/ecmascript-6">
  import Scroll from 'base/scroll/scroll'
  import Loading from 'base/loading/loading'
  import {getData} from 'common/js/dom'

  const TITLE_HEIGHT = 30
  const ANCHOR_HEIGHT = 18

  export default {
    props: {
    	// 接收数据
      data: {
        type: Array,
        default: []
      }
    },
    computed: {
      shortcutList() {
        return this.data.map((group) => {
          return group.title.substr(0, 1)
        })
      },
      fixedTitle() {
        if (this.scrollY > 0) {
        	// 此时是拖到顶部时就不显示fixedTitle
          return ''
        }
        return this.data[this.currentIndex] ? this.data[this.currentIndex].title : ''
      }
    },
    data() {
      return {
        scrollY: -1,
        currentIndex: 0,
        diff: -1
      }
    },
    created() {
      this.probeType = 3
       // 实时监听滚动到哪个索引的区域了,实时派发 scroll 事件，不节流的方式传3
      this.listenScroll = true
      this.touch = {}
      this.listHeight = []
    },
    methods: {
    	// 右侧边栏快速定位列表点击
      onShortcutTouchStart(e) {
        let anchorIndex = getData(e.target, 'index')
        // 获取当前点击元素的下标
        let firstTouch = e.touches[0]
        this.touch.y1 = firstTouch.pageY
        // 触碰开始位置y轴偏移
        this.touch.anchorIndex = anchorIndex
        this._scrollTo(anchorIndex)
      },
      
      // 右侧边栏快速定位列表滑动
      onShortcutTouchMove(e) {
        let firstTouch = e.touches[0]
        // 触碰开始位置
        this.touch.y2 = firstTouch.pageY
        // 触碰开始位置y轴偏移
        let delta = (this.touch.y2 - this.touch.y1) / ANCHOR_HEIGHT | 0
        // 偏差/锚点高度，再向下取整
        let anchorIndex = parseInt(this.touch.anchorIndex) + delta
				// 当前滑动到的锚点
        this._scrollTo(anchorIndex)
      },
      
      refresh() {
        this.$refs.listview.refresh()
      },
      
      scroll(pos) {
        this.scrollY = pos.y
      },
      
      // 计算每个group区块的高度，用数组listHeight存放，比区块多一个数据
      _calculateHeight() {
        this.listHeight = []
        const list = this.$refs.listGroup
        let height = 0
        this.listHeight.push(height)
        for (let i = 0; i < list.length; i++) {
          let item = list[i]
          height += item.clientHeight
          this.listHeight.push(height)
        }
      },
      
      // 处理滑动时,index边界的情况
      _scrollTo(index) {
      	// null时,不操作
        if (!index && index !== 0) {
          return
        }
        if (index < 0) {
        	// 超过顶部
          index = 0
        } else if (index > this.listHeight.length - 2) {
        	 // 超过底部
          index = this.listHeight.length - 2
        }
        this.scrollY = -this.listHeight[index]
        this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0)
      },
      
      selectItem(item) {
      	this.$emit('select', item)
      }
    },
    
    watch: {
      data() {
        setTimeout(() => {
          this._calculateHeight()
        }, 20)
        // watch: 当data发生变化后，计算每个group的高度
      },
      scrollY(newY) {
        const listHeight = this.listHeight
        // 当滚动到顶部，newY>0
        if (newY > 0) {
          this.currentIndex = 0
          return
        }
        // 在中间部分滚动
        for (let i = 0; i < listHeight.length - 1; i++) {
        	// 每一个元素我们都添加了上限和下限,第一个group的下限是第二个group上限,所以总共要多一个,-1
        	// 下限
          let height1 = listHeight[i]
          // 上限
          let height2 = listHeight[i + 1]
          if (-newY >= height1 && -newY < height2) {
          	// newY向上滑是负值,所以-newY是正.
            this.currentIndex = i
            this.diff = height2 + newY
            // 上限值height2,newY为滚动的值,是负值,diff相当于:区块上限-滚过的距离
            return
          }
        }
        // 当滚动到底部，且-newY大于最后一个元素的上限
        this.currentIndex = listHeight.length - 2
      },
      
      // 区块的上限与滚动距离的差距,设置fixtitle的向上偏移(分组标题顶上去的效果)
      diff(newVal) {
        let fixedTop = (newVal > 0 && newVal < TITLE_HEIGHT) ? newVal - TITLE_HEIGHT : 0
        if (this.fixedTop === fixedTop) {
          return
        }
        this.fixedTop = fixedTop
        this.$refs.fixed.style.transform = `translate3d(0,${fixedTop}px,0)`
      }
    },
    components: {
      Scroll,
      Loading
    }
  }

</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"

  .listview
    position: relative
    width: 100%
    height: 100%
    overflow: hidden
    background: $color-background
    .list-group
      padding-bottom: 30px
      .list-group-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
      .list-group-item
        display: flex
        align-items: center
        padding: 20px 0 0 30px
        .avatar
          width: 50px
          height: 50px
          border-radius: 50%
        .name
          margin-left: 20px
          color: $color-text-l
          font-size: $font-size-medium
    .list-shortcut
      position: absolute
      z-index: 30
      right: 0
      top: 50%
      transform: translateY(-50%)
      width: 20px
      padding: 20px 0
      border-radius: 10px
      text-align: center
      background: $color-background-d
      font-family: Helvetica
      .item
        padding: 3px
        line-height: 1
        color: $color-text-l
        font-size: $font-size-small
        &.current
          color: $color-theme
    .list-fixed
      position: absolute
      top: 0
      left: 0
      width: 100%
      .fixed-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
    .loading-container
      position: absolute
      width: 100%
      top: 50%
      transform: translateY(-50%)
</style>
