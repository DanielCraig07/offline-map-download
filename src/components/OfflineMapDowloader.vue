<template>
  <div class="offline-amap-container">
    <div class="search-container">
      <div class="search-row">
        <input placeholder="请输入高德地图URL,例如: https://webapi.amap.com/maps?v=2.0&key=xxx" type="text" v-model="key" class="key-input" />
        <button @click="loadAMap()">加载地图</button>
      </div>
      <div class="search-row">
        <input
          placeholder="请输入地点搜索，如：深圳"
          type="text"
          v-model="searchKey"
          @keyup.enter="onSearch()"
          class="search-input"
        />
        <button @click="onSearch()">搜索</button>
        <button @click="drawRect()">画范围</button>
        <button @click="isShow = true" v-if="rect">下载</button>
      </div>
    </div>

    <div class="bottom-info" ref="info">
      <span>在地图上拖拉画矩形</span>
      当前缩放级别： {{ zoom }},中心经纬度:{{ lng }},{{ lat }}
      <span v-if="rect">选中范围:{{ rectLngLat }}</span>
    </div>

    <div class="map-style-switcher">
      <div
        v-for="item in styleOps"
        :key="item.value"
        class="style-item"
        :class="{ active: selectStyle === item.value }"
        @click="switchMapLayer(item.value)"
      >
        {{ item.label }}
      </div>
    </div>

    <div class="left-tool" v-if="rect">
      <button type="primary" @click="clearRect()">清空范围</button>
      <button type="primary" @click="onEditRectStart()">开始编辑范围</button>
      <button type="primary" @click="onEditRectEnd()">结束编辑范围</button>
    </div>
    <div class="canvas" ref="canvas"></div>

    <el-dialog v-model="isShow" title="下载地图瓦片" width="60%" append-to-body>
      <el-form label-width="100px">
        <el-form-item label="选中范围">
          <el-input :value="rectLngLat" readonly> </el-input>
        </el-form-item>
        <el-form-item label="中心点">
          <el-input :value="centerLnglat" readonly> </el-input>
        </el-form-item>
        <el-form-item label="地图样式">
          <el-select v-model="selectStyle">
            <el-option v-for="item in styleOps" v-bind="item" :key="item.value"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="路径规则">
          <el-input :value="rule" readonly> </el-input>
        </el-form-item>
      </el-form>
      <el-table v-if="rect && tableData" :data="tableData" height="400">
        <el-table-column prop="level" label="缩放级别"></el-table-column>
        <el-table-column prop="num" label="瓦片数量"></el-table-column>
        <el-table-column label="选中">
          <template #default="scope">
            <input type="checkbox" v-if="scope.row" v-model="zoomMap[scope.row.level]" />
          </template>
        </el-table-column>
      </el-table>
      <template #footer>
        <div style="text-align: right">
          <el-button @click="download('zip')">下载为ZIP</el-button>
          <el-button type="primary" @click="download('folder')">下载到文件夹</el-button>
        </div>
      </template>
    </el-dialog>
    <div class="loading" v-show="isLoading">下载进度：{{ process }}%</div>
  </div>
