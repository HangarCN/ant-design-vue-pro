<template>
  <page-header-wrapper :title="false">
    <div class="organization">
      <div class="organizationList">
        <organization-tree-data
          @organizationTreeData="getFormOrganizationTreeData"
          @selectOrganization="selectOrganization"
          @cancelSelect="cancelSelectOrganizationTreeData"
          ref="organizationTree"
        ></organization-tree-data>
      </div>
      <div class="organizationMain">
        <a-spin tip="加载中..." class="position" v-if="organizationloading"> </a-spin>
        <div class="organizationSearch">
          <div class="table-page-search-wrapper">
            <a-form layout="inline">
              <a-row :gutter="48">
                <a-col :md="8" :sm="24">
                  <a-form-item label="名称">
                    <a-input allowClear v-model="searchParameters.searchName" placeholder="请输入" />
                  </a-form-item>
                </a-col>
                <a-col :md="8" :sm="24">
                  <a-form-item label="创建时间">
                    <a-range-picker
                      @change="onChangeData"
                      allowClear
                      v-model="searchParameters.organizationCreateData"
                    />
                  </a-form-item>
                </a-col>
                <template v-if="advanced">
                  <a-col :md="8" :sm="24">
                    <a-form-item label="代码">
                      <a-input v-model="searchParameters.searchCode" placeholder="请输入" allowClear />
                    </a-form-item>
                  </a-col>
                  <a-col :md="8" :sm="24">
                  </a-col>
                  <a-col :md="8" :sm="24"> </a-col>
                </template>
                <a-col :md="8" :sm="24" style="display:flex;justify-content: flex-end">
                  <span>
                    <a-button type="primary" @click="() => this.searchOrganizationTableData()">查询</a-button>
                    <a-button style="margin-left: 8px" @click="handleReset()">重置</a-button>
                    <a @click="() => (this.advanced = !this.advanced)" style="margin-left: 8px">
                      {{ advanced ? '收起' : '展开' }}
                      <a-icon :type="advanced ? 'up' : 'down'" />
                    </a>
                  </span>
                </a-col>
              </a-row>
            </a-form>
          </div>
        </div>
        <div class="organizationTable">
          <div class="organizationTableAdd">
            <div>组织机构列表</div>
            <div class="organizationTableAddButton">
              <a-button type="primary" @click="handleAddOrganization()">
                新增组织
              </a-button>
            </div>
          </div>
          <div class="organizationTableContent">
            <a-table
              :columns="columns"
              :data-source="organizationTableData"
              :pagination="false"
              size="middle"
              :rowKey="
                (record, index) => {
                  return index;
                }
              "
              bordered
            >
              <template slot="action" slot-scope="text, record">
                <a slot="action" href="javascript:;" style="margin-left:5px" @click="handleEdit(record)">编辑</a>
                <a-popconfirm
                  slot="action"
                  title="此操作将删除该条数据，是否继续?"
                  ok-text="是"
                  cancel-text="否"
                  @confirm="handleDelete(record)"
                >
                  <a href="javascript:;" style="margin-left:5px">删除</a>
                </a-popconfirm>
              </template>
            </a-table>
          </div>
        </div>
        <div class="organizationPagination">
          <a-pagination
            v-model="currentPage"
            :page-size-options="pageSizeOptions"
            :total="organizationTableTotal"
            :show-total="total => `共 ${organizationTableTotal} 条`"
            show-size-changer
            :page-size="pageObject.pageSize"
            @change="handlePageNumberChange"
            @showSizeChange="onPageSizeChange"
          >
          </a-pagination>
          跳至 <a-input v-model="jumper" style="width:50px;margin-left:10px;margin-right:10px" @blur="blurJumperInput()"/>页
        </div>
        <a-modal
          v-model="modleVisible"
          :title="form.id ? '编辑组织' : '新增组织'"
          @cancel="() => (this.clearFormData(), (this.modleVisible = false))"
          :confirm-loading="formButtonDisableFlag"
          @ok="onSubmit"
        >
          <a-form-model
            ref="organizationRuleForm"
            :model="form"
            :rules="rules"
            :label-col="labelCol"
            :wrapper-col="wrapperCol"
          >
            <a-form-model-item ref="name" label="组织名称" prop="name">
              <a-input v-model.trim="form.name" placeholder="请输入组织名称" />
            </a-form-model-item>
            <a-form-model-item ref="code" label="组织代码" prop="code">
              <a-input v-model.trim="form.code" placeholder="请输入组织代码" />
            </a-form-model-item>
            <a-form-model-item label="上级组织" prop="parentId">
              <a-tree-select
                :replaceFields="{
                  title: 'name',
                  value: 'id'
                }"
                v-model="form.parentId"
                style="width: 100%"
                :dropdown-style="{ maxHeight: '400px', overflow: 'auto' }"
                :tree-data="formOrganizationTreeData"
                placeholder="请选择上级组织"
                tree-default-expand-all
              >
              </a-tree-select>
            </a-form-model-item>
            <a-form-model-item ref="sortIndex" label="排序索引" prop="sortIndex">
              <a-input type="number" v-model="form.sortIndex" placeholder="请输入排序索引" />
            </a-form-model-item>
            <a-form-model-item label="备注">
              <a-input v-model.trim="form.description" type="textarea" placeholder="请输入备注" :maxLength="500" />
            </a-form-model-item>
          </a-form-model>
        </a-modal>
      </div>
    </div>
  </page-header-wrapper>
