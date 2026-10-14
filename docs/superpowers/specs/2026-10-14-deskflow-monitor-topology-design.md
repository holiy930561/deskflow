# Deskflow Monitor-aware Layout 设计

日期：2026-10-14  
基线：Deskflow `v1.27.0`（protocol v1.8）  
目标分支：`feature/monitor-topology-v1.27`

## 目标

让 Deskflow 把一台计算机上的每块物理显示器作为独立布局节点，从而支持显示器跨计算机交错排列，例如：

```text
[ xdt 左屏 ] [ Mac ] [ xdt 右屏 ]
```

所有显示器尺寸、坐标、缩放、旋转和热插拔状态都由客户端运行时发现并上报。配置不得保存固定分辨率或分割坐标。

最终支持 macOS、Windows、Linux；第一阶段在 Linux/X11 客户端与 macOS 服务端之间完成本地验证。源码开发在 `xdt` Linux 上进行，macOS 和 Windows 通过 GitHub Actions 构建，当前 Mac 只承担最终安装与交互验收。

## 非目标

- 第一阶段不重写 Deskflow 的输入、剪贴板或 TLS 子系统。
- 不要求旧版 Deskflow 理解显示器级布局。
- 不使用 SSH、`xrandr` 脚本或固定像素值作为正式数据通道。
- 不把显示器 EDID、序列号等原始硬件标识写入配置或网络日志。

## 兼容策略

协议 minor version 从 v1.8 扩展到 v1.9，保留现有握手与 `DINF` 消息。

- v1.9 服务端 + v1.9 客户端：启用显示器拓扑能力。
- v1.9 服务端 + v1.8 客户端：把客户端当前 `DINF` 矩形合成为一个 `legacy:aggregate` 显示器。
- v1.8 服务端 + v1.9 客户端：客户端不得发送新消息，行为与 v1.8 相同。
- 未启用显示器级布局的用户继续使用现有计算机级 `links` 配置。

能力必须通过握手版本协商得出，不能向旧服务端发送未知消息。

## 统一数据模型

新增平台无关结构：

```cpp
struct MonitorDescriptor {
  std::string stableId;
  std::string sessionId;
  std::string displayName;
  int32_t x;
  int32_t y;
  int32_t width;
  int32_t height;
  uint32_t scaleNumerator;
  uint32_t scaleDenominator;
  MonitorTransform transform;
  bool primary;
};

struct MonitorTopology {
  uint32_t generation;
  std::vector<MonitorDescriptor> monitors;
};
```

约束：

- 坐标使用客户端鼠标 API 的原生虚拟桌面坐标单位。
- `width`、`height` 必须大于零。
- 每台计算机最多接受 32 块显示器。
- 坐标和尺寸使用 32 位有符号整数，计算边界时使用 64 位中间值防止溢出。
- 同一 generation 内的 `stableId` 与 `sessionId` 必须唯一。
- `generation` 在一次连接内单调递增，回绕按无符号序列号规则处理。

### 显示器身份

`stableId` 是平台稳定身份材料的 SHA-256 摘要，不直接暴露序列号：

- macOS：`CGDisplayCreateUUIDFromDisplayID`；不可用时使用 vendor/product/serial 的摘要。
- Windows：`QueryDisplayConfig` 返回的 target device path 与 EDID 身份材料摘要。
- Linux/X11：XRandR output 的 EDID 摘要，并混入 connector 名称避免无 EDID 设备冲突。
- Linux/Wayland：优先 compositor/Qt 暴露的 manufacturer、model、serial；信息不足时只保证 `sessionId` 稳定。

`sessionId` 用于本次连接中的精确寻址。稳定身份不足时，配置不得静默绑定到另一块显示器，而是把节点标记为待确认。

## 平台发现接口

新增：

```cpp
class MonitorTopologyProvider {
public:
  virtual MonitorTopology currentTopology() const = 0;
  virtual void start(ChangeCallback callback) = 0;
  virtual void stop() = 0;
};
```

平台实现：

