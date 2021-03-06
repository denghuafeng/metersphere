<template>
  <div class="card-container">
    <el-card class="card-content" v-loading="result.loading">
      <template v-slot:header>
        <ms-table-header :is-tester-permission="true" :condition.sync="condition" @search="initTableData"
                         :show-create="false" :tip="$t('commons.search_by_name_or_id')">
          <template v-slot:title>
            <node-breadcrumb class="table-title" :nodes="selectParentNodes" @refresh="refresh" :title="$t('test_track.review_view.all_review')"/>
          </template>
          <template v-slot:button>
            <ms-table-button :is-tester-permission="true" icon="el-icon-video-play"
                             :content="$t('test_track.review_view.start_review')" @click="startReview"/>
            <ms-table-button :is-tester-permission="true" icon="el-icon-connection"
                             :content="$t('test_track.review_view.relevance_case')"
                             @click="$emit('openTestReviewRelevanceDialog')"/>
          </template>
        </ms-table-header>
      </template>

      <executor-edit ref="executorEdit" :select-ids="new Set(Array.from(this.selectRows).map(row => row.id))"
                     @refresh="initTableData"/>
      <status-edit ref="statusEdit" :plan-id="reviewId"
                   :select-ids="new Set(Array.from(this.selectRows).map(row => row.id))" @refresh="initTableData"/>

      <el-table
        class="adjust-table"
        border
        @select-all="handleSelectAll"
        @filter-change="filter"
        @sort-change="sort"
        @select="handleSelectionChange"
        row-key="id"
        @row-click="showDetail"
        :data="tableData">

        <el-table-column
          type="selection"/>
        <el-table-column width="40" :resizable="false" align="center">
          <template v-slot:default="scope">
<!--            <show-more-btn :is-show="scope.row.showMore" :buttons="buttons" :size="selectRows.size"/>-->
          </template>
        </el-table-column>
        <el-table-column
          prop="num"
          sortable="custom"
          :label="$t('commons.id')"
          show-overflow-tooltip>
        </el-table-column>
        <el-table-column
          prop="name"
          :label="$t('commons.name')"
          show-overflow-tooltip>
        </el-table-column>
        <el-table-column
          prop="priority"
          :filters="priorityFilters"
          column-key="priority"
          :label="$t('test_track.case.priority')">
          <template v-slot:default="scope">
            <priority-table-item :value="scope.row.priority" ref="priority"/>
          </template>
        </el-table-column>

        <el-table-column
          prop="type"
          :filters="typeFilters"
          column-key="type"
          :label="$t('test_track.case.type')"
          show-overflow-tooltip>
          <template v-slot:default="scope">
            <type-table-item :value="scope.row.type"/>
          </template>
        </el-table-column>

        <el-table-column
          prop="method"
          :filters="methodFilters"
          column-key="method"
          :label="$t('test_track.case.method')"
          show-overflow-tooltip>
          <template v-slot:default="scope">
            <method-table-item :value="scope.row.method"/>
          </template>
        </el-table-column>

        <el-table-column
          prop="nodePath"
          :label="$t('test_track.case.module')"
          show-overflow-tooltip>
        </el-table-column>

        <el-table-column
          prop="projectName"
          :label="$t('test_track.review.review_project')"
          show-overflow-tooltip>
        </el-table-column>

        <el-table-column
          prop="reviewerName"
          :label="$t('test_track.review.review_creator')"
          show-overflow-tooltip
        >
        </el-table-column>

        <el-table-column
          prop="status"
          :filters="statusFilters"
          column-key="status"
          :label="$t('test_track.review_view.execute_result')">
          <template v-slot:default="scope">
            <span class="el-dropdown-link">
              <status-table-item :value="scope.row.status"/>
            </span>
          </template>
        </el-table-column>

        <el-table-column
          sortable
          prop="updateTime"
          :label="$t('commons.update_time')"
          show-overflow-tooltip>
          <template v-slot:default="scope">
            <span>{{ scope.row.updateTime | timestampFormatDate }}</span>
          </template>
        </el-table-column>
        <el-table-column
          :label="$t('commons.operating')">
          <template v-slot:default="scope">
            <ms-table-operator-button :is-tester-permission="true" :tip="$t('commons.edit')" icon="el-icon-edit"
                                      @exec="handleEdit(scope.row)"/>
            <ms-table-operator-button :is-tester-permission="true" :tip="$t('test_track.plan_view.cancel_relevance')"
                                      icon="el-icon-unlock" type="danger" @exec="handleDelete(scope.row)"/>
          </template>
        </el-table-column>
      </el-table>

      <ms-table-pagination :change="search" :current-page.sync="currentPage" :page-size.sync="pageSize"
                           :total="total"/>

      <test-review-test-case-edit
        ref="testReviewTestCaseEdit"
        :search-param="condition"
        @refresh="initTableData"
        :is-read-only="isReadOnly"
        @refreshTable="search"/>

    </el-card>
    <batch-edit ref="batchEdit" @batchEdit="batchEdit"
                :type-arr="typeArr" :value-arr="valueArr" :dialog-title="$t('test_track.case.batch_edit_case')"/>
  </div>
