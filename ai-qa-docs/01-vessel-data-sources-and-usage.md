# 船舶信息来源与系统应用

**问题：情报系统中，船舶信息的来源，系统中是怎么应用的？**

---

## 一、数据来源

### 1. 民用 AIS — AISStream.io
- **协议**：WebSocket，连接 `wss://stream.aisstream.io/v0/stream`
- **入口**：`scripts/ais-relay.cjs`（中继服务器）
- **认证**：需要 `AISSTREAM_API_KEY` 环境变量
- **内容**：船舶实时位置报告（MMSI、经纬度、航速、航向）

### 2. 军事船只数据
- **入口**：`src/services/military-vessels.ts`
- **方法**：在 AIS 数据基础上，通过 MMSI 号段分析（150+ 国家映射）+ 已知舰艇数据库 + AIS 船型代码识别潜在军事船只
- **叠加来源**：USNI Fleet Tracker（`src/services/usni-fleet.ts`）

---

## 二、数据处理与缓存管道

### 中继服务器（ais-relay.cjs）内存结构

| 数据结构 | 说明 | 容量上限 |
|---------|------|---------|
| `vessels` Map | 全部跟踪船只（MMSI、位置、航速等） | 20,000 艘 |
| `densityGrid` Map | 2° 网格单元聚合船只密度 | 5,000 格 |
| `candidateReports` Map | 高优先级军事候选船只 | 保留 2 小时 |

### 快照生成逻辑（每 5 秒）

- 聚合密度区域（最多 200 个）
- 检测两类 AIS 异常：
  - `gap_spike`：AIS 信号突然消失（暗船/dark ships，阈值 1 小时无报告）
  - `chokepoint_congestion`：战略咽喉点异常拥堵
- 预序列化为 JSON + gzip/brotli 压缩，备用

### 缓存层级

| 层级 | 策略 |
|------|------|
| 中继内存 | 5s 刷新，预压缩缓冲 |
| Upstash Redis | 咽喉点过境事件，TTL 1h |
| HTTP 缓存头 | 60s max-age + 300s CDN + 600s stale-while-revalidate |
| 浏览器轮询 | 每 5 分钟，6 分钟过期判定 |

---

## 三、前端消费链路

### 完整数据流

```
AISStream.io (WebSocket)
    ↓
ais-relay.cjs（5s 聚合 + 异常检测）
    ↓
/ais/snapshot HTTP 端点（预压缩）
    ↓
api/ais-snapshot.js（Vercel Edge Function）
    ↓
src/services/maritime/index.ts（fetchAisSignals，5分钟轮询）
    ↓
DeckGLMap + StrategicPosturePanel + SanctionsPressurePanel
    + military-vessels.ts（军事船只叠加层）
```

### 前端消费组件

| 组件 | 文件 | 用途 |
|------|------|------|
| DeckGLMap | `src/components/DeckGLMap.ts` | 地图渲染：AIS 密度层、异常层、军事船只层 |
| StrategicPosturePanel | `src/components/StrategicPosturePanel.ts` | 军事船只摘要与态势评估 |
| SanctionsPressurePanel | `src/components/SanctionsPressurePanel.ts` | 制裁船只数量展示 |
| data-loader | `src/app/data-loader.ts` | 统一编排 AIS + 军事数据加载 |

---

## 四、核心数据类型（Proto & TypeScript）

### Proto 定义位置
- `proto/worldmonitor/maritime/v1/vessel_snapshot.proto`
- `proto/worldmonitor/maritime/v1/get_vessel_snapshot.proto`
- `proto/worldmonitor/maritime/v1/service.proto`

### 关键类型

```typescript
// AIS 密度区域
interface AisDensityZone {
  id: string; name: string;
  lat: number; lon: number;
  intensity: number;    // 0-1 船只密度
  deltaPct: number;     // 相对基线变化百分比
  shipsPerDay?: number;
}

// AIS 异常事件
interface AisDisruptionEvent {
  id: string; name: string;
  type: 'gap_spike' | 'chokepoint_congestion';
  severity: 'low' | 'elevated' | 'high';
  darkShips?: number;   // 信号消失船只数
  vesselCount?: number;
  description: string;
}
```

---

## 五、关键参数配置

| 参数 | 值 | 位置 |
|------|-----|------|
| 最大跟踪船只数 | 20,000 | `ais-relay.cjs:64` |
| AIS 暗船判定阈值 | 1 小时无报告 | `ais-relay.cjs:6220` |
| 快照生成间隔 | 5 秒 | `ais-relay.cjs:6221` |
| 浏览器轮询间隔 | 5 分钟 | `maritime/index.ts:132` |
| 监控咽喉点数量 | 17 个（霍尔木兹、苏伊士、马六甲等） | `military-vessels.ts` |

---

## 六、关键文件索引

| 职责 | 文件路径 |
|------|---------|
| AIS 中继服务器 | `scripts/ais-relay.cjs` |
| Edge 端点 | `api/ais-snapshot.js` |
| 前端 AIS 服务 | `src/services/maritime/index.ts` |
| 军事船只检测 | `src/services/military-vessels.ts` |
| 地图渲染 | `src/components/DeckGLMap.ts` |
| 数据编排 | `src/app/data-loader.ts` |
