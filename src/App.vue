<template>
  <div class="devjson-app">
    <header class="app-header">
      <div>
        <div class="brand">DevJSON</div>
        <div class="subtitle">程序员 JSON / Go / Proto 转换插件</div>
      </div>
    </header>

    <nav class="tabs">
      <button
        v-for="tab in tabs"
        :key="tab.key"
        :class="['tab-btn', { active: currentTool === tab.key }]"
        @click="switchTool(tab.key)"
      >
        {{ tab.name }}
      </button>
    </nav>

    <section class="toolbar">
      <button class="primary-btn" @click="runTool">执行</button>
      <button @click="formatJson">格式化</button>
      <button @click="compressJson">压缩</button>
      <button @click="copyResult">复制结果</button>
      <button @click="setExampleByTool">示例</button>
      <button class="danger-btn" @click="clearAll">清空</button>
    </section>

    <main class="workspace">
      <section class="panel">
        <div class="panel-title">
          <span>{{ inputTitle }}</span>
          <span>{{ inputInfo }}</span>
        </div>
        <textarea v-model="inputText" :placeholder="placeholder"></textarea>
      </section>

      <section class="panel">
        <div class="panel-title">
          <span>{{ resultTitle }}</span>
          <span :class="['status', statusType]">{{ statusText }}</span>
        </div>

        <div v-if="currentTool === 'path'" class="result-box json-tree">
          <JsonTree v-if="parsedJson !== null" :data="parsedJson" path="" @select-path="setPath" />
          <div v-else class="empty">请先输入 JSON 后执行</div>
        </div>

        <pre v-else class="result-box">{{ resultText }}</pre>
      </section>

      <aside class="side-panel">
        <div class="card">
          <div class="side-title">当前路径</div>
          <div class="path-value">{{ currentPath || '暂无' }}</div>
          <button class="small-btn" @click="copyPath">复制路径</button>
        </div>

        <div class="card">
          <div class="side-title-row">
            <span>转换设置</span>
          </div>

          <label class="field-label">结构体 / Message 名称</label>
          <input v-model="rootName" class="setting-input" placeholder="Response" />

          <label class="field-label">Proto Package</label>
          <input v-model="protoPackage" class="setting-input" placeholder="api.v1" />

          <div class="tip">
            反转说明：Go Struct / Proto 反转为 JSON 会生成字段默认值示例，适合快速构造请求体。
          </div>
        </div>

        <div class="card history-card">
          <div class="side-title-row">
            <span>本地历史</span>
            <button class="small-btn no-margin" @click="clearHistory">清空</button>
          </div>

          <div v-if="historyList.length === 0" class="empty">暂无历史</div>
          <div
            v-for="(item, index) in historyList"
            :key="index"
            class="history-item"
            @click="loadHistory(item)"
          >
            <div>{{ item.type }} / {{ item.time }}</div>
            <p>{{ item.content.slice(0, 60) }}{{ item.content.length > 60 ? '...' : '' }}</p>
          </div>
        </div>
      </aside>
    </main>
  </div>
</template>

<script setup>
import { computed, defineComponent, h, onMounted, ref } from 'vue'

const HISTORY_KEY = 'devjson_history_v2'

const tabs = [
  { key: 'format', name: '格式化' },
  { key: 'path', name: '路径提取' },
  { key: 'go', name: 'JSON转GoStruct' },
  { key: 'proto', name: 'JSON转Proto' },
  { key: 'goReverse', name: 'GoStruct转JSON' },
  { key: 'protoReverse', name: 'Proto转JSON' },
  { key: 'diff', name: 'JSON Diff' },
  { key: 'ts', name: 'JSON转TS Interface' },
  { key: 'java', name: 'JSON转Java Class' },
  { key: 'csharp', name: 'JSON转C# Model' },
  { key: 'python', name: 'JSON转Python Dataclass' }
]

const currentTool = ref('format')
const inputText = ref('')
const resultText = ref('')
const parsedJson = ref(null)
const currentPath = ref('')
const statusText = ref('')
const statusType = ref('')
const historyList = ref([])
const inputInfo = ref('')
const rootName = ref('Response')
const protoPackage = ref('api.v1')

const inputTitle = computed(() => {
  if (currentTool.value === 'goReverse') return '输入 Go Struct'
  if (currentTool.value === 'protoReverse') return '输入 Proto Message'
  if (currentTool.value === 'diff') return '输入两个 JSON，中间用 --- 分隔'
  return '输入 JSON'
})

