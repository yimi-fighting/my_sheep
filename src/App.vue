<template>
  <div class="contain">
    <div class="adverstive" v-if="adverstive">
      <div class="time">广告中:{{ time }}s</div>
    </div>
    <div v-if="step === 0" class="intro">
      <div>
        横向卡片最大平铺排数
        <input v-model="option.x" min="2" max="10" type="range" />
        {{ option.x }}
      </div>
      <div>
        纵向卡片最大平铺排数
        <input v-model="option.y" min="2" max="10" type="range" />
        {{ option.y }}
      </div>
      <div>
        卡片最大堆叠层数
        <input v-model="option.z" min="2" max="10" type="range" />
        {{ option.z }}
      </div>
      <div>
        卡片密度
        <input
          v-model="option.cardRandom"
          min="0"
          max="1"
          step="0.1"
          type="range"
        />
        {{ option.cardRandom }}
      </div>
      <div>
        最大卡片种类
        <input
          v-model="option.maxCardType"
          min="3"
          max="14"
          step="1"
          type="range"
        />
        {{ option.maxCardType }}
      </div>
      <br />
      <button @click="startGame">开始游戏</button>
    </div>
    <div v-else-if="step === 2" class="intro">
      <h1>{{ result ? "You Win！🎉" : "You Lose!😢" }}</h1>
      <button @click="rePlay">再来一轮</button>
      <button @click="setGame">难度调节</button>
    </div>
    <div v-else class="box">
      <div class="card-wrap" :style="cardWrapStyle">
        <div
          v-for="item in cardItemList"
          :key="item.key"
          :class="{ 'item-cover': item.cover }"
          class="card-item"
          :style="item.style"
          @click="clickCard(item)"
        >
          {{ item.content }}
        </div>
        <div
          v-for="item in penddingList"
          :key="item.key"
          class="card-item"
          :style="item.style"
        >
          {{ item.content }}
        </div>
        <div
          v-for="item in saveList"
          :key="item.key"
          class="card-item"
          :style="item.style"
          @click="clickSaveCard(item)"
        >
          {{ item.content }}
        </div>
        <p class="card-tips">剩余空位:{{ 7 - penddingList.length }}/7；</p>
      </div>
      <div class="tools">
        道具：
        <button :disabled="!tools.save" @click="saveCard">取出3个卡片</button>
        <button :disabled="!tools.rand" @click="randCard">随机</button>
        <button @click="rePlay">再来一轮</button>
      </div>
    </div>
  </div>
</template>

<script>
// 卡片类
class CardItem {
  static x = 20;
  static y = 21;
  // 颜色数组
  static colorType = {
    1: { background: "#FFB7DD" },
    2: { background: "#FFCCCC" },
    3: { background: "#FFC8B4" },
    4: { background: "#FFDDAA" },
    5: { background: "#FFEE99" },
    6: { background: "#FFFFBB" },
    7: { background: "#EEFFBB" },
    8: { background: "#CCFF99" },
    9: { background: "#99FF99" },
    10: { background: "#BBFFEE" },
    11: { background: "#AAFFEE" },
    12: { background: "#99FFFF" },
    13: { background: "#CCEEFF" },
    14: { background: "#CCDDFF" },
  };
  // 卡片种类
  static contentType = {
    1: "🥕",
    2: "✂️",
    3: "🥦",
    4: "🥛",
    5: "🌊",
    6: "🧤",
    7: "🧵",
    8: "🌱",
    9: "🔨",
    10: "🌽",
    11: "🌾",
    12: "🐑",
    13: "🪵",
    14: "🔥",
  };

  constructor({ x, y, z, key }) {
    this.x = x;
    this.y = y;
    this.z = z;
    this.key = key;
    const offset = z * 0;
    this.val = key;
    this.style = {
      top: y * CardItem.y + offset + "px",
      left: x * CardItem.x + offset + "px",
      // width: CardItem.x * 2 + "px",
      width: CardItem.x * 2 - 2 + "px",
      height: CardItem.y * 2 - 8 + "px",
      // height: CardItem.y * 2 + "px",
    };
  }

