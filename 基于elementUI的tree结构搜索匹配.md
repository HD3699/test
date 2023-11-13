### 基于elementUI的tree结构搜索匹配
##### 大致需求，对于一个树结构进行对应数据的匹配管理进行定位（并非过滤筛选，匹配得到的树还是完整的数据，会把成功匹配的节点进行标记显示，存在多个匹配数据时可进行上下切换定位）
##### 思路：
1. 对树节点数据进行需要满足的数据匹配，将成功匹配数据项使用特殊class标记
  > 
  ``` HTML
  <div class="tree empty-data" v-show="noSearchRes">
    <img src="~assets/img/empty-data.png">
    <span>没有匹配结果</span>
  </div>
  <el-tree v-show="!noSearchRes" class="tree" ref="tree" :data="data" node-key="id" :expand-on-click-node="false" :default-expanded-keys="expandList">
    <span class="custom-tree-node" slot-scope="{ node, data}">
      <span v-if="!localSearchValue">
        {{ data.name }}
      </span>
      <!-- 通过replaceSearch函数将匹配元素添加标记class -->
      <span v-else v-html="replaceSearch(data)"></span>
    </span>
  </el-tree>
  ```
  ```JS
  replaceSearch (data) {
    if (data.name.indexOf(this.localSearchValue) !== -1 || data.id.indexOf(this.localSearchValue) !== -1) {
      return `<font class='find-sreach-world' style='background: #ffff00;'>${data.name}</font>`
      // 这里要是只想把匹配成功的字符进行高亮标记，可以 ↓ 只标记匹配元素
      // data.name.replace(new RegExp(this.localSearchValue, 'g'), `<font class='find-sreach-world' style='background: #ffff00;'>${this.localSearchValue}</font>`)
    } else {
      return data.name
    }
  }
  ```
2. 为了输入搜索时顺畅，建议添加防抖
  ```js
  import { debounce } from 'lodash'
  export default {
    watch: {
      localSearchValue (newVal, oldVal) {
        this.reflashSreach(newVal, oldVal)
      }
    },
    methods: {
      reflashSreach: debounce(function (newVal, oldVal) {
        if (!newVal) {
          this.noSearchRes = false
          return
        }
        if (newVal && newVal !== oldVal) this.searchCount = 0
        this.getSreachResult(newVal)
      }, 1000)
    }
  }
  ```
3. 通过遍历所有树节点数据，存在匹配数据，记录下该节点的id，加入到展开列表中
  1. 此处不需要将树数据进行平铺展开，可以使用$refs.refName.store.nodesMap得到所有节点的对象{node_id: node_data}
  2. 遍历该对象得到数据项，取其data即是节点数据。
  3. （在树的html定义处声明节点关键key：node-key="id"）做相对应的判断，将满足条件的data的关键属性push添加到树的展开列表expandList中
4. 通过:default-expanded-keys="expandList"，会将满足匹配条件的数据进行自动展开。
5. 在已经展开的树中，通过步骤1标记的class查询
  1. *** 这里要先进行展开再进行通过class查询，否则未展开的数据不会被匹配到。 ***
  2. *** 默认全部展开的话，会再首次加载时有很大的性能问题，所以才有前面那么一堆操作 ***
6. 匹配结构的总数量就是展开列表expandList的length
  ```js
  getSreachResult (newVal) {
    this.expandList = []
    const nodeData = this.$refs.tree.store.nodesMap
    for (let key in nodeData) {
      if (nodeData.hasOwnProperty(key)) {
        const data = nodeData[key].data
        idata.name.indexOf(newVal) !== -1 && this.expandList.push(data.id)
      }
    }
    if (this.expandList.length === 0) return
    this.$nextTick(() => { // 不能同步执行，需要等树展开之后再搜索查找
      this.sreachResult = document.querySelectorAll(`.custom-tree-node .find-sreach-world`)
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
  // 多加两个按钮，@click="searchNext(-1)"、@click="searchNext(1)"，进行上一个下一个切换
  // 这里如果需要首个-1到最后或者最后+1到第一个，修改searchCount值到0或者到searchTotal就好
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
  ```

