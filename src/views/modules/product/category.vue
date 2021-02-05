<template>
  <div>
    <el-button type="danger" @click="deleteIds">批量删除</el-button>
    <el-tree
      :data="meaus"
      draggable
      :allow-drop="allowDrop"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKeys"
      :expand-on-click-node="false"
      ref="menu"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span node-key="node">
          <el-button
            type="text"
            size="mini"
            @click="() => append(data)"
            v-if="node.level <= 2"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            type="text"
            size="mini"
            @click="() => remove(node, data)"
            v-if="data.children.length <= 2"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      :before-close="handleClose"
    >
      <el-form :model="category">
        <el-form-item label="菜单名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="产品计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="cancal">取 消</el-button>
        <el-button type="primary" @click="submit">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      //定义一个最大节点变量
      maxlevel: 0,
      title: "",
      logtype: "",
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: null,
        productUnit: "",
      },
      dialogVisible: false,
      expandedKeys: [],
      meaus: {},
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    //批量删除
    deleteIds() {
      this.$confirm(`确定删除节点吗`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
        //获取所选中到节点
        var allIds = this.$refs.menu.getCheckedNodes();
        //将所有节点数据的catid封装进一个数组中
        let ids = [];
        for (let i = 0; i < allIds.length; i++) {
          ids.push(allIds[i].catId);
        }
        //发送请求 删除节点信息
        this.$http({
          url: this.$http.adornUrl("/product/category/delete"),
          method: "post",
          data: this.$http.adornData(ids, false),
        }).then(({ data }) => {
          this.$message({
            message: "删除成功",
            type: "success",
          });
          this.getCategory()
        });
      });
    },
    //判断是否可拖拽 思路
    //如果拖拽节点的深度加上目标节点的深度 大于3 则不能拖动， 如果拖动节点 拖动到指定节点前面或者后面 则 当前节点深度加上目标节点父节点深度不大于3即可
    allowDrop(draggingNode, dropNode, type) {
      console.log(draggingNode, dropNode, type);
      this.countLevel(draggingNode.data);
      //拖拽节点的深度等于 （拖拽节点深度-当前节点父深度+1）
      var deep = this.maxlevel - draggingNode.data.catLevel + 1;
      //如果是拖拽到节点里 就计算当前深度加上目标深度是否大于3
      if (type == "inner") {
        return deep + dropNode.data.catLevel <= 3;
      }
      if (type == "prev" || type == "next") {
        return deep + dropNode.parent.level <= 3;
      }
    },

    countLevel(nodeData) {
      for (let i = 0; i < nodeData.children.length; i++) {
        if (
          nodeData.children[i] != null &&
          nodeData.children[i].catLevel >= this.maxlevel
        ) {
          this.maxlevel = nodeData.children[i].catLevel;
        }
        this.countLevel(nodeData.children[i]);
      }
    },
    handleClose(done) {
      this.$confirm("确认关闭？")
        .then((_) => {
          (this.category.name = ""),
            (this.category.catId = ""),
            (this.category.parentCid = ""),
            (this.category.catLevel = ""),
            (this.category.sort = ""),
            (this.category.icon = ""),
            (this.category.productUnit = "");
          this.dialogVisible = false;
        })
        .catch((_) => {});
    },

    cancal() {
      (this.category.name = ""),
        (this.category.catId = ""),
        (this.category.parentCid = ""),
        (this.category.catLevel = ""),
        (this.category.sort = ""),
        (this.category.icon = ""),
        (this.category.productUnit = "");
      this.dialogVisible = false;
    },
    //添加和修改公用一个表单
    submit() {
      if (this.logtype == "add") {
        this.addCategory();
      }
      if (this.logtype == "edit") {
        this.editCategory();
      }
    },
    //表单编辑功能
    edit(data) {
      this.logtype = "edit";
      this.title = "编辑分类";
      this.dialogVisible = true;
      //要获取最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.category.name = data.data.name;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.catId = data.data.catId;
      });
    },

    editCategory() {
      var { name, icon, catId, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ name, icon, catId, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getCategory();
        this.expandedKeys = [this.category.parentCid];
        (this.category.name = ""),
          (this.category.catId = ""),
          (this.category.parentCid = ""),
          (this.category.catLevel = ""),
          (this.category.sort = ""),
          (this.category.icon = ""),
          (this.category.productUnit = "");
      });
    },
    //获取列表
    getCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.meaus = data.data;
      });
    },
    handleNodeClick(data) {
      console.log(data);
    },
    //添加分类
    append(data) {
      this.logtype = "add";
      this.title = "添加分类";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLeve * 1 + 1;
    },

    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then((res) => {
        this.$message({
          message: "添加成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getCategory();
        this.expandedKeys = [this.category.parentCid];
        (this.category.name = ""),
          (this.category.catId = ""),
          (this.category.parentCid = ""),
          (this.category.catLevel = ""),
          (this.category.sort = ""),
          (this.category.icon = ""),
          (this.category.productUnit = "");
      });
    },
    //删除分类
    remove(node, data) {
      var ids = [data.catId];

      this.$confirm(`确定删除${data.name}节点吗`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then((data) => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          });
        })
        .then((res) => {
          this.$message({
            message: "删除成功",
            type: "success",
          });
          this.getCategory();
          this.expandedKeys = [node.parent.data.catId];
        });
    },
  },
  //钩子函数
  created() {
    this.getCategory();
  },
};
</script>