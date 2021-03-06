<template>
  <div :class="{'expand': isExpand}">
    <div class="zk-editor-utils flex-row fs-14">
      <ul class="zk-editor-menu flex-row row-left">
        <li v-for="item in utils" class="flex-row">
          <a class="flex-row row-center" :class="{ 'active': !item.unselect && item.active }" @click.stop="selectUtil(item)" href="javascript:;">
            <i :class="`fa fa-${item.icon}`"></i>
          </a>
          <span class="zk-editor-menu-separator" v-if="item.separator">|</span>
          <!--下拉选择框-->
          <dropdown
            v-if="item.dropMenus && item.dropMenus.length"
            :dropStyle="item.dropStyle"
            :dropList.sync="item.dropMenus"
            v-show="item.show"
            @selected="getSelectAtDropdown(item, $event)"
          ></dropdown>
        </li>
      </ul>
      <ul class="zk-editor-menu flex-row row-right">
        <li v-for="item in editorMode" class="flex-row">
          <a class="flex-row row-center" :class="{ 'active': !item.unselect && item.active }" @click.stop="selectModeUtil(item)" href="javascript:;">
            <i :class="`fa fa-${item.icon}`"></i>
          </a>
          <span class="zk-editor-utils-separator" v-if="item.separator">|</span>
        </li>
      </ul>
    </div>
    <div class="zk-editor-body clearfix" :class="{'show-preview': showPreview}">
      <div class="zk-editor-content pull-left" v-html="defaultValue" contenteditable ref="contenteditable" @mousedown.stop="editorMouseDown" @mouseup.stop="editorMouseUp" @keyup.stop="editorKeyUp"></div>
      <transition name="opacity">
        <div class="zk-editor-preview pull-left" v-if="showPreview" v-html="previewHtml"></div>
      </transition>
    </div>
    <!--插入代码弹框-->
    <insert-code :visible.sync="openCodeModal" @code="getInsertCode"></insert-code>
  </div>
</template>

