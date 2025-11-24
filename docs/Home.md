# MediaManager Documentation

Welcome to the **MediaManager** documentation wiki. This resource provides comprehensive guides for understanding, developing, and integrating with MediaManager Core.

---

## What is MediaManager?

MediaManager is a **high-performance music library organizer** with a split architecture:

- **MediaManager Core** - Java daemon that manages your music library
- **MediaManager UI** - Frontend that connects via Unix Socket (local) or gRPC (remote)

The backend-agnostic design allows you to build interfaces in **any language or framework**.

---

## Quick Links

###  Getting Started
- [[About MediaManager]] - Detailed project overview, architecture, and features
- [[Development Environment Setup]] - Install tools and configure your system
- [[Building Your First Client]] - Quick tutorial to get started

###  Core Concepts
- [[Protocol Buffers Guide]] - Understanding `.proto` files and message definitions
- [[Unix Socket Communication]] - Local IPC protocol and implementation
- [[gRPC Communication]] - Network-based RPC for remote access

###  Client Development
- [[Client Development Guide]] - Step-by-step guide to building custom interfaces
- [[Java Client Examples]] - Desktop and CLI implementations
- [[Python Client Examples]] - Scripting and automation
- [[Web Client Examples]] - JavaScript/TypeScript browser-based clients

### Advanced Topics
- [[Server Architecture]] - IPCManager internals and Core design
- [[Command Reference]] - Available operations and messages
- [[Testing and Debugging]] - Tools and troubleshooting techniques
- [[Deployment Strategies]] - Running on NAS, Docker, systemd

---

## Communication Protocols

MediaManager supports two methods:

| Protocol | Use Case | Speed | Security |
|----------|----------|-------|----------|
| **Unix Socket** | Same machine (local IPC) | ~1ms latency | File permissions |
| **gRPC** | Network (client-server) | TBD latency | TLS encryption |

Both use **Protocol Buffers** for efficient binary serialization.

---

## Technology Stack

- **Language**: Java 17+
- **Build**: Maven
- **IPC**: Unix Domain Sockets + gRPC
- **Serialization**: Protocol Buffers 3.x
- **Database**: SQLite / PostgreSQL

---

## Repository

🔗 **MediaManager Core**: [https://git.gustavomiranda.xyz/GHMiranda/MediaManager-core](https://git.gustavomiranda.xyz/GHMiranda/MediaManager-core)

---

## Getting Started

New to MediaManager? Follow this path:

1. Read [[About MediaManager]] to understand the project
2. Set up your environment with [[Development Environment Setup]]
3. Learn the basics in [[Protocol Buffers Guide]]
4. Build your first client following [[Client Development Guide]]

---

## Contributing

Want to contribute or build your own interface? Check out:
- [[Development Environment Setup]] - Tools you'll need
- [[Protocol Buffers Guide]] - Communication contract
- [[Client Development Guide]] - Implementation patterns

---

## Need Help?

- 📖 Browse the guides listed above
- 🐛 Check [[Testing and Debugging]] for common issues
- 💬 Open an issue on the repository

---

**Ready to build?** Start with [[Development Environment Setup]]! 