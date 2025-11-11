# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Mihomo (Meta Kernel) is a rule-based proxy kernel written in Go that supports multiple protocols (VMess, VLESS, Shadowsocks, Trojan, Snell, TUIC, Hysteria, etc.). It provides local HTTP/HTTPS/SOCKS servers, a built-in DNS server with DoH/DoT support, rule-based routing, remote proxy providers, and a comprehensive RESTful API.

## Build Commands

**Standard build:**
```bash
go build
```

**Build with gvisor TUN stack (recommended for production):**
```bash
go build -tags with_gvisor
```

**Platform-specific builds:**
```bash
# Current platform optimized
make linux-amd64-v3    # Linux AMD64 (v3 microarchitecture)
make darwin-arm64      # macOS ARM64
make windows-amd64-v3  # Windows AMD64

# All platforms
make all-arch          # Build for all platforms
make releases          # Build and package releases
```

**Quick build for common platforms:**
```bash
make all  # Builds linux-amd64-v3, linux-arm64, darwin-amd64-v3, darwin-arm64, windows-amd64-v3, windows-arm64
```

## Development Commands

**Run tests:**
```bash
go test ./...
# OR using Makefile
make vet
```

**Lint code:**
```bash
golangci-lint run ./...
# OR using Makefile
make lint
```

**Protocol testing suite (requires Docker):**
```bash
cd test
make test       # Run full protocol test suite
make benchmark  # Run benchmarks (Linux only)
```

**Test configuration file:**
```bash
./mihomo -t -f <config-file>
```

**Run with custom config:**
```bash
./mihomo -f <config-file> -d <home-directory>
```

## Architecture

### Core Components

**Tunnel (`tunnel/`)**: The central traffic processing engine that:
- Manages TCP and UDP packet queuing (multiple UDP worker goroutines based on GOMAXPROCS)
- Implements rule matching and proxy selection through the `mode` (Rule/Global/Direct)
- Maintains NAT table for connection tracking
- Coordinates with the sniffer for protocol detection
- Handles process matching via `findProcessMode`

**Adapter (`adapter/`)**: Defines proxy abstractions with three main categories:
- `adapter/outbound/`: Concrete protocol implementations (Shadowsocks, VMess, VLESS, Trojan, etc.)
- `adapter/outboundgroup/`: Proxy groups (URLTest, LoadBalance, Fallback, Selector, Relay)
- `adapter/provider/`: Remote proxy provider support for dynamic proxy lists
- `adapter/inbound/`: Inbound connection handling

**Hub (`hub/`)**: Configuration and API management layer:
- `hub/executor/`: Configuration parsing and application (ApplyConfig applies config to all components)
- `hub/route/`: RESTful API routes for external control (proxies, rules, connections, logs, DNS, etc.)
- The hub parses config and dispatches it to tunnel, listeners, DNS, etc.

**Listener (`listener/`)**: Implements inbound protocols:
- HTTP/HTTPS, SOCKS4/SOCKS4, Mixed (HTTP+SOCKS), Redir (transparent proxy)
- TProxy (Linux transparent proxy), TUN (with sing-tun integration)
- Protocol-specific listeners: Shadowsocks, Trojan, VMess, VLESS, Hysteria2, TUIC, Mieru
- Each listener converts inbound traffic to internal `C.ConnContext` or `C.PacketAdapter`

**DNS (`dns/`)**: Custom DNS resolver with:
- Multiple upstream types: DoH, DoT, DoQ, UDP, system DNS
- FakeIP support for IP allocation without real DNS queries
- DNS result enhancement and caching
- Policy-based DNS routing (nameserver-policy in config)
- EDNS0 subnet support

**Rules (`rules/`)**: Traffic routing logic:
- `rules/common/`: Rule implementations (DOMAIN, DOMAIN-SUFFIX, DOMAIN-KEYWORD, GEOIP, IP-CIDR, PROCESS-NAME, etc.)
- `rules/logic/`: Logical operators (AND, OR, NOT) for complex rule combinations
- `rules/provider/`: Remote rule set providers

**Component (`component/`)**: Shared utilities and subsystems:
- `component/dialer/`: Outbound dialer with bind interface, routing mark, SO_MARK support
- `component/resolver/`: DNS resolution coordination
- `component/sniffer/`: Protocol sniffing (HTTP, TLS SNI, QUIC) for domain extraction
- `component/fakeip/`: FakeIP pool management
- `component/geodata/`: GeoIP/GeoSite data loading (supports sing-box geosite format)
- `component/mmdb/`: MaxMind DB reader for IP geolocation
- `component/trie/`: Domain trie for efficient domain matching
- `component/process/`: Process name extraction (Linux/Windows/macOS via netlink/procfs)

