<template>
  <el-dialog :title="`${{'add': '添加', edit: '编辑'}[state]}敏感权限分类`" class="auth-type-dialog" append-to-body :visible.sync="innerVisible" @close="close" v-loading="loading">
    <el-form class="auth-form" :rules="rules" :model="formData" ref="authForm">
      <el-form-item label="敏感权限分类名称" prop="authName">
        <el-input class="auth-form-info" v-model="formData.authName" placeholder="请输入..." size="small"></el-input>
      </el-form-item>
      <el-form-item label="快通知对象" prop="userId" class="auth-form-user">
        <el-select class="auth-select-user" filterable :filter-method="filterUserNodeSelect" size="small" clearable v-model="formData.userName" placeholder="请选择账号，可选择多个账号" multiple @change="selectChange" @visible-change="initSelect">
          <el-option :value="formData.userId" class="auth-user-list" style="height: auto;">
            <el-tree :data="userList" show-checkbox node-key="id" ref="userTree" :render-content="renderNode" :props="props" @check-change="handleCheckChange" :filter-node-method="filterUserNode" :default-checked-keys="formData.userId" @node-expand="(data) => handleExpand(data, userDefaultExpand)" @node-collapse="(data) => handleDisExpand(data, userDefaultExpand)"></el-tree>
          </el-option>
        </el-select>
      </el-form-item>
      <el-form-item label="监管权限" prop="selectAuth" class="auth-form-content last-form">
        <el-card class="auth-form-card auth-card-custom" shadow="never">
          <div slot="header" class="auth-card-title">
            <span>所有权限</span>
          </div>
          <el-tree :check-strictly="true" :data="authList" :default-checked-keys="defaultCheckAuthKeys" @check="checkChange" @check-change="authCheckChange" :props="props" node-key="auth_id" show-checkbox ref="tree">
          </el-tree>
        </el-card>
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button size="small" type="primary" @click="save">{{ {'add': '完成添加', edit: '完成编辑'}[state] }}</el-button>
    </div>
  </el-dialog>
</template>

