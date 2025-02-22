<template>
  <div class="business-layout">
    <div class="business-options clearfix">
      <cmdb-auth class="fl" :auth="{ type: $OPERATION.C_BUSINESS }">
        <bk-button slot-scope="{ disabled }"
          class="fl"
          theme="primary"
          :disabled="disabled"
          @click="handleCreate">
          {{$t('新建')}}
        </bk-button>
      </cmdb-auth>
      <div class="options-button fr">
        <cmdb-auth class="inline-block-middle" :auth="{ type: $OPERATION.BUSINESS_ARCHIVE }">
          <icon-button slot-scope="{ disabled }"
            class="mr10"
            icon="icon-cc-history"
            v-bk-tooltips.top="$t('查看已归档业务')"
            :disabled="disabled"
            @click="routeToHistory">
          </icon-button>
        </cmdb-auth>
        <icon-button
          icon="icon-cc-setting"
          v-bk-tooltips.top="$t('列表显示属性配置')"
          @click="columnsConfig.show = true">
        </icon-button>
      </div>
      <div class="options-filter clearfix fr">
        <cmdb-property-selector
          class="filter-selector fl"
          v-model="filter.field"
          :properties="properties"
          :object-unique="objectUnique">
        </cmdb-property-selector>
        <component class="filter-value fl"
          :is="`cmdb-search-${filterType}`"
          :placeholder="filterPlaceholder"
          :class="filterType"
          v-bind="filterComponentProps"
          v-model="filter.value"
          @change="handleFilterValueChange"
          @enter="handleFilterValueEnter"
          @clear="handleFilterValueEnter">
        </component>
      </div>
    </div>
    <bk-table class="business-table"
      v-bkloading="{ isLoading: $loading('post_searchBusiness_list') }"
      :data="table.list"
      :pagination="table.pagination"
      :max-height="$APP.height - 200"
      @sort-change="handleSortChange"
      @page-limit-change="handleSizeChange"
      @page-change="handlePageChange">
      <bk-table-column v-for="column in table.header"
        sortable="custom"
        :key="column.id"
        :prop="column.id"
        :label="column.name"
        min-width="80"
        show-overflow-tooltip>
        <template slot-scope="{ row }">
          <cmdb-property-value
            :theme="column.id === 'bk_biz_id' ? 'primary' : 'default'"
            :value="row[column.id]"
            :show-unit="false"
            :property="column.property"
            @click.native.stop="handleValueClick(row, column)">
          </cmdb-property-value>
        </template>
      </bk-table-column>
      <bk-table-column :label="$t('操作')" fixed="right">
        <template slot-scope="{ row }">
          <cmdb-auth @click.native.stop :auth="{ type: $OPERATION.BUSINESS_ARCHIVE, relation: [row.bk_biz_id] }">
            <template slot-scope="{ disabled }">
              <span class="text-primary"
                style="color: #dcdee5 !important; cursor: not-allowed;"
                v-if="row['bk_biz_name'] === '蓝鲸' && !disabled"
                v-bk-tooltips.top="$t('内置业务不可归档')"
                @click.stop>
                {{$t('归档')}}
              </span>
              <bk-button v-else
                theme="primary"
                :disabled="disabled"
                :text="true"
                @click.stop="handleDelete(row)">
                {{$t('归档')}}
              </bk-button>
            </template>
          </cmdb-auth>
        </template>
      </bk-table-column>
      <cmdb-table-empty
        slot="empty"
        :stuff="table.stuff"
        :auth="{ type: $OPERATION.C_BUSINESS }">
        <i18n path="业务列表提示语" class="table-empty-tips">
          <bk-link theme="primary" place="auth" @click="handleApplyPermission">{{$t('申请查看权限')}}</bk-link>
          <cmdb-auth :auth="{ type: $OPERATION.C_BUSINESS }" place="create">
            <bk-button slot-scope="{ disabled }" text
              theme="primary"
              class="text-btn"
              :disabled="disabled"
              @click="handleCreate">
              {{$t('立即创建')}}
            </bk-button>
          </cmdb-auth>
        </i18n>
      </cmdb-table-empty>
    </bk-table>
    <bk-sideslider
      v-transfer-dom
      :is-show.sync="slider.show"
      :title="slider.title"
      :width="800"
      :before-close="handleSliderBeforeClose">
      <bk-tab :active.sync="tab.active" type="unborder-card" slot="content" v-if="slider.show">
        <bk-tab-panel name="attribute" :label="$t('属性')" style="width: calc(100% + 40px);margin: 0 -20px;">
          <cmdb-form
            ref="form"
            :object-unique="objectUnique"
            :properties="properties"
            :property-groups="propertyGroups"
            :inst="attribute.inst.edit"
            :type="attribute.type"
            :save-auth="saveAuth"
            @on-submit="handleSave"
            @on-cancel="handleSliderBeforeClose">
          </cmdb-form>
        </bk-tab-panel>
      </bk-tab>
    </bk-sideslider>
    <bk-sideslider v-transfer-dom :is-show.sync="columnsConfig.show" :width="600" :title="$t('列表显示属性配置')">
      <cmdb-columns-config slot="content"
        v-if="columnsConfig.show"
        :properties="properties"
        :selected="columnsConfig.selected"
        :disabled-columns="columnsConfig.disabledColumns"
        @on-apply="handleApplayColumnsConfig"
        @on-cancel="columnsConfig.show = false"
        @on-reset="handleResetColumnsConfig">
      </cmdb-columns-config>
    </bk-sideslider>
  </div>
