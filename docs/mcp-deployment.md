# VulnClaw MCP Tool Deployment Guide

## Overview

VulnClaw maintains 5 Model Context Protocol (MCP) services: 2 are local implementations ready out of the box, 2 require external service deployment, and 1 is a remotely hosted service.

| Service | Mode | Status | Purpose |
|---|---|---|---|
| fetch | Local (httpx) | Out of the box | HTTP/HTTPS requests, GET/POST/PUT methods, headers/params/cookies/body/json/form, API testing |
| memory | Local (JSON) | Out of the box | Cross-session memory persistence |
| chrome-devtools | stdio MCP | Requires deployment | Browser automation, JS execution, screenshots |
| burp | SSE MCP | Requires deployment | Packet capture, replay, HTTP interception (replaces Yakit) |
| scanmalware | Remote streamable-http | Out of the box (disabled by default) | URL/domain threat intel, sandbox scan verdicts, CT/DNS infrastructure correlation |

`fetch` is a built-in local tool requiring no external MCP process. The default behavior sends a direct `GET` request and returns the full response body. The model can specify `method`, `headers`, `params`, `cookies`, `body`, `data`, `form`, `json`, `timeout`, `follow_redirects`, `verify_tls`, and `max_body_chars`. Omitting `max_body_chars` or setting it to `0` leaves the response untruncated; only positive integers limit response length. To support self-signed certificates in CTF and lab targets, `verify_tls` defaults to `false`; set it explicitly to `true` when strict certificate validation is required.

---

## 1. Chrome DevTools MCP

### Repository

https://github.com/ChromeDevTools/chrome-devtools-mcp

### Prerequisites

- Node.js LTS (v20+)
- Google Chrome browser (Stable or Chrome for Testing)
- ffmpeg (optional, required for screencast features)

### Installation

No manual installation is required; VulnClaw automatically fetches it via `npx -y chrome-devtools-mcp@latest`.

### Start Chrome Remote Debugging

PowerShell:

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=C:\tmp\chrome-debug
```

cmd:

```bat
"C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir=C:\tmp\chrome-debug
```

Linux/Mac:

```bash
google-chrome --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-debug
```

### VulnClaw Configuration

Edit `~/.vulnclaw/config.yaml` (default Windows path `C:\Users\<username>\.vulnclaw\config.yaml`):

```yaml
mcp:
  servers:
    chrome-devtools:
      enabled: true
      transport:
        type: stdio
        command: npx
        args:
          - "-y"
          - "chrome-devtools-mcp@latest"
          - "--browser-url=http://127.0.0.1:9222"
```

Enable via CLI:

```bash
vulnclaw config set mcp.servers.chrome-devtools.enabled true
```

To customize `--browser-url`, edit `config.yaml` directly.

### Capabilities (31+ Tools)

- **Input automation**: clicks, drag-and-drop, form filling, dialog handling
- **Navigation**: page management, URL navigation, element waiting
- **Performance analysis**: trace recording, Google CrUX integration
- **Network**: request monitoring, network interception
- **Debugging**: screenshots, console logs, Lighthouse auditing
- **Memory**: heap snapshot analysis
- **Emulation**: device and viewport emulation

### Penetration Testing Scenarios

- Navigating to target pages and capturing screenshot evidence
- Executing JavaScript to inspect DOM XSS vulnerabilities
- Monitoring network traffic to discover hidden API endpoints
- Automating form interactions to test CSRF and authentication bypasses

---

## 2. Burp Suite MCP (Yakit Alternative)

### Repository

https://github.com/PortSwigger/mcp-server

### Prerequisites

- Java (available in PATH, verify via `java --version`)
- Burp Suite Professional (Community edition has limited capabilities)
- `jar` utility available in PATH

### Installation Steps

#### Step 1: Clone and Build

```bash
git clone https://github.com/PortSwigger/mcp-server.git burp-mcp
cd burp-mcp
./gradlew embedProxyJar
# On Windows use: gradlew.bat embedProxyJar
# Artifact: build/libs/burp-mcp-all.jar
```

#### Step 2: Load into Burp Suite

1. Open Burp Suite -> Extensions tab
2. Click Add -> select Type: Java
3. Select `build/libs/burp-mcp-all.jar`
4. Click Next to finish loading

#### Step 3: Enable MCP Server

1. Navigate to the MCP tab in Burp Suite
2. Check "Enabled"
3. Default listener runs on `http://127.0.0.1:9876`
4. Optional: modify Host/Port if needed

### VulnClaw Configuration

Edit `~/.vulnclaw/config.yaml`:

```yaml
mcp:
  servers:
    burp:
      enabled: true
      transport:
        type: sse
        url: "http://127.0.0.1:9876"
```

