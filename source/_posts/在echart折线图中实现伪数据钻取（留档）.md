---
title: 在echart折线图中实现伪数据钻取（留档）
date: 2022-03-11 14:12:49
category:
tags:
---

太复杂了懒得一步步分析反正注释够多先原样复制粘贴要用再回来看(逃)

<!--more-->

``` vue
<template>
  <div ref="chart" :style="chartStyle">ChartLine</div>
</template>

<script>
import echarts from 'echarts'
import { convertPathToMap, SEPARATOR } from '@/utils/visual-chart'

export default {
  name: 'ChartLine',
  props: {
    data: {
      type: Array,
      required: true,
      default: () => []
    },
    rows: {
      type: Array,
      required: true,
      default: () => []
    },
    columns: {
      type: Array,
      required: true,
      default: () => []
    },
    values: {
      type: Array,
      required: true,
      default: () => []
    },
    chartType: {
      type: String,
      required: true
    },
    chartTheme: {
      type: String,
      require: true,
      default: 'default'
    },
    chartSeriesType: {
      type: String,
      require: true,
      default: ''
    },
    chartOption: {
      type: Object,
      require: false,
      default: () => ({})
    },
    chartStyle: {
      type: Object,
      require: false,
      default: () => {
        return {
          height: '500px'
        }
      }
    }
  },
  data() {
    return {
      localRows: [],
      localColumns: [],
      localValues: [],
      localData: [],
      // 连接符
      connector: '-',
      chart: null,
      calcOption: {
        xAxis: {
          type: 'category',
          axisLine: {
            lineStyle: {
              color: 'rgba(0, 0, 0, 1)'
            }
          }
        },
        yAxis: {
          type: 'value',
          axisLine: {
            lineStyle: {
              color: 'rgba(0, 0, 0, 1)'
            }
          }
        },
        tooltip: { trigger: 'axis' }
      },
      calcData: {
        legendData: [],
        xAxisData: [],
        seriesObj: {}
      },
      // 当前钻取的剩余行维
      drilldownKey: [],
      // 当前钻取的指标
      drilldownIndex: [],
      // 当前钻取的目标
      drilldownTarget: [],
      // 返回上级钻取的备份
      drilldownBackup: [],
      // 标题组
      titles: [],
      // 标题设置
      titleOption: {
        title: {
          text: '',
          textStyle: {
            fontWeight: 'normal'
          },
          bottom: 0,
          left: 20,
          triggerEvent: true
        }
      }
    }
  },
  computed: {
    watchAllProps() {
      const { rows, columns, values, data } = this
      return { rows, columns, values, data }
    }
  },
  watch: {
    watchAllProps() {
      this.init()
      this.mergeChartOption()
    },
    chartTheme() {
      this.mergeChartTheme()
    },
    chartSeriesType() {
      this.mergeChartOption()
    },
    chartOption: {
      handler(newValue, oldValue) {
        this.mergeChartOption()
      },
      deep: true
    }
  },
  mounted() {
    this.renderChart()
    this.$on('resized', this.handleResize)
    window.addEventListener('resize', this.handleResize)
  },
  created() {
    this.init()
  },
  beforeDestroy() {
    if (this.chart) {
      this.chart.dispose()
    }
    window.removeEventListener('resize', this.handleResize)
  },
  methods: {
    init() {
      this.clear()
      if (this.rows.length || this.columns.length || this.values.length) {
        this.handleDataClone()
        this.setValuesToColAndRow()
        this.handleCalcData()
      } else {
        console.warn('[Warn]: props.rows, props.columns, props.values at least one is not empty.')
      }
    },
    // clone data
    handleDataClone() {
      this.localRows = JSON.parse(JSON.stringify(this.rows))
      this.localColumns = JSON.parse(JSON.stringify(this.columns))
      this.localValues = JSON.parse(JSON.stringify(this.values))
      this.localData = Object.freeze(this.data)
    },
    // set the `values` attribute to rows and columns
    setValuesToColAndRow() {
      const rowKeys = this.localRows.map(({ key }) => key)
      const columnKeys = this.localColumns.map(({ key }) => key)
      const rowValues = this._findCategory(rowKeys, this.localData)
      const columnValues = this._findCategory(columnKeys, this.localData)
      this.localRows.forEach((row) => {
        const { key, values } = row
        this.$set(row, 'values', values || rowValues[key] || [])
      })
      this.localColumns.forEach((column) => {
        const { key, values } = column
        this.$set(column, 'values', values || columnValues[key] || [])
      })
    },
    // 计算值
    handleCalcData() {
      if (!this.localRows.length || !this.localValues.length) return
      const _rowPaths = this._combineRowPaths(
        this.localData,
        ...this.localRows.map(({ key, values }) => { return { key, values } })
      )
      const _rowKeys = this.localRows.map(({ key }) => key)
      const _colPaths = this._combineColPaths(
        ...this.localColumns.map(({ values }) => values)
      )
      const _colKeys = this.localColumns.map(({ key }) => key)
      // 行对应的条件
      const rowConditions = convertPathToMap(_rowPaths, _rowKeys)
      // 列对应的条件
      const colConditions = convertPathToMap(_colPaths, _colKeys)
      // 针对没传入行或列的处理
      !colConditions.length && colConditions.push({})
      !rowConditions.length && rowConditions.push({})
      // const xAxisData = []
      const legendData = []
      // const seriesObj = {}
      // rowConditions.forEach((rowCondition) => {
      //   xAxisData.push(Object.values(rowCondition).join(this.connector))
      // })
      colConditions.forEach((colCondition) => {
        const isEmptyCol = !Object.keys(colCondition).length
        this.localValues.forEach(({ key }) => {
          const seriesName = isEmptyCol ? key : Object.values(colCondition).join(this.connector) + this.connector + key
          legendData.push(seriesName)
          // seriesObj[seriesName] = {
          //   name: seriesName,
          //   type: this.chartType,
          //   data: []
          // }
        })
      })
      const [xAxisData, seriesObj] = this.changeToTree(_rowKeys, legendData, this.data)
      // 计算每个series数据
      // colConditions.forEach((colCondition, colConditionIndex) => {
      //   const isEmptyCol = !Object.keys(colCondition).length
      //   rowConditions.forEach((rowCondition, rowConditionIndex) => {
      //     // 当前单元对应的条件
      //     const conditions = Object.assign({}, rowCondition, colCondition)
      //     // 通过当前单元对应的条件，过滤数据
      //     const filterData = this._filterData(conditions, this.localData)
      //     // 多个值，多条数据
      //     this.localValues.forEach(({ key }) => {
      //       const value = this._reduceValue(filterData, key)
      //       const seriesName = isEmptyCol ? key : Object.values(colCondition).join(this.connector) + this.connector + key
      //       seriesObj[seriesName].data.push(value)
      //     })
      //   })
      // })
      this.calcData.legendData = legendData
      this.calcData.xAxisData = xAxisData
      this.calcData.seriesObj = seriesObj
      // console.log('xAxisData=', xAxisData, 'seriesObj=', seriesObj)
      // console.log('localValues=', this.localValues)
    },
    handleResize() {
      if (this.chart) {
        this.chart.resize()
      }
    },
    renderChart() {
      if (!this.$refs.chart) return
      let option = Object.assign({}, this.chartOption, this.calcOption, this.titleOption)
      option.title.text = this.titles.join(' => ')
      option.legend.data = this.calcData.legendData
      option.xAxis.data = this.calcData.xAxisData
      option = JSON.parse(JSON.stringify(option))
      const series = JSON.parse(JSON.stringify(Object.values(this.calcData.seriesObj)))
      this.parseChartSeries(option, series, this.chartSeriesType)
      option.series = series
      setTimeout(() => {
        if (!this.chart) {
          if (this.chartTheme !== 'default') {
            require('./themes/' + this.chartTheme + '.js')
          }
          this.chart = echarts.init(this.$refs.chart, this.chartTheme)
        }
        this.chart.clear()
        // 移除既有的点击事件
        this.chart.off('click')
        this.chart.setOption(option)
        // 点击节点后触发下钻事件
        this.chart.on('click', 'series', (params) => {
          this.clickNode(params.name)
        })
        // 点击标题后触发返回事件
        this.chart.on('click', 'title', (params) => {
          this.backUp(params)
        })
      }, 0)
    },
    mergeChartTheme() {
      if (!this.$refs.chart) return
      if (this.chart) {
        // 使用刚指定的配置项和数据显示图表
        let option = Object.assign({}, this.chartOption, this.calcOption)
        option.legend.data = this.calcData.legendData
        option.xAxis.data = this.calcData.xAxisData
        option = JSON.parse(JSON.stringify(option))
        const series = JSON.parse(JSON.stringify(Object.values(this.calcData.seriesObj)))
        this.parseChartSeries(option, series, this.chartSeriesType)
        option.series = series
        if (this.chartTheme !== 'default') {
          require('./themes/' + this.chartTheme + '.js')
        }
        this.chart.dispose()
        // 基于准备好的dom，初始化echarts实例
        this.chart = echarts.init(this.$refs.chart, this.chartTheme)
        this.chart.setOption(option)
      }
    },
    mergeChartOption() {
      if (!this.$refs.chart) return
      if (this.chart) {
        let option = Object.assign({}, this.chartOption, this.calcOption)
        option.legend.data = this.calcData.legendData
        option.xAxis.data = this.calcData.xAxisData
        option = JSON.parse(JSON.stringify(option))
        const series = JSON.parse(JSON.stringify(Object.values(this.calcData.seriesObj)))
        this.parseChartSeries(option, series, this.chartSeriesType)
        option.series = series
        this.chart.clear()
        this.chart.setOption(option, true)
      }
    },
    parseChartSeries(option, series, type) {
      if (type === 'arealine') {
        series.forEach((item) => {
          item.areaStyle = {}
        })
        option.xAxis.boundaryGap = false
      } else if (type === 'stackline') {
        series.forEach((item) => {
          item.stack = '总量'
        })
        option.xAxis.boundaryGap = false
      } else if (type === 'stackarealine') {
        series.forEach((item) => {
          item.stack = '总量'
          item.areaStyle = {}
        })
        option.xAxis.boundaryGap = false
      } else if (type === 'percentagestackarealine') {
        const sum_data = []
        for (let j = 0; j < option.xAxis.data.length; j++) {
          let sum = 0
          for (let i = 0; i < series.length; i++) {
            sum += series[i].data[j] ? Number(series[i].data[j]) : 0
          }
          sum_data[j] = sum
        }
        series.forEach((item) => {
          const serieCalcData = item.data.map((data, index) => {
            return (parseFloat(data) / parseFloat(sum_data[index]) * 100).toFixed(2)
          })
          item.data = serieCalcData
          item.stack = '总量'
          item.areaStyle = {}
        })
        option.yAxis.min = 0
        option.yAxis.max = 100
        option.xAxis.boundaryGap = false
        // option.tooltip.formatter = function(params) {
        //   let s = params[0].name + '</br>'
        //   const sum = params.reduce((total, cur) => total + cur.value, 0)
        //   for (let i = 0; i < params.length; i++) {
        //     s += '<span style="display:inline-block;margin-right:5px;border-radius:10px;width:9px;height:9px;background-color:' + params[i].color + '"></span>'
        //     s += params[i].seriesName + ' : ' + (100 * parseFloat(params[i].value) / parseFloat(sum)).toFixed(2) + '%<br>'
        //   }
        //   return s
        // }
      }
    },
    _combineRowPaths(data, ...arrays) {
      const len = arrays.length
      let _result = []
      if (len) {
        const rowPaths = arrays.reduce((prev, curr) => {
          const arr = []
          prev.values.forEach(_prevEl => {
            const prevKey = prev.key.split(SEPARATOR)
            curr.values.forEach(_currEl => {
              const currKey = curr.key
              const conditions = {}
              prevKey.forEach((key, i) => {
                conditions[key] = _prevEl.split(SEPARATOR)[i]
              })
              conditions[currKey] = _currEl
              // 判断数据里是否有该项
              const filter = data.some((data) => {
                let status = true
                for (const key in conditions) {
                  if (conditions[key] !== data[key]) {
                    status = false
                    return
                  }
                }
                return status
              })
              if (filter) {
                arr.push(_prevEl + SEPARATOR + _currEl)
              }
            })
          })
          return { key: prev.key + SEPARATOR + curr.key, values: arr }
        }) || {}
        _result = rowPaths.values || []
      }
      return _result
    },
    _combineColPaths(...arrays) {
      return arrays.length ? arrays.reduce((prev, curr) => {
        const arr = []
        prev.forEach(_prevEl => {
          curr.forEach(_currEl => {
            arr.push(_prevEl + SEPARATOR + _currEl)
          })
        })
        return arr
      }) : arrays
    },
    _findCategory(keys = [], data = []) {
      const _result = {}
      data.forEach(item => {
        keys.forEach(key => {
          // Remove duplicates
          _result[key] = _result[key] || []
          _result[key].push(item[key])
          _result[key] = [...new Set(_result[key])]
        })
      })
      return _result
    },
    _reduceValue(data, key) {
      if (!data.length) return 0
      return data.reduce((sum, item) => { return sum + Number(item[key]) }, 0)
    },
    _filterData(conditions, data) {
      return data.filter((data) => {
        let status = true
        for (const key in conditions) {
          if (conditions[key] !== data[key]) {
            status = false
            return
          }
        }
        return status
      })
    },
    // 将数据转换为树形结构
    changeToTree(keyX, keyY, list) {
      // 将key列表的第一级转化为名称组
      const sets = Array.from(new Set(list.map(i => { return i[keyX[0]] })))
      // console.log('keyX:', keyX)// 按深度排列的名称层
      // console.log('keyY:', keyY)// 数值组的名称
      // console.log('list:', list)// 该级别下的所有数据
      // 将当前层级的树结构压入备份
      /** 因为接下来的操作会对这些传入值直接操作，所以需要保存深拷贝后的值以防止备份失效 */
      const backup = [
        JSON.parse(JSON.stringify(list)),
        JSON.parse(JSON.stringify(keyY)),
        JSON.parse(JSON.stringify(keyX))
      ]
      this.drilldownBackup.push(backup)
      // 将当前层级的名称压入标题组/** 同理，先深拷贝 */
      this.titles.push(JSON.parse(JSON.stringify(keyX))[0])
      // console.log('drilldownBackup:', this.drilldownBackup)
      // console.log('sets:', sets)// 名称组
      // 构建空的钻取树
      const tree = sets.map(i => {
        const release = {}
        release.name = i
        release.children = []
        keyY.map(i => { release[i] = 0; return })
        return release
      })
      const nums = {}
      // 向空的钻取树中灌入数据
      list.map(i => {
        const res = tree.find(j => j.name === i[keyX[0]])
        delete i[keyX[0]]
        // 只有子级还存在项时才会往children中添加数据
        if (i[keyX[1]]) res.children.push(i)
        keyY.map(j => { res[j] += parseInt(i[j]); return })
      })
      // console.log('tree:', tree)
      // 创建数值组
      keyY.map(j => {
        nums[j] = {}
        nums[j].data = []
        nums[j].name = j
        nums[j].type = this.chartType
        tree.map(i => {
          nums[j].data.push(i[j] || 0)
        })
      })
      // console.log('nums:', nums)
      // 储存下级结构以便下钻
      keyX.shift()
      this.drilldownKey = keyX
      this.drilldownIndex = keyY
      this.drilldownTarget = tree
      // console.log('this.drilldownTarget:', this.drilldownTarget)
      return [sets, nums]
    },
    // 点击了数据钻取
    clickNode(name) {
      // 获取子级的内容
      // console.log('this.drilldownTarget:', this.drilldownTarget)
      // console.log('name:', name)
      const child = this.drilldownTarget.find(i => i.name === name).children
      if (child.length >= 1) {
        const [xAxisData, seriesObj] = this.changeToTree(this.drilldownKey, this.drilldownIndex, child)
        this.calcData.xAxisData = xAxisData
        this.calcData.seriesObj = seriesObj
        // console.log(this.calcData.legendData)
        // console.log('xAxisData=', xAxisData, 'seriesObj=', seriesObj)
        this.renderChart()
      }
    },
    // 点击了标题以返回上级
    backUp(val) {
      if (this.drilldownBackup.length > 1) {
        // 吐出当前一级的数据
        this.drilldownBackup.pop()
        this.titles.pop()
        this.titles.pop()
        // 收集上一级的数据
        const [list, keyY, keyX] = this.drilldownBackup.pop()
        const [sets, nums] = this.changeToTree(keyX, keyY, list)
        this.calcData.xAxisData = sets
        this.calcData.seriesObj = nums
        this.renderChart()
      }
    },
    // 清空钻取数据以防止堆积
    clear() {
      this.drilldownKey = []
      this.drilldownIndex = []
      this.drilldownTarget = []
      this.drilldownBackup = []
      this.titles = []
    }
  }
}
</script>

<style lang="scss" scoped>

</style>
```