</template>

<script>

import ClassicEditor from "@ckeditor/ckeditor5-build-classic";
import MsTableOperatorButton from "../../../../common/components/MsTableOperatorButton";
import MsTableOperator from "../../../../common/components/MsTableOperator";
import MethodTableItem from "../../../common/tableItems/planview/MethodTableItem";
import TypeTableItem from "../../../common/tableItems/planview/TypeTableItem";
import StatusTableItem from "../../../common/tableItems/planview/StatusTableItem";
import PriorityTableItem from "../../../common/tableItems/planview/PriorityTableItem";
import StatusEdit from "../../../plan/view/comonents/StatusEdit";
import ExecutorEdit from "../../../plan/view/comonents/ExecutorEdit";
import MsTipButton from "../../../../common/components/MsTipButton";
import MsTableHeader from "../../../../common/components/MsTableHeader";
import NodeBreadcrumb from "../../../common/NodeBreadcrumb";
import MsTableButton from "../../../../common/components/MsTableButton";
import ShowMoreBtn from "../../../case/components/ShowMoreBtn";
import BatchEdit from "../../../case/components/BatchEdit";
import MsTablePagination from '../../../../common/pagination/TablePagination';
import {_filter, _sort, checkoutTestManagerOrTestUser, hasRoles} from "../../../../../../common/js/utils";
import {TEST_CASE_CONFIGS} from "../../../../common/components/search/search-components";
import {ROLE_TEST_MANAGER, ROLE_TEST_USER, TokenKey, WORKSPACE_ID} from "../../../../../../common/js/constants";
import TestReviewTestCaseEdit from "./TestReviewTestCaseEdit";

