<template>
    <!-- 数字输入框 -->
    <div :style="item.style||{}"
         :class="getClass()"
         class="form-item-box">
        <template v-if="!getTextModel">
            <el-input v-model.trim="tempVal"
                      :placeholder="getPlaceholder(item)"
                      :disabled="getDisabled"
                      class="input-wr"
                      type="number"
                      @keydown.native="onKeydown($event)"
                      @blur="e => onBlur(item, e)"
                      @focus="e => onFocus(item, e)"
                      v-bind="bindOptions"
                      :clearable="true">
                <template slot="append">{{ append }}</template>
            </el-input>
            <el-input v-model.trim="dealInputValue"
                      :placeholder="getPlaceholder(item)"
                      :disabled="getDisabled"
                      class="input-readonly"
                      type="input"
                      v-bind="bindOptions"
                      :clearable="true">
                <template slot="append">{{ append }}</template>
            </el-input>
        </template>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle||{}" class="form-input-text">
            {{ dealInputValue || '-' }}
            {{ append }}
        </div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    export default {
        name: 'FormMoneyInput',
        mixins: [ FormMixin ],
        data () {
            return {
                readonly: true,
                // 在编辑模式下，使用这个 tempVal 替代真实的 val
                // 在触发 blur 事件时，用这个更新原来的 val
                tempVal: '',
            };
        },
        computed: {
            // 千分位的数字
            dealInputValue () {
                if (this.value === '') {
                    return '';
                }
                let n = String(this.value);
                // 先转，再处理。如果是万元，先转 把 10000 转为 1
                if (this.append === '万元') {
                    n = this.turnWanToYuan(n);
                    n = this.throwPointRightZero(n);
                }
                const res = n.toString().replace(/\d+/, (n) => {
                    return n.replace(/(\d)(?=(\d{3})+$)/g, ($1) => {
                        return $1 + ',';
                    });
                });
                return res;
            },
            // 后置符号
            append () {
                // 兼容性处理（默认为元）
                if (this.item.append) {
                    return this.item.append;
                } else if (this.item.suffixMsg) {
                    return this.item.suffixMsg;
                } else {
                    return '元';
                }
            },
            val: {
                // 因为单位可能是元，也可能是万元
                // 元可以直接显示，但是万元需要特殊处理后获取
                get () {
                    if (this.value === '') {
                        return '';
                    }
                    if (this.append === '万元') {
                        const v = String(this.value);
                        const wanyuan = this.turnWanToYuan(v);
                        return this.throwPointRightZero(wanyuan);
                    } else {
                        return this.value;
                    }
                },
                set (v) {
                    if (v === '') {
                        this.$emit('input', v);
                        this.statusChangeFn.valueUpdateEvent({
                            [this.item.key]: v,
                        });
                        return;
                    }
                    // 设置时，如果是万元，需要乘以 10000 再处理。如果不是，那么直接处理
                    const newVal = Number(v);
                    let n = String(newVal);

                    // 假如禁止输入负数，那么小于 0 则自动变为 0
                    if (this.item.positive && n && Number(n) < 0) {
                        n = '0';
                    }
                    if (this.append === '万元') {
                        // 乘以 1w，确保是精确结果
                        n = this.setNumberMultiplyTenThousands(n);
                    }
                    this.$emit('input', n);

                    // 只有非子表单的情况下，才会冒泡上去数据变更
                    if (this.formItemType !== 'childForm') {
                        this.statusChangeFn.valueUpdateEvent({
                            [this.item.key]: n,
                        });
                    } else {
                        // 如果是子表单的话，执行内置的变更
                        this.childChangeData.valueUpdateEvent();
                    }
                }
            }
        },
        methods: {
            // 将单位为元的数字，转为单位为万元的数字。例如 12000->1.2
            // 相当于除以 10000
            turnWanToYuan (v) {
                // 0 或者空，则返回自己
                if (v === '0' || v === '') {
                    return v;
                }
                // 1. 判断有没有小数点
                const pointIndex = v.indexOf('.');
                if (pointIndex > -1) {
                    // 有小数点
                    // 判断小数点左边是否小于 5 位。如果小于，说明总数小于万，需要补零
                    const pointList = v.split('.');
                    if (pointList[0].length < 5) {
                        // 需要补零，因为显示的单位是万
                        let s = '0.';
                        for (let i = 0; i < 5 - 1 - pointList[0].length; i++) {
                            s += '0';
                        }
                        s += pointList[0];
                        // 再拼接原来小数点右边的
                        s += pointList[1];

                        return s;
                    } else {
                        // 不需要补零
                        // 把小数点位置往左移动四位
                        const s = pointList[0].slice(0, -4) + '.' + pointList[0].slice(-4) + pointList[1];
                        return s;
                    }
                } else {
                    // 无小数点
                    if (v.length < 5) {
                        // 需要左侧补零，因为显示的单位是万
                        let s = '0.';
                        for (let i = 0; i < 5 - 1 - v.length; i++) {
                            s += '0';
                        }
                        s = s + v;
                        return s;
                    } else {
                        // 不需要补零
                        // 把小数点位置往左移动四位
                        const s = v.slice(0, -4) + '.' + v.slice(-4);
                        return s;
                    }
                }
            },

            // 将数字字符串乘以 1w
            // 因为有些数字乘以 1w 得出的数字并不准确。比如 1.12 * 10000 = 11200.000000000002
            setNumberMultiplyTenThousands (v) {
                const n = String(v);
                // 1、判断有没有小数点，没有小数点直接后面加 4 个 0
                if (v.indexOf('.') === -1) {
                    if (n !== '0') {
                        return `${n}0000`;
                    } else {
                        return '0';
                    }
                } else {
                    // debugger;
                    // 2. 有小数点的，判断小数点后位数是否 > 4，
                    // 2.1 小数点右边如果 <= 4（比如 1.12），那么干掉小数点，并且在数字最右边补零，1.12 -> 11200
                    const pointList = v.split('.');
                    if (pointList[1].length <= 4) {
                        // 需要右侧补零
                        let s = pointList[1];
                        for (let i = 0; i < 4 - pointList[1].length; i++) {
                            s += '0';
                        }
                        // 再拼接左右
                        s = pointList[0] + s;
                        return this.throwPointLeftZero(s);
                    } else {
                        // 2.2 小数点右边如果大于等于 4，则把小数点往右移动 4 位即可
                        const s = pointList[0] + pointList[1].slice(0, 4) + '.' + pointList[1].slice(4);
                        return this.throwPointLeftZero(s);
                    }
                }
            },


            getClass () {
                const c1 = `form-unqiue-${this.item.key} ${this.getTextModel ? '' : 'wti-untext-box'}`;
                const c2 = this.readonly ? 'is-readonly' : 'is-wr';
                return {
                    [c1]: true,
                    [c2]: true
                };
            },
            onKeydown (e) {
                // 假如禁止输入负数，那么干掉负号
                if (this.item.positive) {
                    if (e.key === '-' || e.code === 'Minus') {
                        e.preventDefault();
                        return;
                    }
                }
            },
            onBlur () {
                this.readonly = true;
                if (this.tempVal === '') {
                    this.$emit('input', this.tempVal);

                    this.statusChangeFn.valueUpdateEvent({
                        [this.item.key]: this.tempVal,
                    });
                    return;
                }

                let newVal = this.tempVal;

                // 如果需要补零
                if (this.item.zeroPadding && this.item.zeroPadding > 0) {
                    const l = String(newVal).split('.');
                    // 如果没有小数点（说明只有整数位），自动补零
                    if (l.length === 1) {
                        // 判断 l[0].length 是否为 0，为 0 则说明没填，啥事都不做
                        if (l[0].length === 0) {
                            newVal = '';
                        } else {
                            // 自动补零
                            newVal += '.' + '0'.padEnd(this.item.zeroPadding, '0');
                        }
                    } else {
                        // 此时说明有小数点，那么小数位数多，则去掉多余的。位数小，则补零
                        const currentLength = l[1].length;
                        // 小数位数少，则补零
                        if (currentLength < this.item.zeroPadding) {
                            newVal = l[0] + '.' + l[1].padEnd(this.item.zeroPadding, '0');
                        }
                        // 如果大于
                        if (currentLength > this.item.zeroPadding) {
                            newVal = String(l[0]) + '.' + l[1].slice(0, this.item.zeroPadding);
                        }
                    }
                }
                newVal = String(newVal);
                newVal = newVal.split('.').map((s, index) => {
                    if (index !== 0) {
                        return s;
                    }
                    // 通过正则，匹配首位开始的所有 0（用于去除整数部分开始的 0）
                    const newS = s.replace(/$0+/g, '');
                    // 如果只有 0，那么最后返回 0，确保小数点前有数字
                    if (newS.length === 0) {
                        return '0';
                    } else {
                        return newS;
                    }
                }).join('.');

                // 假如禁止输入负数，那么小于 0 则自动变为 0
                if (this.item.positive && newVal < 0) {
                    newVal = 0;
                }

                this.tempVal = this.throwPointLeftZero(this.tempVal);
                this.tempVal = this.throwPointRightZero(this.tempVal);
                this.val = newVal;
            },
            onFocus () {
                const newValue = this.value;
                if (this.append === '万元') {
                    let n = this.turnWanToYuan(String(newValue));
                    n = this.throwPointRightZero(n);
                    this.tempVal = n;
                } else {
                    this.tempVal = String(newValue);
                }
                this.readonly = false;
            },
        },
    };
</script>

<style scoped lang="less">

    .form-item-box /deep/ .el-input__inner {
        height: 36px;
        line-height: 1px !important;
    }


    .input-wr {
        z-index: 100;
    }

    .input-readonly {
        position: absolute;
        left: 0;
        top: 0;
    }

    .is-wr {
        .input-wr {
            opacity: 1;
        }

        .input-readonly {
            display: none;
        }
    }

    .is-readonly {
        .input-wr {
            opacity: 0;
        }

        .input-readonly {
            //display: none;
        }
    }

</style>
