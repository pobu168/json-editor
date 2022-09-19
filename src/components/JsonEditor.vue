<template>
  <a-row>
    <a-col :span="2"></a-col>
    <a-col :span="9">
      <a-card title="JSON编辑" style="width: 100%">
        <a-button @click="addRootJson">新增顶层字段</a-button>
        <a-tree
        :defaultExpandAll="true"
        :tree-data="treeData"
        >
          <template #title="{ title, key, value, children }">
            <div style="width:500px">
              <span>{{ title }}</span>
              <span v-if="!(children && children.length > 0)">:{{ value }}</span>
              <div style="float:right">
                <a-button size="small" class="btn-padding" @click="addRootNode(key)">
                  <template #icon><PlusOutlined /></template>
                </a-button>
                <a-button size="small" class="btn-padding" @click="editNode(title, key, value, children)">
                  <template #icon><EditOutlined /></template>
                </a-button>
                <a-button size="small" class="btn-padding" @click="delNode(title, key, children)">
                  <template #icon><DeleteOutlined /></template>
                </a-button>
              </div>
            </div>
          </template>
        </a-tree>
        <a-modal v-model:visible="visible" title="节点">
          <template #footer>
            <a-button key="back" type="primary" @click="confirmNode">确定</a-button>
            <a-button key="back" @click="cancelNode">取消</a-button>
          </template>
          <a-form
            :label-col="{ span: 8 }"
            :wrapper-col="{ span: 16 }"
          >
            <a-form-item label="Key">
              <a-input v-model:value="nodeAttrs.title" />
            </a-form-item>
            <a-form-item label="Value" v-if="!hasChildren">
              <a-input v-model:value="nodeAttrs.value" />
            </a-form-item>
            <a-form-item label="Type" v-if="!hasChildren">
              <a-select v-model:value="nodeAttrs.type">
                <a-select-option value="string">String</a-select-option>
                <a-select-option value="number">Number</a-select-option>
                <a-select-option value="json">Json</a-select-option>
              </a-select>
            </a-form-item>
          </a-form>
        </a-modal>
      </a-card>
    </a-col>
    <a-col :span="2"></a-col>
    <a-col :span="9">
      <a-card title="结果展示" style="width: 100%">
        {{finalResult}}
      </a-card>
    </a-col>
    <a-col :span="2"></a-col>
  </a-row>
  <a-divider />
  <div style="width:100%;text-align: center;">
    <a-button type="primary" style="margin: 0 4px" @click="toFinalJson">预览JSON</a-button>
    <a-button style="margin: 0 4px" @click="importJson">导入数据</a-button>
  </div>
  <a-modal v-model:visible="importModalVisible" title="JSON字符串">
    <template #footer>
      <a-button key="back" @click="confirmImport">确定</a-button>
      <a-button key="back" @click="cancelImport">取消</a-button>
    </template>
    <a-textarea type="textarea" v-model:value="importJsonStr" :rows="8"></a-textarea>
  </a-modal>
