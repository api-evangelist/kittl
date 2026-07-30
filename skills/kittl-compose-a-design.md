---
name: Compose a design on the Kittl canvas
description: >-
  Create an artboard and place text, shapes and uploaded images on it with the Kittl SDK,
  using the SdkResult envelope, atomic batches and correct scope declarations.
api: https://sdk-docs.kittl.dev/References/design
surface: sdk
scopes:
  - design:state:read
  - design:state:write
  - uploads:create
operations:
  - kittl.onReady
  - kittl.createBatch
  - kittl.design.board.createStandardBoard
  - kittl.design.board.cloneArtboard
  - kittl.design.board.updateArtboard
  - kittl.design.text.addText
  - kittl.design.text.updateText
  - kittl.design.shape.createPredefinedShape
  - kittl.design.shape.createBasicShape
  - kittl.design.image.addImage
  - kittl.upload.image.upload
  - kittl.design.object.getObject
  - kittl.design.object.getAllByFilter
  - kittl.design.object.removeObject
  - kittl.design.object.rotateObject
  - kittl.state.setSelectedObjectsIds
  - kittl.design.canvas.getExport
generated: '2026-07-19'
method: generated
source: https://sdk-docs.kittl.dev/References/design
---

# Compose a design on the Kittl canvas

## Preconditions

Declare in `manifest.json`:

```json
{ "config": { "scopes": ["design:state:read", "design:state:write", "uploads:create"] } }
```

## Rules that apply to every call

- **Wait for readiness.** Only call SDK APIs inside `kittl.onReady(...)`. `kittl.onReady`
  is itself exempt from scope checks.
- **Every call returns an `SdkResult`:** `{ isOk: true, result }` on success,
  `{ isOk: false, error }` on failure. Always check `isOk` before reading `result`.
  Most methods return a Promise (`SdkResultAsync`); some synchronous design methods return
  `SdkResult` directly — the shape is identical.
- **Narrow errors on `error.name`** — `SdkBadInputError` (validation) or
  `SdkInternalError` (host/SDK fault), both carrying `message` and an optional `cause`.
  Only treat `error.name` as a discriminator for APIs that document `SdkError`;
  `kittl.ai.spendCredits` returns a generic `Error`.
- **There is no idempotency contract.** Re-running a create call creates another object.
  Chain on returned IDs rather than retrying blind.
- Prefer the feature namespaces (`text`, `shape`, `board`) over generic `object` methods.

## Steps

1. **Create the artboard.**

   ```ts
   const boardResult = await kittl.design.board.createStandardBoard({
     title: 'Social post',
     position: { absolute: { left: 100, top: 100 } },
     size: { width: 1080, height: 1080 },
   });
   if (!boardResult.isOk) return;
   const board = boardResult.result;
   ```

2. **Place a shape relative to it.** Position accepts either
   `{ absolute: { left, top } }` or `{ relative: { to, location } }`, where `to` is
   another object's ID or `'viewport'`.

   ```ts
   await kittl.design.shape.createPredefinedShape({
     shapeType: 'rectangle',
     position: { relative: { to: board.id, location: 'center' } },
     size: { width: 920, height: 920, applyViewportScale: false },
     objectProperties: { fillColor: '#f4f4f4' },
   });
   ```

3. **Add text and select it.**

   ```ts
   const addResult = await kittl.design.text.addText({
     text: 'Launch campaign',
     position: { absolute: { left: 140, top: 120 } },
     size: { width: 360, height: 80 },
     textProperties: { fontSize: 48, fill: '#111111', textAlign: 'left' },
   });
   if (addResult.isOk) {
     await kittl.state.setSelectedObjectsIds([addResult.result.id]);
   }
   ```

4. **Upload an image, then place it.** `upload` returns an array; take `objectName` from
   the first entry and pass it as `src`.

   ```ts
   const uploadResult = await kittl.upload.image.upload({ blob });
   if (uploadResult.isOk && uploadResult.result.length > 0) {
     const { objectName } = uploadResult.result[0];
     await kittl.design.image.addImage({
       src: objectName,
       size: { width: 200, height: 200, applyViewportScale: false },
       position: { relative: { to: 'viewport', location: 'center' } },
     });
   }
   ```

5. **Batch multi-object composition.** When placing several objects at once, use a batch
   so they land atomically:

   ```ts
   const batch = await kittl.createBatch();
   await batch.shape.createPredefinedShape({ /* … */ });
   await batch.text.addText({ /* … */ });
   await batch.commit();
   ```

   A batch is **single-use** — calling `commit()` a second time throws.

6. **Read back or export.** `kittl.design.object.getObject({ id })` fetches one object and
   `kittl.design.object.getAllByFilter(predicate)` queries by predicate over
   `NormalizedObject`. Export the canvas with `kittl.design.canvas.getExport`, or grab a
   preview with `getPreviewImage` / `getScreenshot` — all three need `design:state:read`.
