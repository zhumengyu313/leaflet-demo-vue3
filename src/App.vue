<template>
  <div class="container">
    <div id="map-container"></div>
    <div class="table-container">
      <el-table
        ref="tableRef"
        @row-click="rowClick"
        :current-row-key="currentStationId"
        :row-key="row => row.code"
        highlight-current-row
        :data="paginatedStations"
        height="100%"
        border
      >
        <el-table-column prop="name" align="center" label="名称" />
        <el-table-column prop="province" align="center" label="所属省份" />
        <el-table-column prop="code" align="center" label="编码" />
        <el-table-column prop="lng" align="center" label="经度" />
        <el-table-column prop="lat" align="center" label="纬度" />
        <el-table-column prop="height" align="center" label="海拔" />
      </el-table>
    </div>
    <div class="pagination-container">
      <el-pagination
        layout="prev, pager, next"
        @current-change="handleCurrentChange"
        :page-size="pageSize"
        :total="stations.length"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted, watch, type ComponentInstance } from 'vue'
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'
import type { ElTable } from 'element-plus'
import stationsData from '@/stations.json'

interface Station {
  name: string
  province: string
  code: string
  lng: string
  lat: string
  height: string
}

// 响应式数据
const stations = ref<Station[]>(stationsData)
const currentPage = ref(1)
const pageSize = ref(10)
const currentStationId = ref<string | null>(null)
const selectedStation = ref<Station | null>(null)
const tableRef = ref<ComponentInstance<typeof ElTable>>()
const map = ref<L.Map | null>(null)
const customIcon = ref<L.Icon>()
const clickedIcon = ref<L.Icon>()

// 计算paginatedStations用作分页
const paginatedStations = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return stations.value.slice(start, end)
})

// 初始化地图
const initMap = () => {
  map.value = L.map('map-container', {
    crs: L.CRS.EPSG3857,
    center: [31.8847, 117.3114],
    zoomControl: false,
    zoom: 8
  })

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
  }).addTo(map.value)

  // 初始化图标
  customIcon.value = L.icon({
    iconUrl: new URL('@/assets/image/marker.png', import.meta.url).href,
    iconSize: [18, 20],
    iconAnchor: [10, 20],
    popupAnchor: [-3, -26]
  })

  clickedIcon.value = L.icon({
    iconUrl: new URL('@/assets/image/marker-active.png', import.meta.url).href,
    iconSize: [18, 20],
    iconAnchor: [10, 20],
    popupAnchor: [-3, -26]
  })

  updateMarkers(paginatedStations.value)
}
//打点
const updateMarkers = (data: Station[]) => {
  if (!map.value) return
  // 清除现有标记
  map.value.eachLayer((layer:any) => {
    if (layer instanceof L.Marker) {
      map.value.removeLayer(layer)
    }
  })

  data.forEach(station => {
    const marker = L.marker([Number(station.lat), Number(station.lng)], { icon: customIcon.value }).addTo(map?.value)
    marker.on('click', () => {
      resetIconsToDefault()
      currentStationId.value = station.code //高亮对应行
      selectedStation.value = station
      map.value.setView([Number(station.lat), Number(station.lng)], 13)

      marker.unbindPopup()
      marker.bindPopup(`
        <p>名称: ${station.name}</p>
        <p>所属省份: ${station.province}</p>
        <p>编码: ${station.code}</p>
        <p>位置: ${station.lng},${station.lat}</p>
        <p>海拔: ${station.height}m</p>
      `,{
            closeButton: true,
            closeOnClick: false
          }).openPopup()
         // 切换当前为选中图标
      marker.setIcon(clickedIcon.value);
      marker.getPopup().on('remove', () => {
        resetIconsToDefault()
      })
    })
  })
}
// 重置所有标记为默认图标
const resetIconsToDefault = () => {
  // map.value?.eachLayer((layer:any) => {
  //   if (layer instanceof L.Marker) {
  //     layer.setIcon(customIcon.value)
  //   }
  // })
  Object.keys(map.value._layers).forEach((key) => {
        const layer = map.value._layers[key];
        if (layer instanceof L.Marker) {
          layer.setIcon(customIcon.value);
        }
      });
}
//行点击
const rowClick = (row: Station) => {
  resetIconsToDefault()
  currentStationId.value = row.code
  selectedStation.value = row
  map.value.setView([Number(row.lat), Number(row.lng)], 13)
// 循环对象查找对应数据并修改图标和打开弹窗
      Object.keys(map.value._layers).forEach((key) => {
        const layer = map.value._layers[key];
        if (layer && layer._latlng && layer._latlng.lat == row.lat && layer._latlng.lng == row.lng) {
          // 移除之前的弹窗绑定
          layer.unbindPopup();
          layer.bindPopup(`
            <p>名称: ${row.name}</p>
            <p>所属省份: ${row.province}</p>
            <p>编码: ${row.code}</p>
            <p>位置: ${row.lng},${row.lat}</p>
            <p>海拔: ${row.height}m</p>
          `,).openPopup();
          // 切换选中图标
          layer.setIcon(clickedIcon.value);
          // 点击行的也添加弹窗关闭事件监听器
          layer.getPopup().on('remove', () => {
            resetIconsToDefault();
          });
        }
      });

}

const handleCurrentChange = (val: number) => {
  currentPage.value = val
}

// 监听分页数据变化
watch(paginatedStations, (newVal) => {
  updateMarkers(newVal)
})

// 初始化地图
onMounted(() => {
  initMap()
})
</script>

<style scoped>
.container {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
}

#map-container {
  flex: 1;
  position: relative;
}

.table-container {
  height: calc(30% - 20px);
}

.pagination-container {
  height: 20px;
  padding: 10px 0;
}

:deep(.el-table tbody tr:hover) {
  cursor: pointer;
}

:deep(.el-table tbody tr:hover > td .cell) {
  color: var(--el-color-primary) !important;
}
</style>