</template>
<script>
  function getBlob(url, cb) {
    let xhr = new XMLHttpRequest();
    xhr.open('GET', url, true);
    xhr.responseType = 'blob';
    xhr.timeout = 30000;
    xhr.onload = function () {
      if (xhr.status === 200) {
        cb(xhr.response);
      } else {
        cb(null);
      }
    };
    xhr.onerror = function () {
      cb(null);
    };
    xhr.ontimeout = function () {
      cb(null);
    };
    xhr.send();
  }

  function fetchTileWithRetry(url, maxRetries) {
    return new Promise((resolve) => {
      let attempt = 0;
      function tryFetch() {
        getBlob(url, (res) => {
          if (res) {
            resolve(res);
          } else if (attempt < maxRetries) {
            attempt++;
            setTimeout(tryFetch, 1000 * attempt);
          } else {
            resolve(null);
          }
        });
      }
      tryFetch();
    });
  }

  function delay(ms) {
    return new Promise((r) => setTimeout(r, ms));
  }
  const PI = Math.PI;
  const EE = 0.00669342162296594323;
  const A = 6378245.0;

  function outOfChina(lng, lat) {
    return lng < 72.004 || lng > 137.8347 || lat < 0.8293 || lat > 55.8271;
  }

  function transformLatHelper(lng, lat) {
    let ret = -100.0 + 2.0 * lng + 3.0 * lat + 0.2 * lat * lat +
      0.1 * lng * lat + 0.2 * Math.sqrt(Math.abs(lng));
    ret += (20.0 * Math.sin(6.0 * lng * PI) + 20.0 * Math.sin(2.0 * lng * PI)) * 2.0 / 3.0;
    ret += (20.0 * Math.sin(lat * PI) + 40.0 * Math.sin(lat / 3.0 * PI)) * 2.0 / 3.0;
    ret += (160.0 * Math.sin(lat / 12.0 * PI) + 320 * Math.sin(lat * PI / 30.0)) * 2.0 / 3.0;
    return ret;
  }

  function transformLngHelper(lng, lat) {
    let ret = 300.0 + lng + 2.0 * lat + 0.1 * lng * lng +
      0.1 * lng * lat + 0.1 * Math.sqrt(Math.abs(lng));
    ret += (20.0 * Math.sin(6.0 * lng * PI) + 20.0 * Math.sin(2.0 * lng * PI)) * 2.0 / 3.0;
    ret += (20.0 * Math.sin(lng * PI) + 40.0 * Math.sin(lng / 3.0 * PI)) * 2.0 / 3.0;
    ret += (150.0 * Math.sin(lng / 12.0 * PI) + 300.0 * Math.sin(lng / 30.0 * PI)) * 2.0 / 3.0;
    return ret;
  }

  function wgs84ToGcj02(wgsLng, wgsLat) {
    if (outOfChina(wgsLng, wgsLat)) return [wgsLng, wgsLat];
    let dLat = transformLatHelper(wgsLng - 105.0, wgsLat - 35.0);
    let dLng = transformLngHelper(wgsLng - 105.0, wgsLat - 35.0);
    let radLat = wgsLat / 180.0 * PI;
    let magic = Math.sin(radLat);
    magic = 1 - EE * magic * magic;
    let sqrtMagic = Math.sqrt(magic);
    dLat = (dLat * 180.0) / ((A * (1 - EE)) / (magic * sqrtMagic) * PI);
    dLng = (dLng * 180.0) / (A / sqrtMagic * Math.cos(radLat) * PI);
    return [wgsLng + dLng, wgsLat + dLat];
  }

  function lon2tilex(lon, zoom) {
    return Math.floor(((lon + 180) / 360) * Math.pow(2, zoom));
  }
  function lat2tiley(lat, zoom) {
    return Math.floor(
      ((1 -
        Math.log(Math.tan((lat * Math.PI) / 180) + 1 / Math.cos((lat * Math.PI) / 180)) / Math.PI) /
        2) *
        Math.pow(2, zoom)
    );
  }
  let zooms = [];
  for (let i = 3; i <= 18; i++) {
    zooms.push(i);
  }
  import JSZip from 'jszip';
  import { saveAs } from 'file-saver-fixed';
  export default {
    name: 'OfflineAMap',
    data: () => ({
      process: 0,
      selectStyle: '7',
      styleOps: [
        { label: '普通地图', value: '7' },
        { label: '卫星地图', value: '6' },
        { label: '路况地图', value: '8' }
      ],
      rule: `tiles/[z]/[y]/[x].png`,
      zoomMap: {},
      isShow: false,
      zooms,
      map: null,
      zoom: 11,
      searchKey: '',
      mouseTool: null,
      marker: null,
      lat: 39.90923,
      lng: 116.397428,
      isLock: false,
      rect: null,
      theZip: null,
      key: '',
      amapKey: '',
      isLoading: false,
      extraLayer: null,
      mapLoaded: false,
    }),
    computed: {
      tableData() {
        if (this.rect) {
          let list = [];
          this.zooms.forEach((z) => {
            list.push({ level: z, num: this.getTileCount(z) });
          });
          return list;
        }
        return [];
      },
      centerLnglat() {
        if (this.rect) {
          return this.rect.getOptions().bounds.getCenter().toString();
        }
        return '';
      },
      rectLngLat() {
        if (this.rect) {
          return this.rect.getOptions().bounds.toString();
        }
        return null;
      }
    },
    methods: {
      loadAMap() {
        let match = this.key.match(/key=([^&]+)/);
        if (!match) {
          this.$message.warning('URL中需包含key=参数');
          return;
        }
        this.amapKey = match[1];
        if (this.mapLoaded) {
          this.initMap();
          return;
        }
        let script = document.createElement('script');
        script.src = this.key;
        document.head.appendChild(script);
        script.onload = () => {
          this.mapLoaded = true;
          this.$message.success('地图加载成功');
          this.initMap();
        };
        script.onerror = () => {
          this.$message.error('地图脚本加载失败，请检查URL');
        };
      },
      initMap() {
        this.map = new AMap.Map(this.$refs.canvas, {
          resizeEnable: true,
          zoom: this.zoom,
          center: [this.lng, this.lat]
        });
        this.marker = new AMap.Marker();
        this.marker.setPosition([this.lng, this.lat]);
        this.map.add(this.marker);
        this.map.on('zoomend', () => { this.zoomCenter(); });
        this.map.on('moveend', () => { this.zoomCenter(); });
        AMap.plugin(['AMap.MouseTool', 'AMap.RectangleEditor'], () => {
          this.mouseTool = new AMap.MouseTool(this.map);
        });
      },
      onEditRectEnd() {
        if (this.rectangleEditor) this.rectangleEditor.close();
      },
      onEditRectStart() {
        if (this.rectangleEditor) this.rectangleEditor.open();
      },
      getTileCount(zoom) {
        let b = this.rect.getOptions().bounds;
        let x = lon2tilex(b.northEast.lng, zoom);
        let y = lat2tiley(b.northEast.lat, zoom);

        let x1 = lon2tilex(b.southWest.lng, zoom);
        let y1 = lat2tiley(b.southWest.lat, zoom);

        let startx = Math.min(x, x1),
          endx = Math.max(x, x1);
        let starty = Math.min(y, y1),
          endy = Math.max(y, y1);

        return (endx - startx + 1) * (endy - starty + 1);
      },
      async writeBlobToFolder(x, y, z, tilesDir) {
        let url = `http://wprd04.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=${
          this.selectStyle || 7
        }&x=${x}&y=${y}&z=${z}`;
        let res = await fetchTileWithRetry(url, 3);
        if (!res) return false;
        let zDir = await tilesDir.getDirectoryHandle(String(z), { create: true });
        let yDir = await zDir.getDirectoryHandle(String(y), { create: true });
        let fileHandle = await yDir.getFileHandle(`${x}.png`, { create: true });
        let writable = await fileHandle.createWritable();
        await writable.write(res);
        await writable.close();
        return true;
      },
      writeBlob(x, y, z) {
        let url = `http://wprd04.is.autonavi.com/appmaptile?lang=zh_cn&size=1&style=${
          this.selectStyle || 7
        }&x=${x}&y=${y}&z=${z}`;
        return fetchTileWithRetry(url, 3).then((res) => {
          if (res) {
            this.theZip.file(`tiles/${z}/${y}/${x}.png`, res);
            return true;
          }
          return false;
        });
      },
      getTileLayer() {
        let b = this.rect.getOptions().bounds;
        let list = [];
        for (let k in this.zoomMap) {
          if (this.zoomMap[k]) {
            let z = parseInt(k);
            let x = lon2tilex(b.northEast.lng, z);
            let y = lat2tiley(b.northEast.lat, z);

            let x1 = lon2tilex(b.southWest.lng, z);
            let y1 = lat2tiley(b.southWest.lat, z);

            let startx = Math.min(x, x1),
              endx = Math.max(x, x1);
            let starty = Math.min(y, y1),
              endy = Math.max(y, y1);

            for (let i = startx; i <= endx; i++) {
              for (let j = starty; j <= endy; j++) {
                list.push({ x: i, y: j, z });
              }
            }
          }
        }
        return list;
      },
      download(mode) {
        if (mode === 'folder' && !window.showDirectoryPicker) {
          this.$message.warning('当前浏览器不支持直接写入文件夹，请使用Chrome/Edge浏览器，或选择ZIP下载');
          return;
        }
        let tiles = this.getTileLayer();
        this.$msgbox({
          title: '是否下载?',
          message: `共 ${tiles.length} 个瓦片，大概需要${(tiles.length * 0.1 / 6).toFixed(2)}秒`,
          showConfirmButton: true,
          showCancelButton: true
        }).then(async () => {
          this.isShow = false;
          this.isLoading = true;
          this.process = 0;
          let failCount = 0;

          if (mode === 'folder') {
            let rootDir;
            try {
              rootDir = await window.showDirectoryPicker({ mode: 'readwrite' });
            } catch {
              this.isLoading = false;
              return;
            }
            let tilesDir = await rootDir.getDirectoryHandle('tiles', { create: true });

            for (let i = 0; i < tiles.length; i += 6) {
              let batch = tiles.slice(i, i + 6);
              let results = await Promise.all(
                batch.map((t) => this.writeBlobToFolder(t.x, t.y, t.z, tilesDir))
              );
              failCount += results.filter((r) => !r).length;
              this.process = (((i + batch.length) / tiles.length) * 100).toFixed(2);
              if (i + 6 < tiles.length) await delay(200);
            }
            this.process = 100;

            let selectedZooms = Object.keys(this.zoomMap).filter(k => this.zoomMap[k]).map(Number);
            let minZ = Math.min(...selectedZooms);
            let maxZ = Math.max(...selectedZooms);
            let readmeHandle = await rootDir.getFileHandle('README.md', { create: true });
            let writable = await readmeHandle.createWritable();
            await writable.write(`# 文件夹目录\n${this.rule}\n\n# 当前地图瓦片\n范围:${this.rectLngLat}\n中心点:${this.centerLnglat}\n最小缩放:${minZ}\n最大缩放:${maxZ}`);
            await writable.close();

            this.isLoading = false;
            if (failCount > 0) {
              this.$message.warning(`下载完成，但有 ${failCount} 个瓦片下载失败`);
            } else {
              this.$message.success('全部瓦片已写入文件夹');
            }
          } else {
            getBlob('tiles.zip', (res) => {
              JSZip.loadAsync(res).then(async (zip) => {
                this.theZip = zip;

                for (let i = 0; i < tiles.length; i += 6) {
                  let batch = tiles.slice(i, i + 6);
                  let results = await Promise.all(
                    batch.map((t) => this.writeBlob(t.x, t.y, t.z))
                  );
                  failCount += results.filter((r) => !r).length;
                  this.process = (((i + batch.length) / tiles.length) * 100).toFixed(2);
                  if (i + 6 < tiles.length) await delay(200);
                }
                this.process = 100;

                let selectedZooms = Object.keys(this.zoomMap).filter(k => this.zoomMap[k]).map(Number);
                let minZ = Math.min(...selectedZooms);
                let maxZ = Math.max(...selectedZooms);
                this.theZip.file(
                  `README.md`,
                  `# 文件夹目录\n${this.rule}\n\n# 当前地图瓦片\n范围:${this.rectLngLat}\n中心点:${this.centerLnglat}\n最小缩放:${minZ}\n最大缩放:${maxZ}`
                );
                this.theZip.generateAsync({ type: 'blob' }).then((blob) => {
                  saveAs(blob, '离线高德地图瓦片' + new Date().getTime() + '.zip');
                  if (failCount > 0) {
                    this.$message.warning(`下载完成，但有 ${failCount} 个瓦片下载失败`);
                  } else {
                    this.$message.success('全部瓦片下载成功');
                  }
                });
                this.isLoading = false;
              });
            });
          }
        });
      },
      async onSearch() {
        if (!this.map) {
          this.$message.warning('请先加载地图');
          return;
        }
        if (!this.searchKey) {
          this.$message.warning('请输入搜索关键词');
          return;
        }
        try {
          let res = await fetch(
            `https://nominatim.openstreetmap.org/search?format=json&q=${encodeURIComponent(this.searchKey)}&limit=1&accept-language=zh`,
            { headers: { 'User-Agent': 'OfflineMapDownloader/1.0' } }
          );
          let data = await res.json();
          if (data && data.length > 0) {
            let wgsLng = parseFloat(data[0].lon);
            let wgsLat = parseFloat(data[0].lat);
            let [gcjLng, gcjLat] = wgs84ToGcj02(wgsLng, wgsLat);
            let lnglat = [gcjLng, gcjLat];
            this.marker.setPosition(lnglat);
            this.map.setCenter(lnglat);
            this.map.setZoom(13);
            this.$message.success('已定位到: ' + data[0].display_name);
          } else {
            this.$message.error('未找到该地点，请检查输入');
          }
        } catch (e) {
          console.error(e);
          this.$message.error('搜索请求失败，请检查网络');
        }
      },
      clearRect() {
        if (this.rectangleEditor) {
          this.rectangleEditor.close();
          this.map.remove(this.rectangleEditor);
        }
        if (this.rect) {
          this.map.remove(this.rect);
        }

        this.rect = null;
      },
      drawRect() {
        this.clearRect();
        this.$nextTick(() => {
          this.mouseTool.rectangle({
            fillColor: '#1e90ff',
            fillOpacity: 0.2,
            strokeColor: '#1e90ff'
          });
          this.mouseTool.on('draw', (e) => {
            this.rect = e.obj;
            this.mouseTool.close(false);
            this.rectangleEditor = new AMap.RectangleEditor(this.map, this.rect);

            this.rectangleEditor.open();
          });
        });
      },

      zoomCenter() {
        if (!this.isLock) {
          this.isLock = true;
          setTimeout(() => {
            this.zoom = Math.round(this.map.getZoom());
            let center = this.map.getCenter();
            this.lat = center.lat;
            this.lng = center.lng;
            this.isLock = false;
          }, 1000);
        }
      },
      switchMapLayer(style) {
        this.selectStyle = style;
        if (!this.map) return;
        if (this.extraLayer) {
          this.map.remove(this.extraLayer);
          this.extraLayer = null;
        }
        if (style === '6') {
          this.extraLayer = new AMap.TileLayer.Satellite();
          this.map.add(this.extraLayer);
        } else if (style === '8') {
          this.extraLayer = new AMap.TileLayer.Traffic({ autoRefresh: true });
          this.map.add(this.extraLayer);
        }
      }
    },
    mounted() {
      if (window.AMap) {
        this.initMap();
      }
    }
  };
