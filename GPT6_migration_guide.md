# Azure Environment Migration Guide for Chatbot and Utilities API

This document summarizes the model-routing and Azure environment-variable changes made during the GPT-6 migration.

The main goals of the migration were:

- move from model-specific naming to role-based naming;
- use a single Azure Responses endpoint for modern OpenAI models;
- keep HECA/photo-annotation specialist routing explicit;
- keep Astra-specific payload behavior separate from the generic HECA/annotation role;
- make Azure deployment names authoritative and pass them through unchanged;
- preserve an easy rollback path to the MAIN model.

---

## 0. Utilities API client-facing environment-variable migration

The Utilities API previously used a single fast-model environment variable:

```text
OPENAI_MODEL_NAME_GPT35=gpt-4.1-mini-csai
```

Replace it with:

```text
OPENAI_MODEL_NAME_FAST=gpt-6-luna
```

Environment-variable migration:

| Old environment variable | New environment variable / status |
|---|---|
| `OPENAI_MODEL_NAME_GPT35` | `OPENAI_MODEL_NAME_FAST` |
| `AZURE_OPENAI_RESPONSES_ADDRESS` | **Unchanged** |
| `AZURE_OPENAI_ADDRESS_AUDIO` | **Unchanged** |
| `USE_MANAGED_IDENTITY` | **Unchanged** |
| `IS_AZURE` | **Unchanged** |

The Utilities API does **not** require `OPENAI_MODEL_NAME_MAIN` or `OPENAI_MODEL_NAME_HECA_ANNOTATION`.

Current Utilities API model configuration:

```env
OPENAI_MODEL_NAME_FAST=gpt-6-luna
```

The remaining Azure AI, storage, search, SharePoint, container, and application settings are unchanged by this model-role migration.

## 1. Chatbot client-facing environment-variable migration

| Old env var | New env var / status | Meaning |
|---|---|---|
| `OPENAI_MODEL_NAME_GPT4` | `OPENAI_MODEL_NAME_MAIN` | Primary/general-purpose deployment |
| `OPENAI_MODEL_NAME_GPT35` | `OPENAI_MODEL_NAME_FAST` | Fast/inexpensive internal deployment |
| `OPENAI_MODEL_NAME_THINKING` | **Removed** | No separate "thinking" model role anymore; use MAIN where appropriate |
| `OPENAI_MODEL_NAME_ASTRA` | `OPENAI_MODEL_NAME_HECA_ANNOTATION` | HECA/photo-annotation specialist deployment |
| `USE_ASTRA_ONE_SHOT_ANNOTATION` | `USE_HECA_ANNOTATION_SPECIALIST` | Enables/disables specialist routing |
| `AZURE_OPENAI_ADDRESS_GPT4` | **Removed** | Replaced by unified Responses endpoint |
| `AZURE_OPENAI_ADDRESS_GPT35` | **Removed** | Replaced by unified Responses endpoint |
| `AZURE_OPENAI_ADDRESS_THINKING` | **Removed** | Replaced by unified Responses endpoint |
| `OWN_AZURE_OPENAI_ADDRESS_GPT4` | **Removed** | No longer needed |
| `OWN_AZURE_OPENAI_ADDRESS_GPT35` | **Removed** | No longer needed |
| `AZURE_OPENAI_KEY_GPT4` | `AZURE_OPENAI_KEY` | Single Azure AI resource key, only when not using Managed Identity |
| `AZURE_OPENAI_KEY_GPT35` | `AZURE_OPENAI_KEY` | Same single resource key |
| `AZURE_OPENAI_RESPONSES_ADDRESS` | **Unchanged** | Unified Azure Responses endpoint |
| `AZURE_OPENAI_ADDRESS_AUDIO` | **Unchanged** | Audio/transcription remains a separate endpoint |
| `USE_MANAGED_IDENTITY` | **Unchanged** | Azure authentication mode |

The intended client configuration is now conceptually:

```text
OPENAI_MODEL_NAME_MAIN=<actual Azure deployment name for main model>
OPENAI_MODEL_NAME_FAST=<actual Azure deployment name for fast model>
OPENAI_MODEL_NAME_HECA_ANNOTATION=<actual Azure specialist deployment name>

USE_HECA_ANNOTATION_SPECIALIST=TRUE

AZURE_OPENAI_RESPONSES_ADDRESS=https://.../openai/responses?api-version=...
AZURE_OPENAI_ADDRESS_AUDIO=https://.../audio/transcriptions?api-version=...

USE_MANAGED_IDENTITY=TRUE
```

If Managed Identity is disabled:

```text
AZURE_OPENAI_KEY=<single Azure AI resource key>
```

instead of separate GPT4/GPT35 keys.

