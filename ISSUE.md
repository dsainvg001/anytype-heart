# Feature Request: Single-File SPA HTML Export & Publishing for Channels / Spaces

## Problem Statement
When publishing a page or channel from Anytype to the web, the current mechanism exports page snapshots as Protobuf/JSON data (`index.json.gz`), requiring specialized frontend clients/readers to parse snapshots and handle navigation. Furthermore, sub-pages and internal links (`anytype://`) are not automatically formatted as zero-dependency, navigable web routes out of the box.

---

## Proposed Solution
Introduce an optional **Single-File SPA (Single Page Application) HTML Exporter & Publisher** in `anytype-heart`. When enabled, this feature converts an entire channel (the root page and all nested sub-pages) into a self-contained HTML file (`index.html`) featuring lightweight client-side JavaScript routing (`window.history.pushState` and `popstate`).

### Key Benefits
1. **Zero Server Changes Needed**: Works out of the box with the existing `anytype-publish-server` without modifying backend services.
2. **Instant Web Compatibility**: The generated `index.html` renders directly in any web browser or static web host without requiring frontend reader frameworks.
3. **Seamless Sub-page Navigation**: Sub-pages are rendered as `<div class="page" id="route-id" data-path="/sub-page">` sections, and internal object links are automatically transformed into client-side `navigate()` calls.

---

## Technical Design & Architecture

### 1. Opt-In Protocol Extension (`pb/commands.proto`)
Add an optional format enum to `RpcPublishingCreateRequest` to ensure **100% backward compatibility**:

```protobuf
enum PublishFormat {
  FORMAT_DEFAULT = 0;   // Standard index.json.gz + files/ bundle (Default)
  FORMAT_HTML_SPA = 1;  // Single-file SPA HTML bundle
}

message RpcPublishingCreateRequest {
  string space_id = 1;
  string object_id = 2;
  string uri = 3;
  bool join_space = 4;
  PublishFormat format = 5; // Optional export format flag
}
```

### 2. Single-File SPA Generator (`core/publish/html_spa_exporter.go`)
Create a component in `anytype-heart` that reads the unencrypted object snapshots exported by `exportService.Export(..., IncludeNested: true)` and constructs `index.html`:

1. **Tree Traversal**: Parse the decrypted root page and sub-page snapshots generated in temporary export directory.
2. **Block-to-HTML Rendering**: Translate text, headers, lists, code blocks, and media attachments into standard HTML elements.
3. **Link Rewriting**: Convert internal `anytype://<spaceId>/<objectId>` links into web routes:
   ```html
   <a href="/sub-page" onclick="navigate(event, '/sub-page')">Sub Page Title</a>
   ```
4. **Embedded SPA Router Script**: Embed vanilla JavaScript inside `index.html`:
   ```javascript
   function router() {
       const path = window.location.pathname;
       document.querySelectorAll('.page').forEach(el => el.classList.remove('active'));
       let target = Array.from(document.querySelectorAll('.page')).find(el => el.getAttribute('data-path') === path);
       if (!target) target = document.querySelector('.page'); // Default fallback
       if (target) target.classList.add('active');
   }

   function navigate(event, path) {
       event.preventDefault();
       window.history.pushState({}, "", path);
       router();
   }

   window.addEventListener('popstate', router);
   window.addEventListener('DOMContentLoaded', router);
   ```

### 3. Pipeline Integration (`core/publish/service.go`)
In `publishToPublishServer`:
- Check `req.Format`.
- If `FORMAT_DEFAULT`: Run standard pipeline (`processExportedData` -> `createIndexFile` -> `publishToServer`).
- If `FORMAT_HTML_SPA`: Generate `index.html` via `SingleFileHtmlBuilder`, place alongside `files/` directory, and upload via `publishClientService.UploadDir`.

---

## Zero-Regression Strategy
- Default behavior (`FORMAT_DEFAULT = 0`) must remain strictly unchanged.
- All existing relation whitelists, snapshot generation, and publish client calls continue working as the default flow.

---

## Acceptance Criteria
- [ ] `RpcPublishingCreateRequest` supports `format = FORMAT_HTML_SPA`.
- [ ] Sub-pages exported with `IncludeNested: true` are transformed into route sections (`<div class="page">`) inside `index.html`.
- [ ] Internal object links are rewritten to trigger client-side `navigate()` router calls.
- [ ] Unit tests added for `SingleFileHtmlBuilder` rendering and block conversion.
- [ ] Existing `go test ./core/publish/...` tests pass without regression.
