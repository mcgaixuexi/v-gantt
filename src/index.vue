<template>
  <div class="v-gantt">
    <gantt-tree
      :data="ganttData"
      :bus="bus"
      :scroll-top.sync="scrollTop"
      @delete="onDelete"
      @add="onAdd"
      @edit="onEdit"
      @move="onMove"
      :treeAttrs="treeAttrs"
    >
      <!--@slot 左侧树 header -->
      <slot slot="header" name="tree-header"></slot>
    </gantt-tree>
    <gantt-chart
      :data="ganttData"
      :bus="bus"
      :scroll-top.sync="scrollTop"
      :drag-data="dragData"
      :resize-data="resizeData"
    />
  </div>
</template>
<script lang="ts">
import Vue, { PropType } from 'vue'
import _clonedeep from 'lodash.clonedeep'
import dayjs from './utils/day'
import GanttTree from './components/gantt-tree.vue'
import GanttChart from './components/gantt-chart.vue'
import {
  GanttPropData,
  GanttData,
  GanttPropItem,
  GanttItem,
  GanttPropGroup,
  GanttGroup,
  CollapsedMap,
  GanttPropNode,
  Bus,
  GanttNode,
  GanttPropMilestone,
  GanttMilestone,
  ColUnit,
} from '@/utils/types'
import {
  isGroup,
  search,
  transformGroupToItem,
  transformItemToGroup,
  isMilestone,
} from '@/utils'
import EventEmitter from '@/utils/event-emitter'

function transform(data: GanttPropData): GanttData {
  return data.map((d) => {
    if (isGroup(d)) {
      return transformGroup(d)
    } else if (isMilestone(d)) {
      return transformMilestone(d)
    } else {
      return transformItem(d)
    }
  })
}

function transformItem(d: GanttPropItem): GanttItem {
  return { ...d }
}

function transformGroup(g: GanttPropGroup): GanttGroup {
  return {
    id: g.id,
    name: g.name,
    children: transform(g.children),
    get startDate() {
      return this.children.reduce((result, c) => {
        const startDate = isMilestone(c) ? c.date : c.startDate
        return !result || dayjs(startDate).isBefore(result) ? startDate : result
      }, '')
    },
    get endDate() {
      return this.children.reduce((result, c) => {
        const endDate = isMilestone(c) ? c.date : c.endDate
        return !result || dayjs(endDate).isAfter(result) ? endDate : result
      }, '')
    },
    get progress() {
      let finished = 0
      let total = 0
      this.children.forEach((c) => {
        if (isMilestone(c)) return
        const duration = dayjs.$duration(c.startDate, c.endDate)
        finished += duration * c.progress
        total += duration
      })
      return finished / total
    },
  }
}

function transformMilestone(m: GanttPropMilestone): GanttMilestone {
  return { ...m }
}