---

## 2. Azure deployment-name naming convention
Azure custom deployment names must begin with the canonical GPT family/model
identifier used by the application.

Examples:
```
gpt-5.4-prod
gpt-5.6-sol-prod
gpt-6-sol-prod
gpt-6-luna-prod
gpt-6-astra-prod
```

For Astra specifically, the deployment name must contain both `gpt-6` and `astra`.

Do not use names such as:

```
prod-main
fast-prod
heca-prod
prod-gpt-6-sol
```
Because application logic currently inspects the model/deployment string to determine API family and certain model-specific capabilities.

## 3. R internal-name migration

| Old R name | New R name / status |
|---|---|
| `openai_model_name_gpt4` | `openai_model_name_main` |
| `openai_model_name_gpt35` | `openai_model_name_fast` |
| `openai_model_name_astra` | `openai_model_name_heca_annotation` |
| `use_astra_one_shot_annotation` | `use_heca_annotation_specialist` |
| `use_astra_for_this_call` | `use_specialist_for_this_call` |
| `my_model_thinking` | **Retired**; use `openai_model_name_main` at the relevant call site |
| `my_model_better_but_slow` | **Retired**; substantive route is MAIN |
| `my_model_fast_short_history` | **Retired**; fast route is FAST |
| `my_model_fast_long_history` | **Retired**; fast route is FAST |
| `my_model_function_calling` | **Retired**; initial mode/function/free-chat route is MAIN |
| `my_model_function_calling_for_suggestions` | **Retired**; suggestions use FAST |
| `my_model_vision` | **Retired as a generic role**; vision routing is now task-specific |

The old generic vision role should not be globally replaced with a single model variable.

Current behavior is task-specific:

```text
HECA / photo annotation
    specialist enabled  -> HECA_ANNOTATION
    specialist disabled -> MAIN

simple image relevance classifier
    -> FAST
```

---

## 4. Python internal-name migration

| Old Python name | New name / status |
|---|---|
| `openai_model_name_gpt4` | `openai_model_name_main` |
| `openai_model_name_gpt35` | `openai_model_name_fast` |
| `openai_model_name_thinking` | **Removed** |
| `openai_model_name_astra` | `openai_model_name_heca_annotation` if present |
| `azure_openai_key_gpt4` | `azure_openai_key` |
| `azure_openai_key_gpt35` | `azure_openai_key` |
| `azure_openai_address_gpt4` | **Removed** |
| `azure_openai_address_gpt35` | **Removed** |
| `azure_openai_address_thinking` | **Removed** |
| `azure_responses_API_address` | **Retained** |
| `azure_completions_API_addresses` | Compatibility-only; no longer sourced from per-model endpoint env vars |
| `is_astra` | **Retained intentionally** as a model-capability check, not a routing-role check |

Important distinction:

```python
my_model == openai_model_name_heca_annotation
```

means:

> this is the configured HECA/annotation specialist.

Whereas:

```python
is_astra = (
    'gpt-6' in my_model.lower()
    and 'astra' in my_model.lower()
)
```

means:

> this underlying deployment is specifically Astra and can receive Astra-specific payload behavior such as `detail='original'`.

These are intentionally separate concepts.

---

## 5. Model-role mapping after the migration

| Role/task | Configured model |
|---|---|
| Main/general-purpose | `OPENAI_MODEL_NAME_MAIN` |
| Fast internal tasks | `OPENAI_MODEL_NAME_FAST` |
| HECA specialist | `OPENAI_MODEL_NAME_HECA_ANNOTATION` when enabled |
| Photo annotation specialist | `OPENAI_MODEL_NAME_HECA_ANNOTATION` when enabled |
| HECA fallback | MAIN |
| Photo annotation fallback | MAIN |

Current intended deployments:

```text
MAIN             -> GPT-6 Sol
FAST             -> GPT-6 Luna
HECA_ANNOTATION  -> GPT-6 Astra
```

The application code is now role-based, so the underlying models can change later without renaming application variables again.

---

## 6. Specialist switch semantics

The specialist switch is intentionally explicit.

```text
USE_HECA_ANNOTATION_SPECIALIST=FALSE
```

means:

> intentionally do not use the specialist; HECA/photo annotation falls back to MAIN.

Whereas:

```text
USE_HECA_ANNOTATION_SPECIALIST=TRUE
OPENAI_MODEL_NAME_HECA_ANNOTATION=
```

is a configuration error and startup should fail.

MAIN and FAST are always required.

This preserves the distinction between:

- an intentional rollback; and
- an accidentally missing specialist deployment variable.

---

## 7. Azure routing architecture change

Previously, deployment information could effectively exist in two places:

