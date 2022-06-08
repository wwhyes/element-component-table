
<script>
import T from 'element-ui/lib/table'
export default {
  props: Object.assign({}, T.props, {
    data: {
      type: Function,
      required: true,
      default: () => {
        return Promise.reject()
      }
    },
    height: {
      type: String,
      default: '100%'
    },
    showPagination: {
      type: Boolean,
      default: true
    },
    paginationPostion: {
      type: String,
      default: 'bottom'
    },
    layoutTop: {// 分页控件-上
      type: String,
      default: () => 'total, prev, slot, next'
    },
    layoutBottom: {// 分页控件-下
      type: String,
      default: () => 'total, sizes, prev, pager, next'
    },
    pageSize: {// 每页多少条数据
      type: Number,
      default: () => 10
    },
    prevText: {// 替代图标显示的下一页文字
      type: String,
      default: () => ''
    },
    nextText: {// 替代图标显示的下一页文字
      type: String,
      default: () => ''
    }
  }),
  data () {
    return {
      loading: false,
      localData: [],
      localPagination: {
        currentPage: 1,
        pageSize: 10,
        total: 0
      },
      // 存储表格onchange时的filters
      filters: {}
    }
  },
  methods: {
    clearSelection () {
      this.$refs.stable.clearSelection()
    },
    /**
     * 表格重新加载方法
     * 如果参数为 true, 则强制刷新到第一页
     * @param Boolean bool
     */
    refresh (bool = false) {
      bool && (this.localPagination.currentPage = 1)
      this.loadData()
    },
    /**
     * 加载数据方法
     * @param {Object} pagination 分页选项器
     * @param {Object} filters 过滤条件
     */
    loadData (pagination, filters = this.filters, ...other) {
      this.localData = []
      this.filters = filters
      this.loading = true
      const parameter = Object.assign(
        {
          page: pagination?.page || (this.showPagination && this.localPagination?.currentPage) || undefined,
          page_size: pagination?.pageSize || (this.showPagination && this.localPagination?.pageSize) || undefined
        },
        { ...filters }
      )
      const result = this.data(parameter, ...other)
      // 对接自己的通用数据接口需要修改下方代码中的 r.pageNo, r.totalCount, r.data
      // eslint-disable-next-line
      if ((typeof result === 'object' || typeof result === 'function') && typeof result.then === 'function') {
        return result.then(r => {
          let localPagination = null

          if (this.showPagination) {
            localPagination = Object.assign({}, this.localPagination, {
              currentPage: r.current_page || r.current,
              pageSize: r.page_size || r.per_page,
              total: r.total
            })
          }

          this.localPagination = localPagination
          // 为防止删除数据后导致页面当前页面数据长度为 0 ,自动翻页到上一页
          if (r.data.length === 0 && this.showPagination && this.localPagination.currentPage > 1) {
            this.localPagination.currentPage--
            this.loadData()
            return
          }

          // 这里用于判断接口是否有返回 r.totalCount 且 this.showPagination = true 且 pageNo 和 pageSize 存在 且 totalCount 小于等于 pageNo * pageSize 的大小
          // 当情况满足时，表示数据不满足分页大小，关闭 table 分页功能
          try {
            if ((this.showPagination && r.total_count && r.total_count <= (this.localPagination.currentPage * this.localPagination.pageSize))) {
              this.localPagination.hideOnSinglePage = true
            }
          } catch (e) {
            this.localPagination = null
          }
          this.localData = r.data // 返回结果中的数组数据
        }).finally(() => {
          this.loading = false
        })
      }

      return Promise.reject()
    },
    handlePageChange (currentPage) {
      this.localPagination.currentPage = currentPage
      this.loadData()
    },
    handlePageSizeChange (pageSize) {
      this.localPagination.currentPage = 1
      this.localPagination.pageSize = pageSize
      this.loadData()
    }
  },
  watch: {
    pageSize: {
      handler (newValue) {
        this.localPagination.pageSize = newValue
      },
      immediate: true
    }
  },
  created () {
    this.loadData()
  },
  render () {
    const { loading, paginationPostion, showPagination, localPagination, handlePageChange, handlePageSizeChange } = this
    const props = {}
    const localKeys = Object.keys(this.$data)
    Object.keys(T.props).forEach(k => {
      const localKey = `local${k.substring(0, 1).toUpperCase()}${k.substring(1)}`

      if (localKeys.includes(localKey)) {
        props[k] = this[localKey]
        return props[k]
      }

      this[k] && (props[k] = this[k])
    })

    const table = (
      <el-table {...{ props, on: this.$listeners, ref: 'stable' }}>
        { Object.keys(this.$slots).map(name => (<template slot={name}>{this.$slots[name]}</template>)) }
      </el-table>
    )

    const paginationConfig = paginationPostion === 'top'
      ? { layout: this.layoutTop, 'prev-text': this.prevText, 'next-text': this.nextText }
      : {
        layout: this.layoutBottom,
        pageSizes: [10, 30, 50, 100],
        'prev-text': this.prevText,
        'next-text': this.nextText
      }
    const pagination = showPagination
      ? (
        <div class="table-pagination">
          <el-pagination
            {...{
              props: Object.assign({ background: true }, paginationConfig, localPagination),
              class: 'el-table-pagination'
            }}
            on-current-change={handlePageChange}
            on-size-change={handlePageSizeChange}
          >
            <span class="el-table-pagination__numb">{localPagination?.currentPage || 1}/{Math.ceil((localPagination?.total || 0) / (localPagination?.pageSize || 10))}</span>
          </el-pagination>
        </div>
      )
      : null

    const tableWrapperClass = `table-wrapper table-wrapper--pagination-postion-${paginationPostion}`
    return (
      <div v-loading={loading} class={tableWrapperClass}>
        { paginationPostion === 'both' ? pagination : null }
        { table }
        { pagination }
      </div>
    )
  }
}

