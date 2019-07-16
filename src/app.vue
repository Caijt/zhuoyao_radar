<template>
    <div id="app">
        <right-nav
            :version="APP_VERSION"
            :settings="settings"
            :show-menu.sync="showMenu"
            :mode="mode"
        ></right-nav>
        <!-- <normal-nav :version="APP_VERSION" :settings="settings" :show-menu="showMenu"></normal-nav> -->

        <div id="buttons">
            <div>
                <el-button
                    size="mini"
                    @click="getYaolingInfo"
                    type="primary"
                    :loading="searching"
                >查找妖灵</el-button>
                <el-button size="mini" @click="filterDialogVisible = true">自定义筛选</el-button>
                <el-button size="mini" @click="coordDialogVisible = true">妖灵坐标</el-button>
                <el-button size="mini" @click="clearAllMarkers();yaolingCoordList=[]">清空妖灵</el-button>
                <el-button
                    size="mini"
                    @click="stopSearch"
                    type="danger"
                    v-show="searching"
                >停止搜索</el-button>
                <el-button
                    size="mini"
                    :type="this.mode=='wide'?'success':'danger'"
                    @click="openWideDialog"
                >大范围：{{this.mode=="wide"?(Math.pow(max_range,2)+"格"):"关闭"}}</el-button>
                <el-input
                    v-model="address"
                    size="mini"
                    autocomplete="on"
                    @keyup.enter.native="goMap"
                    placeholder="地址定位"
                    class="search-input"
                >
                    <el-button slot="append" icon="el-icon-search" @click="goMap"></el-button>
                </el-input>
            </div>
            <div style="font-size: 12px;position:absolute;top:35px">
                <div>当前线程数: {{sockets.length}}/{{thread}}</div>
                <template v-for="(socket, index) in sockets">
                    <p
                        :key="index"
                        v-if="socket"
                    >线程.{{index+1}} {{socket.task ? `正在执行任务.${socket.task.taskIndex}` : '空闲'}}</p>
                </template>
                <div v-if="radarTask">任务进度:{{closedTask}}/{{radarTask.tasks.length}}</div>
            </div>
        </div>
        <div id="qmap"></div>
        <radar-progress
            :show="progressShow"
            :max-range="max_range"
            :thread="thread"
            :percent="progressPercent"
        ></radar-progress>
        <el-dialog title="自定义筛选" :visible.sync="filterDialogVisible" class="filter-dialog">
            <div class="check">
                <el-checkbox v-model="settings.use_custom">启用</el-checkbox>
            </div>
            <el-row :gutter="10" class="filter-list">
                <el-col
                    v-for="(yl, index) in settings.custom_filter"
                    :key="index"
                    :xs="12"
                    :sm="8"
                    :md="6"
                    :lg="4"
                    :xl="3"
                >
                    <div class="filter-content" :class="{active : yl.on}" @click="yl.on = !yl.on">
                        <img :src="'https://hy.gwgo.qq.com/sync/pet/'+yl.img" :alt="yl.name" />
                        <span :class="{active:up(yl.id)}">{{ yl.name }}</span>
                    </div>
                </el-col>
            </el-row>
        </el-dialog>
        <el-dialog title="妖怪坐标" :visible.sync="coordDialogVisible" size="mini" width="80%" top="0">
            <el-table :data="yaolingCoordList" size="mini">
                <el-table-column label="头像" prop="name" width="80" align="center">
                    <template slot-scope="{row}">
                        <img :src="row.headimg" style="width:40px" />
                    </template>
                </el-table-column>
                <el-table-column label="名字" prop="name" width="100" />
                <el-table-column label="腾讯坐标" prop="qqcoord">
                    <template slot-scope="{row}">
                        <span
                            class="c-link"
                            v-clipboard:copy="row.qqcoord"
                            v-clipboard:success="copySuccess"
                        >{{row.qqcoord}}</span>
                    </template>
                </el-table-column>
                <el-table-column label="安卓坐标" prop="ancoord">
                    <template slot-scope="{row}">
                        <span
                            class="c-link"
                            v-clipboard:copy="row.ancoord"
                            v-clipboard:success="copySuccess"
                        >{{row.ancoord}}</span>
                    </template>
                </el-table-column>
                <el-table-column label="苹果坐标" prop="applecoord">
                    <template slot-scope="{row}">
                        <span
                            class="c-link"
                            v-clipboard:copy="row.applecoord"
                            v-clipboard:success="copySuccess"
                        >{{row.applecoord}}</span>
                    </template>
                </el-table-column>
            </el-table>
        </el-dialog>
        <el-dialog
            title="大范围设置"
            :visible.sync="wideDialogVisible"
            width="400px"
            size="mini"
            top="0"
        >
            <el-form size="mini" label-width="50px">
                <el-form-item label="启用">
                    <el-switch
                        style="float:none"
                        v-model="wideDialogSettings.enable"
                        active-value="wide"
                        inactive-value="temp"
                    ></el-switch>
                </el-form-item>
                <el-form-item label="范围" v-show="wideDialogSettings.enable=='wide'">
                    <el-slider
                        v-model="wideDialogSettings.range"
                        :format-tooltip="formatTooltip"
                        :max="10"
                        :step="0.5"
                    ></el-slider>
                </el-form-item>
            </el-form>
            <div slot="footer">
                <el-button type="primary" size="mini" @click="saveWideSettings">确定</el-button>
            </div>
        </el-dialog>
        <el-dialog title="妖灵详情" top="0" :visible.sync="infoDialogVisible" size="mini" class="test">
            <el-row :gutter="20">
                <el-col :span="8" :xs="{span:24}">
                    <img :src="yaolingInfo.headimg" style="display:block;margin:0 auto" />
                </el-col>
                <el-col :span="16" :xs="{span:24}">
                    <el-form label-width="80px" size="mini">
                        <el-form-item label="名称">{{yaolingInfo.name}}</el-form-item>
                        <el-form-item label="腾讯坐标">
                            <span
                                class="c-link"
                                v-clipboard:copy="yaolingInfo.qqcoord"
                                v-clipboard:success="copySuccess"
                            >{{yaolingInfo.qqcoord}}</span>
                        </el-form-item>
                        <el-form-item label="安卓坐标">
                            <span
                                class="c-link"
                                v-clipboard:copy="yaolingInfo.ancoord"
                                v-clipboard:success="copySuccess"
                            >{{yaolingInfo.ancoord}}</span>
                        </el-form-item>
                        <el-form-item label="苹果坐标">
                            <span
                                class="c-link"
                                v-clipboard:copy="yaolingInfo.applecoord"
                                v-clipboard:success="copySuccess"
                            >{{yaolingInfo.applecoord}}</span>
                        </el-form-item>
                    </el-form>
                </el-col>
            </el-row>
        </el-dialog>
    </div>
