<template>
  <a-card :bordered="false">

    <!-- 查询区域 -->
    <div class="table-page-search-wrapper">
      <a-form layout="inline" @keyup.enter.native="searchQuery">
        <a-row :gutter="24">
          <a-col :xl="6" :lg="7" :md="8" :sm="24">
            <a-form-item label="公司名称">
              <!--<a-input placeholder="请输入公司名称" v-model="queryParam.companyName"></a-input>-->
              <a-select v-model="queryParam.companyId" placeholder="请选择公司编号" show-search :filter-option="filterOption">
                <a-select-option v-for="item in companyList" :key="item.id" :value="item.id">{{item.companyName}}</a-select-option>
              </a-select>
            </a-form-item>
          </a-col>
          <a-col :xl="6" :lg="7" :md="8" :sm="24">
            <span style="float: left;overflow: hidden;" class="table-page-search-submitButtons">
              <a-button type="primary" @click="searchQuery" icon="search">查询</a-button>
              <a-button type="primary" @click="searchReset" icon="reload" style="margin-left: 8px">重置</a-button>
              <a @click="handleToggleSearch" style="margin-left: 8px">
                {{ toggleSearchStatus ? '收起' : '展开' }}
                <a-icon :type="toggleSearchStatus ? 'up' : 'down'"/>
              </a>
            </span>
          </a-col>
        </a-row>
      </a-form>
    </div>


    <!-- 操作按钮区域 -->
    <div class="table-operator">
      <a-button @click="handleAdd" type="primary" icon="plus">新增</a-button>
      <a-button type="primary" icon="download" @click="handleExportXls('公司表')">导出</a-button>
      <a-upload name="file" :showUploadList="false" :multiple="false" :headers="tokenHeader" :action="importExcelUrl" @change="handleImportExcel">
        <a-button type="primary" icon="import">导入</a-button>
      </a-upload>
      <!--<a-dropdown v-if="selectedRowKeys.length > 0">
        <a-menu slot="overlay">
          <a-menu-item key="1" @click="batchDel"><a-icon type="delete"/>删除</a-menu-item>
        </a-menu>
        <a-button style="margin-left: 8px"> 批量操作 <a-icon type="down" /></a-button>
      </a-dropdown>-->
    </div>

    <!-- table区域-begin -->
    <div>

      <a-table
        ref="table"
        size="middle"
        bordered
        rowKey="id"
        :columns="columns"
        :dataSource="dataSource"
        :pagination="ipagination"
        :loading="loading"
        :rowSelection="{selectedRowKeys: selectedRowKeys, onChange: onSelectChange,type:type}"
        class="j-table-force-nowrap"
        @change="handleTableChange">

        <template slot="htmlSlot" slot-scope="text">
          <div v-html="text"></div>
        </template>
        <template slot="imgSlot" slot-scope="text">
          <span v-if="!text" style="font-size: 12px;font-style: italic;">无图片</span>
          <img v-else :src="getImgView(text)" height="25px" alt="" style="max-width:80px;font-size: 12px;font-style: italic;"/>
        </template>
        <template slot="pcaSlot" slot-scope="text">
          <div>{{ getPcaText(text) }}</div>
        </template>
        <template slot="fileSlot" slot-scope="text">
          <span v-if="!text" style="font-size: 12px;font-style: italic;">无文件</span>
          <a-button
            v-else
            :ghost="true"
            type="primary"
            icon="download"
            size="small"
            @click="uploadFile(text)">
            下载
          </a-button>
        </template>

        <span slot="action" slot-scope="text, record">
          <a @click="handleEdit(record)">编辑</a>

          <a-divider type="vertical" />
          <a-dropdown>
            <a class="ant-dropdown-link">更多 <a-icon type="down" /></a>
            <a-menu slot="overlay">
              <a-menu-item>
                <a-popconfirm title="确定删除吗?" @confirm="() => handleDelete(record.id)">
                  <a>删除</a>
                </a-popconfirm>
              </a-menu-item>
            </a-menu>
          </a-dropdown>
        </span>

      </a-table>
    </div>
    <!-- table区域-end -->

    <a-tabs defaultActiveKey="1">
      <a-tab-pane tab="部件列表" key="1">
        <!--<Jeecg-Order-Customer-List ref="JeecgOrderCustomerList"></Jeecg-Order-Customer-List>-->
        <!--<mpi-component-list ref="MpiComponentList"></mpi-component-list>-->
        <mpi-equipment-component-list ref="MpiEquipmentComponentList"></mpi-equipment-component-list>
      </a-tab-pane>
    </a-tabs>

    <!-- 表单区域 -->
    <!--<jeecgOrderDMain-modal ref="modalForm" @ok="modalFormOk"></jeecgOrderDMain-modal>-->
    <!--<MpiEquipmentComponentModal ref="modalForm" @ok="modalFormOk"></MpiEquipmentComponentModal>-->
    <mpiEquipment-modal ref="modalForm" @ok="modalFormOk"></mpiEquipment-modal>

  </a-card>
