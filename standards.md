# Standards & API Reference

> Project: CI/CD Pipeline Optimizer · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

| Standard | URL | Description |
|----------|-----|-------------|
| **ISO 20022** | https://www.iso20022.org | Web service-based application programming interface standard for financial services; provides REST API best practices and design-first methodology for API development with structured data models applicable to CI/CD data interchange. |
| **ISO/TS 23029:2020** | https://www.iso.org/standard/74353.html | Web-service-based application programming interface (WAPI) in financial services; defines logical and technical layered approaches for API development including transformational rules and RESTful design principles relevant to pipeline API architecture. |
| **ISO 8601** | https://www.iso.org/iso-8601-date-and-time-format.html | Date and time representation standard; mandated for representing timestamps in API payloads (e.g., 2026-05-03T14:30:00Z), essential for build logs and CI metrics. |
| **ISO 4217** | https://www.iso.org/iso-4217-currency-codes.html | Currency code standard (three-letter codes); applicable to cost-aware scheduling and financial FinOps integration for CI runner pricing models. |

### W3C & IETF Standards

| Standard | URL | Description |
|----------|-----|-------------|
| **RFC 7231 (HTTP/1.1 Semantics and Content)** | https://tools.ietf.org/html/rfc7231 | IETF specification defining HTTP methods, status codes, and semantics; foundational for REST API design patterns used by all CI/CD platforms and pipeline APIs. |
| **RFC 8288 (Web Linking)** | https://tools.ietf.org/html/rfc8288 | Defines link relations and hypermedia navigation; applicable to pipeline API responses linking jobs, artifacts, and workflow steps. |
| **OpenAPI Specification v3.2.0** | https://spec.openapis.org/oas/v3.2.0.html | Industry standard for describing REST APIs with machine-readable contracts; essential for documenting CI/CD platform APIs and enabling client code generation. Supports JSON Schema draft 2020-12. |
| **JSON Schema 2020-12** | https://json-schema.org/draft/2020-12 | Standard for describing and validating JSON data structures; used by OpenAPI 3.1+ and critical for validating pipeline configuration files, build artifacts, and API payloads. |
| **GraphQL Specification** | https://spec.graphql.org | Query language and runtime for APIs; increasingly adopted for flexible CI/CD data queries and alternative to REST for complex pipeline analytics. |
| **Arazzo Workflow Specification** | https://spec.openapis.org/arazzo/v1.0.0 | Emerging W3C standard for describing multi-step API workflows and dependencies; applicable to pipeline orchestration and job scheduling logic. |

### Data Model & API Specifications

| Standard | URL | Description |
|----------|-----|-------------|
| **JSON Schema** | https://json-schema.org | Vocabulary for annotating and validating JSON; used extensively for validating pipeline YAML schemas, build outputs, and test result formats. |
| **Protocol Buffers (protobuf)** | https://developers.google.com/protocol-buffers | Google's serialization format; efficient alternative to JSON for high-throughput build metrics streaming and internal CI/CD tool communication. |
| **Apache Avro** | https://avro.apache.org | Schema-based serialization format; used for build event streaming and ensuring schema compatibility across distributed pipeline components. |

### Supply Chain & Build Security Standards

| Standard | URL | Description |
|----------|-----|-------------|
| **SLSA (Supply Chain Levels for Software Artifacts)** | https://slsa.dev | CNCF/Google framework defining levels 0–3 for build provenance, artifact integrity, and supply chain security; pipeline optimisers must preserve provenance attestation steps and SLSA-compliant signing. |
| **Sigstore & Cosign** | https://docs.sigstore.dev/cosign | CNCF standard for keyless artifact signing using identity-based certificates; enables secure signing in CI pipelines without managing long-lived keys. Cosign is primary CLI tool for container and binary signing. |
| **In-Toto Attestation Framework** | https://in-toto.io | CNCF specification for recording and verifying the integrity of software supply chain steps; used by Cosign for attestations and critical for compliance with SLSA requirements. |
| **CycloneDX (ECMA-424)** | https://cyclonedx.org | Full-stack Software Bill of Materials (SBOM) standard published as ECMA-424; provides vulnerability tracking with native VEX support. Pipeline optimisers must preserve SBOM generation steps. |
| **SPDX (Software Package Data Exchange)** | https://spdx.dev | Open standard for communicating SBOM information including provenance, licensing, and security; complementary to CycloneDX with broader licensing compliance focus. Both are approved under U.S. Executive Order 14028. |

### Testing & Documentation Standards