<script>
  import hljs from 'highlight.js';
  import { editorMenu, editorMode } from '../utils/editor/menu';
  import { getSelection, getCurrentRange, removeAllRanges, restoreSelection, isSelectionEmpty } from '../utils/editor/selection';
  import { setUtilsStatusByRange } from '../utils/editor/utilStatus';
  import { setContentByRange } from '../utils/editor/content';

  export default {
    name: 'editor',
    props: {
      defaultValue: {
        type: String,
        default: ' <p><br></p> '
      }
    },
    data () {
      return {
        utils: [],
        editorMode: [...editorMode],
        dropdownUtil: null,
        showPreview: false,
        isExpand: false,
        timer: null,
        previewHtml: '',
        openCodeModal: false,
        _currentRange: null,
      };
    },
    created() {
      this.copyMenuData();
    },
    mounted() {
      const el = this.$refs.contenteditable;
      // 设置焦点
      el.focus();
      window.addEventListener('click', this.hideDropdownMenu, false);
    },
    methods: {
      highlightBlock(timestamp) {
        let blocks = document.querySelectorAll(`code[data-${timestamp}]`);
        blocks.forEach((block) => {
          hljs.highlightBlock(block);
        });
        const el = this.$refs.contenteditable;
        this.refreshPreviewHtml(el.innerHTML);
      },
      getInsertCode(data) {
        restoreSelection(this._currentRange);
        this.openCodeModal = false;
        const timestamp = Date.now();
        document.execCommand('insertHTML', false, `<pre><code class="${data.lang}" data-${timestamp}>${data.code}</code></pre><p><br></p>`);
        window.requestAnimationFrame(this.highlightBlock.bind(this, timestamp));
      },
      hideDropdownMenu() {
        if (this.dropdownUtil) {
          this.dropdownUtil.show = false;
        }
      },
      getSelectAtDropdown(util, selectedItem) {
        util.show = false;
        const el = this.$refs.contenteditable;
        const selection = getSelection();
        Array.from(el.childNodes, (child) => {
          // 判断选择的节点是否在选择区域内，如果在，就设置该节点的样式
          if (selection.containsNode(child, true)) {
            setContentByRange(child, util.name, selectedItem.val);
          }
          return child;
        });
        this.refreshPreviewHtml(el.innerHTML);
      },
      selectModeUtil(item) {
        // 当是活跃的工具（可选中）的元素时，才设置状态
        if (!item.unselect) {
          item.active = !item.active;
        }
        if (item.name === 'preview') {
          this.showPreview = item.active;
          const el = this.$refs.contenteditable;
          this.refreshPreviewHtml(el.innerHTML);
        } else if (item.name === 'expand') {
          this.isExpand = item.active;
        }
      },
      /**
       * 选择工具栏选项，并让选择的高亮/回复正常
       * @param item
       */
      selectUtil(item) {
        this._currentRange = getCurrentRange(this._currentRange);
        // 判断如果有选择项就阻止
        if (item.modal && isSelectionEmpty(this._currentRange)) {
          this.openCodeModal = true;
          return;
        }
        if (item.dropMenus && item.dropMenus.length) {
          item.show = !item.show;
          this.dropdownUtil = item;
          return;
        }
        // 当是活跃的工具（可选中）的元素时，才设置状态
        if (!item.unselect) {
          item.active = !item.active;
        }
        if (!item.dropMenus) {
          document.execCommand(item.name, false, null);
        }
        const el = this.$refs.contenteditable;
        this.refreshPreviewHtml(el.innerHTML);
      },
      editorKeyUp(ev) {
        // 当editor 内容删除完后，应该添加如下内容
        if (!ev.target.innerHTML) {
          ev.target.innerHTML = '<p><br></p>';
        }
        this.refreshPreviewHtml(ev.target.innerHTML);
      },
      /**
       * 更新preview 界面
       * @param html
       */
      refreshPreviewHtml(html) {
        if (this.value !== html) {
          this.$emit('refresh', html);
        }
        if (!this.showPreview || this.previewHtml === html) {
          return;
        }
        if (this.timer) {
          clearTimeout(this.timer);
        }
        this.timer = setTimeout(() => {
          this.previewHtml = html;
        }, 300);
      },
      editorMouseDown() {
        // 将所有的区域都从选区中移除。
        removeAllRanges();
      },
      editorMouseUp() {
        // 修改工具栏状态
        setUtilsStatusByRange(this.utils);
      },
      copyMenuData() {
        // 重新组装工具栏带有menus属性的下拉数据
        this.utils = editorMenu.map((item) => {
          if (item.dropMenus && item.dropMenus.length) {
            item.show = false;
          }
          return item;
        });
      },
    },
    components: {
      dropdown: () => import('./editor/dropdown.vue'),
      InsertCode: () => import('./editor/InsertCode.vue')
    },
    destroyed() {
      window.removeEventListener('click', this.hideDropdownMenu, false);
    }
  };
</script>

<style lang="scss" scoped>
  @import "../styles/color";
  .zk-editor-{
    &utils{
      background-color: $c-white;
      border: 1px solid $c-border;
      border-bottom: none;
    }
    &menu{
      padding: 4px 10px;
      & > li{
        position: relative;
        margin-right: 8px;
        & > a{
          padding: 4px;
          width: 26px;
          color: $c-gray;
          &.active, &:hover{
            color: $c-blue;
            background-color: $c-light-gray;
          }
        }
      }
      // 功能分隔线
      &-separator{
        font-size: 0;
        display: inline-block;
        height: 16px;
        width: 0;
        border-left: 1px solid $c-gray;
        margin-left: 8px;
      }
    }
    &body{
      position: relative;
      width: 100%;
      height: 300px;
      resize: vertical;
      overflow: auto;
      border: 1px solid $c-border;
      background-color: $c-white;
      .zk-editor-content{
        background-color: $c-white;
        word-wrap:break-word;
        word-break:break-all;
        position: absolute;
        left: 0;
        top: 0;
        height: 100%;
        width: 100%;
        padding: 10px;
        transition: width .5s;
        z-index: 2;
      }
      .zk-editor-preview{
        background-color: $c-white;
        position: absolute;
        right: 0;
        top: 0;
        padding: 10px;
        width: 50%;
        height: 100%;
        z-index: 1;
      }
      &.show-preview{
        .zk-editor-content{
          width: 50%;
          border-right: 1px solid $c-border;
        }
      }
    }
  }
  .expand{
    position: fixed;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
  }
  .code-input{
    min-height: 200px;
    resize: vertical;
  }
  // 动画过渡
  .opacity-enter-active, .opacity-leave-active {
    transition: all .5s;
  }
  .opacity-enter, .opacity-leave-to {
    opacity: 0;
  }
</style>
