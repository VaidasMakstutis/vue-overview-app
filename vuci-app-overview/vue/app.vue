<template>
<div>
  <div v-if="checkedCards === false">
    <h1 style="fontSize: 24px; text-align: center; padding: 30px">No selected cards! <strong>Please go to the settings bar</strong></h1>
  </div>
  <div class="cards-wrapper">
    <Card v-for="card in cardsData"
      :data="card"
      :key="card.id"
      @handleCheckedCards='handleCheckedCards'
    />
  </div>
    <CardsSettings
      :data="cardsData"
      :visible="drawerVisible"
      @close="drawerVisible = !drawerVisible"
      @save-visible="saveCardsSettings"
    />
</div>
</template>

<script>

import Card from './components/Card.vue'
import CardsSettings from './components/CardsSettings.vue'

export default {
  components: { Card, CardsSettings },
  data () {
    return {
      visible: [],
      drawerVisible: false,
      cardsData: [],
      checkedCards: true
    }
  },
  timers: {
    update: { time: 1000, autostart: true, repeat: true, immediate: true },
    updateCpuUsage: {
      time: 1000,
      autostart: true,
      immediate: true,
      repeat: true
    }
  },
  methods: {
    async getSystemInfo () {
      this.system = await this.$system.getInfo()
      const systemObj = { title: 'System', data: [], cpuProgress: {} }
      const memoryData = { title: 'Memory usage', data: [], type: 'memory-progress' }
      this.ramUsage = Math.floor(((this.system.memory.total - this.system.memory.free) / this.system.memory.total) * 100)
      const { root } = await this.$system.getDiskInfo()
      this.flashUsage = Number(parseFloat((root.used / root.total) * 100).toFixed(2))
      systemObj.cpuProgress = { title: 'CPU load', value: this.cpuUsage }
      systemObj.data.push({ title: 'Router uptime', value: '%t'.format(this.system.uptime) })
      systemObj.data.push({ title: 'Local device time', value: this.localTime() })
      memoryData.data.push({ title: 'Ram', value: this.ramUsage })
      memoryData.data.push({ title: 'Flash', value: this.flashUsage })
      systemObj.data.push(memoryData)
      systemObj.data.push({ title: 'Firmware version', value: this.system.release && this.system.release.revision })
      return systemObj
    },
    async getInterfacesInfo () {
      const interfacesData = []
      await this.$network.load()
      const interfaces = await this.$network.getInterfaces()
      for (const networkInt of interfaces) {
        const address = !networkInt.status['ipv4-address'] ? '-' : networkInt.status['ipv4-address'][0].address + '/' + networkInt.status['ipv4-address'][0].mask
        const interfaceData = { title: networkInt.name, data: [] }
        interfaceData.data.push({ title: 'type', value: networkInt.status.device })
        interfaceData.data.push({ title: 'Ip address', value: address })
        interfacesData.push(interfaceData)
      }
      return interfacesData
    },
    async getSystemEvents () {
      const dataObj = { title: 'Recent system events', data: [] }
      await this.$log.get({ table: 'SYSTEM', limit: 5 }).then((r) => {
        for (const record of r.log) {
          dataObj.data.push({ title: this.formatDate(record.TIME), value: record.TEXT })
        }
      })
      return dataObj
    },
    async getNetworkEvents () {
      const dataObj = { title: 'Recent network events', data: [] }
      await this.$log.get({ table: 'NETWORK', limit: 5 }).then((r) => {
        for (const record of r.log) {
          dataObj.data.push({ title: this.formatDate(record.TIME), value: record.TEXT })
        }
      })
      return dataObj
    },
    async update () {
      const systemCard = await this.cardsData.find(c => c.title === 'System')
      if (systemCard) {
        systemCard.data = (await this.getSystemInfo()).data
        systemCard.cpuProgress = (await this.getSystemInfo()).cpuProgress
      }
    },
    async updateCpuUsage () {
      await this.$rpc.call('system', 'cpu_time').then((times) => {
        if (!this.lastCPUTime) {
          this.cpuUsage = 0
          this.lastCPUTime = times
          return
        }
        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]
        let total1 = 0
        let total2 = 0
        this.lastCPUTime.forEach((t) => {
          total1 += t
        })
        times.forEach((t) => {
          total2 += t
        })
        this.cpuUsage = Math.floor(
          ((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100
        )
        this.lastCPUTime = times
      })
    },
    handleCheckedCards (checked) {
      if (checked) {
        this.checkedCards = true
        return this.checkedCards
      } else {
        this.checkedCards = false
        return this.checkedCards
      }
    },
    async saveCardsSettings (checked) {
      await this.$uci.load('overview')
      const delSid = await this.$uci.sections('overview')
      for (let i = 0; i < Object.keys(delSid).length; i++) {
        await this.$uci.del('overview', delSid[i]['.name'])
      }
      const sid = await this.$uci.add('overview', 'visible_cards')
      await this.$uci.set('overview', sid, 'name', checked)
      await this.$uci.save()
      await this.$uci.apply()
      await this.getAllCards()
    },
    async getAllCards () {
      this.cardsData = []
      for (const netInterface of await this.getInterfacesInfo()) {
        this.cardsData.push(netInterface)
      }
      this.cardsData.push(await this.getNetworkEvents())
      this.cardsData.push(await this.getSystemEvents())
      this.cardsData.push(await this.getSystemInfo())
    },
    formatDate () {
      const d = new Date()
      const year = d.getFullYear().toString().padStart(2, 0)
      const month = (d.getMonth() + 1).toString().padStart(2, 0)
      const date = d.getDate().toString().padStart(2, 0)
      const hour = d.getHours()
      const min = d.getMinutes()
      const sec = d.getSeconds()
      return `${year}-${month}-${date} %02d:%02d:%02d`.format(hour, min, sec)
    },
    localTime () {
      const d = new Date(this.system.localtime * 1000)
      const year = d.getFullYear().toString().padStart(2, 0)
      const month = (d.getMonth() + 1).toString().padStart(2, 0)
      const date = d.getDate().toString().padStart(2, 0)
      const hour = d.getHours()
      const min = d.getMinutes()
      const sec = d.getSeconds()
      return `${year}-${month}-${date} %02d:%02d:%02d`.format(hour, min, sec)
    }
  },
  async created () {
    await this.getAllCards()
  }
}
</script>

<style scoped>
.cards-wrapper {
  margin: 25px;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  grid-auto-rows: 1fr;
  grid-gap: 15px;
}
</style>
