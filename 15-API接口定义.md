# 09. API接口定义

> 通信架构设计详见 **[14-通信架构设计](./14-通信架构设计.md)**（gRPC核心、gRPC-Web、HTTP/JSON、API文档生成）

---

## 9.0 服务概览

SyncTV 定义了三类 gRPC 服务：

| 服务 | 用途 | 通信方向 |
|-----|------|---------|
| **ClientService** | 客户端通信 | 客户端 ↔ 服务器 |
| **ClusterService** | 集群管理 | 节点 ↔ 节点 |
| **StreamService** | 流转发 | 节点 ↔ 节点 |

**Proto 文件组织**：

```
proto/
├── client.proto        # ClientService - 客户端服务
├── cluster.proto       # ClusterService - 集群同步
├── stream.proto        # StreamService - 流转发
├── common.proto        # 公共消息类型
└── models.proto        # 数据模型
```

---

## 9.1 ClientService（客户端服务）

### 9.1.1 服务定义

```protobuf
syntax = "proto3";
package synctv.client;

service ClientService {
  // 认证
  rpc Login(LoginRequest) returns (LoginResponse);
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc RefreshToken(RefreshTokenRequest) returns (RefreshTokenResponse);
  rpc Logout(LogoutRequest) returns (LogoutResponse);

  // 房间管理
  rpc CreateRoom(CreateRoomRequest) returns (CreateRoomResponse);
  rpc JoinRoom(JoinRoomRequest) returns (JoinRoomResponse);
  rpc LeaveRoom(LeaveRoomRequest) returns (LeaveRoomResponse);
  rpc GetRoomInfo(GetRoomInfoRequest) returns (GetRoomInfoResponse);
  rpc ListRooms(ListRoomsRequest) returns (ListRoomsResponse);

  // 影片管理
  rpc AddMovie(AddMovieRequest) returns (AddMovieResponse);
  rpc RemoveMovie(RemoveMovieRequest) returns (RemoveMovieResponse);
  rpc GetMovieList(GetMovieListRequest) returns (GetMovieListResponse);
  rpc SetCurrentMovie(SetCurrentMovieRequest) returns (SetCurrentMovieResponse);

  // 播放控制
  rpc Play(PlayRequest) returns (PlayResponse);
  rpc Pause(PauseRequest) returns (PauseResponse);
  rpc Seek(SeekRequest) returns (SeekResponse);
  rpc SetPlaybackRate(SetPlaybackRateRequest) returns (SetPlaybackRateResponse);

  // 实时消息流（替代 WebSocket）
  rpc MessageStream(stream ClientMessage) returns (stream ServerMessage);
}
```

### 9.1.2 认证接口

#### Login - 用户登录

```protobuf
message LoginRequest {
  string username = 1;
  string password = 2;
}

message LoginResponse {
  User user = 1;
  string access_token = 2;
  string refresh_token = 3;
  int64 expires_in = 4;
}
```

#### Register - 用户注册

```protobuf
message RegisterRequest {
  string username = 1;
  string password = 2;
  string email = 3;
}

message RegisterResponse {
  User user = 1;
  string access_token = 2;
  string refresh_token = 3;
}
```

### 9.1.3 实时消息流

#### MessageStream - 双向流通信

详细的消息流设计参见 **[16-实时消息流](./16-实时消息流.md)**

```protobuf
// 客户端 → 服务端
message ClientMessage {
  string message_id = 1;
  int64 timestamp = 2;

  oneof payload {
    PingMessage ping = 10;
    ChatMessage chat = 11;
    DanmakuMessage danmaku = 12;
    PlayControlMessage play_control = 13;
    SeekMessage seek = 14;
  }
}

// 服务端 → 客户端
message ServerMessage {
  string message_id = 1;
  int64 timestamp = 2;

  oneof payload {
    PongMessage pong = 10;
    RoomStateUpdate room_state = 11;
    ChatBroadcast chat = 12;
    DanmakuBroadcast danmaku = 13;
    MemberUpdate member = 14;
    ErrorMessage error = 15;
  }
}
```

---

## 9.2 ClusterService（集群同步服务）

### 9.2.1 服务定义

```protobuf
syntax = "proto3";

package synctv.cluster;

// 集群节点服务
service ClusterService {
  // 注册节点
  rpc RegisterNode(RegisterNodeRequest) returns (RegisterNodeResponse);

  // 心跳
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);

  // 获取所有节点
  rpc GetNodes(GetNodesRequest) returns (GetNodesResponse);

  // 同步事件流（双向流）
  rpc SyncEvents(stream ClusterEvent) returns (stream ClusterEvent);
}
```

