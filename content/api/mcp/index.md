---
date: 2018-09-29T20:00:00+08:00
title: MCP API
weight: 360
description : "介绍istio的MCP API"
---

Istio MCP API

MCP是 Mesh Configuration Protocol 的简写。

MCP 的定义在 istio/api 项目中，路径为 `mcp/v1alpha1`。

## 资源相关定义

### Resource

Resource 通过 Mesh Configuration Protocol 传输的资源。每个资源由公共元数据和特定于类型的资源有效负载组成。

| Field      | Type                  | Description          |
| ---------- | --------------------- | -------------------- |
| `metadata` | `Metadata`            | 描述资源的公共元数据 |
| `body`     | `google.protobuf.Any` | 资源的主负载         |

### Metadata

Metadata 是 Mesh Configuration Protocol中所有资源必须具有的元数据信息。

| Field         | Type                        | Description                                                  |
| ------------- | --------------------------- | ------------------------------------------------------------ |
| `name`        | `string`                    | 资源的完全限定名称。在集合的上下文中是唯一的。<br/><br/>完全限定名称由目录(directory)和基本名称(basename)组成。directory标识资源在资源层次结构中的位置。 basename标识该目录上下文中的特定资源名称。<br/><br/>directory和basename由一个或多个段(Segment)组成。段必须是有效的[DNS labels](https://tools.ietf.org/html/rfc1123)。"/"是段之间的分隔符。最右边的段是基本名称。 basename左侧的所有段构成目录。向左移动的段表示资源层次结构中较高的位置，类似于反向DNS表示法。<br/><br/> `/<org>/<team>/<subteam>/<resource basename>` <br/><br/> 空目录表示资源位于层次结构的根目录，例如<br/><br/> `/<globally scoped resource>` <br/><br/> Kubernetes资源层次结构有两个级别：namespaces和集群作用域（如global）。<br/><br/>Namespace资源的完全限定名称形式是：<br/><br/>  `/<k8s namespace>/<k8s resource name>` <br/><br/> 集群作用域的资源位于层次结构的根目录，其格式为：<br/><br/> `/<k8s resource name>` |
| `createTime`  | `google.protobuf.Timestamp` | 资源的创建时间                                               |
| `version`     | `string`                    | 资源版本。 这用于在资源更新时确定资源是何时更改。它应该被消费者/接收器(sinks)视为不透明。 |
| `labels`      | `map<string, string>`       | 字符串键和值的映射，可用于组织和分类集合中的资源。           |
| `annotations` | `map<string, string>`       | 字符串键和值的映射, source和sink可用于传递关于此资源的任意元数据。 |

### Resources

Resources不需要包含跟踪资源的完整快照。相反，它们是MCP客户端状态的差值。每资源版本允许源和接收器以资源粒度跟踪状态。MCP增量会话始终位于gRPC双向流的上下文中。这允许 MCP source 跟踪连接到它的 MCP sink 的状态。

在Incremental MCP中，nonce字段是必需的，用于将 Resources 配对到 RequestResources ACK或NACK。

| Field               | Type         | Description                                                  |
| ------------------- | ------------ | ------------------------------------------------------------ |
| `systemVersionInfo` | `string`     | 响应数据的版本（用于调试）。                                 |
| `collection`        | `string`     | 正在请求的资源集合的类型，例如，`istio/networking/v1alpha3/VirtualService` 或 `k8s/<apiVersion>/<kind>` |
| `resources`         | `Resource[]` | 包含在通用MCP资源消息中的响应资源。 这些是与 RequestResources 消息中的类型 url 匹配的类型化资源。当 incremental 为true时，它包含为指定集合添加/更新的资源数组。这会修改sink上的现有集合。当 incremental 为 false 时，它包含指定集合的完整资源集。 这将取代以前提供的所有资源。 |
| `removedResources`  | `string[]`   | 已删除的资源的名称，并要从MCP sink 节点中删除。已删除的资源如果是缺失的，则可以忽略。当 incremental 为true时，它包含要为指定集合删除的资源名称数组。这会修改 sink 上的现有资源集合。当 incremental 为false时，应忽略此字段。 |
| `nonce`             | `string`     | 必要参数。 nonce为 RequestChange 提供了一种唯一引用 RequestResources 的方法。 |
| `incremental`       | `bool`       | 标识此资源响应是否是增量更新。如果接收器请求增量，源应该只发送增量更新。 |

### RequestResources

RequestResource可以在两种情况下发送：

MCP双向更改流中的初始消息，和作为对先前资源的ACK或NACK响应。 在这种情况下，responsenonce设置为Resources中的nonce值。 ACK/NACK由errordetail的存在确定。

- ACK (nonce!=“”,error_details==nil)
- NACK (nonce!=“”,error_details!=nil)
- New/Update request (nonce==“”,error_details ignored)

| Field                     | Type                  | Description                                                  |
| ------------------------- | --------------------- | ------------------------------------------------------------ |
| `sinkNode`                | `SinkNode`            | 发起请求的 sink node                                         |
| `collection`              | `string`              | 正在请求的资源集合的类型，例如istio/networking/v1alpha3/VirtualService k8s// |
| `initialResourceVersions` | `map<string, string>` | 当RequestResources是流中的第一个请求时，必须填充initial*resource*versions。 否则，必须省略initialresourceversions。 key是MCP客户端已知的MCP资源的资源名称。 map中的value是关联的资源级别版本信息。 |
| `responseNonce`           | `string`              | 当RequestResources是响应先前RequestResources的ACK或NACK消息时，responsenonce必须是RequestResources中的nonce。 否则必须省略responsenonce。 |
| `errorDetail`             | `google.rpc.Status`   | 当无法应用先前接收的资源时填充此信息。error_details中的 message 字段提供与故障相关的源内部错误。 |
| `incremental`             | `bool`                | 请求指定集合的增量更新。 source可以选择遵守此请求或忽略并在相应的资源响应中提供完整状态更新。 |

TBD： initialResourceVersions 是如何使用的？第一次请求时如何知道 资源版本？

## 资源交换服务

ResourceSource 和 ResourceSink 服务对于消息交换在语义上是等同的。唯一有意义的区别是谁启动连接并打开流。以下高级概述适用于两种服务变体。

在建立连接和流之后，sink 发送 RequestResource 消息以请求资源的初始集合。当有新的资源可用于请求类型时， source 会发送资源消息。作为响应，sink 发送另一个 RequestResource 以对接收的资源进行 ACK/NACK 并请求下一组资源。

### ResourceSource

sink 是gRPC客户端的服务。 sink 负责启动连接和打开流。

```proto
// Sink，作为gRPC客户端，与 source 建立新的资源流。
// Sink 向 source 发送 RequestResources 消息并从源接收 Resources 消息。
rpc EstablishResourceStream(stream RequestResources) returns (stream Resources) {}
```

### ResourceSink

source 是gRPC客户端的服务。 source 负责启动连接和打开流。

```proto
// source，作为gRPC客户端，与 sink 建立新的资源流。
// Sink 向 source 发送 RequestResources 消息并从源接收 Resources 消息。
rpc EstablishResourceStream(stream Resources) returns (stream RequestResources) {}
```

## 聚合服务

### AggregatedMeshConfigService 

聚合网格配置服务允许单个管理服务器通过单个gRPC流提供所有API更新。

```protobuf
service AggregatedMeshConfigService {
  // StreamAggregatedResources提供了跨多种资源类型进行细致的排序更新的功能。 
  // 单个流与多个独立的MeshConfigRequest/MeshConfigResponses序列一起使用，
  // 这些序列通过类型URL复用。
  rpc StreamAggregatedResources(stream MeshConfigRequest)
      returns (stream MeshConfigResponse) {
  }

  // IncrementalAggregatedResources 提供了增量更新客户端资源的功能。
  // 这支持了MCP资源可扩展性的目标。
  rpc IncrementalAggregatedResources(stream IncrementalMeshConfigRequest)
      returns (stream IncrementalMeshConfigResponse) {
  }
}
```

### MeshConfigRequest

MeshConfigRequest为给定客户端请求一组相同类型的版本化资源。

| Field           | Type                | Description                                                  |
| --------------- | ------------------- | ------------------------------------------------------------ |
| `versionInfo`   | `string`            | 请求消息中提供的 versioninfo 使用最近成功处理的响应接收的versioninfo，或者是第一个请求则设置为空。预期在收到响应之后不会发送新请求，直到客户端实例准备好对新配置进行ACK/NACK。 通过分别返回应用的新API配置版本或先前的API配置版本来进行ACK/NACK。 每个type_url（见下文）都有一个与之关联的独立版本。 |
| `sinkNode`      | `SinkNode`          | 发起请求的 sink node                                         |
| `typeUrl`       | `string`            | 正在请求的资源的类型, e.g. “type.googleapis.com/istio.io.networking.v1alpha3.VirtualService”. |
| `responseNonce` | `string`            | 对应于MeshConfigResponse的nonce，进行ACK / NACK。 请参阅上面有关version_info和MeshConfigResponse nonce注释的讨论。 如果没有可用的nonce，这可能是空的，例如，在启动时。 |
| `errorDetail`   | `google.rpc.Status` | 当前一个MeshConfigResponse无法更新配置时，将填充此选项。 error_details中的message字段提供与失败相关的客户端内部异常。 它仅供手动调试时使用，不保证在客户端版本中提供的字符串是稳定的。 |

### MeshConfigResponse

MeshConfigResponse提供一组相同类型的版本化资源以响应MeshConfigRequest。

| Field         | Type         | Description                                                  |
| ------------- | ------------ | ------------------------------------------------------------ |
| `versionInfo` | `string`     | 响应数据的版本。                                             |
| `resources`   | `Resource[]` | 包含在公共MCP *Resource* 消息中的响应资源。                  |
| `typeUrl`     | `string`     | 包含在提供的资源中的资源类型URL。 如果资源非空，这必须与包装器消息中的type_url一致。 |
| `nonce`       | `string`     | nonce提供了一种在后续的MeshConfigRequest中显式地ACK特定MeshConfigResponse的方法。 客户端可能已经在此MeshConfigResponse之前的流上向管理服务器发送了其他消息，这些消息在响应发送时未被处理。 nonce允许管理服务器忽略先前版本的任何进一步的MeshConfigRequests，直到带有nonce的MeshConfigRequest。 |