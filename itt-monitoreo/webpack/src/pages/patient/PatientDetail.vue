<template>
<div>
  <!-- Content Header (Page header) -->
  <section class="content-header">
    <h1> {{ title }} </h1>
    <ol class="breadcrumb">
      <li><router-link :to="{ path: '/dashboard' }"><i class="fa fa-dashboard"></i> Dashboard</router-link></li>
      <li><router-link :to="{ path: '/user' }">Patients</router-link></li>
      <li class="active">Detail</li>
    </ol>
  </section>

  <!-- Main content -->
  <section class="content">

    <div class="row">
      <!-- Left col -->
      <section class="col-lg-7">

        <!-- Custom tabs (Charts with tabs)-->
        <div class="nav-tabs-custom">
          <!-- Tabs within a box -->
          <ul class="nav nav-tabs pull-right">
            <li class="active"><a href="#revenue-chart" data-toggle="tab">Area</a></li>
            <!-- <li><a href="#sales-chart" data-toggle="tab">Donut</a></li> -->
            <li class="pull-left header"><i class="fa fa-area-chart"></i> Sensor Readings</li>
          </ul>
          <div class="tab-content no-padding" v-if="!disabledStatusTable">
            <!-- Morris chart - Sales -->
            <div class="chart tab-pane active" id="revenue-chart" style="position: relative;">
              <line-chart :chart-data="datacollection" :height="200"></line-chart>
            </div>
            <!-- <div class="chart tab-pane" id="sales-chart" style="position: relative;"></div> -->
          </div>
          <!-- Message to show when the user state is offline-->
          <div v-if="disabledStatusTable" class="tab-content no-padding" style="text-align: center;">
            <br><br>
            <i>No data available while the sensor is offline.</i>
            <br><br><br>
          </div>
        </div>
        <!-- /.nav-tabs-custom -->

      </section>
      <!-- /.Left col -->
      <!-- right col (We are only adding the ID to make the widgets sortable)-->
      <section class="col-lg-5">

        <!-- Widget: user widget style 1 -->
        <div class="box box-widget widget-user-2">
          <!-- Add the bg color to the header using any of the bg-* classes -->
          <div class="widget-user-header bg-yellow">
            <div class="widget-user-image">
              <img class="img-circle" v-bind:src="user.gravatarUrl" alt="Patient Avatar">
            </div>
            <!-- /.widget-user-image -->
            <h3 class="widget-user-username">{{user.firstName + ' ' + user.lastName}}</h3>
            <h5 class="widget-user-desc">Birth Date: {{ moment(user.birthDate).format('MMMM DD YYYY') }}</h5>
          </div>
          <div class="box-footer no-padding">
            <ul class="nav nav-stacked">
              <li><a>Status <span class="pull-right badge"
                :class="{ 'bg-green': user.status === 'stable', 'bg-red': user.status === 'critical', 'bg-gray': user.status === 'offline' }">
                {{ user.status }}</span></a>
              </li>
              <li><a>Insurance Company <span class="pull-right text-muted">{{ user.insuranceCompany }}</span></a></li>
              <li>
                <a>Address(es)
                  <span class="pull-right text-muted">
                    <ul><li v-for="item in user.addresses">{{item}}</li></ul>
                  </span>
                </a>
                <br/>
              </li>
              <li>
                <a>Phone Number(s)
                  <span class="pull-right text-muted">
                    <ul><li v-for="item in user.phoneNumbers">{{item}}</li></ul>
                  </span>
                </a>
                <br/>
              </li>
            </ul>
          </div>
        </div>
        <!-- /.widget-user -->

      </section>
      <!-- right col -->
    </div>
    <!-- Second row -->
    <div class="row">
      <section class="col-lg-12">
        <!-- Table Panel -->
        <div class="box">
          <div class="box-header with-border">
            <h3 class="box-title"> <i class="fa fa-fw fa-file-text-o"></i> History Records</h3>
          </div>
          <div class="box-body form-horizontal">

            <div class="form-group">
              <label for="dateRange" class="col-lg-2 control-label">Date Range</label>
              <div class="col-lg-6">
                <date-range-picker id="dateRange" v-on:date-range-changed="onDateRangeChanged"></date-range-picker>
              </div>
              <div class="col-lg-4">
                <button v-on:click="getSearchReport" class="btn btn-primary">
                  Search &nbsp; <span class="fa fa-search" aria-hidden="true"></span>
                </button>
              </div>
            </div>
            <hr>
            <div class="chart tab-pane" id="report-chart" style="position: relative;" v-if="showReportChart">
              <line-chart-time :chart-data="reportcollection" :height="100"></line-chart-time>
            </div>

          </div>
          <!-- Loading (remove the following to stop the loading)-->
          <div v-if="loadingReportChart" class="overlay"><i class="fa fa-refresh fa-spin"></i></div>
        </div>
        <!-- End of Table Panel -->
      </section>
    </div>
    <!-- /.Second row -->
  </section>

