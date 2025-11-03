# 分布式同步观影软件开发方案

## 项目概述

开发一款支持多人同步观影和聊天的跨平台应用，采用C++实现，支持分布式架构，无需中心化服务器。

## 技术架构方案

### 1. 跨平台开发框架选择

#### 推荐方案：Qt + QML

- **优势**：
    - 原生C++支持，性能优秀
    - 完整的跨平台支持（iOS/Android/Windows/macOS/iPad）
    - 内置网络库（QNetworkAccessManager）
    - 强大的多媒体支持（Qt Multimedia）
    - 成熟的P2P网络组件

#### 备选方案：Flutter with C++ Plugin

- **优势**：现代UI框架，开发效率高
- **劣势**：需要额外的C++插件开发工作

### 2. 分布式网络架构

#### 2.1 P2P网络拓扑

```
房主设备 ←→ 参与者A
    ↕        ↗  ↘
参与者B ←→ 参与者C
```

#### 2.2 核心网络组件

- **发现机制**：
    
    - 局域网：mDNS/Bonjour自动发现
    - 互联网：STUN/TURN服务器辅助NAT穿透
    - 邀请码：房主生成短码，其他人输入加入
- **通信协议**：
    
    - WebRTC：音视频传输和P2P连接
    - 自定义协议：同步控制命令（播放、暂停、进度等）

#### 2.3 推荐的第三方库

- **libp2p**：成熟的P2P网络库
- **WebRTC C++ SDK**：音视频传输
- **OpenDHT**：分布式哈希表，用于节点发现

### 3. 视频同步策略

#### 3.1 离线视频方案

##### 方案A：实时推流同步

```cpp
// 伪代码示例
class VideoStreamer {
    void streamToAllPeers(VideoFrame frame, int64_t timestamp) {
        for (auto& peer : connectedPeers) {
            peer.sendFrame(frame, timestamp);
        }
    }
};
```

##### 方案B：预缓存同步（推荐）

```cpp
class PreCacheSystem {
    // 房主预先推流
    void preStreamVideo(std::string videoPath, int cacheAheadSeconds) {
        // 将视频分片推送给所有设备
        auto chunks = segmentVideo(videoPath, chunkDuration);
        for (auto& chunk : chunks) {
            broadcastChunk(chunk);
        }
    }
    
    // 同步播放控制
    void syncPlay(int64_t globalTimestamp) {
        broadcastCommand({PLAY, globalTimestamp});
    }
};
```

#### 3.2 在线视频方案

```cpp
class OnlineVideoSync {
    struct SyncCommand {
        enum Type { PLAY, PAUSE, SEEK };
        Type command;
        std::string videoUrl;
        int64_t timestamp;
        int64_t globalSyncTime;
    };
    
    void syncOnlineVideo(std::string url, int64_t startTime) {
        // 所有设备同时加载相同URL
        SyncCommand cmd = {PLAY, url, startTime, getCurrentGlobalTime()};
        broadcastCommand(cmd);
    }
};
```

### 4. 时间同步机制

#### 4.1 NTP时间同步

```cpp
class TimeSync {
private:
    int64_t timeOffset = 0;  // 与房主的时间差
    
public:
    int64_t getSyncedTime() {
        return std::chrono::duration_cast<std::chrono::milliseconds>
               (std::chrono::system_clock::now().time_since_epoch()).count() 
               + timeOffset;
    }
    
    void calibrateTime(int64_t hostTime, int64_t networkDelay) {
        int64_t localTime = getCurrentTime();
        timeOffset = hostTime - localTime + networkDelay/2;
    }
};
```

#### 4.2 播放同步算法

```cpp
class PlaybackSync {
    void adjustPlayback(int64_t targetTimestamp) {
        int64_t currentPos = player.getCurrentPosition();
        int64_t syncedTime = timeSync.getSyncedTime();
        int64_t targetPos = targetTimestamp - (syncedTime - startTime);
        
        int64_t drift = currentPos - targetPos;
        if (abs(drift) > SYNC_THRESHOLD) {
            if (drift > 0) {
                player.setPlaybackSpeed(0.98);  // 稍微减速
            } else {
                player.setPlaybackSpeed(1.02);  // 稍微加速
            }
        } else {
            player.setPlaybackSpeed(1.0);
        }
    }
};
```

### 5. 数据结构设计

#### 5.1 核心数据模型

```cpp
struct Room {
    std::string roomId;
    std::string hostPeerId;
    std::vector<Peer> participants;
    VideoSession currentVideo;
    ChatSession chat;
};

struct VideoSession {
    std::string videoId;
    std::string source;  // "local" or URL
    int64_t startTime;   // 全局开始时间
    PlaybackState state; // PLAYING, PAUSED, BUFFERING
    int64_t currentPosition;
};

struct Peer {
    std::string peerId;
    std::string displayName;
    NetworkAddress address;
    ConnectionState state;
    int64_t lastHeartbeat;
};
```