  setValue(val) {
    this.val = val;
    // 设置卡片种类
    this.content = CardItem.contentType[val];
    // 给卡片添加颜色样式
    Object.assign(this.style, CardItem.colorType[val]);
  }
}
export default {
  data() {
    return {
      step: 0,
      option: {
        x: 6,
        y: 4,
        z: 8,
        cardRandom: 0.2,
        maxCardType: 11,
      },
      win: false,
      cardMap: [],
      // 存放卡片信息列表
      cardItemList: [],
      // 存储池列表
      penddingList: [],
      saveList: [],
      calcValueList: [],
      xUnit: 0,
      yUnit: 0,
      tools: {
        save: true,
        rand: true,
      },
      timer: 0,
      // 是否显示广告
      adverstive: false,
      // 广告时间
      time: 3,
    };
  },
  methods: {
    // 开始游戏按钮
    initGame() {
      this.step = 1;
      // 获取地图
      this.getmap(this.option);
      this.penddingList = [];
      this.saveList = [];
      this.tools.save = true;
      this.tools.rand = true;
      // 根据最大卡片种类，为每个位置设置卡片种类
      this.setCardValue({ maxCardType: Number(this.option.maxCardType) });
      this.calcCover();
    },

    // 构建地图
    getmap({ x, y, z, cardRandom } = {}) {
      // x,y*2是因为每一张卡片占四个小格子
      this.xUnit = x * 2;
      this.yUnit = y * 2;
      // 初始化地图daxiao
      const cardMap = new Array(z);
      for (let k = 0; k < z; k++) {
        cardMap[k] = new Array(this.yUnit);
        for (let i = 0; i < this.yUnit; i++) {
          cardMap[k][i] = new Array(this.xUnit).fill(0);
        }
      }
      // 判断那些位置可以放置卡片，将卡片信息存储在cardItemList中
      const cardItemList = [];
      let key = 0;
      for (let k = 0; k < z; k++) {
        const shrinkSpeed = 3;
        const shrink = Math.floor((z - k - 1) / shrinkSpeed);
        const shrinkX = Math.min(Math.floor(this.xUnit / 2) - 2, shrink);
        const shrinkY = Math.min(Math.floor(this.yUnit / 2) - 2, shrink);
        // 行
        for (let i = shrinkY; i < this.yUnit - 1 - shrinkY; i++) {
          // 列
          for (let j = shrinkX; j < Math.ceil((this.xUnit - 1) / 2); j++) {
            let canSetCard = true;
            if (j > 0 && cardMap[k][i][j - 1]) {
              // 左边不能有牌
              canSetCard = false;
            } else if (i > 0 && cardMap[k][i - 1][j]) {
              // 上边不能有牌
              canSetCard = false;
            } else if (i > 0 && j > 0 && cardMap[k][i - 1][j - 1]) {
              // 左上不能有牌
              canSetCard = false;
            } else if (i > 0 && cardMap[k][i - 1][j + 1]) {
              // 右上不能有牌
              canSetCard = false;
            } else if (k > 0 && cardMap[k - 1][i][j]) {
              // 正底不能有牌
              canSetCard = false;
            } else if (Math.random() >= cardRandom) {
              // 根据卡片密度，随机设置
              canSetCard = false;
            }
            if (canSetCard) {
              key++;
              const cardItem = new CardItem({ x: j, y: i, z: k, key });
              cardMap[k][i][j] = cardItem;
              cardItemList.push(cardItem);
              // 对称放置
              const mirrorX = this.xUnit - 2 - j;
              if (mirrorX > j) {
                key++;
                const cardItem = new CardItem({
                  x: mirrorX,
                  y: i,
                  z: k,
                  key,
                });
                cardMap[k][i][mirrorX] = cardItem;
                cardItemList.push(cardItem);
              }
            }
          }
        }
      }
      // 为了达到卡片成三倍数的关系，去掉顶层多出来的卡片
      cardItemList.reverse();
      // 将放置卡片的位置设置为3的倍数
      for (let i = 1; i <= key % 3; i++) {
        const clearItem = cardItemList.pop();
        cardMap[clearItem.z][clearItem.y][clearItem.x] = 0;
      }
      cardItemList.reverse();
      this.cardMap = cardMap;
      this.cardItemList = cardItemList;
    },

    // 设置卡片种类
    setCardValue({ maxCardType } = {}) {
      // 卡片种类
      const valStack = new Array(maxCardType);
      this.calcValueList = new Array(maxCardType + 1).fill(0);
      // 给卡片设置值
      this.cardItemList.forEach((item) => {
        // 随机一个种类
        const value = Math.ceil(Math.random() * maxCardType);
        if (valStack[value]) {
          // 将item赋值这个种类的卡片
          valStack[value].push(item);
          // 将他们进行三个三个配对
          if (valStack[value].length === 3) {
            valStack[value].forEach((item) => {
              item.setValue(value);
            });
            valStack[value] = null;
          }
        } else {
          valStack[value] = [item];
        }
      });

      // 将未进行配对成功的部分，重新配对
      let count = 2;
      valStack.forEach((list, index) => {
        list &&
          list.forEach((item) => {
            count++;
            item.setValue(Math.floor(count / 3));
          });
      });
    },

    // 计算遮挡关系
    calcCover() {
      // 构建一个遮挡 map
      // console.log(this.xUnit);
      let coverMap = new Array(this.yUnit);
      for (let i = 0; i <= this.yUnit; i++) {
        coverMap[i] = new Array(this.xUnit).fill(false);
      }
      // 从后往前，后面的层数高
      for (let i = this.cardItemList.length - 1; i >= 0; i--) {
        const item = this.cardItemList[i];
        const { x, y, z } = item;
        if (coverMap[y][x]) {
          item.cover = true;
        } else if (coverMap[y][x + 1]) {
          item.cover = true;
        } else if (coverMap[y + 1][x]) {
          item.cover = true;
        } else if (coverMap[y + 1][x + 1]) {
          item.cover = true;
        } else {
          item.cover = false;
        }
        // 将卡片的四个位置，在遮挡map中设置为true
        coverMap[y][x] = true;
        coverMap[y + 1][x] = true;
        coverMap[y][x + 1] = true;
        coverMap[y + 1][x + 1] = true;
      }
    },
    // 点击取出的三张卡片中的卡片
    clickSaveCard(item) {
      this.cardItemList.push(item);
      const index = this.saveList.indexOf(item);
      this.saveList = this.saveList
        .slice(0, index)
        .concat(this.saveList.slice(index + 1));
      this.clickCard(item);
    },

    removeThree() {
      this.penddingList.some((item) => {
        // 判断pendding中是否可以消除
        if (this.calcValueList[item.val] === 3) {
          // 清除peddinglist中三张重复的
          this.penddingList = this.penddingList.filter((newItem) => {
            return newItem.val !== item.val;
          });
          // 移动penddinglist中的卡片
          this.penddingList.forEach((item, index) => {
            item.style.top = "160%";
            item.style.left = this.leftOffset + index * CardItem.x * 2 + "px";
          });
          this.calcValueList[item.val] = 0;
          // 如果卡片列表为空，step=2，胜利
          if (this.cardItemList.length === 0) {
            this.step = 2;
            this.result = true;
          }
        }
      });

      // 如果penddinglist列表>=7，step=2,失败
      if (this.penddingList.length >= 7) {
        this.step = 2;
        this.result = false;
      }
    },
    // 点击卡片
    clickCard(item) {
      // 清除定时器
      clearTimeout(this.timer);
      // 判断是否有三个相同的可以移除
      this.removeThree();
      // 将点击的卡片放入队列中
      this.penddingList.push(item);
      // 将点击的卡片从卡片总数组中移除
      const index = this.cardItemList.indexOf(item);
      this.cardItemList = this.cardItemList
        .slice(0, index)
        .concat(this.cardItemList.slice(index + 1));
      // 重新计算覆盖关系
      this.calcCover();
      // 对应的卡片种类+1
      this.calcValueList[item.val]++;
      setTimeout(() => {
        item.style.top = "160%";
        item.style.left =
          this.leftOffset +
          (this.penddingList.length - 1) * CardItem.x * 2 +
          "px";
      }, 0);

      this.timer = setTimeout(() => {
        this.removeThree();
      }, 500);
    },
    // 打乱cardItemList数组
    randCard() {
      new Promise((resolve) => {
        this.adverstive = true;
        this.time = 3;
        let timers = setInterval(() => {
          this.time--;
          if (this.time <= 0) {
            clearInterval(timers);
            this.adverstive = false;
            resolve();
          }
        }, 1000);
      }).then(() => {
        if (!this.tools.rand) {
          return;
        }
        this.tools.rand = false;
        const length = this.cardItemList.length;
        this.cardItemList.forEach((item) => {
          const randNum = Math.floor(length * Math.random());
          const newItem = this.cardItemList[randNum];
          let temp;
          temp = item.style.left;
          item.style.left = newItem.style.left;
          newItem.style.left = temp;
          temp = item.style.top;
          item.style.top = newItem.style.top;
          newItem.style.top = temp;
          temp = item.x;
          item.x = newItem.x;
          newItem.x = temp;
          temp = item.y;
          item.y = newItem.y;
          newItem.y = temp;
          temp = item.z;
          item.z = newItem.z;
          newItem.z = temp;
        });

        this.cardItemList.sort((a, b) => a.z - b.z);
        this.calcCover();
      });
    },

    // 从下面的列表中取出三张卡片
    saveCard() {
      new Promise((resolve) => {
        this.adverstive = true;
        this.time = 3;
        let timers = setInterval(() => {
          this.time--;
          if (this.time <= 0) {
            clearInterval(timers);
            this.adverstive = false;
            resolve();
          }
        }, 1000);
      }).then(() => {
        if (!this.tools.save) {
          return false;
        }
        this.tools.save = false;
        // 取出penddingList的前三张卡片
        this.saveList = this.penddingList.slice(0, 3);
        // 异步将前三张卡片向上移动
        setTimeout(() => {
          this.saveList.forEach((item, index) => {
            item.style.top = "110%";
            item.style.left = this.leftOffset + index * CardItem.x * 2 + "px";
            this.calcValueList[item.val]--;
          });
        }, 0);
        this.penddingList = this.penddingList.slice(3);
        this.penddingList.forEach((item, index) => {
          item.style.top = "160%";
          item.style.left = this.leftOffset + index * CardItem.x * 2 + "px";
        });
      });
    },
    // 开始
    startGame() {
      this.initGame();
    },
    // 设置
    setGame() {
      this.step = 0;
    },
    // 重来
    rePlay() {
      this.initGame();
    },
  },
  computed: {
    cardWrapStyle() {
      return {
        width: (this.xUnit + 2) * CardItem.x + "px",
        height: (this.yUnit + 1) * CardItem.y + "px",
      };
    },
    leftOffset() {
      const wrapWidth = (this.xUnit + 2) * CardItem.x;
      return (wrapWidth - 7 * CardItem.x * 2) / 2;
    },
  },
};
</script>

