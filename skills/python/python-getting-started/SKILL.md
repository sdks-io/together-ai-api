---
name: "python-getting-started"
description: "Together APIs Python SDK identity and lookup layer (Python only) — install, import root, base URL/environments, the auth pattern, the SDK map that ships at the SDK root (`sdk-map.md` + `map/operations/`) and how to traverse it, and the module table naming the one file owning each fact the map leaves to the source. Load this before answering any Together APIs Python SDK contract question or writing any SDK code."
---

# Getting started with the Together APIs Python SDK

> **Who this skill is for.** This is the **lookup layer** for anyone writing Together APIs Python SDK code — it is yours to follow directly and fully. Ground every contract fact here (in the SDK map, and in the source modules it names) rather than in recall, and carry those facts onto a contract sheet before you implement. Load `python-integrate-together-apis` for the workflow that wraps this skill.

This is the **SDK-specific** entry point. For general patterns that apply to any APIMatic-generated Python SDK (client construction, auth, calling endpoints, models, error handling, resilience, testing), see the companion API-agnostic skills: `python-client-initialization`, `python-authentication`, `python-calling-endpoints`, `python-models`, `python-error-handling`, `python-configuration-resilience` and `python-testing`.

**This page and those companion skills are complementary — load both.** This page is authoritative for the SDK's *identity and surface* (what to install, what to import, the controllers, which module owns which fact); the companion skills are the *usage layer* on top — the best-practice way to call each piece and the gotchas a signature cannot show. Reading a module in the installed package does not remove the need to load the skill for that step, so at each step below, load the companion *and* confirm names against the installed package.

## SDK identity

Verified against `together_apis/` and `pyproject.toml` of the generated package at version `2.0.0`. **Re-verify after a version bump** — this page is a snapshot, not a live read.

| Fact | Value |
| --- | --- |
| API | Together APIs |
| Distribution name (what you install) | `t-ai-apimatic-sdk` |
| Import root (what you import) | `together_apis` — not the string you install |
| Source repository | https://github.com/sdks-io/t-ai-apimatic-python-sdk |
| Source branch | `main` |
| Version | `2.0.0` |
| Sync client class | `TogetherApisClient` (alias `Client`) |
| Async client class | `AsyncTogetherApisClient` (alias `AsyncClient`) |
| Client construction | **keyword-only**: `environment` · `base_url` · `timeout` (default `30.0`) · `retry_options` · `bearer_auth`, and the transport override — `custom_http_client` on the sync client, `custom_async_http_client` on the async one (the names differ; see step 1) |
| Auth | **Bearer** token — set `bearer_auth` |
| Environments | 3 environments selected by `environment` (default `"production"`), overridable with `base_url` |
| Base-URL config | `ServerConfig` (`together_apis/server/server_config.py`), frozen, `extra="forbid"` |
| Python floor | **`>=3.10`** (classifiers list `3.10–3.14`) |
| Runtime dependencies | `httpx2 (>=2.13.1,<3.0.0)` · `certifi (>=2024.7.4)` · `pydantic[email] (>=2.11.0,<3.0.0)` · `typing-extensions (>=4.13.0,<5.0.0)` |
| Typing | ships `py.typed`; the package is checked under `mypy --strict` with `warn_unreachable`. Callers get full inference — **a type error against this SDK is a real contract violation, not noise** |
| Line length / lint | `ruff`, 120 cols (only relevant when editing the SDK itself) |
| Surface | 201 operations across 42 controllers · 567 models · 68 unions · 207 enums · 77 per-operation error unions |

The table above is **orientation, not a copy-paste recipe** — it gives you the names and facts (install, import roots, the auth *pattern*, the base-URL knob), while the actual integration code comes from the companion skills. Load each one as you reach its step (see **Integration workflow** below) and confirm its types against the installed package.

## Install

Into the same environment your project runs in, install the distribution from its package index:

```bash
pip install t-ai-apimatic-sdk
python -c "import together_apis, pathlib; print(pathlib.Path(together_apis.__file__).parent)"
```

**Run the second line too.** A `pip install` that reports success is not proof the SDK is importable where your project runs. If the probe raises `ModuleNotFoundError`, the install went into a different interpreter: re-run it through the one your project uses (`python -m pip install …`) and probe again. If it prints a path whose contents are not this SDK, what you installed is not this distribution.

An index name can belong to an unrelated project — install it from its own repository instead, at <https://github.com/sdks-io/t-ai-apimatic-python-sdk>:

```bash
pip install "t-ai-apimatic-sdk @ git+https://github.com/sdks-io/t-ai-apimatic-python-sdk.git@main"
```

The distribution carries its own `pyproject.toml`, so `pip` builds and installs it like any released package, copied into site-packages. Do not vendor its source into your project, add its directory to `sys.path`, or install it editable (`-e`) from a throwaway clone — an editable install points at that directory, so deleting the clone breaks every import.

Once the second line prints this SDK's directory, write the imports from the table above: the distribution name you install and the package name you import are not the same string. Requires Python 3.10 or newer.

## Imports — the package splits its surface across four modules

