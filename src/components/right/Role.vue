<template>
  <div>
    <el-breadcrumb separator-class="el-icon-arrow-right">
      <el-breadcrumb-item :to="{ path: '/' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>
    <div>
      <el-button type="success" plain @click='dialogVisible4Add = true'>添加角色</el-button>
    </div>
    <el-table
      border
      :data="tableData"
      style="width: 100%">
      <el-table-column
        type="expand"
        width="50">
        <template slot-scope="scope">
          <el-row :key="item.id" v-for="item in scope.row.children">
            <el-col :span="3">
              <el-tag @close="deleteRight(scope.row, item.id)" closable>{{item.authName}}</el-tag>
              <i v-if="item.children.length>0" class="el-icon-arrow-right arrow"></i>
            </el-col>
            <el-col :span="21">
              <el-row class="authlist" :key="tag.id" v-for="tag in item.children">
                <el-col :span="4">
                  <el-tag @close="deleteRight(scope.row, tag.id)" type="success" closable>{{tag.authName}}</el-tag>
                  <i v-if="tag.children.length>0" class="el-icon-arrow-right arrow"></i>
                </el-col>
                <el-col :span="20">
                  <el-tag @close="deleteRight(scope.row, btn.id)" :key="btn.id" type="warning" closable v-for="btn in tag.children">{{btn.authName}}</el-tag>
                </el-col>
              </el-row>
            </el-col>
          </el-row>
        </template>
      </el-table-column>
      <el-table-column
        type="index"
        width="50">
      </el-table-column>
      <el-table-column
        prop="roleName"
        label="角色名称"
        width="200">
      </el-table-column>
      <el-table-column
        prop="roleDesc"
        width="500"
        label="描述">
      </el-table-column>
      <el-table-column
        width="260"
        label="操作">
        <template slot-scope="scope">
          <el-button @click='editHandler(scope.row)' size='small' type="primary" icon="el-icon-edit"></el-button>
          <el-button @click='deleteHandler(scope.row)' size='small' type="danger" icon="el-icon-delete"></el-button>
          <el-button @click='grantHandler(scope.row)' size='small' type="warning" icon="el-icon-check"></el-button>
        </template>
      </el-table-column>
    </el-table>
    <!-- 添加角色弹窗 -->
    <el-dialog
      title="添加角色"
      @close='closeUserDialog("add")'
      :visible="dialogVisible4Add"
      width="50%">
      <el-form ref="roleform" :rules="rules" :model="role" label-width="80px">
        <el-form-item label="角色名称" prop='roleName'>
          <el-input v-model="role.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop='roleDesc'>
          <el-input v-model="role.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible4Add = false">取 消</el-button>
        <el-button type="primary" @click="submitRole">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 编辑角色弹窗 -->
    <el-dialog
      title="编辑角色"
      @close='closeUserDialog("edit")'
      :visible="dialogVisible4Edit"
      width="50%">
      <el-form ref="eroleform" :rules="rules" :model="erole" label-width="80px">
        <el-form-item label="角色名称" prop='roleName'>
          <el-input v-model="erole.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述" prop='roleDesc'>
          <el-input v-model="erole.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible4Edit = false">取 消</el-button>
        <el-button type="primary" @click="submitRole4Edit">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 角色授权弹窗 -->
    <el-dialog
      title="角色授权"
      @close='closeUserDialog("grand")'
      :visible="dialogVisible4Grant"
      width="50%">
        <el-tree
          ref='tree'
          :props="treeProps"
          :data="treeData"
          show-checkbox
          default-expand-all
          node-key="id"
          :default-checked-keys="selectTree"
          >
        </el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible4Grant = false">取 消</el-button>
        <el-button type="primary" @click="submitGrant">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
import {submitGrant, rightList, roleList, addRole, getRoleById, editRole, deleteRole, deleteRoleRight} from '../../api/api.js'