<script>
import temporaryPermissionsApi from '@/allApi/temporaryPermissions'
import unifyApi from '@/allApi/unify'
import authApi from '@/allApi/auth'
const COMPANYID = 4861
export default {
  props: ['visible', 'state', 'editAuth'],
  components: {
    tooltipText: require('@/components/tooltip-text.vue').default
  },
  computed: {
    innerVisible: {
      get () {
        return this.visible
      },
      set (visible) {
        this.$emit('update:visible', visible)
      }
    }
  },
  data () {
    return {
      loading: false,
      userList: [],
      userDefaultExpand: [],
      userSelectWord: '',
      formData: {
        authName: '',
        user: [],
        userId: [],
        userName: [],
        selectAuth: []
      },
      defaultCheckAuthKeys: [],
      authList: [],
      rules: {
        authName: [
          { required: true, message: '请输入权限分类名称', trigger: 'blur' }
        ],
        selectAuth: [
          { required: true, validator: this.validateSelectAuth, trigger: 'blur' }
        ]
      },
      props: {
        children: 'children',
        label: 'auth_name',
        value: 'id'
      }
    }
  },
  watch: {
    innerVisible (visible) {
      this.$refs.authForm && this.$refs.authForm.resetFields()
      this.userSelectWord = ''
      this.formData = {
        authName: '',
        user: [],
        userId: [],
        userName: [],
        selectAuth: []
      }
      this.userSelectWord = ''
      this.userDefaultExpand = []
      this.defaultCheckAuthKeys = []
      this.initData(visible)
    },
    userSelectWord (val) {
      this.$refs.userTree.filter(val)
    },
    defaultCheckAuthKeys: {
      // 传过来的是选中数据，根据子节点的完全选中状态，判断是否是半选中状态
      handler (keys) {
        if (keys && keys.length) {
          this.$nextTick(() => {
            keys.forEach(key => {
              this.checkChange(key)
            })
          })
        }
      },
      immediate: true
    }
  },
  methods: {
    validateSelectAuth (rule, value, callback) {
      if (!this.formData.selectAuth || !this.formData.selectAuth.length) callback(new Error('请选择权限'))
      else callback()
    },
    async initData (visible) {
      if (visible) {
        this.loading = true
        await this.getUserList()
        await this.getAuthList()
        let sentId = null
        if (this.state === 'edit') sentId = this.editAuth.id
        if (!sentId) {
          this.loading = false
          return
        }
        try {
          let resAuthData = await temporaryPermissionsApi.getSensitivePermissionsInfo({id: sentId})
          this.formData.authName = resAuthData.name
          this.formData.userId = resAuthData.message_user
          this.formData.userName = resAuthData.message_user_name
          this.defaultCheckAuthKeys = resAuthData.sensitive_auth
        } catch (err) {
          console.log(err)
        }
        this.loading = false
      }
    },
    // #region 下拉树
    initSelect (visible) {
      if (visible) {
        this.$nextTick(() => {
          this.$refs.userTree.setCheckedKeys(this.formData.userId)
        })
      } else {
        this.userSelectWord = ''
      }
    },
    selectChange (e) {
      this.formData.user = this.formData.user.filter(item => e.includes(item.label))
      this.formData.userId = this.formData.user.map(item => item.id)
      this.$refs.userTree.setCheckedNodes(this.formData.userId)
    },
    handleCheckChange () {
      let res = this.$refs.userTree.getCheckedNodes(true, true)
      let [userName, userId, user] = [[], [], []]
      res.forEach(item => {
        if (!userName.includes(item.label)) userName.push(item.label)
        userId.push(item.id)
        user.push(item)
      })
      this.formData.user = user
      this.formData.userId = userId
      this.formData.userName = userName
    },
    handleExpand (data, expandList) {
      let flag = false
      expandList.some(item => {
        if (item === data.id) {
          flag = true
          return true
        }
      })
      if (!flag) {
        expandList.push(data.id)
      }
    },
    handleDisExpand (data, expandList) {
      expandList.some((item, i) => {
        if (item === data.id) {
          expandList.length = i
        }
      })
    },
    // select-tree 实现搜索过滤。
    //   待解决问题：
    //     1、搜索后点击选择后，搜索结果情况，回到了没有搜索状态；
    //     2、搜索后选择父级，选中的是完整数据的父级，不是搜索后的集合
    // 1、外层select的搜索
    filterUserNodeSelect (value) {
      this.userSelectWord = value
    },
    // 2、拿到搜索值，进行树的过滤
    filterUserNode (value, data) {
      if (!value) return true
      return data.id.indexOf(value) !== -1 || data.label.indexOf(value) !== -1
    },
    // #endregion 下拉树
    // #region 获取某公司下用户信息
    getUserList () {
      unifyApi.getDepartmentTree({company_id: COMPANYID}).then(res => {
        let userIdList = []
        this.userList = this.formatTree(res, 'department_id' + COMPANYID, userIdList)
      })
    },
    formatTree (tree, parentId, userIdList) {
      if (tree === undefined || tree.length === 0) return []
      let arr = []
      for (let i = 0; i < tree.length; i++) {
        let obj = {}
        if (tree[i].department_id !== undefined) {
          const id = 'department_id' + tree[i].department_id
          obj = {
            id,
            parent_id: parentId,
            data_permission_type: tree[i].data_permission_type,
            label: tree[i].department_name,
            children: [...this.formatTree(tree[i].children, id, userIdList), ...this.formatTree(tree[i].users, id, userIdList)],
            is_main: false
          }
          if (obj.children.length === 0) {
            obj.disabled = true
            delete obj.children
          }
        } else if (tree[i].user_id !== undefined) {
          if (userIdList.includes(tree[i].user_id)) continue
          else userIdList.push(tree[i].user_id)
          const id = tree[i].user_id + ''
          obj = {
            id,
            udep_id: tree[i].udep_id,
            parent_id: parentId,
            status: tree[i].status,
            data_permission_type: tree[i].data_permission_type,
            label: tree[i].user_name,
            is_main: tree[i].is_main
          }
        }
        arr.push(obj)
      }
      return arr
    },
    // #region 搬运 来自公司管理
    getIconType (id) {
      if (this.isDepartment(id)) return 'eztest eztest-bumen'
      else return 'eztest eztest-geren'
    },
    getFontColor (data) {
      const { status = 1 } = data
      const dataPermissionType = data.data_permission_type
      if (status === 0) return 'padding-left: 5px;color: #d9d9d9'
      let color = ''
      switch (dataPermissionType) {
        case 1:
          color = 'black'
          break
        case 2:
          color = '#FFAC18'
          break
        case 3:
          color = '#9AC137'
          break
        default:
          color = 'black'
          break
      }
      return 'padding-left: 5px;color: ' + color
    },
    getThemeColor () {
      const themeType = JSON.parse(window.sessionStorage.getItem('themeType') || '{}')
      if (themeType.theme) {
        return `color: #${themeType.theme}`
      }
      return 'color: #366fff'
    },
    isDepartment (id) {
      return id.indexOf('department_id') !== -1
    },
    showAsterisk (data) {
      return !data.is_main && !this.isDepartment(data.id)
    },
    renderNode (h, { data, node }) {
      return (
        <div
          style="display: inline-block; width: 96%;">
          <el-row type="flex">
            <el-col span={12}>
              <i class={this.getIconType(data.id)} style={this.getThemeColor()}></i>
              <span class="el-tree-node__label" style={this.getFontColor(data)}>{data.label}</span>
              <span v-show={this.showAsterisk(data)} style={`
                color: #5C80A4;
                display: inline-block;
                font-size: 14px;
                margin-left: 8px;
                padding: 2px 4px;
                background-color: rgba(54,111,255,.1);
                border-radius: 2px;
              `}>副</span>
            </el-col>
          </el-row>
        </div>
      )
    },
    // #endregion 公司管理
    // #endregion 获取某公司下用户信息
    async getAuthList () {
      this.authList = await authApi.getAuthList()
    },
    // #region 父子级复选框联动
    checkChange (node) {
      const currNode = this.$refs.tree.getNode(node)
      if (currNode.checked) {
        currNode.childNodes.map(res => {
          res.checked = true
          if (!res.isLeaf) this.checkChange(res)
        })
        this.setParentChecked(currNode)
      } else {
        this.deleteParentChecked(currNode.parent)
        this.deleteChildChecked(currNode.childNodes)
      }
      this.authCheckChange()
    },
    setParentChecked (parent) {
      // 如果不是全选中为父级添加半选状态，如果子集全选后，父级也要全选
      const fnode = this.$refs.tree.getNode(parent)
      const isAllChecked = fnode.childNodes.every(k => k.checked && k.indeterminate === false) // 子集是否是全选
      if (!fnode.isLeaf) {
        fnode.indeterminate = !isAllChecked // 子集是否是全选，如果子集全选，则半选状态为假
        fnode.checked = true
      }
      if (fnode.parent) {
        this.setParentChecked(fnode.parent)
      }
    },
    deleteParentChecked (parent, d = false) {
      // 如果取消子节点的选中， 设置父级节点选中状态
      const fnode = this.$refs.tree.getNode(parent)
      const isAllChecked = fnode.childNodes.some(k => d ? (k.checked || k.indeterminate) : k.checked) // 子集是否是全选
      if (!fnode.isLeaf) {
        fnode.indeterminate = isAllChecked // 子集是否是全选，如果子集全选，则半选状态为假
        fnode.checked = isAllChecked
        if (fnode.parent) { // 如果有父节点，则需要去判断父节点是否选中
          this.deleteParentChecked(fnode.parent, true)
        }
      }
    },
    deleteChildChecked (childNodes) {
      // 删除子节点的勾选状态
      if (childNodes && childNodes.length > 0) {
        childNodes.map(k => {
          k.indeterminate = false
          k.checked = false
          this.deleteChildChecked(this.$refs.tree.getNode(k).childNodes)
        })
      }
    },
    // #endregion
    authCheckChange (auth) {
      this.formData.selectAuth = []
      this.$refs.tree.getCheckedNodes(true, true).forEach(item => {
        this.formData.selectAuth.push(item)
      })
    },
    close () {
      this.innerVisible = false
    },
    save () {
      this.$refs.authForm.validate((valid) => {
        if (valid) {
          if (this.state === 'edit') this.formData.id = this.editAuth.id
          this.$emit('save', this.formData)
        }
      })
    }
  }
}
</script>

