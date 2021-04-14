<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      @node-click="handleNodeClick"
    ></el-tree>
  </div>
</template>

<script>
export default {
  components: {},
  props: {},
  data() {
    return {
      menus: [
      ],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  computed: {},
  watch: {},
  methods: {
    handleNodeClick (data) {
      console.log(data);
    },
    // 获取菜单列表
    getMenus () {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据：",data.data);
        this.menus=data.data
      });
    }
  },
  created () {
    this.getMenus ()
  },
  mounted () {},
  beforeCreate () {},
  beforeMount () {},
  beforeUpdate () {},
  updated () {},
  beforeDestroy () {},
  destroyed () {},
  activated () {
    // 如果页面有keep-alive缓存功能，这个函数会触发
  }
}
</script>
<style lang=""></style>
