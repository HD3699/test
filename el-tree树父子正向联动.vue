<!--
 * @Description: 
 * @Date: 2022-03-30 14:51:22
 * @LastEditors: hd
 * @LastEditTime: 2023-05-19 17:28:39
 * @FilePath: \quickDecisionAdmin\src\entries\index\view\Common\manage\role-manage\role-detail\functional-authority.vue
-->
<template>
  <div class="functional-authority" v-loading="loading">
    <el-tree
      v-if="hasAuth(auth.ROLE_DETAIL_FUN_WATCH)"
      :check-strictly="true"
      :data="functionalAuthority.functionTree"
      :default-checked-keys="functionalAuthority.defaultCheckKeys"
      @check="checkChange"
      :props="{
        label: 'auth_name'
      }"
      node-key="auth_id"
      show-checkbox
      ref="tree">
      <section slot-scope="{ node, data }">
        <span>{{node.label}}</span>
        <div class="auth-switch" v-if="data.extend && node.checked">
          <span :class="{'auth-switch_open': true, 'auth-switch_active': +data.extend.state === 1}" @click="data.extend.state = 1">默认打开</span>
          <span :class="{'auth-switch_close': true, 'auth-switch_active': +data.extend.state === 0}" @click="data.extend.state = 0">默认关闭</span>
        </div>
      </section>
    </el-tree>
    <div class="functional-authority__footer">
      <el-button type="primary" v-if="isWatch" @click="cancleHandler">返回</el-button>
      <el-button v-if="!isWatch" @click="cancleHandler">取消</el-button>
      <el-button v-if="!isWatch" type="primary" @click="saveHandler">保存</el-button>
    </div>
  </div>
</template>
<script>
import API from '@/allApi/unify'
import { isEqual } from 'lodash'
import { deepCopy } from '@/lib/utils'
import { formatUrl } from 'Lib/utils/formatUrl'
import auth from '../../code.js'
import { hasAuth } from 'Lib/utils/index.js'
export default {
  name: 'functionalAuthority',
  data () {
    return {
      functionalAuthority: {
        functionTree: [],
        defaultCheckKeys: [],
        role_id: '',
        auth: {}
      },
      loading: false
    }
  },
  computed: {
    isEdit () {
      return this.$route.query.type === 'edit'
    },
    isAdd () {
      return this.$route.query.type === 'add'
    },
    isWatch () {
      return this.$route.query.type === 'watch'
    }
  },
  created () {
    this.auth = auth
    this.initFunctionalAuthority()
  },
  watch: {
    'functionalAuthority.defaultCheckKeys': {
      // 传过来的是选中数据，根据子节点的完全选中状态，判断是否是半选中状态
      handler (keys) {
        if (keys && keys.length) {
          this.$nextTick(() => {
            keys.forEach(key => {
              const fnode = this.$refs.tree.getNode(key)
              const isAllChecked = fnode.childNodes.every(k => k.checked && k.indeterminate === false) // 子集是否是全选
              if (!fnode.isLeaf) {
                fnode.indeterminate = !isAllChecked // 子集是否是全选，如果子集全选，则半选状态为假
                fnode.checked = true
              }
            })
          })
        }
      },
      immediate: true
    }
  },
  methods: {
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
    hasAuth,
    // 角色权限配置交互调整如下
    // 1. 父级节点变更联动所有子级节点
    // 2. 子级节点勾选则父级节点也勾选
    // 3. 子级节点取消不会影响父级节点
    handleNodeChange ({ auth_id: authId, parent, children }, { checkedKeys }) {
      const state = checkedKeys.includes(authId)
      if (children) this.updateChildrenState(children, state)
      if (state && parent) this.updateParentState(parent, state)
    },
    updateChildrenState (list, state) {
      list.forEach(({ auth_id: authId, children }) => {
        this.$refs.tree.setChecked(authId, state)
        if (children) this.updateChildrenState(children, state)
      })
    },
    updateParentState ({ auth_id: authId, parent }, state) {
      this.$refs.tree.setChecked(authId, state)
      if (parent) this.updateParentState(parent, state)
    },
    dataVerification ({ callback }) {
      if (isEqual(this.catchData, this.functionalAuthority.defaultCheckKeys)) {
        callback(true)
      } else {
        this.$confirm(`角色信息发生修改，是否保存`, '编辑提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.toSaveAuths({
            callback: function () {
              callback(true)
            }
          })
        }).catch(_ => {
          callback(true)
        })
      }
    },
    // 绑定父节点，方便节点变更联动
    normalizeAuthList (authList, parent) {
      authList.forEach(auth => {
        if (auth.children) {
          this.normalizeAuthList(auth.children, auth)
        }
        auth.parent = parent
      })
    },
    initFunctionalAuthority () {
      if (this.isAdd) {
        this.catchData = deepCopy(this.functionalAuthority.defaultCheckKeys)
      } else {
        this.loading = true
        this.role_id = JSON.parse(window.sessionStorage.getItem('roleInfo')).basicInfo.role_id
        API.getAuthTree({
          role_id: this.$route.query.id
        }).then(res => {
          const { auths } = JSON.parse(window.sessionStorage.getItem('roleInfo'))
          this.normalizeAuthList(res)
          this.functionalAuthority = {
            functionTree: res,
            defaultCheckKeys: auths
          }
          this.catchData = deepCopy(this.functionalAuthority.defaultCheckKeys)
          if (this.isWatch || !hasAuth(this.auth.ROLE_DETAIL_FUN_EDIT)) {
            this.setDisabled(this.functionalAuthority.functionTree)
          }
          this.loading = false
        })
      }
    },
    setDisabled (arr) {
      if (arr === undefined || arr.length === 0) return
      arr.forEach(item => {
        item.disabled = true
        this.setDisabled(item.children)
      })
    },
    cancleHandler () {
      this.$router.push(formatUrl('/common/manage/role-manage/role-list'))
    },
    saveHandler () {
      const _this = this
      this.toSaveAuths({
        callback: function () {
          _this.$router.push({
            path: formatUrl('/common/manage/role-manage/role-detail/functional-authority'),
            query: {
              type: 'edit',
              id: _this.role_id
            }
          })
          _this.loading = false
        }
      })
    },
    toSaveAuths ({ callback }) {
      this.loading = true
      const auths = JSON.stringify(this.$refs.tree.getCheckedKeys())
      API.saveAuths({
        role_id: this.role_id,
        auth_ids: auths
      }).then(res => {
        this.catchData = deepCopy(this.functionalAuthority.defaultCheckKeys)
        this.updateSeesionData(JSON.parse(auths))
        this.$message.success('保存成功！')
        callback && callback(res)
      }).finally(_ => {
        this.loading = false
      })
    },
    updateSeesionData (auths) {
      let roleInfo = JSON.parse(window.sessionStorage.getItem('roleInfo'))
      roleInfo.auths = auths
      window.sessionStorage.setItem('roleInfo', JSON.stringify(roleInfo))
    }
  }
}
</script>
<style lang="less" scoped>
.functional-authority{
  margin-top: 20px;
  padding: 7px;
  box-sizing: border-box;
  height: calc( 100% - 52px );
  .auth-switch {
    display: inline-block;
    margin-left: 8px;
    &_open, &_close {
      width: 80px;
      padding: 2px 4px;
      text-align: center;
      border: 1px #ccc solid;
    }
    &_active {
      background: #366fff;
      color: white;
    }
    & > span + span {
      border-left: none;
    }
  }
  &__footer{
    width: 298px;
    text-align: center;
    position: absolute;
    bottom: 20px;
    left: calc( 50% - 149px );
  }
}
</style>
