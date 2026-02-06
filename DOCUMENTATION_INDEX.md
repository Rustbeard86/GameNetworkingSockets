# GameNetworkingSockets Documentation Index

Welcome to the GameNetworkingSockets documentation! This index will help you find the information you need.

## Getting Started

**New to GameNetworkingSockets?** Start here:

1. Read the [README.md](README.md) for a high-level overview
2. Follow the [Quick Start in the Integration Guide](INTEGRATION_GUIDE.md#quick-start) to see a minimal working example
3. Review [BUILDING.md](BUILDING.md) to understand build requirements

## Documentation Guides

### [Implementation Guide](IMPLEMENTATION_GUIDE.md)
**For developers who want to understand HOW it works**

This comprehensive guide explains the internal architecture and design of GameNetworkingSockets:

- **Core Architecture**: How the library is structured (ISteamNetworkingSockets, ISteamNetworkingMessages, etc.)
- **Connection Management**: Lifecycle of connections, different connection types (UDP, P2P, SDR)
- **Message Reliability**: How reliable and unreliable messages work, message lanes and priorities
- **SNP Protocol**: The sophisticated reliability layer based on ack vectors (like DCCP/QUIC)
- **Security**: Encryption (AES-GCM-256), key exchange (Curve25519), per-packet authentication
- **P2P & NAT Traversal**: How ICE works, symmetric connect mode, signaling requirements
- **Performance**: Connection quality monitoring, network simulation, optimization tips

**Read this if you:**
- Want to understand the "why" behind design decisions
- Need to debug complex networking issues
- Want to contribute to the project
- Are evaluating whether this library meets your needs

### [Integration Guide](INTEGRATION_GUIDE.md)
**For developers who want to USE it in their game**

A practical, code-heavy guide for integrating GameNetworkingSockets into your game:

- **Quick Start**: 5-minute minimal example to get you running
- **Integration Approaches**: Direct API vs Ad-Hoc messaging vs vcpkg
- **Client-Server Setup**: Complete step-by-step code for creating servers and clients
- **P2P Setup**: How to establish peer-to-peer connections
- **Message Handling Patterns**: Serialization, protobuf, batching
- **Game Loop Integration**: Single-threaded and multi-threaded approaches
- **Error Handling**: Connection failures, send failures, message validation
- **Best Practices**: Choosing reliability, monitoring quality, using lanes, rate limiting
- **Common Patterns**: Lobbies, client prediction, server authority

**Read this if you:**
- Are implementing networking in your game
- Want practical code examples
- Need to understand best practices
- Are porting existing networking code

### [Static Library Guide](STATIC_LIBRARY_GUIDE.md)
**For developers who want to BUILD and LINK the static library**

Comprehensive instructions for building GameNetworkingSockets as a static library and using it in your project:

- **Building**: Platform-specific instructions (Linux, macOS, Windows, MinGW)
- **CMake Integration**: Three approaches (find_package, add_subdirectory, FetchContent)
- **Manual Integration**: Non-CMake build systems (Makefile, Visual Studio, etc.)
- **Linking Requirements**: All system and dependency libraries needed
- **Troubleshooting**: Common build and link errors with solutions

**Read this if you:**
- Want to compile the library yourself
- Need to integrate with an existing build system
- Want static linking (single executable, no DLLs)
- Are having build or link errors

## Quick Reference

### Essential Files

- **[README.md](README.md)** - Project overview, features, quick API overview
- **[BUILDING.md](BUILDING.md)** - How to build the library
- **[README_P2P.md](README_P2P.md)** - Peer-to-peer specific information
- **[SECURITY.md](SECURITY.md)** - Security policy and vulnerability reporting

### API Documentation

The main API is documented in the header files:

- **[include/steam/isteamnetworkingsockets.h](include/steam/isteamnetworkingsockets.h)** - Primary connection-oriented API
- **[include/steam/isteamnetworkingmessages.h](include/steam/isteamnetworkingmessages.h)** - Ad-hoc messaging API
- **[include/steam/isteamnetworkingutils.h](include/steam/isteamnetworkingutils.h)** - Utility functions
- **[include/steam/steamnetworkingtypes.h](include/steam/steamnetworkingtypes.h)** - Types and constants
- **[include/steam/steamnetworkingcustomsignaling.h](include/steam/steamnetworkingcustomsignaling.h)** - Custom signaling for P2P

### Examples

Working code examples are in the [examples/](examples/) directory:

- **[examples/example_chat.cpp](examples/example_chat.cpp)** - Simple client/server chat program
- **[examples/trivial_signaling_client.cpp](examples/trivial_signaling_client.cpp)** - P2P signaling client
- **[examples/trivial_signaling_server.go](examples/trivial_signaling_server.go)** - P2P signaling server

### Tests

Test cases demonstrate advanced features:

- **[tests/test_p2p.cpp](tests/test_p2p.cpp)** - P2P connection test

## Common Use Cases

### "I want to add multiplayer to my game"
1. Start with [Integration Guide - Quick Start](INTEGRATION_GUIDE.md#quick-start)
2. Follow [Basic Client-Server Setup](INTEGRATION_GUIDE.md#basic-client-server-setup)
3. Review [Message Handling Patterns](INTEGRATION_GUIDE.md#message-handling-patterns)

### "I need peer-to-peer for my game"
1. Read [README_P2P.md](README_P2P.md) to understand requirements
2. Follow [P2P Setup in Integration Guide](INTEGRATION_GUIDE.md#p2p-setup)
3. Study [P2P and NAT Traversal in Implementation Guide](IMPLEMENTATION_GUIDE.md#p2p-and-nat-traversal)

### "I'm having build/link errors"
1. Check [Static Library Guide - Troubleshooting](STATIC_LIBRARY_GUIDE.md#troubleshooting)
2. Review [BUILDING.md](BUILDING.md) for platform-specific requirements
3. Verify all dependencies are installed

### "How does this compare to other libraries?"
1. Read [Implementation Guide - Overview](IMPLEMENTATION_GUIDE.md#overview) for design philosophy
2. Review feature list in [README.md](README.md)
3. Check [Implementation Guide - SNP Protocol](IMPLEMENTATION_GUIDE.md#the-snp-protocol) for technical details

### "I want to understand the wire protocol"
1. Read [src/steamnetworkingsockets/clientlib/SNP_WIRE_FORMAT.md](src/steamnetworkingsockets/clientlib/SNP_WIRE_FORMAT.md)
2. Review [Implementation Guide - SNP Protocol](IMPLEMENTATION_GUIDE.md#the-snp-protocol)

## Additional Resources

### External Documentation

- **[Steamworks SDK Documentation](https://partner.steamgames.com/doc/api/ISteamNetworkingSockets)** - Web-based API docs (some features are Steam-only)
- **[Steam Datagram Relay](https://partner.steamgames.com/doc/features/multiplayer/steamdatagramrelay)** - Steam's relay network (not available in open-source version)

### Community

- **GitHub Issues** - Report bugs, request features
- **Examples** - Study working code in [examples/](examples/) and [tests/](tests/)

### Academic Background

The reliability layer is based on research and RFCs:
- DCCP RFC 4340, Section 11.4 (Ack Vector)
- Google QUIC protocol design
- [Glenn Fiedler's article on reliable ordered messages](https://gafferongames.com/post/reliable_ordered_messages/)

## Documentation Overview

```
GameNetworkingSockets/
│
├── README.md                    ← Start here: Project overview
├── DOCUMENTATION_INDEX.md       ← This file: Documentation roadmap
│
├── IMPLEMENTATION_GUIDE.md      ← How it works (architecture, protocols)
├── INTEGRATION_GUIDE.md         ← How to use it (code examples, patterns)
├── STATIC_LIBRARY_GUIDE.md      ← How to build it (static library)
│
├── BUILDING.md                  ← General build instructions
├── BUILDING_WINDOWS_MANUAL.md   ← Windows manual build
├── README_P2P.md                ← P2P specific information
├── SECURITY.md                  ← Security policy
│
├── include/steam/               ← API headers (source of truth)
│   ├── isteamnetworkingsockets.h
│   ├── isteamnetworkingmessages.h
│   └── ...
│
├── examples/                    ← Working code examples
│   ├── example_chat.cpp
│   └── ...
│
└── tests/                       ← Test cases
    └── test_p2p.cpp
```

## Contributing

If you find errors or want to improve the documentation:

1. Check existing GitHub issues
2. Submit a pull request with your improvements
3. Follow the existing documentation style

---

**Need help?** Can't find what you're looking for? Create a GitHub issue with the "documentation" label.
