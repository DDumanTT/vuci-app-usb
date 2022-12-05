<template>
  <a-modal v-if="!!device" :visible="open" :title="`${device.dev} - ${dir}`" :destroyOnClose="true" @cancel="$emit('close')" :afterClose="() => $emit('after-close')" :footer="null" :width="640">
    <a-upload style="display: block; margin-bottom: 1rem;" action="/upload" :data="{path: uploadPath}" :beforeUpload="beforeUpload" @change="handleUploadFinished">
      <a-button> <a-icon type="upload" /> Click to Upload a File </a-button>
    </a-upload>
    <a-table :columns="columns" :data-source="files" :customRow="customRow">
      <span slot="name" slot-scope="_, record"><a-icon :type="record.type === 'file' ? 'file' : 'folder'" /> {{record.name}}</span>
      <span slot="actions" slot-scope="_, record" v-if="record.name !== '..'">
        <a-button type="primary" v-if="record.type === 'file'" @click="handleDownload(record)">Download</a-button>
        <a-popconfirm :title="`Delete ${record.name}?`" ok-text="Yes" cancel-text="No" @confirm="handleDelete(record)">
          <a-button type="danger">Delete</a-button>
        </a-popconfirm>
      </span>
    </a-table>
    <span>
      <a-input placeholder="Directory name" v-model="newDirName" />
      <a-button type="primary" @click="handleCreateDirectory">Create directory</a-button>
    </span>
    <form ref="download" method="POST" action="/download">
      <input v-show="false" type="text" :value="downloadPath" name="path"/>
      <input v-show="false" type="text" :value="downloadFile" name="filename"/>
    </form>
  </a-modal>
</template>

<script>
import path from 'path'

const columns = [
  {
    title: 'Name',
    dataIndex: 'name',
    key: 'name',
    scopedSlots: { customRender: 'name' }
  },
  {
    title: 'Size',
    dataIndex: 'size',
    key: 'size'
  },
  {
    title: 'Actions',
    dataIndex: 'actions',
    key: 'actions',
    scopedSlots: { customRender: 'actions' }
  }
]

export default {
  props: ['device', 'open'],
  data () {
    const customRow = (record) => {
      return {
        on: {
          click: (event) => {
            event.stopPropagation() // doesn't work
            if (event.target.tagName === 'BUTTON') return // hacky fix
            this.handleRowClick(record)
          }
        }
      }
    }

    return {
      columns,
      customRow,
      files: [],
      dir: '/',
      downloadFile: '',
      downloadPath: '',
      newDirName: '',
      uploadPath: ''
    }
  },
  methods: {
    async getFiles () {
      const res = await this.$rpc.ubus('file', 'list', { path: path.join(this.device.mountpoint, this.dir) })
      const files = res.entries.map((f, i) => ({ key: i, ...f }))
      if (this.dir !== '/') {
        const prevDir = path.join(this.dir, '..')
        const prevDirStat = await this.$rpc.ubus('file', 'stat', { path: path.join(this.device.mountpoint, prevDir) })
        prevDirStat.key = 'back'
        prevDirStat.name = '..'
        this.files = [prevDirStat, ...files]
      } else {
        this.files = files
      }
    },
    handleRowClick (file) {
      if (file.type === 'directory') {
        this.dir = path.join(this.dir, file.name)
        this.getFiles()
      }
    },
    async handleDownload (file) {
      let filePath = path.join(this.device.mountpoint, this.dir)
      filePath = path.join(filePath, file.name)
      this.downloadPath = filePath
      this.downloadFile = file.name
      await this.$nextTick()
      this.$refs.download.submit()
    },
    handleDelete (file) {
      this.$rpc.ubus('file', 'remove', { path: this.device.mountpoint + this.dir + file.name })
      this.$message.success(`${file.name} removed`)
      this.getFiles()
    },
    handleCreateDirectory () {
      this.$rpc.ubus('file', 'exec', {
        command: 'mkdir',
        params: ['-p', path.join(this.device.mountpoint, this.dir) + this.newDirName]
      })
      this.$message.success(`Directory ${this.newDirName} created.`)
      this.newDirName = ''
      this.getFiles()
    },
    beforeUpload (file) {
      const pathPartial = path.join(this.device.mountpoint, this.dir)
      this.uploadPath = path.join(pathPartial, file.name)
    },
    handleUploadFinished (info) {
      const status = info.file.status
      if (status !== 'done') return
      this.$message.success(`File ${info.file.name} uploaded.`)
      this.getFiles()
    }
  },
  watch: {
    device: function (newDevice, oldDevice) {
      if (!oldDevice && newDevice) {
        this.getFiles()
      }
    }
  }
}
</script>

<style scoped>
span {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
</style>
