<template>
    <!--  字典下拉框（指通过数据字典获取选项）  -->
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key} ${getTextModel ? '' : 'wti-untext-box'}`"
         class="form-item-box">
        <el-select style="width:100%"
                   v-model="val"
                   :disabled="getDisabled"
                   v-bind="bindOptions"
                   :placeholder="getPlaceholder(item)"
                   v-if="!getTextModel">
            <el-option v-for="option in dynamicDict[item.parentKey]"
                       :key="option[dynamicSelectOption.value]"
                       :label="option[dynamicSelectOption.label]"
                       :disabled="option[dynamicSelectOption.disabled]"
                       :value="option[dynamicSelectOption.value]"/>
        </el-select>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle || {}" class="form-input-text">{{ textModelValue || '-' }}</div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    export default {
        name: 'FormDynamicSelect',
        mixins: [ FormMixin ],
        computed: {
            textModelValue () {
                const content = this.dynamicDict[this.item.parentKey] && this.dynamicDict[this.item.parentKey].find((item) => {
                    return item[this.dynamicSelectOption.value] === this.val;
                });
                return content && content[this.dynamicSelectOption.label] || '';
            },

            val: {
                get () {
                    return this.value;
                },
                set (v) {
                    this.$emit('input', v);
                    this._valueLink(v);
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
    }
</style>
