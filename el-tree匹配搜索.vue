<template>
  <div v-loading="loading">
    <el-row class="header-bread-crumb">
      <el-breadcrumb separator="/">
        <el-breadcrumb-item>首页</el-breadcrumb-item>
        <el-breadcrumb-item>系统管理</el-breadcrumb-item>
        <el-breadcrumb-item><b>权限配置管理</b></el-breadcrumb-item>
      </el-breadcrumb>
    </el-row>
    <el-card>
      <div slot="header">
        <el-row>
          <el-col :span="12">
            <h3 style="line-height: 36px;">权限配置管理</h3>
          </el-col>
          <el-col :span="12" align="right" class="operate-area">
            <el-input class="local-search-input" placeholder="请输入" size="small" prefix-icon="el-icon-search"
              v-model.trim="localSearchValue">
              <template v-if="localSearchValue" slot="suffix">
                <span>{{ searchCount + '/' + searchTotal }}</span>
                <el-divider class="search-divider" direction="vertical"></el-divider>
                <i @click="searchNext(-1)" :disable="true" size="small" class="search-arrow pref el-input__icon el-icon-arrow-up" :class="{'sreach-disable': searchCount < 2}"></i>
                <i @click="searchNext(1)" size="small" class="search-arrow next el-input__icon el-icon-arrow-down" :class="{'sreach-disable': searchCount >= searchTotal}"></i>
              </template>
            </el-input>
            <el-button class="add-auth-button" size="small" type="primary" icon="el-icon-plus" @click="editAuth({}, data)">新增权限</el-button>
          </el-col>
        </el-row>
      </div>
      <div class="auth-tree empty-data" v-show="noSearchRes">
        <img src="~assets/img/common/empty-data.png">
        <span>没有匹配结果</span>
      </div>
      <el-tree v-show="!noSearchRes" class="auth-tree" ref="authTree" :data="data" node-key="auth_id" :expand-on-click-node="false" :default-expanded-keys="expandList">
        <span class="custom-tree-node" slot-scope="{ node, data}">
          <el-row type="flex">
            <el-col :span="12">
              <i :class="{'el-icon-menu': +data.type === 1, 'el-icon-document': +data.type !== 1}" style="color: #366fff"></i>
              <!-- <span style="padding-left: 5px;">{{ data.auth_name + "（权限码：" + data.auth_code + "）" }}</span> -->
              <span style="padding-left: 5px;" v-if="!localSearchValue">
                {{ data.auth_name + (+data.type === 1 ? '' : '（权限码：' + data.auth_code + '）') }}
              </span>
              <span style="padding-left: 5px;" class="auth-main" v-else v-html="replaceSearch(data)"></span>
            </el-col>
            <el-col :span="12" align="right" class="auth-operate-button-group">
              <a class="auth-operate-button" @click="() => editAuth(data, node.parent.data)">编辑</a>
              <a class="auth-operate-button" @click="() => editAuth({ auth_pid: data.auth_id }, data.children)">添加</a>
              <a class="auth-operate-button" @click="() => moveAuth(data, node.parent.data)">移动</a>
              <a class="auth-operate-button" @click="() => copyCode(data)">复制</a>
              <a class="auth-operate-button" @click="() => deleteAuth(data, node.parent.data)">删除</a>
            </el-col>
          </el-row>
        </span>
      </el-tree>
    </el-card>
    <auth-edit :visible.sync="authEditVisible" :data="processingAuth" @save="saveAuth"></auth-edit>
    <auth-move :visible.sync="authMoveVisible" :data="data" :target="moveTarget" @move="init" @close="moveTarget = null"></auth-move>
  </div>
</template>