</template>

<script>
  import { translateAuth } from '@/setup/permission'
  import { mapState, mapGetters, mapActions } from 'vuex'
  import { MENU_RESOURCE_BUSINESS_HISTORY, MENU_RESOURCE_BUSINESS_DETAILS } from '@/dictionary/menu-symbol'
  import cmdbColumnsConfig from '@/components/columns-config/columns-config.vue'
  import cmdbPropertySelector from '@/components/property-selector'
  import RouterQuery from '@/router/query'
  import Utils from '@/components/filters/utils'
  import throttle from  'lodash.throttle'
  export default {
    components: {
      cmdbColumnsConfig,
      cmdbPropertySelector
    },
    data() {
      return {
        objectUnique: [],
        properties: [],
        propertyGroups: [],
        table: {
          header: [],
          list: [],
          pagination: {
            count: 0,
            current: 1,
            ...this.$tools.getDefaultPaginationConfig()
          },
          defaultSort: 'bk_biz_id',
          sort: 'bk_biz_id',
          stuff: {
            type: 'default',
            payload: {
              resource: this.$t('业务')
            }
          }
        },
        filter: {
          field: 'bk_biz_name',
          value: '',
          operator: '$eq'
        },
        slider: {
          show: false,
          title: ''
        },
        tab: {
          active: 'attribute'
        },
        attribute: {
          type: null,
          inst: {
            edit: {},
            details: {}
          }
        },
        columnsConfig: {
          show: false,
          selected: [],
          disabledColumns: ['bk_biz_id', 'bk_biz_name']
        },
        columnsConfigKey: 'biz_custom_table_columns'
      }
    },
    computed: {
      ...mapState('userCustom', ['globalUsercustom']),
      ...mapGetters(['supplierAccount', 'userName']),
      ...mapGetters('userCustom', ['usercustom']),
      ...mapGetters('objectBiz', ['bizId']),
      ...mapGetters('objectModelClassify', ['getModelById']),
      customBusinessColumns() {
        return this.usercustom[this.columnsConfigKey] || []
      },
      globalCustomColumns() {
        return this.globalUsercustom.biz_global_custom_table_columns || []
      },
      saveAuth() {
        const { type } = this.attribute
        if (type === 'create') {
          return { type: this.$OPERATION.C_BUSINESS }
        } if (type === 'update') {
          return { type: this.$OPERATION.U_BUSINESS }
        }
        return null
      },
      model() {
        return this.getModelById('biz') || {}
      },
      filterProperty() {
        const property = this.properties.find(property => property.bk_property_id === this.filter.field)
        return property || null
      },
      filterType() {
        if (this.filterProperty) {
          return this.filterProperty.bk_property_type
        }
        return 'singlechar'
      },
      filterPlaceholder() {
        return Utils.getPlaceholder(this.filterProperty)
      },
      filterComponentProps() {
        return Utils.getBindProps(this.filterProperty)
      }
    },
    watch: {
      'filter.field'() {
        const defaultData = Utils.getDefaultData(this.filterProperty)
        this.filter.value = defaultData.value
        this.filter.operator = defaultData.operator
      },
      'slider.show'(show) {
        if (!show) {
          this.tab.active = 'attribute'
        }
      },
      customBusinessColumns() {
        this.setTableHeader()
      }
    },
    async created() {
      try {
        this.properties = await this.searchObjectAttribute({
          injectId: 'biz',
          params: {
            bk_obj_id: 'biz',
            bk_supplier_account: this.supplierAccount
          },
          config: {
            requestId: 'post_searchObjectAttribute_biz',
            fromCache: true
          }
        })
        await Promise.all([
          this.getPropertyGroups(),
          this.getObjectUnique(),
          this.setTableHeader()
        ])
        this.throttleGetTableData = throttle(this.getTableData, 300, { leading: false, trailing: true })
        this.unwatch = RouterQuery.watch('*', async ({
          page = 1,
          limit = this.table.pagination.limit,
          filter = '',
          operator = '',
          field = 'bk_biz_name'
        }) => {
          this.filter.field = field
          this.table.pagination.current = parseInt(page, 10)
          this.table.pagination.limit = parseInt(limit, 10)
          await this.$nextTick()
          const defaultData = Utils.getDefaultData(this.filterProperty)
          this.filter.operator = operator || defaultData.operator
          // eslint-disable-next-line max-len
          this.filter.value = this.formatFilterValue({ value: filter, operator: this.filter.operator }, defaultData.value)
          this.throttleGetTableData()
        }, { immediate: true })
      } catch (e) {
        // ignore
      }
    },
    beforeDestroy() {
      this.unwatch()
    },
    methods: {
      ...mapActions('objectModelFieldGroup', ['searchGroup']),
      ...mapActions('objectModelProperty', ['searchObjectAttribute']),
      ...mapActions('objectBiz', [
        'searchBusiness',
        'archiveBusiness',
        'updateBusiness',
        'createBusiness',
        'searchBusinessById'
      ]),
      getPropertyGroups() {
        return this.searchGroup({
          objId: 'biz',
          params: {},
          config: {
            fromCache: true,
            requestId: 'post_searchGroup_biz'
          }
        }).then((groups) => {
          this.propertyGroups = groups
          return groups
        })
      },
      getObjectUnique() {
        return this.$store.dispatch('objectUnique/searchObjectUniqueConstraints', {
          objId: 'biz',
          params: {}
        }).then((data) => {
          this.objectUnique = data
          return data
        })
      },
      setTableHeader() {
        return new Promise((resolve) => {
          // eslint-disable-next-line max-len
          const customColumns = this.customBusinessColumns.length ? this.customBusinessColumns : this.globalCustomColumns
          // eslint-disable-next-line max-len
          const headerProperties = this.$tools.getHeaderProperties(this.properties, customColumns, this.columnsConfig.disabledColumns)
          resolve(headerProperties)
        }).then((properties) => {
          this.updateTableHeader(properties)
          this.columnsConfig.selected = properties.map(property => property.bk_property_id)
        })
      },
      updateTableHeader(properties) {
        this.table.header = properties.map(property => ({
          id: property.bk_property_id,
          name: this.$tools.getHeaderPropertyName(property),
          property
        }))
      },
      handleValueClick(row, column) {
        if (column.id !== 'bk_biz_id') {
          return false
        }
        this.$routerActions.redirect({
          name: MENU_RESOURCE_BUSINESS_DETAILS,
          params: {
            bizId: row.bk_biz_id
          },
          history: true
        })
      },
      handleSortChange(sort) {
        this.table.sort = this.$tools.getSort(sort)
        this.handlePageChange(1)
      },
      handleSizeChange(size) {
        this.table.pagination.limit = size
        this.handlePageChange(1)
      },
      handlePageChange(page) {
        this.table.pagination.current = page
        this.getTableData()
      },
      getBusinessList(config = { cancelPrevious: true }) {
        return this.searchBusiness({
          params: this.getSearchParams(),
          config: Object.assign({ requestId: 'post_searchBusiness_list' }, config)
        })
      },
      formatFilterValue({ value: currentValue, operator }, defaultValue) {
        let value = currentValue.toString().length ? currentValue : defaultValue
        const isNumber = ['int', 'float'].includes(this.filterType)
        if (isNumber && value) {
          value = parseFloat(value, 10)
        } else if (operator === '$in') {
          // eslint-disable-next-line no-nested-ternary
          value = Array.isArray(value) ? value : !!value ? [value] : []
        } else if (Array.isArray(value)) {
          value = value.filter(value => !!value)
        }
        return value
      },
      handleFilterValueChange() {
        const hasEnterEvnet = ['float', 'int', 'longchar', 'singlechar']
        if (hasEnterEvnet.includes(this.filterType)) return
        this.handleFilterValueEnter()
      },
      handleFilterValueEnter() {
        RouterQuery.set({
          _t: Date.now(),
          page: 1,
          field: this.filter.field,
          filter: this.filter.value,
          operator: this.filter.operator
        })
      },
      getTableData() {
        this.getBusinessList({ cancelPrevious: true, globalPermission: false }).then((data) => {
          if (data.count && !data.info.length) {
            this.table.pagination.current -= 1
            this.getTableData()
          }
          this.table.list = data.info
          this.table.pagination.count = data.count

          this.table.stuff.type = this.filter.value.toString().length ? 'search' : 'default'

          return data
        })
          .catch(({ permission }) => {
            if (!permission) return
            this.table.stuff = {
              type: 'permission',
              payload: { permission }
            }
          })
      },
      getSearchParams() {
        const params = {
          condition: {
            bk_data_status: { $ne: 'disabled' }
          },
          fields: [],
          page: {
            start: this.table.pagination.limit * (this.table.pagination.current - 1),
            limit: this.table.pagination.limit,
            sort: this.table.sort
          }
        }
        if (!this.filter.value.toString()) {
          return params
        }
        if (this.filter.operator === '$range') {
          const [start, end] = this.filter.value
          params.condition[this.filter.field] = {
            $gte: start,
            $lte: end
          }
          return params
        }
        if (this.filterType === 'objuser') {
          const filterValue  = this.filter.value
          const multiple = filterValue.length > 1
          params.condition[this.filter.field] = multiple ? { $in: filterValue } : filterValue.toString()
          return params
        }
        params.condition[this.filter.field] = { [this.filter.operator]: this.filter.value }
        return params
      },
      async handleEdit(inst) {
        const bizNameProperty = this.$tools.getProperty(this.properties, 'bk_biz_name')
        bizNameProperty.isreadonly = inst.bk_biz_name === '蓝鲸'
        this.attribute.inst.edit = inst
        this.attribute.type = 'update'
      },
      handleCreate() {
        this.attribute.type = 'create'
        this.attribute.inst.edit = {}
        this.slider.show = true
        this.slider.title = `${this.$t('创建')} ${this.model.bk_obj_name}`
      },
      handleDelete(inst) {
        this.$bkInfo({
          title: this.$t('确认要归档', { name: inst.bk_biz_name }),
          subTitle: this.$t('归档确认信息'),
          confirmFn: () => {
            this.archiveBusiness(inst.bk_biz_id).then(() => {
              this.slider.show = false
              this.$success(this.$t('归档成功'))
              this.getTableData()
              this.$http.cancel('post_searchBusiness_$ne_disabled')
            })
          }
        })
      },
      handleSave(values, changedValues, originalValues, type) {
        if (type === 'update') {
          this.updateBusiness({
            bizId: originalValues.bk_biz_id,
            params: values
          }).then(() => {
            this.attribute.inst.details = Object.assign({}, originalValues, values)
            this.getTableData()
            this.handleCancel()
            this.$success(this.$t('修改成功'))
            this.$http.cancel('post_searchBusiness_$ne_disabled')
          })
        } else {
          delete values.bk_biz_id // properties中注入了前端自定义的bk_biz_id属性
          this.createBusiness({
            params: values
          }).then(() => {
            this.handlePageChange(1)
            this.handleCancel()
            this.$success(this.$t('创建成功'))
            this.$http.cancel('post_searchBusiness_$ne_disabled')
          })
        }
      },
      handleCancel() {
        if (this.attribute.type === 'create') {
          this.slider.show = false
        }
      },
      handleApplayColumnsConfig(properties) {
        this.$store.dispatch('userCustom/saveUsercustom', {
          [this.columnsConfigKey]: properties.map(property => property.bk_property_id)
        })
        this.columnsConfig.show = false
      },
      handleResetColumnsConfig() {
        this.$store.dispatch('userCustom/saveUsercustom', {
          [this.columnsConfigKey]: []
        })
        this.columnsConfig.show = false
      },
      routeToHistory() {
        this.$routerActions.redirect({
          name: MENU_RESOURCE_BUSINESS_HISTORY,
          history: true
        })
      },
      handleSliderBeforeClose() {
        if (this.tab.active === 'attribute') {
          const $form = this.$refs.form
          const { changedValues } = $form
          if (Object.keys(changedValues).length) {
            return new Promise((resolve) => {
              this.$bkInfo({
                title: this.$t('确认退出'),
                subTitle: this.$t('退出会导致未保存信息丢失'),
                extCls: 'bk-dialog-sub-header-center',
                confirmFn: () => {
                  resolve(true)
                  this.handleCancel()
                },
                cancelFn: () => {
                  resolve(false)
                }
              })
            })
          }
          this.handleCancel()
          return true
        }
        this.handleCancel()
        return true
      },
      async handleApplyPermission() {
        try {
          const permission = translateAuth({
            type: this.$OPERATION.R_BUSINESS,
            relation: []
          })
          const url = await this.$store.dispatch('auth/getSkipUrl', { params: permission })
          window.open(url)
        } catch (e) {
          console.error(e)
        }
      }
    }
  }
</script>

<style lang="scss" scoped>
    .business-layout {
        padding: 15px 20px 0;
    }
    .options-filter{
        position: relative;
        margin-right: 10px;
        .filter-selector{
            width: 120px;
            border-radius: 2px 0 0 2px;
            margin-right: -1px;
        }
        .filter-value{
            width: 320px;
            border-radius: 0 2px 2px 0;
            /deep/ .bk-form-input {
                border-radius: 0 2px 2px 0;
            }
        }
        .filter-search{
            position: absolute;
            right: 10px;
            top: 9px;
            cursor: pointer;
        }
    }
    .options-button{
        font-size: 0;
        .bk-button {
            width: 32px;
            padding: 0;
            /deep/ .bk-icon {
                line-height: 14px;
            }
        }
    }
    .business-table{
        margin-top: 14px;
    }
    .table-empty-tips {
        display: flex;
        align-items: center;
        justify-content: center;
        .text-btn {
            font-size: 14px;
        }
    }
</style>