```text
OPENAI_MODEL_NAME_GPT4=...
AZURE_OPENAI_ADDRESS_GPT4=https://.../deployments/<deployment>/...
```

and Python had logic to reconcile the two.

The new architecture has one source of truth for the deployment/model identifier:

```text
OPENAI_MODEL_NAME_*=actual model/deployment identifier to send
```

plus one shared transport endpoint:

```text
AZURE_OPENAI_RESPONSES_ADDRESS=https://.../openai/responses...
```

The routing flow is now:

```text
R chooses role
    ↓
chosen_model
    ↓
Python receives my_model
    ↓
get_azure_deployment_name(my_model)
    ↓
same exact string
    ↓
Responses API payload "model"
```

There is no longer any need to:

- extract deployment names from model-specific endpoint URLs;
- infer the deployment name from a legacy URL;
- strip model date suffixes;
- normalize custom Azure deployment names.

The configured Azure deployment name is authoritative and must be sent unchanged.

---

## 8. `get_azure_deployment_name()` behavior

The helper is retained as a central routing abstraction, but it now effectively passes the configured model/deployment name through unchanged.

Conceptually:

```python
def get_azure_deployment_name(my_model, py_logs=None):
    return my_model
```

The helper may still retain logging/documentation, but it should not rewrite or normalize the deployment string.

This matters because Azure deployment names may be custom.

For example:

```text
gpt-6-sol-prod-2026-09-15
```

must remain exactly:

```text
gpt-6-sol-prod-2026-09-15
```

and must not be shortened automatically.

---

## 9. Responses/API-family detection

R-side Responses detection now recognizes GPT-5 and GPT-6 families, with specialist routing explicitly treated as Responses-compatible.

Conceptually:

```r
is_responses_model <-
  isTRUE(use_specialist_for_this_call) ||
  grepl(
    "^gpt-(5|6)",
    chosen_model,
    ignore.case = TRUE
  )
```

Python reasoning-family checks recognize:

```text
gpt-5
gpt-6
o1
o3
```

For robustness, Python should normalize the model name first:

```python
model_name_lower = my_model.lower()
```

and use that normalized value for family detection.

---

## 10. GPT-5.4 coordinate compatibility

This remains intentionally model-version-specific.

The old specialist-disable check:

```r
!isTRUE(use_astra_one_shot_annotation)
```

became:

```r
!isTRUE(use_heca_annotation_specialist)
```

while the GPT-5.4 check remains:

```r
grepl(
  "5\\.4",
  openai_model_name_main,
  ignore.case = TRUE
)
```

Meaning:

```text
specialist enabled
    -> modern absolute-pixel coordinate format

specialist disabled + MAIN contains 5.4
    -> legacy GPT-5.4 coordinate format

specialist disabled + newer MAIN
    -> modern absolute-pixel coordinate format
```

The `"5\\.4"` check remains deliberately version-specific because the coordinate-format behavior genuinely depends on the underlying model version.

---

## 11. Astra-specific image behavior

Astra image handling remains deliberately model-specific.

Conceptually:

```python
model_name_lower = my_model.lower()

is_astra = (
    'gpt-6' in model_name_lower
    and 'astra' in model_name_lower
)
```

Then:

```python
if is_astra:
    new_part['detail'] = 'original'
```

This should not be replaced with:

```python
if my_model == openai_model_name_heca_annotation:
```

because `OPENAI_MODEL_NAME_HECA_ANNOTATION` may later point to a specialist model that does not support `detail='original'`.

The HECA/annotation role and Astra-specific API capability therefore remain separate.

---

## 12. Reasoning / verbosity behavior

Current behavior:

| Mode | Model role | Reasoning | Verbosity |
|---|---|---:|---:|
| HECA, specialist ON | specialist | medium | medium |
| Photo annotation, specialist ON | specialist | medium | low |
| HECA, specialist OFF | MAIN | medium | low |
| Photo annotation, specialist OFF | MAIN | medium | low |
| Prediction | MAIN | medium | low |
| Other final route in that routing block | MAIN | low | low |

Older comments saying that Astra used `"high"` reasoning were updated because the actual production setting is now `"medium"` for latency reasons.

---

# Client migration cheat sheet

## Replace

```text
OPENAI_MODEL_NAME_GPT4
    -> OPENAI_MODEL_NAME_MAIN

OPENAI_MODEL_NAME_GPT35
    -> OPENAI_MODEL_NAME_FAST

OPENAI_MODEL_NAME_ASTRA
    -> OPENAI_MODEL_NAME_HECA_ANNOTATION

USE_ASTRA_ONE_SHOT_ANNOTATION
    -> USE_HECA_ANNOTATION_SPECIALIST
```

## Remove