</template>
<script>
import tempdata from "./lib/tempdata";
import activities from "./lib/activities";
import mixins from "./mixins/mixins";
import bot from "./mixins/bot";
import map from "./mixins/map";
import RightNav from "./components/rightNav";
import socket from "./mixins/socket";
import RadarProgress from "./components/radarProgress";
import RadarTasks from "./lib/RadarTasks";

import { getLocalStorage, setLocalStorage } from "./lib/util";

import {
    FILTER,
    API_KEY,
    CUR_YAOLING_VERSION,
    APP_VERSION,
    WIDE_SEARCH
} from "./lib/config";
import clipboard from "clipboard";
import { randomBytes } from "crypto";
export default {
    name: "zhuoyao-radar",
    mixins: [mixins, bot, map, socket],
    components: {
        RightNav,
        RadarProgress
    },
    data() {
        let showMenu = false;
        let location = getLocalStorage("radar_location");
        let settings = getLocalStorage("radar_settings");
        let defaultSettings = {
            fit: {
                t1: false,
                t2: false,
                all: false,
                nest: false,
                rare: true,
                fish: false,
                feature: false,
                element: false
            },
            auto_search: false,
            show_time: true,
            position_sync: false,
            wide: FILTER.FILTER_WIDE,
            custom_filter: FILTER.FILTER_CUSTOM,
            use_custom: false,
            show_box: true,
            version: APP_VERSION
        };

        let flag = false;
        let ans = [];

        //如果第一次打开网页 则打开菜单
        if (!settings) {
            showMenu = true;
        } else {
            //对于settings.custom_filter
            //在版本更新后availableYaolings发生变动时，custom_filter会保留以前的键
            //因此需要在此处做一个remapping

            if (settings.custom_filter) {
                let sc = settings.custom_filter;
                let scMap = [];
                sc.forEach(o => {
                    scMap[o.id] = o;
                });
                defaultSettings.custom_filter.forEach((v, i, a) => {
                    if (scMap.hasOwnProperty(v.id)) ans.push(scMap[v.id]);
                    else ans.push(v);
                });
                flag = true;
            }
        }

        settings = Object.assign({}, defaultSettings, settings || {});

        if (flag) {
            settings.custom_filter = ans;
        }

        if (!(location && settings.position_sync)) {
            location = {
                longitude: 116.3579177856,
                latitude: 39.9610780334,
                zoom: 16
            };
        }

        let range = Number(this.$parent.range || WIDE_SEARCH.MAX_RANGE);
        let max_range = range * 2 + 1; // 经纬方向单元格最大数
        // 线程数最多6个
        let thread = Math.min(
            Number(this.$parent.thread || WIDE_SEARCH.MAX_SOCKETS),
            6
        );
        if (this.$parent.mode === "temp") {
            range = 0;
            // thread = 1;
        }
        return {
            location,
            settings,
            showMenu,
            APP_VERSION,
            range,
            thread,
            max_range,
            mode: this.$parent.mode,
            sockets: new Array(thread),
            radarTask: null,
            searching: false,
            map: {},
            clickMarker: null, // 点击位置标记
            userMarker: null, // 用户位置标记
            searchBoxMarker: [], //大范围搜索框标记
            searchBoxWideSet: new Set(), //大范围搜索集合
            searchOutboxMarker: null, //外围指引框标记
            currVersion: CUR_YAOLING_VERSION, //190508版本的json 如果有变动手动更新
            yaolings: tempdata.Data,
            upYaolings: activities.Data,
            markers: new Map(), //妖灵markerclass
            messageMap: new Map(), // 缓存请求类型和id
            botMode: false,
            botInterval: null,
            botTime: 0,
            botGroup: "群号",
            botChecked: [],
            botWelcomeInfo: "捉妖扫描机器人2.1启动~有什么问题可以@我哦",
            botLocation: {
                longitude: 116.3579177856,
                latitude: 39.9610780334
            },
            progressShow: false,
            filterDialogVisible: false,
            coordDialogVisible: false,
            wideDialogVisible: false,
            yaolingCoordList: [],
            wideDialogSettings: {
                enable: this.$parent.mode,
                range
            },
            yaolingInfo: {
                headimg: "",
                name: "",
                qqcoord: "",
                ancoord: "",
                applecoord: ""
            },
            infoDialogVisible: false,
            geocoder: null,
            address: "",
            canceling: false
        };
    },
    mounted() {
        // 初始化地图组件
        this.initMap();
        this.geocoder = new qq.maps.Geocoder();
        let _this = this;
        this.geocoder.setComplete(function(result) {
            _this.map.setCenter(result.detail.location);
            _this.$notify({
                message: "已定位到：" + result.detail.address,
                position: "bottom-left"
            });
            _this.clickMap({ latLng: result.detail.location });
        });
        this.geocoder.setError(function() {
            _this.$notify.error("无法定位输入的地址！");
        });
        // 初始化websocket
        this.initSockets();
        // 获取用户位置
        if (!this.settings.position_sync) {
            this.getLocation()
                .then(
                    position => {
                        this.location.longitude = position.longitude;
                        this.location.latitude = position.latitude;

                        var pos = new qq.maps.LatLng(
                            this.location.latitude,
                            this.location.longitude
                        );
                        this.map.panTo(pos);
                        this.userMarker = new qq.maps.Marker({
                            position: pos,
                            map: this.map
                        });
                    },
                    e => {
                        console.log(e);
                        if (e.code === 3) {
                            this.notify("无法获取设备位置信息");
                        }
                    }
                )
                .catch(b => {});
        }

        if (this.mode === "wide") {
            this.notify(
                `大范围搜索开启，当前最大搜索单位:${Math.pow(
                    this.max_range,
                    2
                )}个.线程数:${this.thread}个.`
            );
        }
    },
    methods: {
        /**
         * 跨域获取最新妖灵数据
         */
        getYaolings: function() {
            // var url = 'https://hy.gwgo.qq.com/sync/pet/config/' + this.currVersion;
            // try {
            //     $.getJSON(url,result => {
            //         console.log(result);
            //         this.statusOK = true;
            //     });
            //     console.log("1");
            // } catch(e) {
            //     console.log(e);
            //     console.log("3");
            // }
            // console.log("2");
        },
        /**
         * 根据查询结果过滤数据，打标记
         */
        buildMarkersByData: function(t) {
            if (t && t.length) {
                t.forEach(item => {
                    if (
                        this.fit[0] === "special" ||
                        this.fit.indexOf(item.sprite_id) > -1
                    ) {
                        this.addMarkers(item);
                    }
                });
            }
        },
        /**
         * 获取妖灵数据
         */
        getYaolingInfo: function() {
            if (this.botMode) {
                return;
            }
            // 先清除标记
            this.clearAllMarkers();
            if (this.mode === "normal") {
                alert("123");
                if (this.searchOutboxMarker != null)
                    this.searchOutboxMarker.setMap(null);
                if (this.searchBoxMarker.length > 0) {
                    this.searchBoxMarker[0].setMap(null);
                    this.searchBoxMarker.pop();
                }

                this.buildSearchboxMarker(
                    this.location.latitude,
                    this.location.longitude,
                    true
                );
                this.sendMessage(this.initSocketMessage("1001"));
            } else {
                if (this.searching) {
                    this.notify("请等待此次搜索结束！");
                    return false;
                }

                this.clearAllBox();

                this.radarTask = new RadarTasks({
                    range: this.range,
                    lng: this.location.longitude,
                    lat: this.location.latitude
                });

                if (this.mode === "wide") this.progressShow = true;
                this.lng_count = this.lat_count = 0;
                this.searching = true;
                for (let index = 0; index < WIDE_SEARCH.MAX_SOCKETS; index++) {
                    let socket = this.sockets[index];
                    if (socket) {
                        this.startTaskWithSocket(socket);
                    }
                }
                console.log(this.sockets);
            }
        },
        /**
         * 获取擂台数据
         */
        getLeitaiInfo: function() {
            this.notify("功能开发中!");
            return;
            if (this.botMode) return;
            this.sendMessage(this.initSocketMessage("1001"));
        },
        /**
         * 获取官方配置文件
         */
        getSettingFileName: function() {
            this.sendMessage(this.initSocketMessage("10041"));
        },
        /**
         * 暂未使用
         */
        getBossLevelConfig: function() {
            return;
            this.sendMessage(this.initSocketMessage("10040"));
        },
        /**
         * 是否为活动up妖灵
         */
        up: function(val) {
            return this.upYaolings.hasOwnProperty(val) && this.upYaolings[val];
        },
        copySuccess() {
            this.$message.success("复制成功");
        },
        openWideDialog() {
            this.wideDialogSettings.enable = this.mode;
            // this.wideDialogSettings.thread = this.thread;
            this.wideDialogSettings.range = this.range;
            this.wideDialogVisible = true;
        },
        saveWideSettings() {
            this.mode = this.wideDialogSettings.enable;
            if (this.mode == "temp") {
                this.thread = 1;
                this.range = 0;
            } else {
                // this.thread = this.wideDialogSettings.thread;
                this.range = this.wideDialogSettings.range;
            }
            this.max_range = this.range * 2 + 1;
            this.wideDialogVisible = false;
        },
        formatTooltip(val) {
            return Math.pow(val * 2 + 1, 2) + "格";
        },
        goMap() {
            this.geocoder.getLocation(this.address);
        },
        stopSearch() {
            if (this.radarTask) {
                this.radarTask.closeTasks();
            }
            for (let s of this.sockets) {
                if (s && s.socket) {
                    if (s.timeout) {
                        clearTimeout(s.timeout);
                    }
                    delete s.task;
                    s.socket.close();
                }
            }
            this.searching = false;
            this.progressShow = false;
        }
    },
    computed: {
        fit: function() {
            let ans = [];

            //自定义筛选优先级最高
            if (this.settings.use_custom) {
                return Array.from(
                    new Set(
                        this.settings.custom_filter
                            .filter(item => item.on)
                            .map(item => item.id)
                    )
                );
            }

            if (this.mode === "normal") {
                let _fit = this.settings.fit;
                if (_fit.all) {
                    return ["special"];
                }

                // 根据值把key转换成FILTER_FISH这种，取常量配置中的值
                for (let _f in _fit) {
                    if (_fit[_f]) {
                        let _arr = FILTER[`FILTER_${_f.toLocaleUpperCase()}`];
                        ans = ans.concat(_arr);
                    }
                }
                return Array.from(new Set(ans));
            } else {
                let _fit = this.settings.wide
                    .filter(item => item.on)
                    .map(item => item.id);
                return Array.from(new Set(_fit));
            }
        },
        /**
         * 大范围进度条
         */
        progressPercent: function() {
            let result = 0;
            if (this.radarTask) {
                let tasks = this.radarTask.tasks;
                let _close = tasks.filter(i => {
                    return i.status === "close";
                }).length;
                result = Math.floor((_close / tasks.length) * 100);
            }
            // let _open = tasks.length
            return result;
        },
        /**
         * 已完成任务数
         */
        closedTask: function() {
            let result = 0;
            if (this.radarTask) {
                result = this.radarTask.tasks.filter(i => {
                    return i.status === "close";
                }).length;
            }
            return result;
        }
    },
    watch: {
        settings: {
            handler: function(newV, oldV) {
                console.log("settings update...");
                setLocalStorage("radar_settings", this.settings);
            },
            deep: true
        }
    }
};
</script>
<style lang='less'>
#app {
    // padding-left: 200px;
    position: relative;
    height: 100%;
}
.c-link {
    cursor: pointer;
    text-decoration: underline;
    color: #409eff;
}
.search-input {
    width: 150px !important;
    .el-input-group__append {
        padding: 0 10px;
    }
}
@media (max-width: 768px) {
    .test {
        .el-dialog {
            width: 80%;
        }
    }
}
</style>

