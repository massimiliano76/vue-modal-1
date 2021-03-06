<template>
  <div v-if="mount">
    <portal :selector="appendTo">
      <transition
        name="vm-backdrop-transition"
        :enter-active-class="bgInClass"
        :leave-active-class="bgOutClass"
      >
        <div
          v-show="show"
          :class="['vm-backdrop', id+'-backdrop', bgClass]"
          :style="{ 'z-index': zIndex-1 }"
        ></div>
      </transition>
      <transition
        name="vm-transition"
        :enter-active-class="inClass"
        :leave-active-class="outClass"
        @before-enter="beforeOpen"
        @enter="opening"
        @after-enter="afterOpen"
        @before-leave="beforeClose"
        @leave="closing"
        @after-leave="afterClose"
      >
        <div
          v-show="show"
          ref="vm-wrapper"
          tabindex="0"
          :class="['vm-wrapper', wrapperClass, id]"
          :style="{ 'z-index': zIndex, cursor: enableClose ? 'pointer' : 'default' }"
          @click="clickOutside($event)"
          @keydown="keydown($event)"
        >
          <div
            ref="vm"
            :class="['vm', modalClass]"
            :style="modalStyle"
            role="dialog"
            :aria-label="title"
            aria-modal="true"
          >
            <slot name="titlebar">
              <div class="vm-titlebar">
                <h3 class="vm-title">
                  {{ title }}
                </h3>
                <button
                  v-if="enableClose"
                  type="button"
                  class="vm-btn-close"
                  @click.prevent="close"
                ></button>
              </div>
            </slot>
            <slot name="content">
              <div class="vm-content">
                <slot></slot>
              </div>
            </slot>
          </div>
        </div>
      </transition>
    </portal>
  </div>
</template>

<script>
import { Portal } from '@linusborg/vue-simple-portal';
let animatingZIndex = 0;