<style lang="less" scoped>
.auth-type-dialog {
  /deep/.el-dialog {
    width: 500px;
    background: #FFFFFF;
    box-shadow: 0px 4px 12px 0px rgba(0,0,0,0.1), 0px 1px 4px 0px rgba(0,0,0,0.1);
    border-radius: 8px;
    border: 1px solid #E4E6E8;
    overflow: hidden;
  }
  /deep/.el-dialog__title {
    font-size: 16px;
    font-weight: 500;
    color: #040508;
    line-height: 24px;
  }
  /deep/.el-dialog__header {
    border-bottom: 1px #E4E6E8 solid;
  }
  /deep/.el-dialog__body {
    padding: 12px 24px;
  }
  .auth-form {
    /deep/.el-form-item__label {
      line-height: 22px;
      float: none;
    }
    .auth-form-user {
      .auth-select-user {
        width: 100%;
        /deep/.el-select__tags {
          max-height: 92px;
          overflow-y: auto;
        }
        /deep/.el-input__inner {
          max-height: 96px;
        }
      }
    }
    .auth-form-content {
      margin-bottom: 0;
      .auth-form-card {
        width: 100%;
        /deep/.el-card__body {
          padding: 4px 15px;
        }
        .auth-selected-list {
          line-height: 24px;
          &:hover {
            background-color: #f5f7fa;
          }
          .auth-content-text {
            display: flex;
            align-items: center;
          }
        }
      }
      /deep/.el-card__header {
        height: 40px;
        background: linear-gradient(135deg, #F9F9FF 0%, #F7FAFF 100%);
        border-radius: 4px 4px 0px 0px;
        .auth-card-title {
          line-height: 6px;
        }
      }
      /deep/.el-card__body {
        height: 320px;
        overflow-y: auto;
        .empty-data {
          display: flex;
          flex-direction: column;
          align-items: center;
          margin-top: 70px;
          img {
            width: 80px;
          }
        }
      }
    }
  }
}
.auth-user-list {
  padding: 0 !important;
}
</style>
