<template>
  <div class="home">
	  <div class="left_news_container">
		  <el-drawer
		    :visible.sync="drawer"
		    direction="ltr"
		    :before-close="handleClose"
			:modal="false"
			title="新闻列表">
		    <el-card class="news_list">
				<el-collapse v-model="activeName" accordion>
					<div v-for="(news,index) in news_array" class="text item">
						
						<el-collapse-item :title="news.title" :name="index">
						  <div><a :href="news.url">新闻链接</a></div>
						  <div>{{news.description}}</div>
						</el-collapse-item>
					</div>
				 
				</el-collapse>
		    </el-card>
		  </el-drawer>
	  </div>
	<div id='dialog_container'>
		<el-dialog
		  class="upload_dialog"
		  title="上传照片"
		  :visible.sync="dialogVisible"
		  width="30%">
		  <el-upload
		    class="upload"
			ref='upload'
			:on-change="onUpload"
			:auto-upload="false"
		    action=""
			:limit="1"
			drag
		    multiple>
		    <i class="el-icon-upload"></i>
		    <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
		    <div class="el-upload__tip" slot="tip">只能上传jpg/png文件，且不超过500kb</div>
		  </el-upload>
		  <span>请上传照片</span>
		  <span slot="footer" class="dialog-footer">
		    <el-button @click="dialogVisible = false">取 消</el-button>
		    <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
		  </span>
		</el-dialog>
	</div>
	<div id='user_container'>
		<el-dialog
		  class="upload_dialog"
		  title="冷链信息"
		  :visible.sync="userVisible"
		  width="40%">
		  <el-table
		        :data="tableData"
		        style="width: 100%" v-if="tableVisible">
		        <el-table-column
		          prop="date"
		          label="日期"
		  		  sortable
		          width="180">
		        </el-table-column>
		        <el-table-column
		          prop="name"
		          label="货物信息"
		          width="180">
		        </el-table-column>
		        <el-table-column
		          prop="address"
		          label="地址">
		        </el-table-column>
		      </el-table>
		  <span slot="footer" class="dialog-footer">
		    <el-button @click="userVisible = false">取 消</el-button>
		    <el-button type="primary" @click="userVisible = false">确 定</el-button>
		  </span>
		</el-dialog>
	</div>
	<div class="search_place">
	  <el-input
	    placeholder="请输入溯源码查询"
	    prefix-icon="el-icon-search"
	    v-model="search_place"
	  	id="search_bar" style="width: 60%;">
	   </el-input> <el-button @click="submit">提交</el-button>
	</div>
	<div id="scan_container">
		<el-button id="scan" icon="logo.png" circle style="width: 100px; height: 100px;" @click="scan_dialog">扫码</el-button>
	</div>
	
	<div style="width: 100%; height: 100%;" class="map_container" id="map_container">

    </div>
	<div class="login_button_container">
		<el-button type="primary" @click="login">管理员登录</el-button>
	</div>
  </div>
</template>



<script>
	
