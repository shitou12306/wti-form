<template>
    <!-- 输入搜索框 -->
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key} ${getTextModel ? '' : 'wti-untext-box'}`"
         class="form-item-box">
        <el-autocomplete v-model.trim="val"
                         :placeholder="getPlaceholder(item)"
                         :disabled="getDisabled"
                         class="auto-complte-input"
                         @blur="e => onBlur(item, e)"
                         @focus="e => onFocus(item, e)"
                         :clearable="true"
                         value-key="value"
                         :fetch-suggestions="querySearchAsync"
                         @select="handleSelect"
                         v-bind="bindOptions"
                         v-if="!getTextModel"/>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle||{}" class="form-input-text">{{ val || '-' }}</div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    export default {
        name: 'FormAutoComplete',
        mixins: [ FormMixin ],
        methods: {
            querySearchAsync (queryString, cb) {
                // 如果自定义请求函数，则使用自定义请求函数
                if (this.item.querySearchAsync) {
                    this.item.querySearchAsync(queryString, cb, this);
                    return;
                }

                const payload = {
                    [this.item.searchKey]: queryString
                };
                if (this.item.otherSearchKeys && this.item.otherSearchKeys.length > 0) {
                    this.item.otherSearchKeys.forEach(obj => {
                        payload[obj.key] = obj.value;
                    });
                }

                this.getSpecialAxios().post(this.item.searchUrl, payload).then(res => {
                    // 兼容性处理
                    let data;
                    if (res.request && res.headers) {
                        data = res.data;
                    } else {
                        data = res;
                    }
                    if (data.code === 200) {
                        if (data.data) {
                            const d = data.data.map(item => {
                                if (this.item.mainShowKey) {
                                    return {
                                        value: item[this.item.mainShowKey],
                                        item
                                    };
                                } else {
                                    return {
                                        value: item[this.item.autoCompleteKeys[0]],
                                        item
                                    };
                                }
                            });
                            cb(d);
                            if (this.item.fetchSuggestions) {
                                this.item.fetchSuggestions(d, this.randomId);
                            }
                        } else {
                            cb([]);
                            if (this.item.fetchSuggestions) {
                                this.item.fetchSuggestions([], this.randomId);
                            }
                            this.$message.error('无匹配数据');
                        }
                    } else {
                        cb([]);
                        if (this.item.fetchSuggestions) {
                            this.item.fetchSuggestions([], this.randomId);
                        }
                        this.$message.error(data.msg || '无匹配数据');
                    }
                }).catch(() => {
                    this.$message.error('服务器错误');
                });
            },
            handleSelect (selectedItem) {
                // console.log('selectedItem', selectedItem);
                const payload = {};
                // 将需要更新的数据的值，添加到 payload 里
                this.item.autoCompleteKeys.forEach(key => {
                    payload[key] = selectedItem.item[key];
                });
                // 再调用方法，推到 wti_form 这个组件中
                this.statusChangeFn.updateFormData(payload);

                if (this.item.onSelect) {
                    this.item.onSelect(selectedItem, this.randomId);
                }
            }
        }
    };
</script>

<style scoped lang="less">

    .auto-complte-input {
        width: 100%;
    }

    .form-item-box {

        /deep/ .el-input {
            position: relative;
            width: 100%;
            height: 36px;

            .el-input__inner {
                vertical-align: top;
                height: 36px;
                line-height: 36px;
            }
        }
    }
</style>
