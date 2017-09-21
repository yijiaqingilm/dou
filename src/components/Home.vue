<template>
  <div>
    <div @click="send">派发</div>
    <div @click="reset">reset</div>
    <div>room{{room.id}}, score {{room.basicScore}}</div>
    <div>底pai:
      <div v-for="cards in room.cardsPai">{{cards.type}}, {{cards.value}} ,权重 {{cards.weight}}</div>
    </div>
    <items v-if="room.seatList.head" @action="action" @pass="pass" :seat="room.seatList.head.next"
           :head="room.seatList.head.id"></items>

  </div>

</template>

<script type="text/ecmascript-6">
  import _ from 'lodash'
  import Items from '@/components/childNode/items.vue'
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
    constructor (type, value, weight, continues = false) {
      this.type = type
      this.value = value
      this.weight = weight
      this.checked = false
      this.continues = continues
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
      this.action = false
      this.nextPlayer = null
    }
  }

  class Seat {
    constructor (id, player = null) {
      this.id = id
      this.player = player
      this.next = null
      this.prev = null
      this.action = false
    }
  }

  var SeatList = (function () {
    let getMaxId = (function () {
      let id = 0
      return function () {
        return id++
      }
    })()

    let arrEach = function (seatList, iterator) {
      let currentSeat = seatList.head
      let index = 0
      while (currentSeat.next && currentSeat.next.id !== seatList.head.id) {
        currentSeat = currentSeat.next
        iterator(currentSeat, index++)
      }
      return seatList
    }

    class SeatList {
      constructor () {
        this.head = new Seat(getMaxId())
        this.head.next = this.head
        this.head.prev = this.head
        // this.
        // this.insert = insert;
        // this.remove = remove;
      }

      find (id) {
        let currentSeat = this.head
        while (currentSeat.id !== id && currentSeat.next.id !== this.head.id) {
          currentSeat = currentSeat.next
        }
        return currentSeat
      }

      first () {
        return this.head.next
      }

      last () {
        let lastSeat = this.head
        while (lastSeat.next && lastSeat.next.id !== this.head.id) {
          lastSeat = lastSeat.next
        }
        return lastSeat
      }

      add () {
        let newId = getMaxId()
        let newSeat = new Seat(newId)
        let lastSeat = this.last()
        // 新节点的上节点指向上个节点，下节点指向head，旧的最后一个节点的下节点指向新节点，head的上节点指向新节点
        newSeat.prev = lastSeat
        newSeat.next = this.head
        lastSeat.next = newSeat
        this.head.prev = newSeat

        return newSeat
      }

      nextSeat (seat) {
        if (seat.next !== this.head) {
          return seat.next
        } else {
          return seat.next.next
        }

      }

      prevSeat (seat) {
        if (seat.prev !== this.head) {
          return seat.prev
        } else {
          return seat.prev.prev
        }
      }

      seatForEach (iteratee) {
        return arrEach(this, iteratee)
      }
    }

    return SeatList
  })()

  var Room = (function () {
    const basicScore = 6
    let randomLen = 6
    // 底牌数量
    let cards = 3
    let minPlayer = 3
    let maxPlayer = 3

    class Room {
      constructor (poker = [], basicScore = basicScore) {
        this.id = randomNumber(randomLen)
        this.pokerPai = poker
        this.basicScore = basicScore
        this.canAction = false
        this.cardsPai = []
        this.actionSeat = null
        this.seatList = new SeatList()
        this.outPoker = []
        this.prevAction = {
          poker: [],
          seat: null
        }
      }

      getMinPlayer () {
        return minPlayer
      }

      addPlayer (player) {
        // 越界检查
        /* if (this.seatWrap.length > maxPlayer) {
          return
        }*/
        let seat = this.seatList.add()
        seat.player = player
        console.log('push', seat)
      }

      // 设置底牌
      setCards () {
        this.cardsPai = this.pokerPai.splice(0, cards)
      }

      // 随机分配座位 bigwigs角色  并获取 出手的权利
      randomAllotByKing (amount = 1, rule = playerType.bigwigs) {
        let randomNumber = Math.floor(Math.random() * minPlayer)
        let checkedSeat = this.seatList.find(randomNumber + 1)
        checkedSeat.player.identity = rule
        checkedSeat.player.holdPoker.push(...this.cardsPai)
        checkedSeat.action = true
        pokerSort(checkedSeat.player.holdPoker)
        this.actionSeat = checkedSeat
      }

      // 分配pai
      allot () {
        // 这里需要优化 length 写死
        let len = 3
        let result = chunk(this.pokerPai, len)
        this.seatList.seatForEach((seat, index) => {
          seat.player.holdPoker = result[index]
        })
      }

      // 整理 排序
      pokerSort () {
        // 延迟 2s 排序
        let delayTime = 2000
        setTimeout(() => {
          this.seatList.seatForEach((seat, index) => pokerSort(seat.player.holdPoker))
          this.randomAllotByKing()
        }, delayTime)
      }

      initRoom () {
        this.cardsPai = []
        this.actionSeat = []
        this.seatList.seatForEach((seat, index) => {
          seat.player.holdPoker = []
          seat.player.identity = playerType.slave
          seat.action = false
        })
      }

      /**
       * 分析出手
       * @param pokerArr
       * @return {boolean}
       */
      resolvePoker (pokerArr = []) {
        // 是否可出
        let isResolve = true
        // key ==> type,value=amount
        let sortPoker = {}
        // 最多type的数量
        let maxAmount = 0
        // 最多的type集合
        let maxTypeList = []
        // 其他类型的数量
        let otherTypeList = []
        // 最多的type 是否是连续的
        let isCountinues = false
        pokerArr.forEach((poker) => {
          if (sortPoker[poker.weight]) {
            sortPoker[poker.weight].amount++
          } else {
            sortPoker[poker.weight] = {amount: 1, continues: poker.continues, weight: poker.weight}
          }
        })

        for (let key in sortPoker) {
          if (sortPoker.hasOwnProperty(key) && sortPoker[key].amount > maxAmount) {
            maxAmount = sortPoker[key].amount
          }
        }
        for (let key in sortPoker) {
          if (sortPoker.hasOwnProperty(key) && sortPoker[key].amount === maxAmount) {
            maxTypeList.push(sortPoker[key])
          } else {
            otherTypeList.push(sortPoker[key])
          }
        }
        // 首先判断是否满足连续条件 ==》连续的次数maxTypeList.length
        if (maxTypeList.length >= rule[maxTypeList[0].amount].minContinues) {
          // 排序 ， 2.判断是否是连续的
          pokerSort(maxTypeList)
          for (let i = 0; i < maxTypeList.length; i++) {
            // 当前对象是否可允许连续
            if (maxTypeList[i].continues) {
              // 是否所有的最大的类型 满足连续
              if ((i + 1) < maxTypeList.length) {
                if (maxTypeList[i].weight - 1 === maxTypeList[i + 1].weight) {
                  continue
                } else {
                  isCountinues = false
                  break
                }
              } else {
                isCountinues = true
                break
              }
            } else {
              isCountinues = false
              break
            }
          }
        } else {
          isCountinues = false
        }
        if (isCountinues) {
          let composes = rule[maxTypeList[0].amount].compose
          if (composes.length > 0) {
            let otherType = otherTypeList.length
            let otherAmount = otherTypeList.map((item) => item.amount).reduce((a, b) => a + b, 0)
            isResolve = composes.some((compose) => (compose.amount * maxTypeList.length === otherAmount || compose.amount === otherAmount) && compose.type * maxTypeList.length === otherType)
          } else {
            if (otherTypeList.length === 0) {
              isResolve = true
            } else {
              isResolve = false
            }
          }
        } else {
          if (maxTypeList.length === 1) {
            let composes = rule[maxTypeList[0].amount].compose
            if (composes.length > 0) {
              let otherType = otherTypeList.length
              let otherAmount = otherTypeList.map((item) => item.amount).reduce((a, b) => a + b, 0)
              isResolve = composes.some((compose) => (compose.amount * maxTypeList.length === otherAmount || compose.amount === otherAmount) && compose.type * maxTypeList.length === otherType)
            } else {
              if (otherTypeList.length === 0) {
                isResolve = true
              } else {
                isResolve = false
              }
            }
          } else {
            isResolve = false
          }

        }
        return isResolve
      }

      /**
       * 压制出手
       * @param currentPoker
       * @param prevPoker
       * @return {boolean}
       */
      suppressedPoker (currentPoker = [], prevPoker = []) {
        // if 出手poker len 一样
        if (currentPoker.length === prevPoker.length) {
          // -----出手poker 组成一样
          // -----出手poker 组成类型最小的 要大于被压制的poker组成类型最小的
          return true
        } else {
          // else 出手poker len 不一样
          // poker 组成 为4张类型的
          // poker 组成 为2张类型的

          return true
        }
      }

      /**
       * 分析poker
       * @param poker
       * @return {*}
       */
      analyze (pokerArr) {
        // 是否可出
        let isResolve = true
        let sortPoker = {}
        // 最多type的数量
        let maxAmount = 0
        // 最多的type集合
        let maxTypeList = []
        // 其他类型的数量
        let otherTypeList = []
        // 最多的type 是否是连续的
        let isCountinues = false
        return {
          // 最大的类型
          maxType: 0,
          // 最大的类型的数量
          maxTypeAmount: 0,
          // 最大的类型是否满足连续
          isContinue: false,
          // 最大的类型中最小的值
          maxTypeOfMinValue: 0,
        }
      }

      /**
       * 检查是否满足出手条件
       * @param checkPokerArr
       * @param prevpokerArr
       * @return {*}
       */
      checkAction (checkPokerArr, prevpokerArr = []) {
        if (prevpokerArr.length > 0) {
          console.log('checked', checkPokerArr)
          return this.suppressedPoker(checkPokerArr, prevpokerArr)
        } else {
          return this.resolvePoker(checkPokerArr)
        }

      }

      action () {
        let checkPoker = []
        checkPoker = this.actionSeat.player.holdPoker.filter((poker) => poker.checked)
        let action = this.checkAction(checkPoker, this.actionSeat === this.prevAction.seat ? [] : this.prevAction.poker)
        if (action) {
          checkPoker.forEach((poker) => {
            let index = this.actionSeat.player.holdPoker.indexOf(poker)
            if (index !== -1) {
              this.actionSeat.player.holdPoker.splice(index, 1)
              this.outPoker.push(poker)
            }
          })
          // 记录这次行动
          this.prevAction.poker = [].concat(checkPoker)
          this.prevAction.seat = this.actionSeat
          // 行动完 不能再行动
          this.actionSeat.action = false
          // 设置下一个行动的人
          this.actionSeat = this.seatList.nextSeat(this.actionSeat)
          this.actionSeat.action = true
        } else {
          this.actionSeat.player.holdPoker.forEach((poker) => poker.checked = false)
        }
        console.log('是否可出', action)
      }

      pass () {
        // 直接不行动
        this.actionSeat.action = false
        // 设置下一个行动的人
        this.actionSeat = this.seatList.nextSeat(this.actionSeat)
        this.actionSeat.action = true
      }

    }

    return Room
  })()
  const rule = {
    // key poker type of value
    '1': {
      minContinues: 5,
      compose: []
    },
    '2': {
      minContinues: 3,
      compose: []
    },
    '3': {
      minContinues: 2,
      compose: [{amount: 0, type: 0}, {amount: 1, type: 1}, {amount: 2, type: 1}]
    },
    '4': {
      minContinues: 2,
      compose: [{amount: 0, type: 0}, {amount: 1, type: 1}, {amount: 2, type: 2}, {amount: 2, type: 1}, {
        amount: 4,
        type: 2
      }]
    }
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
      console.log('this.room', this.room)
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
        // 这里需要优化 目前写死 3
        /* let len = 3
        if (len < this.room.getMinPlayer()) {
          alert('不准操作')
          return
        }*/
        this.room.setCards()
        this.room.allot()
        this.room.pokerSort()
        console.log('seatList', this.room.seatList)
      },
      reset () {
        this.initPai()
        this.room.pokerPai = this.pokerArr
        this.room.initRoom()
      },
      initPai () {
        this.pokerArr = []
        // 初始化 pai
        if (pokerPai.length === 0) {
          let pokerNumber = []
          let maxNumber = 10
          for (let i = 3; i <= maxNumber; i++) {
            pokerNumber.push(i + '')
          }
          pokerNumber.push('J')
          pokerNumber.push('Q')
          pokerNumber.push('K')
          pokerNumber.push('A')
          pokerNumber.push('2')
          let noContinuesArr = ['2']
          pokerNumber.forEach((number, index) => {
            for (let type in pokerType) {
              if (Object.prototype.hasOwnProperty.call(pokerType, type) && pokerType[type] !== pokerType.king) {
                this.pokerArr.push(new Poker(pokerType[type], number, index, !noContinuesArr.includes(number)))
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
      },
      findSeat (id) {
        return this.room.seatList.find(id)
      },
      action () {
        console.log('是否执行？')
        this.room.action()
      },
      pass () {
        this.room.pass()
      }
    },
    components: {Items},
    watch: {
      'room.seatList.head': function (value, old) {
        console.log('watch========================', value, old)
      }
    }
  }


</script>
<style lang="scss" scoped type="text/css">
</style>