export default {
  name: "TestReviewTestCaseList",
  components: {
    MsTableOperatorButton, MsTableOperator, MethodTableItem, TypeTableItem,
    StatusTableItem, PriorityTableItem, StatusEdit,
    ExecutorEdit, MsTipButton, TestReviewTestCaseEdit, MsTableHeader,
    NodeBreadcrumb, MsTableButton, ShowMoreBtn, BatchEdit, MsTablePagination
  },
  data() {
    return {
      result: {},
      condition: {},
      tableData: [],
      currentPage: 1,
      pageSize: 10,
      total: 0,
      selectRows: new Set(),
      testReview: {},
      isReadOnly: false,
      isTestManagerOrTestUser: false,
      priorityFilters: [
        {text: 'P0', value: 'P0'},
        {text: 'P1', value: 'P1'},
        {text: 'P2', value: 'P2'},
        {text: 'P3', value: 'P3'}
      ],
      methodFilters: [
        {text: this.$t('test_track.case.manual'), value: 'manual'},
        {text: this.$t('test_track.case.auto'), value: 'auto'}
      ],
      typeFilters: [
        {text: this.$t('commons.functional'), value: 'functional'},
        {text: this.$t('commons.performance'), value: 'performance'},
        {text: this.$t('commons.api'), value: 'api'}
      ],
      statusFilters: [
        {text: this.$t('test_track.plan.plan_status_prepare'), value: 'Prepare'},
        {text: this.$t('test_track.plan_view.pass'), value: 'Pass'},
        {text: '未通过', value: 'UnPass'},
      ],
      showMore: false,
      buttons: [
        {
          name: this.$t('test_track.case.batch_edit_case'), handleClick: this.handleBatchEdit
        },
        {
          name: this.$t('test_track.case.batch_unlink'), handleClick: this.handleDeleteBatch
        }
      ],
      typeArr: [
        {id: 'status', name: this.$t('test_track.plan_view.execute_result')},
        {id: 'executor', name: this.$t('test_track.plan_view.executor')},
      ],
      valueArr: {
        executor: [],
        status: [
          {name: this.$t('test_track.plan_view.pass'), id: 'Pass'},
          {name: this.$t('test_track.plan_view.failure'), id: 'Failure'},
          {name: this.$t('test_track.plan_view.blocking'), id: 'Blocking'},
          {name: this.$t('test_track.plan_view.skip'), id: 'Skip'}
        ]
      },
      editor: ClassicEditor,
      editorConfig: {
        toolbar: [],
      },
    }
  },
  props: {
    reviewId: {
      type: String
    },
    selectNodeIds: {
      type: Array
    },
    selectParentNodes: {
      type: Array
    }
  },
  watch: {
    reviewId() {
      this.refreshTableAndReview();
    },
    selectNodeIds() {
      this.search();
    }
  },
  mounted() {
    this.refreshTableAndReview();
    this.isTestManagerOrTestUser = checkoutTestManagerOrTestUser();
  },
  methods: {
    initTableData() {
      if (this.reviewId) {
        this.condition.reviewId = this.reviewId;
      }
      if (this.selectNodeIds && this.selectNodeIds.length > 0) {
        this.condition.nodeIds = this.selectNodeIds;
      }
      if (this.reviewId) {
        this.result = this.$post(this.buildPagePath('/test/review/case/list'), this.condition, response => {
          let data = response.data;
          this.total = data.itemCount;
          this.tableData = data.listObject;
          this.selectRows.clear();
        });
      }
    },
    showDetail(row, event, column) {
      this.isReadOnly = true;
      this.$refs.testReviewTestCaseEdit.openTestCaseEdit(row);
    },
    refresh() {
      this.condition = {components: TEST_CASE_CONFIGS};
      this.selectRows.clear();
      this.$emit('refresh');
    },
    refreshTableAndReview() {
      this.getTestReviewById();
      this.initTableData();
    },
    refreshTestReviewRecent() {
      if (hasRoles(ROLE_TEST_USER, ROLE_TEST_MANAGER)) {
        let param = {};
        param.id = this.reviewId;
        param.updateTime = Date.now();
        // this.$post('/test/case/review/edit', param);
      }
    },
    search() {
      this.initTableData();
    },
    buildPagePath(path) {
      return path + "/" + this.currentPage + "/" + this.pageSize;
    },
    handleEdit(testCase, index) {
      this.isReadOnly = false;
      if (!checkoutTestManagerOrTestUser()) {
        this.isReadOnly = true;
      }
      this.$refs.testReviewTestCaseEdit.openTestCaseEdit(testCase);
    },
    handleDelete(testCase) {
      this.$alert(this.$t('test_track.plan_view.confirm_cancel_relevance') + ' ' + testCase.name + " ？", '', {
        confirmButtonText: this.$t('commons.confirm'),
        callback: (action) => {
          if (action === 'confirm') {
            this._handleDelete(testCase);
          }
        }
      });
    },
    handleDeleteBatch() {
      this.$alert(this.$t('test_track.plan_view.confirm_cancel_relevance') + " ？", '', {
        confirmButtonText: this.$t('commons.confirm'),
        callback: (action) => {
          if (action === 'confirm') {
            let ids = Array.from(this.selectRows).map(row => row.id);
            this.$post('/test/review/case/batch/delete', {ids: ids}, () => {
              this.selectRows.clear();
              this.$emit("refresh");
              this.$success(this.$t('commons.delete_success'));
            });
          }
        }
      });
    },
    _handleDelete(testCase) {
      let testCaseId = testCase.id;
      this.$post('/test/review/case/delete/' + testCaseId, {}, () => {
        this.$emit("refresh");
        this.$success(this.$t('commons.delete_success'));
      });
    },
    handleSelectAll(selection) {

    },
    handleSelectionChange(selection, row) {

    },
    handleBatch(type) {
      if (this.selectRows.size < 1) {
        this.$warning(this.$t('test_track.plan_view.select_manipulate'));
        return;
      }
      if (type === 'executor') {
        this.$refs.executorEdit.openExecutorEdit();
      } else if (type === 'status') {
        this.$refs.statusEdit.openStatusEdit();
      } else if (type === 'delete') {
        this.handleDeleteBatch();
      }
    },
    openTestReport() {
      this.$refs.testReportTemplateList.open(this.reviewId);
    },
    statusChange(param) {

    },
    getTestReviewById() {
      if (this.reviewId) {
        this.$post('/test/case/review/get/' + this.reviewId, {}, response => {
          this.testReview = response.data;
          this.refreshTestReviewRecent();
        });
      }
    },
    filter(filters) {
      _filter(filters, this.condition);
      this.initTableData();
    },
    sort(column) {
      // 每次只对一个字段排序
      if (this.condition.orders) {
        this.condition.orders = [];
      }
      _sort(column, this.condition);
      this.initTableData();
    },
    batchEdit(form) {
      // let param = {};
      // param[form.type] = form.value;
      // param.ids = Array.from(this.selectRows).map(row => row.id);
      // this.$post('/test/plan/case/batch/edit', param, () => {
      //   this.selectRows.clear();
      //   this.status = '';
      //   this.$post('/test/plan/edit/status/' + this.reviewId);
      //   this.$success(this.$t('commons.save_success'));
      //   this.$emit('refresh');
      // });
    },
    handleBatchEdit() {
      this.getMaintainerOptions();
      this.$refs.batchEdit.open();
    },
    getMaintainerOptions() {
      let workspaceId = localStorage.getItem(WORKSPACE_ID);
      this.$post('/user/ws/member/tester/list', {workspaceId: workspaceId}, response => {
        this.valueArr.executor = response.data;
      });
    },
    startReview() {
      if (this.tableData.length !== 0) {
        this.isReadOnly = false;
        this.$refs.testReviewTestCaseEdit.openTestCaseEdit(this.tableData[0]);
      } else {
        this.$warning("没有关联的评审！");
      }
    }
  }
}
</script>

<style scoped>

  .search {
    margin-left: 10px;
    width: 240px;
  }

  .test-case-status, .el-table {
    cursor: pointer;
  }

</style>

