<template>
  <div id="mapTrack">
    <el-card class="box-card" shadow="never" :body-style="{ padding: '0' }">
      <el-card class="track-detail" :body-style="{ 'padding': '7px 20px' }">
        <i class="el-icon-tickets" style="cursor: pointer;" :style="{ 'color': '#' + (isShowDetail ? '409eff' : '303133') }"
           @click="getTableData()" onselectstart="return false;">
          明细
        </i>
      </el-card>

      <div id="map-test" :style="{ 'height': 'calc(100vh - ' + (isShowDetail ? 297 : 57) + 'px)' }">
        <div id="loadingTip" v-if="beforeInit && initSimplifier">加载数据，请稍候...</div>
      </div>

      <!--选择时间-->
      <div class="guijiset form-table-search">
        <div class="rowset"><label>机械号码：</label><label>{{ vehicleNo }}</label></div>
        <div class="rowset">
          <label>开始时间：</label>
          <el-date-picker
            size="small"
            v-model="startTime"
            type="datetime"
            placeholder="请选择开始时间"
            align="right"
            value-format="yyyy-MM-dd HH:mm:ss"
            :picker-options="pickerOptions"
            @change="timeChangeHandle">
          </el-date-picker>
        </div>
        <div class="rowset">
          <label>结束时间：</label>
          <el-date-picker
            size="small"
            v-model="endTime"
            type="datetime"
            placeholder="请选择结束时间"
            align="right"
            value-format="yyyy-MM-dd HH:mm:ss"
            :picker-options="pickerOptions"
            @change="timeChangeHandle">
          </el-date-picker>
        </div>

        <div class="btn-set-true">
          <!-- 轨迹播放滑块 -->
          <div class="playBox clearBoth">
            <div class="playIcon fl" v-if="isPlay">
              <el-button type="text" v-if="palyStayus === 0" icon="el-icon-video-play" value="开始动画" @click="navgControl('start')" :disabled="beforeInit"></el-button>
              <el-button type="text" v-if="palyStayus === 1" icon="el-icon-video-play" value="继续动画" @click="navgControl('resume')" :disabled="beforeInit"></el-button>
            </div>
            <div class="playIcon fl" v-if="!isPlay">
              <el-button type="text" icon="el-icon-video-pause" value="暂停动画" @click="navgControl('pause')" :disabled="beforeInit"></el-button>
            </div>
            <div class="progress fl" style="margin-right: 0;">
              <el-button type="text" icon="el-icon-d-arrow-left" value="减速" :disabled="isMinSpeed" @click="changeSpeed('less')"></el-button>
            </div>
            <div class="progress fl">
              <el-button type="text" class="speed fl" >{{ times }}X</el-button>
            </div>
            <div class="progress fl">
              <el-button type="text" icon="el-icon-d-arrow-right" value="加速" :disabled="isMaxSpeed" @click="changeSpeed('add')"></el-button>
            </div>
          </div>
          <div class="progress fl" style="margin: 0 0 0 10px;">
            <el-slider v-model="sliderVal" :show-tooltip="false" :format-tooltip="hideFormat" :step="0.0001" :disabled="beforeInit" @change="carReLocate"></el-slider>
          </div>
        </div>
      </div>

      <!--轨迹详细-->
      <div class="track-table" v-if="isShowDetail">
        <el-tabs v-model="activeName" @tab-click="handleClick">
          <el-tab-pane :label="'轨迹明细(' + trackTableDataBak.length + ')'" name="first">
            <el-table
              ref="table"
              size="small"
              :data="trackTableData"
              :header-cell-style="{backgroundColor:'#F3F4F7', height: '25px'}"
              :height="200"
              border
              v-loadmore="loadMore"
              highlight-current-row
              style="width: 100%">
              <el-table-column type="index" width="50"></el-table-column>
              <el-table-column prop="recvtime" label="服务器时间" width="140"></el-table-column>
              <el-table-column prop="gpstime" label="卫星时间" width="140"></el-table-column>
              <el-table-column prop="lng" label="经度" width="120"></el-table-column>
              <el-table-column prop="lat" label="纬度" width="120"></el-table-column>
              <el-table-column prop="speed" label="速度(km/h)" width="90"></el-table-column>
              <el-table-column prop="dpf_before" label="DPF前温(℃)" width="90"></el-table-column>
              <el-table-column prop="dpf_after" label="DPF后温(℃)" width="90"></el-table-column>
              <el-table-column prop="dpf_change" label="DPF前后压差(Kpa)" width="130"></el-table-column>
              <el-table-column prop="dpf_grainbefore" label="DPF前颗粒物浓度(m-1)" width="160"></el-table-column>
              <el-table-column prop="dpf_grainafter" label="DPF后颗粒物浓度(m-1)" width="160"></el-table-column>
              <el-table-column prop="smoke_value" label="烟度值(%)" width="90"></el-table-column>
              <el-table-column prop="statusdescription" label="状态" min-width="90"></el-table-column>
              <el-table-column prop="dir" label="行驶里程(km)" width="100"></el-table-column>
              <el-table-column prop="locationdescription" label="位置" min-width="200" :show-overflow-tooltip="true"></el-table-column>
            </el-table>
          </el-tab-pane>
        </el-tabs>
      </div>
    </el-card>
  </div>