<style>
html {
  width: 100%;
  height: 100%;
  padding: 0px;
  margin: 0px;
}
.contain {
  width: 100%;
  height: 100%;
  position: absolute;
}
.adverstive {
  width: 100%;
  height: 100%;
  background: black;
  opacity: 0.2;
  position: absolute;
  z-index: 999;
  color: white;
}
.time {
  position: absolute;
  right: 10px;
}
.box {
  position: relative;
}

.intro {
  margin: 10% auto 0 auto;
  text-align: center;
}

.card-wrap {
  position: relative;
  margin: 10% auto 0 auto;
}

.card-item {
  font-size: 28px;
  text-align: center;
  position: absolute;
  border-radius: 2px;
  box-sizing: border-box;
  background: #ddd;
  opacity: 1;
  cursor: pointer;
  transition: all 0.3s;
  box-shadow: 0px 3px 0 0 #fff, 0 8px 0 0 #ddd, 0 8px 0 2px #333, 0 0 0 2px #333;
}

.card-item:hover {
  transform: scale3d(1.1, 1.1, 1.1);
  z-index: 1;
}

.item-cover {
  pointer-events: none;
  box-shadow: 0px 3px 0 0 #999, 0 8px 0 0 #666, 0 8px 0 2px #000, 0 0 0 2px #000;
}

.item-cover:after {
  border-radius: 2px;
  content: "";
  position: absolute;
  width: 100%;
  height: 100%;
  left: 0;
  top: 0;
  background: #000;
  opacity: 0.55;
}

.card-tips {
  white-space: nowrap;
  position: absolute;
  left: 50%;
  top: 130%;
  transform: translate(-50%, 0);
  pointer-events: none;
}

.tools {
  position: absolute;
  top: 200%;
  width: 100%;
  left: 0;
  text-align: center;
}

.clear-item {
  pointer-events: none;
}
</style>