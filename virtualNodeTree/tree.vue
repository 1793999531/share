<template>
  <div
    :class="{
      'el-tree--highlight-current': highlightCurrent,
      'is-dragging': !!dragState.draggingNode,
      'is-drop-not-allow': !dragState.allowDrop,
      'is-drop-inner': dragState.dropType === 'inner'
    }"
    class="el-tree"
    role="tree"
  >
    <virtual-list
      :style="{ height: '100%', 'overflow-y': 'auto' }"
      :data-key="getNodeKey"
      :data-sources="visibleList"
      :data-component="itemComponent"
      :keeps="keeps"
      :extra-props="{
        renderAfterExpand,
        showCheckbox,
        renderContent,
        props,
        onNodeExpand: handleNodeExpand}"
    />
    <div v-if="isEmpty" class="el-tree__empty-block">
      <span class="el-tree__empty-text">{{ emptyText }}</span>
    </div>
    <div
      v-show="dragState.showDropIndicator"
      ref="dropIndicator"
      class="el-tree__drop-indicator"
    >
    </div>
  </div>
</template>

<script>
import TreeStore from './model/tree-store';
import VirtualList from 'vue-virtual-scroll-list';
import { getNodeKey, findNearestComponent } from './model/util';
// import ElTreeNode from './tree-node.vue';
import ElVirtualNode from './tree-virtual-node.vue';
import { t } from 'element-ui/src/locale';
import emitter from 'element-ui/src/mixins/emitter';
import { addClass, removeClass } from 'element-ui/src/utils/dom';

