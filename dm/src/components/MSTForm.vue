<script setup>
import { reactive, ref, onMounted } from "vue";
import axios from "@/utlis/axios";

const emit = defineEmits(["getInputForm", "getResponse", "lujingUpdated"]); //定义宏事件
const inputForm = reactive({ count: 0, node: [], edges: [] }); // 输入表单
const transferForm = reactive({ count: 0, node: [], link: [] }); // 组件传输表单
let nodeSelected = ref();
// let LJ = ['AC', 'CF', 'DF', 'BC', 'BE'];
let LJ;
const minCount = 3;
const ruleForm = ref();
const rules = reactive({
  count: [{ required: true }],
  node: [{ required: true, message: "请输入节点", trigger: "change" }],
});

// 根据节点个数显示节点信息
const showNode = (n) => {
  inputForm.node = [];
  for (let i = 0; i < n; i++) {
    inputForm.node.push(String.fromCharCode(65 + i));
  }
};

function changeNode(value) {
  inputForm.node = value.split(",").map((v) => v.trim());
}

function removeEdge(index) {
  inputForm.edges.splice(index, 1)
}

const newEdgeForm = reactive({
  node1: '',
  node2: '',
  weight: 1
})

// 提交表单信息
const submitForm = async (ruleForm) => {
  if (!ruleForm) return;
  await ruleForm.validate((valid) => {
    if (valid) {
      transferForm.count = inputForm.count;
      transferForm.node = inputForm.node;
      // 解析路径信息
      transferForm.link = inputForm.edges.map((e) => [
        e.node1,
        e.node2,
        e.weight,
      ]);
      // 向后端发送请求
      axios
        .post("/scs", {
          num: inputForm.count,
          powerTable: createPowerTable(),
        })
        .then((response) => {
          console.log(response.data);
          LJ = response.data;
        });

      // 传送表单信息
      emit("getInputForm", transferForm);
      emit("lujingUpdated", LJ);
    }
  });
};

// 重置输入框
function resetForm(ruleForm) {
  if (!ruleForm) return
  ruleForm.resetFields()

  showNode(minCount)
  inputForm.edges = []
  newEdgeForm.node1 = ''
  newEdgeForm.node2 = ''
  newEdgeForm.weight = 1
}

// 生成权重表格
const createPowerTable = () => {
  let powerTable = new Array();
  inputForm.node.forEach((col) => {
    let rowArr = new Array();
    inputForm.node.forEach((row) => {
      if (col == row) {
        rowArr.push(0);
      } else {
        let power = -1;
        const len = transferForm.link.length;
        for (let i = 0; i < len; i++) {
          const link = transferForm.link[i];
          if (
            (col == link[0] && row == link[1]) ||
            (col == link[1] && row == link[0])
          ) {
            power = +link[2];
            break;
          }
        }
        rowArr.push(power);
      }
    });
    powerTable.push(rowArr);
  });
  return powerTable;
};

function addEdge() {
  if (!newEdgeForm.node1 || !newEdgeForm.node2) {
    alert('请选择起点和终点')
    return
  }
  if (newEdgeForm.node1 === newEdgeForm.node2) {
    alert('起点和终点不能相同')
    return
  }
  if (newEdgeForm.weight <= 0) {
    alert('权值必须大于0')
    return
  }

  // 插入 edges
  inputForm.edges.push({
    node1: newEdgeForm.node1,
    node2: newEdgeForm.node2,
    weight: newEdgeForm.weight
  })

  // 重置
  newEdgeForm.node1 = ''
  newEdgeForm.node2 = ''
  newEdgeForm.weight = 1
}

onMounted(() => {
  // 节点信息初始化
  showNode(minCount);
});
</script>

<template>
  <el-form
    ref="ruleForm"
    :model="inputForm"
    :rules="rules"
    class="inputArea"
  >
    <!-- 基本信息 -->
    <el-form-item label="节点个数" prop="count">
      <el-input-number
        v-model="inputForm.count"
        :min="minCount"
        :max="10"
        @change="showNode"
      />
    </el-form-item>
    <el-form-item label="节点信息" prop="node">
      <el-input
        v-model.trim="inputForm.node"
        @change="changeNode"
      />
    </el-form-item>

    <!-- 添加边:-->
    <div class="add-edge-inline">
      <span>起点</span>
      <el-select v-model="newEdgeForm.node1" placeholder="起点" style="width: 70px;">
        <el-option
          v-for="(n, idx) in inputForm.node"
          :key="idx"
          :label="n"
          :value="n"
        />
      </el-select>

      <span>终点</span>
      <el-select v-model="newEdgeForm.node2" placeholder="终点" style="width: 70px;">
        <el-option
          v-for="(n, idx) in inputForm.node"
          :key="idx"
          :label="n"
          :value="n"
        />
      </el-select>

      <span>权值</span>
      <el-input-number
        v-model="newEdgeForm.weight"
        :min="1"
        style="width: 100px;"
      />

      <el-button type="primary" @click="addEdge">
        添加
      </el-button>
    </div>

    <!-- 已添加的边：容器固定高度并可滚动 -->
    <div class="edges-list">
  <div
    v-for="(edge, index) in inputForm.edges"
    :key="index"
    class="edge-item"
  >
    <!-- 显示边信息 -->
    <span class="edge-text">
      {{ edge.node1 }} - {{ edge.node2 }} : {{ edge.weight }}
    </span>
    <!-- 删除按钮 -->
    <el-button
      type="danger"
      size="small"
      @click="removeEdge(index)"
    >
      删除
    </el-button>
  </div>
</div>
    <!-- 操作按钮 -->
    <div class="buttons">
      <el-form-item>
        <el-button type="primary" @click="submitForm(ruleForm)">生成</el-button>
        <el-button @click="resetForm(ruleForm)">重置</el-button>
      </el-form-item>
    </div>
  </el-form>
</template>
  
<style scoped>
.inputArea {
  width: 500px;
  border: 1px solid #666;
  border-radius: 8px;
  padding: 15px;
}

.add-edge-inline {
  display: flex;
  align-items: center;
  gap: 6px;
  margin: 10px 0;
}

.edges-list {
  /* 固定或最大高度，超出可滚动 */
  max-height: 70px;      
  overflow-y: auto;
  display: grid;
  grid-template-columns: 1fr 1fr;  
  gap: 8px;                      
}

.edge-item {
  display: flex;               
  align-items: center;
  justify-content: space-between;
  padding: 4px;
  border: 1px solid #eee;      
  border-radius: 4px;
}

:deep(.elTextarea textarea) {
  height: 100px;
  resize: none;
}

.edge-text {
  margin-right: 8px;
}

.buttons {
  display: flex;
  justify-content: space-between;
  padding: 0 10px;
  margin-top: 10px;
}

.selectNode {
  width: 100px;
  margin: 0 8px;
}
</style>
