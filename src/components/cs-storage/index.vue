<template>
  <el-dialog
    title="资源选取"
    :visible.sync="visible"
    :append-to-body="true"
    :close-on-click-modal="false"
    width="769px">

    <!-- 搜索框开始 -->
    <el-form :model="form" style="margin-top: -25px;" size="small" @submit.native.prevent>
      <el-row :gutter="20">
        <el-col :span="10">
          <el-form-item>
            <el-button-group>
              <el-tooltip content="勾选当前页全部资源" placement="top">
                <el-button @click="allCheckBox" icon="el-icon-plus">全选</el-button>
              </el-tooltip>
              <el-tooltip content="反向勾选当前页资源" placement="top">
                <el-button @click="reverseCheckBox" icon="el-icon-minus">反选</el-button>
              </el-tooltip>
              <el-tooltip content="取消当前页勾选" placement="top">
                <el-button @click="cancelCheckBox" icon="el-icon-close">取消</el-button>
              </el-tooltip>
              <el-tooltip content="清除所有已选中勾选" placement="top">
                <el-button @click="checkList = []" icon="el-icon-refresh">清除</el-button>
              </el-tooltip>
            </el-button-group>
          </el-form-item>
        </el-col>

        <el-col :span="14">
          <el-form-item prop="name">
            <el-input
              v-model="form.name"
              placeholder="输入资源名称进行搜索"
              @keyup.enter.native="handleSearch()"
              :clearable="true"
              size="small">
              <el-button slot="append" icon="el-icon-search" @click="handleSearch"/>
            </el-input>
          </el-form-item>
        </el-col>
      </el-row>
    </el-form>

    <!-- 面包屑开始 -->
    <el-breadcrumb class="breadcrumb cs-mb" separator-class="el-icon-arrow-right">
      <el-breadcrumb-item>
        <a class="cs-cp" @click="switchDirectory(0)">资源管理</a>
      </el-breadcrumb-item>

      <el-breadcrumb-item
        v-for="item in naviData"
        :key="item.storage_id">
        <a class="cs-cp" @click="switchDirectory(item.storage_id)">{{item.name}}</a>
      </el-breadcrumb-item>
    </el-breadcrumb>

    <!-- 资源列表开始 -->
    <el-checkbox-group v-model="checkList">
      <ul class="storage-list">
        <li v-for="(item, index) in currentTableData" :key="index">
          <dl>
            <dt>
              <div class="picture cs-m-5">
                <el-checkbox v-if="item.type !== 2" :label="item.storage_id" class="check">
                  {{checkIndex[item.storage_id]}}
                </el-checkbox>
                <el-image fit="fill" :src="item | getImageThumb" @click.native="handleOpen(index)" lazy/>
              </div>

              <el-tooltip placement="top" :enterable="false" :open-delay="300">
                <div slot="content">
                  <span>名称：{{item.name}}</span><br/>
                  <span>日期：{{item.create_time}}</span><br/>
                  <span v-if="item.type === 0">尺寸：{{`${item.pixel['width']},${item.pixel['height']}`}}</span>
                  <span v-else>类型：<i :class="item.type | getFileTypeIocn"/></span>
                </div>
                <span class="storage-name cs-ml-5">{{item.name}}</span>
              </el-tooltip>
            </dt>
          </dl>
        </li>
      </ul>
    </el-checkbox-group>

    <!-- 翻页开始 -->
    <page-footer
      style="margin: 0; padding: 20px 0 0 0;"
      :current="page.current"
      :size="page.size"
      :total="page.total"
      :is-size="false"
      @change="handlePaginationChange"/>

    <!-- 确认,取消 -->
    <div slot="footer" class="dialog-footer">
      <div style="float: left; font-size: 13px;">
        <span v-if="checkList.length > limit && limit !== 0" style="color: #F56C6C;">
          当前已选 {{checkList.length}} 个，最多允许选择 {{limit}} 个资源
        </span>

        <span v-else>当前已选 {{checkList.length}} 个资源</span>
      </div>

      <el-button
        @click="visible = false"
        size="small">取消</el-button>

      <el-button
        type="primary"
        :loading="loadingCollection"
        :disabled="checkList.length > limit && limit !== 0"
        @click="handleConfirm"
        size="small">确定</el-button>
    </div>
  </el-dialog>
</template>

<script>
import storage from './components/mixins/storage'
import {
  getStorageNavi,
  getStorageList,
  getStorageCollection
} from '@/api/upload/storage'