export default {
  name: 'ElTree',

  mixins: [emitter],

  components: {
    VirtualList
    // ElTreeNode
  },

  data() {
    return {
      store: null,
      root: null,
      currentNode: null,
      treeItems: null,
      checkboxItems: [],
      dragState: {
        showDropIndicator: false,
        draggingNode: null,
        dropNode: null,
        allowDrop: true
      },
      itemComponent: ElVirtualNode
    };
  },

  props: {
    data: {
      type: Array
    },
    keeps: {
      type: Number,
      default: 30
    },
    emptyText: {
      type: String,
      default() {
        return t('el.tree.emptyText');
      }
    },
    renderAfterExpand: {
      type: Boolean,
      default: true
    },
    nodeKey: String,
    checkStrictly: Boolean,
    defaultExpandAll: Boolean,
    expandOnClickNode: {
      type: Boolean,
      default: true
    },
    checkOnClickNode: Boolean,
    checkDescendants: {
      type: Boolean,
      default: false
    },
    autoExpandParent: {
      type: Boolean,
      default: true
    },
    defaultCheckedKeys: Array,
    defaultExpandedKeys: Array,
    currentNodeKey: [String, Number],
    renderContent: Function,
    showCheckbox: {
      type: Boolean,
      default: false
    },
    draggable: {
      type: Boolean,
      default: false
    },
    allowDrag: Function,
    allowDrop: Function,
    props: {
      default() {
        return {
          children: 'children',
          label: 'label',
          disabled: 'disabled'
        };
      }
    },
    lazy: {
      type: Boolean,
      default: false
    },
    highlightCurrent: Boolean,
    load: Function,
    filterNodeMethod: Function,
    accordion: Boolean,
    indent: {
      type: Number,
      default: 18
    },
    iconClass: String,
    extraLine: {
      type: Number,
      default: 8
    }
  },

  computed: {
    children: {
      set(value) {
        this.data = value;
      },
      get() {
        return this.data;
      }
    },

    treeItemArray() {
      return Array.prototype.slice.call(this.treeItems);
    },

    isEmpty() {
      const { childNodes } = this.root;
      return !childNodes || childNodes.length === 0 || childNodes.every(({ visible }) => !visible);
    },

    visibleList() {
      return this.flattenTree(this.root.childNodes);
    }
  },

  watch: {
    defaultCheckedKeys(newVal) {
      this.store.setDefaultCheckedKey(newVal);
    },

    defaultExpandedKeys(newVal) {
      this.store.defaultExpandedKeys = newVal;
      this.store.setDefaultExpandedKeys(newVal);
    },

    data(newVal) {
      this.store.setData(newVal);
      let keys = this.getInitCheckedKeys(newVal);
      this.store.setDefaultCheckedKey(keys);
    },

    checkboxItems(val) {
      Array.prototype.forEach.call(val, (checkbox) => {
        checkbox.setAttribute('tabindex', -1);
      });
    },

    checkStrictly(newVal) {
      this.store.checkStrictly = newVal;
    }
  },

  methods: {
    flattenTree(datas) {
      return datas.reduce((conn, data) => {
        conn.push(data);
        if (data.expanded && data.childNodes.length) {
          conn.push(...this.flattenTree(data.childNodes));
        }

        return conn;
      }, []);
    },
    getInitCheckedKeys(datas) {
      let keys = datas.reduce((conn, data) => {
        if (data.checked != null && data.checked === true) {
          conn.push(data[this.nodeKey]);
        }
        if (data.children && data.children.length > 0) {
          let subKeys = this.getInitCheckedKeys(data.children);
          conn = conn.concat(subKeys);
        }
        return conn;
      }, []);

      return keys;
    },
    filter(value) {
      if (!this.filterNodeMethod) throw new Error('[Tree] filterNodeMethod is required when filter');
      this.store.filter(value);
    },

    getNodeKey(node) {
      return getNodeKey(this.nodeKey, node.data);
    },

    getNodePath(data) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in getNodePath');
      const node = this.store.getNode(data);
      if (!node) return [];
      const path = [node.data];
      let parent = node.parent;
      while (parent && parent !== this.root) {
        path.push(parent.data);
        parent = parent.parent;
      }
      return path.reverse();
    },

    getCheckedNodes(leafOnly, includeHalfChecked) {
      return this.store.getCheckedNodes(leafOnly, includeHalfChecked);
    },

    getCheckedKeys(leafOnly) {
      return this.store.getCheckedKeys(leafOnly);
    },

    getCurrentNode() {
      const currentNode = this.store.getCurrentNode();
      return currentNode ? currentNode.data : null;
    },

    getCurrentKey() {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in getCurrentKey');
      const currentNode = this.getCurrentNode();
      return currentNode ? currentNode[this.nodeKey] : null;
    },

    setCheckedNodes(nodes, leafOnly) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCheckedNodes');
      this.store.setCheckedNodes(nodes, leafOnly);
    },

    setCheckedKeys(keys, leafOnly) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCheckedKeys');
      this.store.setCheckedKeys(keys, leafOnly);
    },

    setChecked(data, checked, deep) {
      this.store.setChecked(data, checked, deep);
    },

    getHalfCheckedNodes() {
      return this.store.getHalfCheckedNodes();
    },

    getHalfCheckedKeys() {
      return this.store.getHalfCheckedKeys();
    },

    setCurrentNode(node) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCurrentNode');
      this.store.setUserCurrentNode(node);
    },

    setCurrentKey(key) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCurrentKey');
      this.store.setCurrentNodeKey(key);
    },

    getNode(data) {
      return this.store.getNode(data);
    },

    remove(data) {
      this.store.remove(data);
    },

    append(data, parentNode) {
      this.store.append(data, parentNode);
    },

    insertBefore(data, refNode) {
      this.store.insertBefore(data, refNode);
    },

    insertAfter(data, refNode) {
      this.store.insertAfter(data, refNode);
    },

    handleNodeExpand(nodeData, node, instance) {
      this.broadcast('ElTreeNode', 'tree-node-expand', node);
      this.$emit('node-expand', nodeData, node, instance);
    },

    updateKeyChildren(key, data) {
      if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in updateKeyChild');
      this.store.updateChildren(key, data);
    },

    initTabIndex() {
      this.treeItems = this.$el.querySelectorAll('.is-focusable[role=treeitem]');
      this.checkboxItems = this.$el.querySelectorAll('input[type=checkbox]');
      const checkedItem = this.$el.querySelectorAll('.is-checked[role=treeitem]');
      if (checkedItem.length) {
        checkedItem[0].setAttribute('tabindex', 0);
        return;
      }
      this.treeItems[0] && this.treeItems[0].setAttribute('tabindex', 0);
    },

    handleKeydown(ev) {
      const currentItem = ev.target;
      if (currentItem.className.indexOf('el-tree-node') === -1) return;
      const keyCode = ev.keyCode;
      this.treeItems = this.$el.querySelectorAll('.is-focusable[role=treeitem]');
      const currentIndex = this.treeItemArray.indexOf(currentItem);
      let nextIndex;
      if ([38, 40].indexOf(keyCode) > -1) { // up、down
        ev.preventDefault();
        if (keyCode === 38) { // up
          nextIndex = currentIndex !== 0 ? currentIndex - 1 : 0;
        } else {
          nextIndex = (currentIndex < this.treeItemArray.length - 1) ? currentIndex + 1 : 0;
        }
        this.treeItemArray[nextIndex].focus(); // 选中
      }
      if ([37, 39].indexOf(keyCode) > -1) { // left、right 展开
        ev.preventDefault();
        currentItem.click(); // 选中
      }
      const hasInput = currentItem.querySelector('[type="checkbox"]');
      if ([13, 32].indexOf(keyCode) > -1 && hasInput) { // space enter选中checkbox
        ev.preventDefault();
        hasInput.click();
      }
    }
  },

  created() {
    this.isTree = true;

    this.store = new TreeStore({
      key: this.nodeKey,
      data: this.data,
      lazy: this.lazy,
      props: this.props,
      load: this.load,
      currentNodeKey: this.currentNodeKey,
      checkStrictly: this.checkStrictly,
      checkDescendants: this.checkDescendants,
      defaultCheckedKeys: this.defaultCheckedKeys,
      defaultExpandedKeys: this.defaultExpandedKeys,
      autoExpandParent: this.autoExpandParent,
      defaultExpandAll: this.defaultExpandAll,
      filterNodeMethod: this.filterNodeMethod
    });

    this.root = this.store.root;

    let dragState = this.dragState;
    this.$on('tree-node-drag-start', (event, treeNode) => {
      if (typeof this.allowDrag === 'function' && !this.allowDrag(treeNode.node)) {
        event.preventDefault();
        return false;
      }
      event.dataTransfer.effectAllowed = 'move';

      // wrap in try catch to address IE's error when first param is 'text/plain'
      if (event && event.dataTransfer && event.dataTransfer.setData) {
        event.dataTransfer.setData('text/plain', '');
      }
      dragState.draggingNode = treeNode;
      this.$emit('node-drag-start', treeNode.node, event);
    });

    this.$on('tree-node-drag-over', (event) => {
      const dropNode = findNearestComponent(event.target, 'ElTreeNode');
      const oldDropNode = dragState.dropNode;
      if (oldDropNode && oldDropNode !== dropNode) {
        removeClass(oldDropNode.$el, 'is-drop-inner');
      }
      const draggingNode = dragState.draggingNode;
      if (!draggingNode || !dropNode) return;

      let dropPrev = true;
      let dropInner = true;
      let dropNext = true;
      let userAllowDropInner = true;
      if (typeof this.allowDrop === 'function') {
        dropPrev = this.allowDrop(draggingNode.node, dropNode.node, 'prev');
        userAllowDropInner = dropInner = this.allowDrop(draggingNode.node, dropNode.node, 'inner');
        dropNext = this.allowDrop(draggingNode.node, dropNode.node, 'next');
      }
      event.dataTransfer.dropEffect = dropInner ? 'move' : 'none';
      if ((dropPrev || dropInner || dropNext) && oldDropNode !== dropNode) {
        if (oldDropNode) {
          this.$emit('node-drag-leave', draggingNode.node, oldDropNode.node, event);
        }
        this.$emit('node-drag-enter', draggingNode.node, dropNode.node, event);
      }

      if (dropPrev || dropInner || dropNext) {
        dragState.dropNode = dropNode;
      }

      if (dropNode.node.nextSibling === draggingNode.node) {
        dropNext = false;
      }
      if (dropNode.node.previousSibling === draggingNode.node) {
        dropPrev = false;
      }
      if (dropNode.node.contains(draggingNode.node, false)) {
        dropInner = false;
      }
      if (draggingNode.node === dropNode.node || draggingNode.node.contains(dropNode.node)) {
        dropPrev = false;
        dropInner = false;
        dropNext = false;
      }

      const targetPosition = dropNode.$el.getBoundingClientRect();
      const treePosition = this.$el.getBoundingClientRect();

      let dropType;
      const prevPercent = dropPrev ? (dropInner ? 0.25 : (dropNext ? 0.45 : 1)) : -1;
      const nextPercent = dropNext ? (dropInner ? 0.75 : (dropPrev ? 0.55 : 0)) : 1;

      let indicatorTop = -9999;
      const distance = event.clientY - targetPosition.top;
      if (distance < targetPosition.height * prevPercent) {
        dropType = 'before';
      } else if (distance > targetPosition.height * nextPercent) {
        dropType = 'after';
      } else if (dropInner) {
        dropType = 'inner';
      } else {
        dropType = 'none';
      }

      const iconPosition = dropNode.$el.querySelector('.el-tree-node__expand-icon').getBoundingClientRect();
      const dropIndicator = this.$refs.dropIndicator;
      if (dropType === 'before') {
        indicatorTop = iconPosition.top - treePosition.top;
      } else if (dropType === 'after') {
        indicatorTop = iconPosition.bottom - treePosition.top;
      }
      dropIndicator.style.top = indicatorTop + 'px';
      dropIndicator.style.left = (iconPosition.right - treePosition.left) + 'px';

      if (dropType === 'inner') {
        addClass(dropNode.$el, 'is-drop-inner');
      } else {
        removeClass(dropNode.$el, 'is-drop-inner');
      }

      dragState.showDropIndicator = dropType === 'before' || dropType === 'after';
      dragState.allowDrop = dragState.showDropIndicator || userAllowDropInner;
      dragState.dropType = dropType;
      this.$emit('node-drag-over', draggingNode.node, dropNode.node, event);
    });

    this.$on('tree-node-drag-end', (event) => {
      const { draggingNode, dropType, dropNode } = dragState;
      event.preventDefault();
      event.dataTransfer.dropEffect = 'move';

      if (draggingNode && dropNode) {
        const draggingNodeCopy = { data: draggingNode.node.data };
        if (dropType !== 'none') {
          draggingNode.node.remove();
        }
        if (dropType === 'before') {
          dropNode.node.parent.insertBefore(draggingNodeCopy, dropNode.node);
        } else if (dropType === 'after') {
          dropNode.node.parent.insertAfter(draggingNodeCopy, dropNode.node);
        } else if (dropType === 'inner') {
          dropNode.node.insertChild(draggingNodeCopy);
        }
        if (dropType !== 'none') {
          this.store.registerNode(draggingNodeCopy);
        }

        removeClass(dropNode.$el, 'is-drop-inner');

        this.$emit('node-drag-end', draggingNode.node, dropNode.node, dropType, event);
        if (dropType !== 'none') {
          this.$emit('node-drop', draggingNode.node, dropNode.node, dropType, event);
        }
      }
      if (draggingNode && !dropNode) {
        this.$emit('node-drag-end', draggingNode.node, null, dropType, event);
      }

      dragState.showDropIndicator = false;
      dragState.draggingNode = null;
      dragState.dropNode = null;
      dragState.allowDrop = true;
    });
  },

  mounted() {
    this.initTabIndex();
    this.$el.addEventListener('keydown', this.handleKeydown);
  },

  updated() {
    this.treeItems = this.$el.querySelectorAll('[role=treeitem]');
    this.checkboxItems = this.$el.querySelectorAll('input[type=checkbox]');
  }
};
</script>
<style>
  .el-tree {
    height: 100%;
  }

  .el-scrollbar .el-scrollbar__view .el-select-dropdown__item {
    height: auto;
    max-height: 274px;
    padding: 0;
    overflow: hidden;
    overflow-y: auto
  }

  .el-select-dropdown__item.selected {
    font-weight: 400
  }

  ul li .el-tree .el-tree-node__content {
    height: auto;
    padding: 0 20px
  }

  .el-tree-node__label {
    font-weight: 400
  }

  .el-tree .is-current .el-tree-node__label, .el-tree .is-current .ivu-tree-title {
    color: #409eff;
    font-weight: 700
  }

  .el-tree .is-current .el-tree-node__children .el-tree-node__label {
    color: #606266;
    font-weight: 400
  }


  .el-checkbox {
    color: #606266;
    font-weight: 500;
    font-size: 14px;
    cursor: pointer;
    white-space: nowrap;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    margin-right: 30px
  }

  .el-checkbox.is-bordered {
    padding: 9px 20px 9px 10px;
    border-radius: 4px;
    border: 1px solid #dcdfe6;
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    line-height: normal;
    height: 40px
  }

  .el-checkbox.is-bordered.is-checked {
    border-color: #409eff
  }

  .el-checkbox.is-bordered.is-disabled {
    border-color: #ebeef5;
    cursor: not-allowed
  }

  .el-checkbox.is-bordered + .el-checkbox.is-bordered {
    margin-left: 10px
  }

  .el-checkbox.is-bordered.el-checkbox--medium {
    padding: 7px 20px 7px 10px;
    border-radius: 4px;
    height: 36px
  }

  .el-checkbox.is-bordered.el-checkbox--medium .el-checkbox__label {
    line-height: 17px;
    font-size: 14px
  }

  .el-checkbox.is-bordered.el-checkbox--medium .el-checkbox__inner {
    height: 14px;
    width: 14px
  }

  .el-checkbox.is-bordered.el-checkbox--small {
    padding: 5px 15px 5px 10px;
    border-radius: 3px;
    height: 32px
  }

  .el-checkbox.is-bordered.el-checkbox--small .el-checkbox__label {
    line-height: 15px;
    font-size: 12px
  }

  .el-checkbox.is-bordered.el-checkbox--small .el-checkbox__inner {
    height: 12px;
    width: 12px
  }

  .el-checkbox.is-bordered.el-checkbox--small .el-checkbox__inner:after {
    height: 6px;
    width: 2px
  }

  .el-checkbox.is-bordered.el-checkbox--mini {
    padding: 3px 15px 3px 10px;
    border-radius: 3px;
    height: 28px
  }

  .el-checkbox.is-bordered.el-checkbox--mini .el-checkbox__label {
    line-height: 12px;
    font-size: 12px
  }

  .el-checkbox-button__inner, .el-checkbox__input {
    line-height: 1;
    vertical-align: middle;
    white-space: nowrap;
    outline: 0
  }

  .el-checkbox.is-bordered.el-checkbox--mini .el-checkbox__inner {
    height: 12px;
    width: 12px
  }

  .el-checkbox.is-bordered.el-checkbox--mini .el-checkbox__inner:after {
    height: 6px;
    width: 2px
  }

  .el-checkbox__input {
    cursor: pointer
  }

  .el-checkbox__input.is-disabled .el-checkbox__inner {
    background-color: #edf2fc;
    border-color: #dcdfe6;
    cursor: not-allowed
  }

  .el-checkbox__input.is-disabled .el-checkbox__inner:after {
    cursor: not-allowed;
    border-color: #c0c4cc
  }

  .el-checkbox__input.is-disabled .el-checkbox__inner + .el-checkbox__label {
    cursor: not-allowed
  }

  .el-checkbox__input.is-disabled.is-checked .el-checkbox__inner {
    background-color: #f2f6fc;
    border-color: #dcdfe6
  }

  .el-checkbox__input.is-disabled.is-checked .el-checkbox__inner:after {
    border-color: #c0c4cc
  }

  .el-checkbox__input.is-disabled.is-indeterminate .el-checkbox__inner {
    background-color: #f2f6fc;
    border-color: #dcdfe6
  }

  .el-checkbox__input.is-disabled.is-indeterminate .el-checkbox__inner:before {
    background-color: #c0c4cc;
    border-color: #c0c4cc
  }

  .el-checkbox__input.is-checked .el-checkbox__inner, .el-checkbox__input.is-indeterminate .el-checkbox__inner {
    background-color: #409eff;
    border-color: #409eff
  }

  .el-checkbox__input.is-disabled + span.el-checkbox__label {
    color: #c0c4cc;
    cursor: not-allowed
  }

  .el-checkbox__input.is-checked .el-checkbox__inner:after {
    -webkit-transform: rotate(45deg) scaleY(1);
    transform: rotate(45deg) scaleY(1)
  }

  .el-checkbox__input.is-checked + .el-checkbox__label {
    color: #409eff
  }

  .el-checkbox__input.is-focus .el-checkbox__inner {
    border-color: #409eff
  }

  .el-checkbox__input.is-indeterminate .el-checkbox__inner:before {
    content: "";
    position: absolute;
    display: block;
    background-color: #fff;
    height: 2px;
    -webkit-transform: scale(.5);
    transform: scale(.5);
    left: 0;
    right: 0;
    top: 5px
  }

  .el-checkbox__input.is-indeterminate .el-checkbox__inner:after {
    display: none
  }

  .el-checkbox__inner {
    display: inline-block;
    position: relative;
    border: 1px solid #dcdfe6;
    border-radius: 2px;
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    width: 14px;
    height: 14px;
    background-color: #fff;
    z-index: 1;
    -webkit-transition: border-color .25s cubic-bezier(.71, -.46, .29, 1.46), background-color .25s cubic-bezier(.71, -.46, .29, 1.46);
    transition: border-color .25s cubic-bezier(.71, -.46, .29, 1.46), background-color .25s cubic-bezier(.71, -.46, .29, 1.46)
  }

  .el-checkbox__inner:hover {
    border-color: #409eff
  }

  .el-checkbox__inner:after {
    -webkit-box-sizing: content-box;
    box-sizing: content-box;
    content: "";
    border: 1px solid #fff;
    border-left: 0;
    border-top: 0;
    height: 7px;
    left: 4px;
    position: absolute;
    top: 1px;
    -webkit-transform: rotate(45deg) scaleY(0);
    transform: rotate(45deg) scaleY(0);
    width: 3px;
    -webkit-transition: -webkit-transform .15s ease-in .05s;
    transition: -webkit-transform .15s ease-in .05s;
    transition: transform .15s ease-in .05s;
    transition: transform .15s ease-in .05s, -webkit-transform .15s ease-in .05s;
    -webkit-transform-origin: center;
    transform-origin: center
  }

  .el-checkbox__original {
    opacity: 0;
    outline: 0;
    position: absolute;
    margin: 0;
    width: 0;
    height: 0;
    z-index: -1
  }

  .el-checkbox-button, .el-checkbox-button__inner {
    display: inline-block;
    position: relative
  }

  .el-checkbox__label {
    display: inline-block;
    padding-left: 10px;
    line-height: 19px;
    font-size: 14px
  }

  .el-checkbox:last-of-type {
    margin-right: 0
  }

  .el-checkbox-button__inner {
    font-weight: 500;
    cursor: pointer;
    background: #fff;
    border: 1px solid #dcdfe6;
    border-left: 0;
    color: #606266;
    -webkit-appearance: none;
    text-align: center;
    -webkit-box-sizing: border-box;
    box-sizing: border-box;
    margin: 0;
    -webkit-transition: all .3s cubic-bezier(.645, .045, .355, 1);
    transition: all .3s cubic-bezier(.645, .045, .355, 1);
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    padding: 12px 20px;
    font-size: 14px;
    border-radius: 0
  }

  .el-checkbox-button__inner.is-round {
    padding: 12px 20px
  }

  .el-checkbox-button__inner:hover {
    color: #409eff
  }

  .el-checkbox-button__inner [class*=el-icon-] {
    line-height: .9
  }

  .el-checkbox-button__inner [class*=el-icon-] + span {
    margin-left: 5px
  }

  .el-checkbox-button__original {
    opacity: 0;
    outline: 0;
    position: absolute;
    margin: 0;
    z-index: -1
  }

  .el-checkbox-button.is-checked .el-checkbox-button__inner {
    color: #fff;
    background-color: #409eff;
    border-color: #409eff;
    -webkit-box-shadow: -1px 0 0 0 #8cc5ff;
    box-shadow: -1px 0 0 0 #8cc5ff
  }

  .el-checkbox-button.is-checked:first-child .el-checkbox-button__inner {
    border-left-color: #409eff
  }

  .el-checkbox-button.is-disabled .el-checkbox-button__inner {
    color: #c0c4cc;
    cursor: not-allowed;
    background-image: none;
    background-color: #fff;
    border-color: #ebeef5;
    -webkit-box-shadow: none;
    box-shadow: none
  }

  .el-checkbox-button.is-disabled:first-child .el-checkbox-button__inner {
    border-left-color: #ebeef5
  }

  .el-checkbox-button:first-child .el-checkbox-button__inner {
    border-left: 1px solid #dcdfe6;
    border-radius: 4px 0 0 4px;
    -webkit-box-shadow: none !important;
    box-shadow: none !important
  }

  .el-checkbox-button.is-focus .el-checkbox-button__inner {
    border-color: #409eff
  }

  .el-checkbox-button:last-child .el-checkbox-button__inner {
    border-radius: 0 4px 4px 0
  }

  .el-checkbox-button--medium .el-checkbox-button__inner {
    padding: 10px 20px;
    font-size: 14px;
    border-radius: 0
  }

  .el-checkbox-button--medium .el-checkbox-button__inner.is-round {
    padding: 10px 20px
  }

  .el-checkbox-button--small .el-checkbox-button__inner {
    padding: 9px 15px;
    font-size: 12px;
    border-radius: 0
  }

  .el-checkbox-button--small .el-checkbox-button__inner.is-round {
    padding: 9px 15px
  }

  .el-checkbox-button--mini .el-checkbox-button__inner {
    padding: 7px 15px;
    font-size: 12px;
    border-radius: 0
  }

  .el-checkbox-button--mini .el-checkbox-button__inner.is-round {
    padding: 7px 15px
  }

  .el-checkbox-group {
    font-size: 0
  }

  .el-tree {
    position: relative;
    cursor: default;
    background: #fff;
    color: #606266
  }

  .el-tree__empty-block {
    position: relative;
    min-height: 60px;
    text-align: center;
    width: 100%;
    height: 100%
  }

  .el-tree__empty-text {
    position: absolute;
    left: 50%;
    top: 50%;
    -webkit-transform: translate(-50%, -50%);
    transform: translate(-50%, -50%);
    color: #909399;
    font-size: 14px
  }

  .el-tree__drop-indicator {
    position: absolute;
    left: 0;
    right: 0;
    height: 1px;
    background-color: #409eff
  }

  .el-tree-node {
    white-space: nowrap;
    outline: 0
  }

  .el-tree-node:focus > .el-tree-node__content {
    background-color: #f5f7fa
  }

  .el-tree-node.is-drop-inner > .el-tree-node__content .el-tree-node__label {
    background-color: #409eff;
    color: #fff
  }

  .el-tree-node__content {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -webkit-box-align: center;
    -ms-flex-align: center;
    align-items: center;
    height: 26px;
    cursor: pointer
  }

  .el-tree-node__content > .el-tree-node__expand-icon {
    padding: 6px
  }

  .el-tree-node__content > label.el-checkbox {
    margin-right: 8px
  }

  .el-tree-node__content:hover {
    background-color: #f5f7fa
  }

  .el-tree.is-dragging .el-tree-node__content {
    cursor: move
  }

  .el-tree.is-dragging .el-tree-node__content * {
    pointer-events: none
  }

  .el-tree.is-dragging.is-drop-not-allow .el-tree-node__content {
    cursor: not-allowed
  }

  .el-tree-node__expand-icon {
    cursor: pointer;
    color: #c0c4cc;
    font-size: 12px;
    -webkit-transform: rotate(0);
    transform: rotate(0);
    -webkit-transition: -webkit-transform .3s ease-in-out;
    transition: -webkit-transform .3s ease-in-out;
    transition: transform .3s ease-in-out;
    transition: transform .3s ease-in-out, -webkit-transform .3s ease-in-out
  }

  .el-tree-node__expand-icon.expanded {
    -webkit-transform: rotate(90deg);
    transform: rotate(90deg)
  }

  .el-tree-node__expand-icon.is-leaf {
    color: transparent;
    cursor: default
  }

  .el-tree-node__label {
    font-size: 14px
  }

  .el-tree-node__loading-icon {
    margin-right: 8px;
    font-size: 14px;
    color: #c0c4cc
  }

  .el-tree-node > .el-tree-node__children {
    overflow: hidden;
    background-color: transparent
  }

  .el-tree-node.is-expanded > .el-tree-node__children {
    display: block
  }

  .el-tree--highlight-current .el-tree-node.is-current > .el-tree-node__content {
    background-color: #f0f7ff
  }
</style>