</template>

<script>
  import {lazyAMapApiLoaderInstance} from 'vue-amap'
  import {getAction} from '@/api/manage'
  import MapTrackData from './data/MapTrackData.js'
  import MapTrackTableData from './data/MapTrackTableData.js'

  export default {
    props: {
      vehicleNo: {
        type: String,
        default: ''
      },
    },
    watch:{
      sliderVal(newVal) {
        if (!this.isOnSlider) {
          return false;
        }
        this.sliderChange(newVal)
      }
    },
    data() {
      return {
        activeName: 'first',

        startTime: '',
        endTime: '',
        isShowDetail: false,

        loading: false,
        initSimplifier: false,
        // 信息窗体
        infoWindow: null,
        // 巡航轨迹
        AMap: null,
        map: null,
        PathSimplifier: null,
        beforeInit: true,
        isPlay: true,
        sliderVal: 0, // 进度条
        times: 1, // 倍速
        maxSpeed: 32, // 最高倍速
        navgtrSpeed: 800, // 速度
        isMinSpeed: true,
        isMaxSpeed: false,
        navgtr: null,
        pathSimplifierIns: null,
        actualList: [],
        defaultRenderOptions: {
          renderAllPointsIfNumberBelow: -1, // 描绘路径点，如不需要设为-1
          pathTolerance: 2,
          keyPointTolerance: 0,
          pathLineStyle: {
            lineWidth: 6,
            strokeStyle: '#409eff',
            borderWidth: 1,
            borderStyle: '#eeeeee',
            dirArrowStyle: false
          },
          pathLineHoverStyle: {
            lineWidth: 6,
            strokeStyle: '#ff0000',
            borderWidth: 1,
            borderStyle: '#cccccc',
            dirArrowStyle: false
          },
          dirArrowStyle: {
            stepSpace: 30,
            strokeStyle: "#ffffff",
            lineWidth: 2
          },
          pathLineSelectedStyle: {
            lineWidth: 6,
            strokeStyle: '#409eff',
            borderWidth: 1,
            borderStyle: '#cccccc',
            dirArrowStyle: true
          },
          keyPointStyle: {
            radius: 0,
            fillStyle: 'rgba(8, 126, 196, 1)',
            lineWidth: 1,
            strokeStyle: '#eeeeee'
          },
          keyPointHoverStyle: {
            radius: 0,
            fillStyle: 'rgba(0, 0, 0, 0)',
            lineWidth: 2,
            strokeStyle: '#ffa500'
          },
          keyPointOnSelectedPathLineStyle: {
            radius: 0,
            fillStyle: 'rgba(8, 126, 196, 1)',
            lineWidth: 2,
            strokeStyle: '#eeeeee'
          }
        },
        isCursorAtPathEnd: false,
        palyStayus: 0, //0->未开始  1->行驶中  2->暂停
        value: 0,  // 进度条初始化

        // 分页 *******************
        trackTableData: [],
        trackTableDataBak: [],
        currentPage: 1,
        totalData: 0,
        pageNo: 1,
        pageSize: 10,

        signMarker: null,
        firstArr: [116.397428, 39.90923],
        currentPoint: null,

        //*********
        pickerOptions: {
          disabledDate(time) {
            return time.getTime() > Date.now()
          },
          shortcuts: [{
            text: '今天',
            onClick(picker) {
              picker.$emit('pick', new Date())
            }
          }, {
            text: '昨天',
            onClick(picker) {
              const date = new Date()
              date.setTime(date.getTime() - 3600 * 1000 * 24)
              picker.$emit('pick', date)
            }
          }, {
            text: '一周前',
            onClick(picker) {
              const date = new Date()
              date.setTime(date.getTime() - 3600 * 1000 * 24 * 7)
              picker.$emit('pick', date)
            }
          }]
        },
        timeValue: '',
        trackData: []
      }
    },
    created() {

    },
    mounted() {
      // 为滑动条版绑定监听事件
      let that = this;
      let el = document.getElementsByClassName("el-slider__button-wrapper")[0];
      let el2 = document.getElementsByClassName("el-slider__runway")[0];
      el2.addEventListener("click", that.sliderChange,false);
      el.addEventListener("mousedown", that.openSlider,false);
      // 此处用document是因为，滑动较为随意时，mouseup可能不是作用在el上
      document.addEventListener("mouseup",that.closeSlider,false);

      lazyAMapApiLoaderInstance.load().then(() => {
        // 初始化地图
        this.initMap()
        // 初始化点位
        this.initMarker()
        // 初始化信息窗体
        this.initInfoWindow()
      })
    },
    methods: {
      handleClick(tab, event) {
        console.log(tab, event);
      },
      getTableData(isTimeChange) {
        this.pageNo = 1
        this.trackTableData = []
        if (!isTimeChange) {
          this.isShowDetail = !this.isShowDetail;
        } else{
          if (this.$refs.table) { this.$refs.table.bodyWrapper.scrollTop = 0 }
        }
        if (this.startTime && this.endTime && this.vehicleNo) {
          this.loading = true
          let uri = '?startTime=' + this.startTime +
            '&endTime=' + this.endTime +
            '&vid=' + this.vehicleNo +
            '&pageNo=' + this.pageNo +
            '&pageSize=' + this.pageSize;
          // getAction('/emission/list' + uri).then(resp => {
          //   if (resp.success) {
          //     this.trackData = resp.result.records
          //     this.currentPage = resp.result.current
          //     this.totalData = resp.result.total
          //   }
          // }).finally(() => {
          //   this.loading = false
          // });
          this.trackTableDataBak = MapTrackTableData.mapTrackTableData()
          if (this.trackTableDataBak.length > 0) {
            this.trackTableData = this.trackTableDataBak.slice(this.pageNo - 1, this.pageSize + this.pageNo - 1);
            this.pageNo++
            this.aq = true
          }
        }
      },
      loadMore() {
        console.log(this.loadSign)
        if(this.aq === false){
          return
        }
        if(this.page === 1){
          this.page++
        }

        // this.$axios({
        //   method:'get',
        //   url:this.api+'admin/StockLevel',
        //   params:{
        //     enabled:this.value1,
        //     page:this.page,
        //     limit:this.limit
        //   }
        // }).then(res=>{
        //   if(res.data.status==1){
            let trackTableData = this.trackTableDataBak.slice((this.pageNo - 1) * this.pageSize, this.pageSize + ((this.pageNo - 1) * this.pageSize));
            if(trackTableData && trackTableData.length > 0){
              this.pageNo++
              trackTableData.forEach(res => {
                this.trackTableData.push(res)
              });
            }else{
              this.aq = false
            }
          // }
        // })
      },
      // 时间改变
      timeChangeHandle() {
        if (this.startTime && this.endTime) {
          this.queryTimes()
        }
      },
      // 初始化巡航组件实例
      initPathSimplifier() {
        let that = this
        AMapUI.load(['ui/misc/PathSimplifier'], (PathSimplifier) => {
          if (!PathSimplifier.supportCanvas) {
            alert('当前环境不支持 Canvas！')
            return
          }

          that.initSimplifier = true
          that.signMarker.setLabel({})
          let startTime = this.timeValue + ' 00:00:00';
          let endTime = this.timeValue + ' 23:59:59';
          let uri = '?startTime=' + startTime + '&endTime=' + endTime + '&vid=' + this.vehicleNo
          // getAction('/emission/queryLineArr' + uri).then(resp => {
          //   if (resp.success) {
              // 如果已存在巡航轨迹，则删除
              if (window.pathSimplifierIns && that.pathSimplifierIns) {
                //通过该方法清空上次传入的轨迹
                that.pathSimplifierIns.setData([]);
              }
              // TODO
              let data = MapTrackData.mapTrackData()
              let linArray = data.result.lineArray
              let pointDataList = data.result.pointDataList
              // 初始化坐标点
              if (linArray.length > 0) {
                that.signMarker.show()
                that.signMarker.setPosition(linArray[0])

                that.actualList = linArray

                //创建一个巡航轨迹路线
                that.pathSimplifierIns = new PathSimplifier({
                  zIndex: 100,//地图层级，
                  map: this.map, //所属的地图实例
                  //巡航路线轨迹列表
                  getPath: (pathData, pathIndex) => {
                    return pathData.path
                  },
                  //hover每一个轨迹点，展示内容
                  getHoverTitle: function(pathData, pathIndex, pointIndex) {
                    /*if (pointIndex >= 0) {
                    return pathData.name + '，点：' + pointIndex + '/' + pathData.path.length;
                  }
                  return pathData.name + '，点数量' + pathData.path.length;*/
                    return ''
                  },
                  //自定义样式，可设置巡航器样式，巡航轨迹样式，巡航轨迹点击、hover等不同状态下的样式，不设置则用默认样式，详情请参考api文档 renderOptions:{}
                  //绘制路线节点
                  renderOptions: that.defaultRenderOptions
                })
                window.pathSimplifierIns = that.pathSimplifierIns
                //设置数据
                that.pathSimplifierIns.setData([{
                  name: '轨迹路线',
                  path: that.actualList
                }])
                that.pathSimplifierIns.setSelectedPathIndex(0)

                function onload() {
                  that.pathSimplifierIns.renderLater()
                }

                function onerror(e) {
                  console.log('图片加载失败！')
                }

                //对第一条线路（即索引 0）创建一个巡航器
                let image = PathSimplifier.Render.Canvas.getImageContent('/t_car.png', onload, onerror)
                that.navgtr = that.pathSimplifierIns.createPathNavigator(0, {
                  loop: false, //循环播放
                  speed: that.navgtrSpeed, //巡航速度，单位千米/小时
                  pathNavigatorStyle: {
                    width: 26,
                    height: 52,
                    //使用图片
                    content: image, // 自定义巡航样式
                    strokeStyle: null,
                    fillStyle: null,
                    //经过路径的样式
                    pathLinePassedStyle: {
                      lineWidth: 6,
                      strokeStyle: '#69f81e',
                      dirArrowStyle: {
                        stepSpace: 15,
                        strokeStyle: '#FFF'
                      }
                    }
                  }
                })

                that.navgtr.on('start resume', function() {
                  that.navgtr._startTime = Date.now()
                  that.navgtr._startDist = this.getMovedDistance()
                })
                that.navgtr.on('stop pause', function() {
                  that.navgtr._movedTime = Date.now() - that.navgtr._startTime
                  that.navgtr._movedDist = this.getMovedDistance() - that.navgtr._startDist
                })
                that.navgtr.on('move', function(data, position) {
                  that.isCursorAtPathEnd = false
                  let idx = position.dataItem.pointIndex //走到了第几个点
                  let tail = position.tail //至下一个节点的比例位置
                  let totalIdx = idx + tail
                  let len = position.dataItem.pathData.path.length
                  // 设置当前点位
                  that.currentPoint = that.actualList[idx]
                  // 打开信息窗体
                  let content = [
                    '<div style="padding: 5px;">',
                    '<div>接收时间 : ' + pointDataList[idx].receiveTime + '</div>',
                    '<div>速&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;度 : ' + pointDataList[idx].cs + '(km/h)</div>',
                    '<div>经&nbsp;&nbsp;纬&nbsp;&nbsp;度 : ' + pointDataList[idx].lng + ',' + pointDataList[idx].lat + '</div>',
                    '</div>'
                  ]
                  that.infoWindow.setContent(content.join(''))
                  that.infoWindow.open(that.map, that.actualList[idx])

                  // 计算下一个距离速度
                  // that.navgtr._movedTime = Date.now() - that.navgtr._startTime;
                  // that.navgtr._movedDist = this.getMovedDistance() - that.navgtr._startDist;
                  // that.navgtr._realSpeed = (that.navgtr._movedDist / that.navgtr._movedTime * 3600);
                  // let point = trackList[idx]
                  if (idx < len - 1) {
                    // that.navgtr.setSpeed(that.navgtr._realSpeed * that.times);
                    that.navgtr.setSpeed(that.navgtrSpeed * that.times)
                  }
                  // 进度条实时展示tail
                  !that.isOnSlider && (that.sliderVal = (totalIdx / len) * 100)
                  // 已经播放时间
                  // let sTime = (pointObj.stampTime-startStampTime)/1000;
                  // let sTime = parseInt((endPoint.stampTime - startPoint.stampTime) / 1000 * that.sliderVal / 100);
                  //
                  // that.passedTime = that.getTime(sTime);
                  // 如果到头了，回到初始状态
                  if (that.navgtr.isCursorAtPathEnd()) {
                    that.isCursorAtPathEnd = true
                    that.initPlayBox()
                  }
                })

                // 加载完成
                that.beforeInit = false
              } else {
                that.signMarker.hide()
                that.signMarker.setLabel({})
              }
            // }
          // })
          // .finally(() => {
            // 隐藏加载
            that.initSimplifier = false
          // })
        })
      },

      openSlider(){
        this.isOnSlider = true;
      },
      closeSlider(){
        this.isOnSlider = false;
      },
      hideFormat() {
        return '';
      },
      // 控制播放按钮
      navgControl(type) {
        if (!this.navgtr || !type) {
          return
        }
        if (type === 'start' || type === 'resume') {
          this.isPlay = false
          this.palyStayus = 2
          // 如果已经到了终点，重新定位坐标
          if (this.isCursorAtPathEnd && this.actualList.length > 0) {
            this.map.setCenter(this.actualList[0])
          }
        } else if (type === 'pause') {
          this.isPlay = true
          this.palyStayus = 1
        }
        this.navgtr[type]()
      },
      // 滑块改变位置
      sliderChange(val){
        if (this.beforeInit) {
          return
        }
        // 先改为播放状态
        if (this.palyStayus === 0) {
          this.navgControl('start')
          this.navgControl('pause')
        }
        // 移动巡航器
        let newVal = typeof(newVal)==='number' ? val : this.sliderVal
        let num = parseInt((newVal / 100) * this.actualList.length);
        let decimal = String((newVal / 100) * this.actualList.length).split('.')[1]||0
        this.navgtr.moveToPoint(num, Number('0.'+decimal));
        this.pathSimplifierIns.renderLater();
      },
      // 改变倍速
      changeSpeed(operate) {
        let base = 2
        this.isMinSpeed = false
        this.isMaxSpeed = false
        if (operate === 'add') {
          if (this.times * base === this.maxSpeed) {
            this.times *= base;
            this.isMaxSpeed = true
            this.isMinSpeed = false
          } else if (this.times + base > this.maxSpeed) {
            this.times = this.maxSpeed;
            this.isMaxSpeed = true
            this.isMinSpeed = false
          } else {
            this.times *= base;
          }
        } else if (operate === 'less') {
          if (this.times / base === 1) {
            this.times /= base;
            this.isMinSpeed = true
            this.isMaxSpeed = false
          } else if (this.times - base < 1) {
            this.times /= 1;
            this.isMinSpeed = true
            this.isMaxSpeed = false
          } else {
            this.times /= base;
          }
        }
      },
      // 分页
      handleSizeChange(val) {
        this.pageSize = val
        this.queryTimes(true)
      },
      handleCurrentChange(val) {
        this.pageNo = val
        this.queryTimes(true)
        // this.queryTimes()
      },

      // 查询当天车辆数据采集时间
      queryTimes(pageChange) {
        if (this.startTime && this.endTime && this.vehicleNo) {
          this.getTableData(true)
          /*getAction('/emission/noPage/list' + uri).then(resp => {
            if (resp.success) {
              this.trackData = resp.result
            }
          })*/
          if (!pageChange) {
            if(this.infoWindow) this.infoWindow.close()
            this.initPlayBox()
            this.beforeInit = true
            // 初始化巡航组件
            this.initPathSimplifier()
          }
        } else if (!this.vehicleNo) {
          this.$message({
            message: '请选择机械',
            type: 'warning'
          })
        } else if (!this.startTime || !this.endTime) {
          this.$message({
            message: '请选择时间区间！',
            type: 'warning'
          })
        }
      },
      /* 获取table行号 */
      tableRowClassName({row, rowIndex}) {
        row.index = rowIndex;
      },
      // 选中表单时间触发，查询指定时间车辆位置
      queryPointInfo(row, column) {
        // console.log(this.trackData[row.index])
        let position = [this.trackData[row.index].lng, this.trackData[row.index].lat]
        this.signMarker.setPosition(position)
        this.signMarker.setLabel({
          offset: new AMap.Pixel(0, -15), // 设置文本标注偏移量
          content: '<div style="background-color: rgba(0,0,0,0);">' + position[0] + ',' + position[1] + '</div>', // 设置文本标注内容
          direction: 'top' // 设置文本标注方位
        })
        this.map.setCenter(position)
      },

      // 初始化地图
      initMap() {
        // map-test为div的id
        // zooms: [3, 18], // 地图缩放范围
        // eslint-disable-next-line no-undef
        this.map = new AMap.Map('map-test', {
          resizeEnable: true, // 窗口大小调整
          // eslint-disable-next-line no-undef
          center: this.firstArr, // 地图中心点
          zoom: 5
        })
      },
      initMarker() {
        // 引入Marker,绘制点标记
        this.signMarker = new AMap.Marker({
          map: this.map,
          offset: new AMap.Pixel(-4, -4),
          visible: false,
          content: '<div style="text-align:center; background-color: hsla(180, 100%, 50%, 0.9); height: 10px; width: 10px; border: 1px solid hsl(180, 100%, 40%); border-radius: 12px; box-shadow: hsl(180, 100%, 50%) 0px 0px 1px;"></div>'
        })
      },
      initInfoWindow() {
        // 创建 infoWindow 实例
        this.infoWindow = new AMap.InfoWindow({
          anchor: 'bottom-center',
          autoMove: false,
          offset: new AMap.Pixel(0, -20),
          content: ''  //传入 dom 对象，或者 html 字符串
        })
      },

      /**
       * 轨迹回放重新定位
       */
      carReLocate() {
        // 鼠标从滑动条抬起时，重新定位
        if (this.currentPoint) {
          let timeout = setTimeout(() => {
            clearTimeout(timeout)
            this.map.setCenter(this.currentPoint)
          }, 0)
        }
      },
      // 初始化播放器状态
      initPlayBox() {
        // 暂停
        this.navgControl('pause')
        this.playIcon = 'start';
        this.isPlay = true // 播放图标
        this.palyStayus = 0 // 继续状态
        this.sliderVal = 0; // 进度条清0
      }
    }
  }
