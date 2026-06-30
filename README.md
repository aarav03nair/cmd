# Monorepo Documentation — Generation Spec

Hand this whole file to Claude Code. It produces a self-contained `library-knowledge/` directory
documenting the full module (the whole monorepo) and every submodule inside it.

Codebase: Java + Spring Boot, multi-module Gradle.

Headings carry retrieval later, so keep every doc heading-rich and follow the templates exactly.

---

## Output directory layout

```
library-knowledge/
  00_INDEX.md                  <- full-module overview + submodule dependency graph
    00_INTERLINKAGES.md          <- how submodules interact, in prose
      00_GLOSSARY.md               <- terms + aliases mapped to submodules
        submodule-a/
            a-overview.md
              submodule-b/
                  b-overview.md
                    ...
                    ```

                    One folder + one overview doc per submodule, plus the three `00_*` full-module docs at the root.

                    ---

                    ## Per-submodule doc template (`<submodule>/<x>-overview.md`)

                    Fill every section from the actual code. Each `##` should read like a question a developer asks.

                    ```markdown
                    ---
                    submodule: <id>
                    path: <path/in/monorepo>
                    owner: <name>
                    last_updated: <YYYY-MM-DD>
                    ---

                    # <Submodule Name> — <one-line purpose>

                    ## Purpose
                    What this submodule exists to do, in 2–4 sentences. The problem it solves.

                    ## Role in the monorepo
                    Where it sits. What it depends on (upstream) and what depends on it (downstream).
                    One paragraph + the local slice of the dependency graph:

                    ```mermaid
                    graph LR
                      upstream-x --> THIS[<Submodule Name>]
                        THIS --> downstream-y
                        ```

                        ## Architecture
                        Internal design — main components/layers and how they interact. Include a diagram.

                        ```mermaid
                        graph TD
                          A[Entry point] --> B[Core component]
                            B --> C[Sub-component]
                            ```

                            ## Internal hierarchy
                            Package/folder structure and what each part holds. Short annotated tree.

                            ## Public API / contracts
                            For a Spring Boot library, cover:
                            - Published artifact coordinates (group:artifact:version).
                            - Auto-configuration classes (entries in META-INF/spring/...AutoConfiguration.imports).
                            - Beans this module exposes to consumers.
                            - @ConfigurationProperties keys / config knobs and their defaults.
                            - Key public classes/interfaces and signatures.
                            - What's exposed via the `api` Gradle config (real public surface; `implementation` is internal).

                            ## Key flows
                            Step-by-step walkthrough of the 1–3 most important execution paths.

                            ## Dependencies
                            - Internal: other submodules, split by `api project(':x')` (contract) vs `implementation project(':x')` (internal).
                            - External: libraries/BOMs, with why each is here and any versions pinned via the platform/BOM/version catalog.

                            ## Consumed by (reverse dependencies)
                            Which submodules and known external apps depend on THIS one, and on what specifically
                            (which beans/classes/config). Answers "if I change this, what breaks?" — list each consumer + the surface it relies on.

                            ## Implementing changes here
                            - Change recipes: for common edits, task → files/packages to touch → pattern to follow → how to verify.
                              e.g. "To add a new <thing>: implement interface `X` in package `p.q`, register via `...`, add a test in `XTest`, run `./gradlew :module:test`."
                              - Extension points: interfaces/SPIs to implement, auto-config hooks, beans to override, config props to add.
                              - Conventions: house patterns a change should match (naming, error handling, logging, config wiring).
                              - Where NOT to change / off-limits: contract or fragile code that looks editable.
                              - Patch-upgrade hotspots: BOM conflicts, version pins, breaking-change risk areas.

                              ## Testing changes
                              How this submodule is tested and how a dev validates a change locally
                              (key test classes, the Gradle task to run, fixtures/conventions).

                              ## Ownership & key decisions
                              Who owns/knows this submodule (who to ask) and the non-obvious design decisions + the reasoning behind them.

                              ## Gotchas / known issues
                              Non-obvious traps and footguns.
                              ```

                              ---

                              ## Full-module doc 1 — `00_INDEX.md`

                              ```markdown
                              # <Library Name> — Overview

                              One paragraph: what the whole library is, who uses it, why it matters.

                              ## Submodule map

                              | Submodule | Path | Purpose (one line) | Doc |
                              |-----------|------|--------------------|-----|
                              | Submodule A | path/a | ... | submodule-a/a-overview.md |
                              | Submodule B | path/b | ... | submodule-b/b-overview.md |

                              ## Dependency graph (between submodules)

                              ```mermaid
                              graph TD
                                A[Submodule A] --> B[Submodule B]
                                  A --> C[Submodule C]
                                    B --> C
                                    ```

                                    ## How to ask
                                    - "I need to do X — where does it go?" → that submodule's "Implementing changes here".
                                    - "How is X built / what does it do" → that submodule's overview.
                                    - "If I change A, what breaks?" → the graph + A's "Consumed by".
                                    - "How do A and B interact" → 00_INTERLINKAGES.md + both overviews.
                                    ```

                                    ---

                                    ## Full-module doc 2 — `00_INTERLINKAGES.md`

                                    State each interaction explicitly in prose — one short subsection per relationship.

                                    ```markdown
                                    # <Library Name> — Interlinkages

                                    How the submodules interact. Each entry: what flows between them, via what surface, and the
                                    blast radius of changing it.

                                    ## A → B
                                    Submodule A calls B's `<class/bean>` to do <what>. A depends on B's <contract/config>.
                                    If you change <X in B>, it affects A's <Y>. Tested together via <test>.

                                    ## A → C
                                    ...

                                    ## Shared contracts / cross-cutting concerns
                                    Beans, config properties, or base classes multiple submodules rely on, and who owns them.

                                    ## Change-impact quick reference
                                    | If you change... | ...these are affected | Why |
                                    |------------------|-----------------------|-----|
                                    | B's <thing>      | A, D                  | both consume B's <surface> |
                                    ```

                                    ---

                                    ## Full-module doc 3 — `00_GLOSSARY.md`

                                    ```markdown
                                    # <Library Name> — Glossary

                                    - **<Term / acronym>** — definition. Also called: <aliases>. Lives in: <submodule>.
                                    - **<Internal name>** — what it maps to in the code / which submodule.
                                    ```

                                    ---

                                    ## Instructions for Claude Code (paste at the top of your session)

                                    ```
                                    Goal: produce the library-knowledge/ documentation directory, following this spec exactly.
                                    Use Gradle as the source of truth — don't infer module structure.

                                    Step 1 — Recon + scaffold:
                                    - Read settings.gradle(.kts): the include(...) entries ARE the submodule list.
                                    - Build the inter-submodule graph from each build.gradle's project(':x') deps, distinguishing
                                      `api` (contract) from `implementation` (internal) edges. Build BOTH directions (what each
                                        module depends on AND what depends on it — needed for "Consumed by").
                                        - Run `./gradlew projects` and `./gradlew :<module>:dependencies` to confirm the real resolved
                                          tree and BOM/platform constraints. Cross-check against declared deps.
                                          - Identify each module's Spring auto-config (AutoConfiguration.imports), exposed beans, and
                                            @ConfigurationProperties. Note libraries vs runnable apps.
                                            - Generate 00_INDEX.md and create empty per-submodule files from the template.

                                            Step 2 — Per-submodule docs:
                                            - Fill each overview from the template, grounded in actual code, including "Consumed by".
                                            - Generate accurate Mermaid diagrams for architecture and local dependency slices.

                                            Step 3 — Full-module docs:
                                            - Write 00_INTERLINKAGES.md: one prose subsection per real interaction between submodules
                                              (what flows, via what surface, blast radius), plus the change-impact table.
                                              - Write 00_GLOSSARY.md: domain terms, acronyms, aliases mapped to submodules.

                                              Step 4 — Verify:
                                              - Re-read every doc against the code; fix drift; flag anything uncertain.
                                              - Cross-check 00_INTERLINKAGES.md against the Gradle edges — every dependency edge should have a
                                                described interaction, and vice versa.
                                                - Confirm heading structure (H1/H2/H3) matches the templates.
                                                - Output a single clean, self-contained library-knowledge/ directory.
                                                ```
                                                # Monorepo Documentation — Generation Spec

                                                Hand this whole file to Claude Code. It produces a self-contained `library-knowledge/` directory
                                                documenting the full module (the whole monorepo) and every submodule inside it.

                                                Codebase: Java + Spring Boot, multi-module Gradle.

                                                Headings carry retrieval later, so keep every doc heading-rich and follow the templates exactly.

                                                ---

                                                ## Output directory layout

                                                ```
                                                library-knowledge/
                                                  00_INDEX.md                  <- full-module overview + submodule dependency graph
                                                    00_INTERLINKAGES.md          <- how submodules interact, in prose
                                                      00_GLOSSARY.md               <- terms + aliases mapped to submodules
                                                        submodule-a/
                                                            a-overview.md
                                                              submodule-b/
                                                                  b-overview.md
                                                                    ...
                                                                    ```

                                                                    One folder + one overview doc per submodule, plus the three `00_*` full-module docs at the root.

                                                                    ---

                                                                    ## Per-submodule doc template (`<submodule>/<x>-overview.md`)

                                                                    Fill every section from the actual code. Each `##` should read like a question a developer asks.

                                                                    ```markdown
                                                                    ---
                                                                    submodule: <id>
                                                                    path: <path/in/monorepo>
                                                                    owner: <name>
                                                                    last_updated: <YYYY-MM-DD>
                                                                    ---

                                                                    # <Submodule Name> — <one-line purpose>

                                                                    ## Purpose
                                                                    What this submodule exists to do, in 2–4 sentences. The problem it solves.

                                                                    ## Role in the monorepo
                                                                    Where it sits. What it depends on (upstream) and what depends on it (downstream).
                                                                    One paragraph + the local slice of the dependency graph:

                                                                    ```mermaid
                                                                    graph LR
                                                                      upstream-x --> THIS[<Submodule Name>]
                                                                        THIS --> downstream-y
                                                                        ```

                                                                        ## Architecture
                                                                        Internal design — main components/layers and how they interact. Include a diagram.

                                                                        ```mermaid
                                                                        graph TD
                                                                          A[Entry point] --> B[Core component]
                                                                            B --> C[Sub-component]
                                                                            ```

                                                                            ## Internal hierarchy
                                                                            Package/folder structure and what each part holds. Short annotated tree.

                                                                            ## Public API / contracts
                                                                            For a Spring Boot library, cover:
                                                                            - Published artifact coordinates (group:artifact:version).
                                                                            - Auto-configuration classes (entries in META-INF/spring/...AutoConfiguration.imports).
                                                                            - Beans this module exposes to consumers.
                                                                            - @ConfigurationProperties keys / config knobs and their defaults.
                                                                            - Key public classes/interfaces and signatures.
                                                                            - What's exposed via the `api` Gradle config (real public surface; `implementation` is internal).

                                                                            ## Key flows
                                                                            Step-by-step walkthrough of the 1–3 most important execution paths.

                                                                            ## Dependencies
                                                                            - Internal: other submodules, split by `api project(':x')` (contract) vs `implementation project(':x')` (internal).
                                                                            - External: libraries/BOMs, with why each is here and any versions pinned via the platform/BOM/version catalog.

                                                                            ## Consumed by (reverse dependencies)
                                                                            Which submodules and known external apps depend on THIS one, and on what specifically
                                                                            (which beans/classes/config). Answers "if I change this, what breaks?" — list each consumer + the surface it relies on.

                                                                            ## Implementing changes here
                                                                            - Change recipes: for common edits, task → files/packages to touch → pattern to follow → how to verify.
                                                                              e.g. "To add a new <thing>: implement interface `X` in package `p.q`, register via `...`, add a test in `XTest`, run `./gradlew :module:test`."
                                                                              - Extension points: interfaces/SPIs to implement, auto-config hooks, beans to override, config props to add.
                                                                              - Conventions: house patterns a change should match (naming, error handling, logging, config wiring).
                                                                              - Where NOT to change / off-limits: contract or fragile code that looks editable.
                                                                              - Patch-upgrade hotspots: BOM conflicts, version pins, breaking-change risk areas.

                                                                              ## Testing changes
                                                                              How this submodule is tested and how a dev validates a change locally
                                                                              (key test classes, the Gradle task to run, fixtures/conventions).

                                                                              ## Ownership & key decisions
                                                                              Who owns/knows this submodule (who to ask) and the non-obvious design decisions + the reasoning behind them.

                                                                              ## Gotchas / known issues
                                                                              Non-obvious traps and footguns.
                                                                              ```

                                                                              ---

                                                                              ## Full-module doc 1 — `00_INDEX.md`

                                                                              ```markdown
                                                                              # <Library Name> — Overview

                                                                              One paragraph: what the whole library is, who uses it, why it matters.

                                                                              ## Submodule map

                                                                              | Submodule | Path | Purpose (one line) | Doc |
                                                                              |-----------|------|--------------------|-----|
                                                                              | Submodule A | path/a | ... | submodule-a/a-overview.md |
                                                                              | Submodule B | path/b | ... | submodule-b/b-overview.md |

                                                                              ## Dependency graph (between submodules)

                                                                              ```mermaid
                                                                              graph TD
                                                                                A[Submodule A] --> B[Submodule B]
                                                                                  A --> C[Submodule C]
                                                                                    B --> C
                                                                                    ```

                                                                                    ## How to ask
                                                                                    - "I need to do X — where does it go?" → that submodule's "Implementing changes here".
                                                                                    - "How is X built / what does it do" → that submodule's overview.
                                                                                    - "If I change A, what breaks?" → the graph + A's "Consumed by".
                                                                                    - "How do A and B interact" → 00_INTERLINKAGES.md + both overviews.
                                                                                    ```

                                                                                    ---

                                                                                    ## Full-module doc 2 — `00_INTERLINKAGES.md`

                                                                                    State each interaction explicitly in prose — one short subsection per relationship.

                                                                                    ```markdown
                                                                                    # <Library Name> — Interlinkages

                                                                                    How the submodules interact. Each entry: what flows between them, via what surface, and the
                                                                                    blast radius of changing it.

                                                                                    ## A → B
                                                                                    Submodule A calls B's `<class/bean>` to do <what>. A depends on B's <contract/config>.
                                                                                    If you change <X in B>, it affects A's <Y>. Tested together via <test>.

                                                                                    ## A → C
                                                                                    ...

                                                                                    ## Shared contracts / cross-cutting concerns
                                                                                    Beans, config properties, or base classes multiple submodules rely on, and who owns them.

                                                                                    ## Change-impact quick reference
                                                                                    | If you change... | ...these are affected | Why |
                                                                                    |------------------|-----------------------|-----|
                                                                                    | B's <thing>      | A, D                  | both consume B's <surface> |
                                                                                    ```

                                                                                    ---

                                                                                    ## Full-module doc 3 — `00_GLOSSARY.md`

                                                                                    ```markdown
                                                                                    # <Library Name> — Glossary

                                                                                    - **<Term / acronym>** — definition. Also called: <aliases>. Lives in: <submodule>.
                                                                                    - **<Internal name>** — what it maps to in the code / which submodule.
                                                                                    ```

                                                                                    ---

                                                                                    ## Instructions for Claude Code (paste at the top of your session)

                                                                                    ```
                                                                                    Goal: produce the library-knowledge/ documentation directory, following this spec exactly.
                                                                                    Use Gradle as the source of truth — don't infer module structure.

                                                                                    Step 1 — Recon + scaffold:
                                                                                    - Read settings.gradle(.kts): the include(...) entries ARE the submodule list.
                                                                                    - Build the inter-submodule graph from each build.gradle's project(':x') deps, distinguishing
                                                                                      `api` (contract) from `implementation` (internal) edges. Build BOTH directions (what each
                                                                                        module depends on AND what depends on it — needed for "Consumed by").
                                                                                        - Run `./gradlew projects` and `./gradlew :<module>:dependencies` to confirm the real resolved
                                                                                          tree and BOM/platform constraints. Cross-check against declared deps.
                                                                                          - Identify each module's Spring auto-config (AutoConfiguration.imports), exposed beans, and
                                                                                            @ConfigurationProperties. Note libraries vs runnable apps.
                                                                                            - Generate 00_INDEX.md and create empty per-submodule files from the template.

                                                                                            Step 2 — Per-submodule docs:
                                                                                            - Fill each overview from the template, grounded in actual code, including "Consumed by".
                                                                                            - Generate accurate Mermaid diagrams for architecture and local dependency slices.

                                                                                            Step 3 — Full-module docs:
                                                                                            - Write 00_INTERLINKAGES.md: one prose subsection per real interaction between submodules
                                                                                              (what flows, via what surface, blast radius), plus the change-impact table.
                                                                                              - Write 00_GLOSSARY.md: domain terms, acronyms, aliases mapped to submodules.

                                                                                              Step 4 — Verify:
                                                                                              - Re-read every doc against the code; fix drift; flag anything uncertain.
                                                                                              - Cross-check 00_INTERLINKAGES.md against the Gradle edges — every dependency edge should have a
                                                                                                described interaction, and vice versa.
                                                                                                - Confirm heading structure (H1/H2/H3) matches the templates.
                                                                                                - Output a single clean, self-contained library-knowledge/ directory.
                                                                                                ```
                                                                                                