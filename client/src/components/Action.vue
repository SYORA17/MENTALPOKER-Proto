<template>
  <div class="action-container">
    <div class="action"
         v-show="isAction">
      <div class="action-type action-btn">
        <span @click="action('fold')">fold</span>
        <span @click="action('check')"
                v-show="showActionBtn('check')">check</span>
        <span @click="action('call')"
              v-show="showActionBtn('call')">call</span>
        <span @click="otherSizeHandle()"
              v-show="showActionBtn('raise')">more</span>
        <span @click="action('allin')"
              v-show="!showActionBtn('raise')">allin</span>
      </div>
      <div class="raise-size">
        <div class="not-allin"
             v-show="showActionBtn('raise')">
          <i v-for="size in raiseSizeMap"
             @click="raise(size)"
             v-show="showActionSize(size)"
          > {{Math.floor(size)}}</i>
        </div>
      </div>
      <div class="action-other-size"
           v-if="isRaise">
        <div class="action-other-size-body">
          <div class="size"
               v-show="currPlayer && raiseSize < currPlayer.counter">
            <input type="number" v-model="raiseSize">
          </div>
          <div class="size"
               v-show="currPlayer && raiseSize === currPlayer.counter">Allin
          </div>
          <range :max="currPlayer && currPlayer.counter"
                 :min="0"
                 :is-horizontal="true"
                 v-model="raiseSize"
                 @change="getActionSize"></range>
          <div class="btn"
               @click="addSize">ok
          </div>
        </div>
        <div class="shadow"
             @click="isRaise = false"></div>
      </div>
      <div>
        <iAudio :play="playClick && audioStatus" type="click"></iAudio>
        <iAudio :play="playFold && audioStatus" type="fold"></iAudio>
        <iAudio :play="playRaise && audioStatus" type="raise"></iAudio>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import range from './Range.vue';
import iAudio from './Audio.vue';
import { IPlayer } from '@/interface/IPlayer';

