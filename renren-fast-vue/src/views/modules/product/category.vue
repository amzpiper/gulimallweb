<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" @node-click="handleNodeClick" :expand-on-click-node="false"
      show-checkbox node-key="catId" :default-expanded-keys="expandedKey">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button type="text" size="mini" v-if="node.level <= 2" @click="() => append(data)">
            Append
          </el-button>
          <el-button type="text" size="mini" v-if="node.childNodes.length == 0" @click="() => remove(node, data)">
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
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
        expandedKey: []
      };
    },
    computed: {},
    watch: {},
    methods: {
      handleNodeClick(data) {
        console.log(data);
      },
      // 获取菜单列表
      getMenus() {
        this.dataListLoading = true;
        this.$http({
          url: this.$http.adornUrl("/product/category/list/tree"),
          method: "get",
        }).then(({
          data
        }) => {
          console.log("成功获取到菜单数据：", data.data);
          this.menus = data.data
        });
      },
      //添加分类
      append(data) {
        console.log("append", data)
      },
      //删除分类
      remove(node, data) {
        console.log("remove", node, data)
        var ids = [data.catId]
        this.$confirm(`确定删除"${data.name}"操作?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            if (data && data.code === 0) {
              this.$message({
                message: '操作成功',
                type: 'success',
                duration: 1500,
                onClose: () => {
                  }
              })
              //更新删除后的分类列表
              this.getMenus()
              //设置默认展开的菜单
              this.expandedKey = [node.parent.data.catId]
            } else {
              this.$message.error(data.msg)
            }
          })
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '取消删除'
          }); 
        })
      },
    },
    created() {
      this.getMenus()
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
    }
  }

</script>
<style lang=""></style>
