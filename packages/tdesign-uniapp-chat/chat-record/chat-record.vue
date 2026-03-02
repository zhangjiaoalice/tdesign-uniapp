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
          v-if="processStatus === 'recording' || processStatus === 'confirm'"
          :class="[classPrefix + '-audio-input-ani', 'fade-in']"
        >
          <!-- 气泡内容 -->
          <view
            class="bubble-container"
            :class="[bubbleStatusClass]"
          >
            <!-- 1. 录音中：音量条 (只在正常状态显示) -->
            <view
              v-if="processStatus === 'recording' && interactStatus === 'normal'"
              class="audio-wave"
            >
              <view
                v-for="i in 13"
                :key="i"
                class="wave-item"
              />
            </view>
            <!-- 2. 取消状态：红色气泡，省略号 -->
            <view
              v-else-if="interactStatus === 'release_cancel'"
              class="cancel-icon"
            />
            <!-- 3. 转文字状态或确认状态：可编辑输入框 -->
            <view
              v-else-if="interactStatus === 'release_convert' || processStatus === 'confirm'"
              class="convert-content"
            >
              <textarea
                v-model="translateResult"
                class="editable-textarea"
                :maxlength="-1"
              />
              <!-- 省略号仅在未释放时显示 -->
              <view
                v-if="interactStatus === 'release_convert'"
                class="convert-dots"
              />
            </view>
          </view>
        </view>

        <!-- 底部区域 -->
        <view
          v-if="processStatus === 'recording' || processStatus === 'confirm'"
          :class="[classPrefix + '-audio-input__ft', processStatus, interactStatus, 'fade-in']"
        >
          <!-- 状态提示文案 -->
          <view class="tips-text">
            <template v-if="processStatus === 'recording'">
              <text v-if="interactStatus !== 'normal'">
                Audio
              </text>
              <text v-else>
                Release to send
              </text>
            </template>
            <template v-else-if="processStatus === 'processing'">
              Processing...
            </template>
            <!-- confirm 状态不显示提示文案 -->
          </view>

          <!-- 状态4：松手后的确认按钮区域 (Send / Cancel) -->
          <view
            v-if="processStatus === 'confirm'"
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
                Cancel
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
              <text>Send</text>
            </view>
          </view>

          <!-- 底部大圆背景和异形按钮 (仅在录音/处理阶段显示) -->
          <template v-else>
            <!-- 大圆背景 -->
            <view :class="[classPrefix + '-audio-input__ft__bg']" />

            <!-- 左侧按钮 -->
            <view
              class="shape-btn left-btn"
              :class="{ active: interactStatus === 'release_cancel' }"
            >
              <view
                v-if="interactStatus === 'release_cancel'"
                :key="'cancel-' + (interactStatus === 'release_cancel')"
                class="btn-hint"
              >
                <text class="word word-1">
                  Release
                </text>
                <text class="word word-2">
                  to cancel
                </text>
              </view>

              <view class="btn-label">
                <text class="word">
                  Cancel
                </text>
              </view>
            </view>

            <!-- 右侧按钮 -->
            <view
              class="shape-btn right-btn"
              :class="{ active: interactStatus === 'release_convert' }"
            >
              <view
                v-if="interactStatus === 'release_convert'"
                :key="'convert-' + (interactStatus === 'release_convert')"
                class="btn-hint"
              >
                <text class="word word-1">
                  Release
                </text>
                <text class="word word-2">
                  to edit
                </text>
                <text class="word word-3">
                  text
                </text>
              </view>

              <view class="btn-label">
                <text class="word word-1">
                  Convert
                </text>
                <text class="word word-2">
                  to Text
                </text>
              </view>
            </view>
          </template>
        </view>
      </view>
    </view>
  </view>
</template>

<script>
import { prefix } from 'tdesign-uniapp/common/config';
import { uniComponent } from 'tdesign-uniapp/common/src/index';
import tIcon from 'tdesign-uniapp/icon/icon.vue';

const name = `${prefix}-chat-record`;

let startRecordTimer = null; // 语音录制定时器-模拟长按
let recordTimer = null;

// 录音管理器实例
// eslint-disable-next-line no-undef
const plugin = requirePlugin('WechatSI');
const manager = plugin.getRecordRecognitionManager();

console.error('managerxxxxxx================', manager);

// 交互阈值配置
const MOVE_THRESHOLD_Y = 60; // 垂直阈值 px（约 120rpx），滑到 shape-btn 附近
const MOVE_THRESHOLD_X = 30; // 水平阈值 px（约 60rpx）

