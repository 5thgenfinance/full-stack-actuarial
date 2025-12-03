# YAML Format and Best Practices User Guide

## Introduction

YAML (YAML Ain't Markup Language) is a human-readable data serialization format commonly used for configuration files, data exchange, and defining parameters for applications including Large Language Models (LLMs). This guide covers YAML syntax fundamentals, best practices, and specific instructions for configuring LLM environment variables.

---

## Part 1: YAML Fundamentals

### 1.1 Basic Structure

A YAML file begins with three dashes (`---`) and optionally ends with three periods (`...`):

```yaml
---
# Your YAML content here
key: value
...
```

### 1.2 Core Data Types

YAML supports the following primitive data types:

#### Scalars (Single Values)

| Type | Example | Notes |
|------|---------|-------|
| String | `name: "John Doe"` | Quotes optional for simple strings |
| Integer | `count: 42` | No quotes needed |
| Float | `price: 19.99` | Decimal numbers |
| Boolean | `enabled: true` | Use lowercase `true`/`false` |
| Null | `value:` | Empty value or explicit `null` |

#### Strings

```yaml
# Plain scalar (unquoted)
name: Hello World

# Double-quoted (allows escape sequences)
message: "Line 1\nLine 2"

# Single-quoted (literal, no escape processing)
path: 'C:\Users\Name'
```

#### Booleans

**Best Practice:** Use only lowercase `true` and `false` for YAML 1.2 compatibility:

```yaml
# Good
is_active: true
is_disabled: false

# Bad (deprecated in YAML 1.2)
is_active: yes
is_disabled: no
is_enabled: on
```

#### Numbers

```yaml
integer: 42
negative: -17
float: 3.14159
scientific: 1.0e+5
```

### 1.3 Mappings (Key-Value Pairs)

Mappings are unordered collections of key-value pairs:

```yaml
person:
  name: "Alice"
  age: 30
  email: "alice@example.com"
```

### 1.4 Sequences (Lists/Arrays)

Sequences are ordered collections denoted by hyphens:

```yaml
# Block style (recommended)
fruits:
  - apple
  - banana
  - orange

# Flow style (JSON-like)
colors: [red, green, blue]
```

### 1.5 Nested Structures

Combine mappings and sequences for complex data:

```yaml
company:
  name: "Tech Corp"
  employees:
    - name: "Alice"
      role: "Developer"
      skills:
        - Python
        - YAML
    - name: "Bob"
      role: "Designer"
      skills:
        - Figma
        - CSS
```

---

## Part 2: Indentation Rules

### 2.1 Core Principles

1. **Use spaces only** - Tabs are forbidden
2. **Recommended:** 2 spaces per indentation level
3. **Consistency:** Maintain the same indentation within each level

```yaml
# Good (2 spaces)
parent:
  child:
    grandchild: value

# Bad (tabs or inconsistent spacing)
parent:
	child:    # Tab character - INVALID
    grandchild: value
```

### 2.2 Sequence Indentation

The YAML creators recommend "zero-indented" sequences:

```yaml
# Recommended style
mapping:
  key: value
sequence:
- item1
- item2

# Alternative style (also valid)
sequence:
  - item1
  - item2
```

### 2.3 EditorConfig Setup

Add this to your `.editorconfig` file:

```ini
[*.{yaml,yml}]
indent_size = 2
indent_style = space
```

---

## Part 3: Multi-line Strings

### 3.1 Literal Style (`|`)

Preserves newlines exactly as written:

```yaml
script: |
  #!/bin/bash
  echo "Hello"
  echo "World"
```

**Result:** `#!/bin/bash\necho "Hello"\necho "World"\n`

### 3.2 Folded Style (`>`)

Folds newlines into spaces (for long paragraphs):

```yaml
description: >
  This is a long description
  that spans multiple lines.
  It will be folded into a single line.
```

**Result:** `This is a long description that spans multiple lines. It will be folded into a single line.\n`

### 3.3 Chomping Indicators

Control trailing newlines:

| Indicator | Behavior |
|-----------|----------|
| `|` or `>` | Single trailing newline (default) |
| `|-` or `>-` | No trailing newline (strip) |
| `|+` or `>+` | Keep all trailing newlines (keep) |

```yaml
stripped: |-
  No trailing newline here
kept: |+
  Keep all trailing newlines


clipped: |
  One trailing newline
```

---

## Part 4: Anchors and Aliases

Reduce repetition by reusing values:

### 4.1 Defining Anchors (`&`)

```yaml
defaults: &default_settings
  timeout: 30
  retries: 3
  logging: true
```

### 4.2 Using Aliases (`*`)

```yaml
production:
  <<: *default_settings
  environment: production

staging:
  <<: *default_settings
  environment: staging
  retries: 5  # Override default
```

### 4.3 Complete Example

```yaml
# Define common environment variables
common_env: &common
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: 100

development:
  <<: *common
  DEBUG: true
  DATABASE_URL: "localhost:5432"

production:
  <<: *common
  DEBUG: false
  DATABASE_URL: "prod-db:5432"
```

---

## Part 5: Special Characters and Escaping

### 5.1 Characters Requiring Quotes

Use quotes when your string contains:

- Colons followed by space (`: `)
- Leading special characters (`@`, `&`, `*`, `!`, `|`, `>`, etc.)
- Hash symbols (` #` - comment indicator)
- Commas in flow style
- Brackets or braces (`[]`, `{}`)

```yaml
# Requires quotes
url: "https://example.com:8080/path"
message: "Note: This needs quotes"
comment: "Value with # symbol"

# No quotes needed
simple: Hello World
path: /usr/local/bin
```

### 5.2 Escape Sequences (Double Quotes Only)

```yaml
# Common escape sequences
newline: "Line1\nLine2"
tab: "Col1\tCol2"
quote: "He said \"Hello\""
backslash: "C:\\Users\\Name"
unicode: "\u0041"  # Letter A
```

---

## Part 6: Environment Variables in YAML

### 6.1 Shell-Style References

```yaml
# Reference environment variables
database:
  host: ${DATABASE_HOST}
  port: ${DATABASE_PORT:-5432}  # Default value
  password: ${DB_PASSWORD}
```

### 6.2 Platform-Specific Syntax

**Docker Compose:**
```yaml
services:
  app:
    environment:
      - API_KEY=${API_KEY}
      - DEBUG=${DEBUG:-false}
```

**Kubernetes:**
```yaml
spec:
  containers:
    - name: app
      env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: api-key
```

**GitHub Actions:**
```yaml
env:
  MY_VAR: ${{ secrets.MY_SECRET }}
```

---

## Part 7: LLM Configuration Parameters

This section covers environment variables and parameters for configuring Large Language Models.

### 7.1 Temperature (`temperature`)

**Purpose:** Controls randomness/creativity in model outputs.

| Value Range | Effect |
|-------------|--------|
| 0.0 | Deterministic, always picks highest probability token |
| 0.1 - 0.3 | Focused, factual responses |
| 0.7 - 0.9 | Balanced creativity |
| 1.0+ | High creativity, more diverse/random outputs |

```yaml
llm_config:
  # Factual/deterministic tasks
  temperature: 0.2
  
  # Creative writing
  # temperature: 0.9
```

**Use Cases:**
- **Low (0.0-0.3):** Fact-based QA, summarization, code generation, translation
- **Medium (0.4-0.7):** General conversation, balanced tasks
- **High (0.8-1.0+):** Creative writing, brainstorming, poetry

### 7.2 Top-P / Nucleus Sampling (`top_p`)

**Purpose:** Controls cumulative probability threshold for token selection.

| Value | Effect |
|-------|--------|
| 0.1 | Only top 10% probability tokens considered |
| 0.9 | Top 90% probability tokens considered (more diverse) |
| 1.0 | All tokens considered |

```yaml
llm_config:
  # Conservative (focused)
  top_p: 0.9
  
  # More diverse
  # top_p: 0.95
```

**Best Practice:** Generally, modify either `temperature` OR `top_p`, not both simultaneously, as they can interact unpredictably.

### 7.3 Top-K Sampling (`top_k`)

**Purpose:** Limits selection to the K most likely tokens.

```yaml
llm_config:
  # Consider only top 50 tokens
  top_k: 50
  
  # Disabled (consider all tokens)
  # top_k: -1
```

| Value | Effect |
|-------|--------|
| 1 | Greedy decoding (always pick most likely) |
| 10-50 | Conservative, focused outputs |
| 100+ | More diverse possibilities |
| -1 or 0 | Disabled |

### 7.4 Min-P Sampling (`min_p`)

**Purpose:** Dynamic alternative to Top-K/Top-P. Filters tokens whose probability is below a threshold relative to the top token.

```yaml
llm_config:
  # Tokens must be at least 5% as likely as the top token
  min_p: 0.05
  
  # More restrictive
  # min_p: 0.1
```

**Advantage:** Adapts dynamically to context, maintaining diversity while ensuring coherence.

### 7.5 Maximum Tokens (`max_tokens`)

**Purpose:** Limits the maximum length of generated response.

```yaml
llm_config:
  # Short responses
  max_tokens: 256
  
  # Long-form content
  # max_tokens: 4096
```

**Note:** This limits output only. Total context = input tokens + output tokens, which must fit within the model's context window.

### 7.6 Seed (`seed`)

**Purpose:** Enables reproducible outputs by controlling randomness.

```yaml
llm_config:
  # Set for reproducibility
  seed: 42
  
  # Random (default)
  # seed: null
```

**Important:** 
- Same seed + same parameters + same prompt = mostly identical outputs
- Not 100% guaranteed due to backend infrastructure changes
- With `temperature: 0`, seed has minimal effect (greedy decoding)

### 7.7 Frequency Penalty (`frequency_penalty`)

**Purpose:** Penalizes tokens based on how often they've appeared in the output.

| Value Range | Effect |
|-------------|--------|
| -2.0 to 0.0 | Encourages repetition |
| 0.0 | No penalty (default) |
| 0.0 to 2.0 | Reduces repetition |

```yaml
llm_config:
  # Reduce repetition
  frequency_penalty: 0.5
  
  # Strong anti-repetition
  # frequency_penalty: 1.5
```

### 7.8 Presence Penalty (`presence_penalty`)

**Purpose:** Penalizes tokens that have appeared at all (binary check, not frequency-based).

```yaml
llm_config:
  # Encourage new topics/words
  presence_penalty: 0.6
  
  # Neutral
  # presence_penalty: 0.0
```

**Difference from Frequency Penalty:**
- Frequency penalty scales with repetition count
- Presence penalty is binary (appeared or not)

### 7.9 Repetition Penalty (`repetition_penalty`)

**Purpose:** Multiplicative penalty on repeated tokens (common in open-source models).

```yaml
llm_config:
  # Mild penalty
  repetition_penalty: 1.1
  
  # Strong penalty
  # repetition_penalty: 1.3
```

| Value | Effect |
|-------|--------|
| 1.0 | No penalty |
| 1.1 - 1.2 | Mild repetition reduction |
| 1.3 - 1.5 | Strong repetition reduction |

**Caution:** High values can cause unnatural phrasing as the model avoids necessary repetition.

### 7.10 Stop Sequences (`stop`)

**Purpose:** Strings that signal the model to stop generating.

```yaml
llm_config:
  stop:
    - "\n\n"
    - "User:"
    - "</output>"
    - "END"
```

**Use Cases:**
- Prevent chatbot from generating user messages
- Stop after structured output markers
- Control response length
- Cost management (fewer tokens)

### 7.11 Complete LLM Configuration Example

```yaml
---
# LLM Configuration File
# Environment: Production

model_settings:
  # Model identification
  model_name: "gpt-4"
  api_base: ${OPENAI_API_BASE:-https://api.openai.com/v1}
  api_key: ${OPENAI_API_KEY}

  # Sampling parameters
  temperature: 0.7
  top_p: 0.9
  top_k: -1  # Disabled
  min_p: 0.0
  
  # Output control
  max_tokens: 2048
  
  # Reproducibility
  seed: null  # Random

  # Repetition control
  frequency_penalty: 0.3
  presence_penalty: 0.2
  repetition_penalty: 1.0  # Disabled for OpenAI
  
  # Stop conditions
  stop:
    - "Human:"
    - "User:"
    - "</response>"

# Task-specific presets
presets:
  factual_qa: &factual
    temperature: 0.1
    top_p: 0.9
    max_tokens: 512
    frequency_penalty: 0.0
    presence_penalty: 0.0
    
  creative_writing: &creative
    temperature: 0.9
    top_p: 0.95
    max_tokens: 4096
    frequency_penalty: 0.5
    presence_penalty: 0.5
    
  code_generation: &code
    temperature: 0.2
    top_p: 0.95
    max_tokens: 2048
    frequency_penalty: 0.0
    presence_penalty: 0.0
    stop:
      - "```"
      - "# End of code"

# Use presets with aliases
tasks:
  summarization:
    <<: *factual
    max_tokens: 256
    
  story_generation:
    <<: *creative
    
  python_coding:
    <<: *code
```

---

## Part 8: Best Practices Summary

### General YAML

1. **Use 2-space indentation** - Industry standard, improves readability
2. **Avoid tabs** - Strictly use spaces only
3. **Use lowercase booleans** - `true`/`false` for YAML 1.2 compatibility
4. **Quote when necessary** - Strings with special characters
5. **Add comments** - Improve documentation with `#`
6. **Validate your YAML** - Use linters like `yamllint`
7. **Use `.yaml` extension** - Preferred over `.yml`
8. **Keep files modular** - Split large configs into multiple files
9. **Use anchors for DRY** - Don't repeat yourself

### LLM Parameters

1. **Start conservative** - Begin with low temperature, adjust as needed
2. **Don't mix Temperature and Top-P** - Modify one at a time
3. **Use seeds for testing** - Enable reproducibility during development
4. **Set appropriate max_tokens** - Balance quality vs. cost
5. **Test stop sequences** - Ensure they don't cut off valid content
6. **Document your choices** - Comment why specific values were chosen

---

## Part 9: Quick Reference Card

### YAML Syntax

```yaml
# Comment
key: value                    # Mapping
list:                         # Sequence
  - item1
  - item2
nested:                       # Nesting
  child: value
literal: |                    # Preserve newlines
  Line 1
  Line 2
folded: >                     # Fold to spaces
  Long text
  continues here
anchor: &name value           # Define anchor
alias: *name                  # Use anchor
merge:                        # Merge anchor
  <<: *name
  extra: value
```

### LLM Parameter Defaults

| Parameter | Typical Default | Range |
|-----------|----------------|-------|
| temperature | 1.0 | 0.0 - 2.0 |
| top_p | 1.0 | 0.0 - 1.0 |
| top_k | -1 (disabled) | -1 to ∞ |
| max_tokens | Model-dependent | 1 to context limit |
| seed | null (random) | Any integer |
| frequency_penalty | 0.0 | -2.0 to 2.0 |
| presence_penalty | 0.0 | -2.0 to 2.0 |
| repetition_penalty | 1.0 | 1.0 - 2.0 |

---

## Appendix: Recommended Tools

- **yamllint** - YAML linter and validator
- **yq** - Command-line YAML processor
- **VS Code YAML Extension** - Syntax highlighting and validation
- **EditorConfig** - Consistent formatting across editors
- **PyYAML / ruamel.yaml** - Python YAML libraries
- **js-yaml** - JavaScript YAML library

---

*Last Updated: December 2025*
