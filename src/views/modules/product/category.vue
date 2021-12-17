<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
      style="padding: 0px 30px 0 10px"
    >
    </el-switch>
    <el-button type="primary" @click="batchDrag()">批量拖拽</el-button>
    <el-button type="danger" @click="batchDelete()">批量删除</el-button>
    <div>&#12288;</div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <!-- 一级分类和二级分类才可以添加子分类 -->
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            添加分类
          </el-button>
          <!-- 只有没有子分类的情况才可以删除该分类 -->
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除分类
          </el-button>

          <el-button type="text" size="mini" @click="() => edit(node, data)">
            修改分类
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogFormVisible"
      :close-on-click-modal="false"
      width="600px"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData()">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import Vue from "vue";
// 这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
// 例如：import 《组件名称》 from '《组件路径》';

export default {
  // import引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      draggingNodeMaxLevel: 0,
      title: "",
      dialogType: "",
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [2],
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 0,
        sort: 0,
        productCount: 0,
        icon: "",
        productUnit: "",
        catId: null,
      },
      dialogFormVisible: false,
    };
  },
  // 计算属性 类似于data概念
  computed: {},
  // 监控data中的数据变化
  watch: {},
  // 方法集合
  methods: {
    // 批量删除
    batchDelete() {
      // 1. 获取所有被选中的节点
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();

      // 2. 将选中的节点的 catId 取出
      let checkedNodeKeys = [];
      let checkedNodeNames = [];
      for (let i = 0; i < checkedNodes.length; i++) {
        checkedNodeKeys.push(checkedNodes[i].catId);
        checkedNodeNames.push(checkedNodes[i].name);
      }

      // 3. 发送请求进行删除
      this.$confirm(`确定删除【${checkedNodeNames}】菜单项?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(checkedNodeKeys, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
                duration: 1500,
                onClose: () => {
                  // 1. 获取新的分类数据
                  this.getTreeMenuData();
                },
              });
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {});
    },

    // 批量拖拽
    batchDrag() {
      this.$http({
        url: this.$http.adornUrl("/product/category/batch/update"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "操作成功",
            type: "success",
            duration: 1500,
            onClose: () => {
              // 1. 获取新的分类数据
              this.getTreeMenuData();
              // 2. 默认展开之前删除分类的父分类
              this.expandedKey = this.pCid;
              // 4. 清空拖拽形成的数据
              this.draggingNodeMaxLevel = 0;
              this.updateNodes = [];
              this.pCid = [];
            },
          });
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    // 拖拽成功完成时触发的事件
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log(draggingNode, dropNode, dropType);
      // 1. 获取当前节点拖动后的父节点id 和 拖动后的兄弟节点数组
      let pCid = 0;
      let slibings = [];
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        slibings = dropNode.childNodes;
      } else {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId; // 如果是将当前节点拖动到根节点位置，那么就没有父节点 id，需要置为 0
        slibings = dropNode.parent.childNodes;
      }
      this.pCid.push(pCid);

      // 2. 获取当前节点拖动后的顺序和层级（即在拖动后的父节点的 childNodes 中的序号）
      for (let i = 0; i < slibings.length; i++) {
        let updateNode = {};
        if (slibings[i].data.catId == draggingNode.data.catId) {
          // 如果是遍历到当前节点，则需要通过的当前节点的 catId 来更新 parentCid、catLevel 和 sort 字段
          let catLevel = draggingNode.level; // 拖动前的层级
          if (slibings[i].level != draggingNode.level) {
            console.log("拖动的节点的层级发生了变化...");
            // 如果当前节点的层级在拖动前后发生了变化，则需要更新当前节点的层级
            catLevel = slibings[i].level;
            // 以及更新当前节点的子节点层级
            this.updateChildNodesLevel(slibings[i]);
          }
          updateNode = {
            catId: slibings[i].data.catId,
            parentCid: pCid,
            catLevel: catLevel,
            sort: i,
          };
        } else {
          // 如果是遍历到当前节点拖动后的兄弟节点，则需要通过的兄弟节点的 catId 来更新 sort 字段
          updateNode = { catId: slibings[i].data.catId, sort: i };
        }
        this.updateNodes.push(updateNode);
      }

      console.log("所有要修改的节点", this.updateNodes);
    },

    // 更新 node 节点的子节点的层级
    updateChildNodesLevel(node) {
      if (node.childNodes != null && node.childNodes.length != 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let childNode = node.childNodes[i].data;
          // 需要也将这些节点加入到 updateNodes 中更新到数据库
          this.updateNodes.push({
            catId: childNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodesLevel(node.childNodes[i]);
        }
      }
    },

    // 拖拽时判定目标节点能否被放置
    allowDrop(draggingNode, dropNode, type) {
      // 核心判断条件：被拖动的当前节点的总层数 + 该节点拖动后所处位置的父节点的层数 <= 3 才可以允许拖动
      // console.log(draggingNode, dropNode, type);

      // 1. 计算当前拖动节点的子节点的最大层级是多少
      this.draggingNodeMaxLevel = 0;
      this.countDraggingNodeTotalLevel(draggingNode);

      // 2. 计算当前拖动节点及其子节点的层数是多少（即当前节点为根节点形成的树有多少层）
      // 如果 draggingNodeMaxLevel = 0，表示没有子节点，那么其本身层级数即为最大层级数
      this.draggingNodeMaxLevel =
        this.draggingNodeMaxLevel == 0
          ? draggingNode.level
          : this.draggingNodeMaxLevel;
      console.log("当前节点的最大层级数", this.draggingNodeMaxLevel);
      console.log("当前节点本身的层级数", draggingNode.level);
      let deep = Math.abs(this.draggingNodeMaxLevel - draggingNode.level) + 1;
      console.log("以当前节点为根节点形成的树有多少层", deep);

      // 3. 计算
      if (type == "inner") {
        // 如果是 inner，则表示 dropNode 就是拖动后的父节点
        return deep + dropNode.level <= 3;
      } else {
        // 如果是 next 或者是 prev 则表示 dropNode.parent 才是拖动后的父节点
        return deep + dropNode.parent.level <= 3;
      }
    },

    // 计算当前拖动节点的子节点的最大层级是多少
    countDraggingNodeTotalLevel(node) {
      // 使用 node.childNodes 中每一个节点的 level 值去判断，这样能够保证得到的是最新的层级
      if (node.childNodes != null && node.childNodes.length != 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          this.draggingNodeMaxLevel =
            node.childNodes[i].level > this.draggingNodeMaxLevel
              ? node.childNodes[i].level
              : this.draggingNodeMaxLevel;
          this.countDraggingNodeTotalLevel(node.childNodes[i]);
        }
      }
    },

    // 重置 category 对象
    resetCategory(
      parentCid,
      catLevel,
      showStatus,
      sort,
      productCount,
      icon,
      productUnit
    ) {
      this.category = {
        parentCid,
        catLevel,
        showStatus,
        sort,
        productCount,
        icon,
        productUnit,
      };
    },

    // 提交分类数据：1、添加分类，2、修改分类
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },

    // 修改分类
    edit(node, data) {
      console.log("data", data);
      // 1. 修改 dialogType 为 add
      this.dialogType = "edit";

      // 2. 修改 title 为 添加菜单
      this.title = "修改菜单";

      // 3. 显示表单
      this.dialogFormVisible = true;

      // 4. 从数据库获取数据，进行回显数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${node.data.catId}`),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        console.log("data.category", data.category);
        this.category.catId = data.category.catId;
        this.category.name = data.category.name;
        this.category.icon = data.category.icon;
        this.category.productUnit = data.category.productUnit;
        this.category.parentCid = data.category.parentCid;
      });
    },

    // 发送修改菜单请求
    editCategory() {
      // 选择性发送，那么数据库就会选择性更新，不发送的不会发生改变
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "操作成功",
            type: "success",
            duration: 1500,
            onClose: () => {
              // 1. 关闭表单
              this.dialogFormVisible = false;
              // 2. 获取新的分类数据
              this.getTreeMenuData();
              // 3. 默认展开之前删除分类的父分类
              this.expandedKey = [this.category.parentCid];
              console.log(this.expandedKey, "edit");
              // 4. 清空 category 对象
              this.resetCategory(0, 0, 0, 0, 0, "", "");
            },
          });
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    // 添加菜单
    append(data) {
      // 1. 修改 dialogType 为 add
      this.dialogType = "add";
      // 2. 修改 title 为 添加菜单
      this.title = "添加菜单";
      // 3. 显示表单
      this.dialogFormVisible = true;
      // 4. 封装数据
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.showStatus = 1;
      this.category.sort = 0;
      this.category.productCount = 0;
    },

    // 发送添加菜单请求
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "操作成功",
            type: "success",
            duration: 1500,
            onClose: () => {
              // 1. 关闭表单
              this.dialogFormVisible = false;
              // 2. 获取新的分类数据
              this.getTreeMenuData();
              // 3. 默认展开之前删除分类的父分类
              this.expandedKey = [this.category.parentCid];
              console.log(this.expandedKey, "add");
              // 4. 清空 category 对象
              this.resetCategory(0, 0, 0, 0, 0, "", "");
            },
          });
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    // 删除菜单
    remove(node, data) {
      const ids = [node.data.catId];
      this.$confirm(`确定删除【${node.data.name}】菜单项?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
                duration: 1500,
                onClose: () => {
                  // 1. 获取新的分类数据
                  this.getTreeMenuData();
                  // 2. 默认展开之前删除分类的父分类
                  this.expandedKey = [node.data.parentCid];
                },
              });
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {});
    },

    // 获取分类数据
    getTreeMenuData() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menus = data.menus;
      });
    },
  },
  // 生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getTreeMenuData();
    this.expandedKey = [1, 2];
  },
  // 生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, // 生命周期 - 创建之前
  beforeMount() {}, // 生命周期 - 挂载之前
  beforeUpdate() {}, // 生命周期 - 更新之前
  updated() {}, // 生命周期 - 更新之后
  beforeDestroy() {}, // 生命周期 - 销毁之前
  destroyed() {}, // 生命周期 - 销毁完成
  activated() {}, // 如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>