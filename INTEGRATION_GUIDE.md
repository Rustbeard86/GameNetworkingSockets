# GameNetworkingSockets Integration Guide

## Table of Contents
1. [Quick Start](#quick-start)
2. [Integration Approaches](#integration-approaches)
3. [Basic Client-Server Setup](#basic-client-server-setup)
4. [P2P Setup](#p2p-setup)
5. [Message Handling Patterns](#message-handling-patterns)
6. [Game Loop Integration](#game-loop-integration)
7. [Error Handling](#error-handling)
8. [Best Practices](#best-practices)
9. [Common Patterns](#common-patterns)

## Quick Start

### Prerequisites

Before integrating GameNetworkingSockets into your game:

1. **Choose your build method**:
   - Use vcpkg (recommended for easy dependency management)
   - Build from source (more control, see [BUILDING.md](BUILDING.md))
   - Use pre-built binaries (if available for your platform)

2. **Understand your networking needs**:
   - Client-server or peer-to-peer?
   - Reliable messages, unreliable, or both?
   - Do you need cross-platform support?

3. **Review dependencies**:
   - C++11 or later compiler
   - OpenSSL or libsodium for crypto
   - Google protobuf
   - WebRTC (only if using P2P with ICE)

### 5-Minute Example

Here's a minimal example to get you started:

**Server:**
```cpp
#include <steam/steamnetworkingsockets.h>
#include <steam/isteamnetworkingutils.h>

// Initialize library
SteamDatagramErrMsg errMsg;
if (!GameNetworkingSockets_Init(nullptr, errMsg)) {
    printf("Failed to initialize: %s\n", errMsg);
    return 1;
}

// Create server
ISteamNetworkingSockets *pInterface = SteamNetworkingSockets();
SteamNetworkingIPAddr serverAddr;
serverAddr.Clear();
serverAddr.m_port = 27015;

HSteamListenSocket hListenSocket = pInterface->CreateListenSocketIP(serverAddr, 0, nullptr);

// Game loop
while (running) {
    pInterface->RunCallbacks();
    // Handle connections and messages
}

// Cleanup
pInterface->CloseListenSocket(hListenSocket);
GameNetworkingSockets_Kill();
```

**Client:**
```cpp
#include <steam/steamnetworkingsockets.h>

// Initialize library
SteamDatagramErrMsg errMsg;
if (!GameNetworkingSockets_Init(nullptr, errMsg)) {
    printf("Failed to initialize: %s\n", errMsg);
    return 1;
}

// Connect to server
ISteamNetworkingSockets *pInterface = SteamNetworkingSockets();
SteamNetworkingIPAddr serverAddr;
serverAddr.ParseString("127.0.0.1:27015");

HSteamNetConnection hConnection = pInterface->ConnectByIPAddress(serverAddr, 0, nullptr);

// Game loop
while (running) {
    pInterface->RunCallbacks();
    
    // Send message
    const char *msg = "Hello, server!";
    pInterface->SendMessageToConnection(
        hConnection, 
        msg, 
        strlen(msg), 
        k_nSteamNetworkingSend_Reliable, 
        nullptr
    );
    
    // Receive messages
    SteamNetworkingMessage_t *pMessages[32];
    int numMessages = pInterface->ReceiveMessagesOnConnection(hConnection, pMessages, 32);
    for (int i = 0; i < numMessages; i++) {
        // Process message
        printf("Received: %.*s\n", pMessages[i]->m_cbSize, (char*)pMessages[i]->m_pData);
        pMessages[i]->Release();
    }
}

// Cleanup
pInterface->CloseConnection(hConnection, 0, nullptr, false);
GameNetworkingSockets_Kill();
```

## Integration Approaches

### Approach 1: Direct API Usage (ISteamNetworkingSockets)

**Best for:**
- Games being built from scratch
- Games already using connection-oriented networking
- Maximum control and features

**Characteristics:**
- Explicit connection management
- Full control over message sending/receiving
- Access to all features (lanes, priorities, etc.)
- More code but more flexibility

### Approach 2: Ad-Hoc Messaging (ISteamNetworkingMessages)

**Best for:**
- Porting UDP-based games
- Simple P2P scenarios
- Quick prototyping

**Characteristics:**
- Automatic connection management
- Send messages by identity (not connection handle)
- Simpler API, less code
- Limited feature access compared to direct API

**Example:**
```cpp
ISteamNetworkingMessages *pMessages = SteamNetworkingMessages();

// Send to peer by identity
SteamNetworkingIdentity recipient;
recipient.ParseString("peer:12345");

pMessages->SendMessageToUser(
    recipient,
    myData, 
    dataSize,
    k_nSteamNetworkingSend_Reliable,
    0  // Lane 0
);

// Receive from any peer
SteamNetworkingMessage_t *pIncomingMsgs[32];
int numMsgs = pMessages->ReceiveMessagesOnChannel(0, pIncomingMsgs, 32);
```

### Approach 3: vcpkg Integration

**Best for:**
- Projects already using vcpkg
- Want easy dependency management
- Standard configuration is acceptable

**Steps:**
1. Add to `vcpkg.json`:
```json
{
  "dependencies": ["gamenetworkingsockets"]
}
```

2. Configure CMake with vcpkg toolchain
3. Link against the library

See [examples/vcpkg_example_chat](examples/vcpkg_example_chat/README.md) for complete example.

## Basic Client-Server Setup

### Step 1: Initialize the Library

**Do this once at startup:**

```cpp
#include <steam/steamnetworkingsockets.h>
#include <steam/isteamnetworkingutils.h>

bool InitializeNetworking() {
    #ifdef STEAMNETWORKINGSOCKETS_OPENSOURCE
        // Open source version
        SteamDatagramErrMsg errMsg;
        if (!GameNetworkingSockets_Init(nullptr, errMsg)) {
            fprintf(stderr, "GameNetworkingSockets_Init failed: %s\n", errMsg);
            return false;
        }
    #else
        // Steamworks SDK version
        SteamDatagram_SetAppID(YOUR_APP_ID);
        SteamDatagram_SetUniverse(k_EUniversePublic);
        
        SteamDatagramErrMsg errMsg;
        if (!SteamDatagramClient_Init(errMsg)) {
            fprintf(stderr, "SteamDatagramClient_Init failed: %s\n", errMsg);
            return false;
        }
    #endif
    
    // Enable debug output (optional)
    SteamNetworkingUtils()->SetDebugOutputFunction(
        k_ESteamNetworkingSocketsDebugOutputType_Msg,
        DebugOutput
    );
    
    return true;
}

void DebugOutput(ESteamNetworkingSocketsDebugOutputType eType, const char *pszMsg) {
    printf("%s\n", pszMsg);
}
```

### Step 2: Create Server

```cpp
class GameServer {
private:
    ISteamNetworkingSockets *m_pInterface;
    HSteamListenSocket m_hListenSocket;
    std::map<HSteamNetConnection, ClientInfo> m_clients;
    
public:
    bool Start(uint16_t port) {
        m_pInterface = SteamNetworkingSockets();
        
        SteamNetworkingIPAddr serverAddr;
        serverAddr.Clear();
        serverAddr.m_port = port;
        
        // Create listen socket with optional config
        SteamNetworkingConfigValue_t opts[1];
        opts[0].SetPtr(
            k_ESteamNetworkingConfig_Callback_ConnectionStatusChanged,
            (void*)OnConnectionStatusChanged
        );
        
        m_hListenSocket = m_pInterface->CreateListenSocketIP(
            serverAddr, 
            1,      // Number of config options
            opts
        );
        
        if (m_hListenSocket == k_HSteamListenSocket_Invalid) {
            fprintf(stderr, "Failed to create listen socket\n");
            return false;
        }
        
        printf("Server listening on port %d\n", port);
        return true;
    }
    
    void Update() {
        // Process callbacks
        m_pInterface->RunCallbacks();
        
        // Receive messages from all clients
        for (auto &pair : m_clients) {
            HSteamNetConnection hConn = pair.first;
            
            SteamNetworkingMessage_t *pMessages[32];
            int numMessages = m_pInterface->ReceiveMessagesOnConnection(
                hConn, 
                pMessages, 
                32
            );
            
            for (int i = 0; i < numMessages; i++) {
                HandleClientMessage(hConn, pMessages[i]);
                pMessages[i]->Release();
            }
        }
    }
    
    void SendToClient(HSteamNetConnection hConn, const void *data, size_t size, bool reliable) {
        int sendFlags = reliable ? 
            k_nSteamNetworkingSend_Reliable : 
            k_nSteamNetworkingSend_Unreliable;
            
        m_pInterface->SendMessageToConnection(
            hConn,
            data,
            size,
            sendFlags,
            nullptr
        );
    }
    
    void BroadcastToAllClients(const void *data, size_t size, bool reliable) {
        for (auto &pair : m_clients) {
            SendToClient(pair.first, data, size, reliable);
        }
    }
    
private:
    static void OnConnectionStatusChanged(
        SteamNetConnectionStatusChangedCallback_t *pInfo
    ) {
        // Handle connection state changes
        // (Implementation shown in callback section below)
    }
    
    void HandleClientMessage(HSteamNetConnection hConn, SteamNetworkingMessage_t *pMsg) {
        // Process game message from client
        printf("Received %d bytes from client\n", pMsg->m_cbSize);
    }
};
```

### Step 3: Create Client

```cpp
class GameClient {
private:
    ISteamNetworkingSockets *m_pInterface;
    HSteamNetConnection m_hConnection;
    bool m_bConnected;
    
public:
    bool Connect(const char *serverAddress) {
        m_pInterface = SteamNetworkingSockets();
        m_bConnected = false;
        
        SteamNetworkingIPAddr serverAddr;
        if (!serverAddr.ParseString(serverAddress)) {
            fprintf(stderr, "Invalid server address: %s\n", serverAddress);
            return false;
        }
        
        // Set connection callback
        SteamNetworkingConfigValue_t opts[1];
        opts[0].SetPtr(
            k_ESteamNetworkingConfig_Callback_ConnectionStatusChanged,
            (void*)OnConnectionStatusChanged
        );
        
        m_hConnection = m_pInterface->ConnectByIPAddress(serverAddr, 1, opts);
        
        if (m_hConnection == k_HSteamNetConnection_Invalid) {
            fprintf(stderr, "Failed to create connection\n");
            return false;
        }
        
        printf("Connecting to %s...\n", serverAddress);
        return true;
    }
    
    void Update() {
        m_pInterface->RunCallbacks();
        
        if (!m_bConnected) {
            return;  // Wait for connection
        }
        
        // Receive messages
        SteamNetworkingMessage_t *pMessages[32];
        int numMessages = m_pInterface->ReceiveMessagesOnConnection(
            m_hConnection, 
            pMessages, 
            32
        );
        
        for (int i = 0; i < numMessages; i++) {
            HandleServerMessage(pMessages[i]);
            pMessages[i]->Release();
        }
    }
    
    void SendToServer(const void *data, size_t size, bool reliable) {
        if (!m_bConnected) {
            return;
        }
        
        int sendFlags = reliable ? 
            k_nSteamNetworkingSend_Reliable : 
            k_nSteamNetworkingSend_Unreliable;
            
        m_pInterface->SendMessageToConnection(
            m_hConnection,
            data,
            size,
            sendFlags,
            nullptr
        );
    }
    
    void Disconnect() {
        if (m_hConnection != k_HSteamNetConnection_Invalid) {
            m_pInterface->CloseConnection(m_hConnection, 0, "Client disconnect", false);
            m_hConnection = k_HSteamNetConnection_Invalid;
        }
        m_bConnected = false;
    }
    
private:
    static void OnConnectionStatusChanged(
        SteamNetConnectionStatusChangedCallback_t *pInfo
    ) {
        GameClient *pClient = /* get instance pointer */;
        
        if (pInfo->m_info.m_eState == k_ESteamNetworkingConnectionState_Connected) {
            pClient->m_bConnected = true;
            printf("Connected to server!\n");
        }
        else if (pInfo->m_info.m_eState == k_ESteamNetworkingConnectionState_ClosedByPeer ||
                 pInfo->m_info.m_eState == k_ESteamNetworkingConnectionState_ProblemDetectedLocally) {
            printf("Connection lost: %s\n", pInfo->m_info.m_szEndDebug);
            pClient->m_bConnected = false;
        }
    }
    
    void HandleServerMessage(SteamNetworkingMessage_t *pMsg) {
        // Process game message from server
        printf("Received %d bytes from server\n", pMsg->m_cbSize);
    }
};
```

### Step 4: Handle Connection State Changes

Implement callback to handle connection lifecycle:

```cpp
void OnConnectionStatusChanged(SteamNetConnectionStatusChangedCallback_t *pInfo) {
    switch (pInfo->m_info.m_eState) {
        case k_ESteamNetworkingConnectionState_None:
            // Not connected, connecting, connected, or disconnecting
            break;
            
        case k_ESteamNetworkingConnectionState_Connecting:
            // In the process of connecting
            printf("Connection %u: Connecting...\n", pInfo->m_hConn);
            break;
            
        case k_ESteamNetworkingConnectionState_FindingRoute:
            // P2P: Finding route to peer
            printf("Connection %u: Finding route...\n", pInfo->m_hConn);
            break;
            
        case k_ESteamNetworkingConnectionState_Connected:
            // Successfully connected
            printf("Connection %u: Connected!\n", pInfo->m_hConn);
            
            // On server: Accept the connection
            if (pInfo->m_eOldState == k_ESteamNetworkingConnectionState_Connecting) {
                if (SteamNetworkingSockets()->AcceptConnection(pInfo->m_hConn) != k_EResultOK) {
                    SteamNetworkingSockets()->CloseConnection(
                        pInfo->m_hConn, 
                        0, 
                        "Failed to accept", 
                        false
                    );
                    break;
                }
            }
            break;
            
        case k_ESteamNetworkingConnectionState_ClosedByPeer:
        case k_ESteamNetworkingConnectionState_ProblemDetectedLocally:
            // Connection closed or failed
            printf("Connection %u: Closed (%s)\n", 
                   pInfo->m_hConn, 
                   pInfo->m_info.m_szEndDebug);
            
            // Clean up connection
            SteamNetworkingSockets()->CloseConnection(pInfo->m_hConn, 0, nullptr, false);
            break;
            
        case k_ESteamNetworkingConnectionState_Dead:
            // Connection is fully closed and can be forgotten
            break;
    }
}
```

## P2P Setup

For peer-to-peer connections, you need additional infrastructure. See [README_P2P.md](README_P2P.md) for full details.

### Requirements

1. **Signaling Service**: To exchange connection information
2. **STUN Servers**: To discover public IPs
3. **TURN Servers** (optional): For relay fallback

### Basic P2P Example

```cpp
#include <steam/steamnetworkingcustomsignaling.h>

class MySignalingService : public ISteamNetworkingConnectionSignaling {
    // Implement signaling protocol to exchange ICE candidates
    // See examples/trivial_signaling_client.cpp for reference
};

// Initialize with ICE support
bool InitP2P() {
    // Must compile with USE_STEAMWEBRTC=ON
    #ifndef STEAMNETWORKINGSOCKETS_ENABLE_ICE
        fprintf(stderr, "ICE support not compiled in!\n");
        return false;
    #endif
    
    // Initialize library
    SteamDatagramErrMsg errMsg;
    if (!GameNetworkingSockets_Init(nullptr, errMsg)) {
        return false;
    }
    
    // Configure STUN servers
    SteamNetworkingConfigValue_t opts[1];
    opts[0].SetString(
        k_ESteamNetworkingConfig_P2P_STUN_ServerList,
        "stun:stun.l.google.com:19302"
    );
    
    SteamNetworkingUtils()->SetConfigValueStruct(
        opts[0],
        k_ESteamNetworkingConfig_Global,
        0
    );
    
    return true;
}

// Connect using P2P with custom signaling
HSteamNetConnection ConnectP2P(SteamNetworkingIdentity &remoteIdentity) {
    ISteamNetworkingSockets *pInterface = SteamNetworkingSockets();
    
    // Enable symmetric connect for P2P
    SteamNetworkingConfigValue_t opts[2];
    opts[0].SetInt32(k_ESteamNetworkingConfig_SymmetricConnect, 1);
    opts[1].SetPtr(
        k_ESteamNetworkingConfig_Callback_ConnectionStatusChanged,
        (void*)OnConnectionStatusChanged
    );
    
    // Create custom signaling instance
    MySignalingService *pSignaling = new MySignalingService();
    
    HSteamNetConnection hConn = pInterface->ConnectP2PCustomSignaling(
        pSignaling,
        &remoteIdentity,
        0,      // Virtual port
        2,      // Number of options
        opts
    );
    
    return hConn;
}
```

## Message Handling Patterns

### Pattern 1: Serialized Game Messages

Use a message header to identify message types:

```cpp
enum MessageType {
    MSG_PLAYER_MOVE = 1,
    MSG_PLAYER_SHOOT = 2,
    MSG_CHAT = 3,
    // ... more types
};

struct MessageHeader {
    uint16_t type;
    uint16_t size;
};

// Sending
void SendPlayerMove(HSteamNetConnection hConn, const Vector3 &position) {
    struct {
        MessageHeader header;
        Vector3 position;
    } msg;
    
    msg.header.type = MSG_PLAYER_MOVE;
    msg.header.size = sizeof(Vector3);
    msg.position = position;
    
    SteamNetworkingSockets()->SendMessageToConnection(
        hConn,
        &msg,
        sizeof(msg),
        k_nSteamNetworkingSend_Unreliable,  // Positions can be unreliable
        nullptr
    );
}

// Receiving
void HandleMessage(SteamNetworkingMessage_t *pMsg) {
    if (pMsg->m_cbSize < sizeof(MessageHeader)) {
        return;  // Invalid message
    }
    
    MessageHeader *header = (MessageHeader*)pMsg->m_pData;
    void *payload = (char*)pMsg->m_pData + sizeof(MessageHeader);
    
    switch (header->type) {
        case MSG_PLAYER_MOVE:
            if (header->size == sizeof(Vector3)) {
                Vector3 *pos = (Vector3*)payload;
                OnPlayerMove(*pos);
            }
            break;
            
        case MSG_PLAYER_SHOOT:
            OnPlayerShoot();
            break;
            
        case MSG_CHAT:
            char chatMsg[256];
            memcpy(chatMsg, payload, std::min(header->size, 255u));
            chatMsg[std::min(header->size, 255u)] = '\0';
            OnChatMessage(chatMsg);
            break;
    }
}
```

### Pattern 2: Protobuf Messages

For complex data structures, use Protocol Buffers:

```protobuf
// game_messages.proto
syntax = "proto3";

message PlayerMove {
    float x = 1;
    float y = 2;
    float z = 3;
    float rotation = 4;
}

message GameStateUpdate {
    int32 game_time = 1;
    repeated PlayerState players = 2;
}
```

```cpp
#include "game_messages.pb.h"

void SendPlayerMove(HSteamNetConnection hConn, const Vector3 &pos, float rotation) {
    PlayerMove msg;
    msg.set_x(pos.x);
    msg.set_y(pos.y);
    msg.set_z(pos.z);
    msg.set_rotation(rotation);
    
    std::string serialized;
    msg.SerializeToString(&serialized);
    
    SteamNetworkingSockets()->SendMessageToConnection(
        hConn,
        serialized.data(),
        serialized.size(),
        k_nSteamNetworkingSend_Unreliable,
        nullptr
    );
}

void HandlePlayerMoveMessage(SteamNetworkingMessage_t *pMsg) {
    PlayerMove msg;
    if (msg.ParseFromArray(pMsg->m_pData, pMsg->m_cbSize)) {
        Vector3 pos(msg.x(), msg.y(), msg.z());
        float rotation = msg.rotation();
        OnPlayerMove(pos, rotation);
    }
}
```

### Pattern 3: Batching Small Messages

Reduce overhead by batching:

```cpp
class MessageBatcher {
private:
    std::vector<uint8_t> m_buffer;
    size_t m_maxSize;
    
public:
    MessageBatcher(size_t maxSize = 1200) : m_maxSize(maxSize) {
        m_buffer.reserve(maxSize);
    }
    
    bool AddMessage(uint16_t type, const void *data, size_t size) {
        if (m_buffer.size() + sizeof(MessageHeader) + size > m_maxSize) {
            return false;  // Buffer full, need to flush
        }
        
        MessageHeader header = {type, (uint16_t)size};
        size_t offset = m_buffer.size();
        m_buffer.resize(offset + sizeof(header) + size);
        
        memcpy(&m_buffer[offset], &header, sizeof(header));
        memcpy(&m_buffer[offset + sizeof(header)], data, size);
        
        return true;
    }
    
    void Flush(HSteamNetConnection hConn, int sendFlags) {
        if (m_buffer.empty()) {
            return;
        }
        
        SteamNetworkingSockets()->SendMessageToConnection(
            hConn,
            m_buffer.data(),
            m_buffer.size(),
            sendFlags,
            nullptr
        );
        
        m_buffer.clear();
    }
};

// Usage
MessageBatcher batcher;
batcher.AddMessage(MSG_PLAYER_MOVE, &moveData, sizeof(moveData));
batcher.AddMessage(MSG_PLAYER_SHOOT, &shootData, sizeof(shootData));
batcher.Flush(hConn, k_nSteamNetworkingSend_Unreliable);
```

## Game Loop Integration

### Single-Threaded Integration

Most games can use single-threaded integration:

```cpp
void GameLoop() {
    while (running) {
        // 1. Process input
        ProcessInput();
        
        // 2. Update networking (callbacks and receive messages)
        SteamNetworkingSockets()->RunCallbacks();
        ReceiveAndProcessMessages();
        
        // 3. Update game logic
        UpdateGameLogic(deltaTime);
        
        // 4. Send network updates
        SendGameStateUpdates();
        
        // 5. Render
        Render();
        
        // 6. Frame timing
        LimitFrameRate();
    }
}

void ReceiveAndProcessMessages() {
    for (auto hConn : activeConnections) {
        SteamNetworkingMessage_t *pMessages[32];
        int numMessages = SteamNetworkingSockets()->ReceiveMessagesOnConnection(
            hConn, 
            pMessages, 
            32
        );
        
        for (int i = 0; i < numMessages; i++) {
            ProcessGameMessage(pMessages[i]);
            pMessages[i]->Release();
        }
    }
}

void SendGameStateUpdates() {
    // Send player state to server
    PlayerState state = GetLocalPlayerState();
    SendPlayerState(serverConnection, state);
    
    // Or from server to clients
    for (auto client : clients) {
        SendWorldState(client.connection, worldState);
    }
}
```

### Multi-Threaded Integration

For multi-threaded games, use a dedicated network thread:

```cpp
class NetworkManager {
private:
    std::thread m_networkThread;
    std::atomic<bool> m_running;
    std::mutex m_messageQueueMutex;
    std::queue<NetworkMessage> m_incomingMessages;
    std::queue<NetworkMessage> m_outgoingMessages;
    
public:
    void Start() {
        m_running = true;
        m_networkThread = std::thread(&NetworkManager::NetworkThreadFunc, this);
    }
    
    void Stop() {
        m_running = false;
        if (m_networkThread.joinable()) {
            m_networkThread.join();
        }
    }
    
    void NetworkThreadFunc() {
        while (m_running) {
            // Process callbacks
            SteamNetworkingSockets()->RunCallbacks();
            
            // Receive messages
            ReceiveMessages();
            
            // Send queued messages
            SendQueuedMessages();
            
            // Sleep briefly to avoid spinning
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }
    
    void QueueMessage(const NetworkMessage &msg) {
        std::lock_guard<std::mutex> lock(m_messageQueueMutex);
        m_outgoingMessages.push(msg);
    }
    
    bool GetNextMessage(NetworkMessage &msg) {
        std::lock_guard<std::mutex> lock(m_messageQueueMutex);
        if (m_incomingMessages.empty()) {
            return false;
        }
        msg = m_incomingMessages.front();
        m_incomingMessages.pop();
        return true;
    }
    
private:
    void ReceiveMessages() {
        // Receive and queue messages for game thread
        SteamNetworkingMessage_t *pMessages[32];
        int numMessages = SteamNetworkingSockets()->ReceiveMessagesOnConnection(
            connection, 
            pMessages, 
            32
        );
        
        std::lock_guard<std::mutex> lock(m_messageQueueMutex);
        for (int i = 0; i < numMessages; i++) {
            NetworkMessage msg;
            msg.CopyFrom(pMessages[i]);
            m_incomingMessages.push(msg);
            pMessages[i]->Release();
        }
    }
    
    void SendQueuedMessages() {
        std::lock_guard<std::mutex> lock(m_messageQueueMutex);
        while (!m_outgoingMessages.empty()) {
            NetworkMessage &msg = m_outgoingMessages.front();
            SteamNetworkingSockets()->SendMessageToConnection(
                msg.connection,
                msg.data,
                msg.size,
                msg.sendFlags,
                nullptr
            );
            m_outgoingMessages.pop();
        }
    }
};
```

## Error Handling

### Connection Failures

```cpp
void OnConnectionStatusChanged(SteamNetConnectionStatusChangedCallback_t *pInfo) {
    if (pInfo->m_info.m_eState == k_ESteamNetworkingConnectionState_ProblemDetectedLocally) {
        // Connection failed or lost
        switch (pInfo->m_info.m_eEndReason) {
            case k_ESteamNetConnectionEnd_App_Generic:
                printf("Generic application error\n");
                break;
            case k_ESteamNetConnectionEnd_Misc_Timeout:
                printf("Connection timed out\n");
                // Show "Connection Lost" dialog
                break;
            case k_ESteamNetConnectionEnd_Misc_InternalError:
                printf("Internal error\n");
                break;
            // ... handle other error codes
        }
        
        // Attempt reconnection or return to menu
        HandleConnectionLost(pInfo->m_hConn);
    }
}
```

### Send Failures

```cpp
bool SendMessageSafe(HSteamNetConnection hConn, const void *data, size_t size) {
    EResult result = SteamNetworkingSockets()->SendMessageToConnection(
        hConn,
        data,
        size,
        k_nSteamNetworkingSend_Reliable,
        nullptr
    );
    
    if (result != k_EResultOK) {
        printf("Failed to send message: %d\n", result);
        
        if (result == k_EResultInvalidParam) {
            // Invalid parameters
            return false;
        }
        else if (result == k_EResultInvalidState) {
            // Connection not ready or closed
            return false;
        }
        else if (result == k_EResultLimitExceeded) {
            // Send buffer full, wait and try again
            return false;
        }
    }
    
    return true;
}
```

### Message Validation

Always validate incoming messages:

```cpp
bool ValidateMessage(SteamNetworkingMessage_t *pMsg) {
    // Check minimum size
    if (pMsg->m_cbSize < sizeof(MessageHeader)) {
        printf("Message too small\n");
        return false;
    }
    
    MessageHeader *header = (MessageHeader*)pMsg->m_pData;
    
    // Check message type is valid
    if (header->type == 0 || header->type >= MSG_TYPE_COUNT) {
        printf("Invalid message type: %d\n", header->type);
        return false;
    }
    
    // Check size matches
    if (pMsg->m_cbSize != sizeof(MessageHeader) + header->size) {
        printf("Size mismatch\n");
        return false;
    }
    
    return true;
}
```

## Best Practices

### 1. Choose Right Reliability for Each Message Type

```cpp
// Unreliable: Frequent updates where loss is acceptable
SendPlayerPosition(hConn, position);  // Send every frame

// Reliable: Important events that must arrive
SendChatMessage(hConn, message);
SendGameEvent(hConn, event);
SendPlayerDeath(hConn, playerId);
```

### 2. Monitor Connection Quality

```cpp
void MonitorConnection(HSteamNetConnection hConn) {
    SteamNetConnectionRealTimeStatus_t status;
    if (SteamNetworkingSockets()->GetConnectionRealTimeStatus(
        hConn, &status, 0, nullptr) == k_EResultOK) {
        
        if (status.m_nPing > 200) {
            // High latency - show warning, reduce update rate
            ShowLagIndicator();
        }
        
        if (status.m_flConnectionQualityLocal < 0.5f) {
            // Poor connection quality
            ReduceNetworkTraffic();
        }
    }
}
```

### 3. Use Lanes for Priority

```cpp
void ConfigureConnectionLanes(HSteamNetConnection hConn) {
    // Lane 0: Critical game events (high priority)
    // Lane 1: Player input (medium priority)
    // Lane 2: Chat and other non-critical (low priority)
    
    int lanes[3] = {0, 1, 2};
    int priorities[3] = {1000, 500, 100};
    int weights[3] = {50, 30, 20};
    
    SteamNetworkingSockets()->ConfigureConnectionLanes(
        hConn,
        3,
        lanes,
        priorities,
        weights
    );
}

// Send on specific lane
void SendCriticalEvent(HSteamNetConnection hConn, const void *data, size_t size) {
    SteamNetworkingSockets()->SendMessageToConnection(
        hConn,
        data,
        size,
        k_nSteamNetworkingSend_Reliable | (0 << 16),  // Lane 0
        nullptr
    );
}
```

### 4. Limit Send Rate

```cpp
void SendWithRateLimit(HSteamNetConnection hConn) {
    // Check if we can send now
    SteamNetworkingMicroseconds usecNow = SteamNetworkingUtils()->GetLocalTimestamp();
    SteamNetworkingMicroseconds usecWhenCanSend = 
        SteamNetworkingSockets()->GetNextAvailableSendTime(hConn);
    
    if (usecWhenCanSend <= usecNow) {
        // OK to send
        SendMessage(hConn, data, size);
    } else {
        // Wait until usecWhenCanSend or queue for later
        QueueForLater(data, size, usecWhenCanSend);
    }
}
```

### 5. Clean Shutdown

```cpp
void Shutdown() {
    // Close all connections gracefully
    for (HSteamNetConnection hConn : connections) {
        SteamNetworkingSockets()->CloseConnection(
            hConn,
            0,
            "Shutting down",
            true  // Enable linger mode for graceful close
        );
    }
    
    // Close listen sockets
    for (HSteamListenSocket hSocket : listenSockets) {
        SteamNetworkingSockets()->CloseListenSocket(hSocket);
    }
    
    // Give time for graceful close
    for (int i = 0; i < 100; i++) {
        SteamNetworkingSockets()->RunCallbacks();
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
    
    // Shutdown library
    GameNetworkingSockets_Kill();
}
```

## Common Patterns

### Pattern: Lobby System

```cpp
class Lobby {
    struct Player {
        HSteamNetConnection connection;
        std::string name;
        bool ready;
    };
    
    std::vector<Player> players;
    
    void BroadcastPlayerList() {
        // Build player list message
        PlayerListMsg msg;
        for (const auto &player : players) {
            msg.add_player(player.name, player.ready);
        }
        
        // Send to all players
        for (const auto &player : players) {
            SendMessage(player.connection, msg);
        }
    }
    
    void OnPlayerReady(HSteamNetConnection hConn) {
        for (auto &player : players) {
            if (player.connection == hConn) {
                player.ready = true;
                BroadcastPlayerList();
                
                if (AllPlayersReady()) {
                    StartGame();
                }
                break;
            }
        }
    }
};
```

### Pattern: Client Prediction

```cpp
class ClientPrediction {
    struct InputCommand {
        uint32_t sequence;
        Vector3 move;
        float rotation;
        uint64_t timestamp;
    };
    
    std::deque<InputCommand> unacknowledgedInputs;
    Vector3 predictedPosition;
    
    void SendInput(const InputCommand &input) {
        // Send to server
        SendInputToServer(input);
        
        // Store for reconciliation
        unacknowledgedInputs.push_back(input);
        
        // Predict locally
        predictedPosition = ApplyInput(predictedPosition, input);
    }
    
    void OnServerUpdate(uint32_t lastAckedSequence, Vector3 serverPosition) {
        // Remove acknowledged inputs
        while (!unacknowledgedInputs.empty() && 
               unacknowledgedInputs.front().sequence <= lastAckedSequence) {
            unacknowledgedInputs.pop_front();
        }
        
        // Reconcile: Start from server position and replay unacked inputs
        Vector3 recomputedPosition = serverPosition;
        for (const auto &input : unacknowledgedInputs) {
            recomputedPosition = ApplyInput(recomputedPosition, input);
        }
        
        // If prediction was wrong, snap or interpolate
        if (Distance(recomputedPosition, predictedPosition) > threshold) {
            predictedPosition = recomputedPosition;
        }
    }
};
```

### Pattern: Server Authority

```cpp
class AuthoritativeServer {
    void OnClientInput(HSteamNetConnection hConn, const InputCommand &input) {
        Player *player = GetPlayer(hConn);
        if (!player) return;
        
        // Validate input (anti-cheat)
        if (!ValidateInput(input)) {
            return;  // Ignore invalid input
        }
        
        // Apply input on server
        player->position = ApplyInput(player->position, input);
        
        // Broadcast to other clients
        for (auto &otherClient : clients) {
            if (otherClient.connection != hConn) {
                SendPlayerState(otherClient.connection, player->id, player->position);
            }
        }
        
        // ACK back to sender
        SendInputAck(hConn, input.sequence, player->position);
    }
};
```

## Next Steps

- **Build Setup**: See [STATIC_LIBRARY_GUIDE.md](STATIC_LIBRARY_GUIDE.md) for building options
- **Implementation Details**: See [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) for deep dive
- **Examples**: Study complete examples in [examples/](examples/) directory
- **API Reference**: See header files in [include/steam/](include/steam/) for full API

## Common Issues and Solutions

### Issue: Connection Timeout

**Symptoms**: Connection never completes, timeout after 10-20 seconds

**Solutions**:
- Check firewall settings
- Verify server is listening on correct port
- Check NAT/router configuration
- Enable debug output to see detailed connection attempt info

### Issue: High Latency

**Symptoms**: Poor connection quality, high ping

**Solutions**:
- Check network path (use ping/traceroute)
- Configure send rate limits appropriately
- Use unreliable messages for frequent updates
- Consider geographic server placement

### Issue: Messages Not Received

**Symptoms**: Some messages don't arrive

**Solutions**:
- Check if using unreliable vs reliable correctly
- Call `RunCallbacks()` regularly in game loop
- Check `ReceiveMessagesOnConnection()` return value
- Verify message size doesn't exceed limits

For more help, see the [examples](examples/) and [tests](tests/) directories for working code samples.
