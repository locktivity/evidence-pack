# Control Mapping Example

A worked `evidencepack/control-mapping@v1` document ([control-mappings.json](control-mappings.json)) showing all three evidence anchors:

- a value inside a JSON artifact (RFC 6901 pointer),
- a quoted passage in a document (resolved against the document's extracted text),
- a table row in a spreadsheet document (resolved against its tabular extraction),
- plus a whole-artifact mapping with no anchor.

The mapping ships as an ordinary embedded artifact. Its manifest entry:

```json
{
  "type": "embedded",
  "path": "artifacts/control-mappings.json",
  "digest": "sha256:c71b2e0d4f8a9c6b3e5d7f1a0b9c8e2d4f6a8b0c1d3e5f7a9b1c3d5e7f9a0b2c",
  "size": 1830,
  "schema": "evidencepack/control-mapping@v1",
  "content_type": "application/json"
}
```

The entry is in the artifact index, so the mapping is covered by `pack_digest` and by any attestation over the manifest. Changing the mapping or a cited artifact after sealing fails pack verification.

Format: [spec/v1/schemas/control-mapping.md](../../spec/v1/schemas/control-mapping.md). Validator behavior: profiles companion spec, Sections 1.4 and 6.6.
