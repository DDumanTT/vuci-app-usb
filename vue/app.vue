<template>
  <div>
    <a-table :columns="columns" :data-source="devices" :loading="loading">
      <span slot="actions" slot-scope="_, record" style="display: flex; gap: 0.5rem">
        <a-button type="primary" @click="handleBrowse(record)">Browse</a-button>
        <a-popconfirm :title="`Unmount device ${record.dev}?`" ok-text="Yes" cancel-text="No" @confirm="handleUnmount(record)">
          <a-button type="danger">Unmount</a-button>
        </a-popconfirm>
      </span>
      <span slot="memory" slot-scope="_, record">
        <a-progress :percent="parseInt(record.used_percentage)" />
      </span>
    </a-table>
    <browse-device :device="selectedDevice" :open="modalOpen" @close="modalOpen = false" @after-close="device = null" />
  </div>
</template>

<script>
import BrowseDevice from './components/BrowseDevice.vue'

const columns = [
  {
    title: 'Mountpoint',
    dataIndex: 'mountpoint',
    key: 'mountpoint'
  },
  {
    title: 'Device',
    dataIndex: 'dev',
    key: 'dev'
  },
  {
    title: 'Filesystem',
    dataIndex: 'fs',
    key: 'fs'
  },
  {
    title: 'Used memory',
    dataIndex: 'memory',
    key: 'memory',
    scopedSlots: { customRender: 'memory' }
  },
  {
    title: 'Actions',
    dataIndex: 'actions',
    key: 'actions',
    scopedSlots: { customRender: 'actions' }
  }
]

export default {
  components: { BrowseDevice },
  data () {
    return {
      columns,
      devices: [],
      modalOpen: false,
      selectedDevice: null,
      loading: true
    }
  },
  methods: {
    async execUsbCommand (...params) {
      return await this.$rpc.ubus('file', 'exec', {
        command: '/bin/fmt-usb-msd.sh',
        params: params
      })
    },
    async getDevices () {
      const res = await this.execUsbCommand('unmountable')
      const devices = JSON.parse(res.stdout)
      this.devices = Object.values(devices).map((d, i) => { d.key = i; return d })
      this.loading = false
    },
    handleBrowse (device) {
      this.modalOpen = true
      this.selectedDevice = device
    },
    async handleUnmount (device) {
      this.$message.loading('Unmounting device ' + device.dev)
      const res = await this.execUsbCommand('unmount', device.dev)
      if (res.code === 0) {
        this.$message.success('Unmounted device ' + device.dev)
        this.devices = this.devices.filter(d => d.dev !== device.dev)
        return
      }
      this.$message.error('Failed to unmount device ' + device.dev)
    }
  },
  created () {
    this.getDevices()
  }
}
</script>