Python does not re-export child modules transitively, so `from together_apis import models` alone does **not** make enums, error unions, or runtime types reachable. Import each kind of type from the module that owns it.

`together_apis/__init__.py` exports exactly 6 names beside the `together_apis.models` subpackage it re-exports:

```python
from together_apis import (
    AsyncClient,
    AsyncTogetherApisClient,
    Client,
    Environment,
    ServerConfig,
    TogetherApisClient,
)
```

Everything else comes from its own subpackage, and the split matters because each kind of type a caller reaches for lives in a different module:

| What you need | Where it lives |
| --- | --- |
| Domain models, their `…Dict` companions | `together_apis.models` |
| Enums (and their open `…OrStr`/`…OrInt` aliases) | `together_apis.models.enums` |
| `ApiError` · `RawError` · `ApiResult` · `RequestOptions` · `HttpClient` · `SdkBaseModel` · `UNSET` · `Optional` · `FileInput` · `AsyncFileInput` · `NamedFile` · `BinaryContent` · `AsyncBinaryContent` · `BinaryReader` · `AsyncBinaryReader` · `FileResponse` · `AsyncFileResponse` | `together_apis.core` |
| Per-operation error *unions* | `together_apis.errors` (`AddAdapterErrorBody`, …) |

`together_apis.core` re-exports its whole public surface (a curated `__all__`), so import from `…core` rather than from the private modules beneath it (`…core.results`, `…core.exceptions`, `…core.auth.schemes`).

## Environments

The constructor takes an `environment` keyword — a **string literal alias**, not an enum — followed by `base_url`. The declared environments and the base URL each resolves to:

| `environment=` | Base URL | Hosting |
| --- | --- | --- |
| `"production"` *(default)* | `https://api.together.ai/v1` | Default environment for APIs |
| `"environment2"` | `https://api-inference.together.ai/v2` | Optimized environment for inference |
| `"environment3"` | `https://api.together.ai/v2` | — |

Consequences to state on every contract sheet that touches configuration:

- Omitting `environment` gives you **`"production"`**, silently — it is whatever the description listed first, not necessarily production.
- `base_url` overrides the environment's URL entirely when both are passed.
- `ServerConfig` is a frozen pydantic model with `extra="forbid"`: a misspelled keyword raises `ValidationError` at construction rather than being ignored.
- `timeout` is validated too — the base client raises `ValueError` for any non-positive value.

## Auth pattern (one scheme)

A bearer token, exposed as the client's `bearer_auth=` keyword taking a plain string.

```python
from together_apis import Client

client = Client(bearer_auth="<bearer_auth>")
```

**Every credentials keyword is optional at the type level and that is a trap worth flagging on every sheet.** Omit it and the client is built with `no_auth`: every request goes out unauthenticated. Nothing fails at construction. Most APIs then answer `401`; an API that serves anonymous traffic at all answers `200` and hides the omission entirely — verify the keyword is set rather than waiting for a `401` to tell you.

See `python-authentication` for the full picture, including what a *failed token fetch* raises — it is not what a caller expects, and it is the single most common surprise in this SDK.

## Controllers

Controllers and their operation counts (`client.<attr>`):

