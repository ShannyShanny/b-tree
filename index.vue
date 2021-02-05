<template>
    <virtual-list ref="virtualList"
                  :option="{ height: bigListHeight, itemHeight: 32 }"
                  :filterFn="filterFn"
                  :list="flattenTree">
        <template slot-scope="{item}">
            <div class="b-tree-node__wrap" :style="{ paddingLeft: getPaddingLeft(item) }">
                <i v-if="item[props.children] && item[props.children].length"
                   class="el-tree-node__expand-icon el-icon-caret-right"
                   :style="{ transform: item.expand ? 'rotate(90deg)' :  'rotate(0)' }"
                   @click="toggleExpand(item)"></i>
                <el-checkbox v-if="showCheckbox"
                             v-model="item.checked"
                             class="b-tree__checkbox"
                             :disabled="item.disabled"
                             :indeterminate="item.isIndeterminate"
                             @change="toggleChecked($event, item)">
                </el-checkbox>
                <slot :item="item">
                    <span v-if="renderContent"
                          class="b-tree-node_value common-ellipsis"
                          v-overflow-tooltip
                          v-qtip="renderContent(item)"
                          qanchor="top"
                          v-html="getRenderHtml(item)"
                          @click="toggleChecked(!item.checked, item)"></span>
                    <span v-else
                          v-qtip="item[props.optionName]"
                          qanchor="top"
                          v-overflow-tooltip
                          @click="toggleChecked(!item.checked, item)">{{item[props.optionName]}}</span>
                </slot>
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
    import VirtualList from './virtual_list';
    import cloneDeep from 'lodash/cloneDeep';
    import throttle from 'lodash/throttle';
    import DOMPurify from 'dompurify';
    import { DOMPURIFY_CONFIG } from 'const/text';

    interface Props {
        optionName?: string;
        children?: string;
        disabled?: string;
    }

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
        @Prop({default: false}) showCheckbox!: boolean;
        @Prop({default: false}) defaultExpandAll!: boolean;
        @Prop({
            default: 0,
            validator: (val) =>{
                return val >= 0;
            }
        }) defaultExpandedLevel!: number; // 默认展开的级别
        @Prop({
            default: () => {
                return [];
            }
        }) defaultSelected!: any[]; // 默认展示的值，可传对象数组或者基本类型（id）数组
        @Prop({
            default: () => {
                return [];
            }
        }) defaultExpandedKeys!: string[];
        @Prop({
            default: () => {
                return [];
            }
        }) defaultCheckedKeys!: string[];
        @Prop({default: 500 }) bigListHeight!: number; // 虚拟列表的高度配置
        @Prop({default: 20}) timeout!: number;
        @Prop({default: ''}) filterValue!: string;
        @Prop({default: 'id'}) idProperty!: string;
        @Prop({default: () => {
                return {
                    optionName: 'name',
                    children: 'children'
                };
            }
        }) props!: Props;
        @Prop({default: null}) renderContent!: Function | null;
        @Prop({default: (): boolean => true }) beforeRemove!: Function;    // 去勾选前的回调，默认return true

        private disableItems: any[] = []; // 记录disable的条目，用于回勾父节点
        private flattenTree: any[] = []; // 拍平后的树

        @Watch('tree', {deep: true})
        updateFlattenTree () {
            let tree = cloneDeep(this.tree);

            this.flattenTree = this.flatten(tree, 'children', 1, {
                level: 0,
                visible: true,
                expand: true,
                children: this.tree
            });

            this.resetExpandStatus();

            // 拍平后，如果有设置默认数据，要设置check
            this.resetCheckedKeys();
        }

        @Watch('defaultCheckedKeys')
        updateDefaultCheck () {

            // 还没有树数据的时候，不用设置check
            if (!this.flattenTree.length) {
                return;
            }

            this.resetCheckedKeys();
            this.emitAllSelect();
        }

        mounted () {
            let vm = this;
            vm.debounceFilter = throttle(vm.filterTree, 300, {
                'leading': false, // 是否在每个等待时延的开始被调用
                'trailing': true // 是否在每个等待时延的结束被调用
            });
        }

        getRenderHtml (item: string) {
            return DOMPurify.sanitize(this.renderContent(item), DOMPURIFY_CONFIG);
        }

        getPaddingLeft (item: any) {
            if (item.level === 1) {
                return 0;
            }

            let paddingLeft = 24 * (item.level - 1); // 24 箭头
            if (this.showCheckbox) {
                paddingLeft += 14 * (item.level - 1); // 14 checkBox
            }

            // 如果当前节点是叶子节点，但兄弟节点有子集（为了对齐兄弟节点的箭头和复选框,或箭头和文字）
            let brotherHasChildren = item.parent.children.find(item => item.children && item.children.length);
            if (!item.children && brotherHasChildren) {
                paddingLeft += 6;
            }

            return paddingLeft + 'px';
        }

       // 发送已选数据
        emitAllSelect () {

            // 发送的是叶子节点
            let rs = this.flattenTree.filter(item => item.checked && (!item[this.props.children] || !item[this.props.children]));
            this.$emit('select-change', rs);
        }

        clearChecked () {
            this.defaultSelected = [];
            this.resetCheckedKeys();
        }

        // 判断是否为正常的id，数字中0也可以为id
        isNormalId (item: any) {
            return !!item || item === 0;
        }

        // 重置选中值
        resetCheckedKeys () {
            let vm = this,
                list = (this.defaultSelected || []).map(item => item[vm.idProperty] || item).filter(item => vm.isNormalId(item)),
                defaultList = (this.defaultCheckedKeys || []).filter(item => vm.isNormalId(item));

            if (!vm.flattenTree.length) {
                return;
            }

            for (let i in vm.flattenTree) {
                if (!vm.flattenTree.hasOwnProperty(i) || !vm.flattenTree[i][vm.idProperty]) {
                    continue;
                }
                let item = vm.flattenTree[i];

                let isDefault = defaultList.includes(item[vm.idProperty]);
                if (list.includes(item[vm.idProperty]) || isDefault) {
                    this.$set(item, 'checked', true);
                    isDefault && vm.$set(item, 'disabled', true);
                    !isDefault && vm.$set(item, 'disabled', false);
                    vm.toggleParentCheck(item);
                } else {
                    vm.$set(item, 'checked', false);
                    vm.$set(item, 'disabled', false);
                    vm.toggleParentCheck(item);
                }
            }
        }

         // 监听勾选的变化
        async toggleChecked (val: boolean, item: any) {

            if (!val) {
                let doDel = await this.beforeRemove();
                if (!doDel) {
                    item.checked = !val; // 如果不允许删除，将删除的值改回
                    return;
                }
            }

            this.$set(item, 'checked', val);
            this.$set(item, 'isIndeterminate', false);

            // 递归修改子节点状态
            this.disableItems = [];

            this.toggleChildrenStatus(item[this.props.children], val, 'checked');

            this.disableItems.forEach(item => {
                this.toggleParentCheck(item);
            });

            // 更新所有节点的状态
            if (item.level !== 1) {
                this.toggleParentCheck(item);
            }

            // 发送已选数据
            this.emitAllSelect();
        }

        filter (val: string) {
            this.debounceFilter(val);
        }

        filterTree (val: string) {

            if (!val) {
                this.resetExpandStatus();
                return;
            }

            this.flattenTree.forEach(item => {
                this.$set(item, 'visible', false);
                if (item[this.props.optionName].indexOf(val) > -1) {
                    this.$set(item, 'visible', true);
                    this.toggleParent2Expand(item); // 如果筛选出来的不是1级，要让其父级的状态也改成可视的，维持树形结构
                }
            });
        }

        // 重设展开状态
        resetExpandStatus () {
            this.flattenTree.forEach(item => {
                if (this.defaultExpandAll) {

                    // 全展示
                    item.expand = true;
                    this.$set(item, 'visible', true);

                    return;
                }

                if (this.defaultExpandedKeys.length) {

                    // 指定key展示
                    let isExpandItem = this.defaultExpandedKeys.includes(item[this.idProperty]) && item.children && item.children.length;
                    if (item.level === 1) {
                        item.expand = isExpandItem;
                        this.$set(item, 'visible', true);
                    } else if (isExpandItem) {
                        item.expand = true;
                        this.$set(item, 'visible', true);
                        this.toggleChildrenStatus(item[this.props.children], true); // 如果expand为true，子节点都要看得到
                        this.toggleParent2Expand(item);
                    } else {
                        item.expand = false;
                        let visible = (item.parent.expand && item.parent.visible) || false;
                        this.$set(item, 'visible', visible);
                    }

                    return;
                }


                // 展开到XX级，默认是展示到1级，即展开0级
                if (item.level <= this.defaultExpandedLevel) {
                    item.expand = true;
                    this.$set(item, 'visible', true);
                } else if (item.level === this.defaultExpandedLevel + 1) {
                    item.expand = false;
                    this.$set(item, 'visible', true);
                } else {
                    item.expand = false;
                    this.$set(item, 'visible', false);
                }
            });
        }

        // 拍平数组
        flatten (list: any[], childKey = 'children', level = 1, parent: any = null) {
            let vm = this,
                arr = [];

            list.forEach(item => {
                item.level = level;
                item.parent = parent;
                arr.push(item);
                if (item[childKey]) {
                    arr.push(
                        ...vm.flatten(
                            item[childKey],
                            childKey,
                            level + 1,
                            item
                        )
                    );
                }
            });
            return arr;
        };

        // 过滤条件，只展示打开的节点，即visible为true
        filterFn (arr: any[]) {
            return (arr || []).filter(
                item => item.visible
            );
        }

        updateView () {

            this.$refs.virtualList && this.$refs.virtualList.updateView();
        }

        // 展开树
        toggleExpand (item: any) {
            let isExpand = item.expand;
            if (isExpand) {
                this.collapse(item); // 折叠
            } else {
                this.expand(item); // 展开
            }
            this.updateView();
        }

        // 修正父节点的展开和可视状态，如某一节点勾选了（或筛选中了），父节点也要展开
        toggleParent2Expand (item: any) {
            let vm = this,
                curItem = item.parent;
            if (curItem) {
                if (curItem.expand && curItem.visible) { // 节省计算，如果父节点的状态已经是要设置的状态，说明父节点的父节点也已经是要设置的状态，所以不再递归
                    return;
                }
                curItem.expand = true;
                curItem.visible = true;
                vm.toggleParent2Expand(curItem);
            }
        }

        // 修正父节点的勾选状态，如某一节点勾选了，父节点要变成半选或者全选
        toggleParentCheck (item: any) {
            let curItem = item.parent;
            let {hasCheck, hasNoCheck} = this.recursionGetChildrenCheck({
                children: curItem[this.props.children],
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
        recursionGetChildrenCheck ({children, hasCheck, hasNoCheck}: { children: any[]; hasCheck: boolean; hasNoCheck: boolean }) {
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
        toggleChildrenStatus (children: any[], status: boolean, type = 'visible') {
            let isCheck = type === 'checked';
            (children || []).forEach(item => {
                if (!isCheck) {
                    this.$set(item, type, status);
                    if (item.expand) {
                        this.toggleChildrenStatus(item[this.props.children], status, type);
                    }
                } else {
                    if (item.disabled) {
                        if (status !== item.checked) {
                            this.disableItems.push(item);
                        }
                        return;
                    }
                    this.$set(item, type, status);
                    this.$set(item, 'isIndeterminate', false);
                    this.toggleChildrenStatus(item[this.props.children], status, type);
                }
            });
        }

        //展开节点
        expand (item: any) {
            item.expand = true;
            this.toggleChildrenStatus(item[this.props.children], true);
        }

        //折叠节点
        collapse (item: any) {
            item.expand = false;
            this.toggleChildrenStatus(item[this.props.children], false);
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
    .b-tree-node__wrap {
        display: flex;
        width: 100%;
        .b-tree-node_value {
            flex: 1;
            position: relative;
            top: 2px;
        }
        .b-tree__checkbox {
            margin-right: 8px !important;
        }
    }
    /deep/ .b-list__content {
        padding-left: 0;
    }
    .el-tree-node__expand-icon {
        padding: 6px;
    }
</style>
