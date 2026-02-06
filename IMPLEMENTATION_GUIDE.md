# GameNetworkingSockets Implementation Guide

## Table of Contents
1. [Overview](#overview)
2. [Core Architecture](#core-architecture)
3. [Connection Management](#connection-management)
4. [Message Types and Reliability](#message-types-and-reliability)
5. [The SNP Protocol](#the-snp-protocol)
6. [Security and Encryption](#security-and-encryption)
7. [P2P and NAT Traversal](#p2p-and-nat-traversal)
8. [Performance and Quality of Service](#performance-and-quality-of-service)

## Overview

GameNetworkingSockets is a sophisticated networking library designed for real-time multiplayer games. It provides a connection-oriented API similar to TCP but with message-oriented semantics like UDP, specifically optimized for gaming workloads.

### Why GameNetworkingSockets?

Traditional networking solutions have limitations for games:
- **TCP**: Stream-oriented (not message-oriented), head-of-line blocking affects all data, congestion control not optimized for real-time traffic
- **Raw UDP**: No reliability, ordering, or built-in security; requires custom implementation of these features
- **GameNetworkingSockets**: Combines the best of both - reliable when needed, unreliable when appropriate, with game-specific optimizations

## Core Architecture

### Main Components

The library is structured around several key interfaces and subsystems:

```
┌─────────────────────────────────────────────────────┐
│          Application Layer (Your Game)              │
└─────────────────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
┌────────▼────────┐           ┌─────────▼──────────┐
│ISteamNetworking │           │ISteamNetworking    │
│Sockets          │           │Messages            │
└────────┬────────┘           └─────────┬──────────┘
         │                               │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │   Connection Management       │
         │   - UDP Direct                │
         │   - P2P (ICE/WebRTC)          │
         │   - SDR Relay (Steam only)    │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │   SNP (SteamNetworkingProtocol)│
         │   - Reliability Layer         │
         │   - Fragmentation/Reassembly  │
         │   - Ack Vectors               │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │   Security Layer              │
         │   - AES-GCM-256 Encryption    │
         │   - Curve25519 Key Exchange   │
         └───────────────┬───────────────┘
                         │
         ┌───────────────▼───────────────┐
         │   Transport Layer (UDP)       │
         └───────────────────────────────┘
```

### ISteamNetworkingSockets

This is the primary interface for connection-oriented networking. Key features:

- **Connection Creation**: Create connections to remote hosts by IP address or abstract identity
- **Message Sending**: Send reliable or unreliable messages with configurable lanes/priorities
- **Connection State**: Monitor connection status, quality metrics, and remote peer information
- **Configuration**: Fine-tune behavior with extensive configuration options

### ISteamNetworkingMessages

A simpler "ad-hoc" interface similar to UDP's `sendto()`/`recvfrom()` model:

- Automatically manages connections under the hood
- Send messages by specifying recipient identity (not connection handle)
- Ideal for porting existing UDP-based code
- Uses symmetric connect mode for P2P scenarios

### ISteamNetworkingUtils

Utility functions for the library:

- Configuration management
- Debug output control
- Time synchronization
- Network simulation for testing (artificial lag, packet loss)

## Connection Management

### Connection Lifecycle

```
┌──────────────┐
│   Created    │  ConnectByIPAddress() or CreateListenSocket()
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Connecting  │  Handshake, key exchange, authentication
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Connected   │  Active data transfer
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ ClosedByPeer │  Or Problem/FinWait/Linger depending on closure type
└──────┬───────┘
       │
       ▼
┌──────────────┐
│    Dead      │  Connection object can be destroyed
└──────────────┘
```

### Connection Types

1. **Direct UDP Connections**
   - Simplest form: Client connects to server IP:Port
   - No special infrastructure required beyond reachable network
   - Used in: LAN games, dedicated servers with public IPs

2. **P2P Connections (ICE)**
   - NAT traversal using ICE (Interactive Connectivity Establishment)
   - Requires: Signaling service, STUN servers, optionally TURN relays
   - Process:
     1. Both peers connect to signaling service
     2. Exchange ICE candidates through signaling
     3. STUN discovers public IPs and opens NAT pinholes
     4. Direct connection established or falls back to TURN relay

3. **SDR Relay Connections (Steam only)**
   - Traffic routed through Steam's global relay network
   - Benefits: DDoS protection, optimal routing, no IP exposure
   - Not available in open-source version

### Connection Establishment

**Server Side:**
```cpp
// Create listen socket
ISteamNetworkingSockets *pInterface = SteamNetworkingSockets();
SteamNetworkingIPAddr serverAddr;
serverAddr.Clear();
serverAddr.m_port = 27015;

HSteamListenSocket hListenSocket = pInterface->CreateListenSocketIP(serverAddr, 0, nullptr);

// Poll for new connections
SteamNetworkingMessage_t *pIncomingMsg = nullptr;
while (true) {
    pInterface->RunCallbacks();
    // Process state change callbacks for new connections
}
```

**Client Side:**
```cpp
// Connect to server
SteamNetworkingIPAddr serverAddr;
serverAddr.ParseString("192.168.1.100:27015");

HSteamNetConnection hConnection = pInterface->ConnectByIPAddress(serverAddr, 0, nullptr);

// Wait for connection
while (true) {
    pInterface->RunCallbacks();
    // Check for connected state in callback
}
```

## Message Types and Reliability

### Reliability Modes

GameNetworkingSockets supports multiple message delivery modes:

1. **Unreliable (k_nSteamNetworkingSend_Unreliable)**
   - Fire and forget - no acknowledgment required
   - May arrive out of order or not at all
   - Lowest latency
   - Use for: Player position updates, cosmetic effects

2. **Unreliable No Delay (k_nSteamNetworkingSend_UnreliableNoDelay)**
   - Like Unreliable, but bypasses Nagle's algorithm
   - Sends immediately even if packet not full
   - Use for: Time-critical data that can't wait for batching

3. **Reliable (k_nSteamNetworkingSend_Reliable)**
   - Guaranteed delivery and ordering within the same lane
   - Retransmitted if lost
   - Use for: Chat messages, game events, state changes

4. **Reliable No Nagle (k_nSteamNetworkingSend_ReliableNoNagle)**
   - Like Reliable, but sends immediately
   - Use for: Time-critical reliable data

### Message Lanes and Priorities

Messages can be assigned to different "lanes" with priorities:

```cpp
// Configure lanes for a connection
int nLanes[3] = {1, 2, 3};  // Lane numbers
int nPriorities[3] = {1000, 100, 10};  // Higher = more priority
int nWeights[3] = {100, 50, 10};  // Bandwidth sharing weights

pInterface->ConfigureConnectionLanes(
    hConnection, 
    3,           // Number of lanes
    nLanes, 
    nPriorities, 
    nWeights
);
```

**Lane Benefits:**
- Prevent head-of-line blocking between different message types
- Critical data (player input) can bypass less critical data (chat)
- Bandwidth can be fairly shared or prioritized

## The SNP Protocol

SNP (SteamNetworkingProtocol) is the reliability layer that makes GameNetworkingSockets special.

### Key Innovations

1. **Ack Vector Model**
   - Instead of cumulative ACKs (like TCP), receiver sends bit vector
   - Each bit represents whether a specific packet was received
   - Sender knows exactly which packets need retransmission
   - Based on DCCP (RFC 4340) and Google QUIC

2. **Selective Retransmission**
   - Only lost segments are retransmitted
   - No need to resend successfully received data
   - More efficient than TCP's sliding window

3. **Packet Structure**
   ```
   ┌─────────────────────────────────────┐
   │  Header (Packet#, Timestamps, etc)  │
   ├─────────────────────────────────────┤
   │  Inline Stats (optional)            │
   ├─────────────────────────────────────┤
   │  SNP Frames:                        │
   │  - Unreliable message segments      │
   │  - Reliable message segments        │
   │  - Acknowledgments                  │
   │  - Stop waiting frames              │
   │  - Connection close frames          │
   └─────────────────────────────────────┘
   ```

4. **Fragmentation and Reassembly**
   - Messages larger than MTU are automatically fragmented
   - Segments can be interleaved with other messages
   - Efficient packing: Multiple small messages in one packet

### Wire Format Details

See [SNP_WIRE_FORMAT.md](src/steamnetworkingsockets/clientlib/SNP_WIRE_FORMAT.md) for complete protocol specification.

**Example Frame Types:**
- **Unreliable Segment**: `00emosss [message_num] [offset] [size] data`
- **Reliable Segment**: `010mmsss [stream_pos] [size] data`
- **Ack**: `11ssnnnn [block_info]...` (selective acknowledgment ranges)

### Congestion Control

- Tracks packet loss rate and RTT
- Dynamically adjusts send rate
- More aggressive than TCP for real-time requirements
- Recovers from congestion faster

## Security and Encryption

### Encryption

**Per-Packet Encryption:**
- Algorithm: AES-GCM-256 (Authenticated encryption)
- Each packet gets unique IV (Initialization Vector)
- Authentication prevents tampering
- Forward secrecy with ephemeral keys

### Key Exchange

**Process:**
1. Elliptic Curve Diffie-Hellman (Curve25519)
2. Each peer generates ephemeral key pair
3. Public keys exchanged during handshake
4. Shared secret derived using ECDH
5. Session keys derived from shared secret

**Certificate and Authentication:**
- Curve25519 signatures verify peer identity
- Supports custom certificate authorities
- Can integrate with existing authentication systems

### Why This Design?

- **AES-GCM**: Hardware acceleration on modern CPUs, authenticated encryption
- **Curve25519**: Fast, secure, constant-time implementation resists timing attacks
- **Per-packet encryption**: Prevents replay attacks, each packet independently decryptable

## P2P and NAT Traversal

### The NAT Problem

Network Address Translation (NAT) allows multiple devices to share one public IP, but creates challenges for peer-to-peer:

```
Peer A (NAT)          Internet          Peer B (NAT)
10.0.0.5:49001 <---> 1.2.3.4:49001 <?> 5.6.7.8:50123 <---> 192.168.1.10:50123
```

Neither peer knows the other's public IP:port mapping, and NAT typically blocks inbound connections.

### ICE Solution

Interactive Connectivity Establishment (ICE) solves this:

1. **Candidate Gathering**
   - Host candidates: Local LAN IPs
   - Server reflexive: Public IP discovered via STUN
   - Relay candidates: TURN server addresses

2. **Candidate Exchange**
   - Send candidates to peer via signaling service
   - Both peers know all possible addresses

3. **Connectivity Checks**
   - Try all candidate pairs
   - Successful binding creates NAT pinhole
   - Prioritize direct connections over relays

4. **Connection Establishment**
   - Best working candidate pair selected
   - Data flows directly (or via TURN if NAT is restrictive)

### Symmetric Connect Mode

Unique GameNetworkingSockets feature:

**Problem**: Traditional client/server role assignment creates races when both peers connect simultaneously.

**Solution**: Symmetric mode - both peers act as client AND server:
- Either peer can initiate
- Simultaneous initiation handled gracefully
- Single connection results from two attempts
- Simplifies P2P game logic

**Configuration:**
```cpp
SteamNetworkingConfigValue_t opt;
opt.SetInt32(k_ESteamNetworkingConfig_SymmetricConnect, 1);
```

### Requirements for P2P

1. **Signaling Service**: Exchange connection metadata (ICE candidates)
2. **STUN Server**: Discover public IP (public servers available)
3. **TURN Server**: Relay fallback when NAT piercing fails (requires hosting)
4. **Identity System**: Authenticate peers, matchmaking

See [README_P2P.md](README_P2P.md) for complete P2P setup guide.

## Performance and Quality of Service

### Statistics and Monitoring

Real-time connection metrics available:

```cpp
SteamNetConnectionRealTimeStatus_t status;
pInterface->GetConnectionRealTimeStatus(hConnection, &status, 0, nullptr);

// Available metrics:
// - status.m_nPing: Round trip time in milliseconds
// - status.m_flConnectionQualityLocal: 0.0-1.0 quality score
// - status.m_flOutPacketsPerSec: Outbound packet rate
// - status.m_flInBytesPerSec: Inbound data rate
// - status.m_nSendRateBytesPerSecond: Current send rate
```

### Network Simulation

Test your game under adverse conditions:

```cpp
// Add 100ms latency and 5% packet loss
SteamNetworkingUtils()->SetGlobalConfigValueFloat(
    k_ESteamNetworkingConfig_FakePacketLoss_Send, 5.0f);
SteamNetworkingUtils()->SetGlobalConfigValueInt32(
    k_ESteamNetworkingConfig_FakePacketLag_Send, 100);
```

### Optimization Tips

1. **Use Unreliable Messages for Frequent Updates**
   - Position updates, animation states
   - Send frequently enough that loss doesn't matter

2. **Batch Small Messages**
   - Send multiple messages in one API call
   - Reduces syscalls and network overhead

3. **Configure Lanes Appropriately**
   - Separate critical from non-critical traffic
   - Prevent chat messages blocking player input

4. **Monitor Connection Quality**
   - Adjust game behavior based on connection stats
   - Reduce update rate on poor connections
   - Show network indicators to players

5. **Use Message Budgets**
   - Control send rate to avoid overwhelming connection
   - Check `GetNextAvailableSendTime()` before sending

### Threading Considerations

- **Not thread-safe by default**: Call all methods from single thread
- Use callbacks or polling from game loop thread
- For multi-threaded games: Use mutexes or marshal to network thread

## Debugging

### Debug Output

Enable detailed logging:

```cpp
SteamNetworkingUtils()->SetDebugOutputFunction(
    k_ESteamNetworkingSocketsDebugOutputType_Msg, 
    MyDebugOutputFunction
);
```

### Common Issues

1. **Connection Timeout**: Check firewall, NAT, server reachability
2. **High Ping**: Network congestion, routing issues, use connection stats
3. **Packet Loss**: Wireless interference, congested links, simulate to test
4. **Handshake Failure**: Certificate issues, version mismatch, authentication failure

## Next Steps

- **Integration**: See [INTEGRATION_GUIDE.md](INTEGRATION_GUIDE.md) for how to add to your game
- **Static Library**: See [STATIC_LIBRARY_GUIDE.md](STATIC_LIBRARY_GUIDE.md) for building and linking
- **Examples**: Study [examples/example_chat.cpp](examples/example_chat.cpp) for working code
- **API Reference**: See headers in [include/steam/](include/steam/) for detailed API documentation
