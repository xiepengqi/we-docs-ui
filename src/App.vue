<template>
  <el-container
    id="app"
    v-loading.fullscreen.lock="isLoading"
    element-loading-text="拼命加载中"
    element-loading-spinner="el-icon-loading"
    element-loading-background="rgba(0, 0, 0, 0.8)"
  >
    <el-aside class="aside">
      <div class="toolbar">
        <div class="toolbar-row">
          <el-select
            v-model="selectedRepo"
            placeholder="repo"
            size="mini"
            filterable
            class="toolbar-item toolbar-select toolbar-select-wide"
            @change="onRepoChange"
          >
            <el-option
              v-for="item in repoList"
              :key="item"
              :label="item"
              :value="item"
            />
          </el-select>
        </div>
        <div class="toolbar-row">
          <el-select
            v-model="selectedBranch"
            placeholder="branch"
            size="mini"
            filterable
            class="toolbar-item toolbar-select"
            @change="onBranchChange"
          >
            <el-option
              v-for="item in branchList"
              :key="item"
              :label="item"
              :value="item"
            />
          </el-select>
          <el-radio-group v-model="viewMode" size="mini" @change="onModeChange">
            <el-radio-button label="doc">文档</el-radio-button>
            <el-radio-button label="diff">Diff</el-radio-button>
          </el-radio-group>
        </div>
        <div v-if="viewMode === 'diff'" class="toolbar-row">
          <el-select
            v-model="baseBranch"
            placeholder="base"
            size="mini"
            filterable
            class="toolbar-item toolbar-select"
            @change="onBaseChange"
          >
            <el-option
              v-for="item in branchList"
              :key="item"
              :label="item"
              :value="item"
            />
          </el-select>
          <el-select
            v-model="targetBranch"
            placeholder="target"
            size="mini"
            filterable
            class="toolbar-item toolbar-select"
            @change="onTargetChange"
          >
            <el-option
              v-for="item in branchList"
              :key="item"
              :label="item"
              :value="item"
            />
          </el-select>
        </div>
        <el-input
          v-if="viewMode === 'doc'"
          v-model="searchStr"
          placeholder="select..."
          class="search-input"
          clearable
        />
      </div>
      <left-nav
        v-loading="inputLoading"
        :menus="menus"
        class="left-nav"
      />
    </el-aside>
    <el-main>
      <el-button @click="goHome">Home</el-button>
      <div v-if="viewMode === 'doc' && Object.keys(menus).length > 0" v-highlight v-html="htmlDoc" />
      <div v-if="viewMode === 'diff'" v-highlight v-html="htmlDiff" />
    </el-main>
  </el-container>
</template>

<script>
import LeftNav from './components/LeftNav'
import marked from 'marked'

