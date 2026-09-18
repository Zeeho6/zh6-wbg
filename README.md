# WebGIS 交互式地图平台

基于 Vue3 + OpenLayers 的交互式 Web 地图应用，实现城市搜索、行政区划高亮、地图绘制、标记聚合、底图切换等功能。项目用于 WebGIS 前端开发实践，结合测绘 GIS 业务背景，探索空间数据的浏览器端可视化。

## 在线预览

🔗 https://zh6-wbg.vercel.app/

## 功能

- **底图切换**：天地图矢量底图 + 矢量注记、遥感影像 + 影像注记，一键切换
- **城市搜索**：输入城市名，通过高德地理编码获取中心坐标与 adcode，定位并高亮行政区划边界
- **IP 定位**：页面加载时通过高德 IP 定位接口识别当前城市，初始化定位
- **行政区划渲染**：基于本地全国行政区划 GeoJSON，按 adcode 匹配 Feature 并渲染边界
- **地图绘制**：支持直线、圆形、多边形、自由画笔，提供退出绘图与清除绘图
- **标记与聚合**：充电站 / 公交站 / 停车场三类标记，使用 OpenLayers `Cluster` 按像素距离聚合
- **聚合展开**：点击聚合点自动放大地图，使聚合点散开
- **悬停弹窗**：鼠标停留在行政区划边界上显示城市名称与中心经纬度
- **缩放联动**：缩放级别大于 10 时隐藏行政区划与绘制图层，避免遮挡

## 技术栈

| 分类 | 技术 |
|------|------|
| 框架 | Vue3（`<script setup>`） + Vite |
| 地图引擎 | OpenLayers |
| UI 组件 | Element Plus |
| 样式 | Sass |
| 地理编码 | 高德 Web 服务 API（IP 定位、地理编码） |
| 瓦片服务 | 天地图 WMTS |
| 数据源 | 本地全国行政区划 GeoJSON |

## 项目结构
src/
├── assets/
│ └── china.json # 全国行政区划 GeoJSON（本地化）
├── App.vue # 地图主组件
├── main.js
└── style.css

## 核心实现

### 1. 地图初始化

- `Map` + `View(EPSG:4326)` + `TileLayer` + `VectorLayer`
- 天地图矢量底图与注记分层加载，通过 `setVisible` 控制显示
- `Overlay` 承载悬停弹窗，全局仅创建一个实例

### 2. 城市搜索与行政区划渲染

- 高德地理编码接口返回城市中心坐标和 adcode
- 从本地 `china.json` 中通过 `allFeatures.filter()` 按 adcode 匹配 Feature
- 使用 `view.animate` 的 `done` 回调实现「先飞后显」的过渡效果

### 3. 绘制工具

- 基于 OpenLayers `Draw` 交互封装直线、圆形、多边形、自由画笔
- 退出绘图：移除交互，保留已绘图形
- 清除绘图：清空绘制图层

### 4. 标记与聚合

- `Cluster` 源按 40px 距离聚合点要素
- 自定义 `styleFunction`：聚合点显示数量与橙色圆圈，单点显示对应图标
- 通过 `layerFilter` 定位标记图层，聚合点多于 1 个时触发地图放大

### 5. 悬停弹窗

- `pointermove` + `layerFilter` 精确定位行政区划图层
- 500ms 延迟触发，避免鼠标快速划过时闪烁
- 弹窗使用 `pointer-events: none` 实现鼠标事件穿透

### 6. 绘制与标记互斥

- 切换绘制类型时主动退出标记模式
- 切换标记类型时主动移除绘制交互
- 保证同一时刻只有一个交互生效

## 本地运行

```bash
npm install
npm run dev
默认启动在 http://localhost:5173

```
## 环境变量

VITE_AMAP_KEY=高德key
VITE_TIANDITU_KEY=天地图key

说明：

- 高德 Key 类型需为「Web 服务」，并开通「IP 定位」「地理编码」服务
- 天地图 tk 类型为「浏览器端」，域名白名单可留空（不限制）或加入你的域名
- `.env.local` 已在 `.gitignore` 中，不会提交到仓库

## 后续计划

- 接入 GeoServer + PostGIS，支持真实空间数据服务
- 增加三维地图（Cesium）
- 支持要素属性查询与编辑
- 增加更多底图类型（地形、自定义瓦片）
- 在 `onUnmounted` 中清理地图实例，避免内存泄漏

