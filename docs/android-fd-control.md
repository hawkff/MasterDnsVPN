# Android FD Control

MasterDnsVPN can run as a sidecar inside Android VPN apps that provide a
Unix-domain socket protect server. This is intended for NekoBox-style
integrations where the app owns `VpnService.protect(fd)`.

## Configuration

TOML:

```toml
FD_CONTROL_UNIX_SOCKET = "protect_path"
```

Environment override:

```sh
MASTERDNSVPN_PROTECT_PATH=protect_path
```

`MASTERDNSVPN_PROTECT_PATH` wins when both the environment variable and TOML key
are set. When both are empty, MasterDnsVPN keeps its normal desktop behavior and
does not install socket-control callbacks.

## Scope

When fd control is configured, the client protects upstream resolver/carrier
sockets before connect or bind:

- UDP resolver sockets used by pooled query exchange, session setup, MTU probes,
  resolver health checks, and validation.
- Unconnected UDP tunnel worker sockets used to send DNS-carrier packets to
  upstream resolvers.

The local SOCKS/TCP listener and optional local DNS listener are app-facing local
endpoints and are not protected.

## Protocol

The client matches the matsuri/libneko protect server framing:

1. Open an `AF_UNIX`, `SOCK_STREAM` connection to the configured path.
2. Send exactly one socket fd with `SCM_RIGHTS`.
3. Include one dummy payload byte, `0x01`.
4. Read exactly one status byte.
5. Treat `0x01` as success.
6. Treat `0x00`, short reads, read errors, connect errors, send errors, or
   missing fd-passing support as failure.
7. Close the Unix control connection.

Protection failure fails the upstream socket creation. The client does not fall
back to an unprotected resolver socket when fd control is configured.
