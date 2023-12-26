<template>
    <div>
      <a-row :gutter="[8,8]" id="dropZone">
        <draggable v-model="cards" @end="onDragEnd">
          <Card v-for="card in cards" :key="card.position" :id="card.position" :card="card" />
        </draggable>
      </a-row>
      <a-button shape="circle" style="float: right; margin: 20px" type="primary" @click="drawerVisible = !drawerVisible">+</a-button>
      <a-drawer title="Router information" placement="right" :visible="drawerVisible" width="300px"
        @close="drawerVisible = !drawerVisible" :key="rerenderDrawer">
        <div v-for="(card) in cards" :key="card.id">
          <a-switch :checked="card.isVisible === 'true'" @click="handleCardVisibility(card.id)" /> {{ card.title }}
        </div>
      </a-drawer>
    </div>
  </template>
  
  <script>
  import Card from './components/card.vue'
  import draggable from 'vuedraggable'
  export default {
    components: {
      draggable,
      Card
    },
    data () {
      return {
        config: 'overview',
        section: 'card',
        sections: [],
        sysinfo: [],
        interfaces: [],
        rerenderDrawer: 10000,
        cards: [],
        drawerVisible: false,
        lastCPUTime: null,
        cpuPercentage: 100
      }
    },
    timers: {
      addContentToCard: { time: 5000, autostart: true, immediate: true, repeat: true }
    },
    methods: {
      async load () {
        await this.$uci.load(this.config)
        this.sections = await this.$uci.sections(this.config, this.section)
        await this.$network.load()
        this.interfaces = await this.$network.getInterfaces()
      },
      async onDragEnd () {
        this.cards.forEach((card, index) => {
          const whichSection = this.sections.find(section => section.id === card.id)
          this.$uci.set(this.config, whichSection['.name'], 'position', index.toString())
        })
        await this.$uci.save()
        await this.$uci.apply()
        await this.load()
        await this.addContentToCard()
      },
      async handleCardVisibility (id) {
        const whichSection = this.sections.find(section => section.id === id)
        if (whichSection.isVisible === 'true') {
          this.$uci.set(this.config, whichSection['.name'], 'isVisible', 'false')
        } else {
          this.$uci.set(this.config, whichSection['.name'], 'isVisible', 'true')
        }
        await this.$uci.save()
        await this.$uci.apply()
        await this.load()
        await this.addContentToCard()
        this.rerenderDrawer++
      },
      addCards (id, isVisible, position, title, content) {
        this.cards.push({ id, isVisible, position, title, content })
      },
      dateFormatter (time) {
        const d = new Date(time * 1000)
        const year = d.getFullYear()
        const month = d.getMonth() + 1
        const date = d.getDate()
        const hour = d.getHours()
        const min = d.getMinutes()
        const sec = d.getSeconds()
        return `${year}-%02d-%02d %02d:%02d:%02d`.format(month, date, hour, min, sec)
      },
      getSystemInfo () {
        const systemInfo = []
        this.updateCpuUsage()
        this.$system.getInfo().then(({ release, localtime, uptime, memory, disk }) => {
          this.sysinfo = [
            [this.$t('FIRMWARE VERSION'), release.revision],
            [this.$t('home.Local Time'), localtime],
            [this.$t('ROUTER UPTIME'), '%t'.format(uptime)]
          ]
          this.memPercentage = Math.floor(((memory.total - memory.free) / memory.total) * 100)
          this.flashMemPercentage = Math.floor(((((disk.root.total + disk.tmp.total) - (disk.tmp.free + disk.root.free)) / (disk.tmp.total + disk.root.total))) * 100) // disk
          const customFlashMemorys = ['RAM/FLASH', this.memPercentage + '/' + this.flashMemPercentage]
          const formattedTime = this.dateFormatter(this.sysinfo[1][1])
          const deviceTime = ['LOCAL DEVICE TIME', formattedTime]
          const cpuUsage = ['cpu_time', this.cpuPercentage]
          systemInfo.push(this.sysinfo[2], deviceTime, customFlashMemorys, this.sysinfo[0], cpuUsage)
        })
        return systemInfo
      },
      async updateCpuUsage () {
        const times = await this.$rpc.call('system', 'cpu_time')
        if (!this.lastCPUTime) {
          this.cpuPercentage = 0
          this.lastCPUTime = times
          return
        }
        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]
        let total1 = 0
        let total2 = 0
        this.lastCPUTime.forEach(t => {
          total1 += t
        })
        times.forEach(t => {
          total2 += t
        })
        this.cpuPercentage = Math.floor(((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100)
        this.lastCPUTime = times
      },
      getLastEvents (type) {
        const transformedLog = []
        this.$rpc.call('log', 'log', { table: type, limit: 4, order: 'DESC' }).then(res => {
          res.log.forEach(r => {
            const formattedDate = this.dateFormatter(r.TIME)
            transformedLog.push([formattedDate, r.TEXT])
          })
        })
        return transformedLog
      },
      getInterfaceContent (interf, type) {
        let transformedData
        for (let j = 0; j < interf.length; j++) {
          const temp = interf[j]
          if (interf[j].name === type && temp.status.up === true) {
            transformedData = [['TYPE', temp.status.device], ['IP ADDRESS', temp.status['ipv4-address'][0].address + '/' + temp.status['ipv4-address'][0].mask]]
            break
          } else if (interf[j].name === type && temp.status.device) {
            transformedData = [['TYPE', temp.status.device], ['STATUS', 'Down']]
            break
          } else if (interf[j].name === type) {
            transformedData = [['TYPE', 'NO DEVICE'], ['STATUS', 'Down']]
            break
          } else {
            transformedData = [['TYPE', 'NO DEVICE'], ['STATUS', 'Down']]
            break
          }
        }
        this.interfaces.shift()
        return transformedData
      },
      async addContentToCard () {
        await this.load()
        this.cards = []
        if (this.sections) {
          for (let index = 0; index < this.sections.length; index++) {
            const current = this.sections[index]
            switch (current.title) {
              case 'SYSTEM': {
                const systemInfo = this.getSystemInfo()
                this.addCards(current.id, current.isVisible, current.position, current.title, systemInfo)
                break
              }
              case 'RECENT NETWORK EVENTS': {
                const networkLogs = this.getLastEvents('NETWORK')
                this.addCards(current.id, current.isVisible, current.position, current.title, networkLogs)
                break
              }
              case 'RECENT SYSTEM EVENTS': {
                const systemLogs = this.getLastEvents('SYSTEM')
                this.addCards(current.id, current.isVisible, current.position, current.title, systemLogs)
                break
              }
              default: {
                const titleLowerCase = current.title ? current.title.toLowerCase() : ''
                const transformedWanData = this.getInterfaceContent(this.interfaces, titleLowerCase)
                this.addCards(current.id, current.isVisible, current.position, current.title, transformedWanData)
                break
              }
            }
          }
          this.cards.sort(function (a, b) {
            return a.position - b.position
          })
        }
      }
    },
    async created () {
      await this.load()
      this.addContentToCard()
    }
  }
  </script>
  <style>
  </style>