</script>

<style lang="less" scoped>
.table-wrapper {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.table-wrapper > .el-table {
  flex: 1;
  min-height: 0;
}

.table-wrapper > .el-table--border {
  margin-bottom: 1px;
  box-shadow: inset -1px -1px 2px #ebeef5;
}

.table-wrapper /deep/ .el-table::before {
  display: none;
}

.table-wrapper /deep/ .el-table .cell {
  white-space: nowrap;
}

.table-wrapper /deep/ .el-table__expanded-cell[class*=cell] {
  padding: 0;
}

.table-wrapper /deep/ .el-table-column--selection .cell {
  padding-left: 10px;
  padding-right: 10px;
}

.table-wrapper /deep/ p {
  overflow: hidden;
  text-overflow: ellipsis;
}

.table-pagination {
  text-align: right;

  /deep/ .el-icon {
    margin: 0;
  }
}

.el-table-pagination__numb {
  text-align: center;
}

.table-wrapper--pagination-postion-bottom {
  .table-pagination {
    margin-top: 20px;
  }

  .el-table-pagination.el-pagination {
    margin: 0 10px;
    padding: 4px 20px;
    display: inline-block;
    border: 1px solid #dbdbdb;
    border-radius: 18px;
  }
  .el-table-pagination.el-pagination.is-background .el-pager li:not(.disabled).active {
    background-color: #fff0eb;
    border:1px solid #f36f3a;
    color: #f36f3a;
  }

  .el-pagination.is-background .el-pager li:not(.disabled).number {
    background-color: #fff;
    border:1px solid #e2e2e2;
    color: #606266;
  }
}

.table-wrapper--pagination-postion-top {
  flex-direction: column-reverse;

  .table-pagination {
    margin-bottom: 10px;
  }
}
.el-table-pagination.el-pagination /deep/.btn-prev span,
.el-table-pagination.el-pagination /deep/.btn-next span {
  padding: 0 5px;
}
</style>