function getCollapsedMap(data: GanttPropData): CollapsedMap {
  const map: CollapsedMap = {}
  const loop = (data: GanttPropData) => {
    data.forEach((d) => {
      map[d.id] = false
      if (isGroup(d)) loop(d.children)
    })
  }
  loop(data)
  return map
}
export default Vue.extend({
  name: 'VGantt',
  components: { GanttTree, GanttChart },
  props: {
    /**
     * 行高
     */
    rowH: {
      type: Number,
      default: 30,
    },
    /**
     * 列宽
     */
    colW: {
      type: Number,
      default: 80,
    },
    /**
     * 甘特图数据
     */
    data: {
      type: Array as PropType<GanttPropData>,
      required: true,
    },
    /**
     * 默认视图
     * @values day, week, month
     */
    view: {
      type: String as PropType<ColUnit>,
      default: 'day',
    },
    /**
     * el-tree 属性设置
     * [el-tree文档](http://element-cn.eleme.io/#/zh-CN/component/tree)
     */
    treeAttrs: {
      type: Object,
      default: () => ({}),
    },
  },
  data: () => ({
    collapsedMap: {} as CollapsedMap,
    scrollTop: 0,
    dragData: {
      node: null as null | GanttNode,
      movedCols: 0,
    },
    id: 1,
    resizeData: {
      node: null as null | GanttNode,
      resizedCols: 0,
    },
    /**
     * 组件公交车🚌，存放组件级别的变量
     */
    bus: {
      rowH: 0,
      _colW: 0,
      colUnit: ColUnit.Day,
      get colW(): number {
        const { _colW, colUnit } = this
        switch (colUnit) {
          case ColUnit.Week:
            return _colW * 0.5
          case ColUnit.Month:
            return _colW * 0.2
          case ColUnit.Day:
          default:
            return _colW
        }
      },
      collapsedMap: {} as CollapsedMap,
      ee: new EventEmitter(),
    } as Bus,
  }),
  computed: {
    ganttData(): GanttData {
      return transform(this.data)
    },
  },
  watch: {
    data: {
      handler(v) {
        // 保留折叠状态
        this.collapsedMap = {
          ...getCollapsedMap(v),
          ...this.collapsedMap,
        }
      },
      immediate: true,
    },
  },
  created() {
    this.initBus()
  },
  methods: {
    initBus() {
      Object.assign(this.bus, {
        rowH: this.rowH,
        _colW: this.colW,
        collapsedMap: this.collapsedMap,
        colUnit: this.view,
      })
      this.$watch('rowH', (v) => (this.bus.rowH = v))
      this.$watch('colW', (v) => (this.bus._colW = v))
      this.$watch('collapsedMap', (v) => (this.bus.collapsedMap = v))
      this.$watch('view', (v) => (this.bus.colUnit = v))
      this.$watch('bus.colUnit', (u) => this.$emit('update:view', u))
      const { ee } = this.bus
      ee.on(ee.Event.DragStart, this.onDragStart)
      ee.on(ee.Event.Drag, this.onDrag)
      ee.on(ee.Event.DragEnd, this.onDragEnd)
      ee.on(ee.Event.ResizeStart, this.onResizeStart)
      ee.on(ee.Event.Resize, this.onResize)
      ee.on(ee.Event.ResizeEnd, this.onResizeEnd)
    },
    onAdd({ id, done }: { id: GanttPropNode['id']; done: Function }) {
      const newData = _clonedeep(this.data)
      // 找到该节点和其父节点
      const [node, parent] = search(id, newData) as [
        GanttPropNode,
        GanttPropGroup?,
      ]
      console.log(node, 'node')
      console.log(parent, 'parent')
      // node.children.push()
      // 添加子节点
      this.addNode(node, parent, newData)
      /**
       * 支持通过 data.sync 同步数据
       * @property {GanttPropData} data - 新的 data
       */
      this.$emit('update:data', newData)
      // 如果没有监听事件，直接 done，否则由外部控制何时关闭弹窗
      if (this.$listeners.add) {
        /**
         * 节点添加事件（开发中）
         */
        this.$emit('add', { id })
      } else {
        // done()
      }
    },
    onEdit({
      id,
      name,
      done,
    }: {
      id: GanttPropNode['id']
      name: GanttPropNode['name']
      done: Function
    }) {
      const newData = _clonedeep(this.data)
      // 找到该节点和其父节点
      const [node, parent] = search(id, newData) as [
        GanttPropNode,
        GanttPropGroup?,
      ]

      this.editNode(node, parent, newData, name)
      /**
       * 支持通过 data.sync 同步数据
       * @property {GanttPropData} data - 新的 data
       */
      this.$emit('update:data', newData)
      // 如果没有监听事件，直接 done，否则由外部控制何时关闭弹窗
      if (this.$listeners.edit) {
        /**
         * 节点修改事件（开发中）
         */
        this.$emit('edit', { id })
      } else {
        // done()
      }
      console.log('====================================')
      console.log(id, name)
      console.log('====================================')
    },
    onDelete({ id, done }: { id: GanttPropNode['id']; done: Function }) {
      const newData = _clonedeep(this.data)
      // 找到该节点和其父节点
      const [node, parent] = search(id, newData) as [
        GanttPropNode,
        GanttPropGroup?,
      ]
      // 删除该节点
      this.deleteNode(node, parent, newData)
      /**
       * 支持通过 data.sync 同步数据
       * @property {GanttPropData} data - 新的 data
       */
      this.$emit('update:data', newData)
      // 如果没有监听事件，直接 done，否则由外部控制何时关闭弹窗
      if (this.$listeners.delete) {
        /**
         * 节点删除事件（开发中）
         */
        this.$emit('delete', { id, done })
      } else {
        done()
      }
    },
    addNode(
      node: GanttPropNode,
      parent: GanttPropGroup | undefined,
      root: GanttPropData,
    ) {
      const newChild = {
        id: this.id++,
        name: 'test',
        children: [],
        startDate: dayjs().$format(),
        endDate: dayjs().$format(),
        process: 0,
      }
      // @ts-ignore
      if (!node.children) {
        this.$set(node, 'children', [])
      }
      // @ts-ignore
      node.children.push(newChild)
      console.log(parent, 'parent')
      console.log(node, 'node')
      console.log(root, 'root')
    },
    editNode(
      node: GanttPropNode,
      parent: GanttPropGroup | undefined,
      root: GanttPropData,
      name: string,
    ) {
      if (parent) {
        const i = parent.children.indexOf(node)
        parent.children[i].name = name
        // 判断其父节点是否需要转换成叶节点
        // if (parent.children.length === 0) {
        //   // 获得当前计算出的 GanttNode 数据（包含起始时间）
        //   const [d] = search(parent.id, this.ganttData) as [GanttGroup]
        //   transformGroupToItem(parent, d)
        // }
      } else {
        const i = root.indexOf(node)
        root[i].name = name
      }
    },
    deleteNode(
      node: GanttPropNode,
      parent: GanttPropGroup | undefined,
      root: GanttPropData,
    ) {
      if (parent) {
        const i = parent.children.indexOf(node)
        parent.children.splice(i, 1)
        // 判断其父节点是否需要转换成叶节点
        if (parent.children.length === 0) {
          // 获得当前计算出的 GanttNode 数据（包含起始时间）
          const [d] = search(parent.id, this.ganttData) as [GanttGroup]
          transformGroupToItem(parent, d)
        }
      } else {
        const i = root.indexOf(node)
        root.splice(i, 1)
      }
    },
    /**
     * 在 tree 里发生的拖拽事件。
     * 处于停用状态，启用时要重新梳理逻辑
     */
    onMove({
      id,
      pid,
      index,
      done,
    }: {
      id: GanttPropNode['id']
      pid?: GanttPropNode['id']
      index: number
      done: Function
    }) {
      const newData = _clonedeep(this.data)
      const [node, oldParent] = search(id, newData) as [
        GanttPropNode,
        GanttPropGroup?,
      ]
      // 从旧位置删除节点
      this.deleteNode(node, oldParent, newData)
      // 在新位置放置节点
      let parent: GanttPropNode | undefined
      if (pid) {
        ;[parent] = search(pid, newData) as [GanttPropNode]
        if (isGroup(parent)) {
          parent.children.splice(index, 0, node)
        } else if (isMilestone(parent)) {
          // impossible
          throw new Error(`里程碑「${parent.name}」不能做群组节点！`)
        } else {
          transformItemToGroup(parent)
          ;((parent as unknown) as GanttPropGroup).children.push(node)
        }
      } else {
        newData.splice(index, 0, node)
      }

      this.$emit('update:data', newData)
      // 如果没有监听事件，直接 done，否则由外部控制何时关闭弹窗
      if (this.$listeners.move) {
        /**
         * 在树中拖拽节点事件（开发中）
         */
        this.$emit('move', { node, parent, done })
      } else {
        done()
      }
    },
    onDragStart({ id }: { id: GanttItem['id'] }) {
      this.dragData.node = search(id, this.ganttData)[0] as GanttNode
    },
    onDrag({ movedCols }: { movedCols: number }) {
      this.dragData.movedCols = movedCols
    },
    onDragEnd() {
      const { node, movedCols } = this.dragData
      this.dragData.node = null
      this.dragData.movedCols = 0
      if (!node || !movedCols) return
      const newData = _clonedeep(this.data)
      const [root] = search(node.id, newData) as [GanttPropNode]
      function loop(n: GanttPropNode) {
        if (isGroup(n)) {
          n.children.forEach(loop)
        } else if (isMilestone(n)) {
          n.date = dayjs.$add(n.date, movedCols)
        } else {
          n.startDate = dayjs.$add(n.startDate, movedCols)
          n.endDate = dayjs.$add(n.endDate, movedCols)
        }
      }
      loop(root)

      this.$emit('update:data', newData)
      /**
       * 甘特图节点左右横移事件
       * @property {GanttPropNode} node - 被移动的节点
       * @property {number} movedDays - 移动的天数
       */
      this.$emit('dragged', { node: root, movedDays: movedCols })
    },
    onResizeStart({ id }: { id: GanttItem['id'] }) {
      this.resizeData.node = search(id, this.ganttData)[0] as GanttNode
    },
    onResize({ resizedCols }: { resizedCols: number }) {
      this.resizeData.resizedCols = resizedCols
    },
    onResizeEnd() {
      const { node, resizedCols } = this.resizeData
      this.resizeData.node = null
      this.resizeData.resizedCols = 0
      if (!node || !resizedCols) return
      const newData = _clonedeep(this.data)
      const [root] = search(node.id, newData) as [GanttPropItem]
      root.endDate = dayjs.$add(root.endDate, resizedCols)

      this.$emit('update:data', newData)
      /**
       * 甘特图节点拉伸事件
       * @property {GanttPropNode} node - 被拉伸的节点
       * @property {number} resizedDays - 拉伸的天数
       */
      this.$emit('resized', { node: root, resizedDays: resizedCols })
    },
  },
})
</script>
<style lang="less" scoped>
.v-gantt {
  min-height: 300px;
  display: flex;

  .gantt-tree {
    flex: 0 0 auto;
    width: 250px;
    height: 100%;
  }

  .gantt-chart {
    flex: 1 0 auto;
    height: 100%;
  }
}
</style>
