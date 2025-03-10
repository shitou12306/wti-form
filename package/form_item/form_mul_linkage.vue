<template>
    <!--  三级联动下拉框（通过数据字典获取选项）  -->
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key} ${getTextModel ? '' : 'wti-untext-box'}`"
         class="form-item-box">
        <el-row v-if="!getTextModel">
            <!-- 默认 3 级联动 -->
            <el-col v-for="i in (item.linkLevel || 3)" :key="i" :span="24 / (item.linkLevel || 3)">
                <el-select style="width:100%"
                           v-model="val[i-1]"
                           :disabled="getDisabled"
                           :placeholder="getSelectPlaceholder(item)"
                           v-bind="bindOptions"
                           @change="v => onChange(v, i)">
                    <el-option v-for="items in getOptions(i)"
                               :key="items[dynamicSelectOption.value]"
                               :label="items[dynamicSelectOption.label]"
                               :value="items[dynamicSelectOption.value]"/>
                </el-select>
            </el-col>
        </el-row>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle || {}" class="form-input-text">{{ textModelValue() }}</div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    // 多级联动下拉框
    export default {
        name: 'FormMulLinkage',
        mixins: [ FormMixin ],
        data () {
            return {
                hasLoaded: true
            };
        },
        watch: {
            'item': {
                handler () {
                    this.hasLoaded = false;
                },
                immediate: false
            },
            'val': {
                handler () {
                    this.hasLoaded = false;
                },
                immediate: false
            },
            'item.linkLevel': {
                handler () {
                    this.initVal();
                },
                immediate: true
            }
        },
        created () {
            this.loadDict(this.item.firstParentKey);
        },
        computed: {
            val: {
                get () {
                    return this.value;
                },
                set (v) {
                    this.$emit('input', v);
                    // 只有非子表单的情况下，才会冒泡上去数据变更
                    if (this.formItemType !== 'childForm') {
                        this.statusChangeFn.valueUpdateEvent({
                            [this.item.key]: v,
                        });
                    } else {
                        // 如果是子表单的话，执行内置的变更
                        this.childChangeData.valueUpdateEvent();
                    }
                }
            },
        },
        methods: {
            getTextModelValueList () {
                // 1、这里的文本值，是 val每个元素 翻译为 label 后的组合
                // 2、因为是多级联动，且是文本模式，而我们只有其 value 值
                // 3、所以我们需要先拿到字典里该项，将 value 翻译为 label 才可以。
                // 4、而为了拿到字典里该项，所以我们需要知道每一个值的父 key，然后去请求接口拿到该父 key 下的所有元素，
                //    再遍历这些元素，通过 value 匹配的方式，拿到该元素的 label
                // 5、具体来说，第一个值的父 key 是 this.item.firstParentKey；剩下每个的父 key，是他前一个值的 val
                // 而具体操作过程中，我们要先去判断这些是否已经缓存到了 this.dynamicDict 里，没有的话，我们再去请求接口拿值，可以提高效率

                // 创建父 key 列表
                const parentKeyList = [];
                if (!this.dynamicDict[this.item.firstParentKey] || this.dynamicDict[this.item.firstParentKey].length === 0) {
                    parentKeyList.push(this.item.firstParentKey);
                }
                if (this.val && this.val instanceof Array) {
                    this.val.forEach(v => {
                        // 有值，
                        if (v) {
                            // 且其不在 this.dynamicDict 里，或者 this.dynamicDict[v] 里面该值为空
                            // 才放到请求列表里
                            if (!this.dynamicDict[v] || this.dynamicDict[v].length === 0) {
                                parentKeyList.push(v);
                            }
                        }
                    });
                }
                this.hasLoaded = true;
                // 然后去调接口获取该 父 key 下面的所有值
                this.loadDict(parentKeyList);
            },

            // 获取文本值的时候
            textModelValue () {
                // 先创建一个数组，他的长度跟 this.item.linkLevel 一样
                const text = new Array(this.item.linkLevel).fill('');
                let needLoadDict = false;
                for (let i = 0; i < this.item.linkLevel; i++) {
                    // 拿到这个值
                    const v = this.val[i];
                    // 再去遍历 this.dynamicDict 中父 key 列表，去取值
                    let pKey = '';
                    if (i === 0) {
                        // 第一个值的父 key 取这个
                        pKey = this.item.firstParentKey;
                    } else {
                        // 后面值的父 key 取前一个的值
                        pKey = this.val[i - 1];
                    }
                    // 如果父 key 有值，则说明理论上可以从 this.dynamicDict[pKey] 里找到他的值
                    if (this.dynamicDict[pKey] && this.dynamicDict[pKey].length > 0) {
                        let t = '';
                        let isGet = false;
                        this.dynamicDict[pKey].forEach(item => {
                            if (item[this.dynamicSelectOption.value] === v) {
                                isGet = true;
                                t = item[this.dynamicSelectOption.label];
                            }
                        });
                        // 如果没找到 label，则该值取 对应的 value
                        if (!isGet) {
                            t = v;
                        }
                        text[i] = t;
                        continue;
                    } else {
                        // 如果父 key 没在 this.dynamicDict[pKey] 找到
                        // 此时说明该去请求一下接口，但是现在的值，应该暂时显示为 value 来替换 label
                        text[i] = v;
                        needLoadDict = true;
                    }
                }

                // 去调接口加载数据字典
                if (needLoadDict && !this.hasLoaded) {
                    this.getTextModelValueList();
                }
                // 但是现在还是需要返回值的，所以返回文字内容
                return text.join('') || '-';
            },
            // 根据 item.linkLevel 更新 val 长度
            initVal () {
                this.$emit('input', new Array(this.item.linkLevel || 3).fill(''));
            },
            // 获取下拉框列表
            getOptions (i) {
                if (String(i) === '1') {
                    // 第一个选项，下拉框内容的父 key 是 item.firstParentKey
                    return this.dynamicDict[this.item.firstParentKey] || [];
                } else {
                    // 后续的选项，下拉框的父 key ，是前一个的值
                    return this.dynamicDict[this.val[i - 2]] || [];
                }
            },

            // 异步加载字典
            // todo 这里的数据字典请求接口，应该最后合并到一起，由一个专门的数据字典请求管理器去请求，减低接口重复请求的情况
            loadDict (parentCode) {
                let payload = null;
                if (this.dynamicSelectOption.queryKey) {
                    if (parentCode instanceof Array) {
                        payload = {
                            [this.dynamicSelectOption.queryKey]: [ ...parentCode ]
                        };
                    } else {
                        payload = {
                            [this.dynamicSelectOption.queryKey]: [ parentCode ]
                        };
                    }
                } else {
                    if (parentCode instanceof Array) {
                        payload = [ ...parentCode ];
                    } else {
                        payload = [ parentCode ];
                    }
                }
                this.$set(this.dynamicDict, parentCode, []);
                // console.log('nul linkage load dict');
                // 否则，根据当前的值，去请求数据字典
                this.getSpecialAxios().post(this.dynamicSelectOption.dictUrl, payload).then(res => {
                    // 兼容性处理
                    let data;
                    if (res.request && res.headers) {
                        data = res.data;
                    } else {
                        data = res;
                    }
                    if (data.code === 200) {
                        if (data.data.length > 0) {
                            // 因为可能多个地方同时调这个接口的原因，为了避免重复将内容添加到里面，所以，
                            // 这里在赋值之前，需要先判断一下 parentCodeList 的每个值，其对应的 dynamicDict 里的哪一个数组，是否是空的
                            // 如果不是空的，则将其置为空数组
                            let list;
                            if (parentCode instanceof Array) {
                                list = [ ...parentCode ];
                            } else {
                                list = [ parentCode ];
                            }
                            list.forEach(pCode => {
                                if (!this.dynamicDict[pCode] || this.dynamicDict[pCode].length > 0) {
                                    this.$set(this.dynamicDict, pCode, []);
                                }
                            });

                            // 加载到结果
                            data.data.forEach(item => {
                                const pCode = item[this.dynamicSelectOption.parentKey];
                                this.dynamicDict[pCode].push(
                                    item
                                );
                            });

                            // 强制更新一遍组件的内容
                            this.$forceUpdate();
                        }
                    } else {
                        this.$message.error(data.msg);
                    }
                }).catch(e => {
                    console.log(e);
                    this.$message.error('数据字典加载错误，请刷新页面重试');
                });
            },

            // select 选择选项事件触发，
            // 值的索引从 0 开始，index 从 1 开始（即 val[0]对应 i=1）
            onChange (v, index) {
                // debugger;
                // 当 change 事件触发后，将其后的内容都置为空
                for (let i = index; i < this.item.linkLevel; i++) {
                    this.val[i] = '';
                }
                // 如果当前是最后一个，则无需请求
                if (index === this.item.linkLevel) {
                    return;
                }
                // 然后根据这个值，先判断当前数据字典里，是否有值
                if (this.dynamicDict[v] && this.dynamicDict[v].length > 0) {
                    // 有值，且长度不为0，则显然不需要再去请求数据字典了
                    return;
                } else {
                    this.loadDict(v);
                }
            },
        }
    };
</script>

<style scoped lang="less">

    .form-item-box {
        /deep/ .el-input {
            position: relative;
            width: 100%;
            height: 36px;

            .el-input__inner {
                position: absolute;
                width: 100%;
                height: 36px;
                line-height: 36px;
                padding-right: 10px;
                padding-left: 12px;
            }
        }

        /deep/ .el-row {
            height: 36px;

            .el-col {
                height: 36px;
            }
        }
    }

</style>