<script>
  import authApi from '@/allApi/auth'
  import AuthEdit from './auth-edit'
  import AuthMove from './auth-move'
  import Clipboard from 'clipboard'
  import { debounce } from 'lodash'
  export default {
    data () {
      return {
        loading: false,
        data: [],
        currentNodeId: 0,
        authEditVisible: false,
        processingAuth: {},
        processingAuthParent: {},
        moveTarget: null,
        authMoveVisible: false,
        localSearchValue: '',
        sreachResult: [],
        searchCount: 0,
        searchTotal: 0,
        expandList: [],
        noSearchRes: false
      }
    },
    components: {
      AuthEdit,
      AuthMove
    },
    watch: {
      localSearchValue (newVal, oldVal) {
        this.reflashSreach(newVal, oldVal)
      }
    },
    created () {
      this.init()
    },
    methods: {
      async init () {
        this.loading = true
        const data = await authApi.getAuthList() // await rbacApi.getAuthList()
        this.loading = false
        if (!data) return
        this.data = data
      },
      editAuth (auth, parent) {
        this.processingAuth = auth
        this.processingAuthParent = !parent ? { children: [] } : Array.isArray(parent) ? { children: parent } : parent
        this.authEditVisible = true
      },
      moveAuth (data, parent) {
        this.moveTarget = data
        // this.moveTarget.parent = parent
        let parentName = [data.auth_name]
        while (parent && parent.auth_id) { // 刨根问底，一直找到该节点所有祖先节点名称
          parentName.unshift(parent.auth_name)
          parent = this.$refs.authTree.getNode(parent.auth_pid) ? this.$refs.authTree.getNode(parent.auth_pid).data : null
        }
        this.moveTarget.parentName = parentName.join('/')
        this.authMoveVisible = true
      },
      async saveAuth (auth) {
        let index = this.processingAuthParent.children.findIndex(child => +child.auth_id === +auth.auth_id)
        let item = this.processingAuthParent.children[index]
        if (index !== -1) {
          if (!(auth.children && auth.children.length)) {
            auth.children = item.children || []
          }
          this.processingAuthParent.children.splice(index, 1, auth)
        } else {
          this.processingAuthParent.children.push(auth)
        }
        this.init()
        this.$nextTick(() => {
          this.expandList.push(auth.auth_pid)
        })
      },
      async deleteAuth (auth, parent) {
        parent = Array.isArray(parent) ? parent : parent.children
        this.$confirm(`是否永久删除权限：${auth.auth_name}？`, '提示', {
          type: 'warning'
        }).then(async () => {
          let data = await authApi.deleteAuth(auth.auth_id) // await rbacApi.deleteAuth(auth.auth_id)
          if (!data) return
          let index = parent.findIndex(child => child.auth_id === auth.auth_id)
          parent.splice(index, 1)
          this.$message.success('删除成功！')
        })
      },
      replaceSearch (data) {
        if (data.auth_name.indexOf(this.localSearchValue) !== -1 || data.auth_code.indexOf(this.localSearchValue) !== -1) {
          return data.auth_name.replace(new RegExp(this.localSearchValue, 'g'), `<font class='find-sreach-world' style='background: #ffff00;'>${this.localSearchValue}</font>`)
          + (+data.type === 1 ? '' : '（权限码：' + data.auth_code.replace(new RegExp(this.localSearchValue, 'g'), `<font class='find-sreach-world' style='background: #ffff00;'>${this.localSearchValue}</font>`) + '）')
        } else {
          return data.auth_name + (+data.type === 1 ? '' : '（权限码：' + data.auth_code + '）')
        }
      },
      reflashSreach: debounce(function (newVal, oldVal) {
        if (!newVal) {
          this.noSearchRes = false
          return
        }
        if (newVal && newVal !== oldVal) {
          this.searchCount = 0
        }
        this.getSreachResult(newVal)
      }, 1000),
      /* 整体的搜索匹配逻辑思路：
      1、replaceSearch    直接对树节点数据进行匹配字符，使用特殊class标记
      2、reflashSreach    防抖中间函数，主要作用只是做防抖
      3、getSreachResult
        a、通过遍历所有树节点数据，存在匹配数据，记录下该节点的id，加入到展开列表中（$refs.refName.store.nodesMap）
        b、在已经展开的树中，直接通过步骤1标记的class查询
      ***这里要先进行展开再进行通过class查询，否则未展开的数据不会被匹配到。
      ***默认全部展开的话，会再首次加载时有很大的性能问题
      4、上下定位，通过步骤3得到的元素列表，直接做对应的效果展示就好了
      5、还需要对定位展示的元素进行是否在可视窗口内，不存在就scrollIntoView
      */
      getSreachResult (newVal) {
        this.expandList = []
        const nodeData = this.$refs.authTree.store.nodesMap
        for (let key in nodeData) {
          if (nodeData.hasOwnProperty(key)) {
            const data = nodeData[key].data
            if (data.auth_name.indexOf(newVal) !== -1 || data.auth_code.indexOf(newVal) !== -1) {
              this.expandList.push(data.auth_id)
            }
          }
        }
        this.$nextTick(() => {
          this.sreachResult = document.querySelectorAll(`.auth-main .find-sreach-world`)
          this.searchTotal = this.sreachResult.length
          if (!this.searchTotal) {
            this.searchCount = 0
            this.noSearchRes = true
            // this.$message.error('没有找到对应匹配的数据')
          } else {
            this.noSearchRes = false
            this.searchCount = 1
            this.sreachResult[0].style.background = '#ff9632'
            this.sreachResult[0].scrollIntoView()
          }
        })
      },
      searchNext (next) {
        this.sreachResult[this.searchCount - 1].style.background = '#ffff00' // 恢复之前数据背景色
        if (next > 0) {
          // console.log('下一个')
          if (this.searchCount < this.searchTotal) this.searchCount = this.searchCount + 1
        } else {
          // console.log('上一个')
          if (this.searchCount > 1) this.searchCount = this.searchCount - 1
        }
        this.sreachResult[this.searchCount - 1].style.background = '#ff9632' // 将当前背景修改
        if (!this.isInViewPort(this.sreachResult[this.searchCount - 1])) this.sreachResult[this.searchCount - 1].scrollIntoView()
      },
      isInViewPort (el) {
        // 判断匹配的数据是否在可视范围
        const viewHeight = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight
        const viewWidth = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth
        const {top, right, left, bottom} = el.getBoundingClientRect()
        return top >= 200 && left >= 0 && bottom <= viewHeight - 100 && right <= viewWidth
        // 此处 200，100 是上下偏移量，viewHeight是可视区域高度，但是树的范围不是整个浏览器窗口，离下面还有差不多100px的距离。顶部同理有200px左右
      },
      copyCode (data) {
        console.log(data.auth_name, data.auth_code)
        let text = document.createElement('button')
        let clipboard = new Clipboard(text)
        clipboard.on('success', () => {
          this.$message.success('复制成功！')
          clipboard.destroy()
        })
        text.dataset.clipboardText = data.auth_code || ''
        text.click()
      }
    }
  }