- `X11MonitorTopologyProvider`：使用 XRandR 查询 output/CRTC/EDID，并监听 `RRScreenChangeNotify`、`RRCrtcChangeNotify`。
- `WaylandMonitorTopologyProvider`：使用现有 Qt/portal 可用信息；明确记录 stable identity 不可用的情况。
- `MacMonitorTopologyProvider`：使用 CoreGraphics 显示器 API，并注册 `CGDisplayRegisterReconfigurationCallback`。
- `WindowsMonitorTopologyProvider`：使用 DisplayConfig，并监听 `WM_DISPLAYCHANGE` 与设备变更。

`PlatformComputer` 负责持有 provider，并在拓扑变化时发布 `ComputerMonitorTopologyChanged` 事件。

## 协议扩展

新增查询消息 `QMTL` 和数据消息 `DMTL`。

连接流程：

1. 现有握手与 `DINF` 完成；
2. 双方协商结果支持 protocol v1.9；
3. 服务端发送 `QMTL`；
4. 客户端回复完整 `DMTL`；
5. 后续拓扑变化由客户端主动发送新的 `DMTL`。

`DMTL` 包含 generation 与一个有长度前缀的确定性二进制 payload。payload 按 network byte order 编码：

```text
schemaVersion: u16
monitorCount:  u16
repeated monitorCount times:
  stableId:        utf8-string
  sessionId:       utf8-string
  displayName:     utf8-string
  x, y, w, h:      i32 × 4
  scaleNumerator:  u32
  scaleDenominator:u32
  transform:       u16
  flags:           u16
```

安全限制：

- payload 最大 64 KiB；
- 单个字符串最大 512 bytes；
- 严格验证 UTF-8、数量、长度、重复 ID、坐标溢出与非法 transform；
- 整包验证成功后原子替换旧拓扑；解析失败继续保留上一代拓扑并记录限速日志。

## 服务端布局模型

新增 `MonitorGraph`，节点键为：

```text
computerCanonicalName / monitorStableId
```

每个节点持有运行时矩形、在线状态和四条可分段边。边只保存节点身份与边缘区间，不保存像素尺寸。

现有 `Config::CellMap` 保留，用于计算机级兼容模式。新增 monitor-aware 配置层：

```text
section: monitor-links
  mac/<monitor-id>:
    left  = xdt/<left-monitor-id>
    right = xdt/<right-monitor-id>
  xdt/<left-monitor-id>:
    right = mac/<monitor-id>
  xdt/<right-monitor-id>:
    left  = mac/<monitor-id>
end
```

序列化配置时只保存稳定 ID 与边关系。显示器名称和当前几何仅用于 UI 展示。

## 光标路由

新增 `CursorRouter`，从 `Server` 中分离以下职责：

1. 根据当前计算机坐标确定当前显示器；
2. 用上一坐标与新坐标形成运动线段；
3. 找出该线段最先穿越的显示器边；
4. 查询 `MonitorGraph` 是否为该边配置了目标节点；
5. 按源边比例映射到目标边，并把坐标推进目标显示器内部一个坐标单位；
6. 若目标属于另一台计算机，调用现有 leave/enter 流程；
7. 若目标属于同一计算机但不是自然相邻显示器，执行同客户端绝对坐标 warp；
8. 若没有自定义边，保持操作系统原有的同机显示器移动行为。

这使内部物理接缝可以被重映射。例如 `xdt` 两块显示器之间的接缝可以指向 Mac，而无需知道显示器是 1920、2560 或其他宽度。

高速移动一次穿过多个边界时，只处理第一个有效交点，并丢弃切换后的剩余 delta，避免递归切换和来回弹跳。

## 动态变化处理

客户端连接与每次显示器变化时发送完整拓扑快照。

服务端收到新 generation 后：

1. 完整校验；
2. 用 stable ID 对齐旧节点；
3. 更新几何与在线状态；
4. 原子替换拓扑；
5. 重新定位当前指针所属显示器；
6. 若当前显示器消失，把指针移动到同机 primary 显示器中心；
7. 如果配置引用的显示器离线，禁用对应边，不自动绑定相似名称的设备。

