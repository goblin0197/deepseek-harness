# Agent Note: Direct client-address fence for Host configuration

Status: implemented

English | [中文](2026-08-28-configuration-client-address-fence.zh.md)

## Problem

The Web GUI can bind all interfaces for a LAN browser, while the Host settings service reads and writes deployment configuration. Host and Origin trust can authorize the LAN authority, but they do not identify the browser's direct TCP peer. The Client UI therefore needs an explicit capability for configuration access; deriving it from the page hostname confuses the server address with the browser address and leaves a LAN settings request in memory-only mode.

## Decision

`dsh-client-connection` accepts `configurationClientAddresses`, a list of IPv4 or IPv6 literals matched against Node's direct `socket.remoteAddress`. Loopback peers are always allowed. IPv4-mapped IPv6 peers normalize to their IPv4 literal, IPv6 literals compare case-insensitively, and malformed or whitespace-padded entries fail plugin load. Forwarded address headers are never consulted.

The Connection carrier applies this check after browser authentication to `/api/settings` and its descendants, returning 403 for a disallowed peer. The authenticated frontend index receives a boolean capability through a structured global injection. The Client settings owner selects the Host mirror only when that capability is true; other browsers retain the memory mirror. The capability is an access hint, not identity: all Host API routes still require the existing Host/Origin fence and signed browser session.

The local LAN overlay binds the GUI to `0.0.0.0`, trusts the server's LAN authority, and explicitly permits `192.168.1.249` as a configuration client address. The old broad configuration toggle is not retained.

## Alternatives considered

**Infer permission from the browser page hostname.** Rejected: the hostname identifies the server socket (`192.168.1.100` in the reported deployment), not the client at `192.168.1.249`.

**Trust forwarded client-address headers.** Rejected: headers are caller-controlled unless a separate proxy contract is established, so they cannot authorize Host settings.

**Use the peer address as API identity.** Rejected: a socket peer is only a narrow configuration capability. Browser session authentication remains the identity for every Host API method and stream.

**Keep a broad trusted-host configuration switch.** Rejected: a Host/Origin authority grant should not silently grant settings writes to every client that can name that authority. The explicit address list makes the extra privilege visible in composition.

## Consequences

LAN browsers can use Host settings only when their direct address is configured, while loopback development remains unchanged. A network change or a different client address requires changing the overlay and restarting the Web process. Proxies and port-forwarding deployments need an explicit, separately reviewed peer-address policy; this decision does not interpret forwarding headers or add proxy identity support. Static assets and non-settings API routes keep their existing reachability and browser-session rules.