</div>
</template>

<script>
import $ from 'jquery';
import '../../../static/plugins/bootstrap-datepicker/js/bootstrap-datepicker.js';
import auth from '@/services/auth';
import LineChart from '@/components/LineChart';
import LineChartTime from '@/components/LineChartTime';
import Datepicker from '@/directives/Datepicker';
import moment from 'moment';
//import '../../../static/plugins/bootstrap-daterangepicker/js/daterangepicker.js'
import DateRangePicker from '@/components/DateRangePicker';
import api from '@/services/api';


import router from '@/router/index';

export default {
  name: 'PatientDetail',
  data() {
    return {
      id: this.$route.params.id,
      title: 'Patient Detail',
      user: {},
      datacollection: {},
      reportcollection: {},
      showReportChart: false,
      updateInterval: 500,  // fetch data over x milliseconds
      loadingStatusTable: true,
      loadingReportChart: false,
      disabledStatusTable: false,
      patients: [],
      username: null,
      startDate: null,
      endDate: null,
    };
  },
  components: {
    LineChart,
    LineChartTime,
    DateRangePicker
  },
  directives: {
    Datepicker,
    //Daterangepicker
  },
  methods: {
    moment() { return moment(); },
    fillData() {
      let self = this;
      api.get('/bucket?username=' + this.user.username + '&mins=60').then(function(response) {
          let results = response.data;
          let dateresult = results.map(a => a.timestamp);
          let bpmresult = results.map(a => a.bpm);
          self.datacollection = {
            labels: dateresult,
            datasets: [
              {
                label: 'Beats Per Minute (BPM)',
                backgroundColor: '#5bf8bf', //greenish     //'#f87979', //reddish
                data: bpmresult
              }
            ]
          };

          setTimeout(function() {
            console.log('Calling fillData()...')
            self.fillData();
          }, self.updateInterval);
        })
        .catch(function(error) {
          console.log(error);
        });
    },
    getSearchReport: function() {
      this.loadingReportChart = true;
      let self = this;
      api.get('/bucketrange?username=' + this.username + '&startDate=' + this.startDate + '&endDate=' + this.endDate).then(function(response) {
        console.log(response.data);
        let results = response.data;
        let dateresult = results.map(a => a.timestamp);
        let bpmresult = results.map(a => a.bpm);

        self.reportcollection = {
          labels: dateresult,
          datasets: [
            {
              label: 'Beats Per Minute (BPM)',
              backgroundColor: '#5bf8bf', //greenish     //'#f87979', //reddish
              data: bpmresult
            }
          ]
        };
        self.showReportChart = true;
        self.loadingReportChart = false;

      }).catch(function(error) {
        console.log(error);
        self.loadingReportChart = false;
        self.showReportChart = false;
      });
    },

    onDateRangeChanged: function(picker) {
      this.startDate = picker.startDate.format('YYYY-MM-DDTHH:mm:ss');
      this.endDate = picker.endDate.format('YYYY-MM-DDTHH:mm:ss');
      this.username = this.user.username;
    },
  },
  created() {
    let self = this;
    api.get('/user/' + self.id + '?populate=[roles,assignedDoctor,assignedPatient]').then(function(response) {
        self.user = response.data;
        // Hide the graph if the wearable status is offline
        if (self.user.status == 'offline') {
          self.disabledStatusTable = true;
        } else {
          self.disabledStatusTable = false;
        }

        setTimeout(function() {
          console.log('Calling fillData()...')
          self.fillData();
        }, self.updateInterval);
      })
      .catch(function(error) {
        console.log(error);
      });
  },
};
</script>
<style src="../../../static/plugins/bootstrap-datepicker/css/bootstrap-datepicker.min.css"></style>
<style src="../../../static/plugins/bootstrap-daterangepicker/css/daterangepicker.css"></style>
<!-- Theme style -->
<style src="../../../static/adminLte/css/AdminLTE.min.css"></style>
<style>
.box .datepicker-inline,
.box .datepicker-inline .datepicker-days,
.box .datepicker-inline>table,
.box .datepicker-inline .datepicker-days>table {
  width: 100%;
}
</style>
