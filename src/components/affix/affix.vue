<template>
    <div>
        <div ref="point" :class="classes" :style="styles">
            <slot></slot>
        </div>
        <div v-show="slot" :style="slotStyle"></div>
    </div>
</template>
<script>
    /*
    *   组件思路：
    *   1. 给window添加scroll事件，实时监听，主要获取：
    *       a. 元素距离视口上部距离 this.$el.getBoundingclientRect() [有兼容性问题，主要是左上角的点位置问题]
    *       b. 页面滚动的距离 window.pageYOffset || document.documentElement.scrollTop
    *   
    *   2. 在滚动的过程中分两个情况 【其实是三个情况，等于的情况忽略】
    *       当滚动是从上往下时：
    *           a. 视口顶部将要遮住元素【即元素距离上部视口距离还未等于0】, 这个过程中有个等式一直成立，
    *              即 元素上部视口距离 + window滚动距离 = window还未滚动时，元素距离适口高度
    *               
    *              作者等式为 elOffset.top - this.offsetTop) > scrollTop
    *              即 （滚动条的滚动距离 + 此时元素距离视口上部距离 - 固定时距离视口高度【此时姑且为0】）- 滚动条的滚动距离 > 0 
    *              其实最后实质就是： 元素距离视口上部的距离 < 0 【使用数学去减一下上边作者的思路】时，进行触发吸顶操作
    *       
    *           b. 然后此时继续往下滚，元素一直吸附在视口顶部
    *              吸顶操作，即 给 元素的style对象设置值： 
    *                           position => fixed; top => offsetTop[此时姑且为0]； width => this.$el.offsetWidth;
    *   
    *       当滚动是从下往上时：
    *           a. 视口顶部将要即将露出元素原来的位置时，
    *              
    *              作者等式为 elOffset.top - this.offsetTop) > scrollTop
    *              即 （滚动条的滚动距离 + 此时元素距离视口上部距离 - 固定时距离视口高度【此时姑且为0】）- 滚动条的滚动距离 < 0 
    *              其实最后实质就是： 元素距离视口上部的距离 > 0 【使用数学去减一下上边作者的思路】时，进行取消吸顶操作
    *               
    *              取消吸顶操作： 元素的 style 属性值清空即可
    */
    import { on, off } from '../../utils/dom';
    const prefixCls = 'ivu-affix';

    function getScroll(target, top) {
        const prop = top ? 'pageYOffset' : 'pageXOffset';
        const method = top ? 'scrollTop' : 'scrollLeft';

        let ret = target[prop];

        if (typeof ret !== 'number') {
            ret = window.document.documentElement[method];
        }

        return ret;
    }

    function getOffset(element) {
        const rect = element.getBoundingClientRect();

        const scrollTop = getScroll(window, true);
        const scrollLeft = getScroll(window);

        const docEl = window.document.body;
        const clientTop = docEl.clientTop || 0;
        const clientLeft = docEl.clientLeft || 0;

        return {
            top: rect.top + scrollTop - clientTop,
            left: rect.left + scrollLeft - clientLeft
        };
    }

    export default {
        name: 'Affix',
        props: {
            offsetTop: {
                type: Number,
                default: 0
            },
            offsetBottom: {
                type: Number
            }
        },
        data () {
            return {
                affix: false,
                styles: {},
                slot: false,
                slotStyle: {}
            };
        },
        computed: {
            offsetType () {
                let type = 'top';
                if (this.offsetBottom >= 0) {
                    type = 'bottom';
                }

                return type;
            },
            classes () {
                return [
                    {
                        [`${prefixCls}`]: this.affix
                    }
                ];
            }
        },
        mounted () {
//            window.addEventListener('scroll', this.handleScroll, false);
//            window.addEventListener('resize', this.handleScroll, false);
            on(window, 'scroll', this.handleScroll);
            on(window, 'resize', this.handleScroll);
        },
        beforeDestroy () {
//            window.removeEventListener('scroll', this.handleScroll, false);
//            window.removeEventListener('resize', this.handleScroll, false);
            off(window, 'scroll', this.handleScroll);
            off(window, 'resize', this.handleScroll);
        },
        methods: {
            handleScroll () {
                const affix = this.affix;
                const scrollTop = getScroll(window, true);
                const elOffset = getOffset(this.$el);
                const windowHeight = window.innerHeight;
                const elHeight = this.$el.getElementsByTagName('div')[0].offsetHeight;

                // Fixed Top
                if ((elOffset.top - this.offsetTop) < scrollTop && this.offsetType == 'top' && !affix) {
                    this.affix = true;
                    this.slotStyle = {
                        width: this.$refs.point.clientWidth + 'px',
                        height: this.$refs.point.clientHeight + 'px'
                    };
                    this.slot = true;
                    this.styles = {
                        top: `${this.offsetTop}px`,
                        left: `${elOffset.left}px`,
                        width: `${this.$el.offsetWidth}px`
                    };

                    this.$emit('on-change', true);
                } else if ((elOffset.top - this.offsetTop) > scrollTop && this.offsetType == 'top' && affix) {
                    this.slot = false;
                    this.slotStyle = {};
                    this.affix = false;
                    this.styles = null;

                    this.$emit('on-change', false);
                }

                // Fixed Bottom
                if ((elOffset.top + this.offsetBottom + elHeight) > (scrollTop + windowHeight) && this.offsetType == 'bottom' && !affix) {
                    this.affix = true;
                    this.styles = {
                        bottom: `${this.offsetBottom}px`,
                        left: `${elOffset.left}px`,
                        width: `${this.$el.offsetWidth}px`
                    };

                    this.$emit('on-change', true);
                } else if ((elOffset.top + this.offsetBottom + elHeight) < (scrollTop + windowHeight) && this.offsetType == 'bottom' && affix) {
                    this.affix = false;
                    this.styles = null;

                    this.$emit('on-change', false);
                }
            }
        }
    };
</script>