| Standard | URL | Description |
|----------|-----|-------------|
| **IEEE 829-2008 (ISO/IEC/IEEE 29119-3:2013)** | https://standards.ieee.org/ieee/829/3787 | Standard for software test documentation defining formats for test plans, cases, procedures, and reports; applicable to test result data ingest and analytics in pipeline optimisation tools. |
| **DORA Metrics (Four Key Metrics)** | https://dora.dev | Industry framework from Google's DevOps Research and Assessment program; defines Deployment Frequency, Lead Time for Changes, Change Failure Rate, and Mean Time to Recovery (MTTR) as benchmarks for high-performing engineering organisations. Essential for pipeline performance assessment. |

### Observability & Monitoring Standards

| Standard | URL | Description |
|----------|-----|-------------|
| **OpenTelemetry (OTLP Protocol)** | https://opentelemetry.io | Vendor-neutral standard for collecting, processing, and exporting observability data (metrics, traces, logs); emerging practice for emitting build and test spans as OTEL traces to unified observability backends (Datadog, Elastic, Google Cloud Trace, etc.). |
| **OpenMetrics** | https://openmetrics.io | Standard for exposing metrics in machine-readable format; enables Prometheus-compatible metrics scraping from CI/CD platforms for integration with monitoring stacks. |

---

## Similar Products — Developer Documentation & APIs

### Jenkins

- **Description:** Open-source automation server with Pipeline-as-Code support; widely adopted for on-premises CI/CD with extensive plugin ecosystem (2400+ plugins). Supports distributed builds across multiple agents.
- **API Documentation:** https://www.jenkins.io/doc
- **Pipeline Documentation:** https://www.jenkins.io/solutions/pipeline
- **SDKs/Libraries:** Jenkins API bindings available in Python, Java, Go, Ruby; Jenkins Workflow DSL (Groovy-based)
- **Developer Guide:** https://www.jenkins.io/doc/book/pipeline
- **Standards:** REST API, XML/JSON responses; Pipeline as Code (declarative and scripted DSL); OpenTelemetry plugin available for observability
- **Authentication:** Jenkins authentication token, OAuth 2.0 integration with GitHub/GitLab
- **Adoption:** 28% market share among CI/CD tools; strong in enterprises and open-source projects

### GitLab CI/CD

- **Description:** Native CI/CD platform tightly integrated with GitLab's version control and project management; supports distributed runners, container-based execution, and advanced caching.
- **API Documentation:** https://docs.gitlab.com/api
- **CI/CD Documentation:** https://docs.gitlab.com/ci
- **Pipelines API:** https://docs.gitlab.com/api/templates/gitlab_ci_ymls
- **Variables API:** https://docs.gitlab.com/api/project_level_variables
- **Development Guide:** https://docs.gitlab.com/development/cicd
- **SDKs/Libraries:** Python (python-gitlab), Ruby, Node.js client libraries; .gitlab-ci.yml YAML DSL
- **Standards:** REST API, YAML pipeline format, supports OpenTelemetry exports for pipeline observability
- **Authentication:** Personal Access Tokens, OAuth 2.0
- **Features:** Distributed runners, artifact caching, dependency proxies, Sigstore integration, SBOM generation

### GitHub Actions

- **Description:** Cloud-native CI/CD platform deeply integrated with GitHub; uses reusable Actions (composable workflow units) and matrix builds for parallelisation. Over 27,000 community Actions available.
- **API Documentation:** https://docs.github.com/en/rest/actions
- **Workflows API:** https://docs.github.com/en/rest/actions/workflows
- **Workflow Runs API:** https://docs.github.com/en/rest/actions/workflow-runs
- **Getting Started:** https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api
- **Quickstart:** https://docs.github.com/en/rest/quickstart
- **SDKs/Libraries:** Octokit (JavaScript/TypeScript, Python, Go); GitHub Action runners (Docker, Node.js, shell)
- **Standards:** REST API, YAML workflow syntax, OpenTelemetry GitHub Action available, native Sigstore support
- **Authentication:** Personal Access Tokens (PAT), GitHub App tokens, GITHUB_TOKEN context variable
- **Adoption:** Largest CI/CD user base globally; leading adoption for open-source projects

### CircleCI

