<template>
  <flexbox align="stretch">
    <flexbox-item style="margin-right: 50px;">
      <div class="area-title">定位</div>
      <el-autocomplete style="width: 100%;"
                       v-model="searchInput"
                       :fetch-suggestions="querySearchAsync"
                       placeholder="请输入详细位置名称"
                       @blur="inputBlur"
                       @focus="inputFocus"
                       @select="handleSelect">
        <template slot-scope="{ item }">
          <div class="name">{{ item.address + item.title}}</div>
        </template>
      </el-autocomplete>
      <div id="choicemap"
           class="map"></div>
      <div class="area-title">详细地址</div>
      <el-input v-model="detail_address"
                placeholder=""></el-input>
    </flexbox-item>
    <flexbox-item>
      <div class="area-title">省/市/区</div>
      <v-distpicker @selected="onAddressSelected"
                    :province="addressSelect.province"
                    :city="addressSelect.city"
                    :area="addressSelect.area"></v-distpicker>
    </flexbox-item>
  </flexbox>
</template>
<script type="text/javascript">
import VDistpicker from 'v-distpicker'

export default {
  name: 'xh-customer-address', // 新建 客户位置
  components: {
    VDistpicker
  },
  computed: {},
  watch: {
    point_address: function(newValue) {
      this.valueChange()
    },
    detail_address: function(newValue) {
      this.valueChange()
    }
  },
  data() {
    return {
      map: null,
      /** 搜索地图输入框 */
      searchInput: '',
      searchCopyInput: '', // 避免修改
      /** 完整地址输入框 */
      detail_address: '',
      point_address: null, // 经纬度点
      /** 区域选择 */
      addressSelect: {
        province: '',
        city: '',
        area: ''
      },
      /** 防止联动情况  */
      canExecute: true
    }
  },
  props: {
    value: {
      type: Object,
      default: () => {
        return {}
      }
    },
    /** 索引值 用于更新数据 */
    index: Number,
    /** 包含数据源 */
    item: Object
  },
  mounted() {
    var map = new BMap.Map('choicemap', { enableMapClick: false })
    map.centerAndZoom(new BMap.Point(116.404, 39.915), 14)
    // map.disableDragging() //禁止拖拽
    // map.disableDoubleClickZoom()
    // map.disableScrollWheelZoom()
    // map.disableContinuousZoom()
    map.enableScrollWheelZoom()
    this.map = map
    if (this.value && JSON.stringify(this.value) !== '{}') {
      this.initInfo(this.value)
    } else {
      var geolocation = new BMap.Geolocation()
      var self = this
      geolocation.getCurrentPosition(
        function(r) {
          if (this.getStatus() == BMAP_STATUS_SUCCESS) {
            self.addMarkerLabel(r.point)
          }
        },
        { enableHighAccuracy: true }
      )
    }
  },
  methods: {
    initInfo(val) {
      this.searchInput = val.location
      this.detail_address = val.detail_address
      if (Object.prototype.toString.call(val.address) == '[object Array]') {
        var address = {}
        for (let index = 0; index < val.address.length; index++) {
          index === 0 ? (address['province'] = val.address[0]) : ''
          index === 1 ? (address['city'] = val.address[1]) : ''
          index === 2 ? (address['area'] = val.address[2]) : ''
        }
        this.addressSelect = address
      }
      if (val.lng != 0 && val.lat != 0) {
        this.point_address = new BMap.Point(val.lng, val.lat)
        this.addMarkerLabel(this.point_address)
      }
    },
    querySearchAsync(queryString, cb) {
      if (queryString) {
        var options = {
          onSearchComplete: function(results) {
            if (local.getStatus() == BMAP_STATUS_SUCCESS) {
              var address = []
              for (var i = 0; i < results.getCurrentNumPois(); i++) {
                address.push(results.getPoi(i))
              }
              cb(address)
            } else {
              cb([])
            }
          },
          pageCapacity: 20
        }
        var local = new BMap.LocalSearch(this.map, options)
        local.search(queryString)
      } else {
        cb([])
      }
    },
    /** 搜索结果选择 */
    handleSelect(item) {
      this.searchInput = item.address + item.title
      this.searchCopyInput = this.searchInput // 只能通过这种方式修改

      this.detail_address = this.searchInput
      this.addMarkerLabel(item.point)
      this.point_address = item.point
      this.mapSelectArea(item)
    },
    /** Input 失去焦点  searchInput 只能通过选择更改*/
    inputBlur() {
      if (this.searchCopyInput !== this.searchInput) {
        this.searchInput = this.searchCopyInput
      }
    },
    inputFocus() {
      this.searchCopyInput = this.searchInput
    },
    // 创建标注
    addMarkerLabel(point) {
      this.map.clearOverlays()
      this.map.centerAndZoom(point, 14)
      this.map.addOverlay(new BMap.Marker(point))
    },
    /** 区域选择 */
    onAddressSelected(data) {
      this.addressSelect.province = data.province.value
      this.addressSelect.city = data.city.value
      this.addressSelect.area = data.area.value
      this.valueChange()
      // this.areaSelectMap(data)
    },
    // /** 区域选择地图 区域选择  不影响 定位 */
    // areaSelectMap(data) {
    //   if (this.canExecute) {
    //     this.canExecute = false
    //     this.searchInput =
    //       data.province.value + data.city.value + data.area.value

    //     this.detail_address = this.searchInput
    //     // 创建地址解析器实例
    //     var myGeo = new BMap.Geocoder()
    //     // 将地址解析结果显示在地图上，并调整地图视野
    //     var self = this
    //     myGeo.getPoint(this.searchInput, function(point) {
    //       if (point) {
    //         // self.map.centerAndZoom(point, 14)
    //         self.addMarkerLabel(point)
    //       }
    //     })

    //     setTimeout(() => {
    //       self.canExecute = true
    //     }, 500)
    //   }
    // },
    /** 地图选择区域 */
    mapSelectArea(data) {
      if (this.canExecute) {
        this.canExecute = false
        // this.addressSelect.province = data.province ? data.province : "";
        // this.addressSelect.city = data.city ? data.city : "";
        /** 因为poi里面不包含区域 所以需要逆地址解析 */
        var myGeo = new BMap.Geocoder()
        // 根据坐标得到地址描述
        var self = this
        myGeo.getLocation(
          new BMap.Point(data.point.lng, data.point.lat),
          function(result) {
            if (result) {
              // 获取经纬度点
              self.point_address = result.point
              self.addressSelect.province = result.addressComponents.province
                ? result.addressComponents.province
                : ''
              self.addressSelect.city = result.addressComponents.city
                ? result.addressComponents.city
                : ''
              self.addressSelect.area = result.addressComponents.district
                ? result.addressComponents.district
                : ''
            }
          }
        )

        setTimeout(() => {
          self.canExecute = true
        }, 500)
      }
    },
    // 值更新的回调
    valueChange() {
      this.$emit('value-change', {
        index: this.index,
        value: {
          address: [
            this.addressSelect.province,
            this.addressSelect.city,
            this.addressSelect.area
          ],
          location: this.searchInput,
          detail_address: this.detail_address,
          lat: this.point_address ? this.point_address.lat : '',
          lng: this.point_address ? this.point_address.lng : ''
        }
      })
    }
  }
}
</script>
<style rel="stylesheet/scss" lang="scss" scoped>
.map {
  height: 150px;
  width: 100%;
  overflow: hidden;
  margin-top: 5px;
}

.area-title {
  font-size: 12px;
  color: #aaa;
  padding-left: 10px;
}

.distpicker-address-wrapper /deep/ select {
  height: 34px;
  font-size: 12px;
  border-radius: 0.1rem;
}
</style>