const placeholder = computed(() => {
  if (currentTool.value === 'goReverse') return '请粘贴 Go Struct，例如：type Response struct { Code int `json:"code"` }'
  if (currentTool.value === 'protoReverse') return '请粘贴 proto message，例如：message Response { int32 code = 1; }'
  if (currentTool.value === 'diff') return '{"name":"a"}\n---\n{"name":"b"}'
  return '请粘贴 JSON 数据...'
})

const resultTitle = computed(() => {
  const map = {
    format: '格式化结果',
    path: 'JSON 树 / 点击字段生成路径',
    go: 'Go Struct 结果',
    proto: 'Proto 结果',
    goReverse: 'JSON 示例',
    protoReverse: 'JSON 示例',
    diff: 'Diff 结果',
    ts: 'TypeScript Interface',
    java: 'Java Class',
    csharp: 'C# Model',
    python: 'Python Dataclass'
  }
  return map[currentTool.value] || '结果'
})

const JsonTree = defineComponent({
  name: 'JsonTree',
  props: {
    data: { type: null, required: true },
    path: { type: String, default: '' }
  },
  emits: ['select-path'],
  setup(props, { emit }) {
    const renderValue = (value, path) => {
      if (Array.isArray(value)) {
        return h('div', { class: 'tree-block' }, [
          h('span', '['),
          h('ul', value.map((item, index) => {
            const itemPath = path ? `${path}[${index}]` : `[${index}]`
            return h('li', [
              h('span', {
                class: 'json-node json-key',
                onClick: () => emit('select-path', itemPath)
              }, `[${index}]`),
              h('span', ': '),
              renderValue(item, itemPath)
            ])
          })),
          h('span', ']')
        ])
      }

      if (value && typeof value === 'object') {
        return h('div', { class: 'tree-block' }, [
          h('span', '{'),
          h('ul', Object.keys(value).map((key) => {
            const childPath = path ? `${path}.${key}` : key
            return h('li', [
              h('span', {
                class: 'json-node json-key',
                onClick: () => emit('select-path', childPath)
              }, key),
              h('span', ': '),
              renderValue(value[key], childPath)
            ])
          })),
          h('span', '}')
        ])
      }

      const typeClass = value === null ? 'json-null' : `json-${typeof value}`
      return h('span', {
        class: ['json-node', typeClass],
        onClick: () => emit('select-path', path)
      }, JSON.stringify(value))
    }

    return () => renderValue(props.data, props.path)
  }
})

onMounted(() => {
  loadHistoryList()
  setExampleByTool()
})

function switchTool(tool) {
  currentTool.value = tool
  currentPath.value = ''
  resultText.value = ''
  parsedJson.value = null
  setExampleByTool()
}

function parseJson(text = inputText.value) {
  if (!text.trim()) throw new Error('请输入 JSON 数据')
  return JSON.parse(text)
}

function runTool() {
  try {
    if (currentTool.value === 'format') return runFormat()
    if (currentTool.value === 'path') return runPath()
    if (currentTool.value === 'go') return runGoStruct()
    if (currentTool.value === 'proto') return runProto()
    if (currentTool.value === 'goReverse') return runGoStructReverse()
    if (currentTool.value === 'protoReverse') return runProtoReverse()
    if (currentTool.value === 'diff') return runDiff()
    if (currentTool.value === 'ts') return runTs()
    if (currentTool.value === 'java') return runJava()
    if (currentTool.value === 'csharp') return runCsharp()
    if (currentTool.value === 'python') return runPython()
  } catch (error) {
    showError(error)
  }
}

function runFormat() {
  const obj = parseJson()
  parsedJson.value = obj
  resultText.value = JSON.stringify(obj, null, 2)
  updateInputInfo(obj)
  saveHistory('format', inputText.value)
  setStatus('执行成功', 'success')
}

function formatJson() {
  try {
    const obj = parseJson()
    inputText.value = JSON.stringify(obj, null, 2)
    resultText.value = inputText.value
    updateInputInfo(obj)
    saveHistory('format', inputText.value)
    setStatus('格式化成功', 'success')
  } catch (error) {
    showError(error)
  }
}

function compressJson() {
  try {
    const obj = parseJson()
    inputText.value = JSON.stringify(obj)
    resultText.value = inputText.value
    updateInputInfo(obj)
    saveHistory('compress', inputText.value)
    setStatus('压缩成功', 'success')
  } catch (error) {
    showError(error)
  }
}

function runPath() {
  const obj = parseJson()
  parsedJson.value = obj
  resultText.value = ''
  updateInputInfo(obj)
  saveHistory('path', inputText.value)
  setStatus('点击字段生成路径', 'success')
}

