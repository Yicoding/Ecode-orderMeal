<template>
  <div class="shop-box">
    <div class="shop-content" ref="shopContent">
      <div style="height: 30px; background-color: #eee;"></div>
      <ul class="menu-ul" v-show="listData.length > 0">
        <li v-for="item in listData" :key="item.id" @click="jump(item)">
          <div style="float: left; width: 8%; height: 120px; position: relative;">
            <div class="radio-check" v-show="item.isCheck == 'false'" @click.stop="checkLine(item)"></div>
            <div class="radio-not-check" v-show="item.isCheck == 'true'" @click.stop="checkLine(item)">&radic;</div>
          </div>
          <div style="float: left; width: 92%;">
            <div class="menu-left-shop">
              <img v-lazy="item.picture" alt=""/>
            </div>
            <div class="menu-right">
              <div>
                <h4 v-text="item.name"></h4>
                <p style="margin-top: 12px;" v-text="item.descript"></p>
              </div>
              <div class="munu-price">
                <div class="left" style="width: 40%;">
                  &yen;{{item.price}}
                </div>
                <div class="menu-shop">
                  <transition name="fade">
                    <div v-if="item.total > 1" class="shop-first" @click.stop="remove(item)">-</div>
                  </transition>
                  <transition name="fade">
                    <div v-if="item.total > 0" class="shop-second" v-text="item.total"></div>
                  </transition>
                  <div class="shop-third" @click.stop="add(item)">+</div>
                </div>
              </div>
            </div>
          </div>
        </li>
      </ul>
      <div v-show="listData.length == 0" class="empty-content">购物车空空如也，快去一些添加进来吧 ----</div>
    </div>
    <div class="shop-footer">
      <div class="shop-footer-left">
        <div style="float: left; height: 50px; position: relative;">
          <div class="radio-check-foot" v-show="!isCheckAll" @click="CheckAll('true')"></div>
          <div class="radio-not-check-foot" v-show="isCheckAll" @click="CheckAll('false')">&radic;</div>
        </div>
        <mt-button class="left" size="small" type="danger" :plain="true" style="margin: 8px 5px 0 15px;" @click="deleteCheck">删除</mt-button>
        <div class="left">合计：&yen;{{priceAll}}</div>
      </div>
      <div class="shop-footer-right" @click="gosettle">去结算 <span style="font-size: 0.8em;">({{checkNum}})</span></div>
    </div>
    <transition name="fade">
      <div class= "returnTop"  @click="backToTop" v-show="show"></div>
    </transition>
  </div>
</template>

