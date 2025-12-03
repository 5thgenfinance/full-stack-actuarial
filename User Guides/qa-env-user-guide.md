# User Guide: Configuring a Test (QA) Environment with YAML

This guide explains how to configure a **QA test environment** for an LLM-enabled application using YAML. It focuses on:

- Environment selection (dev / qa / prod)
- Deterministic QA parameter profiles (temperature, top_p, seed, etc.)
- Enforcing a **clean context window** in QA
- Practical integration patterns for CI/CD and automated testing

---

## 1. High-Level Design

A robust YAML-based configuration for QA usually has three layers:

1. **Global runtime settings** – which environment is active.
2. **Per-environment blocks** – dev/qa/prod.
3. **LLM profiles** – detailed parameters for how the LLM behaves.

Conceptual layout:

```yaml
runtime:                 # 1. Global runtime settings
  active_env: qa

environments:            # 2. Per-environment blocks
  dev: {...}
  qa:  {...}
  prod:{...}

llm_profiles:            # 3. LLM parameter profiles
  qa_profile: {...}
  prod_profile: {...}
```

Your application reads `runtime.active_env`, selects the corresponding environment block, and then loads the relevant LLM profile.

---

## 2. Defining Environments in YAML

### 2.1 Environment Switch

Use an environment variable like `APP_ENV` to switch between `dev`, `qa`, and `prod`:

```yaml
# env-config.yaml

runtime:
  # Default to QA if APP_ENV is not set
  active_env: ${APP_ENV:-qa}

environments:
  dev:
    llm_profile: dev_profile
    base_url: "https://dev-api.example.com"

  qa:
    llm_profile: qa_profile
    base_url: "https://qa-api.example.com"

  prod:
    llm_profile: prod_profile
    base_url: "https://api.example.com"
```

In your code, you:

1. Resolve `runtime.active_env`.
2. Use it to pick `environments[active_env]`.
3. Read `llm_profile` from that environment and load that profile from `llm_profiles`.

---

## 3. QA LLM Profile: Deterministic & Test-Friendly

### 3.1 Goals for QA Mode

In QA/testing, the LLM should be:

- **Predictable** – stable outputs for the same inputs.
- **Bounded** – limited response length.
- **Isolated** – no hidden context from previous tests.
- **Observable** – configurable logging of prompts, outputs, and parameters.

### 3.2 Canonical QA Profile (YAML)

```yaml
# llm-profiles.yaml

llm_profiles:

  qa_profile: &qa_profile
    description: "Deterministic QA profile for automated testing"

    # Model selection (adjust to your provider)
    model: "gpt-4.1-mini"   # Small, cheaper model sufficient for tests

    # Sampling / randomness
    temperature: 0.1        # Very low for near-deterministic outputs
    top_p: 0.9              # Conservative nucleus sampling
    top_k: -1               # Disabled; rely on top_p
    min_p: 0.0

    # Output length control
    max_tokens: 512         # Bounded, test-friendly responses

    # Reproducibility
    seed: 42                # Fixed seed for regression test repeatability

    # Repetition behavior
    frequency_penalty: 0.0
    presence_penalty: 0.0
    repetition_penalty: 1.0

    # Stop conditions – prevent the model from continuing indefinitely
    stop:
      - "User:"
      - "Human:"
      - "</response>"

    # Context policy (logical layer – enforced by your app)
    context_policy:
      clear_history_before_call: true  # Always start from a clean context
      use_history: false               # Ignore stored chat history in QA
      max_history_messages: 0          # Disallow automatic history inclusion

    # Logging / observability hints (optional, app-defined)
    logging:
      log_prompts: true
      log_completions: true
      log_parameters: true
      redact_secrets: true

  # Example: production profile for comparison
  prod_profile: &prod_profile
    description: "Production profile for end users"
    model: "gpt-4.1"
    temperature: 0.7
    top_p: 0.95
    top_k: -1
    min_p: 0.0
    max_tokens: 2048
    seed: null              # Allow natural variation
    frequency_penalty: 0.3
    presence_penalty: 0.2
    repetition_penalty: 1.0
    stop:
      - "User:"
      - "Human:"
    context_policy:
      clear_history_before_call: false
      use_history: true
      max_history_messages: 10
    logging:
      log_prompts: true
      log_completions: true
      log_parameters: false
      redact_secrets: true
```

---

## 4. Enforcing QA Settings via YAML + Code

### 4.1 Wiring Environment to Profile

Combine `env-config.yaml` and `llm-profiles.yaml` logically:

```yaml
# combined-config.yaml (conceptual)

runtime:
  active_env: ${APP_ENV:-qa}

environments:
  dev:
    llm_profile: dev_profile
  qa:
    llm_profile: qa_profile
  prod:
    llm_profile: prod_profile

llm_profiles:
  qa_profile: *qa_profile        # included from llm-profiles.yaml
  prod_profile: *prod_profile
  # dev_profile: ...
```

In your application initialization:

1. Load the merged YAML configuration.
2. Determine `env_name = runtime.active_env`.
3. Look up `profile_name = environments[env_name].llm_profile`.
4. Resolve `params = llm_profiles[profile_name]`.
5. Apply `params` to every LLM call in QA.

### 4.2 Clean Context Window in QA

The **clean context window** behavior is driven by `context_policy` in the QA profile. Your code should:

- **Before each QA test call:**
  - If `context_policy.clear_history_before_call == true`, clear any in-memory or persisted chat history.
  - If `context_policy.use_history == false`, ensure you compose the request with only:
    - The system prompt, and
    - The current test input.

- **Example logical request shape (pseudo-YAML):**

```yaml
request:
  model: {{params.model}}
  messages:
    - role: system
      content: "You are a test-time QA assistant. Respond concisely and deterministically."
    - role: user
      content: "${TEST_PROMPT}"
  temperature: {{params.temperature}}
  top_p: {{params.top_p}}
  max_tokens: {{params.max_tokens}}
  seed: {{params.seed}}
  stop: {{params.stop}}
```

The key idea: **do not append prior messages** in QA mode, even if your production app normally does.

---

## 5. Pattern for CI/CD and Automated Tests

### 5.1 YAML for a Test Runner

For a test harness (e.g., a small service or script) controlled by YAML:

```yaml
# test-runner.yaml

runtime:
  active_env: qa

llm_test_runner:
  env_from: runtime.active_env
  cases_file: "./test-cases.yaml"
  fail_on_non_determinism: true

llm_profiles:
  qa_profile: &qa_profile
    # (same as before)

  prod_profile: &prod_profile
    # (same as before)
```

### 5.2 Test Cases YAML (Example)

```yaml
# test-cases.yaml

test_cases:
  - id: "math-001"
    description: "Simple arithmetic correctness"
    prompt: "What is 2 + 2?"
    expected_substring: "4"

  - id: "code-001"
    description: "Python function docstring"
    prompt: "Write a Python function to add two numbers."
    expected_substring: "def add("
```

Your test runner will:

1. Load `test-runner.yaml` and `llm_profiles.qa_profile`.
2. For each test case, send a **single-turn** prompt using the QA profile.
3. Assert that outputs match expected strings/substrings.

---

## 6. Recommended Defaults for QA

Use the following values as a strong starting point for QA mode:

- **temperature:** `0.0–0.2` (0.1 recommended)
- **top_p:** `0.8–0.95` (0.9 recommended)
- **top_k:** `-1` (disabled) unless your provider suggests otherwise
- **max_tokens:** `256–512` for most QA checks
- **seed:** fixed integer (e.g., `42`)
- **frequency_penalty:** `0.0`
- **presence_penalty:** `0.0`
- **repetition_penalty:** `1.0` unless using open-source models that require it
- **stop:** at least one boundary token (e.g., `"User:"`, `"END"`, or your app’s message delimiter)

These settings balance **determinism**, **cost control**, and **debuggability**.

---

## 7. Security & Operational Considerations

- **Do not hardcode secrets** in YAML – reference environment variables (`${OPENAI_API_KEY}`) instead.
- **Redact sensitive values** from logs when `logging.redact_secrets: true`.
- In QA, prefer **mocked or rate-limited** keys to avoid production quota impact.
- Keep QA and prod profiles side-by-side in the same repository to simplify diffing.

---

## 8. Summary Checklist

1. **Create environment switch:** `runtime.active_env` with `APP_ENV`.
2. **Define `qa_profile`** with low temperature, fixed seed, bounded `max_tokens`.
3. **Include `context_policy`** in QA profile to enforce clean context.
4. **Wire app logic** to:
   - Load env + profile from YAML.
   - Apply all QA parameters to LLM calls in QA mode.
   - Clear / ignore history whenever QA is active.
5. **Use YAML for test cases** and run them under QA profile in CI/CD.

With this structure, simply setting `APP_ENV=qa` will:

- Enforce deterministic LLM behavior.
- Guarantee a clean context window for each test.
- Provide a single source of truth for QA configuration in YAML.
