<template>
    <el-form ref="form" :model="formData" :label-width="labelWidth">
        <!-- 此处 v-for="item in formItme" 循环的是整个数组，数组里面有对象 -->
        <el-form-item v-for="item in formItme" :key="item.prop" :label="item.label" :prop="item.prop" :rules="item.rules">
            <!-- Input-->
            <!-- v-for循环之后，form表单里具体有什么内容，即el-form-item，是根据 v-if="item.type ===  "来决定的 -->
            <!--  :style="{width: item.width}"改变el-input自带宽度的方法 -->
            <!-- :disabled="item.disabled"的意思是，如果item.disabled是true的话，就禁用input框 -->
            <el-input v-if="item.type === 'Input'" v-model.trim="formData[item.prop]" :placeholder="item.placeholder" :style="{width: item.width}" :disabled="item.disabled"></el-input>
            <!-- Select-->
            <el-select filterable v-if="item.type === 'Select'" :aaaa="item.options" v-model.trim="formData[item.prop]" :placeholder="item.placeholder" :style="{width: item.width}" :disabled="item.disabled">
                <el-option v-for="selectItem in item.options" :key="selectItem.value || selectItem[item.select_vlaue]" :value="selectItem.value || selectItem[item.select_vlaue]" :label="selectItem.label || selectItem[item.select_label]"></el-option>
            </el-select>
            <!-- 禁启用 -->
            <el-radio-group v-if="item.type === 'Disabled'" v-model="formData[item.prop]">
                <el-radio v-for="radio in radio_disabled" :label="radio.value" :key="radio.value">{{ radio.label }}</el-radio>
            </el-radio-group>
            <!-- slot -->
            <!-- form表单可以不用template包裹，与table不同 -->
            <slot v-if="item.type === 'Slot'" :name="item.slotName" />
            <!-- 省市区 -->
            <el-radio-group v-if="item.type === 'Radio'" v-model="formData[item.prop]">
                <el-radio v-for="radio in item.options" :label="radio.value" :key="radio.value">{{ radio.label }}</el-radio>
            </el-radio-group>
            <!-- 富文本编辑器 -->
            <template v-if="item.type === 'Wangeditor'">
                <Wangeditor :isClear="wangeditorClear" ref="wangeditor" :value="formData[item.prop]" :content.sync="formData[item.prop]" />
            </template>
            <!-- 文件上传 -->
            <template v-if="item.type === 'Upload'">
                <Upload :imgUrl="formData[item.prop]" :value.sync="formData[item.prop]" />
            </template>
        </el-form-item>
        <!-- 按钮 -->
        <el-form-item>
            <!-- 单数组也可以循环，循环后拿到单数组里面的json对象的键值对 -->
            <!-- item.handler && item.handler前面这个是判断是否有这个方法，有的话再执行这个方法 -->
            <!-- form表单组件通信间的方法调用问题，item.handler()实际上是调用的parking/add.vue里面的form_handler的handler方法 -->
            <el-button v-for="item in formHandler" :key="item.key" :type="item.type" @click="item.handler && item.handler()">
                {{ item.label }}
            </el-button>
        </el-form-item>
    </el-form>
</template>
<script>
// 省市区
import CityArea from "@c/common/cityArea";
// 富文本
import Wangeditor from "@c/common/wangeditor";
// upload
import Upload from "@c/upload";
export default {
    name: "Form",
    components: { CityArea, Wangeditor, Upload },
    props: {
        labelWidth: {
            type: String,
            default: "120px"
        },
        formData: {
            type: Object,
            default: () => {}
        },
        // 父组件中data传过来的formitem值
        formItme: {
            type: Array,
            // 接收类型为对象时default的写法
            default: () => []
        },
        // 父组件传过来按钮值
        formHandler: {
            type: Array,
            default: () => []
        }
    },
    data(){
        return {
            // 禁启用数据 
            radio_disabled: this.$store.state.config.radio_disabled,
            type_msg: {
                "Input": "请输入",
                "Radio": "请选择",
                "Select": "请选择"
            },
            // 清除富文本
            wangeditorClear: false  // true false
        }
    },
    methods: {
        /** 重置表单 */
        resetForm(){
            this.$refs.form.resetFields();
            // 清除富文本内容
            // 清楚富文本有两种方法，第一次是this.$refs.wangeditor直接调用wangeditor的方法
            if(this.$refs.wangeditor) { this.wangeditorClear = !this.wangeditorClear; }
        },
        initFormData(){
            // 下面数组的循环遍历操作
            // 如果是json的循环遍历，则代码为
            // for(let key in this.config) {
            //    if(Object.keys(this.table_config).includes(key)) {
            //     this.table_config[key] = this.config[key]
            //    }
            this.formItme.forEach(item => {
                // rules 规则
                if(item.required) { this.rules(item); }
                // 自定义检验规则
                if(item.validator) { item.rules = item.validator; }
            })
        },
        rules(item){
            const requiredRules = [{ required: true, message: item.required_msg || `${this.type_msg[item.type]}${item.label}`, trigger: 'change' }];
            // 其他的 rules 规则 
            if(item.rules && item.rules.length > 0) {
                item.rules = requiredRules.concat(item.rules);
            }else{
                item.rules = requiredRules;
            }
        }
    },
    watch: {
        formItme: {
            handler(newValue){
                this.initFormData()
            },
            immediate: true
        }
    }
}
</script>
<style lang="scss" scoped>

</style>