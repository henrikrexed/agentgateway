# MCP Response Compression

MCP tool call responses often contain large JSON payloads (tables, lists of objects) that consume significant context window tokens for LLM-based agents. Response compression converts these JSON payloads into compact tabular formats before they reach the client, reducing token usage without losing information.

## Supported Formats

| Format     | Description                                                                 |
|------------|-----------------------------------------------------------------------------|
| `markdown` | Converts JSON arrays of objects into markdown tables with headers and rows.  |
| `tsv`      | Converts to tab-separated values with a header row.                         |
| `csv`      | Converts to comma-separated values with proper CSV escaping.                |
| `none`     | No conversion applied (default).                                            |

### What Gets Compressed

Compression targets `CallToolResult` messages containing text content. It handles:

- **Direct arrays of objects** — flattened into tabular rows.
- **Wrapper objects with array fields** — scalar metadata is preserved as header lines, arrays are rendered as tables.
- **Nested values** — arrays of 5 or fewer items are shown inline; larger arrays show the first 5 items plus a count. Nested objects display as `{...}`.

Non-convertible JSON (scalars, deeply nested structures without tabular data) is left unchanged.

### Example

**Before (JSON, ~250 tokens):**
```json
[
  {"name": "web-server", "status": "running", "cpu": 45.2, "memory": 1024},
  {"name": "db-primary", "status": "running", "cpu": 78.1, "memory": 4096},
  {"name": "cache", "status": "stopped", "cpu": 0, "memory": 0}
]
```

**After (markdown, ~80 tokens):**
```
| name | status | cpu | memory |
| --- | --- | --- | --- |
| web-server | running | 45.2 | 1024 |
| db-primary | running | 78.1 | 4096 |
| cache | stopped | 0 | 0 |
```

## Configuration

### Kubernetes CRD

Response compression is configured per-target in the `AgentgatewayBackend` CRD under `mcpTargetSelector`:

```yaml
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
metadata:
  name: my-mcp-backend
spec:
  mcp:
    targets:
    - name: my-server
      backendRef:
        name: my-mcp-service
        port: 8080
      responseCompression:
        enabled: true
        format: "markdown"    # or "tsv", "csv"
```

#### Fields

| Field                              | Type    | Default  | Description                                          |
|------------------------------------|---------|----------|------------------------------------------------------|
| `responseCompression.enabled`      | boolean | `false`  | Whether to enable response compression for this target. |
| `responseCompression.format`       | string  | `"none"` | The compression format: `markdown`, `tsv`, `csv`, or `none`. |

> [!NOTE]
> Response compression is currently available only through the Kubernetes CRD configuration path.
> Local static configuration (`config.yaml`) does not yet support this field.

## Metrics

When compression is enabled, the following Prometheus metrics are exposed:

| Metric                                         | Type      | Description                                              |
|------------------------------------------------|-----------|----------------------------------------------------------|
| `mcp_response_compression_original_bytes`      | Histogram | Original response size in bytes before compression.      |
| `mcp_response_compression_compressed_bytes`    | Histogram | Response size in bytes after compression.                |
| `mcp_response_compression_ratio`               | Histogram | Ratio of compressed to original size (0.0–1.0).         |
| `mcp_response_compression_total`               | Counter   | Total number of compressions performed.                  |
| `mcp_response_compression_skipped_total`       | Counter   | Total responses that were not eligible for compression.  |

All metrics include labels: `gateway`, `listener`, `route`, `target`, and `format`.

## Architecture

The compression pipeline flows through these layers:

1. **CRD** — User sets `responseCompression` on an MCP target in the `AgentgatewayBackend` resource.
2. **Controller** — Translates the CRD field into the xDS `MCPTarget.ResponseCompression` proto message.
3. **xDS** — The proxy receives the configuration and maps the format string to an internal `CompressionFormat` enum.
4. **Proxy handler** — The `compress_stream()` function wraps the upstream response stream. For each `CallToolResult` message with text content, it calls `compress_response()` to convert JSON to the target format. Metrics are recorded for each compression attempt.

Compression is transparent to clients — they receive the already-converted text content in the tool call response.
