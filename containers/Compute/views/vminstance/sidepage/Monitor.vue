<template>
  <div>
    <a-alert class="mb-2" :type="alertType" v-if="visible">
        <install-agent-form slot="message" :data="data" :serverColumns="serverColumns" @onInstall="handleInstallResult" />
    </a-alert>
    <monitor
      :time.sync="time"
      :timeGroup.sync="timeGroup"
      :monitorList="monitorList"
      :singleActions="singleActions"
      :loading="loading"
      @refresh="fetchData" />
  </div>
</template>

<script>
import _ from 'lodash'
import InstallAgentForm from '../components/InstallAgentForm'
import { ONECLOUD_MONITOR, VMWARE_MONITOR, OTHER_MONITOR } from '@Compute/views/vminstance/constants'
import { metricItems } from '@Compute/views/node-alert/constants'
import { UNITS, autoComputeUnit, getRequestT } from '@/utils/utils'
import Monitor from '@/sections/Monitor'
import { HYPERVISORS_MAP } from '@/constants'
import WindowsMixin from '@/mixins/windows'
import { getSignature } from '@/utils/crypto'

export default {
  name: 'VminstanceMonitorSidepage',
  components: {
    Monitor,
    InstallAgentForm,
  },
  mixins: [WindowsMixin],
  props: {
    data: { // listItemData
      type: Object,
      required: true,
    },
    serverColumns: {
      type: Array,
      required: true,
    },
  },
  data () {
    const visible = this.data.status === 'running' && (!this.data.metadata || this.data.metadata['sys:monitor_agent'] !== 'true')
    return {
      singleActions: [
        {
          label: this.$t('compute.text_382'),
          action: async obj => {
            const alertManager = new this.$Manager('nodealerts', 'v1')
            const { metric } = obj.constants
            const { data: { data = [] } } = await alertManager.list({
              params: {
                scope: this.$store.getters.scope,
                type: 'guest',
                node_id: this.data.id,
                metric,
              },
            })
            if (data && data.length) {
              if (data.length === 1) {
                this.createDialog('UpdateNodeAlert', {
                  data,
                  alertType: 'guest',
                  alertManager,
                  hypervisor: this.data.hypervisor,
                  metricOpts: this.metricOpts,
                })
              } else {
                throw Error(this.$t('compute.text_383'))
              }
            } else { // 新建报警
              this.createDialog('CreateNodeAlert', {
                alertType: 'guest',
                nodeId: this.data.id,
                metric,
                alertManager,
                hypervisor: this.data.hypervisor,
                metricOpts: this.metricOpts,
              })
            }
          },
          meta: (obj) => {
            const ret = {
              validate: true,
              tooltip: '',
            }
            if (_.get(obj, 'constants.fromItem', '').startsWith('agent_')) {
              ret.validate = false
              ret.tooltip = this.$t('compute.text_1287', [''])
            }
            return ret
          },
        },
      ],
      loading: false,
      visible: visible,
      alertType: 'warning',
      time: '1h',
      timeGroup: '1m',
      monitorList: [],
    }
  },
  computed: {
    hypervisor () {
      return this.data.hypervisor
    },
    monitorConstants () {
      if (this.hypervisor === HYPERVISORS_MAP.esxi.key) {
        return VMWARE_MONITOR
      } else if (this.hypervisor === HYPERVISORS_MAP.kvm.key) {
        return ONECLOUD_MONITOR
      } else {
        return OTHER_MONITOR
      }
    },
    serverId () {
      return this.data.id
    },
    hasMemMetric () {
      return this.hypervisor === HYPERVISORS_MAP.esxi.key
    },
    metricOpts () {
      const opts = [metricItems['vm_cpu.usage_active'], metricItems['vm_netio.bps_recv'], metricItems['vm_netio.bps_sent'], metricItems['vm_diskio.read_bps'], metricItems['vm_diskio.write_bps']]
      if (this.hasMemMetric) {
        opts.splice(1, 0, metricItems['vm_mem.used_percent'])
      }
      return opts
    },
  },
  created () {
    this.fetchData()
    this.fetchDataDebounce = _.debounce(this.fetchData, 500)
    this.baywatch(['time', 'timeGroup', 'data.id'], this.fetchDataDebounce)
  },
  methods: {
    handleInstallResult (ret) {
      if (ret && ret.status === 'succeed') {
        this.alertType = 'success'
        this.$nextTick(() => {
          setTimeout(() => { this.visible = false }, 3000)
        })
      }
    },
    async fetchData () {
      this.loading = true
      const resList = []
      for (let idx = 0; idx < this.monitorConstants.length; idx++) {
        const val = this.monitorConstants[idx]
        try {
          const { data } = await new this.$Manager('unifiedmonitors', 'v1')
            .performAction({
              id: 'query',
              action: '',
              data: this.genQueryData(val),
              params: { $t: getRequestT() },
            })
          resList.push({ title: val.label, constants: val, series: data.series })
          if (idx === this.monitorConstants.length - 1) {
            this.loading = false
            this.getMonitorList(resList)
          }
        } catch (error) {
          this.loading = false
          throw error
        }
      }
    },
    baywatch (props, watcher) {
      const iterator = function (prop) {
        this.$watch(prop, watcher)
      }
      props.forEach(iterator, this)
    },
    getMonitorList (resList) {
      this.monitorList = resList.map(result => {
        const { unit, transfer } = result.constants
        const isSizestrUnit = UNITS.includes(unit)
        let series = result.series
        if (!series) series = []
        if (isSizestrUnit || unit === 'bps') {
          series = series.map(serie => {
            return autoComputeUnit(serie, unit, transfer)
          })
        }
        return {
          title: result.title,
          series,
          constants: result.constants,
        }
      })
    },
    genQueryData (val) {
      const data = {
        metric_query: [
          {
            model: {
              measurement: val.fromItem,
              select: [
                [
                  {
                    type: 'field',
                    params: [val.seleteItem],
                  },
                  { // 对应 mean(val.seleteItem)
                    type: 'mean',
                    params: [],
                  },
                  { // 确保后端返回columns有 val.label 的别名
                    type: 'alias',
                    params: [val.label],
                  },
                ],
              ],
              tags: [
                {
                  key: 'vm_id',
                  value: this.serverId,
                  operator: '=',
                },
              ],
            },
          },
        ],
        scope: this.$store.getters.scope,
        from: this.time,
        interval: this.timeGroup,
        unit: true,
      }
      data.signature = getSignature(data)
      return data
    },
  },
}
</script>
