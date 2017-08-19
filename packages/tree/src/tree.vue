<template>
  <div class="el-tree" :class="{ 'el-tree--highlight-current': highlightCurrent }">
    <el-tree-node
      v-for="child in root.childNodes"
      :node="child"
      :props="props"
      :key="getNodeKey(child)"
      :render-content="renderContent"
      @node-expand="handleNodeExpand">
    </el-tree-node>
    <div class="el-tree__empty-block" v-if="!root.childNodes || root.childNodes.length === 0">
      <span class="el-tree__empty-text">{{ emptyText }}</span>
    </div>
  </div>
</template>

<script>
  import TreeStore from './model/tree-store';
  import {t} from 'element-ui/src/locale';
  import emitter from 'element-ui/src/mixins/emitter';
  import v3platform from 'element-ui/src/utils/v3platform';

  export default {
    name: 'ElTree',

    mixins: [emitter],

    components: {
      ElTreeNode: require('./tree-node.vue')
    },

    data() {
      return {
        store: null,
        root: null,
        currentNode: null
      };
    },

    props: {
      data: {
        type: Array
      },
      emptyText: {
        type: String,
        default() {
          return t('el.tree.emptyText');
        }
      },
      nodeKey: String,
      checkStrictly: Boolean,
      defaultExpandAll: Boolean,
      expandOnClickNode: {
        type: Boolean,
        default: true
      },
      autoExpandParent: {
        type: Boolean,
        default: true
      },
      defaultCheckedKeys: Array,
      defaultExpandedKeys: Array,
      renderContent: Function,
      showCheckbox: {
        type: Boolean,
        default: false
      },
      props: {
        default() {
          return {
            children: 'children',
            label: 'label',
            icon: 'icon',
            disabled: 'disabled'
          };
        }
      },
      lazy: {
        type: Boolean,
        default: false
      },
      highlightCurrent: Boolean,
      currentNodeKey: [String, Number],
      load: Function,
      filterNodeMethod: Function,
      accordion: Boolean,
      indent: {
        type: Number,
        default: 16
      },
      entityCode: String
    },

    computed: {
      children: {
        set(value) {
          this.data = value;
        },
        get() {
          return this.data;
        }
      }
    },

    watch: {
      defaultCheckedKeys(newVal) {
        this.store.defaultCheckedKeys = newVal;
        this.store.setDefaultCheckedKey(newVal);
      },
      defaultExpandedKeys(newVal) {
        this.store.defaultExpandedKeys = newVal;
        this.store.setDefaultExpandedKeys(newVal);
      },
      currentNodeKey(newVal) {
        this.store.setCurrentNodeKey(newVal);
        this.store.currentNodeKey = newVal;
      },
      data(newVal) {
        this.store.setData(newVal);
      }
    },

    methods: {
      filter(value) {
        if (!this.filterNodeMethod) throw new Error('[Tree] filterNodeMethod is required when filter');
        this.store.filter(value);
      },
      getNodeKey(node, index) {
        const nodeKey = this.nodeKey;
        if (nodeKey && node) {
          return node.data[nodeKey];
        }
        return index;
      },
      getCheckedNodes(leafOnly) {
        return this.store.getCheckedNodes(leafOnly);
      },
      getCheckedKeys(leafOnly) {
        return this.store.getCheckedKeys(leafOnly);
      },
      setCheckedNodes(nodes, leafOnly) {
        if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCheckedNodes');
        this.store.setCheckedNodes(nodes, leafOnly);
      },
      setCheckedKeys(keys, leafOnly) {
        if (!this.nodeKey) throw new Error('[Tree] nodeKey is required in setCheckedNodes');
        this.store.setCheckedKeys(keys, leafOnly);
      },
      setChecked(data, checked, deep) {
        this.store.setChecked(data, checked, deep);
      },
      handleNodeExpand(nodeData, node, instance) {
        this.broadcast('ElTreeNode', 'tree-node-expand', node);
        this.$emit('node-expand', nodeData, node, instance);
      },
      getTreeNode(data) {
        let children = this.$children;
        if (children && children.length > 0) {
          let iter = function(data, parent) {
            if (parent.node && (parent.node.data.id === data.id)) {
              return parent;
            } else {
              let list = parent.$children;
              if (list && list.length > 0) {
                for (let j = 0, l = list.length; j < l; j++) {
                  let item = list[j];
                  let rs = iter(data, item);
                  if (rs) {
                    return rs;
                  }
                }
              }
            }
          };
          for (let i = 0, len = children.length; i < len; i++) {
            let child = iter(data, children[i]);
            if (child) {
              return child;
            }
          }
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
        defaultCheckedKeys: this.defaultCheckedKeys,
        defaultExpandedKeys: this.defaultExpandedKeys,
        autoExpandParent: this.autoExpandParent,
        defaultExpandAll: this.defaultExpandAll,
        filterNodeMethod: this.filterNodeMethod
      });

      this.root = this.store.root;
      if (this.showCheckbox) {
        v3platform.markDsMultipleSelect(this, this.entityCode);
      }
      v3platform.registerCurrentHandler(this, (function(_this) {
          return function(entityCode, current) {
              if (_this.entityCode === entityCode) {
                   _this.$nextTick(function() {
                      let data = current.toMap();
                      let node = _this.getTreeNode(data);
                      if (node) {
                        node.handleCurrentChange();
                      }
                  });
              }
          };
      })(this));
      v3platform.registerSelectHandler(this, (function(_this) {
          return function(entityCode, records, isSel) {
              if (_this.entityCode === entityCode) {
                  _this.$nextTick(function() {
                      for (let i = 0, len = records.length; i < len; i++) {
                          let data = records[i].toMap();
                          let node = _this.store.getNodeFromTree(data);
                          if (node) {
                            node.setChecked(isSel, false);
                          }
                      }
                  });
              }
          };
      })(this));
    }
  };
</script>
