<template>
<tr-dashboard :title="filter.dateRange ? '历史成交' : '当日成交'">
    <div slot="dashboard-header">
         <tr-dashboard-header-item>
            <i class="fa fa-refresh mouse-over" title="刷新" @click="handleRefresh"></i>
        </tr-dashboard-header-item>
        <tr-dashboard-header-item width="101px">
            <el-input 
                size="mini"
                placeholder="关键字"
                prefix-icon="el-icon-search"
                v-model.trim="searchKeyword"
                >
            </el-input>
        </tr-dashboard-header-item>
        <tr-dashboard-header-item width="199px">
            <el-date-picker
                v-model.trim="filter.dateRange"
                size="mini"
                type="daterange"
                range-separator="～"
                start-placeholder="开始日期"
                end-placeholder="结束日期">
            </el-date-picker>
        </tr-dashboard-header-item>
    </div>
    <tr-table
        v-if="rendererTable"
        :data="tableData"
        :schema="schema"
        :renderCellClass="renderCellClass"
    ></tr-table>
</tr-dashboard>

</template>

<script>
import {mapGetters} from 'vuex'
import moment from 'moment'
import {offsetName} from '@/assets/config/tradingConfig'
import { debounce, throttleInsert, throttle } from "@/assets/js/utils"
export default {
    name: 'trades-record',
    props: {
        currentId: {
            type: String,
            default:''
        },
        pageType: {
            type: String,
            default:''
        },
        getDataMethod: {
            type: Function,
            default: () => {}
        },
        //接收推送的数据
        nanomsgBackData: {
            type: Object,
            default: {}
        }
    },

    data() {
        this.throttleInsertTrade = throttleInsert(500, 'unshift')
        return {
            rendererTable: false,
            searchKeyword: '',
            filter: {
                id: '',
                dateRange: null
            },
            getDataLock: false,
            tableData: Object.freeze([])
        }
    },

    computed:{
        schema(){
            return [{
                type: 'text',
                label: '成交时间',
                prop: 'tradeTime',
                width: '160px'
            },{
                type: 'text',
                label: '代码',
                prop: 'instrumentId',
                width: '70px'
            },{
                type: 'text',
                label: '买卖',
                prop: 'side',
            },{
                type: 'text',
                label: '开平',
                prop: 'offset',
            },{
                type: 'text',
                label: '成交价',
                prop: 'price',
            },{
                type: 'text',
                label: '成交量',
                prop: 'volume',
            },{
                type: 'text',
                label: this.pageType == 'account' ? '策略': '账户',
                prop: this.pageType == 'account' ? 'clientId': 'accountId',
            }]
        }
    },

    watch: {
        searchKeyword: debounce(function(val) {
            const t = this;
            t.filter.id = val;
        }),

        filter:{
            deep: true,
            handler() {
                const t = this;
                t.currentId && t.init(true)
            }
        },
        
        currentId(val) {
            const t = this;
            t.resetData();
            if(!val) return;
            t.rendererTable = false;
            t.$nextTick().then(() => {
                t.rendererTable = true;
                t.init();
            })
        },

        //接收推送返回的数据
        nanomsgBackData(val) {
            const t = this;
            if(!val || t.getDataLock) return
            t.dealNanomsg(val)
        }

    },

    mounted(){
        const t = this;
        t.rendererTable = true;
        t.resetData();
        t.currentId && t.init();
    },

    methods:{
        handleRefresh(){
            const t = this;
            t.resetData();
            t.currentId && t.init();
        },

        //重置数据
        resetData() {
            const t = this;
            this.searchKeyword = '';
            this.filter = {
                id: '',
                dateRange: null
            };
            t.getDataLock = false;
            t.tableData = Object.freeze([]);
            return true;
        },

        init: debounce(function() {
            const t = this
            t.$emit('startNanomsg')
            t.getData()
        }),

        //首次获取数据
        getData() {
            const t = this
            if(t.getDataLock) return new Promise((resolve, reject) => reject(new Error('get-data-lock')));
            //获得获取数据的方法名
            t.getDataLock = true
            t.tableData = Object.freeze([])
            //id:用户或者交易id，filter：需要筛选的数据
            return t.getDataMethod(t.currentId, t.filter).then(res => {
                if(!res || !res.data.length) {
                    t.tableData = Object.freeze([])
                    return;
                }
                t.tableData = Object.freeze(t.dealData(res.data))
            }).finally(() => t.getDataLock = false)
        },

        //对返回的数据进行处理
        dealData(data) {
            const t = this
            const historyData = data || []
            const tableData = historyData.map(item => (t.dealTrade(item)))
            return tableData
        },

        //处理需要的数据及顺序
        dealTrade(item) {
            return {
                id: item.account_id.toString() + '_' + item.id.toString() + '_' + item.trade_time.toString(),
                tradeTime: item.trade_time && moment(item.trade_time/1000000).format('YYYY-MM-DD HH:mm:ss'),
                instrumentId: item.instrument_id,
                side: item.side ? (item.side == '0' ? '买' : '卖') : '',
                offset: offsetName[item.offset],
                price: item.price,
                volume: item.volume,
                clientId: item.client_id,
                accountId: item.account_id
            }     
        },

        dealNanomsg(data) {
            const t = this
            //当有日期筛选的时候，不需要推送数据
            if(t.filter.dateRange) return
            //如果存在筛选，则推送的数据也需要满足筛选条件
            const {id, dateRange} = t.filter
            const {trade_time} = data
            if(!((data.instrument_id.includes(id) || data.client_id.includes(id)) )) return
            const tradeData = t.dealTrade(data)
            t.throttleInsertTrade(tradeData).then(tradeList => {
                if(!tradeList) return;
                let oldTableData = t.tableData.slice(0, 500);
                oldTableData = [...tradeList, ...oldTableData]
                //更新数据
                t.tableData = Object.freeze(oldTableData)
            })
           
        },

        renderCellClass(prop, item){
            if(prop === 'side'){
                if(item.side === '买') return 'red'
                else if(item.side === '卖') return 'green'
            }
            if(prop === 'offset'){
                if(item.offset === '开仓') return 'red'
                else if(item.offset === '平仓') return 'green'
            }
        }
    }
}
</script>

<style lang="scss">
.trades-record{
    height: 100%;
}
</style>