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
      draggable
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
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
      expandedKey: [],//默认展示的节点
      maxLevel:0, //计算最大的层级
      updateNodes:[],//修改层级时保存所有重新排序的信息
      title: "提示",//增加修改分类提示框的标题
      dialogVisible: false,
      dialogType: "", //eidt , add
      category: { //分类model
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
    //计算拖拽节点修改层级
    allowDrop(draggingNode, dropNode, type) {
      //draggingNode：当前节点，dropNode：目标节点，type：那个位置
      //1.判断规则-放到里面：当前节点总层数+父节点及节点总层数<=3
      //2.放到后边：当前节点总层数+父节点及以上的层数<=3
      console.log("allowDrop:",draggingNode, dropNode, type)

      //1)被拖到的当前节点总层数
      this.countNodeLevel(draggingNode.data);
      console.log("正在拖到的节点深度",this.maxLevel)
      //2)当前深度
      let deep = (this.maxLevel - draggingNode.data.catLevel) + 1
      console.log("深度",deep)
      //3）当前正在拖动的节点+目标节点所在的深度不大于3即可
      if(type == "inner"){
        console.log("拖拽后深度",deep +dropNode.level)
        return deep +dropNode.level <=3;
      }else{
        console.log("拖拽后深度",deep +dropNode.parent.level)
        return deep +dropNode.parent.level <=3;
      }
    },
    //计算当前节点总层数
    countNodeLevel(data){
      //找到最深的子节点层级
      if(data.children != null && data.children.length > 0){
        for(let i=0;i<data.children.length;i++){
          if(data.children[i].catLevel > this.maxLevel){
            this.maxLevel = data.children[i].catLevel
          }
          this.countNodeLevel(data.children[i]);
        }
      }
    },
    //拖拽成功后
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType, ev);
      //1.当前节点最新父id
      // dropNode的父节点中含有拖拽后的所有子节点数据，需要把父节点的所有子节点排个序
      let pCid = 0;
      let siblings = null;//保存所有兄弟节点
      if(dropType == "before" ||dropType == "after"){
        pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.childNodes;
        // 2.情况：如果不是inner，那么dropNode的所有子节点就是他的所有兄弟节点
        siblings = dropNode.parent.childNodes; 
        console.log("非inner-兄弟节点 ",siblings)
      }else{
        dropNode.data.catId;
        // 2.情况：如果是inner，那么dropNode的所有子节点就是他的所有兄弟节点
        siblings = dropNode.childNodes;
        console.log("inner-兄弟节点 ",siblings)
      }

      //2.当前拖拽的最新顺序,在dropNode的dropType位置-应该时拖拽后的节点的所在的父节点的子节点拿出来排个序
      //遍历所有兄弟节点重新排序
      for(let i=0;i<siblings.length;i++){
        //updateNodes
        //修改所有兄弟的sort顺序和父子关系的，需要判断一下拖拽的节点的id修改一下父节点
        if(siblings[i].data.catId == draggingNode.data.catId){
          //如果遍历的是当前拖拽的节点,还要修改父id
          //并且层级发生了变化,当前节点和拖拽的节点的层级不一样
          let catLevel = draggingNode.data.catLevel;
          if(siblings[i].data.catLevel != draggingNode.data.catLevel){
            if(dropType =="before" ||dropType == "after"){
              catLevel = draggingNode.data.catLevel;
            }else{
              catLevel = draggingNode.data.catLevel;
            }
          }
          this.updateNodes.push({catId :siblings[i].data.catId,sort:i,parentCid:pCid,catLevel:catLevel})
        }else{
          //其他兄弟元素只改顺序，不修改父id
          this.updateNodes.push({catId :siblings[i].data.catId,sort:i})
        }
      }

      //3.当前拖拽节点的最新层级
      // dropNode.parent.childNodes[0].level

      console.log("updateNodes",this.updateNodes)
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
        //更新添加后的分类列表
        this.getMenus();
        //设置默认展开的菜单
        this.expandedKey = [this.category.parentCid];
      });
      //关闭对话框
      this.dialogVisible = false;
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
