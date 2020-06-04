<template>
  <div id="app" class="container">
    <div v-if="dialog"
      :style="'top:' + tooltip.y + 'px;' + 'left:' + tooltip.x + 'px;'"
      @mousemove="dialog = true"
      @mouseout="dialog = false"
      class="tooltip"
    >
      {{tooltip.name}}
    </div>
  </div>
</template>

<script type="text/ecmascript-6">
  import * as d3 from 'd3'

  export default {
    name: 'app',
    data () {
      return {
        relation: {},
        cname: '',
        dialog: false,
        tooltip: null
      }
    },
    methods: {
      showd3 () {
        const ts = this
//        获取body高度和宽度
        let height = document.body.clientHeight
        let width = document.body.clientWidth
//        节点大小（圆圈大小）
        const nodeSize = 50
//        初始化时连接线的距离长度
        const linkDistance = 130
//        赋值数据集
        var nodes = this.relation.nodes
        var links = this.relation.links

//      设置画布，获取id为app的对象，添加svg，这里的图像用了svg，意为可缩放矢量图形，它与其他图片格式相比较，svg更加小，因为是矢量图，放大不会失帧。具体可以自行百度svg相关知识
        var svg = d3.select('#app').append('svg')
          .attr('class', 'svg')//给svg设置了一个class样式，主要作用是长宽设置为100%
//        设置力布局，使用d3 v4版本的力导向布局
        var force = d3.forceSimulation()
          .force('center', d3.forceCenter(width / 2, height / 2))//设置力导向布局的中心点，创建一个力中心，设置为画布长宽的一半，所以拓扑图会在画布的中心点
          .force('charce', d3.forceManyBody().strength(0))//节点间的作用力，如果不设置.strength(-60）的话，默认是-30
          .force('collide', d3.forceCollide(nodeSize * 2))//使用默认的半径创建一个碰撞作用力。radius默认所有的节点都为1

//        设置缩放
//        svg下嵌套g标签，缩放都在g标签上进行
        var g = svg.append('g')
//        d3.zoom是设置缩放，pc端是滚轮进行缩放，在移动端可以通过两指进行缩放
        var zoomObj = d3.zoom()
          .scaleExtent([0.5, 1.2]) // 设置缩放范围
          .on('zoom', () => {
            //监听zoom事件，zoom发生时，调用该方法
            const transform = d3.event.transform //获取缩放和偏移的数据，不懂得同学可以自行通过console.log(d3.event.transform)滑动滚轮查看数据变化
            g.attr('transform', transform)   // 设置缩放和偏移量 transform对象自带toString()方法
          })
          .on('end', () => {
//            该方法在缩放时间结束后回调
            // code
          })
        svg.call(zoomObj)
//        绘制箭头
        //箭头
        // eslint-disable-next-line no-unused-vars
        var marker =
          g.append('marker')
            .attr('id', 'resolved')
            //.attr("markerUnits","strokeWidth")//设置为strokeWidth箭头会随着线的粗细发生变化
            .attr('markerUnits', 'userSpaceOnUse')//用于确定marker是否进行缩放。取值strokeWidth和userSpaceOnUse，
            .attr('viewBox', '0 0 10 10')//坐标系的区域
            .attr('refX', nodeSize)//箭头坐标
            .attr('refY', 5)
            .attr('markerWidth', 12)//标识的大小
            .attr('markerHeight', 12)
            .attr('orient', 'auto')//绘制方向，可设定为：auto（自动确认方向）和 角度值
            // .attr('stroke-width', 2)//箭头宽度
            .append('path')
            .attr('d', 'M0,0L10,5L0,10')//箭头的路径
            .attr('fill', '#ff7438')//箭头颜色

//        设置连线
        let animationId = []
        var edgesLine = g.selectAll('line')
          .data(links)
          .enter()
          .append('path')
          .attr('class', 'edgelabel')//添加class样式
          .style('stroke', '#ff7438')//添加颜色
          .style('stroke-width', 3)//连接线粗细度
          .attr('marker-end', 'url(#resolved)')//设置线的末尾为刚刚的箭头
          .attr('cursor', 'pointer')
          .on('mousemove', (d, i) => {
            console.log(d)
            ts.dialog = true
            ts.tooltip = {}
            ts.tooltip.x = event.x + 5
            ts.tooltip.y = event.y + 5
            ts.tooltip.name = d.relation
            node.style('opacity', node => {
              if (d.target.name === node.name || d.source.name === node.name) {
                return 1
              } else {
                return 0.5
              }
            })
            edgesLine.style('opacity', line => {
              if (line.target.name === d.target.name && line.source.name === d.source.name) {
                return 1
              } else {
                return 0.1
              }
            })
            edgesLine.attr('stroke-dasharray', line => {
              if (line.source.name === d.source.name && line.target.name === d.target.name) {
                return [5, 2]
              } else {
                return 0
              }
            })
            var index = 0
            let animation = function () {
              index += 1
              edgesLine.attr('stroke-dashoffset', line => {
                if (line.source.name === d.source.name && line.target.name === d.target.name) {
                  return index * -0.1
                } else {
                  return 0
                }
              })
              animationId.push(window.requestAnimationFrame(animation))
            }
            animation()
          })
          .on('mouseout', (d, i) => {
            ts.dialog = false
            node.style('opacity', 1)
            edgesLine.style('opacity', 1)
            edgesLine.attr('stroke-dasharray', line => {
              return 0
            })
            for (let i = 0; i < animationId.length; i++) {
              window.cancelAnimationFrame(animationId[i])
            }
          })
//        设置拖拽
        var drag = d3.drag()
          .on('start', (d, i) => {
            if (!d3.event.active) {
//              拖拽开始回调
              force.alphaTarget(0.1).restart() // 这个方法可以用在在交互时重新启动仿真，比如拖拽了某个节点，重新进行布局。这个必须要进行设置不然会拖动不了。
            }
            d.fixed = true //偏移后固定不动
//            d3.event.sourceEvent.stopPropagation()
            d.fx = d.x//记录当前默认位置（x - 节点当前的 x-位置，如果要为某个节点设置默认的位置，则需要为该节点设置如下两个属性:fx =x位置）
            d.fy = d.y
          })
          .on('drag', (d, i) => {
//            拖动时，设置拖动后默认位置的x，y
            d.fx = d3.event.x
            d.fy = d3.event.y
          })
          .on('end', (d, i) => {
//            拖动结束后
            if (!d3.event.active) {
              force.alphaTarget(0)
            }
          })

//        设置节点
        var node = g.selectAll('circle')
          .data(nodes)
          .enter()
          .append('circle')
          .attr('r', nodeSize)
          .attr('class', (d, i) => {
//            为不同的节点设置不同的css样式
            if (d.type === 0) {
              return 'nodeOrange'
            } else if (d.type === 1) {
              return 'nodeBlue'
            } else if (d.type === 2) {
              return 'nodeRed'
            }
          })
          .attr('id', (d, i) => {
//            为每个节点设置不同的id
            return 'node' + i
          })
          .on('mousemove', (d, i) => {
            ts.dialog = true
            ts.tooltip = d
            ts.tooltip.x = event.x + 5
            ts.tooltip.y = event.y + 5
            var nodeSet = new Set()
            edgesLine.style('opacity', function (line) {
              if (line.source.name === d.name || line.target.name === d.name) {
                nodeSet.add(line.source.name)
                nodeSet.add(line.target.name)
                return 1
              } else {
                return 0.1
              }
            })
            edgesLine.attr('stroke-dasharray', line => {
              if (line.source.name === d.name || line.target.name === d.name) {
                return [5, 2]
              } else {
                return 0
              }
            })
            var index = 0
            let animation = function () {
              index += 1
              edgesLine.attr('stroke-dashoffset', line => {
                if (line.source.name === d.name || line.target.name === d.name) {
                  return index * -0.1
                } else {
                  return 0
                }
              })
              animationId.push(window.requestAnimationFrame(animation))
            }
            animation()
            node.style('opacity', node => {
              if (nodeSet.has(node.name)) {
                return 1
              } else {
                return 0.5
              }
            })
            d3.select('#node' + i).raise()
          })
          .on('mouseout', (d, i) => {
            ts.dialog = false
            edgesLine.style('opacity', 1)
            node.style('opacity', 1)
            edgesLine.attr('stroke-dasharray', line => {
              return 0
            })
            for (let i = 0; i < animationId.length; i++) {
              window.cancelAnimationFrame(animationId[i])
            }
          })
          .call(drag)//监听拖动事件
//        设置node和edge
        force.nodes(nodes)
          .force('link', d3.forceLink(links).distance(linkDistance).strength(1))
          .restart()
//        tick 表示当运动进行中每更新一帧时
        force.on('tick', function () {
//          //更新连接线的位置
          edgesLine.attr('d', function (d) {
            var path = 'M ' + d.source.x + ' ' + d.source.y + ' L ' + d.target.x + ' ' + d.target.y
            return path
          })

          //更新结点图片和文字
          node.attr('cx', function (d) {
            return d.x
          })
          node.attr('cy', function (d) { return d.y })

          // nodeText.attr('x', function (d) { return d.x })
          // nodeText.attr('y', function (d) { return d.y })
          // //动态更新sptan 的x的坐标
          // nodeText.selectAll('tspan')
          //   .attr('x', function (d) {
          //     return d.x
          //   })
        })
      }
    },
    created () {
      this.$nextTick(() => {
        this.relation = JSON.parse('{"nodes":[{"name":"BetterVicky","type":0},{"name":"杭州市高新区（滨江）萧宏小额贷款有限公司","type":1},{"name":"浙江合德建设有限公司","type":1},{"name":"杭州萧山党山企业担保有限公司","type":1},{"name":"林爱萍","type":2},{"name":"申盛集团有限公司","type":2}],"links":[{"source":0,"target":1,"relation":"对外投资"},{"source":0,"target":2,"relation":"对外投资"},{"source":0,"target":3,"relation":"对外投资"},{"source":1,"target":2,"relation":"合作"},{"source":4,"target":0,"relation":"投资"},{"source":5,"target":0,"relation":"投资"}],"code":200,"message":"请求成功"}')
        console.log(this.relation)
        this.showd3()
      })
    }
  }
</script>

<style lang="stylus" rel="stylesheet/stylus">
  .container
    height 100%
    .tooltip
      position absolute
      top 0
      left 0
      z-index 10
      min-width 200px
      max-width 500px
      min-height 200px
      background #f0f0f0
    .exit
      position absolute
      top 20px
      left 20px
      width 60px
      height 25px
      background #0583f2
      border none
      border-radius 2px
      color #fff
      z-index 200
      &:hover
        background #1e82d9

  .labeltext
    font-size: 16px;
    font-family: SimSun;
    fill: #ff7438;

  .nodetext
    font-size: 12px;
    font-family: SimSun;
    fill: #fff;
    position relative

  .linetext
    font-size: 12px
    font-weight bold
    font-family: SimSun
    fill: #000 !important
    color #000
    fill-opacity: 1.0

  .svg
    position relative
    width 100%
    height 100%

  .edgepath
    pointer-events none
    stroke-width 0.5px

  .nodeOrange
    position relative
    fill #ff7438 !important

  .nodeRed
    position relative
    fill #ff4238 !important

  .nodeBlue
    position relative
    fill #029ed9 !important
</style>