进入目标前发现缓存为空或 generation 未完成时，服务端发送 `QMTL` 并暂时把指针夹在当前边缘；收到有效拓扑后用户再次越界即可切换。

## GUI

服务器布局界面按计算机分组展示每块显示器：

- 计算机是容器；显示器是可拖动节点；
- 节点显示名称、分辨率、缩放、旋转和在线状态；
- 默认按照客户端上报的相对位置排列；
- 用户可以把不同计算机的显示器交错排列；
- 旧版客户端显示为单一 aggregate 节点；
- 稳定身份丢失时显示“需要重新绑定”，不自动替换。

第一阶段允许通过文本配置与测试 fixture 验证路由，GUI 编辑能力在核心模型稳定后接入。

## 错误处理

- 非法拓扑：拒绝整包，保留上一代数据。
- 客户端断开：节点离线，图配置保留。
- 显示器热拔出：禁用引用该节点的边。
- ID 冲突：拒绝整包并请求重新上报。
- 无 stable ID：允许 session 内使用，但不持久化自动绑定。
- 版本不支持：回退 aggregate 模式。
- 消息洪泛：同一客户端拓扑更新按时间窗口限速并合并为最新 generation。

## 测试策略

### 单元测试

- 拓扑编码/解码与所有非法输入；
- protocol v1.8/v1.9 协商矩阵；
- stable ID 对齐、热插拔、重排和 generation 处理；
- 水平、垂直、旋转、负坐标、不同 DPI 的边缘映射；
- 内部接缝、外边缘、边缘区间与高速跨越；
- 旧配置和旧客户端 aggregate 回退。

### 平台测试

- X11：使用 XRandR fixture 和 `xrandr` 虚拟/实际双屏布局；
- Wayland：provider 数据可用性与 fallback；
- macOS：CoreGraphics 枚举、重排回调、权限和光标 warp；
- Windows：DisplayConfig 枚举、`WM_DISPLAYCHANGE` 和 DPI/旋转。

### CI

GitHub Actions 建立 Ubuntu、macOS、Windows 构建矩阵：

- 每个平台运行格式检查、静态分析、单元测试和原生构建；
- 协议序列化测试使用同一组 golden vectors；
- macOS 产出未公证测试 `.app` artifact，供当前 Mac 安装验收；
- Windows 产出测试安装包；
- Linux 产出原生包或 Flatpak 测试 artifact；
- CI 使用实现时 GitHub 官方仍支持的 hosted runner 标签，避免把会退役的 runner 名称固化进架构契约。

## 本地第一阶段验收

环境：当前 Mac 作为服务端，`xdt` Linux/X11 作为客户端。

必须满足：

1. `xdt` 自动上报两块实际显示器，不读取固定分辨率配置；
2. Mac 左边缘进入 `xdt` 物理左屏；
3. Mac 右边缘进入 `xdt` 物理右屏；
4. `xdt` 两块屏幕的内侧边缘进入 Mac；
5. 修改任一屏分辨率或交换排列后，无需重启即可更新；
6. 断开一块屏幕后不会进入无效坐标；
7. TLS、键盘、剪贴板和现有热键没有回归；
8. v1.27 官方客户端连接时仍可按 aggregate 模式使用。

## 交付阶段

1. 协议结构、golden vectors 与纯逻辑 `MonitorGraph/CursorRouter`；
2. Linux/X11 provider 与 Linux 测试客户端；
3. Mac 服务端构建和真实三屏交错布局验证；
4. GUI monitor layout；
5. macOS provider；
6. Windows provider；
7. Linux/Wayland provider 完善、兼容性与上游化整理。

## 回滚

- 当前官方 Deskflow 应用和配置不在开发阶段原地覆盖。
- 测试版本使用独立应用名称、配置目录或显式 `--settings` 文件。
- Git 分支以 `v1.27.0` 为基线，任何阶段均可切回官方安装包。