- **Description:** Cloud-native CI/CD platform with Docker-based execution, intelligent caching, and performance optimisation; strong test analytics and flaky test detection capabilities.
- **API Documentation:** https://circleci.com/docs/api-developers-guide
- **API v2 Reference:** https://circleci.com/docs/api/v2/index.html
- **Developer Hub:** https://circleci.com/developer
- **API Access Guide:** https://support.circleci.com/hc/en-us/articles/115014005208-CircleCI-API-Access
- **SDKs/Libraries:** Python (CircleCI SDK), Node.js, Go; Postman collections for API exploration
- **Standards:** REST API v2 (v1.1 deprecated), JSON responses, supports Docker layer caching, OAuth 2.0, OpenTelemetry
- **Authentication:** API tokens with granular scopes; Circle-Token header
- **Rate Limiting:** 429 HTTP status with exponential backoff retry strategy
- **Features:** 40% performance improvement vs GitHub Actions (benchmarks 2025); Free tier: 6,000 credits/month

### Buildkite

- **Description:** Cloud-agnostic CI/CD platform with self-hosted agent model; used by Uber, DoorDash, Pinterest at scale for extreme parallelisation and cost efficiency.
- **API Documentation:** https://buildkite.com/docs/apis/rest-api
- **APIs Overview:** https://buildkite.com/docs/apis
- **Builds API:** https://buildkite.com/docs/apis/rest-api/builds
- **Pipelines API:** https://buildkite.com/docs/apis/rest-api/pipelines
- **Agents API:** https://buildkite.com/docs/apis/rest-api/agents
- **Agent REST API:** https://buildkite.com/docs/apis/agent-api
- **Token Management:** https://buildkite.com/docs/apis/managing-api-tokens
- **SDKs/Libraries:** REST API only (HTTP); client libraries in Python, Ruby, Node.js available via community
- **Standards:** REST API v2, JSON responses; API Differences: https://buildkite.com/docs/apis/api-differences
- **Authentication:** Bearer token authentication in Authorization header; granular scopes per token
- **Features:** Self-hosted agents with unlimited minutes; per-seat pricing; strong for on-premises and cost-sensitive organisations

### Harness CI/CD

- **Description:** Enterprise CI/CD platform with AI-driven test intelligence, flaky test detection, and build analytics; tightly integrated with Harness CD ecosystem for end-to-end software delivery.
- **API Documentation:** https://apidocs.harness.io
- **Developer Hub:** https://developer.harness.io/docs
- **Getting Started:** https://developer.harness.io/docs/continuous-integration
- **API Swagger Spec:** http://localhost:3000/swagger or raw YAML at http://localhost:3000/openapi.yaml
- **SDKs/Libraries:** OpenAPI/Swagger-based SDKs; Harness Terraform Provider; REST API for custom integrations
- **Standards:** REST API using JSON/YAML/form-data, OpenAPI spec available, supports Git Experience for pipeline-as-code
- **Authentication:** API tokens sent in Authorization header; Postman/curl compatible
- **Integration:** Terraform, REST API, Git Experience for automation and pipeline management
- **Features:** AI test intelligence unique to Harness; strong DORA metrics integration; complex enterprise contracts

### Launchable

- **Description:** AI-powered SaaS platform specialising in ML-based test selection and optimisation; reduces test execution time by running only the tests most likely to catch failures given the current changeset.
- **API Documentation:** https://docs.launchableinc.com (main docs portal)
- **Changelog:** https://changelog.launchableinc.com
- **Recent Features (2025):** Smart subset optimization targets; unhealthy tests export to CSV
- **SDKs/Libraries:** Python CLI, language-agnostic test session recording; integrates via CLI wrapper
- **Supported Test Frameworks:** JUnit, pytest, RSpec, Jest, Maven Surefire, Gradle Test (test-runner agnostic)
- **Standards:** REST API for programmatic integration; language/framework agnostic
- **Authentication:** API key-based authentication for test session recording
- **Integration:** Jenkins, GitHub Actions, CircleCI, GitLab CI; transparent CLI wrapper around existing test commands
- **Features:** ML model trained on historical test failures; flaky test detection; cold-start mitigation strategies

### Nx Cloud

- **Description:** Commercial SaaS platform for JavaScript/TypeScript monorepos providing distributed task execution (DTE), remote computation caching, and affected-command intelligence; best-in-class monorepo incremental CI.
- **API Documentation:** https://nx.dev/docs/reference/nx-cloud
- **Distributed Task Execution:** https://nx.dev/docs/features/ci-features/distribute-task-execution
- **Nx Agents Guide:** https://nx.dev/docs/concepts/more-concepts/illustrated-dte
- **Release Notes:** https://nx.dev/docs/reference/nx-cloud/release-notes
- **2026 Roadmap:** https://nx.dev/blog/nx-2026-roadmap
- **SDKs/Libraries:** Nx CLI (npm package); Node.js API; integrates with any CI system via CLI
- **Standards:** REST API, JSON configuration, supports OpenAPI-style task graph; Nx Console VS Code extension for local development
- **Authentication:** OAuth 2.0 with GitHub/GitLab; API tokens for CI integration
- **Features:** Automatic affected-command detection; auto-scaling ephemeral agents (Nx Agents); task-centric distribution with resource tracking (CPU/RAM per task, 2025+); free tier + enterprise plans