### 9.1.2 节点注册

```protobuf
message RegisterNodeRequest {
  string node_id = 1;
  string address = 2;
  NodeCapabilities capabilities = 3;
}

message RegisterNodeResponse {
  bool success = 1;
  repeated NodeInfo nodes = 2;  // 所有节点列表
}

message NodeCapabilities {
  bool can_handle_rtmp = 1;
  bool can_handle_http = 2;
  int32 max_streams = 3;
}
```

### 9.1.3 心跳机制

```protobuf
message HeartbeatRequest {
  string node_id = 1;
  NodeStats stats = 2;
}

message HeartbeatResponse {
  int64 timestamp = 1;
}

message NodeStats {
  int32 active_streams = 1;
  int32 connected_clients = 2;
  double cpu_usage = 3;
  double memory_usage = 4;
}
```

### 9.1.4 节点信息查询

```protobuf
message GetNodesRequest {
  bool only_alive = 1;
}

message GetNodesResponse {
  repeated NodeInfo nodes = 1;
}

message NodeInfo {
  string node_id = 1;
  string address = 2;
  NodeCapabilities capabilities = 3;
  NodeStats stats = 4;
  int64 last_heartbeat = 5;
}
```

### 9.1.5 集群事件

```protobuf
// 集群事件
message ClusterEvent {
  string event_id = 1;
  string node_id = 2;
  int64 timestamp = 3;

  oneof payload {
    RoomEvent room = 10;
    PlaybackEvent playback = 11;
    MemberEvent member = 12;
    ChatEvent chat = 13;
    LiveEvent live = 14;
  }
}

message RoomEvent {
  enum Type {
    CREATED = 0;
    UPDATED = 1;
    DELETED = 2;
  }
  Type type = 1;
  string room_id = 2;  // nanoid(12)
  bytes data = 3;  // JSON encoded
}

message PlaybackEvent {
  enum Type {
    TOGGLE = 0;
    SEEK = 1;
    RATE_CHANGED = 2;
    MOVIE_CHANGED = 3;
  }
  Type type = 1;
  string room_id = 2;  // nanoid(12)
  bytes data = 3;
}

message MemberEvent {
  enum Type {
    JOINED = 0;
    LEFT = 1;
    PERMISSIONS_CHANGED = 2;
  }
  Type type = 1;
  string room_id = 2;  // nanoid(12)
  string user_id = 3;  // nanoid(12)
  bytes data = 4;
}

message ChatEvent {
  string room_id = 1;  // nanoid(12)
  string message_id = 2;  // nanoid(12)
  string user_id = 3;  // nanoid(12)
  string content = 4;
}

message LiveEvent {
  enum Type {
    STARTED = 0;
    STOPPED = 1;
  }
  Type type = 1;
  string stream_key = 2;
  bytes data = 3;
}
```

## 9.3 StreamService（流转发服务）

### 9.3.1 服务定义

```protobuf
syntax = "proto3";

package synctv.stream;

// 流转发服务
service StreamRelayService {
  // 拉取RTMP流 (拉流节点收到FLV请求时调用)
  rpc PullRtmpStream(PullRtmpRequest) returns (stream RtmpPacket);

  // 获取流状态
  rpc GetStreamStatus(GetStreamStatusRequest) returns (GetStreamStatusResponse);

  // 停止流
  rpc StopStream(StopStreamRequest) returns (StopStreamResponse);
}
```

### 9.2.2 RTMP流转发

```protobuf
message PullRtmpRequest {
  string stream_key = 1;
  string requesting_node_id = 2;
}

message RtmpPacket {
  enum PacketType {
    VIDEO = 0;
    AUDIO = 1;
    METADATA = 2;
  }
  PacketType type = 1;
  int64 timestamp = 2;
  bytes data = 3;
}
```

### 9.2.3 流状态查询

```protobuf
message GetStreamStatusRequest {
  string stream_key = 1;
}

message GetStreamStatusResponse {
  bool is_live = 1;
  string primary_node_id = 2;
  StreamStats stats = 3;
  int64 start_time = 4;
}

message StreamStats {
  int64 bitrate = 1;
  int32 viewer_count = 2;
  int64 total_bytes = 3;
}
```

### 9.2.4 停止流

