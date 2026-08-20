# Feature Prompt: Single-File SPA HTML Channel Exporter & Publisher

## Objective
Implement an optional **Single-File SPA HTML Export & Publishing** capability in `anytype-heart`. When enabled, this feature converts an entire channel (root page and all nested sub-pages/objects) into a self-contained HTML file (`index.html`) with client-side JavaScript routing.

---

## CRITICAL REQUIREMENT: ZERO REGRESSION / ZERO DISRUPTION
- **Existing publishing functionality MUST NOT be altered or broken.**
- All current protobuf snapshot exports (`index.json.gz`), relation whitelisting, and `publishclient` workflows must continue to function unchanged as the default behavior.
- The Single-File SPA HTML mode must be introduced as an **additive, opt-in feature** (e.g. via an explicit request flag in `RpcPublishingCreateRequest` like `export_format = HTML_SPA` or a dedicated service mode).

---

## Architecture & Technical Requirements

### 1. Request Protocol Extension
- Extend Protobuf commands/messages in `pb/commands.proto` (or protobuf definitions) to support an optional export format flag in `RpcPublishingCreateRequest`:
  ```protobuf
  enum PublishFormat {
    FORMAT_DEFAULT = 0; // Existing index.json.gz + files/ format
    FORMAT_HTML_SPA = 1; // Single-file SPA HTML export
  }
  ```

### 2. Single-File SPA Generator (`core/publish/html_spa_exporter.go`)
Create a dedicated component that converts exported object blocks and sub-pages into HTML sections:
1. **Traverse Page Graph**: Walk through the root page and all nested sub-pages exported by `exportService.Export(..., IncludeNested: true)`.
2. **Block-to-HTML Renderer**:
   - Convert text, headings, lists, code blocks, and media references to valid HTML tags.
   - Convert internal object/page links (e.g., `anytype://<spaceId>/<objectId>`) into client-side route navigation attributes: `<a href="/sub-path" onclick="navigate(event, '/sub-path')">`.
3. **HTML SPA Template**:
   - Wrap all pages into `<div id="route-<id>" data-path="<path>" class="page">` elements.
   - Embed lightweight Vanilla JS for routing using `window.history.pushState`, `popstate`, and `DOMContentLoaded` listeners.
   - Include inline CSS for responsive layout, dark/light theme options, and page visibility toggling (`.page { display: none; } .page.active { display: block; }`).

### 3. File Asset & Attachment Bundling
- Ensure media/file attachments (images, audio, attachments from the `files/` directory) remain referenced with relative web-safe paths (`files/<hash>.<ext>`) or embedded as data URIs if below a configurable size threshold.
- The publish bundle directory must contain `index.html` (the SPA application) alongside `files/`.

### 4. Integration in `core/publish/service.go`
- In `publishToPublishServer`, inspect the request format:
  - If `FORMAT_DEFAULT` (or unspecified): Execute the exact existing pipeline (`processExportedData` -> `createIndexFile` -> `publishToServer`).
  - If `FORMAT_HTML_SPA`: Generate `index.html` via `SingleFileHtmlBuilder`, place it alongside `files/`, and proceed with standard uploading via `publishClientService.UploadDir`.

---

## Verification & Testing Checklist
1. **Regression Test**: Run all existing publish unit tests (`go test ./core/publish/...`) to ensure `FORMAT_DEFAULT` behavior remains 100% backward compatible.
2. **SPA Exporter Test**: Write unit tests for `SingleFileHtmlBuilder` verifying:
   - Root page and all sub-pages are rendered into `<div class="page">` blocks.
   - Internal links are transformed into `navigate()` client-side routing calls.
   - Client-side JS script tag is properly embedded.
3. **End-to-End Bundle Test**: Verify `tempPublishDir` output produces a valid `index.html` and `files/` folder without generating errors or corrupting existing publish server interactions.