</template>

<script>

  import {deleteAction} from '@/api/manage'
  import { mixinDevice } from '@/utils/mixin'
  import { JeecgListMixin } from '@/mixins/JeecgListMixin'
  import MpiEquipmentModal from '../equipment/modules/MpiEquipmentModal'

  import MpiEquipmentComponentList from '@/views/platform/equipmentComponent/MpiEquipmentComponentList'

  import {filterMultiDictText} from '@/components/dict/JDictSelectUtil'
  import Area from '@/components/_util/Area'
  import { loadCategoryData } from '@/api/api'
  import { getAction } from '@/api/manage'

  import '@/assets/less/TableExpand.less'

  export default {
    name: "EquipmentList1",
    mixins: [JeecgListMixin, mixinDevice],
    components: {
      /*JeecgOrderTicketModal,
      JeecgOrderCustomerModal,
      JeecgOrderDMainModal,
      JeecgOrderCustomerList,
      JeecgOrderTicketList,*/
      MpiEquipmentModal,
      MpiEquipmentComponentList
    },
    data() {
      return {
        /* 公司信息 */
        companyList: [],
        description: '设备管理页面',
        /* 分页参数 */
        ipagination:{
          current: 1,
          pageSize: 5,
          pageSizeOptions: ['5', '10', '20'],
          showTotal: (total, range) => {
            return range[0] + "-" + range[1] + " 共" + total + "条"
          },
          showQuickJumper: true,
          showSizeChanger: true,
          total: 0
        },
        // 表头
        columns: [
          {
            title: '#',
            dataIndex: '',
            key:'rowIndex',
            width:60,
            align:"center",
            customRender:function (t,r,index) {
              return parseInt(index)+1;
            }
          },
          {
            title:'设备名称',
            align:"center",
            dataIndex: 'equipmentName'
          },
          {
            title:'网关名称', //title:'网关编号',
            align:"center",
            dataIndex: 'gatewayName' //dataIndex: 'gatewayId_dictText'
          },
          {
            title:'设备类型',
            align:"center",
            dataIndex: 'equipmentType',
            customRender: (text) => (text ? filterMultiDictText(this.dictOptions['equipmentType'], text) : '')
          },
          /*{
            title:'一级地区',
            align:"center",
            dataIndex: 'level1Region',
            scopedSlots: {customRender: 'pcaSlot'}
          },
          {
            title:'二级地区',
            align:"center",
            dataIndex: 'level2Region',
            scopedSlots: {customRender: 'pcaSlot'}
          },
          {
            title:'三级地区',
            align:"center",
            dataIndex: 'level3Region',
            scopedSlots: {customRender: 'pcaSlot'}
          },*/
          {
            title:'安装时间',
            align:"center",
            dataIndex: 'deployTime'
          },
          {
            title:'安装地址',
            align:"center",
            dataIndex: 'deployAddress'
          },
          {
            title:'经度',
            align:"center",
            dataIndex: 'longitude'
          },
          {
            title:'纬度',
            align:"center",
            dataIndex: 'latitude'
          },
          {
            title:'通讯方式',
            align:"center",
            dataIndex: 'comType'
          },
          {
            title:'产能系数',
            align:"center",
            dataIndex: 'productCoefficient'
          },
          {
            title:'设备状态',
            align:"center",
            dataIndex: 'status_dictText'
          },
          {
            title:'维护规则编号',
            align:"center",
            dataIndex: 'maintainRule'
          },
          {
            title:'设备图片',
            align:"center",
            dataIndex: 'equipmentImage',
            scopedSlots: {customRender: 'imgSlot'}
          },
          {
            title:'设备评分',
            align:"center",
            dataIndex: 'equipmentScore'
          },
          {
            title:'备注',
            align:"center",
            dataIndex: 'remarks',
            scopedSlots: { customRender: 'component' }
          },
          /*{
            title:'创建人',
            align:"center",
            dataIndex: 'createBy'
          },
          {
            title:'创建时间',
            align:"center",
            dataIndex: 'createTime'
          },
          {
            title:'修改人',
            align:"center",
            dataIndex: 'updateBy'
          },
          {
            title:'修改时间',
            align:"center",
            dataIndex: 'updateTime'
          },*/
          /*{
            title:'删除标识0-正常,1-已删除',
            align:"center",
            dataIndex: 'delFlag_dictText'
          },*/
          {
            title: '操作',
            dataIndex: 'action',
            align:"center",
            // fixed:"right",
            width:147,
            scopedSlots: { customRender: 'action' }
          }
        ],
        // 分页参数
        type: "radio",
        url: {
          list: "/system/mpiEquipment/list",
          delete: "/system/mpiEquipment/delete",
          deleteBatch: "/system/mpiEquipment/deleteBatch",
          exportXlsUrl: "/system/mpiEquipment/exportXls",
          importExcelUrl: "system/mpiEquipment/importExcel",
          queryCompanyList: "/system/mpiCompany/allList" // 全查询公司信息
        },
        dictOptions:{},
        pcaData:''
      }
    },
    computed: {
      importExcelUrl: function(){
        return `${window._CONFIG['domianURL']}/${this.url.importExcelUrl}`;
      },
    },
    created() {
      this.pcaData = new Area()
    },
    mounted() {
      this.queryCompanyList()
    },
    methods: {
      filterOption(input, option) {
        return (
          option.componentOptions.children[0].text.toLowerCase().indexOf(input.toLowerCase()) >= 0
        );
      },
      /* 全查询公司信息 */
      queryCompanyList() {
        getAction(this.url.queryCompanyList,null).then((res)=>{
          if(res.success){
            /*that.$message.success(res.message);
            that.$emit('ok');*/
            //console.log("result : " + JSON.stringify(res.result))
            this.companyList = res.result
          }else{
            that.$message.warning(res.message);
          }
        })
      },
      getPcaText(code){
        return this.pcaData.getText(code);
      },
      initDictConfig(){
        loadCategoryData({code:'B03'}).then((res) => {
          if (res.success) {
            this.$set(this.dictOptions, 'equipmentType', res.result)
          }
        })
      },
      clickThenCheck(record) {
        return {
          on: {
            click: () => {
              this.onSelectChange(record.id.split(","), [record]);
            }
          }
        };
      },
      onSelectChange(selectedRowKeys, selectionRows) {
        this.selectedRowKeys = selectedRowKeys;
        this.selectionRows = selectionRows;
        //console.log("id : " + selectedRowKeys)
        //console.log("id1 : " + selectionRows)
        this.$refs.MpiEquipmentComponentList.getComponentMain(this.selectedRowKeys[0]); //改变子列表
        //this.$refs.JeecgOrderCustomerList.getOrderMain(this.selectedRowKeys[0]);
        //this.$refs.JeecgOrderTicketList.getOrderMain(this.selectedRowKeys[0]);
      },
      onClearSelected() {
        this.selectedRowKeys = [];
        this.selectionRows = [];
        this.$refs.JeecgOrderCustomerList.queryParam.mainId = null;
        this.$refs.JeecgOrderTicketList.queryParam.mainId = null;
        this.$refs.JeecgOrderCustomerList.loadData();
        this.$refs.JeecgOrderTicketList.loadData();
        this.$refs.JeecgOrderCustomerList.selectedRowKeys = [];
        this.$refs.JeecgOrderCustomerList.selectionRows = [];
        this.$refs.JeecgOrderTicketList.selectedRowKeys = [];
        this.$refs.JeecgOrderTicketList.selectionRows = [];
      },

      handleDelete: function (id) {
        var that = this;
        deleteAction(that.url.delete, {id: id}).then((res) => {
          if (res.success) {
            that.$message.success(res.message);
            that.loadData();
            this.$refs.MpiGatewayList.loadData() // 重新加载子组件列表
            //this.$refs.JeecgOrderCustomerList.loadData();
            //this.$refs.JeecgOrderTicketList.loadData();
          } else {
            that.$message.warning(res.message);
          }
        });
      },
      searchQuery:function(){
        this.selectedRowKeys = [];
        this.selectionRows = [];
        //this.$refs.JeecgOrderCustomerList.queryParam.mainId = null;
        //this.$refs.JeecgOrderTicketList.queryParam.mainId = null;
        //this.$refs.MpiEquipmentComponentList.loadData();
        //console.log("CompanyId : " + this.queryParam.companyId)
        this.$refs.MpiEquipmentComponentList.getEquipmentIdsByCompany(this.queryParam.companyId)
        // this.$refs.JeecgOrderTicketList.loadData();
        //this.$refs.JeecgOrderCustomerList.selectedRowKeys = [];
        //this.$refs.JeecgOrderCustomerList.selectionRows = [];
        //this.$refs.JeecgOrderTicketList.selectedRowKeys = [];
        //this.$refs.JeecgOrderTicketList.selectionRows = [];
        this.loadData();
      },
      searchReset() {
        this.queryParam = {}
        this.$refs.MpiEquipmentComponentList.getEquipmentIdsByCompany("")
        this.loadData(1);
      },
      loadData(arg) {
        if(!this.url.list){
          this.$message.error("请设置url.list属性!")
          return
        }
        //加载数据 若传入参数1则加载第一页的内容
        if (arg === 1) {
          this.ipagination.current = 1;
        }
        var params = this.getQueryParams();//查询条件
        this.loading = true;
        //console.log("查询条件 ：" + JSON.stringify(params))
        getAction(this.url.list, params).then((res) => {
          if (res.success) {
            this.dataSource = res.result.records;
            this.ipagination.total = res.result.total;
          }
          if(res.code===510){
            this.$message.warning(res.message)
          }
          this.loading = false;
        })
      },
    }
  }
</script>
<style scoped>
  .ant-card-body .table-operator {
    margin-bottom: 18px;
  }

  .ant-table-tbody .ant-table-row td {
    padding-top: 15px;
    padding-bottom: 15px;
  }

  .anty-row-operator button {
    margin: 0 5px
  }

  .ant-btn-danger {
    background-color: #ffffff
  }

  .ant-modal-cust-warp {
    height: 100%
  }

  .ant-modal-cust-warp .ant-modal-body {
    height: calc(100% - 110px) !important;
    overflow-y: auto
  }

  .ant-modal-cust-warp .ant-modal-content {
    height: 90% !important;
    overflow-y: hidden
  }
</style>