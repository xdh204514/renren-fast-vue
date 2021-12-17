<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
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