<template>
    <virtual-list ref="virtualList"
                  :option="{ height: bigListHeight, itemHeight: 32 }"
                  :filterFn="filterFn"
                  :list="flattenTree">
        <template slot-scope="{item}">
            <div :style="{ paddingLeft: 24 * (item.level - 1) + 'px'}">
                <i :class="item.expand ? 'b-tree__expand' : 'b-tree__close'"
                   @click="toggleExpand(item)"
                   v-if="item.children && item.children.length" />
                <span v-else style="margin-right:5px"></span>
                <el-checkbox
                    v-model="item.checked"
                    :disabled="item.disabled"
                    :indeterminate="item.isIndeterminate"
                    @change="toggleChecked($event, item)">
                        <span v-if="renderFun" class="tree-node" v-html="renderFun(item)"></span>
                        <span v-else>{{item[optionName]}}</span>
                </el-checkbox>
            </div>
        </template>
    </virtual-list>
</template>


<script lang="ts">

    /**
     * @description: 支持大数据量的树组件
     * @author: cxs
     * @date: 2021/01/25
     */

    import {Component, Vue, Prop, Watch} from 'vue-property-decorator';
    import {Checkbox} from 'element-ui';
    import isUndefined from 'lodash/isUndefined';
    import VirtualList from './virtual_list';
    import cloneDeep from 'lodash/cloneDeep';

    @Component({
        components: {
            VirtualList,
            ElCheckbox: Checkbox
        }
    })
    export default class BigDataTree extends Vue {
        @Prop({
            default: () => {
                return [];
            }
        }) tree!: any[];
        @Prop({
            default: () => {
                return [];
            }
        }) value!: any[];
        @Prop({default: null}) renderContent!: any;
        @Prop({default: false}) defaultExpand!: boolean;
        @Prop({default: 500 }) bigListHeight!: any; // 虚拟列表的高度配置
        @Prop({default: 20}) timeout!: number;
        @Prop({default: ''}) filterValue!: string;
        @Prop({default: 1}) selectLevel!: number; // 收集的数据，默认收集一级目录下的
        @Prop({default: 'id'}) idProperty!: string;
        @Prop({default: 'name'}) optionName!: string;
        @Prop({default: null}) renderFun!: Function;
        @Prop({default: () => true, type: Function}) beforeRemove!: any;    // 默认return true

        private bacFlattenTree = [];
        private flattenTree = []; // 拍平后的树

        @Watch('tree')
        updateFlattenTree () {
            let tree = cloneDeep(this.tree);
            this.flattenTree = this.flatten(tree, 'children', 1, {
                level: 0,
                visible: true,
                expand: true,
                children: this.tree
            });
            this.bacFlattenTree = this.flattenTree; // 备份，用于筛选
            if (this.value.length) {
                this.setCheckedKeys(this.value);
            }
        }

        @Watch('value')
        updateCheck (val) {
            this.setCheckedKeys(val);
        }

        // 发送已选数据
        emitAllSelect () {
            let rs = this.bacFlattenTree.filter(item => item.checked && item.level > this.selectLevel);
            this.$emit('check-change', rs);
        }

        // 设置选中值
        setCheckedKeys (list) {
            let vm = this;
            list = list.map(item => item[vm.idProperty]);
            if (!vm.bacFlattenTree.length) {
                return;
            }

            // 没有直接全置为false，即什么都没选
            if (!list.length) {
                vm.bacFlattenTree.forEach(item => {

                    this.$set(item, 'checked', false);
                    this.$set(item, 'isIndeterminate', false);
                });
                return;
            }
            for (let i in vm.bacFlattenTree) {
                if (!vm.bacFlattenTree.hasOwnProperty(i)) {
                    return;
                }
                let item = vm.bacFlattenTree[i];
                if (item[vm.idProperty] && list.includes(item[vm.idProperty])) {
                    this.$set(item, 'checked', true);
                    vm.toggleParentCheck(item);
                } else {
                    this.$set(item, 'checked', false);
                    vm.toggleParentCheck(item);
                }
            }
        }

        // 监听勾选的变化
        async toggleChecked (val, item) {

            if (!val) {
                let doDel = await this.beforeRemove();
                if (!doDel) {
                    item.checked = !val; // 如果不允许删除，将删除的值改回
                    return;
                }
                this.$emit('on-delete', item);
            }
            this.$set(item, 'checked', val);
            this.$set(item, 'isIndeterminate', false); // 当前节点非半选

            // 递归修改子节点状态
            this.toggleChildrenStatus(item.children, val, 'checked');

            // 更新所有节点的状态
            if (item.level !== 1) {
                this.toggleParentCheck(item);
            }

            // 发送已选数据
            this.emitAllSelect();
        }

        filter (val) {
            if (!val) {
                this.flattenTree = this.bacFlattenTree;

                this.flattenTree.forEach(item => {

                    // 回到初始状态: 默认只展开第一层，即level为0的展开；默认只看第一层数据，即level小于2的都展示
                    if (item.level > 0) {
                        item.expand = false;
                    } else {
                        item.expand = true;
                    }
                    if (item.level <= this.selectLevel) {
                        item.visible = true;
                    } else {
                        item.visible = false;
                    }
                });

                return;
            }

            let filterRs = this.bacFlattenTree.filter(item => item.name.indexOf(val) > -1 && item.level > this.selectLevel);

            // todo 保留层级结构

            this.flattenTree = filterRs;

            this.flattenTree.forEach(item => item.visible = true); // 筛选后的数据都要可见
        }

                // 拍平数组
        flatten (list, childKey = 'children', level = 1, parent = null, defaultExpand = true) {
            let vm = this,
                arr = [];
            list.forEach(item => {
                item.level = level;

                // todo 后续支持按key来配置
                if (item.level > 0) {
                    item.expand = false;
                } else {
                    item.expand = true;
                }

                // if (isUndefined(item.expand)) {
                //     item.expand = defaultExpand;
                // }
                if (isUndefined(item.visible)) {
                    item.visible = true;
                }
                if (!parent.visible || !parent.expand) {
                    item.visible = false;
                }
                item.parent = parent;
                arr.push(item);
                if (item[childKey]) {
                    arr.push(
                        ...vm.flatten(
                            item[childKey],
                            childKey,
                            level + 1,
                            item,
                            defaultExpand
                        )
                    );
                }
            });
            return arr;
        };

        // 过滤条件，只展示打开的节点，即visible为true
        filterFn (arr) {
            return (arr || []).filter(
                item => item.visible
            );
        }

        updateView () {

            this.$refs.virtualList && this.$refs.virtualList.updateView();
        }

        // 展开树
        toggleExpand (item) {
            let isExpand = item.expand;
            if (isExpand) {
                this.collapse(item); // 折叠
            } else {
                this.expand(item); // 展开
            }
            this.updateView();
        }

        // 修复父节点状态，如某一节点勾选了，父节点要变成半选或者全选
        toggleParentCheck (item) {
            let curItem = item.parent;
            let {hasCheck, hasNoCheck} = this.recursionGetChildrenCheck({
                children: curItem.children,
                hasCheck: false,
                hasNoCheck: false
            });

            if (hasCheck && !hasNoCheck) {

                // 如果是全选
                this.$set(curItem, 'checked', true);
                this.$set(curItem, 'isIndeterminate', false);
            } else if (!hasCheck && hasNoCheck) {

                // 如果是全去勾选
                this.$set(curItem, 'checked', false);
                this.$set(curItem, 'isIndeterminate', false);
            } else {

                // 半选
                this.$set(curItem, 'checked', false);
                this.$set(curItem, 'isIndeterminate', true);
            }

            // 当前节点的父节点状态
            if (curItem.level > 1) {
                this.toggleParentCheck(curItem);
            }
        }
                // 判断子节点的勾选状态
        recursionGetChildrenCheck ({children, hasCheck, hasNoCheck}) {
            for (let i in children) {
                if (!children.hasOwnProperty(i)) {
                    return;
                }
                let item = children[i];
                if (!hasCheck && (item.checked || item.isIndeterminate)) {
                    hasCheck = true;
                }
                if (!hasNoCheck && !item.checked) {
                    hasNoCheck = true;
                }
                if (hasCheck && hasNoCheck) {
                    return {hasCheck, hasNoCheck};
                }
            }
            return {hasCheck, hasNoCheck};
        }

        // 递归子节点，修改状态
        toggleChildrenStatus (children, status, type = 'visible') {
            (children || []).forEach(node => {
                this.$set(node, type, status);
                if (type === 'checked') {
                    this.$set(node, 'isIndeterminate', false);
                }
                if (node.children) {
                    this.toggleChildrenStatus(node.children, status, type);
                }
            });
        }

        //展开节点
        expand (item) {
            item.expand = true;
            this.toggleChildrenStatus(item.children, true);
        }

        //折叠节点
        collapse (item) {
            item.expand = false;
            this.toggleChildrenStatus(item.children, false);
        }

        //折叠所有
        collapseAll (level = 1) {
            this.flattenTree.forEach(item => {
                item.expand = false;
                if (item.level !== level) {
                    item.visible = false;
                }
            });
            this.updateView();
        }
               //展开所有
        expandAll () {
            this.flattenTree.forEach(item => {
                item.expand = true;
                item.visible = true;
            });

            this.updateView();
        }
    }
</script>

<style lang="less" scoped>

    /*todo 节点宽度要动态计算*/
    .tree-node {
        display: inline-block;
        width: 130px;
    }

    .b-tree__content__item {
        padding: 5px;
        box-sizing: border-box;

        display: flex;
        justify-content: space-between;
        position: relative;
        align-items: center;
        cursor: pointer;
    }
    .b-tree__content__item:hover,
    .b-tree__content__item__selected {
        background-color: #d7d7d7;
    }
    .b-tree__content__item__icon {
        position: absolute;
        left: 0;
        color: #c0c4cc;
        z-index: 10;
    }
    .b-tree__close {
        display: inline-block;
        width: 0;
        height: 0;
        overflow: hidden;
        font-size: 0;
        margin-right: 5px;
        border-width: 5px;
        border-color: transparent transparent transparent #c0c4cc;
        border-style: dashed dashed dashed solid;
    }
    .b-tree__expand {
        display: inline-block;
        width: 0;
        height: 0;
        overflow: hidden;
        font-size: 0;
        margin-right: 5px;
        border-width: 5px;
        border-color: #c0c4cc transparent transparent transparent;
        border-style: solid dashed dashed dashed;
    }
</style>
