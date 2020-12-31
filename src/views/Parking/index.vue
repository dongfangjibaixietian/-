<template>
    <div>
        <div class="filter-form">
            <el-row>
                <el-col :span="21">
                    <el-form :inline="true" :model="form" class="demo-form-inline">
                        <el-form-item label="区域">
                           <CityArea ref="cityArea" :cityAreaValue.sync="form.area" @callback="callbackComponent" />
                        </el-form-item>
                        <el-form-item label="类型">
                            <el-select v-model="form.type" placeholder="停车场" class="width-120">
                                <el-option v-for="item in parking_type" :label="item.label" :value="item.value" :key="item.value"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item label="禁启用">
                            <el-select v-model="form.status" placeholder="请选择" class="width-120">
                                <el-option v-for="item in parking_status" :label="item.label" :value="item.value" :key="item.value"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item label="关键字">
                            <el-select v-model="search_key" placeholder="请选择" class="width-120">
                                <el-option label="停车场名称" value="parkingName"></el-option>
                                <el-option label="详细区域" value="address"></el-option>
                            </el-select>
                        </el-form-item>
                        <el-form-item >
                            <el-input v-model="keyword" placeholder="请输入关键字按Enter搜索"></el-input>
                        </el-form-item>
                        <el-form-item>
                            <el-button type="danger" @click="search">搜索</el-button>
                        </el-form-item>
                    </el-form>
                </el-col>
                <el-col :span="3">
                    <div class="pull-right">
                        <router-link to="/parkingAdd">
                            <el-button type="danger">新增停车场</el-button>
                        </router-link>
                    </div>
                </el-col>
            </el-row>
        </div>
        <TabalData ref="table" :config="table_config">
            <!--禁启用-->
            <!-- 在el-table-column中加入其它的el组件时，需要用template做数据的交换 -->
            <!-- 此处的原理代码可以写成下面所示 -->
            <!-- <el-table-column prop="disabled" label="禁启用">
                <template slot-scope="scope">
                    <el-switch 
                    v-model="scope.row.disabled" 关键在于此处，绑定scope插槽的row属性的disabled值
                    active-color="#13ce66" inactive-color="#ff4949"></el-switch>
                </template>
            </el-table-column> -->
            <template v-slot:status="slotData">
                <!-- 遇到:时。最好是从前面往后读。即把status赋值给slotData ，v-slot:status="slotData"，后面的slotData变量，代表的是子组件引用插槽时传入给父组件的数据-->
                <!-- 即子组件用:data="scope.row"返回数据。父组件用v-slot:status="slotData"接收数据 -->
                <!-- 使用v-slot插槽命令时，绑定的第一个东西就是插槽名字，这里是status插槽，这个插槽名字也跟下面定义的数据当中slotName要一致，才能在子组件中显示 -->
                <el-switch :disabled="slotData.data.id == switch_disabled" @change="switchChange(slotData.data)" v-model="slotData.data.status" active-color="#13ce66" inactive-color="#ff4949"> </el-switch>
            </template>
            <!--查看地图-->
            <template v-slot:lnglat="slotData">
                <el-button type="success" size="mini" @click="showMap(slotData.data)">查看地图</el-button>
            </template>
        </TabalData>
        <MapLocation :flagVisible.sync="map_show" :data="parking_data" />
    </div>
