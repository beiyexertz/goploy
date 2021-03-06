<template>
  <el-row class="app-container">
    <el-row class="app-bar" type="flex" justify="end">
      <el-button type="primary" icon="el-icon-plus" @click="handleAdd">添加</el-button>
    </el-row>
    <el-table
      border
      stripe
      highlight-current-row
      :data="tableData"
      style="width: 100%"
    >
      <el-table-column prop="name" label="服务器" min-width="140" />
      <el-table-column prop="ip" label="IP" min-width="140">
        <template slot-scope="scope">
          {{ scope.row.ip }}:{{ scope.row.port }}
        </template>
      </el-table-column>
      <el-table-column prop="owner" label="sshKey所有者" width="100" show-overflow-tooltip />
      <el-table-column prop="description" label="描述" min-width="140" show-overflow-tooltip />
      <el-table-column prop="insertTime" label="创建时间" width="160" />
      <el-table-column prop="updateTime" label="更新时间" width="160" />
      <el-table-column prop="operation" label="操作" width="220" fixed="right">
        <template slot-scope="scope">
          <el-button size="small" type="primary" @click="handleEdit(scope.row)">编辑</el-button>
          <el-button size="small" type="warning" @click="handleInstall(scope.row)">安装</el-button>
          <el-button size="small" type="danger" @click="handleRemove(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <el-row type="flex" justify="end" style="margin-top: 10px;">
      <el-pagination
        hide-on-single-page
        :total="pagination.total"
        :page-size="pagination.rows"
        background
        layout="prev, pager, next"
        @current-change="handlePageChange"
      />
    </el-row>
    <el-dialog title="服务器设置" :visible.sync="dialogVisible">
      <el-form ref="form" v-loading="formProps.loading" :element-loading-text="formProps.loadingMessage" :rules="formRules" :model="formData" label-width="120px">
        <el-form-item label="服务器名称" prop="name">
          <el-input v-model="formData.name" autocomplete="off" />
        </el-form-item>
        <el-form-item label="IP" prop="ip">
          <el-input v-model="formData.ip" autocomplete="off" />
        </el-form-item>
        <el-form-item label="port" prop="port">
          <el-input v-model.number="formData.port" autocomplete="off" />
        </el-form-item>
        <el-form-item label="sshKey所有者" prop="owner">
          <el-input v-model="formData.owner" autocomplete="off" placeholder="root" />
        </el-form-item>
        <el-form-item label="描述" prop="description">
          <el-input
            v-model="formData.description"
            type="textarea"
            :autosize="{ minRows: 2}"
            placeholder="请输入描述"
          />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-row type="flex" justify="space-between">
          <el-button type="success" @click="check">测试连接</el-button>
          <el-row>
            <el-button @click="dialogVisible = false">取 消</el-button>
            <el-button :disabled="formProps.disabled" type="primary" @click="submit">确 定</el-button>
          </el-row>
        </el-row>
      </div>
    </el-dialog>
    <el-dialog title="安装模板" :visible.sync="templateDialogVisible">
      <el-row class="template-dialog">
        <el-form ref="templateForm" :rules="templateFormRules" :model="templateFormData" label-width="90px">
          <el-form-item label="选择模板" prop="templateId">
            <el-select v-model="templateFormData.templateId" placeholder="选择模板" style="width:100%">
              <el-option
                v-for="(item, index) in templateOption"
                :key="index"
                :label="item.name"
                :value="item.id"
              />
            </el-select>
          </el-form-item>
        </el-form>
        <el-row>
          <el-collapse accordion @change="handleTokenChange">
            <el-collapse-item v-for="(item, index) in installPreviewList.slice().reverse()" :key="index" :name="item.token">
              <template slot="title">
                <span style="margin-right: 10px">token: {{ item.token }}</span>
                <el-tag v-if="item.installState === 1" type="success" effect="plain">成功</el-tag>
                <el-tag v-else type="danger" effect="plain">失败</el-tag>
              </template>
              <codemirror :value="installTrace" :options="cmOptions" />
            </el-collapse-item>
          </el-collapse>
        </el-row>
      </el-row>
      <div slot="footer" class="dialog-footer">
        <el-button @click="templateDialogVisible = false">取 消</el-button>
        <el-button :disabled="templateFormProps.disabled" type="primary" @click="install">确 定</el-button>
      </div>
    </el-dialog>
    <el-dialog title="安装进度" :visible.sync="installDialogVisible">
      <codemirror :value="installLog" :options="cmOptions" />
    </el-dialog>
  </el-row>
