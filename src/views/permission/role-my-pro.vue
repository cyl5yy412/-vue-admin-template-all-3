<template>
  <div class="app-container">
    <!-- <el-button type="primary" @click="handleAddRole">新增</el-button> -->

    <el-table :data="rolesList" style="width: 100%;margin-top:30px;" border>
      <el-table-column align="center" label="Role Key" width="220">
        <template slot-scope="scope">
          <!-- {{ scope.row.key }} -->
          {{ scope.row.roleId }}
        </template>
      </el-table-column>
      <el-table-column align="center" label="Role Name" width="220">
        <template slot-scope="scope">
          <!-- {{ scope.row.name }} -->
          {{ scope.row.roleName }}
        </template>
      </el-table-column>
      <el-table-column align="header-center" label="Description">
        <template slot-scope="scope">
          <!-- {{ scope.row.description }} -->
          {{ scope.row.roleDesc }}
        </template>
      </el-table-column>
      <el-table-column align="center" label="Operations">
        <template slot-scope="scope">
          <el-button type="primary" size="small" @click="handleEdit(scope)">修改</el-button>
          <el-button type="danger" size="small" @click="handleDelete(scope)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <el-dialog :visible.sync="dialogVisible" :title="dialogType==='edit'?'Edit Role':'New Role'">
      <el-form :model="role" label-width="80px" label-position="left">
        <el-form-item label="角色名">
          <!-- <el-input v-model="role.name" placeholder="Role Name" /> -->
          <el-input v-model="role.roleName" placeholder="Role Name" />
        </el-form-item>
        <el-form-item label="描述">
          <el-input
            v-model="role.roleDesc"
            :autosize="{ minRows: 2, maxRows: 4}"
            type="textarea"
            placeholder="Role Description"
          />
        </el-form-item>
        <el-form-item label="权限">
          <el-tree
            ref="tree"
            :data="routesData"
            accordion
            highlight-current
            show-checkbox
            node-key="id"
            class="permission-tree"
            :default-checked-keys="permissionsChecked"
            default-expand-all
          />
        </el-form-item>
        <!-- 
:default-checked-keys="permissionsChecked"

            注:没有 该属性值,会直接不显示菜单名字
            :props="defaultProps"
            使选中根节点 不选择子:父子不相关
            :check-strictly="checkStrictly"
            tree默认选择
            :default-checked-keys="permissionsChecked"
            :default-checked-keys="[5]"
            点击节点就选中
            check-on-click-node=true
        -->
      </el-form>
      <div style="text-align:right;">
        <el-button type="danger" @click="dialogVisible=false">Cancel</el-button>
        <el-button type="primary" @click="confirmRole">Confirm</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import path from "path";
import { deepClone } from "@/utils";
import {
  getRoutes,
  getRoles,
  addRole,
  deleteRole,
  updateRole,
  //pro--------------------------
  getRoutesPro
} from "@/api/role";
import service from "../../utils/request";

const defaultRole = {
  key: "",
  name: "",
  description: "",
  routes: [],
  //自定义de属性
  roleId:'',
  roleName:'',
  roleDesc:'',
  permissionsChecked:[],
  checkedKeys:'',
};

