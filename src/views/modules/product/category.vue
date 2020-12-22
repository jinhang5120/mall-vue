<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="拖拽开启"
      inactive-text="拖拽关闭"
    >
    </el-switch>
    <el-button v-if="draggable" size="mini" @click="batchSave">
      批量保存
    </el-button>
    <el-button type="danger" size="mini" @click="batchDelete">
      批量删除
    </el-button>
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
      ref="menusTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="edit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
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
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      deep: 0,
      title: "",
      dialogType: "",
      /* 封装后台的实体类 */
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      expandedKey: [],
    };
  },
  methods: {
    submitData() {
      if (this.dialogType == "append") {
        this.addCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/update"),
        method: "post",
        data: this.$http.adornData(
          {
            catId,
            name,
            icon,
            productUnit,
          } /*只发送部分数据给后端，修改部分数据*/,
          false
        ),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    //添加三级分类的方法
    addCategory() {
      console.log("添加：", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/save"),
        method: "post",
        data: this.$http.adornData(this.category, false), //请求体数据
      }).then(({ data }) => {
        this.$message({
          message: "菜单添加成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/list/tree"),
        method: "get",
      }).then(({ data }) => {
        //用({ data })把返回数据中data部分的数据剥离出来
        this.menus = data.pmsCategoryEntities;
      });
    },
    append(data) {
      this.title = "添加分类";
      this.dialogType = "append";
      console.log(data);
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1; //担心catlevel是字符串，所以先乘再加
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
    },
    edit(data) {
      this.title = "修改分类";
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/pmscategory/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("edit:", data);
        //回显数据
        this.category.name = data.pmsCategory.name;
        this.category.catId = data.pmsCategory.catId;
        this.category.icon = data.pmsCategory.icon;
        this.category.productUnit = data.pmsCategory.productUnit;
        this.category.parentCid = data.pmsCategory.parentCid;
      });
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/pmscategory/delete"),
            method: "post",
            data: this.$http.adornData(ids, false), //请求体数据
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            this.getMenus();
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
    },
    allowDrop(draggingNode, dropNode, type) {
      this.countChildMaxlevel(draggingNode);
      //draggingNodeChildDeep是被拖拽菜单有几层（包括自己）
      let draggingNodeChildDeep = this.maxLevel - draggingNode.level + 1;
      //每次拖拽判断都不一样，因此全局变量要及时置空
      this.maxLevel = 0;
      if (type == "inner") {
        return draggingNodeChildDeep + dropNode.level <= this.deep;
      } else {
        return draggingNodeChildDeep + dropNode.level <= this.deep + 1;
      }
    },
    //求出该节点最大的子节点深度
    countChildMaxlevel(Node) {
      if (Node.childNodes != null && Node.childNodes.length > 0) {
        for (let i = 0; i < Node.childNodes.length; i++) {
          if (this.maxLevel < Node.childNodes[i].level) {
            this.maxLevel = Node.childNodes[i].level;
          }
          this.countChildMaxlevel(Node.childNodes[i]);
        }
      } else {
        this.maxLevel = Node.level;
      }
    },
    countMenusDeep(menus) {
      if (menus.children != null && menus.children.length > 0) {
        for (let i = 0; i < menus.children.length; i++) {
          if (this.deep < menus.children[i].catLevel) {
            this.deep = menus.children[i].catLevel;
          }
          //递归嵌套查询
          this.countMenusDeep(menus.children[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let pCid = 0;
      let siblings = null;
      //1.找到拖拽后的父节点pCid和兄弟节点
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }
      this.pCid.push(pCid);
      //2.遍历兄弟节点，改被拖拽节点中的父节点信息以及子节点信息，改其他节点的sort信息
      for (let i = 0; i < siblings.length; i++) {
        //用catId判断哪一个是被拖拽的
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果拖拽的节点的层级发生变化
          if (siblings[i].level != draggingNode.level) {
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: siblings[i].level,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
    },
    updateChildNodeLevel(node) {
      //保证有子节点
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          this.updateNodes.push({
            catId: node.childNodes[i].data.catId,
            sort: i,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    batchSave() {
      console.log("this.pCid", this.pCid);
      //发送请求后端进行数据库更改
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false), //请求体数据
      }).then(({ data }) => {
        this.$message({
          message: "菜单拖拽修改成功",
          type: "success",
        });
        this.getMenus();
        this.expandedKey = this.pCid;
        //别忘了及时置空全局的信息，否则只会越拖拽越多
        this.updateNodes = [];
        this.pCid = []; //方法里将data里的全局变量重置回默认值是良好的习惯
      });
    },
    batchDelete() {
      let checkedNodes = this.$refs.menusTree.getCheckedNodes();
      console.log("checkedNodes:", checkedNodes);
      let checkedNodesCatIds = [];
      for (let i = 0; i < checkedNodes.length; i++) {
        checkedNodesCatIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`是否删除【${checkedNodesCatIds}】？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/pmscategory/delete"),
            method: "post",
            data: this.$http.adornData(checkedNodesCatIds, false), //请求体数据
          }).then(({ data }) => {
            this.$message({
              message: "菜单批量删除成功",
              type: "success",
            });
            this.getMenus();
          });
        })
        .catch(() => {});
    },
  },
  watch: {
    //监控data里的menus是否发生变化
    menus() {
      //在菜单全部加载完成后算菜单层级
      for (let i = 0; i < this.menus.length; i++) {
        this.countMenusDeep(this.menus[i]);
      }
    },
  },
  created() {
    this.getMenus();
  },
};
</script>

<style  scoped>
</style> 