export default uniComponent({
  name: 'ChatRecord',

  components: {
    tIcon,
  },

  props: {
    // 是否自动发送（用于扩展）
    autoSend: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      classPrefix: name,

      // UI 状态
      showMask: false,
      activeBtnCancel: false,
      activeBtnSend: false,

      // 流程状态
      processStatus: 'idle', // idle | recording | processing | confirm | error
      interactStatus: 'normal', // normal | release_cancel | release_convert

      // 录音数据
      voiceInfo: {
        voicePath: '',
        voiceText: '',
        duration: 0,
      },

      // 转文字结果
      translateResult: '',

      // 手势追踪
      startTouch: { x: 0, y: 0 },

      // 权限状态
      recordAuthStatus: true,

      isStarted: false, // 是否开始录音
      startTime: 0, // 开始录音时间戳
      recordCountDown: -1, // 倒计时秒数

      // 打字机效果
      typeWriterTimer: null, // 打字机定时器
      isTyping: false, // 是否正在打字
    };
  },

  computed: {
    // 气泡样式类名
    bubbleStatusClass() {
      if (this.interactStatus === 'release_cancel') return 'bubble-red';
      if (
        this.interactStatus === 'release_convert'
        || this.processStatus === 'processing'
        || this.processStatus === 'confirm'
      ) {
        return 'bubble-wide';
      }
      return 'bubble-blue';
    },
  },

  watch: {
    /**
     * 监听交互状态变化
     * 当滑到转文字区域时立即开始打字机效果
     */
    interactStatus(newVal) {
      if (newVal === 'release_convert' && this.voiceInfo.voiceText) {
        console.log('[ChatRecord] 进入转文字区域，开始打字机效果');
        this.translateResult = '';
        this.typeWriter(this.voiceInfo.voiceText, 30);
      }
    },
  },

  mounted() {
    this.initRecordManager();
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
    // ==================== 初始化 ====================

    /**
     * 初始化录音管理器
     */
    initRecordManager() {
      if (!manager) return;

      manager.onStart = () => {
        console.log('onStart 开始录音=====');
        this.processStatus = 'recording';
      };


      manager.onRecognize = (res) => {
        console.error('onRecognize 识别中=====:', res.result);
        if (res.result && !res.end) {
          this.voiceInfo.voiceText = res.result;
          if (this.interactStatus === 'release_convert') {
            this.typeWriter(this.voiceInfo.voiceText, 30);
          }
        }
      };

      manager.onStop = (res) => {
        console.error('onStop 录音完成====:', res);
        const { tempFilePath, duration } = res;
        this.voiceInfo.voicePath = tempFilePath;
        this.voiceInfo.duration = Math.floor(duration / 1000) || 1;

        // TODO: 调用语音识别 API
        this.handleRecordFinish();
      };

      manager.onError = (err) => {
        console.error('[ChatRecord] 录音错误:', err);
        this.processStatus = 'error';
        this.interactStatus = 'normal';
        this.translateResult = '';
        // 给用户友好的错误提示
        uni.showToast({
          icon: 'none',
          title: err.msg || '录音识别失败，请重试',
          duration: 2000,
        });
        if (manager) {
          manager.stop();
        }
        this.$emit('error', err);
      };
    },

    // ==================== 权限管理 ====================

    /**
     * 检查录音权限
     */
    async getVoiceAuthSetting() {
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

    /**
     * 申请录音权限
     */
    async applyAuth() {
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

    /**
     * 打开系统设置页面
     */
    openVoiceSetting() {
      // TODO: 引导用户打开设置
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

    /**
     * 开始录音
     */
    async startRecord(e) {
      console.log('startRecord', e);
      // 阻止默认行为
      if (e && e.preventDefault) {
        e.preventDefault();
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

      // 记录起始触摸点
      if (e && e.changedTouches && e.changedTouches[0]) {
        const { clientX, clientY } = e.changedTouches[0];
        this.startTouch = { x: clientX, y: clientY };
      }


      this.showMask = true;
      this.processStatus = 'recording';
      // this.interactStatus = 'normal';
      // this.translateResult = '';

      // 触发开始事件
      // this.$emit('start');

      console.log('[ChatRecord] 模拟录音开始');
      // 500ms 后开始录音
      startRecordTimer = setTimeout(() => {
        if (!this.isStarted) {
          console.log('录音已取消');
          return;
        }
        // 确保录音管理器已初始化
        if (!manager) {
          this.initRecordManager();
        }
        if (manager) {
          manager.start({ duration: 60000, lang: 'zh_CN' });
        }

        // 最多支持 60s 连续录音，50s 开始倒计时
      //   recordTimer = setInterval(() => {
      //     const recordTime = new Date().getTime() - this.startTime;
      //     if (recordTime > 50000) {
      //       if (this.recordCountDown === -1) {
      //         this.recordCountDown = 10;
      //       } else {
      //         this.recordCountDown -= 1;
      //       }
      //     }
      //     if (recordTime > 60000) {
      //       console.log('录音超时，自动停止');
      //       this.stopRecord();
      //     }
      //   }, 1000);
      }, 500);
    },

    /**
     * 手势移动处理
     */
    touchmove(e) {
      // 只有在录音开始后才处理滑动
      if (!this.isStarted || !this.showMask) {
        return;
      }

      if (this.processStatus !== 'recording') return;

      // 优先使用 touches，在 touchmove 中更可靠
      const touch = e.touches && e.touches[0] ? e.touches[0] : e.changedTouches[0];
      if (!touch) return;

      const { clientX, clientY } = touch;
      const deltaX = clientX - this.startTouch.x;
      const deltaY = clientY - this.startTouch.y;

      // 判断手势方向
      if (deltaY < -MOVE_THRESHOLD_Y) {
        // 向上滑动超过阈值（到达 shape-btn 附近）
        if (deltaX < -MOVE_THRESHOLD_X) {
          this.interactStatus = 'release_cancel'; // 左上：取消
        } else if (deltaX > MOVE_THRESHOLD_X) {
          this.interactStatus = 'release_convert'; // 右上：转文字
        } else {
          this.interactStatus = 'normal'; // 正上：无操作
        }
      } else {
        this.interactStatus = 'normal';
      }
    },

    /**
     * 停止录音
     */
    stopRecord() {
      if (this.processStatus !== 'recording') return;

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

      if (this.interactStatus === 'release_cancel') {
        // 取消录音
        this.cancelRecord();
        return;
      } if (this.interactStatus === 'release_convert') {
        // 转换为文字
        this.convertToText();
        return;
      }
      // 直接发送语音
      this.sendVoice();
    },

    /**
     * 触摸取消
     */
    touchcancel() {
      this.cancelRecord();
    },

    // ==================== 业务逻辑 ====================

    /**
     * 取消录音
     */
    cancelRecord() {
      console.log('[ChatRecord] 取消录音');
      // if (manager) manager.stop();
      this.$emit('cancel');
      this.resetState();
    },

    /**
     * 转换为文字
     */
    convertToText() {
      console.log('[ChatRecord] 转换为文字');
      this.processStatus = 'confirm';
      this.interactStatus = 'normal';

      // 打字机效果已在 watch.interactStatus 中处理
      // 如果滑到转文字区域时已显示部分内容，继续完成剩余部分
      if (!this.isTyping && this.voiceInfo.voiceText) {
        const remainingText = this.voiceInfo.voiceText.slice(this.translateResult.length);
        if (remainingText) {
          this.typeWriter(remainingText, 30);
        }
      }
    },

    /**
     * 直接发送语音
     */
    sendVoice() {
      console.log('[ChatRecord] 发送语音');
      this.$emit('send', {
        voicePath: this.voiceInfo.voicePath,
        voiceText: this.voiceInfo.voiceText,
        duration: this.voiceInfo.duration,
      });
      this.resetState();
    },

    /**
     * 录音完成处理
     */
    handleRecordFinish() {
      // 模拟识别延迟
      setTimeout(() => {
        this.processStatus = 'confirm';
      }, 500);
    },

    /**
     * 发送转文字结果
     */
    handleSendVoiceMsg() {
      this.$emit('recognize', this.translateResult);
      this.resetState();
    },

    /**
     * 取消发送
     */
    handleCancelSend() {
      this.$emit('cancel');
      this.resetState();
    },

    // ==================== 状态管理 ====================

    /**
     * 重置所有状态
     */
    resetState() {
      this.showMask = false;
      this.processStatus = 'idle';
      this.interactStatus = 'normal';
      this.translateResult = '';
      this.voiceInfo = { voicePath: '', duration: 0 };
      this.activeBtnCancel = false;
      this.activeBtnSend = false;

      // 清除打字机定时器
      if (this.typeWriterTimer) {
        clearTimeout(this.typeWriterTimer);
        this.typeWriterTimer = null;
      }
      this.isTyping = false;
    },

    // ==================== 打字机效果 ====================

    /**
     * 打字机效果 - 逐字显示文本
     * @param {string} text - 要显示的完整文本
     * @param {number} speed - 打字速度（毫秒/字），默认 50ms
     * @returns {Promise} - 打字完成后 resolve
     */
    typeWriter(text, speed = 50) {
      return new Promise((resolve) => {
        // 如果有正在进行的打字机效果，先清除
        if (this.typeWriterTimer) {
          clearTimeout(this.typeWriterTimer);
          this.typeWriterTimer = null;
        }

        this.isTyping = true;
        let index = 0;
        this.translateResult = '';

        const typeChar = () => {
          if (index < text.length) {
            this.translateResult += text.charAt(index);
            index += 1;
            this.typeWriterTimer = setTimeout(typeChar, speed);
          } else {
            this.isTyping = false;
            this.typeWriterTimer = null;
            resolve();
          }
        };

        typeChar();
      });
    },

    /**
     * 停止打字机效果
     */
    stopTypeWriter() {
      if (this.typeWriterTimer) {
        clearTimeout(this.typeWriterTimer);
        this.typeWriterTimer = null;
      }
      this.isTyping = false;
    },
  },
});
</script>

<style scoped>
@import './chat-record.css';
</style>
