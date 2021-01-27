<template>
  <div class="container" ref="container">
    <canvas
      ref="canvas"
      @click="onClick"
      @mousemove="onMousemove"
      @dblclick="onDbClick"
      @mousedown="onMousedown"
      @mouseup="onMouseup"
    ></canvas>
  </div>
</template>

<script>
export default {
  name: 'Index',
  data () {
    return {
      width: 0,
      height: 0,
      ctx: null,
      pointsArr: [],
      tmpPoint: null,
      isClosePath: false,
      dragPointIndex: -1,
      isMousedown: false,
      isInPolygon: false,
      startPos: {
        x: 0,
        y: 0
      },
      endPos: {
        x: 0,
        y: 0
      },
      cachePointsArr: []
    }
  },
  mounted () {
    this.init()
  },
  methods: {
    /**
     * @Author: 王林25
     * @Date: 2021-01-26 16:08:19
     * @Desc: 初始化
     */
    init () {
      let { width, height } = this.$refs.container.getBoundingClientRect()
      this.width = width
      this.height = height
      let canvas = this.$refs.canvas
      canvas.width = width
      canvas.height = height
      this.ctx = canvas.getContext('2d')
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 16:20:25
     * @Desc: 获取相对于画布的坐标
     */
    toCanvasPos (e) {
      let { left, top } = this.$refs.canvas.getBoundingClientRect()
      return {
        x: e.clientX - left,
        y: e.clientY - top
      }
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 16:17:40
     * @Desc: 点击事件
     */
    onClick (e) {
      let { x, y } = this.toCanvasPos(e)
      if (this.isClosePath) {
        let index = this.checkPointIndex(x, y)
        let diffX = Math.abs(this.endPos.x - this.startPos.x)
        let diffY = Math.abs(this.endPos.y - this.startPos.y)
        if (index !== -1 && diffX <= 1 && diffY <= 1 && !this.pointsArr[index].fictitious) {
          this.pointsArr.splice(index, 1)
          this.render()
        }
        return
      }
      this.pointsArr.push({
        x,
        y
      })
      this.render()
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 17:42:36
     * @Desc: 渲染
     */
    render () {
      // 先去掉之前插入的虚拟顶点
      this.pointsArr = this.pointsArr.filter((item) => {
        return !item.fictitious
      })
      if (this.isClosePath && !this.isMousedown) { // 插入虚拟顶点
        this.insertFictitiousPoints()
      }
      // 先清除画布
      this.ctx.clearRect(0, 0, this.width, this.height)
      // 顶点连线
      this.ctx.beginPath()
      let pointsArr = this.pointsArr.concat(
        this.tmpPoint ? [this.tmpPoint] : []
      )
      pointsArr.forEach((item, index) => {
        if (index === 0) {
          this.ctx.moveTo(item.x, item.y)
        } else {
          this.ctx.lineTo(item.x, item.y)
        }
      })
      this.ctx.closePath()
      this.ctx.lineWidth = 5
      this.ctx.strokeStyle = '#38a4ec'
      this.ctx.lineJoin = 'round'
      this.ctx.fillStyle = 'rgba(0, 136, 255, 0.3)'
      this.ctx.stroke()
      this.ctx.fill()
      // 绘制顶点的圆形
      if (this.isClosePath) {
        this.ctx.save()
        this.ctx.lineWidth = 2
        this.pointsArr.forEach((item) => {
          if (item.fictitious) {
            this.ctx.strokeStyle = '#fff'
            this.ctx.fillStyle = '#1791fc'
          } else {
            this.ctx.strokeStyle = '#1791fc'
            this.ctx.fillStyle = '#fff'
          }
          this.ctx.beginPath()
          this.ctx.arc(item.x, item.y, 6, 0, 2 * Math.PI)
          this.ctx.fill()
          this.ctx.stroke()
        })
        this.ctx.restore()
      }
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 16:54:56
     * @Desc: 插入虚拟顶点
     */
    insertFictitiousPoints () {
      if (!this.isClosePath || this.isMousedown) {
        return
      }
      // 生成虚拟顶点，跟创建线段一样的逻辑，只是计算的是线段的中点位置
      let points = []
      let arr = this.pointsArr
      let len = arr.length
      for (let i = 0; i < len - 1; i++) {
        let p1 = arr[i]
        let p2 = arr[i + 1]
        points.push({
          x: (p1.x + p2.x) / 2,
          y: (p1.y + p2.y) / 2,
          fictitious: true// 这个字段标志是否是虚拟顶点
        })
      }
      points.push({
        x: (arr[len - 1].x + arr[0].x) / 2,
        y: (arr[len - 1].y + arr[0].y) / 2,
        fictitious: true
      })

      // 插入到顶点数组里
      let newArr = []
      for (let i = 0; i < this.pointsArr.length; i++) {
        newArr.push(this.pointsArr[i])
        newArr.push(points.shift())
      }

      this.pointsArr = newArr
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 18:07:09
     * @Desc: 移除虚拟顶点
     */
    removeFictitiousPoints () {
      this.pointsArr = this.pointsArr.filter((item) => {
        return !item.fictitious
      })
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 18:36:04
     * @Desc: 鼠标移动事件
     */
    onMousemove (e) {
      let { x, y } = this.toCanvasPos(e)
      if (this.isClosePath) {
        if (this.isMousedown) {
          if (this.dragPointIndex !== -1) {
            // 是虚拟顶点，转换成真实顶点
            if (this.pointsArr[this.dragPointIndex].fictitious) {
              delete this.pointsArr[this.dragPointIndex].fictitious
            }
            this.dragPointIndex = this.getTrueIndex(this.dragPointIndex)
            this.removeFictitiousPoints()
            let adsorbentPos = this.checkAdsorbent(x, y)
            this.pointsArr.splice(this.dragPointIndex, 1, {
              ...this.pointsArr[this.dragPointIndex],
              x: adsorbentPos[0],
              y: adsorbentPos[1]
            })
            this.render()
          } else if (this.isInPolygon) {
            let diffX = x - this.startPos.x
            let diffY = y - this.startPos.y
            this.pointsArr = this.cachePointsArr.map((item) => {
              return {
                ...item,
                x: item.x + diffX,
                y: item.y + diffY
              }
            })
            this.render()
          }
        }
      } else {
        if (this.tmpPoint) {
          this.tmpPoint.x = x
          this.tmpPoint.y = y
        } else {
          this.tmpPoint = {
            x,
            y
          }
        }
        this.render()
      }
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 09:51:52
     * @Desc: 判断是否需要进行吸附
     */
    checkAdsorbent (x, y) {
      let result = [x, y]
      // 吸附到线段
      let segments = this.createLineSegment()
      let nearestLineResult = this.getPintNearestLine(x, y, segments)
      if (nearestLineResult[0] <= 10) {
        let segment = nearestLineResult[1]
        let nearestPoint = this.getNearestPoint(segment[0].x, segment[0].y, segment[1].x, segment[1].y, x, y)
        if (nearestPoint) {
          result = [nearestPoint.x, nearestPoint.y]
        }
      }
      // 吸附到顶点
      let minDistance = Infinity
      this.pointsArr.forEach((item, index) => {
        if (this.dragPointIndex === index) {
          return
        }
        let distance = this.getTwoPointDistance(item.x, item.y, x, y)
        if (distance <= 10 && distance < minDistance) {
          minDistance = distance
          result = [item.x, item.y]
        }
      })
      return result
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 18:58:34
     * @Desc: 双击事件
     */
    onDbClick () {
      this.isClosePath = true
      this.tmpPoint = null
      this.render()
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 19:39:41
     * @Desc: 鼠标按下事件
     */
    onMousedown (e) {
      if (!this.isClosePath) {
        return
      }
      this.isMousedown = true
      let { x, y } = this.toCanvasPos(e)
      this.startPos.x = x
      this.startPos.y = y
      this.cachePointsArr = this.pointsArr.map((item) => {
        return {
          ...item
        }
      })
      this.dragPointIndex = this.checkPointIndex(x, y)
      this.isInPolygon = this.checkInPolygon(x, y)
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 17:47:47
     * @Desc: 获取顶点真实索引
     */
    getTrueIndex (index) {
      let prevFictitiousCount = 0
      for (let i = 0; i < index; i++) {
        if (this.pointsArr[i].fictitious) {
          prevFictitiousCount++
        }
      }
      return index - prevFictitiousCount
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 19:40:57
     * @Desc: 检测鼠标在哪个顶点内
     */
    checkPointIndex (x, y) {
      let result = -1
      this.pointsArr.forEach((item, index) => {
        this.ctx.beginPath()
        this.ctx.arc(item.x, item.y, 6, 0, 2 * Math.PI)
        if (this.ctx.isPointInPath(x, y)) {
          result = index
        }
      })
      return result
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 20:04:01
     * @Desc: 检测是否在多边形内
     */
    checkInPolygon (x, y) {
      this.ctx.beginPath()
      this.pointsArr.forEach((item, index) => {
        if (index === 0) {
          this.ctx.moveTo(item.x, item.y)
        } else {
          this.ctx.lineTo(item.x, item.y)
        }
      })
      this.ctx.closePath()
      return this.ctx.isPointInPath(x, y)
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-26 19:52:58
     * @Desc: 鼠标松开事件
     */
    onMouseup (e) {
      let { x, y } = this.toCanvasPos(e)
      this.endPos.x = x
      this.endPos.y = y
      this.isMousedown = false
      this.dragPointIndex = -1
      this.cachePointsArr = []
      this.render()
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 09:59:51
     * @Desc: 获取两点距离公式
     */
    getTwoPointDistance (x1, y1, x2, y2) {
      return Math.sqrt(Math.pow(x1 - x2, 2) + Math.pow(y1 - y2, 2))
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 10:21:51
     * @Desc: 创建线段
     */
    createLineSegment () {
      let result = []
      // 创建线段
      let arr = this.pointsArr
      let len = arr.length
      for (let i = 0; i < len - 1; i++) {
        let p1 = arr[i]
        let p2 = arr[i + 1]
        // 跳过两个端点相同的线段
        if (p1.x === p2.x && p1.y === p2.y) {
          continue
        }
        result.push([
          p1,
          p2
        ])
      }
      // 加上起点和终点组成的线段
      result.push([
        arr[len - 1],
        arr[0]
      ])
      // 去掉包含当前拖动点的线段
      if (this.dragPointIndex !== -1) {
        // 如果拖动的是起点，那么去掉第一条和最后一条线段
        if (this.dragPointIndex === 0) {
          result.splice(0, 1)
          result.splice(-1, 1)
        } else { // 其余中间的点则去掉前一根和后一根
          result.splice(this.dragPointIndex - 1, 2)
        }
      }
      return result
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 10:37:22
     * @Desc: 计算里某个点最近的线段
     */
    getPintNearestLine (x, y, lineSegments) {
      let minNum = Infinity
      let minLine
      for (let i = 0; i < lineSegments.length; i++) {
        let item = lineSegments[i]
        let a = item[0]
        let b = item[1]
        let d = this.getLinePointDistance(a.x, a.y, b.x, b.y, x, y)
        if (d < minNum) {
          minNum = d
          minLine = item
        }
      }
      return [minNum, minLine]
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 10:39:40
     * @Desc: 获取点到直线的距离
     */
    getLinePointDistance (x1, y1, x2, y2, x, y) {
      // 直线垂直于x轴
      if (x1 === x2) {
        return Math.abs(x - x1)
      } else {
        let B = -1
        let A, C
        A = (y2 - y1) / (x2 - x1)
        C = 0 - B * y1 - A * x1
        return Math.abs((A * x + B * y + C) / Math.sqrt(A * A + B * B))
      }
    },

    /**
     * @Author: 王林25
     * @Date: 2021-01-27 14:25:54
     * @Desc: 获取线段上离某个点最近的点
     */
    getNearestPoint (x1, y1, x2, y2, x0, y0) {
      let k = (y2 - y1) / (x2 - x1)
      let x = (k * k * x1 + k * (y0 - y1) + x0) / (k * k + 1)
      let y = k * (x - x1) + y1
      // 判断该点的x坐标是否在线段的两个端点内
      let min = Math.min(x1, x2)
      let max = Math.max(x1, x2)
      // 如果在线段内就是我们要的点
      if (x >= min && x <= max) {
        return {
          x,
          y
        }
      } else { // 否则返回最近的端点
        return null
      }
    }
  }
}
</script>

<style lang="less">
.container {
  width: 100%;
  height: 100%;
  display: flex;
  position: relative;
  justify-content: center;
  flex-direction: column;
}
</style>
