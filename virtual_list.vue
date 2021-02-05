<template>
    <div class="b-list"
         ref="scroller"
         :style="{ height: option.height + 'px' }"
         @scroll="handleScroll">
        <div class="b-list__phantom" :style="{ height: `${contentHeight}px` }"></div>
        <div v-if="visibleData.length"
             class="b-list__content" :style="{ transform: `translateY(${offset}px)` }">
            <div v-for="item in visibleData"
                 :key="item[idProperty]"
                 class="b-list__list-view"
                 :style="{ height: option.itemHeight + 'px'}">
                <slot v-bind:item="item"></slot>
            </div>
        </div>
        <div v-else class="no-data__text">无数据</div>
    </div>
</template>


<script lang="ts">

    /**
     * @description: 虚拟列表
     * @author: cxs
     * @date: 2021/01/25
     */

    import {Component, Vue, Prop, Watch} from 'vue-property-decorator';

    @Component
    export default class VirtualList extends Vue {
        @Prop({
            default: () => {
                return [];
            }
        }) list!: any[];
        @Prop({default: null}) filterFn!: Function;
        @Prop({default: 20}) timeout!: number;
        @Prop({default: 'id'}) idProperty!: string;
        @Prop({
            default: () => {
                return {height: 500, itemHeight: 25};
            }
        }) option!: object;

        @Watch('list')
        watchList () {
            this.updateView();
        }


        private lastTime = 0; // 上次滚动的时间
        private offset = 0; // translateY偏移量
        private contentHeight = 0; // 所有展开的节点的总高度
        private visibleData = []; // 可见的数据


        // 可见区域的条数
        get visibleCount () {
            return Math.floor(this.option.height / this.option.itemHeight);
        }

        updateView () {
            this.getContentHeight();
            this.handleScroll();
        }

        // 滚动时，触发可视数据更新
        handleScroll () {
            let currentTime = +new Date();
            if (currentTime - this.lastTime > this.timeout) {
                this.updateVisibleData(this.$refs.scroller.scrollTop);
                this.lastTime = currentTime;
            }
        }

        updateVisibleData (scrollTop = 0) {
            let start =
                Math.floor(scrollTop / this.option.itemHeight) -
                Math.floor(this.visibleCount / 2);
            start = start < 0 ? 0 : start;
            let end = start + this.visibleCount * 2; // 预备可是区域数量的2倍，上下滑动时数据衔接不会出现空白
            let allVisibleData = this.list;
            if (this.filterFn) {
                allVisibleData = this.filterFn(this.list || []);
            }
            this.visibleData = allVisibleData.slice(start, end);
            this.offset = start * this.option.itemHeight;
        }

        getContentHeight () {
            let visibleListLen = (this.list || []).filter(item => item.visible);
            this.contentHeight = visibleListLen * this.option.itemHeight;
        }
    }
</script>

<style lang="less" scoped>
    .no-data__text {
        font-size: 12px;
        text-align: center;
        line-height: 32px;
        color: #999;
    }

    .b-list {
        position: relative;
        overflow-y: scroll;

        &__phantom {
            position: absolute;
            left: 0;
            top: 0;
            right: 0;
            z-index: -1;
        }

        &__content {
            position: absolute;
            left: 0;
            right: 0;
            top: 0;
            min-height: 100px;
            padding-left: 8px;
        }

        &__list-view {
            display: flex;
            align-items: center;
            cursor: pointer;
            font-size: 12px;
        }

    }
</style>
