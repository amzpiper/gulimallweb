<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            type="text"
            size="mini"
            v-if="node.level <= 2"
            @click="() => append(data)"
          >
            添加
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            修改
          </el-button>
          <el-button
            type="text"
            size="mini"
            v-if="node.childNodes.length == 0"
            @click="() => remove(node, data)"
          >
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <!-- 添加分类对话框 -->
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :before-close="handleClose"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">
          确 定
        </el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  components: {},
  props: {},
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [],
      title: "提示",
      dialogVisible: false,
      dialogType: "", //eidt , add
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        productUnit: "",
        icon: "",
      },
    };
  },
  computed: {},
  watch: {},
  methods: {
    // 点击分类
    handleNodeClick(data) {
      console.log(data);
    },
    // 获取菜单列表
    getMenus() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据：", data.data);
        this.menus = data.data;
      });
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //修改分类
    edit(data) {
      console.log("edit", data);
      //服用原添加分类对话框
      this.dialogType = "edit";
      this.title = "修改分类";
      this.dialogVisible = true;
      //回显
      this.category.catId = data.catId;
      this.category.name = data.name;
      //回显应该是发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        //回显
        console.log("要回显的数据", data);
        this.category.catId = data.data.catId;
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
      console.log("category", this.category);
    },
    //发起修改分类
    editCategory() {
      //后端中接收的字段为null时不会修改该字段
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
      });
      //关闭对话框
      this.dialogVisible = false;
      //更新添加后的分类列表
      this.getMenus();
      //设置默认展开的菜单
      this.expandedKey = [this.category.parentCid];
    },
    //添加分类
    append(data) {
      console.log("append", data);
      this.dialogType = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      //赋值到分类对象
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.parentCid = data.catId;
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;

      console.log("category", this.category);
    },
    //添加三级分类
    addCategory() {
      console.log("添加三级分类", this.category);
      console.log("添加三级分类2", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
      });
      //关闭对话框
      this.dialogVisible = false;
      //更新添加后的分类列表
      this.getMenus();
      //设置默认展开的菜单
      this.expandedKey = [this.category.parentCid];
    },
    //关闭添加三级分类对话框
    handleClose(done) {
      this.$confirm("确认关闭？")
        .then((_) => {
          done();
        })
        .catch((_) => {});
    },
    //删除分类
    remove(node, data) {
      console.log("remove", node, data);
      var ids = [data.catId];
      this.$confirm(`确定删除"${data.name}"操作?`, "提示", {
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
                onClose: () => {},
              });
              //更新删除后的分类列表
              this.getMenus();
              //设置默认展开的菜单
              this.expandedKey = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "取消删除",
          });
        });
    },
  },
  created() {
    this.getMenus();
  },
  mounted() {},
  beforeCreate() {},
  beforeMount() {},
  beforeUpdate() {},
  updated() {},
  beforeDestroy() {},
  destroyed() {},
  activated() {
    // 如果页面有keep-alive缓存功能，这个函数会触发
  },
};
</script>
<style lang=""></style>