| Attribute | Class | Ops | Area |
| --- | --- | --- | --- |
| `client.abexperiment_service` | `AbexperimentService` / `AsyncAbexperimentService` | 5 | `a_b_experiment_service_create_ab_experiment` · `a_b_experiment_service_delete_ab_experiment` · `a_b_experiment_service_get_ab_experiment` · `a_b_experiment_service_list_ab_experiments` · `a_b_experiment_service_update_ab_experiment` |
| `client.account` | `Account` / `AsyncAccount` | 1 | `whoami` |
| `client.audio_api` | `AudioApi` / `AsyncAudioApi` | 5 | `audio_speech` · `audio_transcriptions` · `audio_translations` · `realtime_transcription` · `realtime_tts` |
| `client.batches` | `Batches` / `AsyncBatches` | 4 | `cancel_a_batch_job` · `create_a_batch_job` · `get_a_batch_job` · `list_batch_jobs` |
| `client.billing` | `Billing` / `AsyncBilling` | 1 | `get_billing_usage` |
| `client.chat` | `Chat` / `AsyncChat` | 1 | `chat_completions` |
| `client.code_interpreter` | `CodeInterpreter` / `AsyncCodeInterpreter` | 2 | `sessions_list` · `tci_execute` |
| `client.completion` | `Completion` / `AsyncCompletion` | 1 | `completions` |
| `client.config_service` | `ConfigService` / `AsyncConfigService` | 2 | `config_service_get_project_config` · `config_service_list_project_configs` |
| `client.deployment_service` | `DeploymentService` / `AsyncDeploymentService` | 10 | `deployment_service_add_adapter` · `deployment_service_create_deployment` · `deployment_service_delete_deployment` · `deployment_service_get_adapter` · `deployment_service_get_deployment` · `deployment_service_list_adapters` · … |
| `client.deployments` | `Deployments` / `AsyncDeployments` | 9 | `create_a_new_deployment` · `delete_a_deployment` · `get_a_deployment_by_id_or_name` · `get_logs_for_a_deployment` · `get_the_full_configuration_of_a_specific_revision` · `get_the_list_of_deployments` · … |
| `client.deployments_storage` | `DeploymentsStorage` / `AsyncDeploymentsStorage` | 2 | `download_a_file` · `get_a_file_download_url` |
| `client.deployments_volumes` | `DeploymentsVolumes` / `AsyncDeploymentsVolumes` | 2 | `create_a_new_volume` · `get_the_list_of_project_volumes` |
| `client.embeddings` | `Embeddings` / `AsyncEmbeddings` | 1 | `embeddings` |
| `client.endpoint_service` | `EndpointService` / `AsyncEndpointService` | 9 | `endpoint_service_create_endpoint` · `endpoint_service_delete_endpoint` · `endpoint_service_get_endpoint` · `endpoint_service_get_endpoint_analytics` · `endpoint_service_list_endpoint_events` · `endpoint_service_list_endpoints` · … |
| `client.endpoints` | `Endpoints` / `AsyncEndpoints` | 9 | `add_adapter` · `availability_zones` · `create_endpoint` · `delete_endpoint` · `get_endpoint` · `list_adapters` · … |
| `client.files` | `Files` / `AsyncFiles` | 5 | `delete_a_file` · `get_file_contents` · `list_all_files` · `retrieve_file_metadata` · `upload_a_file` |
| `client.fine_tuning` | `FineTuning` / `AsyncFineTuning` | 14 | `cancel_job` · `create_job` · `delete_a_fine_tune_job` · `download_model` · `download_tokenized_dataset` · `estimate_price` · … |
| `client.gpucluster_service` | `GpuclusterService` / `AsyncGpuclusterService` | 5 | `g_pu_cluster_service_create` · `g_pu_cluster_service_delete` · `g_pu_cluster_service_get` · `g_pu_cluster_service_list` · `g_pu_cluster_service_update` |
| `client.hardware` | `Hardware` / `AsyncHardware` | 1 | `list_hardware` |
| `client.health_check_service` | `HealthCheckService` / `AsyncHealthCheckService` | 1 | `health_check_service_list_passive_health_check_alerts` |
| `client.images` | `Images` / `AsyncImages` | 1 | `create_image` |
| `client.instance_cluster_add_on_service` | `InstanceClusterAddOnService` / `AsyncInstanceClusterAddOnService` | 5 | `instance_cluster_add_on_service_create` · `instance_cluster_add_on_service_delete` · `instance_cluster_add_on_service_get` · `instance_cluster_add_on_service_list` · `instance_cluster_add_on_service_update` |
| `client.instance_type_service` | `InstanceTypeService` / `AsyncInstanceTypeService` | 2 | `instance_type_service_get_inference_instance_type` · `instance_type_service_list_inference_instance_types` |
| `client.jobs` | `Jobs` / `AsyncJobs` | 2 | `get_job` · `list_jobs` |
| `client.models_api` | `ModelsApi` / `AsyncModelsApi` | 2 | `models` · `upload_model` |
| `client.placement_profile_service` | `PlacementProfileService` / `AsyncPlacementProfileService` | 2 | `placement_profile_service_get_placement_profile` · `placement_profile_service_list_placement_profiles` |
| `client.queue` | `Queue` / `AsyncQueue` | 5 | `cancel_queue_job` · `clear_queue` · `get_queue_job_status` · `get_queue_metrics` · `submit_queue_job` |
| `client.rl` | `Rl` / `AsyncRl` | 28 | `get_sample` · `optim_step` · `sample` · `weights_sync` · `create_inference_checkpoint` · `create_model_resources` · … |
| `client.region_service` | `RegionService` / `AsyncRegionService` | 1 | `region_service_list` |
| `client.remediation_service` | `RemediationService` / `AsyncRemediationService` | 6 | `remediation_service_approve_remediation` · `remediation_service_cancel_remediation` · `remediation_service_create_remediation` · `remediation_service_get_remediation` · `remediation_service_list_remediations` · `remediation_service_reject_remediation` |
| `client.rerank` | `Rerank` / `AsyncRerank` | 1 | `rerank` |
| `client.rollout_service` | `RolloutService` / `AsyncRolloutService` | 10 | `rollout_service_cancel_rollout` · `rollout_service_create_rollout` · `rollout_service_delete_rollout` · `rollout_service_get_rollout` · `rollout_service_list_rollouts` · `rollout_service_pause_rollout` · … |
| `client.secrets` | `Secrets` / `AsyncSecrets` | 5 | `create_a_new_secret` · `delete_a_secret` · `get_a_secret_by_id_or_name` · `get_the_list_of_project_secrets` · `update_a_secret` |
| `client.shadow_experiment_service` | `ShadowExperimentService` / `AsyncShadowExperimentService` | 10 | `shadow_experiment_service_create_shadow_experiment` · `shadow_experiment_service_create_target` · `shadow_experiment_service_delete_shadow_experiment` · `shadow_experiment_service_delete_target` · `shadow_experiment_service_get_shadow_experiment` · `shadow_experiment_service_get_target` · … |
| `client.shared_volume_service` | `SharedVolumeService` / `AsyncSharedVolumeService` | 5 | `shared_volume_service_create` · `shared_volume_service_delete` · `shared_volume_service_get` · `shared_volume_service_list` · `shared_volume_service_update` |
| `client.supported_models_service` | `SupportedModelsService` / `AsyncSupportedModelsService` | 2 | `supported_models_service_get_supported_model` · `supported_models_service_list_supported_models` |
| `client.video_api` | `VideoApi` / `AsyncVideoApi` | 2 | `create_video` · `retrieve_video` |
| `client.voices` | `Voices` / `AsyncVoices` | 1 | `fetch_voices` |
| `client.volume_service` | `VolumeService` / `AsyncVolumeService` | 13 | `volume_service_create_model` · `volume_service_create_remote_model_upload` · `volume_service_delete_model` · `volume_service_get_model` · `volume_service_get_remote_model_upload` · `volume_service_get_revision_by_model_type` · … |
| `client.volumes` | `Volumes` / `AsyncVolumes` | 3 | `delete_a_volume` · `get_a_volume_by_id_or_name` · `update_a_volume` |
| `client.evaluation` | `Evaluation` / `AsyncEvaluation` | 5 | `create_evaluation_job` · `get_all_evaluation_jobs` · `get_evaluation_job_details` · `get_evaluation_job_status_and_results` · `get_model_list` |