</template>
<script>
import {
  listOrganizations,
  createOrganization,
  updateOrganization,
  deleteOrganizationById,
  loadOrganizationById
} from '@/api/api';
import moment from 'moment';
import organizationTreeData from './components/tree/organizationTree';
const columns = [
  {
    title: '名称',
    dataIndex: 'name',
    width: '20%',
    ellipsis: true
  },
  {
    title: '创建时间',
    dataIndex: 'createTime',
    width: '20%',
    ellipsis: true
  },
  {
    title: '代码',
    dataIndex: 'code',
    width: '20%',
    ellipsis: true
  },
  {
    title: '组织描述',
    dataIndex: 'description',
    width: '20%',
    ellipsis: true
  },
  {
    title: '操作',
    scopedSlots: { customRender: 'action' },
    width: '200px'
  }
];
export default {
  name: 'Organization',
  data() {
    return {
       jumper: '',
      advanced: false, // 控制搜索条件的展开折叠
      formButtonDisableFlag: false, // 表单确定禁用按钮 防止多次点击多次保存
      organizationloading: false, // 加载表格的loading
      searchParameters: {}, // 表格搜索条件值
      modleVisible: false, // 控制弹框
      columns, // 表格头部
      organizationTableData: [], // 表格数据
      pageSizeOptions: this.$store.state.user.defaultPaginationOptions, // 分页下拉
      currentPage: 1, // 默认分页当前页
      pageObject: {
        pageNumber: 0,
        pageSize: this.$store.state.user.defaultPaginationPagesize // 一页展示多少条数据
      },
      organizationTableTotal: 0, // 表格数据总数
      labelCol: { span: 7 },
      wrapperCol: { span: 14 },
      form: {
        // 表单数据
        name: undefined, // 组织名称
        parent: undefined, // 配合后台接口的字段
        parentId: undefined, // 上级组织ID
        code: undefined, // 组织代码
        sortIndex: undefined,
        isEnable: '1', // 状态
        description: undefined // 组织描述
      },
      rules: {
        // 规则验证
        name: [
          { required: true, message: '请输入组织名称', trigger: 'blur' },
          { max: 50, message: '组织名称长度不能大于50', trigger: 'blur' }
        ],
        code: [
          { required: true, message: '请输入组织代码', trigger: 'blur' },
          { max: 50, message: '组织代码长度不能大于50', trigger: 'blur' }
        ]
      },
      formOrganizationTreeData: [] // 表单的树形下拉数据
    };
  },
  components: {
    organizationTreeData
  },
  watch: {
    /**
     * @description: 解决删除分页最后一条没数据的BUG
     * 思路：先获取当前的表格数据总数this.organizationTableTotal
     * 在获取除了当前页数据外的表格总数this.getExceptCurrentPageTableTotalData
     * 如果这两个数相等 说明删除的是当前页最后一条数据 然后使当前页-1 请求数据就可以了
     */
    organizationTableTotal() {
      if (
        this.organizationTableTotal === this.getExceptCurrentPageTableTotalData &&
        this.organizationTableTotal !== 0
      ) {
        this.currentPage -= 1;
        this.pageObject.pageNumber = Number(this.currentPage) - 1;
        this.getOrganizationTableData(this.pageObject, this.searchParameters);
      }
    }
  },
  computed: {
    /**
     * @description: 获取去掉当前页的表格总数
     */
    getExceptCurrentPageTableTotalData: function() {
      return (this.currentPage - 1) * this.pageObject.pageSize;
    }
  },
  created() {
    this.getOrganizationTableData(this.pageObject, this.searchParameters); // 获取表格数据
  },
  methods: {
    /**
     * @description: 获取子组件的组织结构树数据
     * @param {object} organizationData 组织树
     */

    getFormOrganizationTreeData(organizationData) {
      this.formOrganizationTreeData = organizationData;
    },

    /**
     * @description: 获取子组件组织列表选中数据
     * @param {object} selectOrganizationData 部门列表选中对象
     */
    selectOrganization(selectOrganizationData) {
      if (selectOrganizationData.length > 1) {
        this.searchParameters.searchParentIds = selectOrganizationData;
        this.searchParameters.searchParentId = undefined;
      } else {
        this.searchParameters.searchParentIds = undefined;
        this.searchParameters.searchParentId = selectOrganizationData[0];
      }
      this.searchOrganizationTableData();
    },

    /**
     * @description: 取消选中组织树
     */
    cancelSelectOrganizationTreeData() {
      this.searchParameters.searchParentIds = undefined;
      this.searchParameters.searchParentId = undefined;
      this.searchOrganizationTableData();
    },

    /**
     * @description: 获取表格数据
     * @param {object} page 分页参数
     * @param {object} params 搜索参数
     */
    getOrganizationTableData(page, params) {
      this.organizationloading = true;
      listOrganizations(Object.assign({}, page, params))
        .then(res => {
          if (res.code === 200 && res.data.content) {
            this.organizationTableData = res.data.content;
            this.organizationTableData.forEach(item => {
              item.createTime = moment(item.createTime).format('YYYY-MM-DD HH:mm');
            });
            this.organizationTableTotal = res.data.totalElements;
          } else {
            this.organizationTableTotal = res.data.totalElements;
            this.organizationTableData = [];
          }
        })
        .finally(() => {
          this.organizationloading = false;
        });
    },

    /**
     * @description: 表格查询公共的代码
     */
    searchOrganizationTableData() {
      this.currentPage = 1;
      this.pageObject.pageNumber = 0;
      this.getOrganizationTableData(this.pageObject, this.searchParameters);
    },

    /**
     * @description: 新增编辑表单提交
     */
    onSubmit() {
      this.$refs.organizationRuleForm.validate(valid => {
        if (valid) {
          this.formButtonDisableFlag = true;
          if (this.form.parentId) {
              this.form.parent = { id: this.form.parentId };
          }
          if (this.form.id) {
            this.organizationUpdate(this.form);
          } else {
            this.organizationAdd(this.form);
          }
        } else {
          return false;
        }
      });
    },

    /**
     * @description: 新增组织
     * @param {object} organizationAddParam 表单参数
     */
    organizationAdd(organizationAddParam) {
      createOrganization({ body: organizationAddParam })
        .then(res => {
          if (res.code === 201) {
            this.formSuccessOperation(res);
          }
        })
        .finally(() => {
          this.formButtonDisableFlag = false;
        });
    },

    /**
     * @description: 编辑组织
     * @param {object} organizationEditParam 表单参数
     */
    organizationUpdate(organizationEditParam) {
      updateOrganization({ body: organizationEditParam, id: organizationEditParam.id })
        .then(res => {
          if (res.code === 200) {
            this.formSuccessOperation(res);
          }
        })
        .finally(() => {
          this.formButtonDisableFlag = false;
        });
    },

    /**
     * @description: 表单新增编辑成功后的公共的代码 （消息提示 搜索框重置 请求表格数据）
     * @param {object} formSuccessData 表单请求成功后返回的对象
     */
    formSuccessOperation(formSuccessData) {
      this.$message.success(formSuccessData.message);
      this.modleVisible = false;
      this.formButtonDisableFlag = false;
      this.clearFormData();
      this.getOrganizationTableData(this.pageObject, this.searchParameters);
      this.$refs.organizationTree.getTreeData();
    },

    /**
     * @description: 重置表单
     */
    clearFormData() {
      this.$refs.organizationRuleForm.resetFields();
      this.form = this.$options.data.call(this).form;
      this.$refs.organizationTree.getTreeData();
    },

    /**
     * @description: 编辑组织按钮
     * @param {object} organizationTableRowData 表格某一行的数据对象
     */
    handleEdit(organizationTableRowData) {
      loadOrganizationById({ id: organizationTableRowData.id }).then(res => {
        if (res.code === 200) {
          this.form = Object.assign({}, this.form, res.data);
          this.form.parentId = this.form.parent ? this.form.parent.id : undefined;
          this.form.isEnable = String(this.form.isEnable);
          this.disableSelectIdData(organizationTableRowData.id, this.formOrganizationTreeData);
          this.modleVisible = true;
        }
      });
    },

    /**
     * @description: 编辑用户时禁用自己
     * @param {string} selectId 选中ID
     * @param {array} formOrganizationTreeData 组织下拉树
     */
    disableSelectIdData(selectId, formOrganizationTreeData) {
      formOrganizationTreeData.forEach(item => {
        if (item.id === selectId) {
           item.disabled = true;
        }
        if (item.children) {
          this.disableSelectIdData(selectId, item.children);
        }
      });
    },

    /**
     * @description: 点击删除
     * @param {object} organizationTableRowData 表格某一行的数据对象
     */
    handleDelete(organizationTableRowData) {
      deleteOrganizationById({ id: organizationTableRowData.id }).then(res => {
        if (res.code === 200) {
          this.$message.success(res.message);
          this.getOrganizationTableData(this.pageObject, this.searchParameters);
          this.$refs.organizationTree.getTreeData();
        }
      });
    },

    /**
     * @description:  获取分页下拉第几页展示几个
     * @param {string} current 当前页
     * @param {string} pageSize 当前页展示几条
     */
    onPageSizeChange(current, pageSize) {
      this.currentPage = current;
      this.pageObject.pageSize = pageSize;
      this.pageObject.pageNumber = Number(this.currentPage) - 1;
      this.getOrganizationTableData(this.pageObject, this.searchParameters);
    },
        /**
     * @description: 分页跳转输入框改变
     */
    blurJumperInput() {
      if (this.jumper !== '') {
        this.currentPage = Number(this.jumper);
      this.pageObject.pageNumber = Number(this.currentPage) - 1;
      this.getOrganizationTableData(this.pageObject, this.searchParameters);
      }
    },

    /**
     * @description: 获取分页页数改变后的值
     * @param {string} pageNumber UI框架自带
     */
    handlePageNumberChange(pageNumber) {
      this.jumper = '';
      this.currentPage = pageNumber;
      this.pageObject.pageNumber = Number(this.currentPage) - 1;
      this.getOrganizationTableData(this.pageObject, this.searchParameters);
    },
    /**
     * @description: 日期选择器改变
     * @param {array} date UI框架自带
     * @param {array} dateString UI框架自带 时间区间
     */
    onChangeData(date, dateString) {
      this.searchParameters.searchCreateDateBegin = dateString[0];
      this.searchParameters.searchCreateDateEnd = dateString[1];
    },

    /**
     * @description: 新增组织
     */

    handleAddOrganization() {
      this.form.id = undefined;
      this.modleVisible = true;
      this.form.parentId = this.searchParameters.searchParentIds
        ? this.searchParameters.searchParentIds[0]
        : this.searchParameters.searchParentId
        ? this.searchParameters.searchParentId
        : undefined;
    },

    /**
     * @description: 重置搜索条件
     */
    handleReset() {
      const copySearchParameters = JSON.parse(JSON.stringify(this.searchParameters));
      this.searchParameters = {};
      this.searchParameters.searchParentIds = copySearchParameters.searchParentIds;
      this.searchParameters.searchParentId = copySearchParameters.searchParentId;
      this.searchOrganizationTableData();
    }
  }
};
</script>
<style lang="less" scoped>
.deactivate {
  color: red;
}
.enable {
  color: green;
}
.table-page-search-wrapper /deep/ .ant-form-inline .ant-form-item > .ant-form-item-label {
  line-height: 32px;
  padding-right: 8px;
  width: 77px;
}
.organization {
  width: 100%;
  border-radius: 5px;
  display: flex;
  justify-content: space-between;
  .organizationList {
    width: 330px;
    background: white;
    border-radius: 5px;
  }
  .organizationMain {
    width: calc(100% - 340px);
    height: 100%;
    position: relative;
    .position {
      width: 100%;
      height: 100%;
      background: white;
      position: absolute;
      display: flex;
      align-items: center;
      justify-content: center;
      z-index: 999;
    }
    .organizationSearch {
      width: 100%;
      padding: 20px;
      background: white;
      margin-bottom: 10px;
    }
    .organizationTable {
      background: white;
      display: flex;
      flex-direction: column;
      .organizationTableAdd {
        width: 100%;
        height: 64px;
        display: flex;
        align-items: center;
        border-bottom: 1px solid #ececec;
        font-size: 18px;
        justify-content: space-between;
        padding: 0 20px;
        font-weight: 600;
      }
      .organizationTableContent {
        height: calc(100vh - 380px);
        padding: 10px;
        overflow: scroll;
        scrollbar-width: none;//兼容火狐
      }
      .organizationTableContent /deep/ .ant-table-tbody .ant-table-row:nth-child(2n) {
        background: #fafafa;
      }
      .organizationTableContent::-webkit-scrollbar {
        display: none;
      }
    }
    .organizationPagination {
      width: 100%;
      height: 47px;
      border-radius: 5px;
      background: white;
      margin-top: 10px;
      display: flex;
      align-items: center;
      justify-content: flex-end;
      padding: 0 10px 0 0;
    }
  }
}
</style>
