<template>
  <section>
    <div>
      {{seat.id}}
      <div>基本信息：
        {{seat.player.id}}
        {{seat.player.total}}
        {{seat.player.identity}}
      </div>
      <div>
        操作: <span v-if="seat.action"
                  @click="action">action</span>
        <span v-if="seat.action" @click="pass">pass</span>
      </div>
      持有的：
      <ul>
        <li v-for="hold in seat.player.holdPoker" class="hold-poker">
          <input type="checkbox" v-model="hold.checked"> {{hold.type}} , {{hold.value}} ,权重 {{hold.weight}}
        </li>
      </ul>
    </div>
    <items :seat="seat.next" @action="action" @pass="pass" :head="head" v-if="seat.next.id!==head"></items>
  </section>
</template>

<script type="text/ecmascript-6">
  export default {
    name: 'items',
    props: {
      seat: {
        type: null,
      },
      head: null
    },
    data () {
      return {}
    },
    created () {
      console.log('seat==========', this.seat, '======', this.head)
    },
    methods: {
      action () {
        this.$emit('action')
      },
      pass () {
        this.$emit('pass')
      }
    }
  }
</script>
<style lang="scss" scoped type="text/css">
  .hold-poker {

  }
</style>
