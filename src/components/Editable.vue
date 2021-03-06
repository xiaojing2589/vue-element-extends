<template>
  <el-table
    ref="refElTable"
    v-bind="attrs"
    v-on="events">
    <slot></slot>
  </el-table>
</template>

<script>
import { mapGetters } from 'vuex'
import { Table } from 'element-ui'
import XEUtils from 'xe-utils'

export default {
  name: 'ElEditable',
  props: {
    editConfig: Object,
    editRules: Object,

    /**
     * 还原 ElTable 所有属性
     */
    data: Array,
    height: [String, Number],
    maxHeight: [String, Number],
    stripe: Boolean,
    border: Boolean,
    size: { type: String, default: 'small' },
    fit: { type: Boolean, default: true },
    showHeader: { type: Boolean, default: true },
    highlightCurrentRow: Boolean,
    currentRowKey: [String, Number],
    rowClassName: [Function, String],
    rowStyle: [Function, Object],
    cellClassName: [Function, String],
    cellStyle: [Function, Object],
    headerRowClassName: [Function, String],
    headerRowStyle: [Function, Object],
    headerCellClassName: [Function, String],
    headerCellStyle: [Function, Object],
    rowKey: [Function, String],
    emptyText: String,
    defaultExpandAll: Boolean,
    expandRowKeys: Array,
    defaultSort: Object,
    tooltipEffect: { type: String, default: 'dark' },
    showSummary: Boolean,
    sumText: String,
    summaryMethod: Function,
    selectOnIndeterminate: { type: Boolean, default: true },
    spanMethod: Function
  },
  components: {
    ElTable: Table
  },
  provide () {
    return {
      $editable: this
    }
  },
  data () {
    return {
      editProto: {},
      datas: [],
      initialStore: [],
      deleteRecords: [],
      lastActive: null,
      isEmitUpdateActivate: false,
      isClearlActivate: false,
      isManualActivate: false,
      isValidActivate: false
    }
  },
  computed: {
    ...mapGetters([
      'globalClick'
    ]),
    attrs () {
      return {
        class: ['editable', `editable_${this.configs.trigger}`, { 'editable_icon': this.configs.showIcon }],
        data: this.datas,
        height: this.height,
        maxHeight: this.maxHeight,
        stripe: this.stripe,
        border: this.border,
        size: this.size,
        fit: this.fit,
        showHeader: this.showHeader,
        highlightCurrentRow: this.highlightCurrentRow,
        currentRowKey: this.currentRowKey,
        rowClassName: this._rowClassName,
        rowStyle: XEUtils.isFunction(this.rowStyle) ? this._rowStyle : this.rowStyle,
        cellClassName: this._cellClassName,
        cellStyle: XEUtils.isFunction(this.cellStyle) ? this._cellStyle : this.cellStyle,
        headerRowClassName: XEUtils.isFunction(this.headerRowClassName) ? this._headerRowClassName : this.headerRowClassName,
        headerRowStyle: XEUtils.isFunction(this.headerRowStyle) ? this._headerRowStyle : this.headerRowStyle,
        headerCellClassName: XEUtils.isFunction(this.headerCellClassName) ? this._headerCellClassName : this.headerCellClassName,
        headerCellStyle: XEUtils.isFunction(this.headerCellStyle) ? this._headerCellStyle : this.headerCellStyle,
        rowKey: XEUtils.isFunction(this.rowKey) ? this._rowKey : this.rowKey,
        emptyText: this.emptyText,
        defaultExpandAll: this.defaultExpandAll,
        expandRowKeys: this.expandRowKeys,
        defaultSort: this.defaultSort,
        tooltipEffect: this.tooltipEffect,
        showSummary: this.showSummary,
        sumText: this.sumText,
        summaryMethod: this._summaryMethod,
        selectOnIndeterminate: this.selectOnIndeterminate,
        spanMethod: this._spanMethod
      }
    },
    events () {
      return {
        'select': this._select,
        'select-all': this._selectAll,
        'selection-change': this._selectionChange,
        'cell-mouse-enter': this._cellMouseEnter,
        'cell-mouse-leave': this._cellMouseLeave,
        'cell-click': this._cellClick,
        'cell-dblclick': this._cellDBLclick,
        'row-click': this._rowClick,
        'row-contextmenu': this._rowContextmenu,
        'row-dblclick': this._rowDBLclick,
        'header-click': this._headerClick,
        'header-contextmenu': this._headerContextmenu,
        'sort-change': this._sortChange,
        'filter-change': this._filterChange,
        'current-change': this._currentChange,
        'header-dragend': this._headerDragend,
        'expand-change': this._expandChange
      }
    },
    tableData () {
      return this.$refs.refElTable ? this.$refs.refElTable.tableData : this.datas
    },
    configs () {
      let tipConf = this.editConfig ? (this.editConfig.validTooltip || {}) : {}
      return Object.assign({
        trigger: 'click',
        showIcon: true,
        showStatus: true,
        mode: 'cell',
        useDefaultValidTip: false,
        autoClearActive: true
      }, this.editConfig, {
        validTooltip: Object.assign({
          disabled: false,
          offset: 10,
          placement: 'bottom-start'
        }, tipConf, {
          manual: true,
          popperClass: ['editable-valid_tooltip'].concat(tipConf.popperClass ? tipConf.popperClass.split(' ') : []).join(' ')
        })
      })
    }
  },
  watch: {
    /**
     * 监听全局 click 事件
     * 用于处理点击表格外清除激活状态
     */
    globalClick (evnt) {
      if (!this.isManualActivate && !this.isValidActivate && this.lastActive) {
        if (this.configs.autoClearActive) {
          let target = evnt.target
          let clearActiveMethod = this.configs.clearActiveMethod
          let { row, column, cell } = this.lastActive
          let isClearActive = true
          while (target && target.nodeType && target !== document) {
            let trElem = cell.parentNode
            if (this.configs.mode === 'row' ? target === trElem : target === cell) {
              return
            }
            if (this.configs.mode === 'row' && this._hasClass(target, 'editable-row') && target.parentNode === trElem.parentNode) {
              break
            }
            target = target.parentNode
          }
          if (clearActiveMethod) {
            let param = { row: row.data, rowIndex: XEUtils.findIndexOf(this.tableData, item => item === row) }
            if (this.configs.mode === 'cell') {
              Object.assign(param, { column, columnIndex: this.$refs.refElTable ? XEUtils.findIndexOf(this.$refs.refElTable.columns, item => item === column) : null })
            }
            isClearActive = clearActiveMethod(param)
          }
          if (isClearActive) {
            this._validActiveCell().then(() => {
              this._clearValidError(row)
              this._clearActiveData()
              this._restoreTooltip()
              if (this.configs.mode === 'row') {
                this.$emit('clear-active', row.data, evnt)
              } else {
                this.$emit('clear-active', row.data, column, cell, evnt)
              }
            }).catch(e => e)
          }
        }
      } else {
        this.isValidActivate = false
        this.isManualActivate = false
      }
    },
    data (value) {
      if (!this.isEmitUpdateActivate) {
        if (value.length) {
          this.reload(value)
        } else {
          this.clear()
        }
      } else {
        this.isEmitUpdateActivate = false
      }
    }
  },
  created () {
    this._initial(this.data, true)
  },
  methods: {
    /**************************/
    /* Original methods statrt */
    /**************************/
    clearSelection () {
      this.$nextTick(() => this.$refs.refElTable.clearSelection())
    },
    toggleRowSelection (record, selected) {
      this.$nextTick(() => this.$refs.refElTable.toggleRowSelection(this.datas.find(item => item.data === record), selected))
    },
    toggleAllSelection () {
      this.$nextTick(() => this.$refs.refElTable.toggleAllSelection())
    },
    toggleRowExpansion (record, expanded) {
      this.$nextTick(() => this.$refs.refElTable.toggleRowExpansion(this.datas.find(item => item.data === record), expanded))
    },
    setCurrentRow (record) {
      this.$nextTick(() => this.$refs.refElTable.setCurrentRow(this.datas.find(item => item.data === record)))
    },
    clearSort () {
      this.$nextTick(() => this.$refs.refElTable.clearSort())
    },
    clearFilter () {
      this.$nextTick(() => this.$refs.refElTable.clearFilter())
    },
    doLayout () {
      this.$nextTick(() => this.$refs.refElTable.doLayout())
    },
    insert (newRecord) {
      return this.insertAt(newRecord, 0)
    },
    /**************************/
    /* Original methods end */
    /**************************/

    /***************************/
    /* Interior methods statrt */
    /***************************/
    _initial (datas, isReload) {
      if (isReload) {
        this.initialStore = XEUtils.clone(datas, true)
      }
      this.datas = (datas || []).map(record => this._toData(this.datas.some(row => row.data === record) ? record : Object.assign(record, this._defineProp(record))))
    },
    _getData (datas) {
      return (datas || this.datas).map(item => item.data)
    },
    _defineProp (record) {
      let recordItem = Object.assign({}, record)
      let columns = this.$refs.refElTable ? this.$refs.refElTable.columns : []
      columns.forEach(column => {
        if (column.property && !XEUtils.has(recordItem, column.property)) {
          XEUtils.set(recordItem, column.property, null)
        }
      })
      return recordItem
    },
    _toData (item, status) {
      return item && item._EDITABLE_PROTO === this.editProto ? item : {
        _EDITABLE_PROTO: this.editProto,
        data: item,
        store: XEUtils.clone(item, true),
        validActive: null,
        validRule: null,
        showValidMsg: false,
        checked: false,
        editActive: null,
        editStatus: status || 'initial',
        config: {
          size: this.size,
          showIcon: this.configs.showIcon,
          showStatus: this.configs.showStatus,
          mode: this.configs.mode,
          useDefaultValidTip: this.configs.useDefaultValidTip,
          validTooltip: this.configs.validTooltip,
          rules: this.editRules
        }
      }
    },
    _updateData () {
      this.isEmitUpdateActivate = true
      this.$emit('update:data', this.datas.map(item => item.data))
    },
    _rowClassName ({ row, rowIndex }) {
      let clsName = 'editable-row '
      let rowClassName = this.rowClassName
      if (this.configs.mode === 'row' && this._isDisabledEdit(row)) {
        clsName += 'editable-row_disabled '
      }
      if (XEUtils.isFunction(rowClassName)) {
        clsName += rowClassName({ row: row.data, rowIndex }) || ''
      } else if (XEUtils.isString(rowClassName)) {
        clsName += `${rowClassName}`
      }
      return XEUtils.trimRight(clsName)
    },
    _rowStyle ({ row, rowIndex }) {
      return this.rowStyle({ row: row.data, rowIndex })
    },
    _cellClassName ({ row, column, rowIndex, columnIndex }) {
      let clsName = ''
      let cellClassName = this.cellClassName
      if (this.configs.mode === 'cell' && row.editActive && row.editActive === column.property) {
        clsName += 'editable-col_active '
      }
      if (this.configs.showStatus && !XEUtils.isEqual(XEUtils.get(row.data, column.property), XEUtils.get(row.store, column.property))) {
        clsName += 'editable-col_dirty '
      }
      if (row.checked && row.checked === column.property) {
        clsName = 'editable-col_checked '
      }
      if (row.validActive && row.validActive === column.property) {
        clsName += 'valid-error '
      }
      if (this.configs.mode === 'cell' && this._isDisabledEdit(row, column, columnIndex)) {
        clsName += 'editable-col_disabled '
      }
      if (XEUtils.isFunction(cellClassName)) {
        clsName += cellClassName({ row: row.data, column, rowIndex, columnIndex }) || ''
      } else if (XEUtils.isString(cellClassName)) {
        clsName += `${cellClassName}`
      }
      return XEUtils.trimRight(clsName)
    },
    _cellStyle ({ row, column, rowIndex, columnIndex }) {
      return this.cellStyle({ row: row.data, column, rowIndex, columnIndex })
    },
    _headerRowClassName ({ row, rowIndex }) {
      return this.headerRowClassName({ row: row.data, rowIndex })
    },
    _headerRowStyle ({ row, rowIndex }) {
      return this.headerRowStyle({ row: row.data, rowIndex })
    },
    _headerCellClassName ({ row, column, rowIndex, columnIndex }) {
      return this.headerCellClassName({ row: row.data, column, rowIndex, columnIndex })
    },
    _headerCellStyle ({ row, column, rowIndex, columnIndex }) {
      return this.headerCellStyle({ row: row.data, column, rowIndex, columnIndex })
    },
    _rowKey (row) {
      return this.rowKey(row.data)
    },
    _select (selection, row) {
      this.$emit('select', selection.map(item => item ? item.data : item), row.data)
    },
    _selectAll (selection) {
      this.$emit('select-all', selection.map(item => item ? item.data : item))
    },
    _selectionChange (selection) {
      this.$emit('selection-change', selection.map(item => item ? item.data : item))
    },
    _cellMouseEnter (row, column, cell, event) {
      this.$emit('cell-mouse-enter', row.data, column, cell, event)
    },
    _cellMouseLeave (row, column, cell, event) {
      this.$emit('cell-mouse-leave', row.data, column, cell, event)
    },
    _cellClick (row, column, cell, event) {
      this._cellHandleEvent('click', row, column, cell, event)
    },
    _cellDBLclick (row, column, cell, event) {
      this._cellHandleEvent('dblclick', row, column, cell, event)
    },
    /**
     * 触发编辑事件
     * 行模式和列模式可编辑，如果是只读列不能编辑
     * 触发时先校验活动行或列，如果存在校验规则且校验不通过
     * 停止激活新行，聚焦到校验错误行
     */
    _cellHandleEvent (type, row, column, cell, event) {
      if (!this.isClearlActivate && this._hasClass(cell, 'editable-col_edit')) {
        this._validActiveCell().then(() => {
          if (this.lastActive) {
            this._clearValidError(this.lastActive.row)
          }
          if (this.configs.trigger === type) {
            this._triggerActive(row, column, cell, event)
            if (row && this.configs.mode === 'row') {
              this._validRowRules('change', row).catch(({ rule, row, column, cell }) => this._toValidError(rule, row, column, cell))
            } else {
              this._validColRules('change', row, column).catch(rule => this._toValidError(rule, row, column, cell))
            }
          } else {
            if (row.editActive !== column.property) {
              this._clearActiveData()
              row.checked = column.property
            }
          }
        }).catch(e => e).then(() => this.$emit(`cell-${type}`, row.data, column, cell, event))
      } else {
        this.isClearlActivate = false
        this.$emit(`cell-${type}`, row.data, column, cell, event)
      }
    },
    _rowClick (row, event, column) {
      this.$emit('row-click', row.data, event, column)
    },
    _rowContextmenu (row, event) {
      this.$emit('row-contextmenu', row.data, event)
    },
    _rowDBLclick (row, event) {
      this.$emit('row-dblclick', row.data, event)
    },
    _headerClick (column, event) {
      this.$emit('header-click', column, event)
    },
    _headerContextmenu (column, event) {
      this.$emit('header-contextmenu', column, event)
    },
    _sortChange ({ column, prop, order }) {
      this.$emit('sort-change', { column, prop, order })
    },
    _filterChange (filters) {
      this.$emit('filter-change', filters)
    },
    _currentChange (currentRow, oldCurrentRow) {
      if (currentRow && oldCurrentRow) {
        this.$emit('current-change', currentRow.data, oldCurrentRow.data)
      } else if (currentRow) {
        this.$emit('current-change', currentRow.data, null)
      } else if (oldCurrentRow) {
        this.$emit('current-change', null, oldCurrentRow.data)
      }
    },
    _headerDragend (newWidth, oldWidth, column, event) {
      this.$emit('header-dragend', newWidth, oldWidth, column, event)
    },
    _expandChange (row, expandedRows) {
      this.$emit('expand-change', row.data, expandedRows)
    },
    _clearActiveData () {
      this.lastActive = null
      this.datas.forEach(item => {
        item.editActive = null
        item.showValidMsg = false
        item.checked = null
      })
    },
    _restoreTooltip (cell) {
      Array.from(this.$el.querySelectorAll('.disabled-el-tooltip')).forEach(elem => {
        this._removeClass(elem, ['disabled-el-tooltip'])
        this._addClass(elem, ['el-tooltip'])
      })
    },
    /**
     * 阻止带有 tooltip 组件的列
     * 如果行或列被激活编辑时，关闭 tooltip 提示并禁用
     */
    _disabledTooltip (cell) {
      let tElems = ['row', 'manual'].includes(this.configs.mode) ? cell.parentNode.querySelectorAll('td.editable-col_edit>.cell.el-tooltip') : cell.querySelectorAll('.cell.el-tooltip')
      if (this.$refs.refElTable) {
        let refElTableBody = this.$refs.refElTable.$children.find(comp => this._hasClass(comp.$el, 'el-table__body'))
        if (refElTableBody && refElTableBody.$refs.tooltip) {
          refElTableBody.$refs.tooltip.hide()
        }
      }
      Array.from(tElems).forEach(elem => {
        this._removeClass(elem, ['el-tooltip'])
        this._addClass(elem, ['disabled-el-tooltip'])
      })
    },
    _addClass (cell, clss) {
      let classList = cell.className.split(' ')
      clss.forEach(name => {
        if (classList.indexOf(name) === -1) {
          classList.push(name)
        }
      })
      cell.className = classList.join(' ')
    },
    _hasClass (cell, cls) {
      return cell.className.split(' ').includes(cls)
    },
    _removeClass (cell, clss) {
      let classList = []
      cell.className.split(' ').forEach(name => {
        if (clss.indexOf(name) === -1) {
          classList.push(name)
        }
      })
      cell.className = classList.join(' ')
    },
    /**
     * 设置单元格聚焦
     * 默认对文本款类的激活后自动聚焦
     * 如果是自定义渲染，也可以指定 class=editable-custom_input 使该单元格自动聚焦
     * 允许通过单元格渲染配置指定 autofocus 来打开或关闭自动聚焦
     */
    _setCellFocus (cell) {
      let inpElem = cell.querySelector('.el-input>input')
      if (!inpElem) {
        inpElem = cell.querySelector('.el-textarea>textarea')
        if (!inpElem) {
          inpElem = cell.querySelector('.editable-custom_input')
        }
      }
      if (inpElem && this._hasClass(cell, 'editable-col_autofocus')) {
        inpElem.focus()
      }
    },
    _isRowDataChange (row, column) {
      let columns = this.$refs.refElTable.columns
      if (column) {
        return !XEUtils.isEqual(XEUtils.get(row.data, column.property), XEUtils.get(row.store, column.property))
      }
      return !columns.every(column => XEUtils.isEqual(XEUtils.get(row.data, column.property), XEUtils.get(row.store, column.property)))
    },
    _isDisabledEdit (row, column, columnIndex) {
      let param = { row: row.data, rowIndex: XEUtils.findIndexOf(this.tableData, item => item === row) }
      if (this.configs.mode === 'cell') {
        Object.assign(param, { column, columnIndex })
      }
      return this.configs.activeMethod ? !this.configs.activeMethod(param) : false
    },
    _triggerActive (row, column, cell, event) {
      if (!this._isDisabledEdit(row, column)) {
        this._restoreTooltip(cell)
        this._disabledTooltip(cell)
        this._clearActiveData()
        this.lastActive = { row, column, cell }
        row.editActive = column.property
        this.$nextTick(() => {
          this._setCellFocus(cell)
          if (row.editActive !== column.property) {
            this.$emit('edit-active', row.data, column, cell, event)
          }
        })
      }
    },
    _summaryMethod (param) {
      let { columns } = param
      let data = param.data.map(item => item.data)
      let sums = []
      if (this.summaryMethod) {
        sums = this.summaryMethod({ columns, data })
      } else {
        columns.forEach((column, index) => {
          if (index === 0) {
            sums[index] = this.sumText || (this.$t ? this.$t('el.table.sumText') : '合计')
            return
          }
          sums[index] = data.some(item => isNaN(Number(item[column.property]))) ? '' : XEUtils.sum(data, column.property)
        })
      }
      return sums
    },
    _spanMethod ({ row, column, rowIndex, columnIndex }) {
      let rowspan = 1
      let colspan = 1
      let spanMethod = this.spanMethod
      if (XEUtils.isFunction(spanMethod)) {
        var result = spanMethod({ row: row.data, column, rowIndex, columnIndex })
        if (XEUtils.isArray(result)) {
          rowspan = result[0]
          colspan = result[1]
        } else if (XEUtils.isPlainObject(result)) {
          rowspan = result.rowspan
          colspan = result.colspan
        }
      }
      return { rowspan, colspan }
    },
    _validRowRules (type, row) {
      let validPromise = Promise.resolve()
      if (!XEUtils.isEmpty(this.editRules)) {
        let editRules = this.editRules
        let datas = this.tableData
        let columns = this.$refs.refElTable.columns
        let ruleKeys = Object.keys(editRules)
        let trElems = this.$el.querySelectorAll('.editable-row')
        let index = XEUtils.findIndexOf(datas, item => item === row)
        this._clearValidError(row)
        columns.forEach((column, cIndex) => {
          if (ruleKeys.includes(column.property)) {
            validPromise = validPromise.then(rest => new Promise((resolve, reject) => {
              this._validColRules('all', row, column).then(resolve).catch(rule => {
                let rest = { rule, row, column, cell: trElems[index].children[cIndex] }
                return reject(rest)
              })
            }))
          }
        })
      }
      return validPromise
    },
    /**
     * 校验数据
     * 按表格行、列顺序依次校验（同步或异步）
     * 校验规则根据索引顺序依次校验，如果是异步则会等待校验完成才会继续校验下一列
     * 如果校验失败则，触发回调或者Promise，结果返回一个 Boolean 值
     * 如果是传回调方式这返回一个 Boolean 值和校验不通过列的错误消息
     *
     * 参数：required=Boolean 是否必填，min=Number 最小长度，max=Number 最大长度，validator=Function(rule, value, callback) 自定义校验，trigger=blur|change 触发方式
     */
    _validColRules (type, row, column) {
      let property = column.property
      let validPromise = Promise.resolve()
      if (property && !XEUtils.isEmpty(this.editRules)) {
        let rules = this.editRules[property]
        let value = row.data[property]
        if (rules) {
          for (let rIndex = 0; rIndex < rules.length; rIndex++) {
            validPromise = validPromise.then(rest => new Promise((resolve, reject) => {
              let rule = rules[rIndex]
              let isRequired = rule.required === true
              if ((type === 'all' || !rule.trigger || rule.trigger === 'change' || type === rule.trigger) && (isRequired || value)) {
                if (XEUtils.isFunction(rule.validator)) {
                  rule.validator(rule, value, e => {
                    if (XEUtils.isError(e)) {
                      let cusRule = { type: 'custom', message: e.message, rule }
                      return reject(cusRule)
                    }
                    return resolve(rule)
                  })
                } else {
                  let restVal
                  let isNumber = rule.type === 'number'
                  if (isNumber) {
                    restVal = XEUtils.toNumber(value)
                  } else {
                    restVal = '' + (value || '')
                  }
                  if (isRequired && !value) {
                    reject(rule)
                  } else if (value &&
                    ((isNumber && isNaN(value)) ||
                    (XEUtils.isRegExp(rule.pattern) && !rule.pattern.test(value)) ||
                    (XEUtils.isNumber(rule.min) && (isNumber ? restVal < rule.min : restVal.length < rule.min)) ||
                    (XEUtils.isNumber(rule.max) && (isNumber ? restVal > rule.max : restVal.length > rule.max)))
                  ) {
                    reject(rule)
                  } else {
                    resolve(rule)
                  }
                }
              } else {
                resolve(rule)
              }
            }))
          }
        }
      }
      return validPromise
    },
    _validActiveCell () {
      if (this.lastActive && !XEUtils.isEmpty(this.editRules)) {
        let { row, column, cell } = this.lastActive
        if (row && this.configs.mode === 'row') {
          return this._validRowRules('blur', row).catch(({ rule, row, column, cell }) => {
            let rest = { rule, row, column, cell }
            this._toValidError(rule, row, column, cell)
            return Promise.reject(rest)
          })
        } else {
          return this._validColRules('blur', row, column).catch(rule => {
            let rest = { rule, row, column, cell }
            this._toValidError(rule, row, column, cell)
            return Promise.reject(rest)
          })
        }
      }
      return Promise.resolve()
    },
    _clearValidError (row) {
      row.showValidMsg = false
      row.validRule = null
      row.validActive = null
    },
    _toValidError (rule, row, column, cell) {
      row.validRule = rule
      row.validActive = column.property
      this._triggerActive(row, column, cell, { type: 'valid' })
      this.$nextTick(() => {
        if (this.configs.useDefaultValidTip && this.isValidActivate && cell) {
          if (cell.scrollIntoViewIfNeeded) {
            cell.scrollIntoViewIfNeeded()
          } else if (cell.scrollIntoView) {
            cell.scrollIntoView()
          }
        }
        // 解决 ElTooltip 无法自动弹出问题
        setTimeout(() => {
          row.showValidMsg = true
        }, 50)
      })
      this.$emit('valid-error', rule, row, column)
    },
    _deleteData (index) {
      if (index > -1) {
        let items = this.datas.splice(index, 1)
        items.forEach(item => {
          if (item.editStatus === 'initial') {
            this.deleteRecords.push(item)
          }
        })
        return items
      }
      return []
    },
    _clearAllOpers () {
      this.clearSelection()
      this.clearFilter()
      this.clearSort()
    },
    /***************************/
    /* Interior methods end    */
    /***************************/

    /***************************/
    /* Public methods start    */
    /***************************/
    reload (datas) {
      this.deleteRecords = []
      this._clearAllOpers()
      this._clearActiveData()
      this._initial(datas, true)
      this._updateData()
    },
    reloadRow (record) {
      let row = this.datas.find(item => item.data === record)
      if (row) {
        XEUtils.destructuring(row.data, record)
        Object.assign(row, { store: XEUtils.clone(row.data, true) })
      }
    },
    /**
     * 还原更改，以最后一次 reload 或 reloadRow 的数据为源数据或者初始值 data
     * 还原指定行数据
     * 还原整个表格数据
     */
    revert (record) {
      if (record) {
        let { data, store } = this.datas.find(item => item.data === record)
        XEUtils.destructuring(data, XEUtils.clone(store, true))
      } else {
        this._clearAllOpers()
        this.reload(this.initialStore)
      }
    },
    /**
     * 清空
     */
    clear () {
      this.deleteRecords = []
      this._clearActiveData()
      this._initial([])
      this._updateData()
    },
    /**
     * 插入数据
     * 如果是 record 或 rowIndex 则在指定位置新增一行新数据
     * 如果是 null 或 0 则插入到第一行
     * 如果是 -1 或 大于行索引 则使用 push 到表格最后
     */
    insertAt (newRecord, recordOrIndex) {
      let len = this.datas.length
      let recordItem = this._toData(this._defineProp(newRecord), 'insert')
      if (recordOrIndex) {
        if (!XEUtils.isNumber(recordOrIndex)) {
          recordOrIndex = XEUtils.findIndexOf(this.datas, item => item.data === recordOrIndex)
        }
        if (recordOrIndex === -1 || recordOrIndex >= len) {
          this.datas.push(recordItem)
        } else {
          this.datas.splice(recordOrIndex, 0, recordItem)
        }
      } else {
        this.datas.unshift(recordItem)
      }
      this._updateData()
      return recordItem.data
    },
    /**
     * 根据索引删除行数据
     */
    removeByIndex (rowIndex) {
      let row = this.tableData[rowIndex]
      if (row) {
        return this.remove(row.data)
      }
      return null
    },
    removeByIndexs (rowIndexs) {
      return this.removes(rowIndexs.map(index => this.tableData[index] ? this.tableData[index].data : null))
    },
    remove (record) {
      let items = this._deleteData(XEUtils.findIndexOf(this.datas, item => item.data === record))
      this._updateData()
      return items.length ? items[0].data : null
    },
    removes (records) {
      let items = []
      XEUtils.lastEach(this.datas, (item, index) => {
        if (records.includes(item.data)) {
          items = items.concat(this._deleteData(index))
        }
      })
      this._updateData()
      return items.map(item => item.data)
    },
    getSelecteds () {
      return this.$refs.refElTable.selection.map(item => item.data)
    },
    removeSelecteds () {
      this.removes(this.getSelecteds())
    },
    getRecords (rowIndex) {
      let list = this._getData()
      return arguments.length ? list[rowIndex] : list
    },
    getAllRecords () {
      return {
        records: this._getData(),
        selecteds: this.getSelecteds(),
        insertRecords: this.getInsertRecords(),
        removeRecords: this.getRemoveRecords(),
        updateRecords: this.getUpdateRecords()
      }
    },
    getInsertRecords () {
      return this._getData(this.datas.filter(item => item.editStatus === 'insert'))
    },
    getRemoveRecords () {
      return this._getData(this.deleteRecords)
    },
    getUpdateRecords () {
      return this._getData(this.datas.filter(item => item.editStatus === 'initial' && !XEUtils.isEqual(item.data, item.store)))
    },
    clearActive () {
      this.isClearlActivate = true
      this._clearActiveData()
      this._restoreTooltip()
    },
    /**
     * 指定某一行为激活状态
     * 只有当指定为 mode='row' 行编辑模式时
     * 才可以根据索引激活行为编辑状态
     * 如果 preventDefault=false 不阻止 clearActive 行为
     */
    setActiveRow (record, preventDefault) {
      let rowIndex = XEUtils.findIndexOf(this.tableData, item => item.data === record)
      let row = this.tableData[rowIndex]
      if (rowIndex > -1 && this.configs.mode === 'row') {
        this.isManualActivate = preventDefault !== false
        this.datas.forEach(row => {
          if (row.data !== record) {
            this._clearValidError(row)
          }
        })
        this._validRowRules('all', row).then(valid => {
          let columns = this.$refs.refElTable.columns
          let colIndex = XEUtils.findIndexOf(columns, item => item.property)
          let column = columns[colIndex]
          let trElemList = this.$el.querySelectorAll('.el-table__body-wrapper .editable-row')
          let cell = trElemList[rowIndex].children[colIndex]
          this._triggerActive(row, column, cell, { type: 'edit' })
        }).catch(({ rule, row, column, cell }) => {
          this._toValidError(rule, row, column, cell)
        })
        return true
      }
      return false
    },
    isActiveRow (record) {
      return this.lastActive ? this.lastActive.row.data === record : false
    },
    getActiveInfo () {
      if (this.lastActive) {
        let { row, column } = this.lastActive
        let index = XEUtils.findIndexOf(this.datas, item => item === row)
        if (this.configs.mode === 'row') {
          return { row: row.data, $index: index, _row: row, isUpdate: this._isRowDataChange(row) }
        }
        return { row: row.data, column, $index: index, _row: row, isUpdate: this._isRowDataChange(row, column) }
      }
      return null
    },
    isRowChange (record, property) {
      let row = this.datas.find(item => item.data === record)
      return property ? this._isRowDataChange(row, { property }) : this._isRowDataChange(row)
    },
    /**
     * 更新列状态
     * 如果组件值 v-model 发生 change 时，调用改函数用于更新某一列编辑状态
     * 由于缓存策略，但行数据发生增加或删除时，需要更新所有行
     */
    updateStatus (scope) {
      this.$nextTick(() => {
        let { $index, _row, column, store } = scope
        let trElems = store.table.$el.querySelectorAll('.editable-row')
        if (trElems.length) {
          let trElem = trElems[$index]
          let cell = trElem.querySelector(`.${column.id}`)
          if (cell) {
            return this._validColRules('change', _row, column).then(rule => {
              if (this.configs.mode === 'row' ? _row.validActive && _row.validActive === column.property : true) {
                this._clearValidError(_row)
              }
            }).catch(rule => {
              this._toValidError(rule, _row, column, cell)
            })
          }
        }
      })
    },
    checkValid () {
      let row = this.datas.find(item => item.validActive)
      if (row) {
        let index = XEUtils.findIndexOf(this.datas, item => item === row)
        return { error: true, row: row.data, prop: row.validActive, rule: row.validRule, $index: index, _row: row }
      }
      return { error: false }
    },
    /**
     * 对表格某一行进行校验的方法
     * 返回 Promise 对象，或者使用回调方式
     */
    validateRow (record, cb) {
      let rowIndex = XEUtils.findIndexOf(this.tableData, item => item.data === record)
      this.isValidActivate = true
      return new Promise((resolve, reject) => {
        let row = this.tableData[rowIndex]
        this._validRowRules('all', row).then(rest => {
          let valid = true
          if (cb) {
            cb(valid)
          }
          resolve(true)
        }).catch(({ rule, row, column, cell }) => {
          let valid = false
          this._toValidError(rule, row, column, cell)
          if (cb) {
            cb(valid, { [column.property]: [new Error(rule.message)] })
            resolve(valid)
          } else {
            reject(valid)
          }
        })
      })
    },
    /**
     * 对整个表格数据进行校验
     * 返回 Promise 对象，或者使用回调方式
     */
    validate (cb) {
      let validPromise = Promise.resolve(true)
      this.isValidActivate = true
      if (!XEUtils.isEmpty(this.editRules)) {
        let editRules = this.editRules
        let datas = this.tableData
        let columns = this.$refs.refElTable.columns
        let ruleKeys = Object.keys(editRules)
        let trElems = this.$el.querySelectorAll('.editable-row')
        datas.forEach((row, index) => {
          this._clearValidError(row)
          columns.forEach((column, cIndex) => {
            if (ruleKeys.includes(column.property)) {
              validPromise = validPromise.then(rest => new Promise((resolve, reject) => {
                this._validColRules('all', row, column).then(resolve).catch(rule => {
                  let rest = { rule, row, column, cell: trElems[index].children[cIndex] }
                  return reject(rest)
                })
              }))
            }
          })
        })
        return validPromise.then(() => {
          let valid = true
          if (cb) {
            cb(valid)
          }
          return true
        }).catch(({ rule, row, column, cell }) => {
          let valid = false
          this._toValidError(rule, row, column, cell)
          if (cb) {
            cb(valid, { [column.property]: [new Error(rule.message)] })
          }
          return cb ? Promise.resolve(valid) : Promise.reject(valid)
        })
      } else {
        let valid = true
        if (cb) {
          cb(valid)
        }
      }
      return validPromise
    }
    /***************************/
    /* Public methods end      */
    /***************************/
  }
}
</script>
