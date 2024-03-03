<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId"
      :default-expanded-keys="expandedKey" draggable :allow-drop="allowDrop" @node-drop="handleDrop">
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">
            添加
          </el-button>
          <el-button v-if="node.childNodes.length == 0" type="text" size="mini" @click="() => remove(node, data)">
            删除
          </el-button>

          <el-button type="text" size="mini" @click="() => edit(data)">
            修改
          </el-button>
        </span>
      </span></el-tree>

    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
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
  components: {},
  props: {},
  data() {
    return {
      updateNodes: [],
      maxLevel: 0,
      dialogType: "",
      title: "",
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
      dialogVisible: false,
      expandedKey: [],
      menus: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //方法集合
  methods: {
    allowDrop(draggingNode, dropNode, type) {
      // 打印参数以便调试
      console.log("allowDrop", draggingNode, dropNode, type);
      // 计算拖动节点与放置节点之间的最大级别差
      this.countNodeLevel(draggingNode.data);
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      // 这个deep其实就是拖动结束后，所在树的结点层级
      // 根据放置类型，检查是否允许放置拖动节点
      if (type == "inner") {
        // 如果是在放置节点的内部，则深度为放置节点的级别加上拖动节点的深度
        return (deep + dropNode.level) <= 3;
      } else {
        // 如果是在放置节点的外部，则深度为放置节点的父级别加上拖动节点的深度
        return (deep + dropNode.parent.level) <= 3;
      }
    },

    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType) {
      console.log("handleDrop ", draggingNode, dropNode, dropType);
      //1、当前节点最新父节点id
      let pCid = 0;
      let siblings = null;

      if (dropType == "before" || dropType == "after") {
        pCid = dropNode.parent.data.catId == undefined
          ? 0
          : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes

      }

      //2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        {
          if (siblings[i].data.catId == draggingNode.data.catId) {
            //如果遍历的是当前正在拖拽的节点
            let catLevel = draggingNode.level;
            if (siblings[i].level != draggingNode.level) {
              // 当前节点的层级发生变化
              catLevel = siblings[i].level;
              // 修改子节点的层级
              this.updateChildNodeLevel(siblings[i])

            }
            this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel })
          } else {
            this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
          }


        }
      }

      //3、当前拖拽节点的最新层级
      console.log("updateNodes", this.updateNodes)
      // 发送修改请求
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: "菜单排序修改成功",
          type: "success",
        })
        this.getMenus();
        this.expandedKey = [pCid]
        this.updateNodes = []
        this.maxLevel = 0;
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({ catid: cNode.catId, catLevel: node.childNodes[i].level });
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }


    }
    ,

    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    addCategory() {
      console.log("提交的三级分类数据", this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      var data = {
        catId: catId,
        name: name,
        icon: icon,
        productUnit: productUnit,
      };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false),
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
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取到菜单数据", data.data);
        this.menus = data.data;
      });
    },
    append(data) {
      this.dialogType = "add";
      this.title = "添加分类";
      this.category.name = "";
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
    },
    edit(data) {
      this.dialogType = "edit";
      this.title = "修改分类";
      console.log("要修改的数据", data);

      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("要回显的数据", data);
        this.dialogVisible = true;
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】?`, "提示", {
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
            this.$message({
              type: "success",
              message: "菜单删除成功",
            });
            this.getMenus();
            this.expandedKey = [node.parent.data.catId];
          });
          console.log("remove", "node:", node, "data:", data);
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
      console.log("remove", "node:", node, "data:", data);
    },
  },

  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() { },
  beforeCreate() { }, //生命周期 - 创建之前
  beforeMount() { }, //生命周期 - 挂载之前
  beforeUpdate() { }, //生命周期 - 更新之前
  updated() { }, //生命周期 - 更新之后
  beforeDestroy() { }, //生命周期 - 销毁之前
  destroyed() { }, //生命周期 - 销毁完成
  activated() { }, //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>

<style scoped></style>