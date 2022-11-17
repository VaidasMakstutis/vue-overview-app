<template>
 <div v-if="checked && checked.includes(data.title)">
     <a-card class="card">
        <div class="title-wrapper">
          <h2 class="item-title">{{ item.title }}</h2>
          <div v-if="item.cpuProgress">
            <div>{{item.cpuProgress.title}}: ({{ item.cpuProgress.value }}%)</div>
            <a-progress :percent="item.cpuProgress.value" size="small" :show-info="false" style="width:80px" />
          </div>
        </div>
        <hr>
        <div v-for="subitems in item.data" :key="subitems.id">
            <div class="subitems-title">{{ subitems.title }}</div>
            <span class="subitems-value-wrapper" v-if="subitems.type === 'memory-progress'">
              <span v-for="data in subitems.data" :key="data.id">
                <div>
                  <label style="margin-right: 15px">{{ data.title }}: ({{ data.value }}%)</label>
                </div>
                <div>
                  <a-progress :percent="data.value" size="small" :show-info="false" style="width:80px; margin-right: 10px;" />
                </div>
              </span>
            </span>
            <div>{{ subitems.value }}</div>
            <hr>
        </div>
    </a-card>
</div>
</template>

<script>
export default {
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  data () {
    return {
      item: this.data,
      checked: []
    }
  },
  watch: {
    data: {
      deep: true,
      handler: function (newValue, oldValue) {
        this.item = newValue
      }
    }
  },
  async created () {
    await this.$uci.load('overview')
    const sid = await this.$uci.sections('overview')
    this.checked = await sid[0].name
    this.$emit('handleCheckedCards', this.checked)
  }
}
</script>

<style scoped>
.card {
  height: 100%;
}
.title-wrapper {
  display: flex;
  justify-content: space-between;
}
.item-title {
  font-weight: 700;
  text-transform: uppercase;
}
.subitems-title {
  font-weight: 600;
  text-transform: uppercase;
}
.subitems-value-wrapper {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  text-transform: uppercase;
}
</style>