export default {
  data() {
    return {
		userVisible:false,
		tableVisible:false,
		UUID:'',
		ruleForm: {
		  name: '',
		  //num:'',
		  place:'',
		  time:'',
		},
		tableData: [],
		drawer:false,//左侧新闻
	  dialogVisible:false,
	  map:{},
      center: [116.473778, 39.990661],
      zoom: 4,
      pitch: 45,
      rotation: 0,
	  search_place:"",
	  //风险等级地点信息
	  risk_level_areas:[{longitude:111,latitude:35,riskLevel:100}],
	  //相关风险新闻
	  news_array:[]
    };
	
  },

  mounted:function(){
	console.log("mounted");
	var map = new AMap.Map('map_container', {
	    center:[105,37],
	    
	    viewMode:'2D',
	    labelzIndex:100,
	    zoom:4.5,
	    layers:[
	        new AMap.TileLayer({
	            zIndex:7
	        }),
	       
	    ]
	});
	this.map = map;
	console.log(this.map);
	/**var that=this;
	this.$http.get("adminUser/history",{params:{username:"admin"}}).then(function(response){
		var geocoder = new AMap.Geocoder({
		    city: '全国'
		  }) 
		  //console.log(response.data.data);
		for(var i=0;i<response.data.data.length;i++){
			var count=i;
			var input=response.data.data[response.data.data.length-1];
			that.getloc(input,geocoder,that.map);
			
			
		}
	},function(error){
		that.$message.error("提交失败！"+error);
	})**/
	this.getRiskLevelInfo(map);
  },

  //在页面加载后请求风险地区信息
  created:function() {
	console.log("created");
  },
  methods: {
	handleClose(done) {
	        this.$confirm('确认关闭？')
	          .then(_ => {
	            done();
	          })
	          .catch(_ => {});
	      },
	//登录
	// 登录
	login() {
		this.$router.push("/login");
	},
	//获取风险地区信息并绘制标记
	getRiskLevelInfo(map) {
		var that = this;
		// 发送请求，请求风险地区经纬度
		this.$http.get('RiskLevel').then(function(response){
			if(response.data.code === 200){
				that.$message.success("风险地区信息加载成功！");
				//加载用户信息
				that.risk_level_areas = response.data.data;
				console.log("risk_level_areas已更新");
				// 重新绘制标记
				that.drawRiskLevelMarker(map);
			}
			else {
				that.$message.error("风险地区信息加载失败！");
			}
		},function(error){
			that.$message.error("风险地区信息加载失败！");
			console.log(error);
		});
		
	},
	// 为地图绘制风险地区标记
	drawRiskLevelMarker(map) {
		var i = 0;
		map.clearMap();
		//刷新地图覆盖物
		console.log("正在绘制标记");
		console.log(this.risk_level_areas);
		var that = this;
		var onMarkerClick  =  function(e) {
				var id = e.target.getExtData();
				console.log(id);
		        that.drawer = true;
				var i = 0;
				// 发送新闻请求
				console.log("发送新闻请求");
				console.log(this.risk_level_areas);
				that.$http.get('news?highRiskAreaId='+id).then(function(response){
					
					if(response.data.code === 200){
						that.$message.success("风险地区新闻加载成功！");
						that.news_array = response.data.data;
						console.log(that.news_array);
					}
					else {
						that.$message.error("风险地区新闻加载失败！");
					}
				},function(error){
					that.$message.error("风险地区新闻加载失败！");
				});
		    } 
		for(i = 0; i < this.risk_level_areas.length; i++){
			// 创建一个 Marker 实例：
			var marker = new AMap.Marker({
			    position: new AMap.LngLat(this.risk_level_areas[i].longitude,this.risk_level_areas[i].latitude),   // 经纬度对象，也可以是经纬度构成的一维数组[116.39, 39.9]
				cursor: "pointer",
				labelzIndex:120,
				extData:this.risk_level_areas[i].id,
			});
			//marker.on('mouseover',click);
			//AMap.event.addListener(marker,"mouseover",onMarkerClick);
			// console.log("extData:"+marker.extData);
			// 将创建的点标记添加到已有的地图实例：
			marker.on('click',onMarkerClick);
			map.add(marker);
			this.drawBounds(map,this.risk_level_areas[i].adcode,null,[]);
		}
		console.log("绘制标记完成");
	},
	//根据id加载风险地区新闻
	loadNews(marker) {
		this.drawer = true;
		this.drawRiskAreaNews(marker.extData);
	},
	//根据id改善新闻请求并渲染新闻
	drawRiskAreaNews(id) {
		var i = 0;
		// 发送新闻请求
		console.log("发送新闻请求");
		var that = this;
		console.log(this.risk_level_areas);
		this.$http.get('news?highRiskAreaId='+id).then(function(response){
			
			if(response.data.code === 200){
				that.$message.success("风险地区新闻加载成功！");
				that.news_array = response.data.data;
				console.log(that.news_array);
			}
			else {
				that.$message.error("风险地区新闻加载失败！");
			}
		},function(error){
			that.$message.error("风险地区新闻加载失败！");
		});
		
	},
	//行政区划绘制
	drawBounds(map,adcode,district,polygons) {
	        //加载行政区划插件
	        if(!district){
	            //实例化DistrictSearch
	            var opts = {
	                subdistrict: 0,   //获取边界不需要返回下级行政区
	                extensions: 'all',  //返回行政区边界坐标组等具体信息
	                level: 'district'  //查询行政级别为 市
	            };
	            district = new AMap.DistrictSearch(opts);
	        }
	        //行政区查询
	        district.setLevel('city')
	        district.search(adcode, function(status, result) {
	            polygons = [];
	            var bounds = result.districtList[0].boundaries;
	            if (bounds) {
	                for (var i = 0, l = bounds.length; i < l; i++) {
	                    //生成行政区划polygon
	                    var polygon = new AMap.Polygon({
	                        strokeWeight: 1,
	                        path: bounds[i],
	                        fillOpacity: 0.4,
	                        fillColor: 'red',
	                        strokeColor: 'red'
	                    });
	                    polygons.push(polygon);
	                }
	            }
	            map.add(polygons)
	        });
	    },
		scan_dialog(){
			this.dialogVisible=true;
		},
		onUpload(file){
			const isIMAGE=(file.raw.type ==='image/jpeg' || file.raw.type ==='image/png');
			const isOversize=file.size/1024/1024<1;
			if(!isIMAGE){
				this.$message.error('图片格式错误');
				return false;
			}
			if(!isOversize){
				this.$message.error('图片过大');
				return false;
			}
			var reader=new FileReader();
			var dialog_transer=this;
			reader.readAsDataURL(file.raw);
			reader.onload=function(e){
				//信息加载
				var that=this;
				console.log(that.result);
				dialog_transer.$http.post("adminUser/getDetails",{"base64":that.result}).then(function(response){
					console.log(response.data.data);
					for(var i=0;i<response.data.data.placeList.length;i++){
						var newobj={
							date: response.data.data.placeList[i].time,
							name: response.data.data.description,
							address: response.data.data.placeList[i].place
						}
						dialog_transer.tableData.push(newobj);
					}
					dialog_transer.UUID=response.data.data.uuid;
					dialog_transer.ruleForm.name=response.data.data.description;
					dialog_transer.tableVisible=true;
					dialog_transer.dialogVisible=false;
					dialog_transer.userVisible=true;
					var that=this;
					var geocoder = new AMap.Geocoder({
					    city: '全国'
					  }) 
					dialog_transer.getloc(["天津市滨海新区","江苏省南京市栖霞区南京大学"],geocoder,dialog_transer.map);
				},function(error){
					that.$message.error("提交失败！");
				})
			}
		},
		submit(){
			var that=this;
			this.$http.get("adminUser/getDetailsByUUID",{params:{UUID:this.search_place}}).then(function(response){
				//that.UUID=response.data.data.uuid;
				for(var i=0;i<that.tableData.length;i++){
					that.tableData.pop();
				}
				for(var i=0;i<response.data.data.placeList.length;i++){
					var newobj={
						date: response.data.data.placeList[i].time,
						name: response.data.data.description,
						address: response.data.data.placeList[i].place
					}
					that.tableData.push(newobj);
				}
			    that.UUID=response.data.data.uuid;
				that.ruleForm.name=response.data.data.description;
				that.tableVisible=true;
				that.dialogVisible=false;
				that.userVisible=true;
			},function(error){
				that.$message.error("提交失败！"+error);
			})
		},
		getloc: async function(data,geocoder,map){
			var obj={
				isSafe:data.isSafe,
				path:new Array(),
			}
			var destll=new AMap.LngLat(116.368904,39.913423);
			var placell=new AMap.LngLat(110.387271,33.912501); 
			obj.path.push(destll);
			var dest_marker = new AMap.Marker({
			   position: destll,   // 经纬度对象，也可以是经纬度构成的一维数组[116.39, 39.9]
				cursor: "pointer",
				labelzIndex:120,
				map:map,
			});
			obj.path.push(placell)
			var dest_marker = new AMap.Marker({
			    position: placell,   // 经纬度对象，也可以是经纬度构成的一维数组[116.39, 39.9]
				  cursor: "pointer",
				  labelzIndex:120,
				  map:map,
			});
			var polyline = new AMap.Polyline({
			    path: obj.path,  
				map:map,
											  labelzIndex:130,
			    borderWeight: 2, // 线条宽度，默认为 1
			    strokeColor: '#aaff00', // 线条颜色
			    lineJoin: 'round' // 折线拐点连接处样式
			});
			if(obj.isSafe===false){
				polyline.setOptions({strokeColor:"#ff0000"})
			} 
			
		}
  },
  
};

</script>


<style scoped="scoped">
	#search_bar{
		float: left;
		
	}
	.home {
		margin: 0;
		padding: 0;
	}
	/* 左侧新闻栏 */
	.left_news_container {
		position: absolute;
		left: 0;
		top: 50%;
		z-index: 2;
	}
	/* 搜索框 */
	.search_place {
		float: left;
		width: 20%;
		position: absolute;
		left: 50%;
		top: 5px;
		z-index: 2;
		transform: translateX(-50%);
	}
	/* 风险地区列表 */
	.list_of_risk_area {
			position: absolute;
			left: 5px;
			top: 50%;
			transform: translateY(-50%);
			z-index: 2;
			width: 300px;
	}
	/* 地图容器 */
	.map_container {
		position: absolute;
		left: 0;
		top: 0;
	}
	/* 登录按钮 */
	.login_button_container {
		position: absolute;
		right: 5px;
		bottom: 5px;
		z-index: 2;
	}
	#scan_container{
		position: absolute;
		right: 1%;
		top: 5%;
		z-index: 2;
		width: 10%;
		height: 10%;
	}
	.upload /deep/ .el-upload-dragger{
		width: 100%;
		
	}				
					
</style>