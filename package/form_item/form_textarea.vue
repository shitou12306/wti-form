<template>
    <!-- 普通输入框 -->
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key}`"
         class="form-input-box form-area-box">
        <el-input v-model="val"
                  :placeholder="getPlaceholder(item)"
                  :disabled="getDisabled"
                  @blur="e => onBlur(item, e)"
                  @focus="e => onFocus(item, e)"
                  type="textarea"
                  :autosize="item.autosize || false"
                  :rows="item.autosize ? '' : 4"
                  :resize="item.resize || 'none'"
                  v-bind="item"
                  v-if="!getTextModel"/>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle || {}" class="form-input-text">
            {{ val || '-' }}
        </div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    export default {
        name: 'FormInput',
        mixins: [ FormMixin ],
        data () {
            return {};
        },
        computed: {
            // 前置符号
            prepend () {
                // 兼容性处理
                if (this.item.prepend) {
                    return this.item.prepend;
                } else if (this.item.prependMsg) {
                    return this.item.prependMsg;
                } else {
                    return '';
                }
            },
            // 后置符号
            append () {
                // 兼容性处理
                if (this.item.append) {
                    return this.item.append;
                } else if (this.item.appendMsg) {
                    return this.item.appendMsg;
                } else {
                    return '';
                }
            },
        }
    };
</script>

<style scoped lang="less">
    .form-input-box /deep/ .el-input-group__prepend, .single-input .form-input-box /deep/ .el-input-group__append {
        padding: 0 10px;
    }

    .form-area-box {
        /deep/ .el-form-item__content {
            height: auto;
        }
    }


</style>