#### 5.2 消息协议

```cpp
enum MessageType {
    // 房间管理
    JOIN_ROOM,
    LEAVE_ROOM,
    ROOM_INFO,
    
    // 视频控制
    VIDEO_PLAY,
    VIDEO_PAUSE,
    VIDEO_SEEK,
    VIDEO_CHUNK,
    SYNC_REQUEST,
    
    // 聊天
    CHAT_MESSAGE,
    
    // 系统
    HEARTBEAT,
    TIME_SYNC
};

struct NetworkMessage {
    MessageType type;
    std::string senderId;
    int64_t timestamp;
    std::vector<uint8_t> payload;
};
```

### 6. 关键技术挑战与解决方案

#### 6.1 NAT穿透

- **问题**：不同网络环境下的设备连接
- **解决方案**：
    - STUN服务器获取公网IP
    - TURN服务器作为中继（备选）
    - UPnP自动端口映射
    - ICE候选收集和连接建立

#### 6.2 网络抖动处理

```cpp
class JitterBuffer {
private:
    std::queue<TimestampedFrame> buffer;
    const int TARGET_BUFFER_SIZE = 3;  // 3秒缓冲
    
public:
    void addFrame(TimestampedFrame frame) {
        buffer.push(frame);
        while (buffer.size() > TARGET_BUFFER_SIZE * FPS) {
            buffer.pop();
        }
    }
    
    TimestampedFrame getFrameForTime(int64_t timestamp) {
        // 从缓冲区获取最接近目标时间的帧
    }
};
```

#### 6.3 设备性能差异适配

```cpp
class AdaptiveQuality {
    struct QualityLevel {
        int width, height;
        int bitrate;
        int framerate;
    };
    
    void adjustQuality(NetworkStats stats, DeviceCapability device) {
        if (stats.bandwidth < LOW_BANDWIDTH_THRESHOLD || 
            device.processingPower < LOW_POWER_THRESHOLD) {
            switchToLowerQuality();
        }
    }
};
```

### 7. 实现路线图

#### 阶段1：基础框架（4-6周）

1. 搭建Qt跨平台项目结构
2. 实现基本的P2P网络连接
3. 设计核心数据结构和消息协议
4. 简单的房间创建和加入功能

#### 阶段2：视频功能（6-8周）

1. 集成视频播放器（Qt Multimedia或FFmpeg）
2. 实现时间同步机制
3. 开发视频推流和接收功能
4. 基础的播放控制同步

#### 阶段3：高级功能（4-6周）

1. 预缓存系统实现
2. 在线视频支持
3. 聊天功能
4. UI/UX优化

#### 阶段4：优化与发布（4-6周）

1. 性能优化和内存管理
2. 网络异常处理
3. 各平台适配和测试
4. 用户体验优化

### 8. 技术建议

#### 8.1 视频处理

- **编解码**：使用硬件加速（VideoToolbox、MediaCodec、NVENC）
- **格式支持**：H.264/H.265为主，考虑AV1未来支持
- **分辨率适配**：支持多档位质量切换

#### 8.2 网络优化

- **带宽检测**：动态调整视频质量
- **断线重连**：自动重连机制
- **多路径传输**：条件允许时使用多个网络接口

#### 8.3 用户体验

- **离线模式**：允许部分功能离线使用
- **权限管理**：房主控制、投票控制等多种模式
- **表情互动**：实时表情和弹幕功能

### 9. 潜在风险与应对

#### 9.1 技术风险

- **同步精度**：网络延迟导致的不同步
    - 解决：自适应缓冲和播放速度微调
- **设备兼容性**：不同设备的解码能力差异
    - 解决：多格式支持和质量自适应

#### 9.2 法律风险

- **版权问题**：确保用户理解版权责任
- **内容审核**：聊天内容的合规性

### 10. 开源组件推荐

- **网络层**：libp2p-cpp, WebRTC
- **音视频**：FFmpeg, Qt Multimedia
- **UI框架**：Qt Quick/QML
- **加密**：OpenSSL, libsodium
- **序列化**：Protocol Buffers, MessagePack

## 总结

这个项目具有很好的技术可行性，关键在于：

1. 选择合适的P2P网络框架
2. 设计健壮的时间同步机制
3. 实现高效的视频缓存和传输策略
4. 处理各种网络环境和设备差异

建议先实现MVP版本验证核心功能，再逐步添加高级特性。重点关注用户体验的流畅性和网络适应能力。