### Turborepo / Vercel Remote Cache

- **Description:** MIT-licensed open-source monorepo task runner for JavaScript/TypeScript with content-addressed task caching; Vercel Remote Cache provides free SaaS remote caching for all plans (2025).
- **Remote Caching Documentation:** https://vercel.com/docs/monorepos/remote-caching and https://turborepo.dev/docs/core-concepts/remote-caching
- **Caching Guide:** https://turborepo.dev/docs/crafting-your-repository/caching
- **Vercel Deployment Guide:** https://vercel.com/docs/monorepos/turborepo
- **Repository:** https://github.com/vercel/turborepo
- **SDKs/Libraries:** Single binary CLI; zero external dependencies for `turbo` tool
- **Standards:** REST-based Remote Cache API spec (HTTP server contract); JSON/YAML configuration
- **Authentication:** Environment variables for Remote Cache (TURBO_REMOTE_CACHE_SIGNATURE_KEY) in CI systems
- **Features:** Content-addressed caching; `--filter` for affected tasks; full turbo cache hit indicators (FULL TURBO); free Vercel Remote Cache; self-hostable cache option
- **Integration:** GitHub Actions, GitLab CI, CircleCI, Buildkite via standard CLI

---

## Notes

### Key Observations

1. **Cross-cutting OpenAPI & REST adoption:** All major CI/CD platforms expose REST APIs following OpenAPI conventions (or Swagger 2.0). GraphQL remains an emerging alternative, with some platforms offering it as a supplementary query interface.

2. **Supply chain security maturity:** SLSA, Sigstore/Cosign, and SBOM standards (CycloneDX and SPDX) are now table-stakes expectations. Regulatory momentum from EU CRA and US Executive Order 14028 means pipeline optimisers must preserve these steps and support secure signing without degrading pipeline performance.

3. **Observability convergence:** OpenTelemetry (OTLP) is emerging as the standard for CI/CD observability, with native support in GitLab CI (2025+), GitHub Actions, and Jenkins plugins. Platforms supporting OTEL traces enable integration with unified observability backends (Datadog, Elastic, Google Cloud Trace).

4. **Monorepo ecosystem fragmentation:** Nx Cloud and Turborepo dominate JavaScript/TypeScript monorepos with distinct trade-offs (Nx: more features/enterprise; Turborepo: simpler/free cache). Jenkins and CircleCI remain polyglot leaders. No tool unifies analysis across heterogeneous build systems without pre-instrumentation — a significant market gap.

5. **Test optimisation specialisation:** Launchable demonstrates clear ROI for ML-based test selection, but the market lacks tools combining test optimisation with cross-ecosystem pipeline analysis, cost-aware scheduling, and predictive failure triage.

6. **Pricing & licensing landscape:** Open-source (Jenkins, Turborepo) and freemium (GitHub Actions, CircleCI, Nx Cloud) models dominate. Enterprise tools (Develocity, Harness) command premium pricing but lack public rate cards — hindering budget planning for mid-market buyers.

7. **Emerging opportunities:** The 2025 empirical research (89.7% accuracy predicting build failures 1.6 stages ahead) remains uncommercialised. Natural-language pipeline authoring, flaky test root-cause attribution, and cost-aware scheduling are underserved areas where AI/LLM augmentation could provide differentiation.

### Standards Still Evolving

- **MCP (Model Context Protocol):** While specified by Anthropic, MCP is not yet a formal ISO/W3C standard but represents an emerging integration pattern for AI-agent tool use in CI/CD (relevant for AI-driven pipeline optimisation features).
- **DORA metrics standardisation:** Still emerging; while DORA metrics are de facto industry benchmarks, formal standardisation (ISO/IEEE) is pending.
- **AI/ML SBOM (AI/ML-BOM):** CycloneDX v1.7 (October 2025) adds support for AI/ML Bill of Materials; standardisation of AI component provenance in pipelines is nascent.
- **FinOps for CI:** Cost-aware scheduling lacks a formal standard; vendor-specific cost models dominate (AWS, GCP, Azure). Community-driven standardisation is emerging via FinOps Foundation.