<script>
import { MessageBox } from 'mint-ui'
export default {
  activated () {
    setTimeout(() => {
      this.findall()
    }, 500)
  },
  data () {
    return {
      listData: [],
      isCheckAll: false,
      checkNum: 0,
      scrollTop: 0,
      show: false,
      interval: null
    }
  },
  mounted() {
    this.$nextTick(() => {
      this.$refs.shopContent.addEventListener('scroll', this.handleScroll, true)
    })
  },
  computed: {
    user () {
      return this.$store.state.user
    },
    shopNum () {
      return this.$store.state.shopNum
    },
    priceAll () {
      return this.$store.state.priceAll
    },
  },
  methods: {
    handleScroll() {
      this.scrollTop = window.innerHeight || document.documentElement.clientHeight || document.body.clientHeight || 0
      if (this.$refs.shopContent.scrollTop >= this.scrollTop / 2) {
        this.show = true 
      } else {
        this.show = false
      }
    },
    backToTop() {
      clearInterval(this.interval)
      this.interval = setInterval(() => {
        var current = this.$refs.shopContent.scrollTop;
        var step = (0 - current) / 10;
        step = Math.ceil(step);
        current += step;
        if(current <= 10) {
          current = 0 
        }
        this.$refs.shopContent.scrollTop = current
        if (current <= 0) {
          clearInterval(this.interval)
        }
      }, 10)
    },
    findall () {
      this.checkNum = 0
      this.isCheckAll = false
      this.$http.get(this.resource + '/api/shop/findall', {params: {uid: this.user.id}}).then(res => {
        if (res.data.length > 0) {
          console.log(JSON.stringify(res.data))
          this.listData = res.data
          let priceAll = 0
          let isAll = true

          res.data.forEach((item) => {
            if (item.isCheck == 'true'){
              priceAll += item.total * item.price
              this.checkNum += item.total
            } else {
              isAll = false
            }
          })
          this.$store.dispatch('getshopNum', this.checkNum)
          isAll && (this.isCheckAll = true)
          this.$store.dispatch('getpriceAll', priceAll)
        } else {
          this.listData = []
          this.$store.dispatch('getpriceAll', 0)
        }
      })
    },
    checkLine (item) {
      console.log(item.isCheck)
      if (item.isCheck == 'true') {
        this.checkNum -= item.total
        this.$store.dispatch('getshopNum', this.checkNum)
        this.isCheckAll = false
        item.isCheck = 'false'
        this.$store.dispatch('getonepriceAll', {action: 'remove', price: item.price * item.total})
        this.$http.put(this.resource + '/api/shop/changeone', {isCheck: 'false', user_id: this.user.id, menu_id: item.menu_id}).then(() => {
          
        })
      } else {
        this.checkNum += item.total
        this.$store.dispatch('getshopNum', this.checkNum)
        console.log(JSON.stringify(this.listData))
        item.isCheck = 'true'
        this.$store.dispatch('getonepriceAll', {action: 'add', price: item.price * item.total})
        this.$http.put(this.resource + '/api/shop/changeone', {isCheck: 'true', user_id: this.user.id, menu_id: item.menu_id}).then(() => {
          let isAll = true
          this.listData.forEach((item) => {
            if (item.isCheck == 'false'){
              isAll = false
            }
          })
          isAll && (this.isCheckAll = true)
        })
      }
    },
    CheckAll (value) {
      this.isCheckAll = !this.isCheckAll
      this.$http.put(this.resource + '/api/shop/multiple', {isCheck: value, user_id: this.user.id}).then(() => {
        this.findall()
      })
    },
    add (item) {
      item.total ++
      this.$http.put(this.resource + '/api/shop/put', {user_id: this.user.id, menu_id: item.menu_id, action: 'add'}).then((res) => {
        console.log(JSON.stringify(res.data))
        if (item.isCheck == 'true') {
          this.$store.dispatch('getonechangeNum', 'add')
          this.checkNum ++
          this.$store.dispatch('getshopNum', this.checkNum)
          this.$store.dispatch('getonepriceAll', {action: 'add', price: item.price})
        }
      })
    },
    remove (item) {
      item.total > 1 && item.total --
      && this.$http.put(this.resource + '/api/shop/put', {user_id: this.user.id, menu_id: item.menu_id, action: 'remove'}).then((res) => {
        console.log(JSON.stringify(res.data))
        if (item.isCheck == 'true') {
          this.$store.dispatch('getonechangeNum', 'remove')
          this.checkNum --
          this.$store.dispatch('getshopNum', this.checkNum)
          this.$store.dispatch('getonepriceAll', {action: 'remove', price: item.price})
        }
      })
    },
    // 删除选中的商品
    deleteCheck () {
      let dataArr = []
      this.listData.forEach((item) => {
        if (item.isCheck == 'true') {
          dataArr.push(item.menu_id)
        }
      })
      if (dataArr.length > 0) {
        MessageBox.confirm('确定要删除选中的商品吗?').then(action => {
          // console.log(JSON.stringify(this.listData))
          // console.log(dataArr)
          // console.log(String(dataArr))
          this.$http.delete(this.resource + '/api/shop/deleteCheck', {params: {data: String(dataArr), user_id: this.user.id}}).then((res) => {
            this.findall()
            this.$toast('删除成功')
          })
        })
      } else {
        this.$toast('请至少选中一件商品才能删除')
      }
    },
    // 跳转到单个商品详情
    jump (item) {
      this.$router.push({path: '/' + this.user.role.name + '/menuinfo', query: {id: item.menu_id}})
    },
    // 去结算
    gosettle () {
      console.log(JSON.stringify(this.listData))
      let dataArr = []
      this.listData.forEach((item) => {
        if (item.isCheck == 'true') {
          dataArr.push(item)
        }
      })
      if (dataArr.length > 0) {
        MessageBox.confirm('确定要购买选中的商品吗?').then(action => {
          this.$router.push({path: '/orderlist', query: {dataArr: JSON.stringify(dataArr), priceAll: this.priceAll, menuNum: this.checkNum, isDelete: 'true'}})
        })
      } else {
        this.$toast('您还没有选择商品哦')
      }
    },
  },
  beforeDestroy() {
   this.$refs.shopContent.removeEventListener('scroll', this.handleScroll)
    if (this.interval) {
      clearInterval(this.interval)
    }
  },
}
</script>

<style scoped>
  .fade-enter-active, .fade-leave-active {  
    transition: opacity .5s;
  }
  .fade-enter, .fade-leave-to {
    opacity: 0
  }
  .returnTop {
    position: fixed;
    display: inline-block;
    text-align: center;
    cursor: pointer;
    right: 12px;
    bottom: 200px;
    width: 40px;
    height: 40px;
    border-radius: 4px;
    line-height: 45px;
    background-image: url("../assets/ruturnTop.png");
    background-size: 40px 40px;
  }
  
</style>
