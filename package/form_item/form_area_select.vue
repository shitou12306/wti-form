<template>
    <!-- 普通输入框 -->
    <div :style="item.style||{}"
         :class="`form-unqiue-${item.key} ${getTextModel ? '' : 'wti-untext-box'}`"
         class="form-input-box form-item-box">
        <!-- 这是三级联动选择框 -->
        <el-row v-if="!getTextModel">
            <el-col :span="8">
                <el-select v-model="val[0]"
                           class="select"
                           placeholder="请选择"
                           v-bind="bindOptions"
                           @change="v => onChange(v, '0')"
                           :disabled="getDisabled">
                    <el-option v-for="items in dynamicDict[item.firstParentKey || '10020']"
                               :key="items[dynamicSelectOption.value]"
                               :disabled="items[dynamicSelectOption.disabled]"
                               :label="items[dynamicSelectOption.label]"
                               :value="items[dynamicSelectOption.value]"/>
                </el-select>
            </el-col>
            <el-col :span="8">
                <el-select v-model="val[1]"
                           class="select"
                           placeholder="请选择"
                           @change="v => onChange(v, '1')"
                           :disabled="getDisabled">
                    <el-option v-for="items in cityList"
                               :key="items[dynamicSelectOption.value]"
                               :label="items[dynamicSelectOption.label]"
                               :value="items[dynamicSelectOption.value]"/>
                </el-select>
            </el-col>
            <el-col :span="8">
                <el-select v-model="val[2]"
                           class="select"
                           placeholder="请选择"
                           :disabled="getDisabled">
                    <el-option v-for="items in areaList"
                               :key="items[dynamicSelectOption.value]"
                               :label="items[dynamicSelectOption.label]"
                               :value="items[dynamicSelectOption.value]"/>
                </el-select>
            </el-col>
        </el-row>
        <div v-else :class="exposeSpecificClass(parentKey,childFormIndex,item.key)" :style="item.textStyle || {}" class="form-input-text">
            {{ areaText }}
        </div>
    </div>
</template>

<script>
    import FormMixin from './mixin';

    // 这个是已经耦合的，非通用组件
    export default {
        name: 'FormAreaSelect',
        mixins: [ FormMixin ],
        data () {
            return {
                prependMsg: '',
                appendMsg: ''
            };
        },
        computed: {
            cityList () {
                const secondParentKey = this.item.secondParentKey || '10021';
                if (this.val[0]) {
                    return this.dynamicDict[secondParentKey].filter(item => item.bparentCode === this.val[0]);
                }

                return [];
            },
            areaList () {
                const thirdParentKey = this.item.thirdParentKey || '10022';
                if (this.val[1]) {
                    return this.dynamicDict[thirdParentKey].filter(item => item.bparentCode === this.val[1]);
                }

                return [];
            },
            areaText () {
                const firstParentKey = this.item.firstParentKey || '10020';
                const secondParentKey = this.item.secondParentKey || '10021';
                const thirdParentKey = this.item.thirdParentKey || '10022';

                return `${this.getText(firstParentKey, this.val[0])}${this.getText(secondParentKey, this.val[1])}${this.getText(thirdParentKey, this.val[2])}`;
            }
        },
        methods: {
            onChange (v, index) {
                if (index === '0') {
                    this.val[1] = '';
                    this.val[2] = '';
                }
                if (index === '1') {
                    this.val[2] = '';
                }
            },
            getText (pCode, val) {
                if (this.val[0]) {
                    const t = this.dynamicDict[pCode].filter(item => item[this.dynamicSelectOption.value] === val);
                    if (t.length === 0) {
                        return '';
                    } else {
                        return t[0][this.dynamicSelectOption.label];
                    }
                } else {
                    return '';
                }
            },
        }
    };
</script>

<style scoped lang="less">


    .form-input-box /deep/ .el-input {
        position: relative;
        width: 100%;
        height: 36px;

        .el-input__inner {
            height: 36px;
            line-height: 36px;
        }
    }

    .form-input-text {
        position: relative;
        width: 100%;
        height: 36px;
        line-height: 36px;
        font-size: 14px;
        color: #12182A;
    }

    .select {
        width: 100%;
    }
</style>
