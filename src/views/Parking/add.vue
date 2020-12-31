<template>
    <div class="parking-add">
        <!-- :formHandler="form_handle按钮需要额外传，因为有事件等东西 -->
        <VueForm ref="vuForm" :formData="form_data" :formItme="form_item" :formHandler="form_handler">
            <template v-slot:city>
                <!-- 组件props通信的时候，:表示从左往右读，例如:cityAreaValue.sync="form_data.area"表示把CityArea组件中的cityAreaValue赋值给父组件的form_data.area -->
                <CityArea ref="cityArea" :mapLocation="true" :cityAreaValue.sync="form_data.area" @callback="callbackComponent" />
            </template>
            <!-- 插槽的使用详见tabledata组件，这里是一样的 -->
            <template v-slot:amap>
                <div class="address-map">
                    <AMap ref="amap" :options="option_map" @callback="callbackComponent" />
                </div>
            </template>
        </VueForm>
    </div>
</template>
<script>
// AMAP
import AMap from "../amap";
// 组件
import CityArea from "@c/common/cityArea";
import VueForm from "@c/form";
// API
import { ParkingAdd, ParkingDetailed, ParkingEdit } from "@/api/parking";
export default {
    name: "ParkingAdd",
    data() {
        let validatePass = (rule, value, callback) => {
            if(!value) {
                callback(new Error('请输入停车场名称'));
            }else{
                callback();
            }
        }
        let validateNumber = (rule, value, callback) => {
            const regNumber = /^[0-9]*$/;
            if(!value) {
                callback(new Error('请输入可停放车辆'));
            }else if(!regNumber.test(value)){
                callback(new Error('请输入数字'));
            }else{
                callback();
            }
        }
      return {
        // 表单数据配置
        form_data: {
            parkingName: "",
            area: "",
            address: "",
            type: "",
            carsNumber: "",
            status: "",
            lnglat: ""
        },
        // 表单配置
        // 何时用数组何时用对象，其实很简单，对象通常是指json对象，就理解为json对象只有一个大类，数组可以包括很多json对象
        // 此处是把一整个form_item数组传给了子组件Form，使用上很值得学习，要传的东西定义在data中
        form_item: [
            { 
                type: "Input", 
                label: "停车场名称", 
                placeholder: "请输入停车场名称",
                prop: "parkingName", 
                width: "200px",
                validator: [{ validator: validatePass, trigger: 'change' }]
            },
            { 
                type: "Slot", 
                slotName: "city", 
                prop:"area", 
                label: "区域"
            },
            { 
                type: "Input", 
                label: "详细地址", 
                placeholder: "请输入详细地址", 
                prop: "address",
                required: true
            },
            { 
                type: "Radio", 
                label: "类型", 
                prop: "type",
                options: this.$store.state.config.parking_type,
                required: true
            },
            { 
                type: "Input", label: "可停放车辆", placeholder: "请输入数字类型", prop: "carsNumber",
                validator: [{ validator: validateNumber, trigger: 'change' }]
            },
            { 
                type: "Radio", 
                label: "禁启用", 
                prop: "status",
                options: this.$store.state.config.radio_disabled
            },
            { type: "Slot", slotName: "amap", label: "位置" },
            { type: "Input", label: "经纬度", placeholder: "请输入详细地址", prop: "lnglat", disabled: true },
        ],
        // 传过去的按钮属性，注意此处handler: () => this.formValidate()是子组件的点击事件，
        // 这样的话子组件就可以直接通过 @click="item.handler()"调用这个事件了，这个事件要在父组件的methods中定义
        // 注意一定要是箭头函数
        form_handler: [
            { label: "确定", key: "submit",  type: "danger", handler: () => this.formValidate() },
            { label: "重置", key: "reset" },
        ],
        // 这种写法是通过路由获取网址上传过来的query的id值，即http://localhost:8080/#/ParkingAdd？id=70中的id
        id: this.$route.query.id,
        // 地图配置
        option_map: {
            mapLoad: true
        },
        status: this.$store.state.config.radio_disabled,
        type: this.$store.state.config.parking_type,
        // 按钮加载，此处的是element-ui按钮的一个功能，当:loading = button_loading="true"时。按钮为加载状态，此时用户无法再次点击
        button_loading: false
      }
    },
    components: { AMap, CityArea, VueForm },
    beforeMount(){},
    mounted(){},
    methods: {
        /** 提交表单 */
        formValidate(){
            this.$refs.vuForm.$refs.form.validate((valid) => {
                if (valid) {
                    this.id ? this.editParking() : this.addParking();
                } else {
                    console.log('error submit!!');
                    return false;
                }
            });
        },
        callbackComponent(params){
            // 此处的params.function为emit过来的setMapCenter，参数为emit过来的data
            if(params.function) { this[params.function](params.data);  }
        },
        /** 地图加载完成 */
        mapLoad(){
            this.getDetaile();
        },
        /** 新增停车场API */
        addParking(){
            this.button_loading = true;
            ParkingAdd(this.form_data).then(response => {
                this.$message({
                    type: "primary",
                    message: response.message
                })
                this.button_loading = false;
                this.reset("form");
            }).catch(error => {
                this.button_loading = false;
            })
        },
        /** 修改停车场API */
        editParking(){
            let requestData = JSON.parse(JSON.stringify(this.form_data));
            // 此处requestData中没有id属性，但只需要把data里面的id属性赋值给他他就会多一列id属性
            // 要注意与json对象的区别，json对象增加属性是form_data[key] = data[key]，
            requestData.id = this.id;
            this.button_loading = true;
            ParkingEdit(requestData).then(response => {
                this.$message({
                    type: "primary",
                    message: response.message
                })
                this.button_loading = false;
                this.$router.push({
                    name: "ParkingIndex"
                })
            }).catch(error => {
                this.button_loading = false;
            })
        },
        /** 获取详情 */
        getDetaile(){
            if(!this.id) { return false; }
            ParkingDetailed({ id: this.id }).then(response => {
                const data = response.data
                // 还原数据，即把data塞给，或者说赋值给form_data。key是json键值对中key：value中的key
                for(let key in data) {  // 接口请求出来的 
                // Object.keys(this.form_data)方法是拿到this.form_data这个json对象的所有key值，并且用数组存储起来
                // Object.keys(this.form_data).includes(key)意思是判断this.form_data这个json对象的所有key值是否包括后端返回数据里面的key值，
                // 如果包括则返回true，即执行下面的语句
                    if(Object.keys(this.form_data).includes(key)) { 
                        // 对于json对象的操作，只需要把data[key]赋值给另一个json对象的[key]，则这个键值对都会被塞到另一个json对象中
                        this.form_data[key] = data[key];
                    }
                }
                // 设置覆盖物
                const splitLnglat = data.lnglat.split(",");
                const lnglat = {
                    lng: splitLnglat[0],
                    lat: splitLnglat[1]
                }
                this.$refs.amap.setMarker(lnglat);
                // 初始化省市区
                this.$refs.cityArea.initDefault(data.region);
            })
        },
        /** 重置表单 */
        reset(formName){
            this.$refs[formName].resetFields();
            // 清除 cityAray 的值 
            this.$refs.cityArea.clear();
            // 清除地图覆盖物
            this.$refs.amap.clearMarker();
        },
        /** 获取经纬度 */
        getLnglat(data){
            this.form_data.lnglat = data.lnglat.value
        },
        setMapCenter(data){
            this.$refs.amap.setMapCenter(data.address);
        },
    }
}
</script>
<style lass="scss" scoped>
.address-map {
    width: 100%;
    height: 500px;
}
</style>