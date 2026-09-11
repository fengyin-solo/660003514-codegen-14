<template>
  <div class="min-h-screen bg-slate-900 text-slate-200">
    <header class="border-b border-slate-700 px-6 py-4">
      <h1 class="text-2xl font-bold text-cyan-400">SQL 查询可视化与执行计划分析器</h1>
      <p class="text-sm text-slate-500 mt-1">SQL语法解析 · 执行计划树 · ER图 · 表关系聚焦 · 复杂度评分 · 优化建议</p>
    </header>
    <div class="flex flex-col lg:flex-row gap-4 p-4">
      <div class="lg:w-2/5 space-y-4">
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-sm font-bold text-slate-400">SQL 编辑器</h3>
            <div class="flex gap-2">
              <select @change="(e) => { store.sql = SQL_TEMPLATES[+(e.target as HTMLSelectElement).value].sql }" class="text-xs bg-slate-900 border border-slate-600 rounded px-2 py-1 text-slate-300">
                <option v-for="(t, i) in SQL_TEMPLATES" :key="i" :value="i">{{ t.name }}</option>
              </select>
            </div>
          </div>
          <textarea v-model="store.sql" rows="12" class="w-full bg-slate-900 border border-slate-600 rounded px-3 py-2 text-sm font-mono text-green-400 focus:outline-none focus:border-cyan-500 resize-none"></textarea>
          <button @click="store.analyze" class="w-full mt-3 py-2 bg-cyan-600 hover:bg-cyan-500 rounded text-sm font-bold">分析查询</button>
        </div>
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <div class="flex items-center justify-between mb-3">
            <h3 class="text-sm font-bold text-slate-400">数据库 Schema</h3>
            <span class="text-xs text-slate-600">点击表进入关系聚焦</span>
          </div>
          <div class="space-y-2">
            <div v-for="t in SCHEMA_TABLES" :key="t.name" @click="store.toggleFocus(t.name)"
              :class="['cursor-pointer rounded border p-2 text-xs transition-all', schemaRowClass(t.name)]">
              <div class="flex justify-between items-center">
                <span class="font-bold text-slate-200">{{ t.name }}</span>
                <div class="flex items-center gap-2">
                  <span v-if="relationBadge(t.name)" :class="['px-1.5 py-0.5 rounded', relationBadge(t.name)!.cls]">{{ relationBadge(t.name)!.text }}</span>
                  <span class="text-slate-500">{{ t.rowCount.toLocaleString() }} 行</span>
                </div>
              </div>
              <div v-if="store.activeSchema?.name === t.name" class="mt-2 space-y-1">
                <div v-for="c in t.columns" :key="c.name">
                  <div class="flex gap-2 items-baseline">
                    <span :class="c.pk ? 'text-yellow-400' : c.fk ? 'text-blue-400' : 'text-slate-400'">{{ c.pk ? '🔑 ' : c.fk ? '🔗 ' : '  ' }}{{ c.name }}</span>
                    <span class="text-slate-600">{{ c.type }}</span>
                    <span v-if="c.fk" class="text-blue-600">→ {{ c.fk }}</span>
                  </div>
                  <div v-if="c.desc" class="pl-5 text-slate-500">{{ c.desc }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
      <div class="lg:w-3/5 space-y-4">
        <div v-if="store.parsed" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">查询解析结果</h3>
          <div class="grid grid-cols-4 gap-3 text-sm mb-4">
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">类型</div><div class="text-cyan-400 font-bold">{{ store.parsed.type }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">复杂度</div><div class="font-bold" :class="store.complexityLabel.color">{{ store.complexityLabel.label }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">JOIN数</div><div class="text-orange-400 font-bold">{{ store.parsed.joins.length }}</div></div>
            <div class="bg-slate-900 rounded p-2 text-center"><div class="text-xs text-slate-500 mb-1">预估行数</div><div class="text-purple-400 font-bold">{{ store.parsed.estimatedCost }}</div></div>
          </div>
          <div v-if="store.parsed.suggestions.length" class="space-y-1">
            <div class="text-xs text-slate-500 mb-1">优化建议</div>
            <div v-for="(s, i) in store.parsed.suggestions" :key="i" class="text-xs flex items-start gap-2 bg-orange-900/30 border border-orange-700 rounded p-2">
              <span class="text-orange-400">⚠</span><span class="text-orange-300">{{ s }}</span>
            </div>
          </div>
          <div v-else class="text-xs text-green-400 bg-green-900/20 border border-green-700 rounded p-2">✓ 未发现明显性能问题</div>
        </div>
        <div v-if="store.plan" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">执行计划树</h3>
          <div class="overflow-x-auto">
            <div class="font-mono text-xs text-slate-300 space-y-1">
              <PlanNode :node="store.plan" :depth="0" />
            </div>
          </div>
        </div>
        <div v-if="store.parsed" class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <h3 class="text-sm font-bold text-slate-400 mb-3">涉及表与关联关系</h3>
          <canvas ref="erCanvasRef" class="w-full bg-slate-900 rounded" style="height:200px"></canvas>
        </div>
        <div class="bg-slate-800 rounded-lg p-4 border border-slate-700">
          <div class="flex items-center justify-between mb-3 flex-wrap gap-2">
            <h3 class="text-sm font-bold text-slate-400">表关系聚焦模式</h3>
            <div class="flex items-center gap-3 text-xs text-slate-500">
              <span class="flex items-center gap-1"><i class="w-2 h-2 rounded-full bg-cyan-400 inline-block"></i>当前表</span>
              <span class="flex items-center gap-1"><i class="w-2 h-2 rounded-full bg-blue-500 inline-block"></i>上游（被引用）</span>
              <span class="flex items-center gap-1"><i class="w-2 h-2 rounded-full bg-orange-500 inline-block"></i>下游（引用方）</span>
              <button v-if="store.focusedTable" @click="store.clearFocus" class="text-slate-400 hover:text-red-400 border border-slate-600 hover:border-red-500 rounded px-2 py-0.5 transition-colors">退出聚焦</button>
            </div>
          </div>
          <canvas ref="focusCanvasRef" @click="onFocusCanvasClick" class="w-full bg-slate-900 rounded cursor-pointer" style="height:260px"></canvas>
          <p class="text-xs text-slate-600 mt-2">点击画布中的表或左侧 Schema 列表，高亮其上下游关联并联动展示关键字段说明</p>
          <div v-if="store.focusedTable && store.focusRelations" class="mt-3 grid md:grid-cols-2 gap-3">
            <div class="bg-slate-900 rounded p-3 border border-slate-700">
              <div class="text-xs text-slate-500 mb-2">关键字段 · <span class="text-cyan-400 font-bold">{{ store.focusedTable }}</span></div>
              <div v-for="c in focusedKeyColumns" :key="c.name" class="mb-1.5">
                <div class="flex gap-2 items-baseline text-xs">
                  <span :class="c.pk ? 'text-yellow-400' : 'text-blue-400'">{{ c.pk ? '🔑 ' : '🔗 ' }}{{ c.name }}</span>
                  <span class="text-slate-600">{{ c.type }}</span>
                  <span v-if="c.fk" class="text-blue-600">→ {{ c.fk }}</span>
                </div>
                <div v-if="c.desc" class="text-xs text-slate-500 pl-5">{{ c.desc }}</div>
              </div>
            </div>
            <div class="bg-slate-900 rounded p-3 border border-slate-700">
              <div class="text-xs text-slate-500 mb-2">关联关系（{{ allFocusRelations.length }}）</div>
              <div v-for="(r, i) in allFocusRelations" :key="i" class="mb-2">
                <div class="flex items-center gap-2 text-xs">
                  <span :class="['px-1.5 py-0.5 rounded shrink-0', r.direction === 'upstream' ? 'bg-blue-900/60 text-blue-300' : 'bg-orange-900/60 text-orange-300']">{{ r.direction === 'upstream' ? '上游' : '下游' }}</span>
                  <code class="font-mono">
                    <span :class="r.fromTable === store.focusedTable ? 'text-cyan-400' : 'text-slate-300'">{{ r.fromTable }}.{{ r.fromColumn }}</span>
                    <span class="text-slate-600"> → </span>
                    <span :class="r.toTable === store.focusedTable ? 'text-cyan-400' : 'text-slate-300'">{{ r.toTable }}.{{ r.toColumn }}</span>
                  </code>
                </div>
                <div v-if="r.fromDesc" class="text-xs text-slate-500 pl-12">{{ r.fromTable }}.{{ r.fromColumn }}：{{ r.fromDesc }}</div>
              </div>
              <div v-if="!allFocusRelations.length" class="text-xs text-slate-600">该表暂无外键关联</div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, defineComponent, h } from 'vue'
import { useSQLStore, SQL_TEMPLATES, SCHEMA_TABLES } from './store/sql'

const store = useSQLStore()
const erCanvasRef = ref<HTMLCanvasElement | null>(null)
const focusCanvasRef = ref<HTMLCanvasElement | null>(null)

const PlanNode = defineComponent({
  props: { node: Object, depth: Number },
  setup(props) {
    return () => {
      if (!props.node) return null
      const n = props.node as any
      const indent = '  '.repeat(props.depth || 0)
      const opColor = n.operation.includes('Scan') ? '#22c55e' : n.operation.includes('Join') ? '#f97316' : n.operation.includes('Sort') ? '#8b5cf6' : '#06b6d4'
      return h('div', [
        h('div', { style: `padding-left: ${(props.depth || 0) * 20}px` }, [
          h('span', { style: 'color: #475569' }, indent.replace(/\s\s/g, '│ ').replace(/│ $/, '└─')),
          h('span', { style: `color: ${opColor}; font-weight: bold` }, n.operation),
          n.table ? h('span', { style: 'color: #94a3b8' }, ` on ${n.table}`) : null,
          n.index ? h('span', { style: 'color: #eab308' }, ` [${n.index}]`) : null,
          h('span', { style: 'color: #64748b' }, ` cost=${n.cost.toFixed(1)} rows=${n.rows}`),
        ]),
        ...(n.children || []).map((child: any) => h(PlanNode, { node: child, depth: (props.depth || 0) + 1 }))
      ])
    }
  }
})

// 聚焦表的关键字段（主键 + 外键）
const focusedKeyColumns = computed(() => {
  const t = SCHEMA_TABLES.find(s => s.name === store.focusedTable)
  return t ? t.columns.filter(c => c.pk || c.fk) : []
})

const allFocusRelations = computed(() => {
  const r = store.focusRelations
  return r ? [...r.upstream, ...r.downstream] : []
})

function schemaRowClass(name: string) {
  const r = store.relationTo(name)
  if (!store.focusedTable) return 'border-slate-700 hover:border-slate-500'
  if (r === 'self') return 'border-cyan-500 bg-cyan-900/20'
  if (r === 'upstream') return 'border-blue-500/70 bg-blue-900/10'
  if (r === 'downstream') return 'border-orange-500/70 bg-orange-900/10'
  return 'border-slate-800 opacity-40'
}

function relationBadge(name: string) {
  const r = store.relationTo(name)
  if (r === 'self') return { text: '当前', cls: 'bg-cyan-900/60 text-cyan-300' }
  if (r === 'upstream') return { text: '上游', cls: 'bg-blue-900/60 text-blue-300' }
  if (r === 'downstream') return { text: '下游', cls: 'bg-orange-900/60 text-orange-300' }
  return null
}

function drawER() {
  const canvas = erCanvasRef.value
  if (!canvas || !store.parsed) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  const tables = store.parsed.tables
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  canvas.width = canvas.clientWidth
  canvas.height = 200
  const W = canvas.width, H = 200
  const spacing = W / (tables.length + 1)
  const positions: Record<string, { x: number; y: number }> = {}
  tables.forEach((t, i) => { positions[t] = { x: spacing * (i + 1), y: H / 2 } })

  const related = store.focusRelatedTables

  // Draw joins
  store.parsed.joins.forEach(j => {
    const src = positions[tables[0]]
    const dst = positions[j.table]
    if (!src || !dst) return
    ctx.save()
    if (related && (!related.has(tables[0]) || !related.has(j.table))) ctx.globalAlpha = 0.2
    ctx.beginPath()
    ctx.moveTo(src.x, src.y)
    ctx.lineTo(dst.x, dst.y)
    ctx.strokeStyle = '#f97316'
    ctx.lineWidth = 2
    ctx.setLineDash([4, 4])
    ctx.stroke()
    ctx.setLineDash([])
    const mx = (src.x + dst.x) / 2, my = (src.y + dst.y) / 2
    ctx.fillStyle = '#f97316'
    ctx.font = '10px monospace'
    ctx.textAlign = 'center'
    ctx.fillText(j.type, mx, my - 5)
    ctx.restore()
  })

  // Draw table boxes
  tables.forEach((t) => {
    const pos = positions[t]
    if (!pos) return
    const x = pos.x, y = pos.y
    ctx.save()
    if (related && !related.has(t)) ctx.globalAlpha = 0.3
    ctx.fillStyle = '#1e293b'
    ctx.strokeStyle = store.focusedTable === t ? '#06b6d4' : '#3b82f6'
    ctx.lineWidth = store.focusedTable === t ? 3 : 2
    ctx.beginPath()
    ctx.roundRect(x - 50, y - 30, 100, 60, 6)
    ctx.fill()
    ctx.stroke()
    ctx.fillStyle = '#06b6d4'
    ctx.font = 'bold 13px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(t, x, y - 10)
    const schema = SCHEMA_TABLES.find(s => s.name === t)
    if (schema) {
      ctx.fillStyle = '#64748b'
      ctx.font = '10px monospace'
      ctx.fillText(schema.rowCount.toLocaleString() + ' rows', x, y + 10)
    }
    ctx.restore()
  })
}

// ---------- 表关系聚焦图 ----------

const FOCUS_COLORS = { self: '#06b6d4', upstream: '#3b82f6', downstream: '#f97316' }
const NODE_W = 116
const NODE_H = 54
const focusNodePos: Record<string, { x: number; y: number }> = {}

interface Pt { x: number; y: number }

// 矩形边缘与中心连线的交点，用于让连线止于节点边框
function rectEdge(from: Pt, to: Pt): Pt {
  const dx = to.x - from.x, dy = to.y - from.y
  const sx = dx !== 0 ? Math.abs((NODE_W / 2) / dx) : Infinity
  const sy = dy !== 0 ? Math.abs((NODE_H / 2) / dy) : Infinity
  const s = Math.min(sx, sy)
  return { x: from.x + dx * s, y: from.y + dy * s }
}

function drawArrowHead(ctx: CanvasRenderingContext2D, a: Pt, b: Pt) {
  const ang = Math.atan2(b.y - a.y, b.x - a.x)
  const L = 8
  ctx.beginPath()
  ctx.moveTo(b.x, b.y)
  ctx.lineTo(b.x - L * Math.cos(ang - 0.4), b.y - L * Math.sin(ang - 0.4))
  ctx.lineTo(b.x - L * Math.cos(ang + 0.4), b.y - L * Math.sin(ang + 0.4))
  ctx.closePath()
  ctx.fill()
}

function drawFocusGraph() {
  const canvas = focusCanvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  canvas.width = canvas.clientWidth
  canvas.height = 260
  const W = canvas.width, H = canvas.height
  ctx.clearRect(0, 0, W, H)

  // 环形布局所有 Schema 表
  const tables = SCHEMA_TABLES
  const cx = W / 2, cy = H / 2 + 6
  const rx = Math.max(W / 2 - 90, 120), ry = H / 2 - 62
  tables.forEach((t, i) => {
    const angle = -Math.PI / 2 + (i * 2 * Math.PI) / tables.length
    focusNodePos[t.name] = { x: cx + rx * Math.cos(angle), y: cy + ry * Math.sin(angle) }
  })

  const focused = store.focusedTable

  // 外键连线：fromTable.fromColumn → toTable.toColumn
  tables.forEach(t => {
    t.columns.forEach(c => {
      if (!c.fk) return
      const [rt] = c.fk.split('.')
      const from = focusNodePos[t.name]
      const to = focusNodePos[rt]
      if (!from || !to) return
      let color = '#334155', width = 1.2, alpha = 0.9
      if (focused) {
        if (t.name === focused) { color = FOCUS_COLORS.upstream; width = 2.5; alpha = 1 }
        else if (rt === focused) { color = FOCUS_COLORS.downstream; width = 2.5; alpha = 1 }
        else alpha = 0.12
      }
      ctx.save()
      ctx.globalAlpha = alpha
      ctx.strokeStyle = color
      ctx.fillStyle = color
      ctx.lineWidth = width
      if (rt === t.name) {
        // 自关联：节点上方画环形
        ctx.beginPath()
        ctx.arc(from.x, from.y - NODE_H / 2 - 12, 12, 0, Math.PI * 2)
        ctx.stroke()
        ctx.font = '10px monospace'
        ctx.textAlign = 'left'
        ctx.fillText(c.name, from.x + 16, from.y - NODE_H / 2 - 16)
      } else {
        const a = rectEdge(from, to)
        const b = rectEdge(to, from)
        ctx.beginPath()
        ctx.moveTo(a.x, a.y)
        ctx.lineTo(b.x, b.y)
        ctx.stroke()
        drawArrowHead(ctx, a, b)
        ctx.font = '10px monospace'
        ctx.textAlign = 'center'
        ctx.fillText(c.name, (a.x + b.x) / 2, (a.y + b.y) / 2 - 5)
      }
      ctx.restore()
    })
  })

  // 表节点
  tables.forEach(t => {
    const p = focusNodePos[t.name]
    const r = store.relationTo(t.name)
    let border = '#475569', alpha = 1
    if (focused) {
      if (r === 'self') border = FOCUS_COLORS.self
      else if (r === 'upstream') border = FOCUS_COLORS.upstream
      else if (r === 'downstream') border = FOCUS_COLORS.downstream
      else alpha = 0.3
    }
    ctx.save()
    ctx.globalAlpha = alpha
    ctx.fillStyle = '#1e293b'
    ctx.strokeStyle = border
    ctx.lineWidth = r === 'self' ? 3 : 2
    ctx.beginPath()
    ctx.roundRect(p.x - NODE_W / 2, p.y - NODE_H / 2, NODE_W, NODE_H, 8)
    ctx.fill()
    ctx.stroke()
    ctx.fillStyle = focused && r ? border : '#94a3b8'
    ctx.font = 'bold 13px monospace'
    ctx.textAlign = 'center'
    ctx.textBaseline = 'middle'
    ctx.fillText(t.name, p.x, p.y - 8)
    ctx.fillStyle = '#64748b'
    ctx.font = '10px monospace'
    ctx.fillText(t.rowCount.toLocaleString() + ' rows', p.x, p.y + 12)
    ctx.restore()
  })
}

function onFocusCanvasClick(e: MouseEvent) {
  const canvas = focusCanvasRef.value
  if (!canvas) return
  const rect = canvas.getBoundingClientRect()
  const x = e.clientX - rect.left
  const y = e.clientY - rect.top
  for (const [name, p] of Object.entries(focusNodePos)) {
    if (Math.abs(x - p.x) <= NODE_W / 2 && Math.abs(y - p.y) <= NODE_H / 2) {
      store.toggleFocus(name)
      return
    }
  }
}

function redrawAll() {
  drawER()
  drawFocusGraph()
}

onMounted(() => { store.analyze(); setTimeout(redrawAll, 200) })
watch(() => store.parsed, () => setTimeout(drawER, 100), { deep: true })
watch(() => store.focusedTable, () => setTimeout(redrawAll, 50))
</script>
