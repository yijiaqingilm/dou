<template>
  <div>
    <div @click="send">派发</div>
    <div @click="reset">reset</div>
    <div>房间{{room.id}}, 底分 {{room.basicScore}}</div>
    <div>底牌:
      <div v-for="cards in room.cardsPai">{{cards.type}}, {{cards.value}} ,权重 {{cards.weight}}</div>
    </div>
    <ul>
      <li v-for="player in room.players">
        <div>基本信息：{{player.id}} {{player.total}} {{player.identity}}</div>
        <div v-if="room.canAction">操作:</div>
        持有的：
        <ul>
          <li v-for="hold in player.holdPoker">
            {{hold.type}} , {{hold.value}} ,权重 {{hold.weight}}
          </li>
        </ul>
      </li>
    </ul>

  </div>

</template>

<script type="text/ecmascript-6">
  import _ from 'lodash'
  // 乱序
  var shuffle = (arr) => {
    for (let i = 0; i < arr.length; i++) {
      let j = Math.floor(Math.random() * arr.length)
      let value
      value = arr[i]
      arr[i] = arr[j]
      arr[j] = value
    }
    return arr
  }

  var baseSlice = (array, start, end) => {
    let index = -1
    let length = array.length

    if (start < 0) {
      start = -start > length ? 0 : (length + start)
    }
    end = end > length ? length : end
    if (end < 0) {
      end += length
    }
    length = start > end ? 0 : ((end - start) >>> 0)
    start >>>= 0

    var result = Array(length)
    while (++index < length) {
      result[index] = array[index + start]
    }
    return result
  }

  // 分块
  var chunk = (arr, size) => {
    let length = arr.length
    let index = 0
    let resIndex = 0
    let result = new Array(size)
    let chunkSize = length / size
    while (index < length) {
      result[resIndex++] = baseSlice(arr, index, index += chunkSize)
    }
    return result
  }
  // 生成随机数 (只包含 数字)
  var randomNumber = (size) => {
    let numberArr = []
    let randomNumber = ''
    let index = 0
    let len = 0
    let numberLen = 10
    for (let i = 0; i < numberLen; i++) {
      numberArr.push(i)
    }
    len = numberArr.length
    while (index++ < size) {
      randomNumber += numberArr[Math.floor(Math.random() * len)]
    }
    return randomNumber
  }

  // pai排序：从大到小 根据 pai 的权重weight 做判断，当权重相同时 对比 类型的权重
  var pokerSort = (pokerArr) => {
    pokerArr.sort((pokerA, pokerB) => {
      if (pokerA.weight < pokerB.weight) {
        return 1
      } else if (pokerA.weight === pokerB.weight) {
        if (pokerA.type < pokerB.type) {
          return 1
        } else {
          return -1
        }
      } else {
        return -1
      }
    })
    return pokerArr
  }

  let pokerType = {
    spades: 0,
    heart: 1,
    clubs: 2,
    diamonds: 3,
    king: 4
  }
  let playerType = {
    slave: 0,
    bigwigs: 1
  }

  class Poker {
    constructor (type, value, weight) {
      this.type = type
      this.value = value
      this.weight = weight
    }
  }

  class Player {
    constructor (id) {
      this.id = id
      this.total = 0
      this.identity = playerType.slave
      // 持有的
      this.holdPoker = []
      this.proxy = false
    }

  }

  const basicScore = 6

  class Room {
    constructor (poker = [], basicScore = basicScore) {
      // 底牌数量
      this.cards = 3
      let randomLen = 6
      this.minPlayer = 3
      this.maxPlayer = 3
      this.id = randomNumber(randomLen)
      this.players = []
      this.pokerPai = poker
      this.basicScore = basicScore
      this.canAction = false
      this.cardsPai = []
    }

    addPlayer (player) {
      // 越界检查
      if (this.players.length > this.maxPlayer) {
        return
      }
      this.players.push(player)
    }

    setCards () {
      this.cardsPai = this.pokerPai.splice(0, this.cards)
    }

    getPlayerLen () {
      return this.players.length
    }

    // 随机分配 bigwigs角色  并获取 特权
    randomAllotByKing (amount = 1, rule = playerType.bigwigs) {
      let checkedPlayer = this.players[Math.floor(Math.random() * this.getPlayerLen())]
      checkedPlayer.identity = playerType.bigwigs
      checkedPlayer.holdPoker.push(...this.cardsPai)
      pokerSort(checkedPlayer.holdPoker)
    }

    // 分配pai
    allot () {
      let result = chunk(this.pokerPai, this.getPlayerLen())
      this.players.map((player, index) => player.holdPoker = result[index])
    }

    // 整理 排序
    pokerSort () {
      // 延迟 2s 排序
      let delayTime = 2000
      setTimeout(() => {
        this.players.map((player, index) => pokerSort(player.holdPoker))
        this.randomAllotByKing()
      }, delayTime)
    }

    initRoom () {
      this.cardsPai = []
      this.players.forEach((player, index) => player.holdPoker = [])
    }

    /* // 下一个操作
    nextPlayer () {

    }*/

  }

  var pokerPai = []
  export default {
    data () {
      return {
        pokerArr: [],
        room: ''
      }
    },
    created () {
      let time = this.test(this.init)
      console.log(time)
    },
    methods: {
      init () {
        this.initPai()
        // 初始化 room
        this.room = new Room(this.pokerArr)
        console.log('room id', this.room.id)
        // 添加人数
        this.room.addPlayer(new Player('1'))
        this.room.addPlayer(new Player('2'))
        this.room.addPlayer(new Player('3'))
      },
      send () {
        if (this.room.players.length < this.room.minPlayer) {
          alert('不准')
          return
        }
        let time = this.test(() => {
          this.room.setCards()
          this.room.allot()
          this.room.pokerSort()
        })
        console.log('send', time)
      },
      reset () {
        this.initPai()
        this.initRoom()
      },
      initPai () {
        this.pokerArr = []
        // 初始化 pai
        if (pokerPai.length === 0) {
          let pokerNumber = []
          let maxNumber = 10
          for (let i = 3; i <= maxNumber; i++) {
            pokerNumber.push(i)
          }
          pokerNumber.push('J')
          pokerNumber.push('Q')
          pokerNumber.push('K')
          pokerNumber.push('A')
          pokerNumber.push('2')
          pokerNumber.forEach((number, index) => {
            for (let type in pokerType) {
              if (Object.prototype.hasOwnProperty.call(pokerType, type) && pokerType[type] !== pokerType.king) {
                this.pokerArr.push(new Poker(pokerType[type], number, index))
              }

            }
          })
          this.pokerArr.push(new Poker(pokerType.king, ' king', pokerNumber.length))
          this.pokerArr.push(new Poker(pokerType.king, 'big king', pokerNumber.length + 1))
          console.log(this.pokerArr)
        } else {
          this.pokerArr = pokerPai
        }
        // 乱序
        shuffle(this.pokerArr)
        console.log('乱序', this.pokerArr)
      },
      test (func) {
        let start = new Date().getTime()
        func()
        let end = new Date().getTime()
        return (end - start) + 'ms'
      }
    }
  }


</script>
<style lang="scss" scoped type="text/css">
</style>
