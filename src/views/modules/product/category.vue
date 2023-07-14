<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expendedKey"
      draggable
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)">添加</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">修改</el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >删除</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Icon">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="Unit of measure">
          <el-input v-model="category.unit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
//这里可以导入其他文件（比如：组件，工具 js，第三方插件 js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      updateNodes:[],
      maxLevel: 0,
      title: "",
      dialogType: "", //edit\add
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: null,
        icon: "",
        unit: ""
      },
      dialogVisible: false,
      menus: [],
      expendedKey: [],
      defaultProps: {
        children: "children",
        label: "name"
      }
    };
  },
  //计算属性 类似于 data 概念
  computed: {},
  //监控 data 中的数据变化
  watch: {},
  //方法集合
  methods: {
    //这个功能是为了将拖拽进来的数据完全合成后父节点的元素,统一元素
    handleDrop(draggingNode, dropNode,dropType,ev) {
      //当前节点最新的父节点
      let pCid = 0;
      let siblings = null;
      if(dropType == "before" || dropType=="after") {
        pCid = dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      //当前拖拽节点的最新顺序
      for(let i = 0;i<siblings.length;i++) {
          {
            this.updateNodes.push({catId:siblings[i].data.catId,sort:i});
          }
      }

      //当前拖拽节点的最新层级
    },

    countNodeLevel(node) {
      //找到所有的子节点,求出最大深度
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      // console.log("dropnode:",draggingNode,dropNode,type);
      this.countNodeLevel(draggingNode.data);
      //当前正在拖动的节点加上父节点所在的深度不大于三即可
      console.log(this.maxLevel);
      let deep = this.maxLevel - draggingNode.data.catLevel + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
      return false;
    },
    editCategory() {
      var { catId, name, icon, unit } = this.category;
      var data = { catId: catId, name: name, icon: icon, productUnit: unit };
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false)
      })
        .then(({ data }) => {
          this.$message({
            message: "菜单修改成功",
            type: "success"
          });
        })
        .then(() => {
          this.getMenus();

          this.dialogVisible = false;

          this.expendedKey = [this.category.parentCid];
        });
    },
    submitForm() {
      if (this.dialogType == "add") {
        this.addCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      } else {
      }
    },
    edit(data) {
      this.title = "EditType";
      this.dialogType = "edit";
      this.dialogVisible = true;

      //发送请求获得当前节点的最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get"
      }).then(({ data }) => {
        //请求成功
        console.log("要回显得数据", data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.unit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });
    },
    //添加三级分类
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "POST",
        data: this.$http.adornData(this.category, false)
      })
        .then(({ data }) => {
          this.$message({
            message: "菜单添加成功",
            type: "success"
          });
        })
        .then(() => {
          this.getMenus();
          this.dialogVisible = false;

          this.expendedKey = [this.category.parentCid];
        });
    },

    getMenus() {
      // this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get"
      }).then(({ data }) => {
        console.log("成功获取到菜单数据。。。", data);
        this.menus = data.page;
      });
    },

    append(data) {
      this.title = "AddType";
      this.dialogType = "add";
      this.dialogVisible = true;
      // this.dialogVisible = false;
      this.category.parentCid = data.catId;
      //层级
      this.category.catLevel = data.catLevel * 1 + 1;

      //重置
      this.category.sort = 0;
      this.category.showStatus = 1;
      this.category.catId = null;
      this.category.icon = "";
      this.category.unit = "";
      this.category.name = null;
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除[${data.name}]？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false)
          })
            .then(({ data }) => {
              this.$message({
                message: "菜单成功删除",
                type: "success"
              });
            })

            .then(() => {
              this.getMenus();
              this.expendedKey = [node.parent.data.catId];
            });
          //
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除"
          });
        });

      console.log("remove", node, data);
    }
  },
  //生命周期 - 创建完成（可以访问当前 this 实例）
  created() {
    this.getMenus();
  },
  //生命周期 - 挂载完成（可以访问 DOM 元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {} //如果页面有 keep-alive 缓存功能，这个函数会触发
};
</script>
<style scoped>
</style>