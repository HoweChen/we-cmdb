<template>
  <div>
    <Row>
      <Col span="6">
        <span style="margin-right: 10px">根CI类型</span>
        <Select
          v-model="selectedCI"
          filterable
          style="width: 75%;"
          @on-change="onCITypeChange"
        >
          <Option
            v-for="item in allCiTypes"
            :value="item.ciTypeId"
            :key="item.ciTypeId"
            >{{ item.name }}</Option
          >
        </Select>
      </Col>
      <Col span="17" offset="1">
        <span style="margin-right: 10px">综合查询名称</span>
        <Select
          v-model="selectedQueryName"
          filterable
          style="width: 75%;"
          :disabled="!selectedCI"
          @on-change="onQueryNameSelectChange"
        >
          <Option
            v-for="item in queryNameList"
            :value="item.id"
            :key="item.id"
            >{{ item.name }}</Option
          >
        </Select>
      </Col>
    </Row>
    <Row v-if="!!selectedQueryName" style="margin-top: 20px;">
      <WeTable
        :tableData="tableData"
        :tableOuterActions="outerActions"
        :tableInnerActions="innerActions"
        :tableColumns="tableColumns"
        :pagination="pagination"
        :showCheckbox="false"
        @actionFun="actionFun"
        @pageChange="pageChange"
        @pageSizeChange="pageSizeChange"
        @sortHandler="sortHandler"
        @handleSubmit="handleSubmit"
        tableHeight="650"
        ref="table"
      ></WeTable>

      <Modal v-model="originDataModal" title="原始数据" footer-hide>
        <highlight-code lang="json">{{ showRowOriginData }}</highlight-code>
      </Modal>

      <Modal
        v-model="filtersAndResultModal"
        title="报文"
        footer-hide
        width="75"
      >
        <div style="max-height: 600px; overflow: auto;">
          <Row
            >请求URL:
            <highlight-code lang="json">{{ requestURL }}</highlight-code>
          </Row>
          <Row>
            <Col span="11"
              >Payload:
              <highlight-code lang="json">{{ payload }}</highlight-code>
            </Col>
            <Col span="12" offset="1"
              >Result:
              <highlight-code lang="json">{{ tableData }}</highlight-code>
            </Col>
          </Row>
        </div>
      </Modal>
    </Row>
  </div>
</template>

<script>
import {
  getAllCITypes,
  getQueryNames,
  queryIntHeader,
  excuteIntQuery,
  getEnumCodesByCategoryId
} from "@/api/server";
import { components } from "../../const/actions.js";
const innerActions = [
  {
    label: "原始数据",
    props: {
      type: "info",
      size: "small"
    },
    actionType: "showOriginData"
  }
];
const outerActions = [
  {
    label: "显示报文",
    props: {
      type: "success",
      icon: "ios-eye",
      disabled: false
    },
    actionType: "showFiltersAndResult"
  }
];
export default {
  components: {},
  data() {
    return {
      selectedCI: "",
      allCiTypes: [],
      selectedQueryName: "",
      queryNameList: [],
      tableData: [],
      tableColumns: [],
      innerActions,
      outerActions,
      pagination: {
        pageSize: 10,
        currentPage: 1,
        total: 0
      },
      payload: {
        filters: [],
        pageable: {
          pageSize: 10,
          startIndex: 0
        },
        paging: true
        // sorting: {
        //   asc: true,
        //   field: ""
        // }
      },
      originDataModal: false,
      showRowOriginData: "",
      filtersAndResultModal: false,
      showfiltersAndResultModalData: "",
      requestURL: ""
    };
  },
  created() {
    this.getAllCITypes();
  },
  methods: {
    pageChange(current) {
      this.pagination.currentPage = current;
      this.getTableData();
    },
    pageSizeChange(size) {
      this.pagination.pageSize = size;
      this.getTableData();
    },
    async getAllCITypes() {
      let { status, data, message } = await getAllCITypes();
      if (status === "OK") {
        this.allCiTypes = data;
      }
    },
    onCITypeChange(value) {
      value && this.getQueryNameList(value);
    },
    onQueryNameSelectChange(value) {
      if (value) {
        this.getTableHeader(value);
        this.requestURL = `/wecmdb/intQuery/${value}/execute`;
      }
    },
    async getTableHeader(id) {
      this.currentSelectQueryNameId = id;
      let { status, data, message } = await queryIntHeader(id);

      if (status === "OK") {
        this.tableColumns = [];
        data.forEach(_ => {
          if (_.attrUnits) {
            let children = _.attrUnits
              .map(child => {
                return {
                  title: child.attr.name,
                  parentTitle: _.ciTypeName,
                  key: child.attrKey,
                  inputKey: child.attrKey,
                  type: "text",
                  ...components[child.attr.inputType],
                  placeholder: child.attr.name,
                  ...child.attr,
                  ciType: { id: child.attr.referenceId, name: "" }
                };
              })
              .filter(i => i.isDisplayed);
            this.tableColumns.push({
              title: _.ciTypeName,
              align: "center",
              children
            });
          }
        });
        this.getTableData();
        this.$nextTick(() => {
          this.getColumnOptions();
        });
      }
    },
    getColumnOptions() {
      this.tableColumns.forEach(_ => {
        if (_.children) {
          _.children.forEach(async child => {
            if (child.inputType === "select") {
              const { data, status, message } = await getEnumCodesByCategoryId(
                0,
                child.referenceId
              );
              let opts = [];
              if (status === "OK") {
                opts = data
                  .filter(i => i.status === "active")
                  .map(_ => {
                    return {
                      value: _.codeId,
                      label: _.value
                    };
                  });
              }
              this.$set(child, "options", opts);
            }
          });
        }
      });
    },
    async getTableData() {
      this.payload.pageable.pageSize = this.pagination.pageSize;
      this.payload.pageable.startIndex =
        (this.pagination.currentPage - 1) * this.pagination.pageSize;
      let { status, data, message } = await excuteIntQuery(
        this.currentSelectQueryNameId,
        this.payload
      );
      if (status === "OK") {
        this.tableData = data.contents;
        this.pagination.total = data.pageInfo.totalRows;
      }
    },
    reset() {
      this.queryNameList = [];
      this.selectedQueryName = "";
    },
    async getQueryNameList(ciTypeId) {
      this.reset();
      let { status, data, message } = await getQueryNames(ciTypeId);
      if (status === "OK") {
        this.queryNameList = data;
      }
    },
    sortHandler(data) {
      this.payload.sorting = {
        asc: data.order === "asc",
        field: data.key
      };
      this.getTableData();
    },

    handleSubmit(data) {
      this.payload.filters = data;
      this.pagination.currentPage = 1;
      this.getTableData();
    },
    actionFun(type, data) {
      if (type === "showOriginData") {
        this.showRowOriginData = data.weTableForm;
        this.originDataModal = true;
      }
      if (type === "showFiltersAndResult") {
        this.filtersAndResultModal = true;
      }
    }
  }
};
</script>

<style lang="scss" scoped></style>
