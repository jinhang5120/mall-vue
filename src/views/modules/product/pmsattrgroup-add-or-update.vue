<template>
  <el-dialog
    :title="!dataForm.attrGroupId ? '新增' : '修改'"
    :close-on-click-modal="false"
    :visible.sync="visible"
    @closed="closeDialog"
  >
    <el-form
      :model="dataForm"
      :rules="dataRule"
      ref="dataForm"
      @keyup.enter.native="dataFormSubmit()"
      label-width="120px"
    >
      <el-form-item label="attrGroupName" prop="attrGroupName">
        <el-input
          v-model="dataForm.attrGroupName"
          placeholder="attrGroupName"
        ></el-input>
      </el-form-item>
      <el-form-item label="sort" prop="sort">
        <el-input v-model="dataForm.sort" placeholder="sort"></el-input>
      </el-form-item>
      <el-form-item label="descript" prop="descript">
        <el-input v-model="dataForm.descript" placeholder="descript"></el-input>
      </el-form-item>
      <el-form-item label="icon" prop="icon">
        <el-input v-model="dataForm.icon" placeholder="icon"></el-input>
      </el-form-item>
      <el-form-item label="catelogId" prop="catelogId">
        <!--<el-input v-model="dataForm.catelogId" placeholder="catelogId"></el-input> -->
        <el-cascader
          v-model="dataForm.catelogIds"
          :options="categories"
          :props="props"
          filterable
          placeholder="试着搜索"
        ></el-cascader>
      </el-form-item>
    </el-form>
    <span slot="footer" class="dialog-footer">
      <el-button @click="visible = false">取消</el-button>
      <el-button type="primary" @click="dataFormSubmit()">确定</el-button>
    </span>
  </el-dialog>
</template>

<script>
export default {
  data() {
    return {
      props: { value: "catId", label: "name", children: "children" },
      categories: [],
      categoryArray: [],
      visible: false,
      catelogIdThirdLevel: 0,
      dataForm: {
        attrGroupId: 0,
        attrGroupName: "",
        sort: "",
        descript: "",
        icon: "",
        catelogIds: [],
      },
      dataRule: {
        attrGroupName: [
          { required: true, message: "不能为空", trigger: "blur" },
        ],
        sort: [{ required: true, message: "不能为空", trigger: "blur" }],
        descript: [{ required: true, message: "不能为空", trigger: "blur" }],
        icon: [{ required: true, message: "不能为空", trigger: "blur" }],
        catelogIds: [{ required: true, message: "不能为空", trigger: "blur" }],
      },
    };
  },
  methods: {
    closeDialog() {
      this.dataForm.catelogIds = [];
    },
    getCategories() {
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/list/tree"),
        method: "get",
      }).then(({ data }) => {
        //用({ data })把返回数据中data部分的数据剥离出来
        this.categories = data.pmsCategoryEntities;
      });
    },
    getCategoryArray() {
      this.$http({
        url: this.$http.adornUrl("/product/pmscategory/list"),
        method: "get",
      }).then(({ data }) => {
        //用({ data })把返回数据中data部分的数据剥离出来
        this.categoryArray = data.pmsCategoryEntities;
      });
    },
    init(id) {
      this.dataForm.attrGroupId = id || 0;
      this.visible = true;
      this.$nextTick(() => {
        this.$refs["dataForm"].resetFields();
        if (this.dataForm.attrGroupId) {
          this.$http({
            url: this.$http.adornUrl(
              `/product/pmsattrgroup/info/${this.dataForm.attrGroupId}`
            ),
            method: "get",
            params: this.$http.adornParams(),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.dataForm.attrGroupName = data.pmsAttrGroup.attrGroupName;
              this.dataForm.sort = data.pmsAttrGroup.sort;
              this.dataForm.descript = data.pmsAttrGroup.descript;
              this.dataForm.icon = data.pmsAttrGroup.icon;
              this.catelogIdThirdLevel = data.pmsAttrGroup.catelogId;
              if (this.categoryArray.length > 0) {
                this.displayCatelogIds();
              }
            }
          });
        }
      });
    },
    displayCatelogIds() {
      let catId = this.catelogIdThirdLevel;
      while (catId != 0) {
        for (let i = 0; i < this.categoryArray.length; i++) {
          if (this.categoryArray[i].catId == catId) {
            //从头部插入数组
            this.dataForm.catelogIds.unshift(catId);
            catId = this.categoryArray[i].parentCid;
            break;
          }
        }
      }
    },
    // 表单提交
    dataFormSubmit() {
      this.$refs["dataForm"].validate((valid) => {
        if (valid) {
          this.$http({
            url: this.$http.adornUrl(
              `/product/pmsattrgroup/${
                !this.dataForm.attrGroupId ? "save" : "update"
              }`
            ),
            method: "post",
            data: this.$http.adornData({
              attrGroupId: this.dataForm.attrGroupId || undefined,
              attrGroupName: this.dataForm.attrGroupName,
              sort: this.dataForm.sort,
              descript: this.dataForm.descript,
              icon: this.dataForm.icon,
              catelogId: this.dataForm.catelogIds[
                this.dataForm.catelogIds.length - 1
              ],
            }),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
                duration: 1500,
                onClose: () => {
                  this.visible = false;
                  this.$emit(
                    "refreshDataList",
                    this.dataForm.catelogIds[
                      this.dataForm.catelogIds.length - 1
                    ]
                  );
                },
              });
            } else {
              this.$message.error(data.msg);
            }
          });
        }
      });
    },
  },
  watch: {
    categoryArray() {
      this.displayCatelogIds();
    },
  },
  created() {
    this.getCategories();
    this.getCategoryArray();
  },
};
</script>
