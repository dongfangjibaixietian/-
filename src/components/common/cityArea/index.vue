<template>
<!-- 下面的样式语法表示 cascader-input 这个 class 存在与否将取决于数据 initValueFlag 的 truthiness，true则存在，flase则不存在 -->
    <el-cascader filterable :class="{'cascader-input' : initValueFlag}" :placeholder="initValue" v-model="value" :options="cascader_options" :props="cascader_props" @change="changeValue"></el-cascader>
</template>

<script>
// API
import { GetCity } from "@/api/common"
export default {
	name: '',
	data(){
		const _this = this;
		return {
			address: [],
			addressData: {},
			value: [],
			cascader_options: [],
			cascader_props: {
				lazy: true,
				lazyLoad (node, resolve) {
					const level = node.level;
					// 请求参数
					const requestData = {};
					// 声明自定义配置，通过json对象的形式简化代码
					const jsonType = {
						0: { type: "province" },
						1: { type: "city", code: "province" },
						2: { type: "area", code: "city" }
					}

					if(jsonType[level].code) { requestData[`${jsonType[level].code}_code`] = node.value }
					// type
					requestData.type = jsonType[level].type;

					//上面的写法用老式的方法可以这样写，之所以不需要做判断了，是因为jsonType[level]是动态改变的，随层数改变而改变
					// if (level = 0) {		
					// 	requestData.type = "province";
					// }
					// if (level = 1) {		
					// 	requestData.type = "city";
					// 	requestData.province_code = node.value;
					// }
					// if (level = 2) {		
					// 	requestData.type = "area";
					// 	requestData.city_code = node.value;
					// }

					// 省市区的接口
					GetCity(requestData).then(resonse => {
						const data = resonse.data.data;
						// 类型
						const type = jsonType[level].type.toUpperCase();
						// 自定义value、label
						// 循环遍历数组的方法，跟循环遍历json对象的方法不同
						data.forEach(item => {
							item.value = item[`${type}_CODE`];
							item.label = item[`${type}_NAME`];
							// 最后一层选择，控制到哪一层结束并且返回出去
							if(level === 2) { item.leaf = level >= 2; }
						})
						// 存储省市区数据
						_this.addressData[jsonType[level].type] = data;
						// 返回节点数据
						resolve(data);
					})
					// 获取address，此处有一个细节是_this的使用，
					// 像data里面调用method的方法可以直接用this，
					// method调用data里面的数据也可以直接用this，
					// 但是因为这是在第三方框架中写的，所以this指向的是第三方框架
					// 解决办法是先在data最外层绑定this为_this，就可以在第三方框架中调用_this使用method的方法了
					if(node.level !== 0) {
						_this.getAddress(node);
					}
				}
			},
			initValue: "请选择省市区",
			initValueFlag: false,
		}
	},
	methods: {
		/** 初始化默认值  */
		initDefault(value){
			if(value) {
				this.initValueFlag = true;
				this.initValue = value.split(",").join(" / ");
			}
		},
			// 此处要注意value是怎么来的，当一个组件模块用v-model指令绑定数据后，这个组件就自带value值，
			// 后续定义方法可以直接使用value值，所以此时的value是v-model中的value，
			// 由于数据是双向绑定的，即第三方组件会把省市区数据存到value变量上
		changeValue(value){
			// join方法使数组转为字符串，默认逗号连接。如果要用其他的连接，join("/")
			// 这句话翻译：把value赋值给cityAreaValue，并且把更新的cityAreaValue发射给父组件，
			// 一般$emit用法是发送事件父组件用@aaa=“aaa”接收，但是这里事件名已经是update了，所以父组件只需要:cityAreaValue.syn接收数据即可，不用接收事件
			// 此时父组件要加上:cityAreaValue.sync修饰符才能接收改变，从而实现数据的双向流通
			this.$emit("update:cityAreaValue", value.join());
			// 匹配最后一项，区县
			const lastCode = value[value.length - 1];
			// 将this.addressData.area过滤，过滤条件为item.value == lastCode)[0]，并且返回过滤结果
			const area = this.addressData.area.filter(item => item.value == lastCode)[0];
			// 意思是把area.label存储到address变量的第三项中
			this.address[2] = area.label;
			this.getAddress();
		},
		/** 获取中文地址 */
		getAddress(node){
			if(node) {
				const index = node.level - 1;
				this.address[index] = node.label;
			}
			// 为 true 时，执行地图交互
			if(this.mapLocation) {
				this.$emit("callback", {
					function: "setMapCenter",
					data: {
						address: this.address.join("")
					}
				});
			}
		},
		clear(){
			console.log(11111)
			this.value = "";
		}
	},
	components: {},
	props: {
		// 配置
		mapLocation: {
			type: Boolean,
			default: false
		},
		// 这是第一层，父组件传进来，emit是第二层，传出去
		cityAreaValue: {
			type: String,
			default: ""
		}
	}
   
}
</script>
<style lang='scss' scoped>
</style>
<!--
1、组件传入的属性用 props 接收；
2、this.$emit("update")返向修改，结合子组件属性的.sync修饰符。
-->