```protobuf
message StopStreamRequest {
  string stream_key = 1;
  string requesting_node_id = 2;
}

message StopStreamResponse {
  bool success = 1;
}
```

## 9.4 DataSyncService（数据同步服务）

### 9.4.1 服务定义

```protobuf
syntax = "proto3";

package synctv.data;

// 数据同步服务
service DataSyncService {
  // 同步房间状态
  rpc SyncRoomState(SyncRoomStateRequest) returns (SyncRoomStateResponse);

  // 批量查询房间
  rpc BatchGetRooms(BatchGetRoomsRequest) returns (BatchGetRoomsResponse);

  // 查询用户在线状态
  rpc GetUserOnlineStatus(GetUserOnlineStatusRequest) returns (GetUserOnlineStatusResponse);
}
```

### 9.4.2 房间状态同步

```protobuf
message SyncRoomStateRequest {
  string room_id = 1;  // nanoid(12)
  RoomState state = 2;
  string source_node_id = 3;
}

message RoomState {
  string movie_id = 1;  // nanoid(12)
  string status = 2;  // playing|paused
  double current_time = 3;
  double rate = 4;
  int64 updated_at = 5;  // Unix timestamp
}

message SyncRoomStateResponse {
  bool success = 1;
}
```

### 9.4.3 批量查询房间

```protobuf
message BatchGetRoomsRequest {
  repeated string room_ids = 1;  // nanoid(12)
}

message BatchGetRoomsResponse {
  repeated RoomInfo rooms = 1;
}

message RoomInfo {
  string id = 1;  // nanoid(12)
  string name = 2;
  int32 member_count = 3;
  bool is_live = 4;
  bytes data = 5;  // JSON encoded
}
```

### 9.4.4 用户在线状态

```protobuf
message GetUserOnlineStatusRequest {
  repeated string user_ids = 1;  // nanoid(12)
}

message GetUserOnlineStatusResponse {
  map<string, OnlineStatus> statuses = 1;  // key: nanoid(12)
}

message OnlineStatus {
  bool online = 1;
  string room_id = 2;  // nanoid(12), empty = not in any room
  string node_id = 3;
}
```

## 9.5 使用场景示例

### 9.5.1 节点注册流程

```rust
// 新节点启动时注册到集群
pub async fn register_to_cluster(
    node_id: String,
    address: String,
    grpc_client: &mut ClusterServiceClient<Channel>,
) -> Result<Vec<NodeInfo>> {
    let request = RegisterNodeRequest {
        node_id: node_id.clone(),
        address: address.clone(),
        capabilities: Some(NodeCapabilities {
            can_handle_rtmp: true,
            can_handle_http: true,
            max_streams: 100,
        }),
    };

    let response = grpc_client.register_node(request).await?;
    let nodes = response.into_inner().nodes;

    info!(
        "Registered to cluster: {} nodes available",
        nodes.len()
    );

    Ok(nodes)
}
```

### 9.5.2 心跳发送

```rust
// 定期发送心跳
pub async fn send_heartbeat(
    node_id: String,
    stats: NodeStats,
    grpc_client: &mut ClusterServiceClient<Channel>,
) -> Result<()> {
    let request = HeartbeatRequest {
        node_id,
        stats: Some(stats),
    };

    let response = grpc_client.heartbeat(request).await?;
    let timestamp = response.into_inner().timestamp;

    trace!("Heartbeat sent, server timestamp: {}", timestamp);

    Ok(())
}
```

### 9.5.3 事件流订阅

```rust
// 订阅集群事件流
pub async fn subscribe_cluster_events(
    node_id: String,
    mut grpc_client: ClusterServiceClient<Channel>,
    event_handler: Arc<dyn EventHandler>,
) -> Result<()> {
    let (tx, rx) = mpsc::channel(100);

    // 发送本地事件到流
    tokio::spawn(async move {
        // 本地事件发布逻辑
    });

    // 接收远程事件
    let mut stream = grpc_client.sync_events(rx).await?.into_inner();

    while let Some(event) = stream.message().await? {
        if event.node_id != node_id {
            // 处理来自其他节点的事件
            event_handler.handle(event).await?;
        }
    }

    Ok(())
}
```

### 9.5.4 拉流实现