**Transport (`transport/`)**: Protocol-specific transport layers:
- Obfuscation layers (simple-obfs, v2ray-plugin, shadowtls, restls)
- KCP, gRPC (gun), Hysteria transports
- Each transport is used by corresponding outbound adapters

### Data Flow

1. **Inbound**: Listener receives connection → creates `ConnContext`/`PacketAdapter` with metadata
2. **Tunnel**: Queues connection → applies rules → selects proxy/direct
3. **Sniffer**: Optional protocol sniffing to extract real domain from TLS SNI, HTTP Host
4. **DNS**: Resolves domains via configured nameservers or FakeIP
5. **Adapter**: Selected proxy establishes outbound connection using appropriate protocol
6. **Dialer**: Handles low-level socket options (bind interface, routing mark, TFO, MPTCP)

### Configuration System

- Config is YAML-based (see [docs/config.yaml](docs/config.yaml))
- Parsed by `config/config.go` into `config.Config` struct
- Applied atomically via `hub/executor.ApplyConfig()`
- Supports hot reload on SIGHUP signal
- External overrides via CLI flags or environment variables (e.g., `CLASH_CONFIG_FILE`)

### Constants and Interfaces

The `constant/` package defines core interfaces:
- `C.ProxyAdapter`: Base interface for all proxies (Dial, DialUDP, MarshalJSON, etc.)
- `C.Proxy`: Extended interface with health check support (URLTest, DelayHistory)
- `C.Rule`: Interface for routing rules (Match, RuleType, Adapter, Payload)
- `C.Metadata`: Connection metadata (source/dest IP, port, host, process name, etc.)
- `C.Conn` / `C.PacketConn`: Enhanced connection interfaces with chain information

## Key Implementation Details

**Build Tags**: The `with_gvisor` build tag enables gvisor-based TUN stack (sing-tun). Default build uses system TUN stack.

**Version Injection**: Version and build time are injected at build time via ldflags:
```
-X "github.com/metacubex/mihomo/constant.Version=$(VERSION)"
-X "github.com/metacubex/mihomo/constant.BuildTime=$(BUILDTIME)"
```

**Defensive Programming**: `net.DefaultResolver.Dial` is overridden to panic (exits with stack trace) if accidentally called, since mihomo uses custom DNS resolver.

**Process Matching**: Three modes in `find-process-mode`:
- `always`: Force match all connections to process names
- `strict` (default): mihomo decides when to enable based on performance
- `off`: Disable (recommended for routers)

**Special Commands**:
```bash
./mihomo convert-ruleset [...args]  # Convert rule sets between formats
./mihomo generate [...args]         # Generate utilities (e.g., ECH keypairs)
```

## Testing

Tests use Docker-based protocol servers. The test suite validates:
- TCP/UDP pingpong (bidirectional communication)
- Large data transfer
- Protocol correctness for Shadowsocks, VMess, Trojan, Snell with various obfuscation

Run single protocol test:
```bash
cd test
go test -v -run TestShadowsocks
```

## Linting

Enabled linters (`.golangci.yaml`):
- `gofumpt`: Stricter formatting than gofmt
- `staticcheck`: Advanced static analysis
- `govet`: Go's official vet tool
- `gci`: Import ordering (standard → github.com/metacubex/mihomo → third-party)

Import order convention:
1. Standard library
2. Internal mihomo packages (github.com/metacubex/mihomo/...)
3. Third-party packages

## Important Notes

- **Do not commit code using `net.DefaultResolver`** - always use `component/resolver` or DNS subsystem
- **GEOIP/GeoSite data**: Downloaded from `geox-url` in config, cached in home directory
- **RESTful API**: External controller listens on `external-controller` (HTTP) or `external-controller-tls` (HTTPS). Authentication via `secret` token in `Authorization: Bearer` header.
- **Cache/State**: Profile selections and FakeIP mappings are persisted in `<home>/.cache.db` (BBolt database)
- **License**: GPL-3.0. Downstream projects not affiliated with MetaCubeX must not use "mihomo" in their names.

## Common Development Patterns

When adding a new proxy protocol:
1. Create implementation in `adapter/outbound/<protocol>.go` implementing `C.ProxyAdapter`
2. Add transport layer in `transport/<protocol>/` if needed
3. Register parser in `adapter/parser.go`
4. Add corresponding inbound listener in `listener/<protocol>/` if supporting inbound
5. Update protocol list in documentation

When adding a new rule type:
1. Implement `C.Rule` interface in `rules/common/<rule>.go`
2. Add parser case in `rules/parser.go`
3. Ensure efficient matching (use tries for domains, CIDR for IPs)
