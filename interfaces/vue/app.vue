<template>
  <div>
    <a-table :data-source="interfacesDataSource" :columns="columns">
      <template v-slot:network="record">
        <network-badge :iface="record.name" :device="record.getDevice() && record.getDevice().name"/>
      </template>
      <template v-slot:status="record">
        <strong>{{ $t('Uptime') }}</strong>: {{ record.isUp() ? '%t'.format(record.getUptime()) : $t('interfaces.Interface is down') }}<br/>
        <strong>MAC</strong>: {{ record.getDevice() ? record.getDevice().macaddr : '' }}<br/>
        <strong>RX</strong>: {{ '%mB'.format(record.getStatistics().rx_bytes) }}<br/>
        <strong>TX</strong>: {{ '%mB'.format(record.getStatistics().tx_bytes) }}<br/>
        <strong>IPv4</strong>: {{ record.getIPv4Addrs().join(',') }}<br/>
        <strong>IPv6</strong>: {{ record.getIPv6Addrs().join(',') }}<br/>
      </template>
      <template v-slot:action="record">
        <a-button type="primary" size="small" style="margin-right: 5px" @click="ifup(record.name)">{{ $t('interfaces.Start') }}</a-button>
        <a-button type="warning" size="small" style="margin-right: 5px" @click="ifdown(record.name)">{{ $t('interfaces.Stop') }}</a-button>
        <a-button type="primary" size="small" style="margin-right: 5px" @click="edit(record.name)">{{ $t('Edit') }}</a-button>
        <a-popconfirm @confirm="del(record.name)" placement="left" :title="$t('interfaces.DelConfirm')">
          <a-button type="danger" size="small">{{ $t('Delete') }}</a-button>
        </a-popconfirm>
      </template>
    </a-table>
    <a-button type="primary" size="small" style="margin-top: 10px" @click="handleAdd">+ {{ $t('interfaces.Add interface') }}</a-button>
    <a-modal :title="modalTitle" v-model="editModal" :width="800">
      <vuci-form uci-config="network" v-if="editModal && !switchProto">
        <vuci-named-section :name="editorIface" v-slot="{ s }" @change-proto="protoChanged">
          <component :is="protoComponentName(s.proto)" :uci-section="s"/>
        </vuci-named-section>
      </vuci-form>
      <template #footer><div/></template>
    </a-modal>
  </div>
</template>

<script>
import NetworkBadge from './interfaces/NetworkBadge.vue'
import ProtoNone from './interfaces/proto/None.vue'
import ProtoDhcp from './interfaces/proto/Dhcp.vue'
import ProtoStatic from './interfaces/proto/Static.vue'
import ProtoPppoe from './interfaces/proto/Pppoe.vue'

export default {
  components: {
    NetworkBadge,
    ProtoNone,
    ProtoDhcp,
    ProtoStatic,
    ProtoPppoe
  },
  data () {
    return {
      interfaces: [],
      columns: [
        { key: 'network', title: this.$t('Network'), width: 140, scopedSlots: { customRender: 'network' } },
        { key: 'status', title: this.$t('Status'), scopedSlots: { customRender: 'status' } },
        { key: 'action', title: '#', width: 240, scopedSlots: { customRender: 'action' } }
      ],
      editorIface: '',
      editModal: false,
      proto: null,
      switchProto: false,
      sections: []
    }
  },
  timers: {
    load: { time: 3000, autostart: true, immediate: true, repeat: true },
    loadOverview: { time: 3000, autostart: true, immediate: true, repeat: true }
  },
  computed: {
    interfacesDataSource () {
      return this.interfaces.map((v, i) => {
        v.key = i
        return v
      })
    },
    modalTitle () {
      return `${this.$t('interfaces.Configure')} "${this.editorIface}"`
    }
  },
  methods: {
    protoComponentName (proto) {
      if (!proto) return 'proto-none'
      return 'proto-' + proto
    },
    protoChanged (self) {
      if (!self.changed()) return

      this.$confirm({
        title: this.$t('interfaces.Switch protocol'),
        content: this.$t('interfaces.Are you sure you want to switch the protocol?'),
        onOk: () => {
          this.switchProto = true
          this.$nextTick(() => {
            this.$uci.reset()
            this.$uci.set('network', this.editorIface, 'proto', self.model)
            this.$uci.save()
            this.$uci.apply().then(() => {
              this.switchProto = false
            })
          })
        },
        onCancel: () => {
          self.reset()
        }
      })
    },
    load () {
      return this.$network.load().then(() => {
        this.interfaces = this.$network.getInterfaces()
      })
    },
    loadOverview () {
      return this.$uci.load('overview').then(() => {
        this.sections = this.$uci.sections('overview', 'card')
      })
    },
    async updateOverview () {
      const newInterfaces = this.interfaces.filter(intf => !this.sections.some(sec => sec.title === intf.name))
      for (let index = 0; index < newInterfaces.length; index++) {
        const randomPosition = Math.random() * 1000
        this.$spin()
        const sid = this.$uci.add('overview', 'card')
        this.$uci.set('overview', sid, 'title', newInterfaces[index].name)
        this.$uci.set('overview', sid, 'isVisible', 'true')
        this.$uci.set('overview', sid, 'position', randomPosition)
        this.$uci.set('overview', sid, 'id', (Math.floor(Math.random() * 1000) + 1 + this.sections.length).toString())
        await this.$uci.save()
        await this.$uci.apply()
        this.load()
        this.loadOverview()
        this.$spin(false)
      }
    },
    ifup (name) {
      this.$rpc.ubus(`network.interface.${name}`, 'up')
    },
    ifdown (name) {
      this.$rpc.ubus(`network.interface.${name}`, 'down')
    },
    del (name) {
      this.$spin()
      const findOverviewName = this.sections.find(sec => sec.title === name)
      this.$uci.del('network', name)
      if (findOverviewName) {
        this.$uci.del('overview', findOverviewName['.name'])
      }
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.load()
          this.loadOverview()
          this.$spin(false)
        })
      })
    },
    edit (iface) {
      this.proto = this.$uci.get('network', iface, 'proto')
      this.editorIface = iface
      this.editModal = true
    },
    add (name) {
      const randomPosition = Math.random() * 1000
      this.$spin()
      this.$uci.add('network', 'interface', name)
      const sid = this.$uci.add('overview', 'card')
      this.$uci.set('overview', sid, 'title', name)
      this.$uci.set('overview', sid, 'isVisible', 'true')
      this.$uci.set('overview', sid, 'position', randomPosition)
      this.$uci.set('overview', sid, 'id', (Math.floor(Math.random() * 1000) + 1 + this.sections.length).toString())
      this.$uci.save().then(() => {
        this.$uci.apply().then(() => {
          this.load()
          this.loadOverview()
          this.$spin(false)
        })
      })
    },
    handleAdd () {
      this.$prompt({
        title: this.$t('interfaces.Add interface'),
        placeholder: this.$t('Please input a name')
      }).then(name => {
        if (!name) return

        if (name.match(/^[a-zA-Z0-9_]+$/) === null) {
          this.$message.error(this.$t('validator.uci'))
          return
        }

        for (let i = 0; i < this.interfaces.length; i++) {
          if (this.interfaces[i].name === name) {
            this.$message.error(this.$t('Name already used'))
            return
          }
        }

        this.add(name)
      }).catch(() => {})
    }
  },
  async created () {
    await this.load()
    await this.loadOverview()
    await this.updateOverview()
  }
}
</script>