</script>

<style scoped lang="less">
  @deep: ~'>>>';
  #map-test {
    width: 100%;

    @{deep}.amap-marker-label {
      border: 1px solid #ffe099 !important;
      background-color: rgba(255, 255, 255, 0.9) !important;
    }

    @{deep}.amap-info-close {
      display: none;
    }
  }

  .button {
    float: bottom;
  }

  .tabSty {
    overflow-y: auto;
    height: calc(100vh - 355px);
    min-height: 510px;
  }

  .playBox {
    height: 25px;
    width: 275px;
    display: -ms-flexbox;
    display: flex;
    -ms-flex-pack: center;
    justify-content: center;
    -ms-flex-align: center;
    align-items: center;

    .el-button {
      padding: 0px;
      font-size: 25px;
    }

    .playIcon {
      height: 30px;
      display: inline-block;
      width: 30px;
      margin: 0 20px 0 25px;
    }

    .progress {
      display: inline-block;
      /*height: 35px;*/
      width: 300px;
      margin: 0px 15px;
    }

    .speed {
      font-size: 15px;
      font-weight: bolder;
      color: #1890FF;
      width: 40px;
      padding-left: 10px;
      text-align: center;
      cursor: pointer;
    }

    .fl {
      float: left;
    }
  }

  #marker-info {
    .input-item {
      margin-bottom: 5px;
    }
  }

  #loadingTip {
    position: absolute;
    top: 0;
    z-index: 9999;
    left: 0;
    padding: 3px 10px;
    background: red;
    color: #fff;
    font-size: 13px
  }

  .box-card {
    position: relative;
  }

  .guijiset {
    position: absolute;
    top: 10px;
    left: 10px;
    -webkit-box-shadow: 0 0 5px rgba(0,0,0,.3);
    box-shadow: 0 0 5px rgba(0,0,0,.3);
    background: #fff;
    padding: 15px;
    border-radius: 5px;
    z-index: 1000;
  }

  .form-table-search {
    background: #fff;
    border-radius: 5px;
    overflow: hidden;
  }

  .rowset {
    overflow: hidden;
    margin-bottom: 10px;
    text-align: left;
  }

  .rowset label {
    font-size: 12px;
    display: inline-block;
  }

  .track-detail{
    z-index: 100;
    position: absolute;
    top: 15px;
    right: 15px;
    font-size: 15px;
    font-weight: 600;
    border-radius: 0;
  }

  .track-table {
    z-index: 99;
    /*position: absolute;*/
    width: 100vw;
    bottom: 0;

    @{deep}.el-tabs__header{
      margin: 0;
    }
    @{deep}.el-tabs__nav {
      margin-left: 15px;
    }
  }
</style>
