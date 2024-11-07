<template>
  <v-container class="fill-height">
    <v-responsive class="align-centerfill-height mx-auto">
      <!-- <v-img class="mb-4" height="150" src="@/assets/logo.png" /> -->

      <div class="text-center">
        <div class="text-body-2 font-weight-light mb-n1">Welcome to</div>

        <h1 class="text-h2 font-weight-bold">Medical Relation Graph</h1>
      </div>

      <div class="py-4" />

      <v-row>
        <v-col cols="12">
          <v-text-field clearable label="keyword" prepend-icon="$vuetify" v-model="keyWord"></v-text-field>
          <v-btn @click="showAll">
            获取25个节点
          </v-btn>
          <v-btn @click="onClick">
            搜索疾病
          </v-btn>
        </v-col>
      </v-row>
      <v-row>
        <div style="border: #efefef solid 1px; height: calc(100vh - 100px);width: 100%;">
          <relation-graph ref="graphRef$" :options="options" />
        </div>

      </v-row>
    </v-responsive>
  </v-container>
</template>

<script setup>
//
import { onMounted, ref } from 'vue';
import neo4j from 'neo4j-driver'
import RelationGraph from 'relation-graph-vue3'
const graphRef$ = ref()
const options = {
  defaultExpandHolderPosition: 'right',
  layout: {
      layoutName: 'force'
    },
}
var jsonData = ref()
jsonData = {
  rootId: 'a',
  nodes: [
  ],
  lines: [
  ],
}
var keyWord = ref()
var driver = neo4j.driver(
  'neo4j://localhost',
  neo4j.auth.basic('neo4j', 'neo4j')
)


onMounted(() => {

  graphRef$.value.setJsonData(jsonData)


})

const onClick = async () => {
  console.log('search keyword: ' + keyWord.value)
  // var prompt = `MATCH (n:Disease{name:"${keyWord.value}"}) RETURN n LIMIT 50`
  var prompt = `MATCH p = (:Disease{name:"${keyWord.value}"})-[]->() RETURN p`
  neo4jRead(prompt)

}

const showAll = () => {
  console.log('search keyword: ' + keyWord.value)
  var prompt = "MATCH (n:Disease) RETURN n LIMIT 25"
  neo4jRead(prompt)

}

//根据文本生成颜色
const stringToColor = (str) => {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = str.charCodeAt(i) + ((hash << 5) - hash);
  }
  let color = '#';
  for (let i = 0; i < 3; i++) {
    let value = (hash >> (i * 8)) & 0xFF;
    color += ('00' + value.toString(16)).substr(-2);
  }
  return color;
}

//使用固定配色方案
const suedColor = (str) => {
  var colors = ['#D6AFBC', '#D0A1C2', '#A991D2', '#554381', '#CB85AD', '#D4959A', '#CC4882']
  return colors[str.length % colors.length]
}


// neo4j 结果转换赋值 通用方法
const neo4jResultConvert = (result) => {
  result.forEach(element => {
    //每个记录内是一个点或者一条线
    element._fields.forEach(field => {
      if (field.__proto__.__isPath__) {
        jsonData.lines.push({ from: field.start.elementId, to: field.end.elementId, text: field.segments[0].relationship.properties.name, animation: 1 })
        if (jsonData.nodes.findIndex(item => item.id === field.start.elementId) === -1) {
          jsonData.nodes.push({ id: field.start.elementId, text: field.start.properties.name, color: stringToColor(field.start.labels[0]) })
        }
        if (jsonData.nodes.findIndex(item => item.id === field.end.elementId) === -1) {
          jsonData.nodes.push({ id: field.end.elementId, text: field.end.properties.name, color: stringToColor(field.end.labels[0]) })
        }
      } else if(field.__proto__.__isNode__){
        if (jsonData.nodes.findIndex(item => item.id === field.elementId) === -1) {
          jsonData.nodes.push({ id: field.elementId, text: field.properties.name, color: stringToColor(field.labels[0]) })
        }

      }else {
        
      }
    })
  });
  console.log('jsonData', jsonData)
  graphRef$.value.setJsonData(jsonData)
}

// neo4j read 通用方法
const neo4jRead = (prompt) => {
  console.log('neo4jRead')
  var session = driver.session();
  var readTxResultPromise = session.executeRead(txc => {
    // used transaction will be committed automatically, no need for explicit commit/rollback
    var result = txc.run(prompt)
    // at this point it is possible to either return the result or process it and return the
    // result of processing it is also possible to run more statements in the same transaction
    return result
  })

  // returned Promise can be later consumed like this:
  readTxResultPromise
    .then(result => {
      console.log('result.records')
      console.log(result.records)
      neo4jResultConvert(result.records)
      return result.records;
    })
    .catch(error => {
      console.log('error')
      console.log(error)
      return error;
    })
    .then(() => session.close())

}
</script>