export default {
  data() {
    return {
      role: Object.assign({}, defaultRole),
      routes: [],
      rolesList: [],
      dialogVisible: false,
      dialogType: "new",
      checkStrictly: false,
      defaultProps: {
        children: "children",
        label: "title"
      },
      //已拥有的权限
      permissionsChecked: []
    };
  },
  computed: {
    routesData() {
      return this.routes;
    }
  },
  watch: {
    //通过 监听 来实现 初始化已有的权限
    dialogVisible(now, before) {
      if (now == false) {
        //此方法无效
        // this.permissionsChecked = [];
        //element这个el-tree组件是是采用赋值的方式改变是否勾选的，所以你应该使用组件中提供的this.$refs.tree.setCheckedKeys([]);这个方法来清空勾选项。
        this.$refs.tree.setCheckedKeys([]);
      }
    }
  },
  created() {
    // Mock: get all routes and roles list from server
    //获取 role menu
    this.getRoutes();
    this.getRoles();

    // 初始化监听  === 这不是一个方法
    // this.dialogVisible();
  },
  methods: {
    getPermissionsById(scope) {
        console.log(scope)
      service.get("/role/getPermissionsByIdPro/" + scope.row.roleId).then(res => {
        this.permissionsChecked = res.data;
        // 手动选中节点 - 可代替默认选中
         for (const i of res.data) {
          this.$refs.tree.setChecked(i, true, false)
        }
        //接受勾选节点的数组
        //  this.$refs.tree.setCheckedNodes()
      });
      //查看选中
    },
    async getRoutes() {
    //   const res = await getRoutes();
      const res = await getRoutesPro();
      this.serviceRoutes = res.data;
      //不被系统处理 ,直接返回routes
      //   this.routes = this.generateRoutes(res.data)
      this.routes = res.data;
    },
    async getRoles() {
      const res = await getRoles();
      this.rolesList = res.data;
    },

    // Reshape the routes structure so that it looks the same as the sidebar
    generateRoutes(routes, basePath = "/") {
      const res = [];

      for (let route of routes) {
        // skip some route
        if (route.hidden) {
          continue;
        }

        const onlyOneShowingChild = this.onlyOneShowingChild(
          route.children,
          route
        );

        if (route.children && onlyOneShowingChild && !route.alwaysShow) {
          route = onlyOneShowingChild;
        }

        const data = {
          path: path.resolve(basePath, route.path),
          title: route.meta && route.meta.title
        };

        // recursive child routes
        if (route.children) {
          data.children = this.generateRoutes(route.children, data.path);
        }
        res.push(data);
      }
      return res;
    },
    generateArr(routes) {
      let data = [];
      routes.forEach(route => {
        data.push(route);
        if (route.children) {
          const temp = this.generateArr(route.children);
          if (temp.length > 0) {
            data = [...data, ...temp];
          }
        }
      });
      return data;
    },
    handleAddRole() {
      this.role = Object.assign({}, defaultRole);
      if (this.$refs.tree) {
        this.$refs.tree.setCheckedNodes([]);
      }
      this.dialogType = "new";
      this.dialogVisible = true;
    },
    handleEdit(scope) {
      this.dialogType = "edit";
      this.dialogVisible = true;
      this.checkStrictly = true;
      this.role = deepClone(scope.row);
      //加载已拥有的角色权限
      this.getPermissionsById(scope);
      //   this.$nextTick(() => {
      //     const routes = this.generateRoutes(this.role.routes);
      //     this.$refs.tree.setCheckedNodes(this.generateArr(routes));
      //     // set checked state of a node not affects its father and child nodes
      //     this.checkStrictly = false;
      //   });
    },
    handleDelete({ $index, row }) {
      this.$confirm("Confirm to remove the role?", "Warning", {
        confirmButtonText: "Confirm",
        cancelButtonText: "Cancel",
        type: "warning"
      })
        .then(async () => {
          await deleteRole(row.roleId);
          this.rolesList.splice($index, 1);
          this.$message({
            type: "success",
            message: "Delete succed!"
          });
        })
        .catch(err => {
          console.error(err);
        });
    },
    generateTree(routes, basePath = '/', checkedKeys) {
      const res = []

      for (const route of routes) {
        const routePath = path.resolve(basePath, route.path)

        // recursive child routes
        if (route.children) {
          route.children = this.generateTree(route.children, routePath, checkedKeys)
        }

        if (checkedKeys.includes(routePath) || (route.children && route.children.length >= 1)) {
          res.push(route)
        }
      }
      return res
    },
    async confirmRole() {
      const isEdit = this.dialogType === "edit";

      //获得选中的树节点  , 后台需要判断是否选中的根节点
      const checkedPermissions = this.$refs.tree.getCheckedKeys();
      console.log(checkedPermissions +"==="+isEdit);
      const chkNodes = this.$refs.tree.getCheckedNodes();
      console.log(chkNodes +"===");
      //赋值
      this.role.checkedKeys = checkedPermissions

        // const checkedKeys = this.$refs.tree.getCheckedKeys()
        // this.role.routes = this.generateTree(deepClone(this.serviceRoutes), '/', checkedKeys)
      
      
        if (isEdit) {//修改
          // await updateRole(this.role.key, this.role)
          console.log(this.role.roleId)
          await updateRole(this.role.roleId, this.role)
          for (let index = 0; index < this.rolesList.length; index++) {
            if (this.rolesList[index].key === this.role.key) {
              this.rolesList.splice(index, 1, Object.assign({}, this.role))
              break
            }
          }
        } else {//添加
          const { data } = await addRole(this.role)
          this.role.key = data.key
          this.rolesList.push(this.role)
        }

        const { roleDesc, roleId, roleName } = this.role
        this.dialogVisible = false
        this.$notify({
          title: 'Success',
          dangerouslyUseHTMLString: true,
          message: `
              <div>Role Key: ${roleId}</div>
              <div>Role Name: ${roleName}</div>
              <div>Description: ${roleDesc}</div>
            `,
          type: 'success'
        })
    },
    // reference: src/view/layout/components/Sidebar/SidebarItem.vue
    onlyOneShowingChild(children = [], parent) {
      let onlyOneChild = null
      const showingChildren = children.filter(item => !item.hidden)

      // When there is only one child route, the child route is displayed by default
      if (showingChildren.length === 1) {
        onlyOneChild = showingChildren[0]
        onlyOneChild.path = path.resolve(parent.path, onlyOneChild.path)
        return onlyOneChild
      }

      // Show parent if there are no child route to display
      if (showingChildren.length === 0) {
        onlyOneChild = { ... parent, path: '', noShowingChildren: true }
        return onlyOneChild
      }

      return false
    }
  }
};
</script>

<style lang="scss" scoped>
.app-container {
  .roles-table {
    margin-top: 30px;
  }
  .permission-tree {
    margin-bottom: 30px;
  }
}
</style>
