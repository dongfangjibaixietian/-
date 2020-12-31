<template>
    <div class="amap-wrap">
        <el-amap vid="amapContainer" :events="events" class="amap-demo"></el-amap>
    </div>
</template>
<script>
import { AMapManager, lazyAMapApiLoaderInstance } from 'vue-amap';
// 方法
import { getLngLat } from "./common";
import { addressSetMapCenter } from "./location";
import { amapSetMarker, amapClearMarker } from "./marker";
export default {
    name: "Amap",
    data(){
        return {
            // 经纬度
            lnglat: {},
            map: null,
            zoom: 18,
            events: {}
        }
    },
    mounted(){
        // lazyAMapApiLoaderInstance 为了加载高德地图的API
        lazyAMapApiLoaderInstance.load().then(() => {
            this.mapCreate();
            this.map.on("click", (e) => {
                const lnglat = getLngLat(e);
                // 更新经纬度
                this.lnglat = lnglat;
                // 回调父组件方法，$emit用法是$emit("aaa",bbb),第一个aaa父组件用@接收，即@aaa,
                // 然后父组件在method中写aaa方法，即@aaa=“aaa”，aaa中的data即是传过去的bbb值，可以在其他地方绑定这个值
                // 下面这种写法要简单一点，但也需要在父组件中接收，即@callback="callbackComponent"，然后要使用参数里面的函数，
                // 还需在父组件的methods中定义传过去的getLnglat方法
                this.$emit("callback", {
                    function: "getLnglat",
                    data: {
                        lnglat
                    }
                });  // 子组件调父组件的方法
                // 设置点覆盖物
                this.setMarker();
            })
        });
    },
    methods: {
        // 创建地图
        mapCreate(params){
            this.map = new AMap.Map('amapContainer', {
                center: [116.404765, 39.918052],
                zoom: this.zoom, //初始化地图层级
            });
            this.map.on("complete", () => {
                this.mapLoad();
            });
        },
        /**
         * 地图加载完成
         */
        mapLoad(){
            if(this.options.mapLoad) {
                this.$emit('callback', {
                    function: "mapLoad"
                })
            }
        },
        // 销毁地图
        mapDestroy(){
            // 巧用&& 做判断，意思是如果有this.map，则执行this.map.destroy()
            this.map && this.map.destroy();
        },
        setMapCenter(value){
            addressSetMapCenter(value, this.map);
        },
        // 设置点覆盖物，下面这三行的意思是，如果有参数传过来就用参数，如果没有参数就用后面的
        setMarker(lnglat){
            amapSetMarker(lnglat || this.lnglat, this.map);
        },
        /** 清除点覆盖物 */
        clearMarker(){
            amapClearMarker(this.map);
        }
    },
    props: {
        options: {
            type: Object,
            default: () => {}
        }
    }
}
</script>
<style lang="scss">
.amap-wrap { height: 100%; }
</style>