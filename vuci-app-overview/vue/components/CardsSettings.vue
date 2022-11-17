<template>
  <a-drawer
    width="350"
    title="Show or hide cards"
    handle="drawerHandle"
    :closable="false"
    ref="drawer"
    :visible="visible"
    @close="$emit('close')"
  >
    <div class="box" @click="$emit('close')">
      <a-icon class="icon" type="setting" />
    </div>
    <a-checkbox-group
      :style="{'display': 'grid', 'gap': '0.2em'}"
      v-model="checked"
      :options="options"
      @change="onChange"
    />
  </a-drawer>
</template>

<script>
export default {
  props: {
    data: {
      type: Array,
      required: true
    },
    visible: {
      type: Boolean,
      required: true
    }
  },
  data () {
    return {
      checked: []
    }
  },
  methods: {
    onChange (checked) {
      this.$emit('save-visible', checked)
    }
  },
  async created () {
    await this.$uci.load('overview')
    const uciCards = await this.$uci.sections('overview')
    this.checked = uciCards[0].name
  },
  computed: {
    options () {
      return this.data.map((option) => ({
        label: option.title.toUpperCase(),
        value: option.title
      }))
    }
  }
}
</script>

<style scoped>
.box {
  position: fixed;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 5px;
  height: 60px;
  width: 40px;
  background-color: #2d8cf0;
  top: 80px;
  right: 350px;
  border-radius: 5px;
}
.icon {
  font-size: 20px;
  color: #FFF;
}
</style>