export default {
  data () {
    return {
      selectTree: [],
      treeProps: {
        // 匹配树的节点名称和数据名称
        label: 'authName',
        children: 'children'
      },
      treeData: [],
      role: {
        roleName: '',
        roleDesc: ''
      },
      erole: {
        id: '',
        roleName: '',
        roleDesc: ''
      },
      rules: {
        roleName: [
          { required: true, message: '请输入角色名', trigger: 'blur' }
        ],
        roleDesc: [
          { required: true, message: '请输入描述', trigger: 'blur' }
        ]
      },
      dialogVisible4Add: false,
      dialogVisible4Edit: false,
      dialogVisible4Grant: false,
      tableData: [],
      currentRole: ''
    }
  },
  methods: {
    _getThirdRightId (data, arr) {
      // 获取三级权限id
      // var _this = this
      data.forEach((item) => {
        if (!item.children) {
          // [] {} 条件判断表现为true
          // 表示三级权限
          arr.push(item.id)
        } else {
          this._getThirdRightId(item.children, arr)
        }
      })
    },
    submitGrant () {
      // 获取所有选中节点的数据对象列表
      let list = this.$refs['tree'].getCheckedNodes()
      let ids = list.map(item => {
        return item.id + ',' + item.pid
      })
      // 数组去重(Set只允许放不重复的数据)Set的参数是数组
      let tmp = new Set(ids.join(',').split(','))
      let result = [...tmp].join(',')
      // 调用后台接口提交数据
      submitGrant({roleId: this.currentRole, rids: result}).then(res => {
        if (res.meta.status === 200) {
          // 刷新列表
          this.initList()
          // 关闭弹窗
          this.dialogVisible4Grant = false
          // 提示
          this.$message({
            message: res.meta.msg,
            type: 'success'
          })
        }
      })
    },
    grantHandler (row) {
      // 初始化树结构的数据
      rightList({type: 'tree'}).then(res => {
        if (res.meta.status === 200) {
          // 初始化数据
          this.treeData = res.data
          // 设置树形结构的默认选中
          this._getThirdRightId(row.children, this.selectTree)
          // 显示弹窗
          this.dialogVisible4Grant = true
          // 设置当前角色
          this.currentRole = row.id
        }
      })
    },
    deleteRight (row, rightId) {
      deleteRoleRight({roleId: row.id, rightId: rightId}).then(res => {
        if (res.meta.status === 200) {
          // 删除成功，把数据重新赋值
          row.children = res.data
        }
      })
    },
    deleteHandler (row) {
      // console.log(row)
      this.$confirm('此操作将永久删除该用户, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        deleteRole({id: row.id}).then(res => {
          if (res.meta.status === 200) {
            // 刷新列表
            this.initList()
            this.$message({
              type: 'success',
              message: res.meta.msg
            })
          }
        })
      })
    },
    submitRole4Edit (row) {
      // 编辑角色提交表单
      this.$refs['eroleform'].validate(valid => {
        if (valid) {
          editRole(this.erole).then(res => {
            if (res.meta.status === 200) {
              // 关闭弹窗
              this.dialogVisible4Edit = false
              // 刷新列表
              this.initList()
            }
          })
        }
      })
    },
    editHandler (row) {
      getRoleById({id: row.id}).then(res => {
        // 填充表单
        if (res.meta.status === 200) {
          this.erole.id = res.data.roleId
          this.erole.roleName = res.data.roleName
          this.erole.roleDesc = res.data.roleDesc
          // 显示弹窗
          this.dialogVisible4Edit = true
        }
      })
    },
    submitRole () {
      this.$refs['roleform'].validate(valid => {
        if (valid) {
          addRole(this.role).then(res => {
            if (res.meta.status === 201) {
              // 关闭弹窗
              this.dialogVisible4Add = false
              // 刷新列表
              this.initList()
            }
          })
        }
      })
    },
    closeUserDialog (flag) {
      // 关闭弹窗
      if (flag === 'add') {
        this.dialogVisible4Add = false
      } else if (flag === 'edit') {
        this.dialogVisible4Edit = false
      } else {
        this.dialogVisible4Grant = false
      }
    },
    initList () {
      roleList().then(res => {
        if (res.meta.status === 200) {
          this.tableData = res.data
        }
      })
    }
  },
  mounted () {
    this.initList()
  }
}
</script>
<style scoped>
  .el-breadcrumb {
    background-color: #D3DCE6;
    height: 50px;
    line-height: 50px;
    padding-left: 10px;
  }
  .el-tag {
    margin-left: 5px;
  }
  .authlist {
    border-top: solid 1px #D3DCE6;
    padding-bottom: 5px;
  }
  .arrow {
    font-size: 18px;
    font-weight: bold;
  }
</style>
