<template>
  <a-col :span="6" v-if="card.isVisible === 'true'">
    <a-card :title="card.title" :id="id" class="card">
      <div v-for="(item, index) in card.content" :key="index">
        <div v-if="showMemoryProgress(item)">
          <p style="marginBottom: 0;"><strong>MEMORY USAGE</strong></p>
          <a-row :gutter="[8,8]">
            <div v-for="(itm, ind) in item[0].split('/')" :key="ind">
              <a-col :span="12">
                <small style="color: blue;">{{ itm }}: ({{ item[1].split('/')[ind] }}%)</small>
                <vuci-progress-bar :showInfo="false" :value="parseInt(item[1].split('/')[ind])" :barWeight="5" :barWidth="100"></vuci-progress-bar>
              </a-col>
            </div>
          </a-row>
          <hr/>
        </div>
        <div v-if="item[0] !== cpu_name && item[0] !== memory + '/' + flash_memory && item[0] !== flash_memory && item[0] !== memory">
          <p style="marginBottom: 0;"><strong>{{ item[0] }}</strong></p>
          <small>{{ item[1] }}</small>
          <hr />
        </div>
        <div v-else-if="item[0] === memory || item[0] === flash_memory">
          <div v-if="showMemory(item)">
            <small style="color: blue;">{{ itm }}: ({{ item[1].split('/')[ind] }}%)</small>
            <vuci-progress-bar :title="item[0]" :value="item[1]" :barWeight="5" :barWidth="100"></vuci-progress-bar>
          </div>
          <div v-else-if="showMemoryFlash(item)">
            <small style="color: blue;">{{ itm }}: ({{ item[1].split('/')[ind] }}%)</small>
            <vuci-progress-bar :title="item[0]" :value="item[1]" :barWeight="5" :barWidth="100"></vuci-progress-bar>
          </div>
        </div>
      </div>
      <template #extra>
        <div v-if="showExtra()" style="margin: -16px; marginRight: 5px">
          <small style="color: blue;">{{ cpu_title }} ({{ cpu_value() }}%)</small>
          <vuci-progress-bar :showInfo="false" :value="cpu_value()" :barWeight="5" :barWidth="100"></vuci-progress-bar>
        </div>
      </template>
    </a-card>
  </a-col>
</template>

<script>

export default {
  props: {
    isVisible: {
      type: Boolean,
      default: false
    },
    editableField: {
      type: String,
      default: ''
    },
    card: Object,
    draggable: String,
    id: String
  },

  data () {
    return {
      cpu_name: 'cpu_time',
      memory: 'RAM',
      flash_memory: 'FLASH',
      cpu_title: 'CPU load:'
    }
  },
  computed: {
    formattedData () {
      const result = []
      let currentGroup = []

      for (const item of this.card) {
        if (typeof item[1].content !== 'undefined') {
          currentGroup.push(item[0], item[1])
        } else {
          result.push(currentGroup)
          currentGroup = []
          result.push([item[0].content, item[1].content])
        }
      }

      if (currentGroup.length > 0) {
        result.push(currentGroup)
      }

      return result
    }
  },
  methods: {
    showExtra () {
      const isExtraVisible = this.card.content.find(cont => cont[0] === this.cpu_name)
      return isExtraVisible
    },
    cpu_value () {
      const cpuContent = this.card.content.find(cont => cont[0] === this.cpu_name)
      return cpuContent[1]
    },
    showMemoryProgress (current) {
      const isMemoryFieldVisable = current[0] === this.memory + '/' + this.flash_memory
      return isMemoryFieldVisable
    },
    showMemoryFlash (current) {
      const isMemoryFieldVisable = current[0] === this.flash_memory
      return isMemoryFieldVisable
    },
    showMemory (current) {
      const isMemoryFieldVisable = current[0] === this.memory
      return isMemoryFieldVisable
    }
  }

}
</script>
<style>
.card {
  min-height: 390px;
  margin: 10px;
}
</style>