</template>
<script>
import { getList, getTotal, getInstallPreview, getInstallList, add, edit, check, remove, install } from '@/api/server'
import { getOption as getTemplateOption } from '@/api/template'
// require component
import { codemirror } from 'vue-codemirror'
import 'codemirror/mode/shell/shell.js'
import 'codemirror/theme/darcula.css'
// require styles
import 'codemirror/lib/codemirror.css'
import 'codemirror/addon/scroll/simplescrollbars.js'
import 'codemirror/addon/scroll/simplescrollbars.css'
import 'codemirror/addon/display/autorefresh.js'

export default {
  components: {
    codemirror
  },
  data() {
    return {
      dialogVisible: false,
      templateDialogVisible: false,
      installDialogVisible: false,
      tableData: [],
      pagination: {
        page: 1,
        rows: 16,
        total: 0
      },
      templateOption: [],
      installToken: '',
      installPreviewList: [],
      installTraceList: [],
      installLog: '',
      tempFormData: {},
      cmOptions: {
        tabSize: 4,
        mode: 'text/x-sh',
        lineNumbers: false,
        line: false,
        scrollbarStyle: 'overlay',
        theme: 'darcula',
        readOnly: true,
        autoRefresh: true
      },
      formProps: {
        loading: false,
        loadingMessage: '测试连接中，请稍后',
        disabled: false
      },
      formData: {
        id: 0,
        name: '',
        ip: '',
        port: 22,
        owner: '',
        description: ''
      },
      formRules: {
        name: [
          { required: true, message: '请输入服务器名称', trigger: 'blur' }
        ],
        ip: [
          { required: true, message: '请输入服务器ip', trigger: 'blur' }
        ],
        port: [
          { type: 'number', required: true, min: 0, max: 65535, message: '请输入正确服务器端口', trigger: 'blur' }
        ],
        owner: [
          { required: true, message: '请输入SSH-KEY的所有者', trigger: 'blur' }
        ],
        description: [
          { max: 255, message: '描述最多255个字符', trigger: 'blur' }
        ]
      },
      templateFormProps: {
        disabled: false
      },
      templateFormData: {
        templateId: '',
        serverName: '',
        serverId: 0
      },
      templateFormRules: {
        templateId: [
          { required: true, message: '请选择模板', trigger: 'change' }
        ]
      }
    }
  },

  computed: {
    installTrace: function() {
      let intallTrace = ''
      this.installTraceList.forEach(element => {
        if (element.type === 1) {
          intallTrace += '[goploy~]$ ' + element.command + '\n'
          intallTrace += element.detail + '\n'
        } else if (element.type === 2) {
          intallTrace += '[goploy~]$ ' + element.ssh + '\n'
          intallTrace += element.detail + '\n'
        } else if (element.type === 3) {
          intallTrace += '[' + this.templateFormData.serverName + '~]$ ' + element.script + '\n'
          intallTrace += element.detail + '\n'
        }
      })
      return intallTrace
    }
  },

  watch: {
    '$store.getters.ws_message': function(response) {
      if (response.type !== 2) {
        return
      }
      const data = response.message
      Object.assign(data, JSON.parse(data.ext))
      let intallTrace = ''
      if (data.type === 1) {
        intallTrace += '[goploy~]$ ' + data.command + '\n'
        intallTrace += data.detail + '\n'
      } else if (data.type === 2) {
        intallTrace += '[goploy~]$ ' + data.ssh + '\n'
        intallTrace += data.detail + '\n'
      } else if (data.type === 3) {
        intallTrace += '[' + this.templateFormData.serverName + '~]$ ' + data.script + '\n'
        intallTrace += data.detail + '\n'
      }
      this.installLog += intallTrace
    }
  },

  created() {
    this.storeFormData()
    this.getList()
    this.getTotal()
    this.getTemplateOption()
  },

  methods: {
    getList() {
      getList(this.pagination).then((response) => {
        this.tableData = response.data.list
      })
    },

    getTotal() {
      getTotal().then((response) => {
        this.pagination.total = response.data.total
      })
    },

    // 分页事件
    handlePageChange(val) {
      this.pagination.page = val
      this.getList()
    },

    getTemplateOption() {
      getTemplateOption().then((response) => {
        this.templateOption = response.data.list
      })
    },

    handleAdd() {
      this.restoreFormData()
      this.dialogVisible = true
    },

    handleEdit(data) {
      this.formData = Object.assign({}, data)
      this.dialogVisible = true
    },

    handleRemove(data) {
      this.$confirm('此操作将删除该服务器, 是否继续?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        remove(data.id).then((response) => {
          this.$message.success('删除成功')
          this.getList()
          this.getTotal()
        })
      }).catch(() => {
        this.$message.info('已取消删除')
      })
    },

    handleInstall(data) {
      this.templateFormData.serverId = data.id
      this.templateFormData.serverName = data.name
      getInstallPreview(data.id).then(response => {
        this.installPreviewList = response.data.installTraceList || []
      })
      this.templateDialogVisible = true
    },

    handleTokenChange(token) {
      if (token === '') return
      getInstallList(token).then(response => {
        const installTraceList = response.data.installTraceList || []
        this.installTraceList = installTraceList.map(element => {
          Object.assign(element, JSON.parse(element.ext))
          return element
        })
      })
    },

    check() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          this.formProps.loading = true
          this.formProps.disabled = true
          check(this.formData).then(response => {
            this.$message.success('连接成功')
          }).finally(_ => {
            this.formProps.loading = false
            this.formProps.disabled = false
          })
        } else {
          return false
        }
      })
    },

    submit() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          if (this.formData.id === 0) {
            this.add()
          } else {
            this.edit()
          }
        } else {
          return false
        }
      })
    },

    add() {
      this.formProps.disabled = true
      add(this.formData).then((response) => {
        this.getList()
        this.getTotal()
        this.$message.success('添加成功')
      }).finally(() => {
        this.formProps.disabled = this.dialogVisible = false
      })
    },

    edit() {
      this.formProps.disabled = true
      edit(this.formData).then((response) => {
        this.getList()
        this.$message.success('编辑成功')
      }).finally(() => {
        this.formProps.disabled = this.dialogVisible = false
      })
    },

    install() {
      this.$refs.templateForm.validate((valid) => {
        if (valid) {
          this.templateFormProps.disabled = true
          this.installDialogVisible = true
          this.templateFormProps.disabled = this.templateDialogVisible = false
          this.installLog = ''
          install(this.templateFormData.serverId, this.templateFormData.templateId).then((response) => {
            this.$message.success(response.message)
          })
        } else {
          return false
        }
      })
    },

    formatDetail(detail) {
      return detail ? detail.replace(/\n|(\r\n)/g, '<br>') : ''
    },

    storeFormData() {
      this.tempFormData = JSON.parse(JSON.stringify(this.formData))
    },

    restoreFormData() {
      this.formData = JSON.parse(JSON.stringify(this.tempFormData))
    }
  }
}
</script>
<style lang="scss" scoped>
@import "@/styles/mixin.scss";
.template-dialog {
  padding-right: 10px;
  height: 400px;
  overflow-y: auto;
  @include scrollBar();
}
</style>
