<template>
  <view :class="[classPrefix]">
    <view
      v-if="recordAuthStatus"
      :class="[classPrefix + '-hook']"
      @touchstart="startRecord"
      @touchend="stopRecord"
      @touchmove="touchmove"
      @touchcancel="touchcancel"
    >
      <slot
        v-if="$slots.speechInput"
        name="speechInput"
      />
      <view v-else>
        按住 说话
      </view>
    </view>
    <view
      v-else
      :class="[classPrefix + '-hook']"
      @click="openVoiceSetting"
    >
      <slot
        v-if="$slots.speechNoAuth"
        name="speechNoAuth"
      />
      <view v-else>
        请授权麦克风权限
      </view>
    </view>
    <view
      class="cover-ng-bar"
      :class="[classPrefix + '-audio-input', showMask ? 'show' : '']"
    >
      <!-- 遮罩层 -->
      <view
        :class="[classPrefix + '-audio-input__mask']"
        @click="handleCancelSend"
      />

      <view :class="[classPrefix + '-audio-input__main']">
        <!-- 动画图标/气泡区域 -->
        <view
          v-if="showMask"
          :class="[classPrefix + '-audio-input-ani', 'fade-in']"
        >
          <!-- 气泡内容 -->
          <view
            class="bubble-container"
            :class="[bubbleStatusClass]"
          >
            <!-- 1. 录音中正常状态：显示实时识别文字+音量条 -->
            <view
              v-if="status === 'recording' && touchStatus !== 'release_cancel'"
              class="convert-content"
            >
              <text class="convert-text">
                {{ translateResult || '' }}
              </text>
              <!-- 省略号表示正在识别 -->
              <view
                v-if="translateResult"
                class="convert-dots"
              />
              <!-- 音量条 -->
              <view class="audio-wave-mini">
                <view
                  v-for="i in 5"
                  :key="i"
                  class="wave-item"
                />
              </view>
            </view>
            <!-- 2. 录音中取消/异常/未识别状态：红色气泡，省略号 -->
            <view
              v-else-if="status === 'error' || status === 'unknow'"
              class="cancel-icon"
            />
            <!-- 3. 录音结束：显示识别结果（可编辑） -->
            <view
              v-else-if="status === 'stop' || status === 'complete'"
              class="convert-content"
            >
              <textarea
                v-model="translateResult"
                class="editable-textarea"
                :maxlength="-1"
                placeholder="语音识别中..."
              />
            </view>
          </view>
        </view>

        <!-- 底部区域 -->
        <view
          v-if="showMask"
          :class="[classPrefix + '-audio-input__ft', touchStatus, 'fade-in']"
        >
          <!-- 状态4：录音结束后的确认按钮区域 (Send / Cancel) -->
          <view
            v-if="status === 'stop' || status === 'complete'"
            class="confirm-actions"
          >
            <view
              class="action-btn btn-cancel"
              :class="{ active: activeBtnCancel }"
              @click="handleCancelSend"
              @touchstart="activeBtnCancel = true"
              @touchend="activeBtnCancel = false"
              @touchcancel="activeBtnCancel = false"
            >
              <view class="icon-wrapper">
                <t-icon
                  name="rollback"
                  size="48rpx"
                  color="#FFFFFF"
                />
              </view>
              <text class="btn-text">
                取消
              </text>
            </view>
            <view
              class="action-btn btn-send"
              :class="{ active: activeBtnSend }"
              @click="handleSendVoiceMsg"
              @touchstart="activeBtnSend = true"
              @touchend="activeBtnSend = false"
              @touchcancel="activeBtnSend = false"
            >
              <text>发送</text>
            </view>
          </view>

          <!-- 录音中状态提示文案 -->
          <view
            v-else-if="status === 'recording'"
            class="tips-text"
          >
            <text class="text">
              松手完成，上滑取消
            </text>
          </view>

          <!-- 录音异常/未识别状态提示文案 -->
          <view
            v-else-if="status === 'error' || status === 'unknow'"
            class="tips-text"
            @touchstart="restartRecord"
            @touchend="stopRecord"
            @touchcancel="handleCancelSend"
          >
            <text class="text">
              按住重新说话
            </text>
          </view>

          <!-- 录音中：大圆背景和取消按钮 -->
          <!-- 大圆背景 -->
          <view :class="[classPrefix + '-audio-input__ft__bg']" />

          <!-- 取消按钮 -->
          <view
            v-if="status === 'stop' || status === 'error' || status === 'unknow'"
            class="shape-btn left-btn"
            :class="{ active: status === 'stop' || status === 'error' || status === 'unknow' }"
            @click="handleCancelSend"
          >
            <view class="btn-label">
              <text class="word">
                取消
              </text>
            </view>
          </view>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
