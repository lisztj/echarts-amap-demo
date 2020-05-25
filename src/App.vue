/**
* @Description:    下载geoJson文件
* @Author:         TSY
* @CreateDate:     2018/9/5 9:04
* @email:          t@tsy6.com
*/
<template>
    <div class="body">
       
        <div class="echarts"> 
             <div id="map"></div>
              <div id='areaback' @click="cback()" class="back">返回</div>
            <map-range @change="search" ></map-range>
           
        </div>
        
    </div>
</template>

<script>
    import JSZip from 'jszip'
    import saveAs from './saveAs'
    import MapRange from "./MapRange";

    export default {
        name: "demo",
        components: {
            MapRange},
        data() {
            return {
                cityName: '中国',
                areaCode: 10000,
                geoJsonData: '',
                echartsMap: null,
                map: null,
                uimap: null,
                district: null,
                polygons: [],
                cityCode: 100000,
                citySelect: null,
                districtSelect: null,
                opts: {},
                areaData: {},
                mapData: [],
                zip: {},//打包zip
                codeList: [],
                isCodeListLoadComplete: false,//codeList是否全部获取完毕
            }
        },
        mounted() {
            //实例化zip对象
            this.zip = new JSZip();
            this.inits();
            
        },
        watch: {
            isCodeListLoadComplete(val) {
                if (val) {
                    this.loadAllGeoJson();
                }
            }
        },
        methods: {
            inits(){
            this.citySelect = document.getElementById('city');
            this.districtSelect = document.getElementById('district');
            this.echartsMap = this.$echarts.init(document.getElementById('map'));
            this.echartsMap.on('click', this.echartsMapClick);

            this.map = new AMap.Map('container', {
                resizeEnable: true,
                center: [116.30946, 39.937629],
                zoom: 3
            });
            this.opts = {
                subdistrict: 1,   //返回下一级行政区
                showbiz: false  //最后一级返回街道信息
            };
            this.district = new AMap.DistrictSearch(this.opts);//注意：需要使用插件同步下发功能才能这样直接使用
            this.district.search('中国', (status, result) => {
                if (status == 'complete') {
                    this.getData(result.districtList[0], '', 100000);
                }
            });
            },
            cback(){///返回按钮
                  this.inits();
            },
            echartsMapClick(params) {//地图点击事件
                this.$ba.trackEvent('echartsMap', '点击地图', `${params.data.name}-${params.data.cityCode}`);
                if (params.data.level == 'street') return;
                //清除地图上所有覆盖物
                for (var i = 0, l = this.polygons.length; i < l; i++) {
                    this.polygons[i].setMap(null);
                }
                this.cityName = params.data.name;
                this.cityCode = params.data.cityCode;
                this.district.setLevel(params.data.level); //行政区级别
                this.district.setExtensions('all');
                //行政区查询
                //按照adcode进行查询可以保证数据返回的唯一性
                this.district.search(this.cityCode, (status, result) => {
                    if (status === 'complete') {
                        this.getData(result.districtList[0], params.data.level, this.cityCode);
                    }
                });
            },
            loadMapData(areaCode) {
                AMapUI.loadUI(['geo/DistrictExplorer'], DistrictExplorer => {

                    //创建一个实例
                    var districtExplorer = window.districtExplorer = new DistrictExplorer({
                        eventSupport: true, //打开事件支持
                        map: this.map
                    });

                    districtExplorer.loadAreaNode(areaCode, (error, areaNode) => {

                        if (error) {
                            console.error(error);
                            return;
                        }
                        let mapJson = {};
                        mapJson.features = areaNode.getSubFeatures();
                        this.loadMap(this.cityName, mapJson);
                        this.geoJsonData = mapJson;
                    });
                });
            },            
            loopSearch(code) {
                setTimeout(() => {
                    this.district.search(code, (status, result) => {
                        if (status == 'complete') {
                            console.log(`${code}--获取成功`)
                            for (let i in result.districtList[0].districtList) {
                                this.codeList.push({
                                    name: result.districtList[0].districtList[i].name,
                                    code: result.districtList[0].districtList[i].adcode,
                                    level: result.districtList[0].districtList[i].level
                                })
                                //这边没想出来怎么判断数据是否全部加载完毕了，只能采用这种死办法
                                //有更好解决方案的大佬，麻烦告诉我一下，邮箱t@tsy6.com
                                //或者直接Github提交PR，在此不胜感激
                                if (this.codeList.length >= 428) {
                                    console.log('code获取完成');
                                    this.isCodeListLoadComplete = true;
                                }
                                if (result.districtList[0].districtList[i].adcode && result.districtList[0].districtList[i].level != 'city' && result.districtList[0].districtList[i].level != 'district' && result.districtList[0].districtList[i].level != 'street') {
                                    this.loopSearch(result.districtList[0].districtList[i].adcode)
                                }
                            }
                        } else {//第一遍查询出错，再次执行查询
                            console.log(`${code}--第一次获取失败，正在尝试进行第二次获取`)
                            this.district.search(code, (status, result) => {
                                if (status == 'complete') {
                                    console.log(`${code}--第二次获取成功`)
                                    for (let i in result.districtList[0].districtList) {
                                        this.codeList.push({
                                            name: result.districtList[0].districtList[i].name,
                                            code: result.districtList[0].districtList[i].adcode,
                                            level: result.districtList[0].districtList[i].level
                                        })
                                        //这边没想出来怎么判断数据是否全部加载完毕了，只能采用这种死办法
                                        //有更好解决方案的大佬，麻烦告诉我一下，邮箱t@tsy6.com
                                        //或者直接Github提交PR，在此不胜感激
                                        if (this.codeList.length >= 428) {
                                            console.log('code获取完成');
                                            this.isCodeListLoadComplete = true;
                                        }
                                    }
                                } else {
                                    console.log(`${code}--第二次获取失败，请联系email：t@tsy6.com`)
                                }
                            })
                        }
                    });
                }, 500)
            },
            loadAllGeoJson() {//通过codeList加载全部geoJson数据
                console.log('开始加载geoJson数据');
                AMapUI.loadUI(['geo/DistrictExplorer'], DistrictExplorer => {

                    //创建一个实例
                    var districtExplorer = window.districtExplorer = new DistrictExplorer({
                        eventSupport: true, //打开事件支持
                        map: this.map
                    });
                    let mapJson = {};
                    for (let i in this.codeList) {
                        setTimeout(() => {
                            districtExplorer.loadAreaNode(this.codeList[i].code, (error, areaNode) => {
                                if (error) {
                                    this.codeList[i].geo = 'error';
                                    console.log(`${this.codeList[i].name}--${this.codeList[i].code}，geo 数据获取失败，高德地图的锅^_^`)
                                } else {
                                    mapJson.features = areaNode && areaNode.getSubFeatures() || '';
                                    this.codeList[i].geo = mapJson;
                                    console.log(`${this.codeList[i].level}--${this.codeList[i].name}--${this.codeList[i].code}，geo 数据获取成功，马上为你打包`)
                                }

                                if (this.codeList[i].level === 'province') {
                                    this.zip.file(`100000/${this.codeList[i].code}.geoJson`, JSON.stringify(mapJson));
                                } else {
                                    this.zip.file(`100000/${this.codeList[i].code.substring(0, 2)}0000/${this.codeList[i].code}.geoJson`, JSON.stringify(mapJson));
                                }

                                if (this.codeList.every(item => item.geo)) {
                                   
                                    // this.downloadTips = '文件打包压缩中...';
                                    // this.zip.generateAsync({type: "blob"})
                                    //     .then((content) => {
                                    //         saveAs(content, "geoJson数据包.zip");
                                    //         this.downloadTips = '下载geoJson数据';
                                    //         this.isCodeListLoadComplete = false;
                                    //         this.$ba.trackEvent('echartsMap', '文件下载', '打包下载成功');
                                    //     });
                                }
                            });
                        }, 100 * i)
                    }
                });
            },
            loadMap(mapName, data) {///地图配置
                if (data) {
                     
                    var geoCoordMap = {
                        '上海': [119.1803, 31.2891],
                        "福建": [119.4543, 25.9222],
                        "重庆": [108.384366, 30.439702],
                        '北京': [116.4551, 40.2539],
                        "辽宁": [123.1238, 42.1216],
                        "河北": [114.4995, 38.1006],
                        "天津": [117.4219, 39.4189],
                        "山西": [112.3352, 37.9413],
                        "陕西": [109.1162, 34.2004],
                        "甘肃": [103.5901, 36.3043],
                        "宁夏": [106.3586, 38.1775],
                        "青海": [101.4038, 36.8207],
                        "新疆": [87.9236, 43.5883],
                        "西藏": [91.11, 29.97],
                        "四川": [103.9526, 30.7617],
                        "吉林": [125.8154, 44.2584],
                        "山东": [117.1582, 36.8701],
                        "河南": [113.4668, 34.6234],
                        "江苏": [118.8062, 31.9208],
                        "安徽": [117.29, 32.0581],
                        "湖北": [114.3896, 30.6628],
                        "浙江": [119.5313, 29.8773],
                        '内蒙古': [110.3467, 41.4899],
                        "江西": [116.0046, 28.6633],
                        "湖南": [113.0823, 28.2568],
                        "贵州": [106.6992, 26.7682],
                        "云南": [102.9199, 25.4663],
                        "广东": [113.12244, 23.009505],
                        "广西": [108.479, 23.1152],
                        "海南": [110.3893, 19.8516],
                        '黑龙江': [127.9688, 45.368],
                        '台湾': [121.4648, 25.5630]
                    };
                    var chinaDatas = [
                    {
                        name: '北京',
                        value: 86
                    },
                    {
                        name: '福建',
                        value: 65
                    },
                    {
                        name: '广东',
                        value: 53
                    },
                    {
                        name: '上海',
                        value: 48
                    },
                    {
                        name: '河北',
                        value: 2
                    },
                    {
                        name: '天津',
                        value: 6
                    },
                    {
                        name: '山西',
                        value: 12
                    },
                    {
                        name: '陕西',
                        value: 2
                    },
                    {
                        name: '甘肃',
                        value: 4
                    },
                    {
                        name: '宁夏',
                        value: 5
                    },
                    {
                        name: '青海',
                        value: 3
                    },
                    {
                        name: '新疆',
                        value: 0.4
                    },
                    {
                        name: '西藏',
                        value: 0.3
                    },
                    {
                        name: '四川',
                        value: 22
                    },
                    {
                        name: '重庆',
                        value: 12
                    },
                    {
                        name: '山东',
                        value: 9
                    },
                    {
                        name: '河南',
                        value: 0.9
                    },
                    {
                        name: '江苏',
                        value: 24
                    },
                    {
                        name: '安徽',
                        value: 15
                    },
                    {
                        name: '湖北',
                        value: 6
                    },
                    {
                        name: '浙江',
                        value: 15
                    },
                    {
                        name: '内蒙古',
                        value: 0.3
                    },
                    {
                        name: '江西',
                        value: 34
                    },
                    {
                        name: '湖南',
                        value: 12
                    },
                    {
                        name: '贵州',
                        value: 0.8
                    },
                    {
                        name: '广西',
                        value: 16
                    },
                    {
                        name: '海南',
                        value: 37
                    },
                    {
                        name: '辽宁',
                        value: 0.2
                    },
                    {
                        name: '吉林',
                        value: 0.4
                    },
                    {
                        name: '云南',
                        value: 34
                    },
                    {
                        name: '黑龙江',
                        value: 5
                    },
                    {
                        name: '台湾',
                        value: 43
                    }];
                
                var convertData = function(data, type) {
                    /*type:1 地图参数；type:2数据参数*/
                    var res = [];
                    for (var i = 0; i < data.length; i++) {
                        var geoCoord = geoCoordMap[data[i].name];
                        if (geoCoord) {
                            if (type == 2) {
                                res.push({
                                    name: data[i].name,
                                    value: geoCoord.concat(data[i].value),
                                    username: data[i].username,
                                    telphone: data[i].telphone,
                                    address: data[i].address
                                });
                            } else {
                                res.push({
                                    name: data[i].name,
                                    value: geoCoord.concat(data[i].value)
                                });
                            }
                        }
                    }
                    //console.log(res);
                    return res;
                };
                var yData = [];
                var barData = this.mapData;
                barData = barData.sort(function(a,b){
                    return b.value-a.value;
                });
                for(var j =0;j<barData.length;j++){
                    if(j<10){
                        yData.push('0'+j + barData[j].name);
                    }else{
                        yData.push(j + barData[j].name);
                    }
                }

                    this.$echarts.registerMap(mapName, data);
                    var option = {
                        title: [
                            {
                            show: true,
                            text: '复学学生生源分布图',
                            subtext: '单位：万人',
                            subtextStyle:{
                                color:'#ffffff',
                                lineHeight:20
                            },
                            textStyle: {
                                color: '#fff',
                                fontSize: 18
                            },
                            right: 190,
                            top: 20
                        }],
                        tooltip: {
                            show: true,
                            formatter: function(params) {
                                return params.name + '：' + params.data['value'] + '万人'
                            },
                            textStyle: {
                                fontSize: 25
                            }
                        },
                        grid: {
                            right: 190,
                            top: 80,
                            bottom: 20,
                            width: '200',
                            // height:'scroll'
                        },
                        xAxis: {
                            show: false
                        },
                        yAxis: {
                            type: 'category',
                            inverse: true,
                            nameGap: 16,
                            axisLine: {
                                show: false,
                                lineStyle: {
                                    color: '#ddd'
                                }
                            },
                            axisTick: {
                                show: false,
                                lineStyle: {
                                    color: '#ddd'
                                }
                            },
                            axisLabel: {
                                interval: 0,
                                margin: 85,
                                textStyle: {
                                    color: '#fff',
                                    align: 'left',
                                    fontSize: 14
                                },
                                rich: {
                                    a: {
                                        color: '#fff',
                                        backgroundColor: '#f0515e',
                                        width: 20,
                                        height: 20,
                                        align: 'center',
                                        borderRadius: 2
                                    },
                                    b: {
                                        color: '#fff',
                                        backgroundColor: '#24a5cd',
                                        width: 20,
                                        height: 20,
                                        align: 'center',
                                        borderRadius: 2
                                    }
                                },
                                formatter: function(params) {
                                    if (parseInt(params.slice(0, 2)) < 3) {
                                        return [
                                            '{a|' + (parseInt(params.slice(0, 2)) + 1) + '}' + '  ' + params.slice(2)
                                        ].join('\n')
                                    } else {
                                        return [
                                            '{b|' + (parseInt(params.slice(0, 2)) + 1) + '}' + '  ' + params.slice(2)
                                        ].join('\n')
                                    }
                                }
                            },
                            data: yData
                        },
                        visualMap: {
                        type: 'continuous',
                        orient: 'horizontal',
                        itemWidth: 10,
                        itemHeight: 80,
                        text: ['高', '低'],
                        showLabel: true,
                        seriesIndex: [0],
                        min: 0,
                        max: 100,
                        inRange: {
                            color: ['#2c9a42', '#d08a00', '#c23c33']
                        },
                        textStyle: {
                            color: '#7B93A7'
                        },
                        left: 'left',
                        top: 'bottom',
                    },
                    
                        series: [
                            {
                            name: '复学学生人数',
                            type: 'map',
                            roam: false,//是否允许缩放
                            mapType: mapName,
                            selectedMode: 'single',
                            showLegendSymbol: true,
                            visibility: 'off',
                            itemStyle: {
                                normal: {
                                    color: '#ccc',
                                    areaColor: '#fff',
                                    borderColor: '#fff',
                                    borderWidth: 0.8,
                                    label: {
                                        show: true,
                                        textStyle: {
                                            color: "rgb(249, 249, 249)"
                                        }
                                    }
                                },
                                emphasis: {
                                    areaColor: false,
                                    borderColor: '#fff',
                                    areaStyle: {
                                        color: '#fff'
                                    },
                                    label: {
                                        show: true,
                                        textStyle: {
                                            color: "rgb(249, 249, 249)"
                                        }
                                    }
                                }
                            },
                            data: this.mapData,
                         },
                        {
                        name: 'barSer',
                        type: 'bar',
                        roam: false,
                        visualMap: false,
                        zlevel: 2,
                        barMaxWidth: 16,
                        barGap: 0,
                        itemStyle: {
                            normal: {
                                color: function(params) {
                                    var colorList = [{
                                            colorStops: [{
                                                offset: 0,
                                                color: '#f0515e'
                                            }, {
                                                offset: 1,
                                                color: '#ef8554'
                                            }]
                                        },
                                        {
                                            colorStops: [{
                                                offset: 0,
                                                color: '#3c38e4'
                                            }, {
                                                offset: 1,
                                                color: '#24a5cd'
                                            }]
                                        }
                                    ];
                                    if (params.dataIndex < 3) {
                                        return colorList[0]
                                    } else {
                                        return colorList[1]
                                    }
                                },
                                barBorderRadius: [0,15,15,0]
                            }
                        },
                        label:{
                            normal: {
                                show: true,
                                textBorderColor: '#333',
                                textBorderWidth: 2
                            }
                        },
                        data: this.mapData.sort((a,b)=>{
                            return b.value-a.value;
                        })
                    
                }
                ]
                    };
                    this.echartsMap.setOption(option);
                }
            },
            getData(data, level, adcode) {
                var bounds = data.boundaries;
                if (bounds) {
                    for (var i = 0, l = bounds.length; i < l; i++) {
                        var polygon = new AMap.Polygon({
                            map: this.map,
                            strokeWeight: 1,
                            strokeColor: '#0091ea',
                            fillColor: '#80d8ff',
                            fillOpacity: 0.2,
                            path: bounds[i]
                        });
                        this.polygons.push(polygon);
                    }
                    this.map.setFitView();//地图自适应
                }

                //清空下一级别的下拉列表
                if (level === 'province') {
                    this.citySelect.innerHTML = '';
                    this.districtSelect.innerHTML = '';
                } else if (level === 'city') {
                    this.districtSelect.innerHTML = '';
                }

                var subList = data.districtList;
                if (subList) {
                    var contentSub = new Option('--请选择--');
                    var curlevel = subList[0].level;
                    if (curlevel === 'street') {
                        let mapJsonList = this.geoJsonData.features;
                        let mapJson = {};
                        for (let i in mapJsonList) {
                            if (mapJsonList[i].properties.name == this.cityName) {
                                mapJson.features = [].concat(mapJsonList[i]);
                            }
                        }
                        this.mapData = [];
                        this.mapData.push({name: this.cityName, value: Math.random() * 100, level: curlevel});
                        this.loadMap(this.cityName, mapJson);
                        this.geoJsonData = mapJson;
                        return;
                    }
                    var curList = document.querySelector('#' + curlevel);
                    curList.add(contentSub);
                    this.mapData = [];
                    for (var i = 0, l = subList.length; i < l; i++) {
                        var name = subList[i].name;
                        var cityCode = subList[i].adcode;
                        this.mapData.push({
                            name: name,
                            value: (Math.random() * 100).toFixed(2),
                            cityCode: cityCode,
                            level: curlevel
                        });
                        var levelSub = subList[i].level;
                        contentSub = new Option(name);
                        contentSub.setAttribute("value", levelSub);
                        contentSub.center = subList[i].center;
                        contentSub.adcode = subList[i].adcode;
                        curList.add(contentSub);
                    }
                    this.loadMapData(adcode);
                    console.log('arcode',adcode);
                    this.areaData[curlevel] = curList;
                }

            },
            search(area) {
                let obj = this.areaData[area];
                //清除地图上所有覆盖物
                for (var i = 0, l = this.polygons.length; i < l; i++) {
                    this.polygons[i].setMap(null);
                }
                var option = obj[obj.options.selectedIndex];
                var keyword = option.text; //关键字
                var adcode = option.adcode;
                this.cityName = keyword;
                this.cityCode = adcode;
                this.$ba.trackEvent('echartsMap', '筛选地图', `${this.cityName}-${this.cityCode}`);
                this.district.setLevel(option.value); //行政区级别
                this.district.setExtensions('all');
                //行政区查询
                //按照adcode进行查询可以保证数据返回的唯一性
                this.district.search(adcode, (status, result) => {
                    if (status === 'complete') {
                        this.getData(result.districtList[0], obj.id, adcode);
                    }
                });
            }
        }
    }
</script>

<style lang="stylus" scoped>

    * {
        font-size 14px
    }

    .body {
        display flex
        width 100%
    }

    .map, .echarts {
        width 0
        flex 1
    }

    .echarts {
        background: url("./images/bg_bigdata.png") no-repeat
        // background-size 100% 100%
        overflow: hidden;
    height: 960px;
    }

    #map {
        width 100%
        height 100vh
    }

    .tips {
        position fixed
        bottom 30%
        left 40%
        padding 10px 15px
        border-radius 5px
        color #fff
        background rgba(0, 0, 0, .8)
        z-index 999
    }
    .back {
            position: absolute;
            width: 100px;
            padding: 10px 0;
            top: 30px;
            right: 50px;
            background: #1c71fb;
            color: #fff;
            text-align: center;
            border-radius: 5px;
            cursor: pointer;
            z-index: 100;
        }
</style>