@Component({
  components: {
    range,
    iAudio,
  },
})export default class Action extends Vue {
  @Prop() private isAction: boolean = false;
  @Prop() private minActionSize!: number;
  @Prop() private pot!: number;
  @Prop() private prevSize!: number;
  @Prop() private baseSize!: number;
  @Prop() private isPreFlop!: boolean;
  @Prop() private isTwoPlayer!: boolean;
  @Prop() private currPlayer!: IPlayer;
  @Prop() private audioStatus?: boolean;

  private isRaise = false;
  private raiseSize: number = 0;
  private actioned = false;
  private playClick = false;
  private playRaise = false;
  private playFold = false;
  // private raiseSizeMap = {
  //   firsAction: {
  //     two: 2,
  //     three: 3,
  //     four: 4,
  //   },
  //   other: {
  //     oneThirdPot: 0.5,
  //     halfPot: 0.75,
  //     pot: 1,
  //   },
  // };

  @Watch('isAction')
  private wAction(val: boolean) {
    this.actioned = !val;
    this.playClick = false;
    this.playRaise = false;
    this.playFold = false;
  }

  @Watch('raiseSize')
  private wRaiseSize(val: number) {
    this.raiseSize = val > this.currPlayer.counter ? this.currPlayer.counter : val;
  }

  get raiseSizeMap() {
    let size = this.pot > this.baseSize * 4 ? this.pot : this.baseSize * 2;
    if (this.prevSize > 1) {
      size = this.prevSize * 4;
    }
    return size === this.baseSize * 2 ? [ size, 2 * size, 3 * size ] : [ 0.5 * size, 0.75 * size, size ];
  }

  get canActionSize() {
    return Number(this.currPlayer && this.currPlayer.counter + this.currPlayer.actionSize);
  }

  private raise(size: number) {
    this.action(`raise:${Math.floor(size)}`);
  }

  private action(command: string) {
    if (command.indexOf('raise') > -1 || command === 'allin' || command === 'call' ) {
      this.playRaise = true;
    }
    if (command === 'fold' || command === 'check') {
      this.playFold = true;
    }
    if (!this.actioned) {
      this.actioned = true;
      this.$emit('action', command);
      this.isRaise = false;
      this.actioned = false;
    }
  }

  private showActionSize(multiple: number) {
    return this.currPlayer
      && this.currPlayer.counter > Math.floor(multiple)
      && this.prevSize * 2 <= Math.floor(multiple * this.pot)
      && this.baseSize * 2 <= Math.floor(multiple * this.pot);
  }

  private otherSizeHandle() {
    this.isRaise = true;
    this.raiseSize = this.minActionSize;
  }

  private getActionSize(size: number) {
    if (size > this.minActionSize) {
      this.raiseSize = size;
    } else {
      this.$plugin.toast('raise size too small');
    }
  }

  private addSize() {
    if (this.raiseSize === this.currPlayer?.counter) {
      this.action('allin');
    } else {
      this.action(`raise:${this.raiseSize}`);
    }
  }

  private showActionBtn(type: string) {
    // check
    if ('check' === type) {
      return this.prevSize <= 0
        || (this.isPreFlop
          && this.isTwoPlayer
          && this.currPlayer?.type === 'd'
          && this.prevSize === this.baseSize * 2)
        || (this.currPlayer?.type === 'bb' && this.prevSize === this.baseSize * 2 &&
          this.isPreFlop);
    }
    // raise
    if ('raise' === type) {
      return this.canActionSize > this.prevSize * 2;
    }
    // call
    if ('call' === type) {
      return this.canActionSize > this.prevSize
        && this.prevSize > 0
        && !((this.isPreFlop
          && this.isTwoPlayer
          && this.currPlayer?.type === 'd'
          && this.prevSize === 2 * this.baseSize)
          || (this.currPlayer?.type === 'bb' && this.prevSize === 2 * this.baseSize &&
            this.isPreFlop));
    }
    return true;
  }

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="less">
  .action-container {
    .action {
      position: absolute;
      color: #fff;
      width: 80vw;
      top: 65vh;
      left: 50%;
      transform: translateX(-50%);

      .raise-size {
        position: absolute;
        top: -7vh;
        left: 50%;
        width: 53vw;
        margin-left: -26.4vw;
        text-align: center;

        i {
          padding: 2px;
          width: 24px;
          height: 24px;
          display: inline-block;
          font-style: normal;
          font-size: 10px;
          line-height: 24px;
          border-radius: 50%;
          color: #fff;
          border: 1px solid #fff;
          background: rgba(0, 0, 0, .2);
          margin: 10px;
          vertical-align: middle;
        }
      }

      .action-btn {
        span {
          border-radius: 50%;
          width: 40px;
          height: 40px;
          padding: 2px;
          text-align: center;
          margin: 0 10px;
          line-height: 40px;
          border: 1px solid #fff;
          font-size: 14px;
          display: inline-block;
        }
      }

      .action-other-size {
        background-color: rgba(0, 0, 0, 0);
        position: fixed;
        width: 50vw;
        height: 30vh;
        right: -16px;
        top: -35vh;
        z-index: 90;

        .shadow {
          position: absolute;
          top: -30vh;
          width: 99vw;
          height: 100vh;
          right: -5vw;
          z-index: 8;
          overflow: hidden;
          background: linear-gradient(-70deg, black, transparent);
        }

        .action-other-size-body {
          z-index: 9;
          position: absolute;
          width: 50vw;
          height: 30vh;
          .size{
            input{
              background: transparent;
              font-size: 20px;
              width: 50px;
              text-align: center;
              color: #fff;
            }
          }
          .btn {
            position: absolute;
            top: 34vh;
            left: 20vw;
            border: 1px solid #fff;
            border-radius: 50%;
            background-color: rgba(0, 0, 0, 0.4);
            padding: 5px;
            font-size: 12px;
            width: 20px;
            height: 20px;
            line-height: 20px;
          }
        }
      }
    }
  }
</style>