</script>
<style lang="less" scoped>
.header-bread-crumb {
  padding-bottom: 16px;
  margin-bottom: 8px;
  border-bottom: 1px solid #D6D6D6;
}
.operate-area {
  .local-search-input {
    height: 32px;
    width: 280px;
    .search-divider {
      height: 20px;
    }
    .search-arrow {
    }
    .search-arrow:hover {
      background-color: #DFDFDF;
    }
    .sreach-disable {
      background-color: transparent;
      color: #B5B8BD;
      cursor: not-allowed;
    }
    .sreach-disable:hover {
      background-color: transparent;
    }
    /deep/ .el-input__inner {
      padding-right: 110px;
    }
  }
  .add-auth-button {
    height: 32px;
    margin-left: 24px;
  }
}
.auth-tree {
  border: none;
  height: calc(100vh - 300px);
  overflow: auto;
  .custom-tree-node {
    display: inline-block;
    width: 96%;
    .auth-operate-button-group {
      display: none;
      .auth-operate-button {
        cursor: pointer;
        color: #409eff;
        margin-left: 6px;
      }
    }
  }
  .custom-tree-node:hover .auth-operate-button-group {
    display: block;
  }
}
.empty-data {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: column;
  img {
    height: 150px;
    margin-bottom: 10px;
  }
}
</style>
