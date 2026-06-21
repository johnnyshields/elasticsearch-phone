# Migration to Elasticsearch 9.4.2

This fork upgrades `elasticsearch-phone` from its original **Elasticsearch 1.6.0** baseline to
**Elasticsearch 9.4.2** (Lucene 10.4.0, JDK 21).

## Fork / branch survey

Before migrating, all upstream branches and every public fork were reviewed for reusable work:

| Source | State | Reusable? |
| --- | --- | --- |
| `purecloudlabs/elasticsearch-phone` `master` | ES 1.6.0, ES 1.x plugin API (`es-plugin.properties`, `CustomAnalysisBinderProcessor`) | base only |
| `purecloudlabs` `5.1.1-dev` | ES 5.1.1, **modern `AnalysisPlugin`/`TokenizerFactory` API** | **yes — used as the migration base** |
| `purecloudlabs` `dependabot/...-elasticsearch-7.17.23` | version bump only; source still ES 1.x → would not compile | no |
| `purecloudlabs` `AR-4917` | alternative refactor, still ES 1.x API | no |
| All 9 GitHub forks (incl. the most recent, `andyzhangdialpad`, 2023) | still on the ES 1.x API | no |

Conclusion: the only real modernization anywhere was the upstream `5.1.1-dev` branch, so this work
starts from it and carries it forward to ES 9.4.2.

## What changed

### Build (`pom.xml`)
- Elasticsearch `5.1.1` → `9.4.2`; Lucene `6.3.0` → `10.4.0`; Java `1.8` → `21`.
- `libphonenumber` `8.4.0` → `9.0.33`.
- Elasticsearch and Lucene are now `provided` scope (supplied by the node, not bundled).
- Removed the long-dead `inin`/`purecloud` Artifactory repositories, `distributionManagement`,
  and the GPG/nexus-staging/javadoc release plumbing.
- **Dropped `commons-lang3` and `commons-io`** — their few trivial uses were inlined, so the plugin
  now bundles only `libphonenumber`.

### Elasticsearch plugin API
- `AbstractTokenizerFactory` / `AbstractIndexAnalyzerProvider` constructors changed to `super(name)`
  (the four-arg factory/provider constructors are kept only so they can be used as
  `Factory::new` method references for `AnalysisModule.AnalysisProvider`).
- Removed `@Inject` from the analyzer providers.
- `plugin-descriptor.properties` modernized: dropped the obsolete `site` / `jvm` / `isolated`
  keys; `java.version` is now `21`.
- The plugin zip now lays the descriptor and jars at the archive root (not under `elasticsearch/`).

### Lucene
- Replaced the no-longer-published `org.apache.lucene.analysis.miscellaneous.UniqueTokenFilter`
  with a small vendored [`UniqueTokenFilter`](src/main/java/org/elasticsearch/index/analysis/UniqueTokenFilter.java)
  (global de-duplication), preserving the original analyzer behavior.

### Tests
- The old `ESIntegTestCase` integration tests (which used ES-1/5 APIs removed in ES 9 — mapping
  types, `AnalyzeResponse`, `preparePutMapping().setType()`) were replaced with Lucene
  `BaseTokenStreamTestCase` unit tests that exercise the analyzers directly. Every original case is
  preserved (`PhoneAnalyzerTest`, `EmailAnalyzerTest`, `PhoneSearchAnalyzerTest`), plus focused
  tests for the vendored filter and the term extractor.
- The search behavior previously asserted via a running cluster is reproduced at the analysis layer:
  a value matches a query iff every `phone-search` token is among the indexed tokens (mirroring a
  `match` query with the AND operator).

## Verification

- `mvn clean package` — 39 unit tests pass; builds the plugin zip.
- End-to-end smoke test against a real Elasticsearch 9.4.2 Docker node: the plugin installs and
  loads, `_analyze` produces the expected tokens for phone/SIP/email inputs, and `match` queries
  behave correctly.