```rust
// 从主节点拉取RTMP流
pub async fn pull_rtmp_stream(
    stream_key: String,
    node_id: String,
    primary_node_addr: String,
) -> Result<()> {
    let mut client = StreamRelayServiceClient::connect(primary_node_addr).await?;

    let request = PullRtmpRequest {
        stream_key: stream_key.clone(),
        requesting_node_id: node_id,
    };

    let mut stream = client.pull_rtmp_stream(request).await?.into_inner();

    info!("Started pulling RTMP stream: {}", stream_key);

    while let Some(packet) = stream.message().await? {
        // 处理RTMP包
        match packet.packet_type() {
            PacketType::Video => {
                // 处理视频数据
            }
            PacketType::Audio => {
                // 处理音频数据
            }
            PacketType::Metadata => {
                // 处理元数据
            }
        }
    }

    Ok(())
}
```

### 9.5.5 房间状态同步

```rust
// 同步房间状态到其他节点
pub async fn sync_room_state(
    room_id: i64,
    state: RoomState,
    node_id: String,
    peer_addr: String,
) -> Result<()> {
    let mut client = DataSyncServiceClient::connect(peer_addr).await?;

    let request = SyncRoomStateRequest {
        room_id,
        state: Some(state),
        source_node_id: node_id,
    };

    let response = client.sync_room_state(request).await?;

    if response.into_inner().success {
        trace!("Room state synced for room {}", room_id);
    }

    Ok(())
}
```

## 9.6 错误处理

### 9.5.1 gRPC错误码映射

```rust
pub fn map_grpc_error(status: tonic::Status) -> Error {
    match status.code() {
        Code::NotFound => Error::ResourceNotFound,
        Code::PermissionDenied => Error::PermissionDenied,
        Code::Unauthenticated => Error::Unauthenticated,
        Code::Unavailable => Error::ServiceUnavailable,
        Code::DeadlineExceeded => Error::Timeout,
        _ => Error::Internal(status.message().to_string()),
    }
}
```

### 9.5.2 重试策略

```rust
pub async fn with_retry<F, Fut, T>(
    mut operation: F,
    max_retries: usize,
) -> Result<T>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T>>,
{
    let mut attempts = 0;
    let mut delay = Duration::from_millis(100);

    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempts >= max_retries => return Err(e),
            Err(e) => {
                warn!(
                    "Operation failed (attempt {}/{}): {:?}",
                    attempts + 1,
                    max_retries,
                    e
                );
                tokio::time::sleep(delay).await;
                delay = (delay * 2).min(Duration::from_secs(10));
                attempts += 1;
            }
        }
    }
}
```

## 9.7 性能优化

### 9.7.1 连接池

```rust
// gRPC连接池
pub struct GrpcConnectionPool {
    pools: Arc<DashMap<String, Channel>>,
}

impl GrpcConnectionPool {
    pub async fn get_or_connect(&self, addr: &str) -> Result<Channel> {
        if let Some(channel) = self.pools.get(addr) {
            return Ok(channel.clone());
        }

        let channel = Channel::from_shared(addr.to_string())?
            .connect_timeout(Duration::from_secs(5))
            .timeout(Duration::from_secs(30))
            .connect()
            .await?;

        self.pools.insert(addr.to_string(), channel.clone());

        Ok(channel)
    }
}
```

### 9.7.2 压缩配置

```rust
// 启用gRPC压缩
pub fn create_grpc_client(addr: String) -> Result<ClusterServiceClient<Channel>> {
    let channel = Channel::from_shared(addr)?
        .connect_timeout(Duration::from_secs(5))
        .timeout(Duration::from_secs(30))
        .connect_lazy();

    let client = ClusterServiceClient::new(channel)
        .send_compressed(CompressionEncoding::Gzip)
        .accept_compressed(CompressionEncoding::Gzip);

    Ok(client)
}
```

### 9.7.3 流式传输优化

```rust
// 批量发送事件
pub async fn batch_send_events(
    events: Vec<ClusterEvent>,
    mut client: ClusterServiceClient<Channel>,
) -> Result<()> {
    let (tx, rx) = mpsc::channel(100);

    // 在后台任务中发送事件
    tokio::spawn(async move {
        for event in events {
            if tx.send(event).await.is_err() {
                break;
            }
        }
    });

    let mut stream = client.sync_events(rx).await?.into_inner();

    // 接收响应
    while let Some(response) = stream.message().await? {
        trace!("Received event response: {:?}", response);
    }

    Ok(())
}
```

---

**上一章**: [14-通信架构设计](./14-通信架构设计.md)
**下一章**: [16-实时消息流](./16-实时消息流.md)