export default {
  name: 'VueModal',
  components: {
    Portal
  },
  model: {
    prop: 'basedOn',
    event: 'close'
  },
  props: {
    title: {
      type: String,
      default: ''
    },
    baseZindex: {
      type: Number,
      default: 1051
    },
    bgClass: {
      type: String,
      default: ''
    },
    wrapperClass: {
      type: String,
      default: ''
    },
    modalClass: {
      type: String,
      default: ''
    },
    modalStyle: {
      type: Object,
      default: () => ({})
    },
    inClass: {
      type: String,
      default: 'vm-fadeIn'
    },
    outClass: {
      type: String,
      default: 'vm-fadeOut'
    },
    bgInClass: {
      type: String,
      default: 'vm-fadeIn'
    },
    bgOutClass: {
      type: String,
      default: 'vm-fadeOut'
    },
    appendTo: {
      type: String,
      default: 'body'
    },
    live: {
      type: Boolean,
      default: false
    },
    enableClose: {
      type: Boolean,
      default: true
    },
    basedOn: {
      type: Boolean,
      default: false
    }
  },
  data: function () {
    return {
      zIndex: 0,
      id: null,
      show: false,
      mount: false,
      elToFocus: null
    };
  },
  created () {
    if (this.live) {
      this.mount = true;
    }
  },
  mounted () {
    this.id = 'vm-' + this._uid;
    this.$watch('basedOn', function (newVal) {
      if (newVal) {
        this.mount = true;
        this.$nextTick(() => {
          this.show = true;
        });
      } else {
        this.show = false;
      }
    }, {
      immediate: true
    });
  },
  beforeDestroy () {
    this.elToFocus = null;
  },
  methods: {
    close () {
      if (this.enableClose === true) {
        this.$emit('close', false);
      }
    },
    clickOutside (e) {
      if (e.target === this.$refs['vm-wrapper']) {
        this.close();
      }
    },
    keydown (e) {
      if (e.which === 27) {
        this.close();
      }
      if (e.which === 9) {
        // Get only visible elements
        let all = [].slice.call(this.$refs['vm-wrapper'].querySelectorAll('input, select, textarea, button, a'));
        all = all.filter(function (el) {
          return !!(el.offsetWidth || el.offsetHeight || el.getClientRects().length);
        });
        if (e.shiftKey) {
          if (e.target === all[0] || e.target === this.$refs['vm-wrapper']) {
            e.preventDefault();
            all[all.length - 1].focus();
          }
        } else {
          if (e.target === all[all.length - 1]) {
            e.preventDefault();
            all[0].focus();
          }
        }
      }
    },
    getTopZindex () {
      let toret = 0;
      const all = document.querySelectorAll('.vm-wrapper');
      for (let i = 0; i < all.length; i++) {
        if (all[i].display === 'none') {
          continue;
        }
        toret = parseInt(all[i].style.zIndex) > toret ? parseInt(all[i].style.zIndex) : toret;
      }
      return toret;
    },
    modalsVisible () {
      const all = document.querySelectorAll('.vm-wrapper');
      // We cannot return false unless we make sure that there are not any modals visible
      let foundVisible = 0;
      for (let i = 0; i < all.length; i++) {
        if (all[i].display === 'none') {
          continue;
        }
        if (parseInt(all[i].style.zIndex) > 0) {
          foundVisible++;
        }
      }
      return foundVisible;
    },
    handleFocus (wrapper) {
      const autofocus = wrapper.querySelector('[autofocus]');
      if (autofocus) {
        autofocus.focus();
      } else {
        const focusable = wrapper.querySelectorAll('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
        focusable.length ? focusable[0].focus() : wrapper.focus();
      }
    },
    beforeOpen () {
      // console.log('beforeOpen');
      this.elToFocus = document.activeElement;
      const lastZindex = this.getTopZindex();
      if (animatingZIndex) {
        this.zIndex = animatingZIndex + 2;
      } else {
        this.zIndex = (lastZindex === 0) ? this.baseZindex : lastZindex + 2;
      }
      animatingZIndex = this.zIndex;
      this.$emit('before-open');
    },
    opening () {
      // console.log('opening');
      this.$emit('opening');
    },
    afterOpen () {
      // console.log('afterOpen');
      this.handleFocus(this.$refs['vm-wrapper']);
      this.$emit('after-open');
    },
    beforeClose () {
      // console.log('beforeClose');
      this.$emit('before-close');
    },
    closing () {
      // console.log('closing');
      this.$emit('closing');
    },
    afterClose () {
      // console.log('afterClose');
      this.zIndex = 0;
      if (!this.live) {
        this.mount = false;
      }
      this.$nextTick(() => {
        window.requestAnimationFrame(() => {
          const lastZindex = this.getTopZindex();
          if (lastZindex > 0) {
            const all = document.querySelectorAll('.vm-wrapper');
            for (let i = 0; i < all.length; i++) {
              const wrapper = all[i];
              if (wrapper.display === 'none') {
                continue;
              }
              if (parseInt(wrapper.style.zIndex) === lastZindex) {
                if (wrapper.contains(this.elToFocus)) {
                  this.elToFocus.focus();
                } else {
                  // console.log(wrapper);
                  this.handleFocus(wrapper);
                }
                break;
              }
            }
          } else {
            if (document.body.contains(this.elToFocus)) {
              this.elToFocus.focus();
            }
          }
          animatingZIndex = 0;
          this.$emit('after-close');
        });
      });
    }
  }
};
</script>

<style>
  .vm-backdrop {position: fixed; top: 0; right: 0; bottom: 0; left: 0; background-color: rgba(0, 0, 0, 0.5);}
  .vm-wrapper {position: fixed; top: 0; right: 0; bottom: 0; left: 0; overflow-x: hidden; overflow-y: auto; outline: 0;}
  .vm {position: relative; margin: 0px auto; width: calc(100% - 20px); min-width: 110px; max-width:500px; background-color: #fff; top:30px; cursor: default; box-shadow: 0 5px 15px rgba(0, 0, 0, 0.5);}
  .vm-titlebar {padding:10px 15px 10px 15px; overflow: auto; border-bottom: 1px solid #e5e5e5;}
  .vm-title {margin-top:2px; margin-bottom: 0px; display: inline-block; font-size:18px; font-weight: normal;}
  .vm-btn-close {color: #ccc; padding: 0px; cursor: pointer;  background: 0 0; border: 0; float: right; font-size: 24px; line-height: 1em;}
  .vm-btn-close:before {content: '×'; font-family: Arial;}
  .vm-btn-close:hover, .vm-btn-close:focus, .vm-btn-close:focus:hover{color:#bbb; border-color: transparent; background-color: transparent;}
  .vm-content {padding:10px 15px 15px 15px;}
  .vm-content .full-hr {width: auto; border: 0; border-top: 1px solid #e5e5e5; margin-top:15px; margin-bottom:15px; margin-left:-14px; margin-right:-14px;}
  .vm-fadeIn {animation-name: vm-fadeIn;}
  @keyframes vm-fadeIn {0% {opacity: 0} 100% {opacity: 1}}
  .vm-fadeOut {animation-name: vm-fadeOut;}
  @keyframes vm-fadeOut {0% {opacity: 1} 100% {opacity: 0}}
  .vm-fadeIn, .vm-fadeOut {animation-duration: .25s; animation-fill-mode: both;}
</style>