Every controller has an `Async…` peer whose operations are identical in name and parameters and differ solely by being awaited. Do not emit a separate row for an async operation; state the rule once on the sheet.

The SDK map's controller table carries the same counts and links to a page per controller (`map/operations/<controller>.md`) — go there for the operations themselves.

## SDK map — look up first, open the module second

The SDK ships a generated map at its **root** — the directory holding `pyproject.toml`, the `together_apis/` source directory, and these two entries:

- **`sdk-map.md`** — the index: client construction with the full constructor-keyword table, the error-handling model (`ApiError` / `ApiResult` / `RawError`, Case A vs Case B), where models, enums and error aliases live, servers and auth, and the link table into the operations pages.
- **`map/operations/<controller>.md`** — one page per controller, one `###` block per operation: the HTTP verb and route, the sync parsed signature, each parameter's role and wire name, both return types, the error alias with the status each arm maps from, and a **Type sources** table naming the module that declares every *generated* type the operation mentions.

**Installing the distribution does not give you the map.** `pyproject.toml` ships the `together_apis/` package only, so `sdk-map.md` and `map/operations/` are absent from the installed package — they live in the SDK's own source tree, at the root of its repository, <https://github.com/sdks-io/t-ai-apimatic-python-sdk>, in lockstep with the code the map describes — clone it to a temporary directory, outside the project repo:

```bash
git clone --depth 1 --branch main https://github.com/sdks-io/t-ai-apimatic-python-sdk
```

Confirm the map comes from the release you installed before you read anything out of it: the clone above is pinned to `main`, the branch this SDK is released from — the repository's default branch may carry a different version. A map read from any other release describes code you do not have.

Every `Source` path on the map is relative to that SDK root, so `together_apis/models/acceptance_tests_params.py` opens as written from there — and the same path resolves inside the installed package, which is where you read a module's body once the map has named it.

**The map is the locator; the source modules are the shapes.** Read the map first — signatures, routes, parameter roles, return types, error unions, and which module declares a type are all answered there without opening a single `.py` file. Then open the one module the map names for what it deliberately does not carry: a model's members, an enum's values, a field's wire alias. The map says so itself — *"Shapes live only in the source … Never grep for a type."*

**The map carries shapes; what an operation *means* lives elsewhere.** When *what* to pass depends on meaning — which values a field accepts beyond its type, a rule that couples two fields, what a defaulted parameter actually selects — the map will not settle it. Open the operation's docstring in `together_apis/apis/<controller>.py` (or `client.py` where operations sit on the client) and `api-reference.md` at the SDK root for that operation *before* writing the sheet row, and record what you found there. A value you already "know" for a field the map types as a plain `str` is a lookup, not a recall — the memory ban applies to it.

`sdk-map.md` carries the invariants every operation block assumes, so load it before any `map/operations/` page; the pages are written to be read beside it.

## Contract facts — the map first, then the module

**Six of these are map lookups — don't open the module for them:** an operation's signature, parameters and return type; the client's constructor keywords; the `timeout` default; the members of `ApiError` / `Success` / `Failure` / `RawError`; an operation's error union and the status each arm maps from; base-URL and auth wiring. The table below covers everything else, and the full body behind a map row.

Read the one module that owns the fact **inside the installed package**. Locate it first:

```bash
python -c "import together_apis, pathlib; print(pathlib.Path(together_apis.__file__).parent)"
```

Failing that, it is under the project's environment (`.venv/Lib/site-packages/together_apis` on Windows, `.venv/lib/python3.*/site-packages/together_apis` elsewhere). **If the package is not installed, install it first** — see *Install* above; until then mark the fact `UNVERIFIED` and say what would settle it rather than answering from memory. Paths below are relative to that package root:

| Question | Module |
| --- | --- |
| An operation's real signature, parameters and return type | `apis/<controller>.py` |
| Client construction, keywords, controller wiring | `client.py`, `async_client.py`, `base_client.py` |
| Timeout default and validation | `base_client.py` (`DEFAULT_TIMEOUT = 30.0`) |
| The request/response pipeline, 401 handling, 2xx-vs-error split | `core/raw_client.py` |
| Exception shape (`ApiError.error`, `.status_code`, `.headers`) | `core/exceptions.py` |
| `Success`/`Failure`/`RawError` | `core/results.py` |
| Per-call overrides | `core/request_options.py` |
| `UNSET`, `Optional`, `OptionalNullable` | `core/optionality.py` |
| Model base config, `to_dict`/`to_json` | `core/models.py` |
| A model's members, required vs `UNSET`, wire aliases | `models/<model_name>.py` |
| An enum's members and wire values | `models/enums/` |
| Open-enum coercion | `core/converters/open_enum.py` |
| Date/time wire formats — `Date`, `RFC3339DateTime`, `RFC1123DateTime`, `UnixSecondsDateTime` (`Annotated` aliases over `datetime.date` / `datetime.datetime`; not in the map's Type sources) | `core/converters/date_time.py` |
| What a file or binary parameter accepts, and the reader protocols behind it (the aliases are in the import table above; not in the map's Type sources) | `core/files.py` |
| Consuming a downloaded body — `FileResponse` / `AsyncFileResponse`, and what closes it (not in the map's Type sources) | `core/file_responses.py` |
| Transport protocols (the test seam) | `core/transport.py` |
| httpx2 adapter, proxy/TLS knobs | `core/httpx2_transport.py` |
| Token fetch, credential placement | `core/auth/`, `core/auth/models.py` |
| Base-URL resolution | `server/server_config.py`, `server/server.py` |
| An operation's error mapper (status → schema) | `errors/<operation>_error.py` |

**Read scoped.** These modules carry long design docstrings; `grep -n` for the symbol and read the surrounding lines rather than whole files. Never quote a docstring's design rationale onto a contract sheet — the sheet carries facts an implementer must obey, not the reasoning behind them.

Keep lookups cheap — the rules that keep a session's context small:

- Collect the contracts for **every** in-scope operation in **one** pass — signature, required members with wire aliases, the error union, enum values — into a short **contract sheet** in your plan or working notes, then implement from the sheet. Don't re-open a module per member, and never re-look-up a fact the sheet already carries.
- Recurse into a model's members only where the task actually sets them — a full transitive expansion is hundreds of rows and nobody needs it.
- Trust the interpreter over this page: if a name here ever fails to type-check or import, re-read the module the table above names and report the drift; never patch around it from memory.

## Integration workflow — load the companion skill at each step

Before you write the code for each step, load the named companion skill — even if you have already read the relevant module. Each step calls out the trap the signature hides (in *parens*). A typical integration reaches them in this order:

1. **Client construction & lifetime** — load **python-client-initialization** before you write `Client(...)` or `AsyncClient(...)`. (*The signature won't tell you:* the constructor is keyword-only, so nothing can be passed positionally; the client owns an `httpx2` connection pool and you **must** `close()` (sync) or `await aclose()` (async) or use it as a context manager; it must be long-lived and module- or app-scoped, never rebuilt per request; the sync and async clients do not mix; and the transport-override keyword differs by client — `custom_http_client` vs `custom_async_http_client`.)
2. **Authentication** — load **python-authentication** before you set credentials. The one scheme is `bearer_auth=`. (*The signature won't tell you:* every credentials keyword is *optional* — omit it and every request goes out unauthenticated, with no failure at construction and not necessarily a `401` to tell you; Load secrets from the environment or a secret store, never hardcode.)
3. **Calling an endpoint** — load **python-calling-endpoints** before the first `client.<controller>.<operation>(...)` call. (*The signature won't tell you:* every operation splits positional path params (and sometimes the body) from a keyword-only tail after `*`; every keyword-only parameter has a **real** default, so there is no "must pass `None` explicitly" hazard; **4 operations return `None`**, so `with_raw_response` is the only way to observe their status code; a multipart body is **one flat parameter list** — every field, JSON part and file is its own parameter, and an optional part is omitted with `None` rather than by a second call shape; and the two response modes — raising vs `ApiResult` — behave differently on failure.) (*For the file positions:* load **python-file-handling** before you pass anything but a `Path` — a file position accepts far more than `bytes`, **never a `str`**, and nothing validates what you hand it; an operation that returns a file hands back an **unread** `FileResponse` (`AsyncFileResponse` on the async client) that holds its connection until `save_to(...)`, `read()` or a drained `iter_bytes()` closes it, and a transport failure raises from that read rather than from the call.)
4. **Models** — load **python-models** the moment a request/response member is not a plain string or number. (*The signature won't tell you:* `Optional[T]` here is `T | UnsetType`, **not** `typing.Optional` — `None` is not a legal value for it; models are frozen pydantic instances with `…Dict` TypedDict companions; enums are **open** (`…OrStr`/`…OrInt`), so an unknown wire value passes through as a plain `str`/`int` rather than raising; wire aliases differ from Python member names; unknown response fields are **preserved**, not dropped; and serialize via `to_dict`/`to_json`.)
5. **Error handling** — load **python-error-handling** before you write any `try/except`. (*The signature won't tell you:* there is a single `ApiError` type whose `.error` is a **per-operation union**, and a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, and bypasses both response modes; `httpx2` transport exceptions reach your boundary unwrapped; and for an operation that returns a file the call succeeds as soon as the head arrives, so a transport failure raises from the read rather than the call — put the boundary around the consuming code.)
6. **Configuration & resilience** — load **python-configuration-resilience** when you set the base URL, timeouts, retries, proxies, TLS, or logging. (*The signature won't tell you:* **retrying is on by default** — `retry_options` takes a count, a `RetryOptions` or its dict, `max_retries` set to `0` turns it off, and one call overrides it through `request_options`; a `401` is never retried; `timeout` defaults to `30.0` and is a single float that maps onto `httpx2`'s timeout semantics, limiting each wait rather than the whole call; and there is no logging hook — you wrap the transport seam.)
7. **Testing** — load **python-testing** before you stub the SDK. (*The signature won't tell you:* the seam is the **transport protocol** (`HttpClient`/`AsyncHttpClient` in `core/transport.py`) passed as `custom_http_client` — not the client class; a fake transport implements `send` and `close`, the two the protocol declares, and `send` answers with the head alone; assert on the request the SDK actually built, and cover all four failure kinds, decode failures included.)

## What a contract sheet must carry for this SDK

Beyond the usual signatures and model members, a Python sheet is incomplete without these, because each one is a decision the implementer cannot make correctly from the signature alone:

1. **Sync or async** — which client class, and the reminder that the two do not mix. Plus the `close()`/`aclose()` obligation and where the client is held.
2. **The keyword-only boundary** for each operation: what is positional (path params, sometimes the body) and what sits after `*`. Every keyword-only parameter has a real default, so there is no "must pass `None` explicitly" hazard — say so, so nobody writes defensive `None`s.
3. **The file positions** — `files.upload_a_file`, each annotated `FileInput` on the sync client and `AsyncFileInput` on the async one, with `list[…]` for an array part. What those admit is wider than the annotation shows and **never a `str`**, and the arm you pick decides what stays resident, who closes your handle and what the file is called on the wire — so the sheet cites **python-file-handling** for the row rather than restating it.
4. **The 4 operations that return `None`** — `audio_api.realtime_transcription` · `audio_api.realtime_tts` · `deployments_storage.download_a_file` · `endpoints.delete_endpoint`. Their raw peers are `ApiResult[None, …]`, so `with_raw_response` is the only way to observe the status code.
5. **The 3 operations that return a file** — `audio_api.audio_speech` · `files.get_file_contents` · `fine_tuning.download_model`. The payload is `FileResponse` (`AsyncFileResponse` on the async client) with its body unread and its connection still held, so the sheet names the consuming call and where it is closed:
   1. `save_to(...)` · `read()` · a drained `iter_bytes()` — each consumes and closes. Getting it wrong leaks a connection rather than raising, so the row cites **python-file-handling** for the lifecycle: the async spellings, the single-use rule, and the warning an abandoned response emits.
   2. The raw peer observes the status and headers only — a `Success` carries no body of its own, because on a download the body *is* the payload.
6. **Required vs `UNSET`** for every model member the task sets, and the fact that `Optional[T]` here is `T | UnsetType` — **not** `typing.Optional`, so `None` is not a legal value for it.
7. **The `ApiError.error` union** for each operation in scope — there are **ten** typed error bodies in this SDK, so the union is never uniform. Every union is `<Typed> | RawError`:
   1. `ErrorData` — 42 operations (`account.whoami` · `audio_api.audio_speech` · `audio_api.audio_transcriptions` · `audio_api.audio_translations` · `billing.get_billing_usage` · `chat.chat_completions` · `completion.completions` · `embeddings.embeddings` · `endpoints.add_adapter` · `endpoints.create_endpoint` · `endpoints.delete_endpoint` · `endpoints.get_endpoint` · `endpoints.list_adapters` · `endpoints.list_endpoints` · `endpoints.remove_adapter` · `endpoints.update_endpoint` · `files.get_file_contents` · `files.upload_a_file` · `fine_tuning.create_job` · `fine_tuning.delete_a_fine_tune_job` · `fine_tuning.download_tokenized_dataset` · `fine_tuning.estimate_price` · `fine_tuning.preview_tokenized_data` · `hardware.list_hardware` · `models_api.models` · `rl.list_supported_models` · `rerank.rerank` · `shadow_experiment_service.shadow_experiment_service_create_shadow_experiment` · `shadow_experiment_service.shadow_experiment_service_create_target` · `shadow_experiment_service.shadow_experiment_service_delete_shadow_experiment` · `shadow_experiment_service.shadow_experiment_service_delete_target` · `shadow_experiment_service.shadow_experiment_service_get_shadow_experiment` · `shadow_experiment_service.shadow_experiment_service_get_target` · `shadow_experiment_service.shadow_experiment_service_list_shadow_experiments` · `shadow_experiment_service.shadow_experiment_service_list_targets` · `shadow_experiment_service.shadow_experiment_service_update_shadow_experiment` · `shadow_experiment_service.shadow_experiment_service_update_target` · `evaluation.create_evaluation_job` · `evaluation.get_all_evaluation_jobs` · `evaluation.get_evaluation_job_details` · `evaluation.get_evaluation_job_status_and_results` · `evaluation.get_model_list`); distinguishing members `error`
   2. `Any` — 19 operations (`deployments.create_a_new_deployment` · `deployments.delete_a_deployment` · `deployments.get_a_deployment_by_id_or_name` · `deployments.get_logs_for_a_deployment` · `deployments.get_the_full_configuration_of_a_specific_revision` · `deployments.get_the_list_of_deployments` · `deployments.list_revision_history_events_of_a_deployment` · `deployments.roll_back_a_deployment_to_a_previous_revision` · `deployments.update_a_deployment` · `deployments_volumes.create_a_new_volume` · `deployments_volumes.get_the_list_of_project_volumes` · `secrets.create_a_new_secret` · `secrets.delete_a_secret` · `secrets.get_a_secret_by_id_or_name` · `secrets.get_the_list_of_project_secrets` · `secrets.update_a_secret` · `volumes.delete_a_volume` · `volumes.get_a_volume_by_id_or_name` · `volumes.update_a_volume`); distinguishing members *none required*
   3. `QueueError` — 5 operations (`queue.cancel_queue_job` · `queue.clear_queue` · `queue.get_queue_job_status` · `queue.get_queue_metrics` · `queue.submit_queue_job`); distinguishing members *none required*
   4. `BatchErrorResponse` — 4 operations (`batches.cancel_a_batch_job` · `batches.create_a_batch_job` · `batches.get_a_batch_job` · `batches.list_batch_jobs`); distinguishing members *none required*
   5. `dict[str, str]` — 2 operations (`deployments_storage.download_a_file` · `deployments_storage.get_a_file_download_url`); distinguishing members *none required*
   6. `FineTunePreviewDetailError` — 1 operation (`fine_tuning.preview_tokenized_data`); distinguishing members `detail`
   7. `FineTunePreviewMessageError` — 1 operation (`fine_tuning.preview_tokenized_data`); distinguishing members `message`
   8. `FineTunesModelsLimits404Error1` — 1 operation (`fine_tuning.get_model_limits`); distinguishing members *none required*
   9. `FineTunesPreviewResponse` — 1 operation (`fine_tuning.preview_tokenized_data`); distinguishing members *none required*
   10. `QueueCancelResponse` — 1 operation (`queue.cancel_queue_job`); distinguishing members `status`
   11. *(none)* — `abexperiment_service.a_b_experiment_service_create_ab_experiment` · `abexperiment_service.a_b_experiment_service_delete_ab_experiment` · `abexperiment_service.a_b_experiment_service_get_ab_experiment` · `abexperiment_service.a_b_experiment_service_list_ab_experiments` · `abexperiment_service.a_b_experiment_service_update_ab_experiment` · `audio_api.realtime_transcription` · `audio_api.realtime_tts` · `code_interpreter.sessions_list` · `code_interpreter.tci_execute` · `config_service.config_service_get_project_config` · `config_service.config_service_list_project_configs` · `deployment_service.deployment_service_add_adapter` · `deployment_service.deployment_service_create_deployment` · `deployment_service.deployment_service_delete_deployment` · `deployment_service.deployment_service_get_adapter` · `deployment_service.deployment_service_get_deployment` · `deployment_service.deployment_service_list_adapters` · `deployment_service.deployment_service_list_deployments` · `deployment_service.deployment_service_remove_adapter` · `deployment_service.deployment_service_update_adapter` · `deployment_service.deployment_service_update_deployment` · `endpoint_service.endpoint_service_create_endpoint` · `endpoint_service.endpoint_service_delete_endpoint` · `endpoint_service.endpoint_service_get_endpoint` · `endpoint_service.endpoint_service_get_endpoint_analytics` · `endpoint_service.endpoint_service_list_endpoint_events` · `endpoint_service.endpoint_service_list_endpoints` · `endpoint_service.endpoint_service_list_organization_endpoints` · `endpoint_service.endpoint_service_list_public_endpoints` · `endpoint_service.endpoint_service_update_endpoint` · `endpoints.availability_zones` · `files.delete_a_file` · `files.list_all_files` · `files.retrieve_file_metadata` · `fine_tuning.cancel_job` · `fine_tuning.download_model` · `fine_tuning.get_fine_tunes_by_id_by_checkpoints` · `fine_tuning.get_fine_tunes_models_supported` · `fine_tuning.get_metrics` · `fine_tuning.list_all_jobs` · `fine_tuning.list_job` · `fine_tuning.list_job_events` · `gpucluster_service.g_pu_cluster_service_create` · `gpucluster_service.g_pu_cluster_service_delete` · `gpucluster_service.g_pu_cluster_service_get` · `gpucluster_service.g_pu_cluster_service_list` · `gpucluster_service.g_pu_cluster_service_update` · `health_check_service.health_check_service_list_passive_health_check_alerts` · `images.create_image` · `instance_cluster_add_on_service.instance_cluster_add_on_service_create` · `instance_cluster_add_on_service.instance_cluster_add_on_service_delete` · `instance_cluster_add_on_service.instance_cluster_add_on_service_get` · `instance_cluster_add_on_service.instance_cluster_add_on_service_list` · `instance_cluster_add_on_service.instance_cluster_add_on_service_update` · `instance_type_service.instance_type_service_get_inference_instance_type` · `instance_type_service.instance_type_service_list_inference_instance_types` · `jobs.get_job` · `jobs.list_jobs` · `models_api.upload_model` · `placement_profile_service.placement_profile_service_get_placement_profile` · `placement_profile_service.placement_profile_service_list_placement_profiles` · `rl.get_sample` · `rl.optim_step` · `rl.sample` · `rl.weights_sync` · `rl.create_inference_checkpoint` · `rl.create_model_resources` · `rl.create_training_checkpoint` · `rl.custom_forward_backward` · `rl.download_checkpoint` · `rl.estimate_model_resources_cost` · `rl.forward_backward` · `rl.get_checkpoint` · `rl.get_custom_forward_backward_operation` · `rl.get_forward_backward_operation` · `rl.get_inference_checkpoint_operation` · `rl.get_model_resources` · `rl.get_optim_step_operation` · `rl.get_training_checkpoint_operation` · `rl.get_training_session` · `rl.get_weights_sync_operation` · `rl.list_checkpoints` · `rl.list_model_resources` · `rl.list_training_sessions` · `rl.start_training_session` · `rl.stop_model_resources` · `rl.stop_training_session` · `rl.update_training_session` · `region_service.region_service_list` · `remediation_service.remediation_service_approve_remediation` · `remediation_service.remediation_service_cancel_remediation` · `remediation_service.remediation_service_create_remediation` · `remediation_service.remediation_service_get_remediation` · `remediation_service.remediation_service_list_remediations` · `remediation_service.remediation_service_reject_remediation` · `rollout_service.rollout_service_cancel_rollout` · `rollout_service.rollout_service_create_rollout` · `rollout_service.rollout_service_delete_rollout` · `rollout_service.rollout_service_get_rollout` · `rollout_service.rollout_service_list_rollouts` · `rollout_service.rollout_service_pause_rollout` · `rollout_service.rollout_service_preview_rollout_defaults` · `rollout_service.rollout_service_promote_rollout` · `rollout_service.rollout_service_resume_rollout` · `rollout_service.rollout_service_start_rollout` · `shared_volume_service.shared_volume_service_create` · `shared_volume_service.shared_volume_service_delete` · `shared_volume_service.shared_volume_service_get` · `shared_volume_service.shared_volume_service_list` · `shared_volume_service.shared_volume_service_update` · `supported_models_service.supported_models_service_get_supported_model` · `supported_models_service.supported_models_service_list_supported_models` · `video_api.create_video` · `video_api.retrieve_video` · `voices.fetch_voices` · `volume_service.volume_service_create_model` · `volume_service.volume_service_create_remote_model_upload` · `volume_service.volume_service_delete_model` · `volume_service.volume_service_get_model` · `volume_service.volume_service_get_remote_model_upload` · `volume_service.volume_service_get_revision_by_model_type` · `volume_service.volume_service_list_model_files` · `volume_service.volume_service_list_model_revisions` · `volume_service.volume_service_list_models` · `volume_service.volume_service_list_organization_models` · `volume_service.volume_service_list_remote_model_upload_events` · `volume_service.volume_service_list_remote_model_uploads` · `volume_service.volume_service_update_model`: no typed arm, so `.error` is always `RawError`
   12. So `isinstance(e.error, ErrorData)` matches only 42 of 201 operations.
   13. An arm that is not a class (`Any`, `list[...]`, `dict[...]`) is not an `isinstance` target: narrow the class arms first, then branch on `e.status_code` for what remains.
8. **That a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, in both response modes** — `core/raw_client.py` states this in `execute`'s own docstring. **And that the 2xx path declares at least one required member on 144 return types, so a truncated body fails to decode there and passes silently everywhere else.** Any sheet row for a call whose result is used must name the members the implementer has to assert on.
9. **That retrying is on by default** — `retry_options` on the client tunes it, `max_retries` set to `0` turns it off, and `request_options` overrides it per call — so the sheet says whether this integration keeps, tunes or disables it, and for which methods, since a write is repeated only when the policy names its method.
10. **What limits a whole call.** `timeout` limits each wait, not the call, so a reply whose pieces each arrive within it is never cut off. In async code, state the budget and the one place `asyncio.wait_for` enforces it for every call, and what the boundary returns for its `asyncio.TimeoutError`; in sync code, say there is none.
11. **Which environment the `environment` keyword selects**, because omitting it is silently `"production"` of the 3 declared.
12. A **REQUIRED READING** block naming the `python-*` companions that govern the steps, with `MUST load` pointers.