```text
OPENAI_MODEL_NAME_THINKING

AZURE_OPENAI_ADDRESS_GPT4
AZURE_OPENAI_ADDRESS_GPT35
AZURE_OPENAI_ADDRESS_THINKING

OWN_AZURE_OPENAI_ADDRESS_GPT4
OWN_AZURE_OPENAI_ADDRESS_GPT35
```

## If not using Managed Identity

Replace:

```text
AZURE_OPENAI_KEY_GPT4
AZURE_OPENAI_KEY_GPT35
```

with:

```text
AZURE_OPENAI_KEY
```

## Keep

```text
AZURE_OPENAI_RESPONSES_ADDRESS
AZURE_OPENAI_ADDRESS_AUDIO
USE_MANAGED_IDENTITY
```

## Required model-role variables

```text
OPENAI_MODEL_NAME_MAIN
OPENAI_MODEL_NAME_FAST
```

If:

```text
USE_HECA_ANNOTATION_SPECIALIST=TRUE
```

then this is also required:

```text
OPENAI_MODEL_NAME_HECA_ANNOTATION
```

## Astra naming convention

If the HECA/annotation deployment uses GPT-6 Astra, the Azure deployment name configured in:

```text
OPENAI_MODEL_NAME_HECA_ANNOTATION
```

must contain both:

```text
gpt-6
astra
```

Example:

```text
OPENAI_MODEL_NAME_HECA_ANNOTATION=gpt-6-astra-prod-v2
```

This naming convention is required so the application can safely detect Astra-specific image behavior.

---

# Recommended testing matrix

| Test | Expected result |
|---|---|
| Normal free chat | MAIN |
| Fast internal classifier/extractor | FAST |
| HECA + specialist ON | HECA_ANNOTATION |
| Photo annotation + specialist ON | HECA_ANNOTATION |
| Astra photo/HECA call | `detail="original"` applied to image input |
| HECA + specialist OFF | MAIN |
| Photo annotation + specialist OFF | MAIN |
| Specialist ON but specialist env missing | startup failure |
| MAIN missing | startup failure |
| FAST missing | startup failure |
| Azure Responses request | payload `model` exactly equals configured deployment name |
| Custom Azure deployment name | passed through unchanged |
| MAIN = GPT-5.4 + specialist OFF | legacy coordinate rules |
| MAIN = newer model + specialist OFF | absolute-pixel coordinate rules |
| Managed Identity ON | bearer-token auth; no model-specific Azure keys required |
| Managed Identity OFF | one `AZURE_OPENAI_KEY` |
| SaaS OpenAI | configured public model name passed unchanged |
| GPT-6 custom deployment capitalization | reasoning-family detection still works if code lowercases first |

---

# Repository-wide cleanup regex

After the deployment has been validated, perform one final repository-wide search for retired model-routing names.

```regex
\b(?:openai_model_name_thinking|openai_model_name_gpt4|openai_model_name_gpt35|openai_model_name_astra|OPENAI_MODEL_NAME_THINKING|OPENAI_MODEL_NAME_GPT4|OPENAI_MODEL_NAME_GPT35|OPENAI_MODEL_NAME_ASTRA|my_model_thinking|my_model_better_but_slow|my_model_fast_short_history|my_model_fast_long_history|my_model_function_calling|my_model_function_calling_for_suggestions|my_model_vision|use_astra_one_shot_annotation|USE_ASTRA_ONE_SHOT_ANNOTATION|use_astra_for_this_call|AZURE_OPENAI_ADDRESS_THINKING|AZURE_OPENAI_ADDRESS_GPT4|AZURE_OPENAI_ADDRESS_GPT35|OWN_AZURE_OPENAI_ADDRESS_GPT4|OWN_AZURE_OPENAI_ADDRESS_GPT35|azure_openai_address_thinking|azure_openai_address_gpt4|azure_openai_address_gpt35|AZURE_OPENAI_KEY_GPT4|AZURE_OPENAI_KEY_GPT35|azure_openai_key_gpt4|azure_openai_key_gpt35)\b
```

Once obsolete commented code has also been removed, this search should return **zero matches**.

Do not add generic `astra` or `gpt-6-astra` to this cleanup regex. Astra references are still intentionally used for model-specific capability detection, for example:

```python
is_astra = (
    'gpt-6' in model_name_lower
    and 'astra' in model_name_lower
)
```

That check controls Astra-specific API behavior such as:

```python
new_part['detail'] = 'original'
```

After the zero-match cleanup search, an optional second audit can be performed.

Enable case-insensitive search and use:

```regex
\b(?:gpt4|gpt35|thinking|astra)\b
```

This second search is for **manual inspection only**. Its result does not need to be zero because legitimate Astra-specific references should remain.