VulnClaw connects directly to the SSE endpoint exposed by the Burp extension without needing a separate `java -jar` proxy process.

### Capabilities

- **Packet Capture**: Inspect requests and responses in Proxy History
- **Replay**: Construct and send custom HTTP requests
- **Interception**: Modify requests and responses in flight
- **Scanner**: Trigger Burp Scanner (Pro edition)
- **Intruder**: Parameterized fuzzing and attacks

### Yakit Comparison Reference

| Feature | Yakit | Burp MCP |
|---|---|---|
| MITM capture | MITM hijack | Proxy History |
| Request replay | Web Fuzzer | send_http1_request |
| Traffic analysis | Traffic analysis | get_proxy_history |
| Vulnerability scanning | Plugin scan | Burp Scanner |
| MCP integration | Not implemented (Issue #2703) | Official support v1.3.0 |

---

## 3. ScanMalware MCP (Remote Hosted, URL/Domain Threat Intelligence)

### Repositories & Resources

- MCP endpoint: <https://mcp.scanmalware.com/mcp>
- Open-source implementation: <https://github.com/scanmalware/mcp-server> (Apache-2.0)
- API documentation: <https://scanmalware.com/api-docs>

### Prerequisites

No installation, no local processes, and no API key required. This is a remotely hosted service operated by Triop AB, which VulnClaw connects to directly via streamable-http.

> [!WARNING]
> **Privacy Notice**: When enabled, queried domains and URLs are transmitted to scanmalware.com. Because of this, the service defaults to `enabled: false` in `BUILTIN_MCP_SERVERS` and must be enabled explicitly.

### VulnClaw Configuration

Built-in by default; enable via CLI:

```bash
vulnclaw config set mcp.servers.scanmalware.enabled true
```

Or edit `~/.vulnclaw/config.yaml`:

```yaml
mcp:
  servers:
    scanmalware:
      enabled: true
      transport:
        type: streamable-http
        url: https://mcp.scanmalware.com/mcp
```

Anonymous calls are rate-limited to 600 requests/minute. For higher limits, custom HTTP headers can be added under `transport.env` (for streamable-http, `env` maps directly to request headers).

### Capabilities (128 Tools)

- **Scanning**: Submit URLs for sandbox rendering, verdicts, screenshots, and network request graphs
- **Threat Verdicts**: `security_verdict` (risk_level, confidence, risk_factors), YARA matching, IDS alerts
- **Infrastructure Correlation**: Domain <-> IP <-> ASN mapping, JARM hashes, TLS/RDAP, Certificate Transparency (CT) records, and similar domain discovery
- **Fingerprinting**: favicon mmh3, screenshot hashing, TLSH/ssdeep fuzzy hashes, JavaScript fingerprints
- **Content Retrieval**: OCR text, page tech stack, pastejacking events
- **SMQL**: Query scan archives across 120+ filter fields

### Penetration Testing Scenarios

- Checking whether a target domain has historical scan records and verdicts
- Discovering attacker infrastructure via favicon, JARM, or screenshot hashes
- Enumerating subdomains and related lookalike domains with CT records
- Inspecting true redirect chains and client-side JavaScript behavior in phishing analysis

---

## Quick Verification

### Verify Chrome DevTools MCP

```bash
# 1. Start Chrome in remote debugging mode
# 2. Launch VulnClaw
vulnclaw chat

# 3. Enter a test prompt
> Open http://example.com and take a screenshot
```

### Verify Burp MCP

```bash
# 1. Start Burp Suite with MCP extension enabled
# 2. Launch VulnClaw
vulnclaw chat

# 3. Enter a test prompt
> Show Burp proxy capture history
```

### Verify ScanMalware MCP

```bash
# 1. Enable service (no local deployment needed)
vulnclaw config set mcp.servers.scanmalware.enabled true

# 2. Launch VulnClaw
vulnclaw chat

# 3. Enter a test prompt
> Check scan verdicts for example.com on ScanMalware
```

---

## Troubleshooting

### Chrome DevTools Connection Failure

1. Confirm Chrome remote debugging is active: `curl http://127.0.0.1:9222/json`
2. Confirm Node.js is installed: `node --version`
3. Try running manually: `npx -y chrome-devtools-mcp@latest --browser-url=http://127.0.0.1:9222`
4. Confirm `config.yaml` specifies `--browser-url=http://127.0.0.1:9222`

### Burp MCP Connection Failure

1. Confirm the MCP tab in Burp displays "Enabled"
2. Confirm port accessibility: `curl http://127.0.0.1:9876`
3. Confirm Java version: `java --version` (Java 11+ required)
4. Check that the JAR path in Burp extensions is correct