import { prefix } from 'tdesign-uniapp/common/config';
import { uniComponent } from 'tdesign-uniapp/common/src/index';

const name = `${prefix}-chat-record`;

let startRecordTimer = null; // 语音录制定时器-模拟长按
let recordTimer = null;

// eslint-disable-next-line no-undef
const plugin = requirePlugin('WechatSI');
const manager = plugin.getRecordRecognitionManager();
console.error('manager============:', manager);

export default uniComponent({
  name: 'ChatRecord',
  components: {},
  props: {},
  data() {
    return {
      classPrefix: name,
      showMask: false, // 是否展示语音输入操作面板
      touchStatus: 'bottom', // 语音输入 - 当前手指状态 top：取消， bottom：正常
      startTime: 0, // 开始录音时间
      recordCountDown: -1, // 语音输入倒计时
      translateResult: '', // 语音转文字结果
      voiceInfo: {
        voicePath: '',
        duration: 0,
      }, // 语音消息信息（voicePath：语音文件本地路径；mediaTime：语音消息时长）
      recordStatus: '', // 录音状态 start 开始录音， stop 结束录音
      recordAuthSetting: false, // 是否已授权语音输入
      recordAuthStatus: true, // 是否展示拒绝授权文案
      isStarted: false, // 是否点击了 // 解决点击结束太快，授权信息还没拿到的情况
      isRecognizing: false, // 是否正在识别中
      isWaitingForStop: false, // 是否正在等待识别停止
      restartRetryCount: 0, // 重新说话重试次数
      doStartRetryCount: 0, // doStartRecord 重试次数
      activeBtnCancel: false, // 取消按钮激活状态
      activeBtnSend: false, // 发送按钮激活状态
      bottomHeight: 0, // 底部高度
      autoSendHeight: true, // 是否自动抬升发送按钮高度
      windowHeight: 0, // 窗口高度，用于手指滑动判断
    };
  },
  computed: {
    translateSuccess() {
      return this.translateResult?.length;
    },
    showRecordCountDown() {
      return this.recordCountDown >= 0;
    },
    status() {
      // 录音完成状态
      if (this.recordStatus === 'stop' && !this.translateSuccess) return 'unknow';
      if (this.recordStatus === 'stop' && this.translateSuccess) return 'complete';
      if (this.recordStatus === 'error') return 'error';
      return this.recordStatus || 'normal';
    },
    bubbleStatusClass() {
      // 取消状态：红色气泡
      if (this.touchStatus === 'release_cancel') return 'bubble-red';
      // 停止/完成状态：宽气泡
      if (this.status === 'stop' || this.status === 'complete') {
        return 'bubble-wide';
      }
      // 默认：蓝色气泡
      return 'bubble-blue';
    },
  },
  mounted() {
    // 初始化录音管理器
    this.initRecorderManager();
    // 获取窗口高度
    this.getWindowHeight();
  },
  beforeDestroy() {
    if (recordTimer) {
      clearInterval(recordTimer);
      recordTimer = null;
    }
    if (startRecordTimer) {
      clearTimeout(startRecordTimer);
      startRecordTimer = null;
    }
    if (manager) {
      manager.stop();
    }
  },
  methods: {
    /**
     * @description 初始化同声传译插件
     */
    initRecorderManager() {
      // 使用箭头函数确保 this 指向组件实例
      manager.onStop = (res) => {
        console.log('onStop record file path', res);
        console.log('result', res.result);
        const { tempFilePath, duration, result } = res;

        // 标记识别已完成
        this.isRecognizing = false;
        this.isWaitingForStop = false;

        // 如果是取消状态，不保存录音信息
        if (this.touchStatus === 'release_cancel') {
          console.log('用户取消发送，不保存录音');
          this.isStarted = false;
          this.resetRecordState();
          return;
        }

        this.voiceInfo.voicePath = tempFilePath;
        this.voiceInfo.duration = Math.floor(duration / 1000) || 1;

        // 重置录音开始状态，允许再次点击
        this.isStarted = false;

        // 如果有识别结果，保存并进入完成状态
        if (result && result !== '-1') {
          this.translateResult = result;
          this.recordStatus = 'stop';
        } else {
          this.recordStatus = 'stop';
        }
      };

      manager.onStart = (res) => {
        console.log('onStart 成功开始录音识别', res);
        this.recordStatus = 'recording';
        this.isRecognizing = true;
      };

      manager.onRecognize = (res) => {
        console.log('onRecognize 识别中:', res.result);
        if (res.result && !res.end) {
          this.translateResult = res.result;
        }
      };

      manager.onError = (res) => {
        console.error('录音错误:', res.msg);
        this.recordStatus = 'error';
        this.touchStatus = 'bottom';
        this.translateResult = '';

        // 标记识别已完成（出错了）
        this.isRecognizing = false;
        this.isWaitingForStop = false;
        // 重置录音开始状态，允许再次点击
        this.isStarted = false;

        // 给用户友好的错误提示
        uni.showToast({
          icon: 'none',
          title: res.msg || '录音识别失败，请重试',
          duration: 2000,
        });
      };
    },
    /**
     * @description 重置录音状态
     */
    resetRecordState() {
      console.log('resetRecordState 被调用，当前状态:', {
        showMask: this.showMask,
        recordStatus: this.recordStatus,
        isStarted: this.isStarted,
        isWaitingForStop: this.isWaitingForStop,
      });
      this.showMask = false;
      this.translateResult = '';
      this.recordStatus = '';
      this.touchStatus = '';
      this.recordCountDown = -1;
      this.startTime = 0;
      this.isStarted = false;
      this.isRecognizing = false;
      this.restartRetryCount = 0;
      this.doStartRetryCount = 0;
      // 注意：isWaitingForStop 不在此处重置，只应在 onStop/onError 回调中重置
      // 清除可能存在的延迟录音定时器
      if (startRecordTimer) {
        clearTimeout(startRecordTimer);
        startRecordTimer = null;
      }
      console.log('resetRecordState 执行完成，showMask:', this.showMask);
    },
    /**
     * @description 获取窗口高度
     */
    getWindowHeight() {
      uni.getSystemInfo({
        success: (res) => {
          this.windowHeight = res.windowHeight;
        },
      });
    },
    openVoiceSetting() {
      uni.showModal({
        title: '提示',
        content: '即将跳转到设置页',
        success: (res) => {
          if (res.confirm) {
            uni.openSetting({
              success: (res) => {
                this.recordAuthSetting = !!res.authSetting['scope.record'];
                this.recordAuthStatus = !!res.authSetting['scope.record'];
                this.$nextTick(() => {
                  this.$forceUpdate();
                });
              },
            });
          }
        },
      });
    },
    getVoiceAuthSetting() {
      return new Promise((resolve, reject) => {
        uni.getSetting({
          success: (res) => {
            const authSettings = Object.keys(res.authSetting);
            // 是否已经授权了
            this.recordAuthSetting = authSettings.includes('scope.record');
            // 当前授权状态
            this.recordAuthStatus = !!res.authSetting['scope.record'];
            resolve(this.recordAuthSetting);
          },
          fail: () => {
            reject(false);
          },
        });
      });
    },
    applyAuth() {
      return new Promise((resolve, reject) => {
        uni.authorize({
          scope: 'scope.record',
          success: () => {
            this.recordAuthSetting = true;
            this.recordAuthStatus = true;
            resolve(true);
          },
          fail: () => {
            this.recordAuthSetting = false;
            this.recordAuthStatus = false;
            reject(false);
          },
        });
      });
    },
    /**
     * @description 修改语音转文字结果
     * @params value 输入框文本值
     */
    onTranslateResultChange(value) {
      this.translateResult = value;
    },
    /**
     * @description 直接发送语音消息
     */
    handleSendVoiceMsg() {
      if (this.translateResult?.length) {
        this.sendVoiceMsg(this.translateResult);
      }
      this.resetRecordState();
    },
    /**
     * @description 取消发送语音
     */
    handleCancelSend() {
      console.log('handleCancelSend 被调用，当前 showMask:', this.showMask);
      this.resetRecordState();
      console.log('handleCancelSend 执行后，showMask:', this.showMask);
    },
    /**
     * @description 开始录音
     */
    async startRecord(e) {
      // 防止重复触发
      if (this.isStarted) {
        console.log('已经在录音中，忽略');
        return;
      }

      this.isStarted = true;

      // 检查授权
      try {
        await this.getVoiceAuthSetting();
        if (!this.recordAuthSetting) {
          await this.applyAuth();
          this.isStarted = false;
          return;
        }
      } catch (error) {
        console.error('授权检查失败', error);
        this.isStarted = false;
        return;
      }

      // 阻止默认行为
      if (e && e.preventDefault) {
        e.preventDefault();
      }

      // 记录起始触摸点，用于手势判断
      if (e && e.changedTouches && e.changedTouches[0]) {
        this.startTouch = {
          x: e.changedTouches[0].clientX,
          y: e.changedTouches[0].clientY,
        };
      }

      this.touchStatus = 'bottom';
      // 记录开始录音时间
      this.startTime = new Date().getTime();
      // 立即显示 mask，让用户可以上滑取消
      this.showMask = true;

      // 500ms后开始录音，模拟长按效果，避免误操作
      startRecordTimer = setTimeout(() => {
        if (!this.isStarted) {
          console.log('录音已取消');
          return;
        }

        // 确保录音管理器已初始化
        if (!manager) {
          this.initRecorderManager();
        }

        manager.start({ duration: 30000, lang: 'zh_CN' });

        console.log('开始录音---');
        this.showMask = true;
        this.recordStatus = 'recording';

        // 最大支持60s连续录音，50s时开始倒计时
        recordTimer = setInterval(() => {
          const recordTime = new Date().getTime() - this.startTime;
          if (recordTime > 50000) {
            if (this.recordCountDown === -1) {
              this.recordCountDown = 10;
            } else {
              this.recordCountDown -= 1;
            }
          }
          if (recordTime > 60000) {
            console.log('录音超时，自动停止');
            this.stopRecord();
          }
        }, 1000);
      }, 500);
    },
    /**
     * @description 重新说话（点击立即开始，无需长按）
     */
    restartRecord() {
      console.log('restartRecord 被调用，当前 isStarted:', this.isStarted, 'isWaitingForStop:', this.isWaitingForStop, '重试次数:', this.restartRetryCount, '当前状态:', this.status);

      // 防止重复触发
      if (this.isStarted) {
        console.log('已经在录音中，忽略');
        return;
      }

      // 如果正在等待上一次的识别停止，延迟重试
      if (this.isWaitingForStop) {
        this.restartRetryCount += 1;
        if (this.restartRetryCount > 50) {
          console.log('重试次数过多，放弃');
          this.restartRetryCount = 0;
          this.isWaitingForStop = false;
          return;
        }
        console.log('等待上一次识别停止，延迟重试... 次数:', this.restartRetryCount);
        setTimeout(() => {
          this.restartRecord();
        }, 100);
        return;
      }

      // 重置重试计数
      this.restartRetryCount = 0;
      this.isStarted = true;
      console.log('设置 isStarted = true');

      // 检查授权（同步方式）
      this.getVoiceAuthSetting().then(() => {
        console.log('授权状态:', this.recordAuthSetting);
        if (!this.recordAuthSetting) {
          console.log('未授权，申请授权');
          this.applyAuth().then(() => {
            this.isStarted = false;
          });
          return;
        }

        // 授权通过，开始录音
        this.doStartRecord();
      })
        .catch((error) => {
          console.error('授权检查失败', error);
          this.isStarted = false;
        });
    },

    /**
     * @description 实际开始录音
     */
    doStartRecord() {
      console.log('执行 doStartRecord，isRecognizing:', this.isRecognizing, 'isWaitingForStop:', this.isWaitingForStop, '重试次数:', this.doStartRetryCount);

      // 如果还在识别中或等待停止，延迟200ms再试
      if (this.isRecognizing || this.isWaitingForStop) {
        this.doStartRetryCount += 1;
        if (this.doStartRetryCount > 30) {
          console.log('doStartRecord 重试次数过多，放弃');
          this.doStartRetryCount = 0;
          this.isStarted = false;
          return;
        }
        console.log('识别中或等待停止，延迟重试... 次数:', this.doStartRetryCount);
        setTimeout(() => {
          this.doStartRecord();
        }, 200);
        return;
      }

      // 重置重试计数
      this.doStartRetryCount = 0;

      // 清除之前可能存在的录音定时器
      if (recordTimer) {
        clearInterval(recordTimer);
        recordTimer = null;
      }

      // 重置状态
      this.resetRecordState();
      this.touchStatus = 'bottom';
      this.startTime = new Date().getTime();

      // 确保录音管理器已初始化
      if (!manager) {
        console.log('初始化录音管理器');
        this.initRecorderManager();
      }

      // 立即开始录音（无延迟）
      console.log('调用 manager.start');
      try {
        manager.start({ duration: 30000, lang: 'zh_CN' });
      } catch (err) {
        console.error('manager.start 失败:', err);
        this.isRecognizing = false;
        this.isStarted = false;
        return;
      }

      console.log('重新说话，开始录音---');
      this.showMask = true;
      this.recordStatus = 'recording';

      // 最大支持60s连续录音，50s时开始倒计时
      recordTimer = setInterval(() => {
        const recordTime = new Date().getTime() - this.startTime;
        if (recordTime > 50000) {
          if (this.recordCountDown === -1) {
            this.recordCountDown = 10;
          } else {
            this.recordCountDown -= 1;
          }
        }
        if (recordTime > 60000) {
          console.log('录音超时，自动停止');
          this.stopRecord();
        }
      }, 1000);
    },
    /**
     * @description 结束录音
     */
    stopRecord() {
      console.log('结束录音触发', {
        isStarted: this.isStarted,
        startTime: this.startTime,
        touchStatus: this.touchStatus,
        isRecognizing: this.isRecognizing,
      });

      // 标记录音结束
      this.isStarted = false;

      // 清除定时器
      if (recordTimer) {
        clearInterval(recordTimer);
        recordTimer = null;
      }
      if (startRecordTimer) {
        clearTimeout(startRecordTimer);
        startRecordTimer = null;
      }

      this.recordCountDown = -1;

      // 松开的时候，判断有没有授权
      if (!this.recordAuthStatus) {
        console.log('未授权，取消录音');
        this.resetRecordState();
        return;
      }

      // 根据startTime判断是否已经自动触发停止录音接口，避免二次调用
      if (this.startTime === 0) {
        console.log('startTime为0，已停止过，直接关闭mask');
        this.resetRecordState();
        return;
      }

      const recordTime = new Date().getTime() - this.startTime;
      console.log('录音时长', recordTime);

      // 处理上滑取消
      if (this.touchStatus === 'release_cancel') {
        console.log('用户取消发送，直接关闭 mask');
        // 标记正在等待识别停止
        this.isWaitingForStop = true;
        if (manager) {
          manager.stop();
        }
        this.resetRecordState();
        uni.showToast({
          icon: 'none',
          title: '已取消发送',
          duration: 1500,
        });
        return;
      }

      // 根据当前时间判断录音时间是否大于500ms，避免录音时间过短
      if (recordTime > 500) {
        console.log('停止录音，进入确认状态');
        this.voiceInfo.duration = Math.floor(recordTime / 1000) || 1;
        this.startTime = 0;

        // 标记正在等待识别停止
        this.isWaitingForStop = true;
        if (manager) {
          manager.stop();
        }
        // 进入确认状态，显示发送/取消按钮，不自动关闭 mask
        this.recordStatus = 'stop';
      } else {
        console.log('录音时间太短');
        // 标记正在等待识别停止
        this.isWaitingForStop = true;
        this.resetRecordState();
        if (manager) {
          manager.stop();
        }
        uni.showToast({
          icon: 'none',
          title: '说话时间太短',
        });
      }
    },
    /**
     * @description 录音过程中手指移动事件
     */
    touchmove(e) {
      // 只有在录音开始后才处理滑动
      if (!this.isStarted || !this.showMask) {
        return;
      }

      // 根据手势方向判断交互状态
      const { changedTouches } = e;
      if (!changedTouches || !changedTouches[0]) {
        return;
      }

      const { clientX, clientY } = changedTouches[0];
      const deltaX = clientX - this.startTouch.x;
      const deltaY = clientY - this.startTouch.y;
      const oldStatus = this.touchStatus;

      // 判断手势方向：需要向上滑动一定距离才触发
      if (deltaY < -40) {
        // 向上滑动超过阈值
        if (deltaX < -40) {
          // 左上滑动：取消发送
          this.touchStatus = 'release_cancel';
        } else {
          // 其他方向：保持默认
          this.touchStatus = 'bottom';
        }
      } else {
        // 未触发上滑，保持默认状态
        this.touchStatus = 'bottom';
      }

      // 状态变化时打印日志
      if (oldStatus !== this.touchStatus) {
        console.log('手指位置变化', {
          deltaX,
          deltaY,
          touchStatus: this.touchStatus,
        });
      }
    },
    /**
     * @description 录音过程中被系统事件打断，结束录音
     */
    touchcancel() {
      console.log('录音被打断');

      // 标记正在等待识别停止
      this.isWaitingForStop = true;
      if (manager) {
        manager.stop();
      }

      // 注意：isRecognizing 和 isWaitingForStop 会在 onStop 或 onError 回调中重置
      this.resetRecordState();
    },
    /**
     * @description 发送语音消息，可以重写此方法
     * @param {String} voiceMsg 语音消息 String
     */
    sendVoiceMsg(voiceMsg) {
      this.$emit('recognize', voiceMsg);
    },

    focusTextarea(e) {
      if (this.autoSendHeight) {
        this.bottomHeight = e.detail.height;
      }
    },
    blurTextarea() {
      this.bottomHeight = 0;
    },
  },
});
</script>

<style scoped>
@import './chat-record.css';
</style>