export default {
  components: {
    LeftNav
  },
  data() {
    return {
      searchStr: '',
      menus: {},
      searchId: '',
      inputLoading: false,
      repos: {},
      repoList: [],
      branchList: [],
      selectedRepo: '',
      selectedBranch: '',
      viewMode: 'doc',
      baseBranch: '',
      targetBranch: '',
      diffResult: null,
      diffLoading: false
    }
  },
  computed: {
    isLoading() {
      if (this.viewMode === 'diff') {
        return this.diffLoading
      }
      return Object.keys(this.menus).length <= 0
    },
    htmlDoc() {
      return marked(this.buildMd(this.$store.state.content)).replace(/&lt;br&gt;/g, '\n')
    },
    htmlDiff() {
      return marked(this.buildDiffMd(this.diffResult)).replace(/&lt;br&gt;/g, '\n')
    }
  },
  watch: {
    searchStr() {
      if (this.searchId) {
        clearTimeout(this.searchId)
      }
      this.inputLoading = true
      this.searchId = setTimeout(() => {
        this.applySearchFilter()
        this.inputLoading = false
        this.updateUrl()
      }, 1000)
    }
  },
  mounted() {
    this.loadRepos()
    marked.setOptions({
      renderer: new marked.Renderer(),
      gfm: true,
      tables: true,
      breaks: false,
      pedantic: false,
      smartLists: true,
      sanitize: true,
      smartypants: false
    })
  },
  methods: {
    loadRepos() {
      this.$http.get('/repos').then(resp => {
        const repos = (resp.data && resp.data.repos) ? resp.data.repos : {}
        this.repos = repos || {}
        this.repoList = Object.keys(this.repos)

        const query = new URLSearchParams(location.search)
        const qRepo = query.get('repo')
        const qBranch = query.get('branch')
        const qBase = query.get('base')
        const qTarget = query.get('target')
        const qMode = query.get('mode')
        const q = query.get('q') || query.get('search') || ''
        this.searchStr = q

        if (this.repoList.length > 0) {
          this.selectedRepo = qRepo || this.repoList[0]
          this.updateBranchList()
          if (this.branchList.length > 0) {
            this.selectedBranch = qBranch || this.branchList[0]
            this.baseBranch = qBase || this.branchList[0]
            this.targetBranch = qTarget || this.branchList[1] || this.branchList[0]
          }
        }
        this.viewMode = (qMode === 'diff') ? 'diff' : 'doc'
        this.refreshCurrent()
      }).catch(() => {
        this.refreshCurrent()
      })
    },
    updateBranchList() {
      this.branchList = (this.repos[this.selectedRepo] || []).slice()
    },
    onRepoChange() {
      this.updateBranchList()
      this.selectedBranch = this.branchList[0] || ''
      this.baseBranch = this.branchList[0] || ''
      this.targetBranch = this.branchList[1] || this.branchList[0] || ''
      this.refreshCurrent()
    },
    onBranchChange() {
      if (this.viewMode === 'doc') {
        this.loadData()
      }
    },
    onBaseChange() {
      if (this.viewMode === 'diff') {
        this.loadDiff()
      }
    },
    onTargetChange() {
      if (this.viewMode === 'diff') {
        this.loadDiff()
      }
    },
    onModeChange() {
      this.refreshCurrent()
    },
    refreshCurrent() {
      if (this.viewMode === 'diff') {
        this.loadDiff()
      } else {
        this.loadData()
      }
    },
    loadData() {
      const params = {}
      if (this.selectedRepo) {
        params.repo = this.selectedRepo
      }
      if (this.selectedBranch) {
        params.branch = this.selectedBranch
      }
      this.$http.get('/data', params).then(resp => {
        this.menus = resp.data || {}
        window.$weDocs = this.menus
        this.$store.state.content = this.menus
        this.applySearchFilter()
        this.updateUrl()
      })
    },
    loadDiff() {
      if (!this.selectedRepo || !this.baseBranch || !this.targetBranch) {
        this.diffResult = {
          $type: 'diff',
          $base: this.baseBranch,
          $target: this.targetBranch,
          added: [],
          removed: [],
          changed: []
        }
        return
      }
      this.diffLoading = true
      this.$http.get('/diff', {
        repo: this.selectedRepo,
        base: this.baseBranch,
        target: this.targetBranch
      }).then(resp => {
        this.diffResult = resp.data || {}
        this.diffLoading = false
        this.updateUrl()
      }).catch(() => {
        this.diffLoading = false
      })
    },
    updateUrl() {
      const params = new URLSearchParams()
      if (this.selectedRepo) {
        params.set('repo', this.selectedRepo)
      }
      if (this.selectedBranch) {
        params.set('branch', this.selectedBranch)
      }
      if (this.baseBranch) {
        params.set('base', this.baseBranch)
      }
      if (this.targetBranch) {
        params.set('target', this.targetBranch)
      }
      if (this.viewMode) {
        params.set('mode', this.viewMode)
      }
      if (this.searchStr) {
        params.set('q', this.searchStr)
      }
      const qs = params.toString()
      const next = qs ? `?${qs}` : location.pathname
      window.history.replaceState(null, '', next)
    },
    goHome() {
      this.$store.state.content = this.menus
    },
    buildMatchStr(...strs) {
      const x = ''
      if (!strs) {
        return x
      }
      return strs.map(item => item ? item.trim() : '')
        .filter(item => item).join(' ')
    },
    getChildren(data) {
      if ((typeof data) !== 'object') {
        return []
      }
      return Object.keys(data)
        .filter(item => !item.startsWith('$') && (typeof data[item]) === 'object')
        .map(item => ({ key: item, value: data[item] }))
    },
    matchNode(data, key, str) {
      if (!str) {
        return true
      }
      let matchStr = data.$matchStr || this.buildMatchStr(
        key,
        data.$name,
        data.$label,
        data.$title,
        data.$profile,
        data.$url,
        Object.keys(data.$deps || {}).join(',')
      )
      matchStr = matchStr.toUpperCase()
      if (data.$matchStr !== matchStr) {
        this.$set(data, '$matchStr', matchStr)
      }
      for (const k of str.split(/\s+/).filter(item => item)) {
        if (matchStr.indexOf(k.toUpperCase()) === -1) {
          return false
        }
      }
      return true
    },
    applyFilter(data, key, str) {
      if ((typeof data) !== 'object') {
        return false
      }
      const children = this.getChildren(data)
      let childVisible = false
      for (const child of children) {
        if (this.applyFilter(child.value, child.key, str)) {
          childVisible = true
        }
      }
      const selfMatch = this.matchNode(data, key, str)
      const visible = !str ? true : (selfMatch || childVisible)
      if (data.$hidden !== !visible) {
        this.$set(data, '$hidden', !visible)
      }
      return visible
    },
    applySearchFilter() {
      for (const [key, module] of Object.entries(this.menus)) {
        if (key.startsWith('$') || (typeof module) !== 'object') {
          continue
        }
        this.applyFilter(module, key, this.searchStr)
      }
    },
    trimDesc(str) {
      if (!str) {
        return ''
      }

      return str.replace(/[\/*]+/g, '').trim().replace(/[\n]+/g, '<br>')
    },
    buildDeps(json) {
      if (!json.$deps) {
        return ''
      }
      const deps = Object.values(json.$deps || {})
      let str = `##### Deps
`
      str += `|GroupId|ArtifactId|version|
|---|---|---|
`
      for (const item of deps) {
        str += `|${item.groupId}|${item.artifactId}|${this.getVersion(json, item.version)}|
`
      }
      return str
    },
    buildTable(data, title, record) {
      let str = ''
      if (title) {
        str = `#### ${title}
`
      } else {
        str = `##### - ${data.$type}
`
      }
      str += `|字段名称|类型|是否必填|描述|
|---|---|---|---|
`
      const ext = {}
      for (const key of Object.keys(data)) {
        if (key.startsWith('$')) {
          continue
        }
        const item = data[key]
        str += `|${key}|${item.$type}|${this.isRequired(item.$desc)}|${this.trimDesc(item.$desc)}|
`
        if (Object.keys(item).filter(item => !item.startsWith('$')).length > 0) {
          ext[item.$type] = item
        }
      }
      for (const item of Object.values(ext)) {
        if (record[item.$type]) {
          continue
        }
        str += this.buildTable(item, null, record)
        record[item.$type] = item
      }
      return str
    },
    isRequired(str) {
      const items = ['@NotBlank', '@NotEmpty', '@NotNull', '@NoneNull']
      for (const item of items) {
        if (str.indexOf(item) !== -1) {
          return 'True'
        }
      }
      return ''
    },
    getVersion(json, str) {
      if (!str) {
        return str
      }
      const pps = (json.$repo || {}).$properties
      while (str.startsWith('${')) {
        if (str === '${project.version}') {
          str = json.$version
        }
        const x = pps[str.replace(/[${}]/g, '')]
        if (!x) {
          break
        }
        str = x
      }
      return str
    },
    asLine(...strs) {
      return strs.filter(item => item).join('\n').trim()
    },
    buildErrorCode(data) {
      if (!data) {
        return ''
      }
      let str = `#### Error Code
`
      str += `|错误编码|错误信息|
|---|---|
`
      for (const key of Object.keys(data)) {
        str += `|${key}|${data[key]}|
`
      }

      return str
    },
    buildNexusDeps(json) {
      if (!json.$nexusDeps) {
        return ''
      }
      let str = `|模块|最近版本|
|---|---|
`
      Object.keys(json.$nexusDeps)
        .filter(key => !key.startsWith('$')).forEach(key => {
          const list = json.$nexusDeps[key].filter(i => !i.match(/-20.*$/))
          str += `|${key}|${list.filter((e, i) => list.indexOf(e) === i).slice(-9).join(' / ')}|
`
        })

      return `
#### NexusDeps Version
Url: ${json.$nexusBrowseUrl}

${str}
      `
    },
    buildDiffMd(diff) {
      if (!diff) {
        return '加载中...'
      }
      const base = diff.$base || ''
      const target = diff.$target || ''
      const groups = [
        { key: 'http', title: 'HTTP 接口' },
        { key: 'dubbo', title: 'Dubbo 接口' },
        { key: 'enum', title: '枚举' }
      ]

      const totalAdded = (diff.added || []).length
      const totalRemoved = (diff.removed || []).length
      const totalChanged = (diff.changed || []).length

      let md = `
### Diff (${base} -> ${target})

- 新增: ${totalAdded}
- 删除: ${totalRemoved}
- 变更: ${totalChanged}
`

      const renderList = (title, items, withChanges) => {
        if (!items || items.length <= 0) {
          return ''
        }
        let out = `\n#### ${title} (${items.length})\n`
        items.forEach(item => {
          const name = item.title || item.key || item.name || ''
          if (withChanges && item.changes) {
            const fields = Object.keys(item.changes).join(', ')
            out += `- ${name} [${fields}]\n`
          } else {
            out += `- ${name}\n`
          }
        })
        return out
      }

      groups.forEach(group => {
        const block = diff[group.key]
        if (!block) {
          return
        }
        md += `\n### ${group.title}\n`
        md += renderList('新增', block.added, false)
        md += renderList('删除', block.removed, false)
        md += renderList('变更', block.changed, true)
      })

      return md
    },
    buildMd(json) {
      const repoInfo = (json.$repo || json.$branch) ? `
#### Git Repo
\`\`\`
${json.$repo.$repo || ''}
${json.$repo.$branch || ''}
\`\`\`
` : ''
      const httpInfo = `
${json.$url ? `#### URL: ${json.$url}` : ''}

${json.$requestMethod ? `#### Method: ${json.$requestMethod}` : ''}
`
      const desc = `
\`\`\`
${this.asLine(json.$desc,
    json.$profile,
    json.$version ? ('<version>' + this.getVersion(json, json.$version) + '</version>') : '')}
\`\`\`
`
      const params = !json.$params ? '' : this.buildTable(json.$params, 'Params', {})
      const result = !json.$result ? '' : this.buildTable(json.$result, 'Result', {})
      const errorCode = this.buildErrorCode(json.$errorCode)
      const deps = this.buildDeps(json)
      const nexusDeps = this.buildNexusDeps(json)
      const mdStr = `
### ${json.$title}
${httpInfo}
${desc}
${repoInfo}
${params}
${result}
${errorCode}
${deps}
${nexusDeps}
`
      console.log(mdStr)
      return mdStr
    }
  }
}
</script>

<style scoped lang="scss">
  #app {
    height: 100vh;
    overflow: hidden;
    ::v-deep .el-container {
      height: 100%;
    }
    ::v-deep .el-main {
      overflow: auto;
    }
    .aside {
      display: flex;
      flex-direction: column;
      width: 260px;
      min-height: 0;
      overflow: hidden;
    }
    .toolbar {
      position: sticky;
      top: 0;
      width: 100%;
      padding: 5px;
      display: flex;
      flex-wrap: wrap;
      align-items: center;
      gap: 4px;
      z-index: 2;
      background: #fff;
    }
    .toolbar-row {
      display: flex;
      width: 100%;
      gap: 4px;
    }
    .toolbar-item {
      flex: 1 1 auto;
      min-width: 0;
    }
    .toolbar-select {
      flex: 0 0 auto;
      width: 120px;
    }
    .toolbar-select-wide {
      width: 80%;
    }
    .search-input {
      width: 250px;
      ::v-deep .el-input__inner {
        border: 0;
      }
    }
  }
</style>

<style lang="scss">
  table {
    border-collapse: collapse;
    white-space: pre-line;
    border:1px solid;
    font-size: 14px;
    tr td:first-child {
      min-width: 200px;
    }
    tr td:nth-child(2) {
      min-width: 200px;
    }
    tr td:nth-child(3) {
      min-width: 200px;
    }
    tr td:nth-child(4) {
      min-width: 300px;
    }
    thead {
      th
      {
        background-color: #e1dfe3;
        min-width: 200px;
      }
    }
    th {
      border: 1px solid #cad9ea;
      height: 30px;
    }
    td {
      border: 1px solid #cad9ea;
      height: 30px;
      padding-left: 3px;
    }
  }
</style>
