# ant-configurations-and-extends

Ant+Ivy probe for Project #6 (P1) from the Ant coverage plan. Exercises two
catalog patterns in a single-module project: **configuration-mapping** (#6) and
**configuration-extends** (#7).

---

## Catalog patterns exercised

| Catalog ID | Pattern name | Feature under test |
|---|---|---|
| #6 | `configuration-mapping` | Varied `conf=` syntax in `<dependency>` elements: `compile->default`, `runtime->compile(*),master(*)`, `test->default`, `provided->default`, `*->@` |
| #7 | `configuration-extends` | Three-level extends chain: `test extends runtime extends compile`; `provided` and `master` isolated |

---

## Ivy configuration design

Five configurations declared in `ivy.xml`:

```
compile         base compile-time scope
runtime         extends compile   → sees compile deps + its own
test            extends runtime   → sees compile + runtime + its own deps
provided        isolated          → container scope, not inherited
master          isolated          → Ivy artifact-only convention
```

**Extends chain:** `test ──> runtime ──> compile`

### Conf-mapping syntax variety

| Dependency | Ivy configuration | Conf-mapping used | Description |
|---|---|---|---|
| `commons-lang3:3.12.0` | compile | `compile->default` | Standard Maven-style |
| `slf4j-api:1.7.36` | runtime | `runtime->compile(*),master(*)` | Multi-target with wildcard |
| `junit:4.13.2` | test | `test->default` | Standard default mapping |
| `servlet-api:2.5` | provided | `provided->default` | Container scope |
| `commons-io:2.11.0` | master | `*->@` | Identity mapping (same-named conf) |

`hamcrest-core:1.3` is junit's transitive dep; committed to `lib/` for Mode-3 scanning.

### Expected visibility in Mode-1 (aspirational)

| Configuration | Visible JARs |
|---|---|
| compile | commons-lang3 |
| runtime | slf4j-api + commons-lang3 (via extends) |
| test | junit + hamcrest-core + slf4j-api + commons-lang3 (via extends chain) |
| provided | servlet-api (isolated — NOT on test/runtime classpath) |
| master | commons-io artifact only (no transitive deps) |

---

## Mode-3 pivot (SCM scanner reality)

The bolt-4 SCM scanner does **not** invoke Mode-1 Ivy resolution. It falls
through to Mode-3 default path scan: SHA1 fingerprinting of binary files.

All 6 JARs are pre-resolved from Maven Central and committed to `lib/` so
the scanner has real artifacts to fingerprint:

| JAR | Ivy conf | Maven coordinate | SHA1 |
|---|---|---|---|
| `commons-lang3-3.12.0.jar` | compile | `org.apache.commons:commons-lang3:3.12.0` | `c6842c86792ff03b9f1d1fe2aab8dc23aa6c6f0e` |
| `slf4j-api-1.7.36.jar` | runtime | `org.slf4j:slf4j-api:1.7.36` | `6c62681a2f655b49963a5983b8b0950a6120ae14` |
| `junit-4.13.2.jar` | test | `junit:junit:4.13.2` | `8ac9e16d933b6fb43bc7f576336b8f4d7eb5ba12` |
| `hamcrest-core-1.3.jar` | test (transitive) | `org.hamcrest:hamcrest-core:1.3` | `42a25dc3219429f0e5d060061f71acb49bf010a0` |
| `servlet-api-2.5.jar` | provided | `javax.servlet:servlet-api:2.5` | `5959582d97d8b61f4d154ca9e495aafd16726e34` |
| `commons-io-2.11.0.jar` | master | `commons-io:commons-io:2.11.0` | `a2503f302b11ebde7ebc3df41daebe0e4eea3689` |

In Mode-3, all 6 JARs appear as **flat direct dependencies** — no inheritance
or scope isolation is visible from scan output. The `group` field in
`expected-tree.json` records the intended Ivy configuration for downstream
reference.

---

## Expected dependency tree

```
ant-configurations-and-extends (root)
  commons-lang3-3.12.0.jar   [compile]
  slf4j-api-1.7.36.jar       [runtime]
  junit-4.13.2.jar           [test]
  hamcrest-core-1.3.jar      [test / junit transitive]
  servlet-api-2.5.jar        [provided]
  commons-io-2.11.0.jar      [master]
```

All 6 are direct deps with no transitive children (Mode-3 flat scan).
SHA1 fingerprints are the primary regression signal in `autotest_config.json`.

---

## Mend config

**Bucket A** — default-emit with `java` pinned to `"17"` (Ant itself is not
pinnable via `install-tool`; the Ant binary version comes from whatever the
operator installs out-of-band). This is a partial reproducibility limitation:
the Ant version is not controlled by `.whitesource`. Downstream comparators
should treat Ant version as uncontrolled.

**Additional dimension: aspirational Mode-1 flags.** `whitesource.config`
carries `ant.resolveDependencies=true` and `ant.ivyResolveDependencies=true`.
These flags do not activate on the bolt-4 SCM scanner (they are UA-only), but
are included so a pipeline-mode `mend ua` run against the same repo would
attempt Mode-1 Ivy resolution.

No branch scoping, project-token routing, or SAST suppression needed for
this probe.

---

## Probe metadata

```
probe_id:          ant-configurations-and-extends-20260525-170030
pm:                ant
pm_version_tested: ivy-2.5.x (Ivy used by UA's in-process library)
patterns:          [configuration-mapping, configuration-extends]
catalog_ids:       [#6, #7]
priority:          P1
generated:         2026-05-25
bucket:            A (java pinned to 17)
target:            remote
scanner_mode:      mode-3-default-path-scan
```