</template>
<script lang="ts" setup>
  import { PlusOutlined, EditOutlined, DeleteOutlined } from '@ant-design/icons-vue'
  import { message } from 'ant-design-vue'
  import { ref, onMounted } from 'vue'
  //#region 导入JSON
  const importModalVisible = ref(false)
  const importJsonStr = ref('')
  const importJson = () => {
    importModalVisible.value = true
    importJsonStr.value = ''
  }
  const confirmImport = () => {
    const jsonStr = importJsonStr.value
    if (!jsonStr) return
    treeData.value = []
    if (isJsonString(jsonStr)) {
      const jsonObj = JSON.parse(jsonStr)
      if (Array.isArray(jsonObj)) {
        jsonObj.forEach((j:any) => {
          let treeNodes = jsonToTreeNode(j)
          treeData.value.push(treeNodes)
        })
      } else {
        let treeNodes = jsonToTreeNode(jsonObj)
        treeData.value.push(treeNodes)
      }
      importModalVisible.value = false
    }
  }

  const isJsonString = (str: string) => {
    try {
      JSON.parse(str)
      return true
    } catch {
      message.warning('无法转成JSON')
      return false
    }
  }
  const cancelImport = () => {
    importModalVisible.value = false
  }
  
  let timeSnap = new Date().getTime()
  const jsonToTreeNode = (jsonObj: any) => {
    ++timeSnap
    const keys = Object.keys(jsonObj)
    if (keys.length > 1) {
      let children:any = []
      keys.forEach(k => {
        ++timeSnap
        const val = jsonObj[k]
        let child = {
          title: k,
          key: timeSnap,
          value: '',
          children: [] as any
        }
        if (isJson(val) && !Array.isArray(val)) {
          const treeNodes = jsonToTreeNode(val)
          if (Array.isArray(treeNodes)) {
            child.children = treeNodes
          } else {
            child.children.push(treeNodes)
          }
        } else {
          child.value = val
        }
        children.push(child)
      })
      return children
  
    } else if (keys.length === 1) {
      let treeNode: any = {}
      const val = jsonObj[keys[0]]
      treeNode.title = keys[0]
      treeNode.key = timeSnap
      treeNode.children = []
  
      if (isJson(val) && !Array.isArray(val)) {
        const res = jsonToTreeNode(val)
        if (Array.isArray(res)) {
          treeNode.children = res
        } else {
          treeNode.children.push(res)
        }
      } else {
        treeNode.value = jsonObj[keys[0]]
      }
      return treeNode
    }
  }
  //#endregion
  const addRootJson = () => {
    addRootNode('root')
  }
  const treeData = ref([] as any)
  let activeKey = ''
  let isAdd = true
  let hasChildren = ref(false)
  
  const addRootNode = (key: string) => {
    cancelNode()
    activeKey = key
    isAdd = true
    hasChildren.value = false
    visible.value = true
  }
  const editNode = (title: string, key: string, value: string, children: any) => {
    cancelNode()
    activeKey = key
    isAdd = false
    if (children && children.length > 0) {
      hasChildren.value = true
    } else {
      hasChildren.value = false
    }
    nodeAttrs.value.title = title
    nodeAttrs.value.value = value
    nodeAttrs.value.type = 'string'
    if (isJson(value)) {
      nodeAttrs.value.value = JSON.stringify(value)
      nodeAttrs.value.type = 'json'
    } else if (isNumber(value)) {
      nodeAttrs.value.type = 'number'
    }
    visible.value = true
  }
  
  const isJson = (jsonObj: string) => {
    try {
      if (typeof jsonObj == "object") {
        return true
      }
    } catch(e) {}
    return false
  }
  const isNumber = (str: string) => {
    try {
      const isNumber = Number(str).toString()
      if (isNumber === 'NaN') {
        return false
      }
    } catch(e) {}
    return true
  }
  
  const cancelNode = () => {
    activeKey = ''
    visible.value = false
    nodeAttrs.value.title = ''
    nodeAttrs.value.value = ''
    nodeAttrs.value.type = 'string'
  }
  
  const confirmNode = () => {
    if (nodeAttrs.value.title === '') {
      message.warning('Key is required!')
      return
    }
    if (nodeAttrs.value.type === 'number') {
      const isNumber = Number(nodeAttrs.value.value).toString()
      if (isNumber === 'NaN') {
        message.warning('无法转成数字')
        return 
      } else {
        nodeAttrs.value.value = Number(nodeAttrs.value.value)
      }
    } else if (nodeAttrs.value.type === 'json') {
      try {
        nodeAttrs.value.value = JSON.parse(nodeAttrs.value.value)
      } catch {
        message.warning('无法转成JSON')
        return
      }
    } else if (nodeAttrs.value.type === 'string') {
      nodeAttrs.value.value = nodeAttrs.value.value.toString()
    }
    
    if (isAdd) {
      confirmAddNode()
    } else {
      confirmEditNode()
    }
  }
  //#region 新增节点
  const visible = ref(false)
  const nodeAttrs = ref({
    title: '',
    value: '',
    type: 'string'
  })
  
  const confirmAddNode = () => {
    ++timeSnap
    if (activeKey === 'root') {
      treeData.value.push({
        title: nodeAttrs.value.title,
        value: nodeAttrs.value.value,
        key: timeSnap,
        children: []
      })
      visible.value = false
      return
    }
  
  
    var treeDataOld = JSON.parse(JSON.stringify(treeData.value))
    let treeDataNew = aNode(activeKey, treeDataOld)
    treeData.value = treeDataNew
    function aNode(key:any, arr:any) {
      arr.map((item:any, index:any) => {
        if (item.key === key) {
          if (item.children) {
            item.children.push({
              ...nodeAttrs.value,
              key: new Date().getTime()
            })
          } else {
            item.children = [{
              ...nodeAttrs.value,
              key: new Date().getTime()
            }]
          }
          visible.value = false
        }
        if (item.children) {
          aNode(key, item.children)
        }
      })
      return arr
    }
  }
  //#endregion
  const confirmEditNode = () => {
    var treeDataOld = JSON.parse(JSON.stringify(treeData.value))
    let treeDataNew = eNode(activeKey, treeDataOld)
    treeData.value = treeDataNew
    function eNode(key:any, arr:any) {
      arr.map((item:any, index:any) => {
        if (item.key === key) {
          item.title = nodeAttrs.value.title
          item.value = nodeAttrs.value.value
          visible.value = false
        }
        if (item.children) {
          eNode(key, item.children)
        }
      })
      return arr
    }
  }
  const delNode = (title: string, key: string, children: any) => {
    var treeDataOld = JSON.parse(JSON.stringify(treeData.value))
    var treeDataNew = deleteNode(key, treeDataOld)
    treeData.value = treeDataNew
  
    function deleteNode(key:any, arr:any) {
      arr.map((item:any, index:any) => {
        if (item.key === key) {
          arr.splice(index, 1)
        }
        if (item.children) {
          deleteNode(key, item.children)
        }
      })
      return arr
    }
  }
  
  const finalResult = ref()
  const toFinalJson = () => {
    var treeDataOld = JSON.parse(JSON.stringify(treeData.value))
    let res:any = []
    treeDataOld.forEach((tr: any) => {
      res.push({
        [tr.title]:finalJson(tr)
      })
    })
    finalResult.value = res
    function finalJson(js: any) {
      let tmpNode: any = {}
      if (js.children && js.children.length > 0) {
        js.children.forEach((c: any) => {
          tmpNode[c.title] = finalJson(c)
        })
      } else {
        tmpNode = js.value
      }
      return tmpNode
    }
  }
</script>
<style scoped>
</style>