7. 还需要对定位展示的元素进行是否在可视窗口内，不存在就scrollIntoView
  ```js
  isInViewPort (el) {
    // 判断匹配的数据是否在可视范围
    const viewHeight = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight
    const viewWidth = window.innerWidth || document.documentElement.clientWidth || document.body.clientWidth
    const {top, right, left, bottom} = el.getBoundingClientRect()
    return top >= 200 && left >= 0 && bottom <= viewHeight - 100 && right <= viewWidth
    // 此处 200，100 是上下偏移量，viewHeight是浏览器可视区域高度，但是树的范围不是整个浏览器窗口。
    // 离下面还有差不多100px的距离。顶部同理有200px左右（具体数值根据树的位置确定）
  },
  ```

8. 新增需求，需要对节点进行完整路径匹配。比如输入路径：一级层级/二级层级1/三级层级4/四级层级2，能够直接匹配到具体位置。
  1. 可以考虑在获取树数据是，为每一个节点加入一个全路径属性
  ```js
  async init () {
    await ……
    this.setParent()
  },
  setParent () {
    const nodeData = this.$refs.tree.store.nodesMap
    for (let key in nodeData) {
      if (nodeData.hasOwnProperty(key)) {
        const data = nodeData[key].data
        let parent = this.$refs.authTree.getNode(data.pid) ? this.$refs.authTree.getNode(data.pid).data : null
        let pathName = [data.name]
        let pathId = [data.id]
        while (parent && parent.id) { // 刨根问底，一直找到该节点所有祖先节点名称
          pathName.unshift(parent.name)
          pathId.unshift(parent.id)
          parent = this.$refs.tree.getNode(parent.pid) ? this.$refs.authTree.getNode(parent.pid).data : null
        }
        data.pathName = pathName.join('/')
        data.pathId = pathId.join('/')
      }
    }
  },
  ```
  2. 添加节点class标记时，需要修改，进行全匹配，不存在全路径匹配成功情况再进行包含匹配，提高匹配容错率。
  3. 全匹配情况下不能做包含处理，否则搜索父级，所有子级都会被判断为匹配，所以只能做相等
  ```js
  replaceSearch (data) {
    if (data.pathName === this.localSearchValue || data.pathId === this.localSearchValue
      || data.pathName.endsWith(this.localSearchValue) || data.pathId.endsWith(this.localSearchValue)) {
      return `<font class='find-sreach-world' style='background: #ffff00;'>${data.name}</font>`
    } else if (data.name.indexOf(this.localSearchValue) !== -1 || data.code.indexOf(this.localSearchValue) !== -1) {
      return `<font class='find-sreach-world' style='background: #ffff00;'>${data.name}</font>`
    } else {
      return data.auth_name
    }
  },
  ```
  4. getSreachResult做小小的优化修改
  ```js
  reflashSreach: debounce(function (newVal, oldVal) {
    if (!newVal) {
      this.noSearchRes = false
      return
    }
    if (newVal && newVal !== oldVal) this.searchCount = 0
    if (newVal.indexOf('/') !== -1) {
      this.getSreachResult(newVal, 1)
    } else {
      this.getSreachResult(newVal)
    }
  }, 1000),
  getSreachResult (newVal, type = 0) {
    this.expandList = []
    const nodeData = this.$refs.tree.store.nodesMap
    for (let key in nodeData) {
      if (nodeData.hasOwnProperty(key)) {
        const data = nodeData[key].data
        if (type) {
          if (data.pathName === newVal || data.pathId === newVal
            || data.pathName.endsWith(newVal) || data.pathId.endsWith(newVal)) {
            this.expandList.push(data.id)
          }
        } else {
          if (data.name.indexOf(newVal) !== -1 || data.code.indexOf(newVal) !== -1) {
            this.expandList.push(data.id)
          }
        }
      }
    }
    if (type && this.expandList.length === 0) {
      this.getSreachResult(newVal)
      return
    }
    ……
  }
  ```