</script>
<style lang="scss" scoped>
  .loading {
    position: fixed;
    z-index: 99999;
    background-color: rgba(0, 0, 0, 0.5);
    color: white;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 18px;
    height: 100%;
    width: 100%;
    top: 0px;
    left: 0px;
  }
  .offline-amap-container {
    position: absolute;
    height: 100%;
    width: 100%;
    color: black;
    button {
      border: none;
      background: linear-gradient(135deg, #4a90d9, #357abd);
      min-width: 100px;
      color: white;
      height: 38px;
      border-radius: 6px;
      font-size: 14px;
      cursor: pointer;
      transition: all 0.25s ease;
      box-shadow: 0 2px 6px rgba(74, 144, 217, 0.3);
      &:hover {
        transform: translateY(-1px);
        box-shadow: 0 4px 12px rgba(74, 144, 217, 0.4);
        background: linear-gradient(135deg, #5a9ee6, #4a90d9);
      }
      &:active {
        transform: translateY(0) scale(0.97);
        box-shadow: 0 1px 4px rgba(74, 144, 217, 0.3);
      }
    }
    .map-style-switcher {
      position: fixed;
      bottom: 44px;
      left: 12px;
      z-index: 1000;
      display: flex;
      background: rgba(255, 255, 255, 0.95);
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0, 0, 0, 0.15);
      overflow: hidden;
      backdrop-filter: blur(8px);
      .style-item {
        padding: 8px 16px;
        font-size: 13px;
        cursor: pointer;
        transition: all 0.2s ease;
        color: #555;
        border-right: 1px solid #eee;
        user-select: none;
        &:last-child {
          border-right: none;
        }
        &:hover {
          background: #f0f5ff;
          color: #4a90d9;
        }
        &.active {
          background: #4a90d9;
          color: white;
          font-weight: 500;
        }
      }
    }
    .left-tool {
      position: fixed;
      top: 40%;
      left: 0px;
      z-index: 1000;
      display: inline-flex;
      flex-direction: column;
      width: 110px;
      padding: 4px;
      > button {
        width: 100%;
        margin: 0px;
        border-radius: 6px;
        font-size: 13px;
        height: 36px;
      }
      > button:not(:last-child) {
        margin-bottom: 8px;
      }
    }
    .bottom-info {
      position: fixed;
      bottom: 0px;
      display: block;
      width: 100%;
      height: 32px;
      line-height: 32px;
      font-size: 14px;
      color: white;
      text-align: center;
      z-index: 1000;
      background-color: black;
    }
    .search-container {
      position: fixed;
      top: 80px;
      left: 50%;
      transform: translateX(-50%);
      z-index: 1000;
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 6px;
      .search-row {
        display: flex;
        align-items: center;
        height: 40px;
        gap: 4px;
      }
      .key-input {
        width: 500px;
      }
      .search-input {
        width: 294px;
      }
      button {
        margin: 0;
        height: 38px;
        white-space: nowrap;
      }
      input,
      select {
        background-color: white;
        border: 1px solid #d0d7de;
        height: 38px;
        padding: 0 10px;
        line-height: 1;
        outline: none;
        font-size: 14px;
        border-radius: 6px;
        transition: border-color 0.2s ease, box-shadow 0.2s ease;
        &:focus {
          border-color: #4a90d9;
          box-shadow: 0 0 0 3px rgba(74, 144, 217, 0.15);
        }
        &::placeholder,
        &::-webkit-input-placeholder,
        &::-moz-placeholder {
          color: #aaa;
        }
      }
    }
    .canvas {
      height: 100%;
      width: 100%;
    }
  }
</style>
