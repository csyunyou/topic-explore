<template>
  <div class='radar-chart-wrapper'>
    <div v-for='group in filteredRadarData'
         class='radar-group'>
      <div class="title">
        {{group.status}}
      </div>
      <div class="content">
        <radar v-for="(doc,id) in group.docs"
               :doc='doc'
               :sizeColorMap='group.sizeColorMap'
               :funcNumColorMap='group.funcNumColorMap'
               :key="id"></radar>
      </div>
    </div>
  </div>
</template>

<script>
import _ from 'lodash'
import * as d3 from 'd3'
import { getVersion } from '../utils'
import Radar from './Radar.vue'
export default {
  name: 'component_name',
  data () {
    return {
      height: null,
      radarData: [],
      filteredRadarData: []
    }
  },
  props: ['fileGroup', 'prevVer'],
  components: {
    Radar
  },
  watch: {
    fileGroup () {
      console.log(this.fileGroup, this.prevVer)
      const addDocs = this.fileGroup.addDocs.map(d => ({
        size: d.size,
        funcNum: d['func_Num'],
        fileIds: [d.id],
        data: [
          {
            version: 'next',
            ...d
          }
        ]
      }))
      const delDocs = this.fileGroup.delDocs.map(d => ({
        size: d.size,
        funcNum: d['func_Num'],
        fileIds: [d.id],
        data: [
          {
            version: 'pre',
            ...d
          }
        ]
      }))
      let editDocs = []
      let val, tmpArr, preData, nextData, version
      Object.keys(this.fileGroup.editDocsObj).forEach(key => {
        val = this.fileGroup.editDocsObj[key]
        if (val.length === 2) {
          tmpArr = []
          for (let i = 0; i < val.length; i++) {
            version = getVersion(val[i].filename)
            tmpArr.push({
              version: version === this.prevVer ? 'pre' : 'next',
              ...val[i]
            })
            if (version === this.prevVer) preData = val[i]
            else nextData = val[i]
          }
          editDocs.push({
            size: nextData.size - preData.size,
            fileIds: [preData.id, nextData.id],
            funcNum: nextData['func_Num'] - preData['func_Num'],
            data: tmpArr
          })
        }
      })
      this.radarData = [
        {
          status: '增加',
          docs: addDocs,
          sizeColorMap: this.getColorMap(addDocs, 'size'),
          funcNumColorMap: this.getColorMap(addDocs, 'funcNum')
        },
        {
          status: '减少',
          docs: delDocs,
          sizeColorMap: this.getColorMap(delDocs, 'size'),
          funcNumColorMap: this.getColorMap(delDocs, 'funcNum')
        },
        {
          status: '修改',
          docs: editDocs,
          sizeColorMap: this.getColorMap(editDocs, 'size'),
          funcNumColorMap: this.getColorMap(editDocs, 'funcNum')
        }
      ]
      this.filteredRadarData = this.radarData.slice()
    }
  },
  methods: {
    getColorMap (docs, attr) {
      const [minVal, maxVal] = d3.extent(docs, d => d[attr])
      return d3
        .scaleLinear()
        .domain([minVal, maxVal])
        .range(['#d73027', '#1a9850'])
        .interpolate(d3.interpolateHcl)
    }
  },
  created () {
    this.$bus.$on('cluster-selected', ids => {
      if (ids === null) {
        console.log('reset')
        this.filteredRadarData = this.radarData.slice()
        return
      }
      this.radarData.forEach((group, groupId) => {
        this.$set(this.filteredRadarData, groupId, {
          ...group,
          docs: group.docs.filter(doc =>
            doc['fileIds'].some(id => ids.indexOf(id) !== -1)
          )
        })
      })
    })
    this.$bus.$on('topic-selected', selectedTopic => {
      this.radarData.forEach((group, groupId) => {
        console.log(
          group.docs.filter(doc =>
            doc['data'].some(d => {
              // console.log(d)
              return d['Dominant_Topic'] === selectedTopic
            })
          )
        )
        this.$set(this.filteredRadarData, groupId, {
          ...group,
          docs: group.docs.filter(doc =>
            doc['data'].some(d => {
              // console.log(d)
              return d['Dominant_Topic'] === selectedTopic
            })
          )
        })
      })
    })
  }
}
</script>

<style lang="less">
.radar-chart-wrapper {
  display: flex;
  flex-direction: column;
  .radar-group {
    &:nth-child(n + 2) {
      margin-top: 20px;
    }
    flex: 1;
    display: flex;
    .title {
      flex: none;
      display: flex;
      writing-mode: vertical-lr;
      align-items: center;
      justify-content: center;
      border: 1px solid#ebeef5;
      font-weight: bold;
    }
    .content {
      overflow: scroll;
      flex: auto;
      display: flex;
      flex-wrap: wrap;
      .radar {
        flex: none;
      }
      border: 1px solid #ebeef5;
      border-left: none;
    }
  }
}
</style>