function runGoStruct() {
  const obj = parseJson()
  parsedJson.value = obj
  resultText.value = generateGoStruct(obj, rootName.value || 'Response')
  updateInputInfo(obj)
  saveHistory('json-to-go', inputText.value)
  setStatus('Go Struct 生成成功', 'success')
}

function runProto() {
  const obj = parseJson()
  parsedJson.value = obj
  resultText.value = generateProto(obj, rootName.value || 'Response', protoPackage.value || '')
  updateInputInfo(obj)
  saveHistory('json-to-proto', inputText.value)
  setStatus('Proto 生成成功', 'success')
}

function runGoStructReverse() {
  if (!inputText.value.trim()) throw new Error('请输入 Go Struct')
  const json = goStructToJsonExample(inputText.value)
  resultText.value = JSON.stringify(json, null, 2)
  saveHistory('go-to-json', inputText.value)
  setStatus('Go Struct 反转成功', 'success')
}

function runProtoReverse() {
  if (!inputText.value.trim()) throw new Error('请输入 Proto Message')
  const json = protoToJsonExample(inputText.value)
  resultText.value = JSON.stringify(json, null, 2)
  saveHistory('proto-to-json', inputText.value)
  setStatus('Proto 反转成功', 'success')
}

function generateGoStruct(obj, name) {
  if (Array.isArray(obj)) {
    const first = obj[0]
    if (first && typeof first === 'object' && !Array.isArray(first)) {
      return `type ${name} []struct {\n${generateGoFields(first, 1)}\n}`
    }
    return `type ${name} []${inferGoType(first)}`
  }
  return `type ${name} struct {\n${generateGoFields(obj, 1)}\n}`
}

function generateGoFields(obj, level) {
  if (!obj || typeof obj !== 'object' || Array.isArray(obj)) return ''
  const indent = '    '.repeat(level)

  return Object.keys(obj).map((key) => {
    const fieldName = toGoFieldName(key)
    const value = obj[key]
    const tag = '`json:"' + key + '"`'

    if (Array.isArray(value)) {
      if (value.length > 0 && value[0] && typeof value[0] === 'object' && !Array.isArray(value[0])) {
        return `${indent}${fieldName} []struct {\n${generateGoFields(value[0], level + 1)}\n${indent}} ${tag}`
      }
      return `${indent}${fieldName} []${inferGoType(value[0])} ${tag}`
    }

    if (value && typeof value === 'object') {
      return `${indent}${fieldName} struct {\n${generateGoFields(value, level + 1)}\n${indent}} ${tag}`
    }

    return `${indent}${fieldName} ${inferGoType(value)} ${tag}`
  }).join('\n')
}

function inferGoType(value) {
  if (value === null || value === undefined) return 'interface{}'
  if (typeof value === 'string') return 'string'
  if (typeof value === 'boolean') return 'bool'
  if (typeof value === 'number') return Number.isInteger(value) ? 'int' : 'float64'
  if (Array.isArray(value)) return '[]interface{}'
  if (typeof value === 'object') return 'interface{}'
  return 'interface{}'
}

function generateProto(obj, name, packageName = '') {
  const messages = []
  const rootObj = Array.isArray(obj) ? (obj[0] || {}) : obj
  buildProtoMessage(rootObj, name, messages)

  const header = [
    'syntax = "proto3";',
    packageName ? `package ${packageName};` : '',
    ''
  ].filter(Boolean).join('\n')

  return `${header}\n${messages.reverse().join('\n\n')}`
}

function buildProtoMessage(obj, name, messages) {
  if (!obj || typeof obj !== 'object' || Array.isArray(obj)) return name

  const lines = [`message ${name} {`]
  let index = 1

  Object.keys(obj).forEach((key) => {
    const value = obj[key]
    const fieldName = toSnakeCase(key)

    if (Array.isArray(value)) {
      const first = value[0]
      if (first && typeof first === 'object' && !Array.isArray(first)) {
        const childName = toGoFieldName(key) + 'Item'
        buildProtoMessage(first, childName, messages)
        lines.push(`  repeated ${childName} ${fieldName} = ${index};`)
      } else {
        lines.push(`  repeated ${inferProtoType(first)} ${fieldName} = ${index};`)
      }
    } else if (value && typeof value === 'object') {
      const childName = toGoFieldName(key)
      buildProtoMessage(value, childName, messages)
      lines.push(`  ${childName} ${fieldName} = ${index};`)
    } else {
      lines.push(`  ${inferProtoType(value)} ${fieldName} = ${index};`)
    }

    index += 1
  })

  lines.push('}')
  messages.push(lines.join('\n'))
  return name
}

