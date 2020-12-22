<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      node-key="catId"
      ref="menusTree"
      @node-click="nodeClick"
    >
    </el-tree>
  </div>
</template>

<script>
export default {
  name: "category",
  components: {},
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  mounted() {},
  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/list/tree"),
        method: "get",
      }).then(({ data }) => {
        //用({ data })把返回数据中data部分的数据剥离出来
        this.menus = data.pmsCategoryEntities;
      });
    },
    //参数依次为data属性的数组中节点对应的对象，节点对应的node，节点组件本身
    nodeClick(data, node, component) {
      this.$emit("treeNode-click", data, node, component);
    },
  },
  created() {
    this.getMenus();
  },
};
</script>

<style scoped>
</style>