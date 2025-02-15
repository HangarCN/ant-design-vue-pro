<template>
  <div class="organizationTreeMain">
    <div class="organizationTreeSearch">
      <span><a-input placeholder="搜索组织" style="width: 275px" @change="handleSearch" v-model="organizationSearch"/></span>
      <span>
        <a-popover placement="bottomRight">
          <template slot="content">
            <p style="cursor: pointer;" @click="() => (this.expandedKeys = this.allIds)">展开全部</p>
            <span style="cursor: pointer;" @click="() => (this.expandedKeys = [])">折叠全部</span>
          </template>
          <a-icon type="menu-unfold" /> </a-popover
        ></span>
    </div>
    <div class="organizationTreeContent">
      <a-tree
        v-if="isOrganizationTreeShow"
        ref="tree"
        :replaceFields="{
          title: 'name',
          key: 'id'
        }"
        defaultExpandAll
        :tree-data="treeData"
        :expandedKeys.sync="expandedKeys"
        :defaultSelectedKeys.sync="defaultSelectedId"
        @select="handleSelect"
      >
        <template slot="title" slot-scope="{ name }">
          <span
            :title="name"
            v-html="
              name.replace(
                new RegExp(organizationSearch, 'g'),
                '<span style=color:#f50>' + organizationSearch + '</span>'
              )
            "
          ></span>
        </template>
      </a-tree>
    </div>
  </div>
</template>

<script>
import { organizationsTree } from '@/api/api';
export default {
  name: 'OrganizationTree',
  data() {
    return {
      isOrganizationTreeShow: true, // 用作取消选中重新加载组件 要不然默认效果不生效
      defaultSelectedId: [], // 默认选中第一条的ID
      expandedKeys: [], // 控制树结构展开折叠数据组
      allIds: [], // 用来保存树形所有节点ID
      organizationSearch: '', // 组织列表搜索
      treeData: [], // 组织树形结构数据
      selectTreeDataAllIds: [] // 获取选中的组织机构ID包含的所有ID
    };
  },
  created() {
    this.getTreeData();
  },
  methods: {
    /**
     * @description: 获取组织列表树形结构数据
     */
    getTreeData() {
      organizationsTree().then(res => {
        if (res.code === 200) {
          this.treeData = res.data;
          this.$set(this.defaultSelectedId, 0, this.treeData[0].id);
          this.initTreeData(this.treeData);
          this.$emit('organizationTreeData', this.treeData);
        }
      });
    },
    /**
     * @description: 利用递归获取到所有的节点code 并添加scopedSlots 用来搜索变色 获取所有节点的ID并备份 用来做树形结构的默认展开
     * @param {array} treeData 组织列表数据
     */

    initTreeData(treeData) {
      treeData.forEach(item => {
        item.scopedSlots = { title: 'title' };
        this.expandedKeys.push(item.id);
        item.disabled = item.isEnable === 0;
        if (item.children) {
          this.initTreeData(item.children);
        }
      });
      this.allIds = this.expandedKeys;
    },

    /**
     * @description: 部门列表搜索
     */

    handleSearch() {
      // 获取符合条件的ID值
      this.expandedKeys = this.getTreeIDList(this.organizationSearch, this.treeData, []);
      // 获取符合条件的ID值得父级ID
      this.expandedKeys.forEach(item => {
        this.getParentKey(item, this.treeData);
      });
    },
    /**
     * @description:  获取符合条件的ID值
     * @param {string} searchValue 搜索框的值
     * @param {array} treeData 组织树数据
     * @param {array} idList 空数组 用来做返回值
     * @return {array} idList 命中节点的ID数组
     */

    getTreeIDList(searchValue, treeData, idList) {
      // 遍历所有同一级的树
      for (let i = 0; i < treeData.length; i++) {
        const node = treeData[i];
        // 如果该节点存在value值则push
        if (node.name.indexOf(searchValue) !== -1) {
          idList.push(node.id);
        }
        // 如果拥有孩子继续遍历
        if (node.children) {
          this.getTreeIDList(searchValue, node.children, idList);
        }
      }
      return idList;
    },

    /**
     * @description:  获取符合条件的ID值得父级ID
     * @param {string} id 选中的id
     * @param {array} treeData 组织树数组
     * @return {parentId} 选中id的父级id
     */

    getParentKey(id, treeData) {
      let parentId = null;
      for (let i = 0; i < treeData.length; i++) {
        const node = treeData[i];
        if (node.id === id) {
          // 判断该节点是否有父级
          if (node.parent) {
            parentId = node.parent.id;
            // 数组去重添加
            if (this.expandedKeys.indexOf(parentId) === -1) {
              this.expandedKeys.push(parentId);
              // 如果还有父级则拿父级ID重新遍历
              this.getParentKey(node.parent.id, this.treeData);
            }
          }
        }
        // 如果拥有孩子继续遍历
        if (node.children) {
          this.getParentKey(id, node.children);
        }
      }
      return parentId;
    },

    /**
     * @description: 选择组织列表数据
     * @param {array} selectedKeys 选中的组织id数组
     */

    handleSelect(selectedKeys, rowData) {
      this.selectTreeDataAllIds = [];
      if (selectedKeys.length === 0) {
        this.isOrganizationTreeShow = false;
        this.$nextTick(() => {
          this.$set(this.defaultSelectedId, 0, this.treeData[0].id);
          this.isOrganizationTreeShow = true;
        });
        this.$emit('cancelSelect');
        return false;
      }
      if (rowData.node.dataRef.children && rowData.node.dataRef.children.length !== 0) {
        this.getSelectMixTreeAllIds(rowData.node.dataRef.children);
      }
      this.selectTreeDataAllIds.unshift(selectedKeys[0]);
      this.$emit('selectOrganization', this.selectTreeDataAllIds);
    },
    /**
     * @description: 获取选中的组织ID以及下属所有ID
     * @param {array} treeData 当前组织树
     */

    getSelectMixTreeAllIds(treeData) {
      treeData.forEach(item => {
        this.selectTreeDataAllIds.push(item.id);
        if (item.children) {
          this.getSelectMixTreeAllIds(item.children);
        }
      });
    }
  }
};
</script>

<style lang="less" scoped>
.organizationTreeMain {
  width: 100%;
  display: flex;
  flex-direction: column;
    height: calc(100vh - 167px);
  .organizationTreeSearch {
    width: 100%;
    height: 46px;
    border-bottom: 1px solid #ececec;
    padding: 5px;
    display: flex;
    align-items: center;
    justify-content: space-around;
    font-size: 18px;
    cursor: pointer;
  }

  .organizationTreeContent {
    flex: 1;
    overflow: scroll;
    padding: 10px;
    scrollbar-width: none;//兼容火狐
  }
  .organizationTreeContent::-webkit-scrollbar {
    display: none;
  }
    .organizationTreeContent /deep/ .ant-tree li .ant-tree-node-content-wrapper {
    max-width: 240px;
    display: inline-block;
    height: 24px;
    margin: 0;
    padding: 0 5px;
    color: rgba(0, 0, 0, 0.65);
    line-height: 24px;
    text-decoration: none;
    vertical-align: top;
    border-radius: 2px;
    cursor: pointer;
    -webkit-transition: all 0.3s;
    transition: all 0.3s;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
}
</style>