function inferProtoType(value) {
  if (value === null || value === undefined) return 'string'
  if (typeof value === 'string') return 'string'
  if (typeof value === 'boolean') return 'bool'
  if (typeof value === 'number') return Number.isInteger(value) ? 'int64' : 'double'
  return 'string'
}

function goStructToJsonExample(code) {
  const bodyMatch = code.match(/struct\s*\{([\s\S]*)\}/)
  const body = bodyMatch ? bodyMatch[1] : code
  const result = {}
  const lines = body.split('\n').map((line) => line.trim()).filter(Boolean)

  lines.forEach((line) => {
    if (line === '}' || line.startsWith('//')) return
    if (line.includes('struct {')) return

    const tagMatch = line.match(/`json:"([^",]+).*?"`/)
    const parts = line.replace(/`[^`]*`/g, '').trim().split(/\s+/)
    if (parts.length < 2) return

    const fieldName = tagMatch ? tagMatch[1] : toSnakeCase(parts[0])
    const fieldType = parts[1]
    result[fieldName] = goTypeDefaultValue(fieldType)
  })

  return result
}

function goTypeDefaultValue(type) {
  if (type.startsWith('[]')) return []
  if (type === 'string') return ''
  if (type === 'bool') return false
  if (['int', 'int8', 'int16', 'int32', 'int64', 'uint', 'uint8', 'uint16', 'uint32', 'uint64'].includes(type)) return 0
  if (['float32', 'float64'].includes(type)) return 0
  if (type === 'interface{}' || type === 'any') return null
  return {}
}

function protoToJsonExample(proto) {
  const normalized = proto.replace(/\r\n/g, '\n')
  const messageMatch = normalized.match(/message\s+\w+\s*\{([\s\S]*?)\n\}/)
  const body = messageMatch ? messageMatch[1] : normalized
  const result = {}
  const lines = body.split('\n').map((line) => line.trim()).filter(Boolean)

  lines.forEach((line) => {
    if (line.startsWith('//') || line.startsWith('message ') || line === '}') return
    const clean = line.replace(/;$/, '')
    const match = clean.match(/^(repeated\s+)?([\w.]+)\s+(\w+)\s*=\s*\d+/)
    if (!match) return

    const isRepeated = Boolean(match[1])
    const type = match[2]
    const field = match[3]
    result[field] = isRepeated ? [] : protoTypeDefaultValue(type)
  })

  return result
}

function protoTypeDefaultValue(type) {
  if (type === 'string') return ''
  if (type === 'bool') return false
  if (['int32', 'int64', 'uint32', 'uint64', 'sint32', 'sint64', 'fixed32', 'fixed64', 'sfixed32', 'sfixed64', 'double', 'float'].includes(type)) return 0
  if (type === 'bytes') return ''
  return {}
}

function runTs() {
  const obj = parseJson()
  resultText.value = generateTsInterface(obj, rootName.value || 'Response')
  updateInputInfo(obj)
  saveHistory('json-to-ts', inputText.value)
  setStatus('TypeScript Interface 生成成功', 'success')
}

function runJava() {
  const obj = parseJson()
  resultText.value = generateJavaClass(obj, rootName.value || 'Response')
  updateInputInfo(obj)
  saveHistory('json-to-java', inputText.value)
  setStatus('Java Class 生成成功', 'success')
}

function runCsharp() {
  const obj = parseJson()
  resultText.value = generateCsharpModel(obj, rootName.value || 'Response')
  updateInputInfo(obj)
  saveHistory('json-to-csharp', inputText.value)
  setStatus('C# Model 生成成功', 'success')
}

function runPython() {
  const obj = parseJson()
  resultText.value = generatePythonDataclass(obj, rootName.value || 'Response')
  updateInputInfo(obj)
  saveHistory('json-to-python', inputText.value)
  setStatus('Python Dataclass 生成成功', 'success')
}

function toCamelCase(key) {
  const pascal = toGoFieldName(key)
  return pascal.charAt(0).toLowerCase() + pascal.slice(1)
}

// ── TypeScript Interface ──

function generateTsInterface(obj, name) {
  const interfaces = []
  buildTsInterface(obj, name, interfaces)
  return interfaces.reverse().join('\n\n')
}

function buildTsInterface(obj, name, interfaces) {
  const rootObj = Array.isArray(obj) ? (obj[0] || {}) : obj
  if (!rootObj || typeof rootObj !== 'object' || Array.isArray(rootObj)) return

  const lines = [`interface ${name} {`]
  Object.keys(rootObj).forEach((key) => {
    lines.push(`  ${key}: ${inferTsType(rootObj[key], key, interfaces)};`)
  })
  lines.push('}')
  interfaces.push(lines.join('\n'))
}

function inferTsType(value, key, interfaces) {
  if (value === null || value === undefined) return 'null'
  if (typeof value === 'string') return 'string'
  if (typeof value === 'boolean') return 'boolean'
  if (typeof value === 'number') return 'number'
  if (Array.isArray(value)) {
    if (value.length > 0 && value[0] && typeof value[0] === 'object' && !Array.isArray(value[0])) {
      const childName = toGoFieldName(key) + 'Item'
      buildTsInterface(value[0], childName, interfaces)
      return childName + '[]'
    }
    if (value.length > 0) return inferTsType(value[0], key, interfaces) + '[]'
    return 'any[]'
  }
  if (typeof value === 'object') {
    const childName = toGoFieldName(key)
    buildTsInterface(value, childName, interfaces)
    return childName
  }
  return 'any'
}

// ── Java Class ──

function generateJavaClass(obj, name) {
  const classes = []
  buildJavaClass(obj, name, classes)
  const imports = 'import com.fasterxml.jackson.annotation.JsonProperty;\nimport java.util.List;\n'
  return imports + '\n' + classes.reverse().join('\n\n')
}

function buildJavaClass(obj, name, classes) {
  const rootObj = Array.isArray(obj) ? (obj[0] || {}) : obj
  if (!rootObj || typeof rootObj !== 'object' || Array.isArray(rootObj)) return

  const fields = []
  const methods = []

  Object.keys(rootObj).forEach((key) => {
    const value = rootObj[key]
    const javaType = inferJavaType(value, key, classes)
    const fieldName = toCamelCase(key)
    const capName = fieldName.charAt(0).toUpperCase() + fieldName.slice(1)

    fields.push(`    @JsonProperty("${key}")`)
    fields.push(`    private ${javaType} ${fieldName};`)
    fields.push('')
    methods.push(`    public ${javaType} get${capName}() { return ${fieldName}; }`)
    methods.push(`    public void set${capName}(${javaType} val) { this.${fieldName} = val; }`)
  })

  const lines = [`public class ${name} {`, '', ...fields, ...methods, '}']
  classes.push(lines.join('\n'))
}

function inferJavaType(value, key, classes) {
  if (value === null || value === undefined) return 'Object'
  if (typeof value === 'string') return 'String'
  if (typeof value === 'boolean') return 'Boolean'
  if (typeof value === 'number') return Number.isInteger(value) ? 'Integer' : 'Double'
  if (Array.isArray(value)) {
    if (value.length > 0 && value[0] && typeof value[0] === 'object' && !Array.isArray(value[0])) {
      const childName = toGoFieldName(key) + 'Item'
      buildJavaClass(value[0], childName, classes)
      return `List<${childName}>`
    }
    if (value.length > 0) return `List<${inferJavaType(value[0], key, classes)}>`
    return 'List<Object>'
  }
  if (typeof value === 'object') {
    const childName = toGoFieldName(key)
    buildJavaClass(value, childName, classes)
    return childName
  }
  return 'Object'
}

// ── C# Model ──

function generateCsharpModel(obj, name) {
  const classes = []
  buildCsharpClass(obj, name, classes)
  const imports = 'using System.Collections.Generic;\nusing System.Text.Json.Serialization;\n'
  return imports + '\n' + classes.reverse().join('\n\n')
}

function buildCsharpClass(obj, name, classes) {
  const rootObj = Array.isArray(obj) ? (obj[0] || {}) : obj
  if (!rootObj || typeof rootObj !== 'object' || Array.isArray(rootObj)) return

  const lines = [`public class ${name}`, '{']
  Object.keys(rootObj).forEach((key) => {
    const value = rootObj[key]
    const csType = inferCsharpType(value, key, classes)
    const propName = toGoFieldName(key)
    lines.push(`    [JsonPropertyName("${key}")]`)
    lines.push(`    public ${csType} ${propName} { get; set; }`)
    lines.push('')
  })
  lines.push('}')
  classes.push(lines.join('\n'))
}

function inferCsharpType(value, key, classes) {
  if (value === null || value === undefined) return 'object?'
  if (typeof value === 'string') return 'string'
  if (typeof value === 'boolean') return 'bool'
  if (typeof value === 'number') return Number.isInteger(value) ? 'int' : 'double'
  if (Array.isArray(value)) {
    if (value.length > 0 && value[0] && typeof value[0] === 'object' && !Array.isArray(value[0])) {
      const childName = toGoFieldName(key) + 'Item'
      buildCsharpClass(value[0], childName, classes)
      return `List<${childName}>`
    }
    if (value.length > 0) return `List<${inferCsharpType(value[0], key, classes)}>`
    return 'List<object>'
  }
  if (typeof value === 'object') {
    const childName = toGoFieldName(key)
    buildCsharpClass(value, childName, classes)
    return childName
  }
  return 'object'
}

// ── Python Dataclass ──

function generatePythonDataclass(obj, name) {
  const classes = []
  buildPythonDataclass(obj, name, classes)
  return 'from dataclasses import dataclass\nfrom typing import List, Optional, Any\n\n' + classes.join('\n\n')
}

function buildPythonDataclass(obj, name, classes) {
  const rootObj = Array.isArray(obj) ? (obj[0] || {}) : obj
  if (!rootObj || typeof rootObj !== 'object' || Array.isArray(rootObj)) return

  const lines = ['@dataclass', `class ${name}:`]
  Object.keys(rootObj).forEach((key) => {
    lines.push(`    ${key}: ${inferPythonType(rootObj[key], key, classes)}`)
  })
  classes.push(lines.join('\n'))
}

function inferPythonType(value, key, classes) {
  if (value === null || value === undefined) return 'Optional[Any]'
  if (typeof value === 'string') return 'str'
  if (typeof value === 'boolean') return 'bool'
  if (typeof value === 'number') return Number.isInteger(value) ? 'int' : 'float'
  if (Array.isArray(value)) {
    if (value.length > 0 && value[0] && typeof value[0] === 'object' && !Array.isArray(value[0])) {
      const childName = toGoFieldName(key) + 'Item'
      buildPythonDataclass(value[0], childName, classes)
      return `List[${childName}]`
    }
    if (value.length > 0) return `List[${inferPythonType(value[0], key, classes)}]`
    return 'List[Any]'
  }
  if (typeof value === 'object') {
    const childName = toGoFieldName(key)
    buildPythonDataclass(value, childName, classes)
    return childName
  }
  return 'Any'
}

function runDiff() {
  const parts = inputText.value.split('\n---\n')
  if (parts.length < 2) {
    throw new Error('Diff 模式请输入两个 JSON，中间用一行 --- 分隔')
  }

  const left = parseJson(parts[0])
  const right = parseJson(parts.slice(1).join('\n---\n'))
  const diff = compareJson(left, right)

  resultText.value = [
    '新增字段：',
    diff.added.length ? diff.added.join('\n') : '无',
    '',
    '删除字段：',
    diff.removed.length ? diff.removed.join('\n') : '无',
    '',
    '变化字段：',
    diff.changed.length ? diff.changed.join('\n') : '无'
  ].join('\n')

  setStatus('Diff 对比完成', 'success')
}

function compareJson(left, right, path = '') {
  const added = []
  const removed = []
  const changed = []

  const leftKeys = isPlainObject(left) ? Object.keys(left) : []
  const rightKeys = isPlainObject(right) ? Object.keys(right) : []
  const keys = new Set([...leftKeys, ...rightKeys])

  keys.forEach((key) => {
    const currentPath = path ? `${path}.${key}` : key

    if (!isPlainObject(left) || !(key in left)) {
      added.push(currentPath)
      return
    }

    if (!isPlainObject(right) || !(key in right)) {
      removed.push(currentPath)
      return
    }

    if (isPlainObject(left[key]) && isPlainObject(right[key])) {
      const child = compareJson(left[key], right[key], currentPath)
      added.push(...child.added)
      removed.push(...child.removed)
      changed.push(...child.changed)
      return
    }

    if (JSON.stringify(left[key]) !== JSON.stringify(right[key])) {
      changed.push(`${currentPath}: ${JSON.stringify(left[key])} => ${JSON.stringify(right[key])}`)
    }
  })

  return { added, removed, changed }
}

function isPlainObject(value) {
  return value && typeof value === 'object' && !Array.isArray(value)
}

function toGoFieldName(key) {
  return key
    .replace(/[^a-zA-Z0-9_]/g, '_')
    .split(/_|-|\s+/)
    .filter(Boolean)
    .map((word) => word.charAt(0).toUpperCase() + word.slice(1))
    .join('') || 'Field'
}

function toSnakeCase(value) {
  return String(value)
    .replace(/([a-z0-9])([A-Z])/g, '$1_$2')
    .replace(/[^a-zA-Z0-9]+/g, '_')
    .replace(/^_+|_+$/g, '')
    .toLowerCase()
}

function setPath(path) {
  currentPath.value = path || '根节点'
}

function updateInputInfo(obj) {
  const type = Array.isArray(obj) ? 'Array' : typeof obj
  const size = new Blob([inputText.value]).size
  inputInfo.value = `${type} / ${size} bytes`
}

function setStatus(message, type = '') {
  statusText.value = message
  statusType.value = type
}

function showError(error) {
  resultText.value = error.message
  parsedJson.value = null
  setStatus(error.message, 'error')
}

function setExampleByTool() {
  if (currentTool.value === 'goReverse') {
    inputText.value = 'type Response struct {\n    Code int `json:"code"`\n    Msg string `json:"msg"`\n    Online bool `json:"online"`\n    List []string `json:"list"`\n}'
    runTool()
    return
  }

  if (currentTool.value === 'protoReverse') {
    inputText.value = 'syntax = "proto3";\n\nmessage Response {\n  int64 code = 1;\n  string msg = 2;\n  bool online = 3;\n  repeated string list = 4;\n}'
    runTool()
    return
  }

  if (currentTool.value === 'diff') {
    inputText.value = '{"name":"a","age":18}\n---\n{"name":"b","age":18,"city":"深圳"}'
    runTool()
    return
  }

  inputText.value = JSON.stringify({
    code: 1,
    msg: 'success',
    data: {
      list: [
        { title: '示例标题', price: 99.9, online: true },
        { title: '第二个标题', price: 199, online: false }
      ],
      total: 2
    }
  }, null, 2)
  runTool()
}

function clearAll() {
  inputText.value = ''
  resultText.value = ''
  parsedJson.value = null
  currentPath.value = ''
  inputInfo.value = ''
  setStatus('已清空')
}

async function copyText(text) {
  if (!text) return setStatus('没有可复制的内容', 'error')
  try {
    await navigator.clipboard.writeText(text)
    setStatus('复制成功', 'success')
  } catch (_) {
    setStatus('复制失败，请手动复制', 'error')
  }
}

function copyResult() {
  copyText(resultText.value)
}

function copyPath() {
  copyText(currentPath.value)
}

function saveHistory(type, content) {
  if (!content.trim()) return
  const list = getHistory()
  const item = {
    type,
    content,
    time: new Date().toLocaleString()
  }
  const next = [item, ...list.filter((i) => i.content !== content)].slice(0, 20)
  localStorage.setItem(HISTORY_KEY, JSON.stringify(next))
  historyList.value = next
}

function getHistory() {
  try {
    return JSON.parse(localStorage.getItem(HISTORY_KEY) || '[]')
  } catch (_) {
    return []
  }
}

function loadHistoryList() {
  historyList.value = getHistory()
}

function loadHistory(item) {
  inputText.value = item.content
  runTool()
}

function clearHistory() {
  localStorage.removeItem(HISTORY_KEY)
  historyList.value = []
  setStatus('历史记录已清空')
}

</script>

<style>
html, body { margin: 0; padding: 0; height: 100%; }
#app { height: 100%; }
</style>

<style scoped>
* { box-sizing: border-box; }

/* ── 根容器：占满整个视口 ── */
.devjson-app {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  background: #f5f7fb;
  color: #1f2937;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Arial, "PingFang SC", "Microsoft YaHei", sans-serif;
}

/* ── 顶部固定区域 ── */
.app-header {
  flex-shrink: 0;
  height: 56px;
  display: flex;
  align-items: center;
  padding: 0 18px;
  background: #111827;
  color: #fff;
}

.brand { font-size: 20px; font-weight: 800; }
.subtitle { margin-top: 2px; font-size: 12px; color: #cbd5e1; }

.tabs {
  flex-shrink: 0;
  display: flex;
  gap: 4px;
  padding: 10px 16px 0;
  background: #fff;
  overflow-x: auto;
  scrollbar-width: none;
}
.tabs::-webkit-scrollbar { display: none; }

.tab-btn {
  padding: 8px 12px;
  border: 0;
  border-bottom: 3px solid transparent;
  background: transparent;
  color: #475569;
  cursor: pointer;
  font-weight: 700;
  white-space: nowrap;
  font-size: 13px;
}
.tab-btn.active { color: #4f46e5; border-bottom-color: #4f46e5; }

.toolbar {
  flex-shrink: 0;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  padding: 10px 16px;
  background: #fff;
  border-bottom: 1px solid #eef2f7;
}

button {
  border: 1px solid #d8e0ea;
  background: #fff;
  color: #334155;
  border-radius: 8px;
  padding: 7px 12px;
  cursor: pointer;
  font-size: 13px;
}
button:hover { border-color: #4f46e5; color: #4f46e5; }
.primary-btn { background: #4f46e5; border-color: #4f46e5; color: #fff; }
.primary-btn:hover { color: #fff; background: #4338ca; }
.danger-btn:hover { border-color: #dc2626; color: #dc2626; }
.small-btn { margin-top: 8px; padding: 5px 10px; font-size: 12px; }
.no-margin { margin-top: 0; }

/* ── 工作区：占满剩余高度 ── */
.workspace {
  flex: 1;
  min-height: 0;
  display: grid;
  grid-template-columns: 1fr 1fr 220px;
  gap: 12px;
  padding: 12px;
  overflow: hidden;
}

/* ── 输入 / 结果面板 ── */
.panel {
  display: flex;
  flex-direction: column;
  min-height: 0;
}

.panel-title {
  flex-shrink: 0;
  height: 32px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 700;
  color: #334155;
  font-size: 13px;
}

textarea,
.result-box {
  flex: 1;
  min-height: 0;
  width: 100%;
  overflow: auto;
  resize: none;
  border: 1px solid #d8e0ea;
  border-radius: 10px;
  padding: 12px;
  background: #fbfdff;
  font-size: 13px;
  line-height: 1.65;
  font-family: Menlo, Monaco, Consolas, "Courier New", monospace;
  outline: none;
  white-space: pre-wrap;
  word-break: break-word;
}
textarea:focus { border-color: #4f46e5; box-shadow: 0 0 0 3px rgba(79, 70, 229, .1); }

/* ── 右侧边栏 ── */
.side-panel {
  display: flex;
  flex-direction: column;
  gap: 10px;
  min-height: 0;
  overflow: hidden;
}

.card { background: #fff; border: 1px solid #e5e7eb; border-radius: 10px; padding: 12px; flex-shrink: 0; }
.side-title, .side-title-row { font-weight: 800; color: #334155; margin-bottom: 8px; }
.side-title-row { display: flex; justify-content: space-between; align-items: center; }

.setting-input {
  width: 100%;
  height: 32px;
  border: 1px solid #d8e0ea;
  border-radius: 8px;
  padding: 0 10px;
  margin-bottom: 8px;
  outline: none;
  font-size: 13px;
}

.field-label { display: block; margin: 6px 0 4px; color: #475569; font-size: 12px; }
.tip { margin-top: 6px; color: #64748b; font-size: 12px; line-height: 1.6; }
.path-value { min-height: 38px; padding: 8px; border-radius: 8px; background: #f8fafc; border: 1px dashed #cbd5e1; font-family: Menlo, Monaco, Consolas, "Courier New", monospace; word-break: break-all; font-size: 12px; }

/* 历史卡片占满侧边栏剩余高度 */
.history-card { flex: 1; min-height: 0; overflow-y: auto; }

.history-item { padding: 8px; border-radius: 8px; background: #f8fafc; border: 1px solid #edf2f7; margin-bottom: 8px; cursor: pointer; font-size: 12px; color: #475569; }
.history-item:hover { background: #eef2ff; color: #3730a3; }
.history-item p { margin: 4px 0 0; }

.empty, .status { color: #64748b; font-size: 12px; font-weight: 400; }
.success { color: #059669; }
.error { color: #dc2626; }

.json-tree ul { list-style: none; margin: 0 0 0 18px; padding: 0; }
.json-node { cursor: pointer; padding: 2px 4px; border-radius: 4px; }
.json-node:hover { background: #eef2ff; }
.json-key { color: #7c3aed; font-weight: 700; }
.json-string { color: #047857; }
.json-number { color: #dc2626; }
.json-boolean { color: #2563eb; }
.json-null { color: #64748b; }

/* ── 平板（≤ 960px）：侧边栏移到底部横排 ── */
@media (max-width: 960px) {
  .workspace {
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr auto;
    overflow-y: auto;
  }

  .side-panel {
    grid-column: 1 / -1;
    flex-direction: row;
    flex-wrap: wrap;
    overflow: visible;
    min-height: auto;
  }

  .card { flex: 1; min-width: 200px; }
  .history-card { flex: 2 1 300px; max-height: 180px; overflow-y: auto; }
}

/* ── 手机（≤ 600px）：单列垂直排列 ── */
@media (max-width: 600px) {
  .app-header { height: 48px; padding: 0 12px; }
  .brand { font-size: 17px; }
  .subtitle { display: none; }
  .toolbar { padding: 8px 12px; gap: 6px; }
  button { padding: 6px 10px; font-size: 12px; }

  .workspace {
    grid-template-columns: 1fr;
    grid-template-rows: auto;
    padding: 8px;
    gap: 8px;
    overflow-y: auto;
  }

  textarea, .result-box {
    flex: none;
    height: clamp(180px, 35vh, 320px);
  }

  .side-panel {
    flex-direction: column;
    overflow: visible;
    min-height: auto;
  }

  .card { min-width: auto; }
  .history-card { max-height: 200px; flex: none; }
}
</style>