export default {
  name: 'cs-storage',
  mixins: [storage],
  components: {
    PageFooter: () => import('@/components/cs-footer')
  },
  props: {
    // 确认按钮事件
    confirm: {
      type: Function
    },
    // 最大选择数(0表示不限制)
    limit: {
      type: Number,
      required: false,
      default: 0
    }
  },
  data() {
    return {
      visible: false,
      loadingCollection: false,
      naviData: [],
      checkList: [],
      checkIndex: {},
      currentTableData: [],
      isCheckDirectory: false,
      source: '',
      storageType: [],
      form: {
        name: '',
        storage_id: 0,
        order_type: 'desc',
        order_field: 'storage_id'
      },
      page: {
        current: 1,
        size: 48,
        total: 0
      }
    }
  },
  watch: {
    'form.storage_id': {
      handler(val) {
        getStorageNavi(val)
          .then(res => {
            this.naviData = res.data || []
          })
      }
    },
    checkList: {
      handler(val) {
        let checkIndex = {}
        val.forEach((value, index) => {
          checkIndex[value] = index + 1
        })

        this.checkIndex = { ...checkIndex }
      }
    }
  },
  methods: {
    handleStorageDlg(type = [], source = '') {
      this.visible = true
      this.storageType = type
      this.source = source
      this.checkList = []
      this.loadingCollection = false
      this.handleSubmit()
    },
    switchDirectory(val) {
      this.form.name = null
      this.form.storage_id = val || 0
      this.handleSubmit()
    },
    handleOpen(key) {
      if (this.currentTableData[key].type === 2) {
        this.switchDirectory(this.currentTableData[key].storage_id)
      }
    },
    handlePaginationChange(val) {
      this.page = val
      this.$nextTick(() => {
        this.handleSubmit()
      })
    },
    handleSubmit() {
      getStorageList({
        ...this.form,
        type: this.storageType,
        page_no: this.page.current,
        page_size: this.page.size
      })
        .then(res => {
          this.currentTableData = res.data.items || []
          this.page.total = res.data.total_result
        })
    },
    handleConfirm() {
      if (this.checkList.length <= 0) {
        this.$emit('confirm', [], this.source)
        this.visible = false
        return
      }

      this.loadingCollection = true
      getStorageCollection({
        storage_id: this.checkList,
        order_type: this.form.order_type,
        order_field: this.form.order_field
      })
        .then(res => {
          this.checkList = []
          this.visible = false
          this.$emit('confirm', res.data || [], this.source)
        })
        .finally(() => {
          this.loadingCollection = false
        })
    },
    handleSearch() {
      this.page.current = 1
      this.form.storage_id = 0
      this.handleSubmit()
    }
  }
}
</script>

<style lang="scss" scoped>
  .breadcrumb {
    background-color: #FFF;
    border: 1px solid $color-border-1;
    padding: 10px !important;
    line-height: 16px;
  }

  .storage-list {
    overflow: hidden;
    list-style: none;
    padding: 0;
    margin: 0;
    border-style: solid;
    border-color: $color-border-1;
    border-width: 1px 0 0 1px;
    background-color: #FFF;
  }

  .storage-list li {
    float: left;
    opacity: 1;
    border-style: solid;
    border-color: $color-border-1;
    border-width: 0 1px 1px 0;
  }

  .storage-list > li:hover {
    background-color: $color-bg;
  }

  .storage-list li dl dt .picture {
    border: none;
  }

  .storage-list li dl dt .covers .el-image,
  .storage-list li dl dt .picture .el-image {
    background-color: #F5F7FA;
    text-align: center;
    vertical-align: middle;
    display: table-cell;
    width: 80px;
    height: 80px;
    overflow: hidden;
    cursor: pointer;
  }

  .storage-list li:after {
    content: "";
    height: 100%;
  }

  .storage-list li:after,
  .storage-list li span {
    display: inline-block;
    vertical-align: middle;
  }

  .storage-list li span {
    line-height: normal;
    color: $color-text-sub;
    transition: color .15s linear;
  }

  .storage-list .storage-name {
    color: $color-text-main;
    text-overflow: ellipsis;
    white-space: nowrap;
    overflow: hidden;
    font-size: 12px;
    width: 80px;
    height: 20px;
  }

  .check {
    position: absolute;
    margin-top: 2px;
    margin-left: 2px;
    width: 78px;
    height: 80px;
    z-index: 9;

    /deep/ .el-checkbox__label {
      padding-right: 10px;
      line-height: 16px;
      font-size: 12px;
      float: right;
      background: $color-bg;
    }
  }

  .el-image /deep/ .el-image__inner {
    width: auto;
    height: auto;
    max-width: 80px;
    max-height: 80px;
  }
</style>
