<style scope>
  .clearfix:after {content:""; clear:both; display:table;}
</style>
<template>
  <div>
    <div class="clearfix" style="margin-bottom: 20px;">
      <router-link class="ui left floated button" to="/job/executing">查看执行中的任务</router-link>
      <button class="ui left floated icon button" v-on:click="refresh"><i class="refresh icon"></i></button>
      <router-link class="ui right floated primary button" to="/job/create"><i class="add to calendar icon"></i> 新任务</router-link>
    </div>
    <form class="ui form">
      <div class="two fields">
        <div class="field">
          <label>任务分组</label>
          <Dropdown title="选择分组" v-bind:items="groups" v-on:change="changeGroup" :selected="group"/>
        </div>
        <div class="field">
          <label>节点过滤</label>
          <Dropdown title="选择节点" v-bind:items="nodes" v-on:change="changeNode" :selected="node"/>
        </div>
      </div>
    </form>
    <table class="ui hover blue table" v-if="jobs.length > 0">
      <thead>
        <tr>
          <th class="collapsing center aligned">操作</th>
          <th class="collapsing center aligned">状态</th>
          <th width="200px" class="center aligned">分组</th>
          <th class="center aligned">用户</th>
          <th class="center aligned">名称</th>
          <th class="center aligned">最近执行时间</th>
          <th class="center aligned">执行结果</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(job, index) in jobs">
          <td class="center aligned">
            <div class="ui icon dropdown">
              <i class="content icon"></i>
              <div class="menu">
                <div class="item" v-on:click="$router.push('/job/edit/'+job.group+'/'+job.id)">编辑</div>
                <div class="item" v-if="job.pause" v-on:click="changeStatus(job.group, job.id, index, !job.pause)">开启</div>
                <div class="item" v-if="!job.pause" v-on:click="changeStatus(job.group, job.id, index, !job.pause)">暂停</div>
                <div class="divider"></div>
                <div class="item" style="color:red;" v-on:click="removeJob(job.group, job.id, index)">删除</div>
              </div>
            </div>
          </td>
          <td class="center aligned"><i class="icon" v-bind:class="{pause: job.pause, play: !job.pause, green: !job.pause}"></i></td>
          <td>{{job.group}}</td>
          <td>{{job.user}}</td>
          <td><router-link :to="'/job/edit/'+job.group+'/'+job.id">{{job.name}}</router-link></td>
          <td>
            <span v-if="!job.latestStatus">-</span>
            <span v-else>{{formatLatest(job.latestStatus)}}</span>
          </td>
          <td :class="{error: job.latestStatus && !job.latestStatus.success}">
            <span v-if="!job.latestStatus">-</span>
            <router-link v-else :to="'/log/'+job.latestStatus.refLogId">{{job.latestStatus.success ? '成功' : '失败'}}</router-link> |
            <router-link :to="{path: 'log', query: {latest:true, ids: job.id}}">latest</router-link> |
            <a href="#" title="点此选择节点重新执行任务" v-on:click.prevent="showExecuteJobModal(job.name, job.group, job.id)"><i class="icon repeat"></i></a>
          </td>
        </tr>
      </tbody>
    </table>
    <ExecuteJob ref="executeJobModal">
  </div>
</template>

<script>
import Dropdown from './basic/Dropdown.vue';
import ExecuteJob from './ExecuteJob.vue';
import {formatTime, formatDuration} from '../libraries/functions';

export default {
  name: 'job',
  data: function(){
    return {
      groups: [],
      group: '',
      nodes: [],
      node: '',
      jobs: []
    }
  },
  
  mounted: function(){
    this.fillParams();
    var vm = this;

    this.$rest.GET('job/groups').onsucceed(200, (resp)=>{
      !resp.includes('default') && resp.unshift('default');
      resp.unshift({value: '', name: '所有分组'});
      vm.groups = resp;
      this.fetchList(this.buildQuery());
    }).do();

    this.$rest.GET('nodes').onsucceed(200, (resp)=>{
      vm.nodes.push({name: '所有节点', value: ''});
      for (var i in resp) {
        vm.nodes.push(resp[i].id);
      }
    }).do();
  },

  watch: {
    '$route': function(){
      this.fillParams();
      this.fetchList(this.buildQuery());
    }
  },

  methods: {
    fillParams: function(){
      this.group = this.$route.query.group || '';
      this.node = this.$route.query.node || '';
    },

    changeGroup: function(val, text){
      var vm = this;
      this.group = val;
      this.$router.push('job?'+this.buildQuery());
    },

    changeNode: function(val, text){
      var vm = this;
      this.node = val;
      this.$router.push('job?'+this.buildQuery());
    },

    buildQuery: function(){
      var params = [];
      if (this.group) params.push('group='+this.group);
      if (this.node) params.push('node='+this.node);
      return params.join('&');
    },

    fetchList: function(query){
      var vm = this;
      this.$rest.GET('jobs?'+query).onsucceed(200, (resp)=>{
        vm.jobs = resp;
        vm.$nextTick(()=>{
          $(vm.$el).find('table .ui.dropdown').dropdown();
        });
      }).do();
    },

    refresh: function(){
      this.fetchList(this.buildQuery());
    },

    removeJob: function(group, id, index){
      var vm = this;
      this.$rest.DELETE('job/'+group+'-'+id).onsucceed(204, (resp)=>{
        vm.jobs.splice(index, 1);
      }).do();
    },

    changeStatus: function(group, id, index, isPause){
      var vm = this;
      this.$rest.POST('job/'+group+'-'+id, {"pause": isPause}).onsucceed(200, (resp)=>{
        vm.refresh();
      }).do();
    },

    formatExecResult: function(st){
      if (!st) return '-';
      return 
    },

    formatLatest: function(latest){
      return formatTime(latest.beginTime, latest.endTime)+'，于 '+latest.node+' 耗时 '+formatDuration(latest.beginTime, latest.endTime);
    },

    showExecuteJobModal: function(jobName, jobGroup, jobId){
      this.$refs.executeJobModal.show(jobName, jobGroup, jobId);
    }
  },

  components: {
    Dropdown,
    ExecuteJob
  }
}
</script>