</template>
<script>
// 组件
import CityArea from "@c/common/cityArea";
import MapLocation from "@c/dialog/showMapLocation";
import TabalData from "@c/tableData";
// API
import { ParkingDelete, ParkingStatus } from "@/api/parking";
// common
import { address, parkingType } from "@/utils/common";
export default {
    name: "Parking",
    data(){
        return {
            // 表格配置
            table_config: {
                thead: [
                    { label: "停车场名称", prop: "parkingName" },
                    { 
                        label: "类型", 
                        prop: "type",
                        type: "function",
                        // 将table抽离成组件后在el中插入其他el属性的方法--回调
                        // 解决element-ui中表格内容不能插入其他的el组件，只能显示文字的问题
                        // 下面的方式是通过json对象的key拿值并且返回拿到的value值
                        callback: (row, prop) => parkingType(row[prop])
                        // 上面的回调函数可以这样写
                        // prop其实是函数传过来的prop，即prop: "type"中的type值
                        // callback: (row,prop) => {
                        //     const data = this.parkingType.filter(item => item.value == row[prop] );
                        //     if (data && data.length > 0) {
                        //         return data[0].label 把“室内”跟“室外”return出去就行了
                        //     } 
                        // }
                    },
                    { 
                        label: "区域",
                        prop: "address",
                        type: "function",
                        callback: (row, prop) => address(row[prop])
                    },
                    { label: "可停放车辆", prop: "carsNumber" },
                    { 
                        label: "禁启用",
                        prop: "status",
                        type: "slot",
                        slotName: "status"
                    },
                    { 
                        label: "查看位置",
                        prop: "lnglat",
                        type: "slot",
                        slotName: "lnglat"
                    },
                    { 
                        label: "操作",
                        type: "operation",
                        default: {
                            deleteButton: true,
                            editButton: true,
                            editButtonLink: "ParkingAdd"
                        }
                    }
                ],
                url: "parkingList",  // 真实URL请求地址
                data: {
                    pageSize: 10,
                    pageNumber: 1
                }
            },
            // 筛选
            form: {
                area: "",
                type: "",
                status: ""
            },
            switch_disabled: "",
            switch_flag: false,
            //
            search_key: "",
            keyword: "",
            // 禁启用，这里有一个知识点，如果没有引入store，就要用this.$store.state的方式
            // 如果引入了store文件夹，就可以直接用store.state拿值
            parking_status: this.$store.state.config.radio_disabled,
            // 停车场类型
            parking_type: this.$store.state.config.parking_type,
            // 地图显示 
            map_show: false,
            parking_data: {},
            table_loading: false,
            rowId: ""
        }
    },
    components: { CityArea, MapLocation, TabalData },
    methods: {
        callbackComponent(params){
            if(params.function) { this[params.function](params.data); }
        },
        /** search */
        search(e){
            console.log(e);
            const requestData = {
                pageSize: 10,
                pageNumber: 1
            }
            
            // 过滤筛选
            // JSON.stringify(this.form)把this.form转换为字符串，实现深度拷贝，自此filterData和this.form就是两个完全不同的东西，只是值相同，保存的位置不同了
            // JSON.parse再转换为json对象

            const filterData = JSON.parse(JSON.stringify(this.form));
            for(let key in filterData) {
                if(filterData[key]) {
                    // 要拿json对象键值对中的值通过filterData[key]的方式，key是键，filterData[key]是值，
                    // 把一个json对象的值塞到到另一个json对象末尾的方法
                    requestData[key] = filterData[key];
                }
            }
            // 关键字
            if(this.keyword && this.search_key) { requestData[this.search_key] = this.keyword; }
            // 调用子组件的方法
            this.$refs.table.requestData(requestData);
        },
        /** 禁启用 */
        switchChange(data){
            if(this.switch_flag) { return false; }
            // 这里的知识点是，定义方法时把参数一起传过来，然后再用requestData做一层封装，最后直接把requestData传给接口
            const requestData = {
                id: data.id,
                status: data.status
            }
            // this.switch_flag = true;
            this.switch_disabled = data.id; 
             //上面是第一种方式：组件本身的属性处理，
            //  这种方式的原理是，点击事件后把所点击行的id赋值给switch_disabled，而当switch_disabled=slotData.data.id时，触发组件的disabled属性
            ParkingStatus(requestData).then(response => {
                this.$message({
                    type: 'success',
                    message: response.message
                });
                this.switch_disabled = "";
                // this.switch_flag = false;
            }).catch(error => {
                this.switch_disabled = "";
                // this.switch_flag = false;
            })
        },
        /** 显示地图 */
        showMap(data){
            this.map_show = true;
            this.parking_data = data;
        }
    },
    // DOM元素渲染之前（生命周期）
    beforeMount(){},
    // DOM元素渲染完成（生命周期）
    mounted(){},
}

</script>
<style lass="scss" scoped>

</style>