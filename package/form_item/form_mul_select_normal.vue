<template>
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key} ${getTextModel ? '' : 'wti-untext-box'}`"
         class="form-item-box">
        <el-select style="width:100%"
                   v-model="val"
                   multiple
                   collapse-tags
                   :disabled="getDisabled"
                   :placeholder="getSelectPlaceholder(item)"
                   v-bind="bindOptions"
                   v-if="!getTextModel">
            <el-option v-for="option in item.options"
                       :key="option.value"
                       :label="option.label"
                       :value="option.value"/>
        </el-select>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle||{}" class="form-input-text">{{ textModelValue || '-' }}</div>
    </div>
</template>

<script>

    import FormMixin from './mixin';

    export default {
        name: 'FormMulSelectNormal',
        mixins: [ FormMixin ],
        computed: {
            textModelValue () {
                if (this.item.options) {
                    let val = '';
                    this.item.options.forEach(item => {
                        if (item.value === this.value) {
                            val = item.label;
                        }
                    });
                    return val;
                } else {
                    return '';
                }
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


    .form-item-box /deep/ .el-input {
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

        .el-input__inner:focus {
            border-color: #8D94A5;
        }
    }

    .el-select-dropdown__item.selected {
        color: #606266;
        font-weight: normal;
    }

</style>
