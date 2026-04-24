# Prompt Templating in LocalAI

LocalAI uses Go's `text/template` package to render prompts before sending them to backends. This allows models to have custom chat formats (ChatML, Llama2, Alpaca, etc.) defined in their config files.

## Overview

When a chat completion request comes in, LocalAI:
1. Looks up the model's config for a `template` section
2. Renders the template with the messages from the request
3. Passes the rendered string to the backend

## Template Configuration

In a model YAML config:

```yaml
name: my-model
backend: llama-cpp
parameters:
  model: my-model.gguf

template:
  chat: |
    {{- range .Messages}}
    {{- if eq .Role "system"}}<<SYS>>
    {{.Content}}
    <</SYS>>
    {{- else if eq .Role "user"}}[INST] {{.Content}} [/INST]
    {{- else if eq .Role "assistant"}} {{.Content}}
    {{- end}}
    {{- end}}
  completion: "{{.Input}}"
  chat_message: "{{.Role}}: {{.Content}}"
```

## Built-in Templates

LocalAI ships with a set of predefined templates in `./templates/`:

- `chatml.tmpl` — ChatML format used by many models
- `llama2.tmpl` — Meta's Llama 2 chat format
- `alpaca.tmpl` — Alpaca instruction format
- `mistral.tmpl` — Mistral instruct format

You can reference them by name in the config:

```yaml
template:
  chat: chatml
```

## Template Data Structures

The template receives one of these structs depending on context:

### ChatTemplate
```go
type ChatTemplateData struct {
    Messages []Message
    Functions []Function  // populated if tools/functions are present
    FunctionCall interface{}
}

type Message struct {
    Role    string  // "system", "user", "assistant", "tool"
    Content string
    Name    string  // optional, for tool messages
}
```

### CompletionTemplate
```go
type CompletionTemplateData struct {
    Input string
}
```

## Template Functions

LocalAI registers custom template functions you can use:

| Function | Description |
|----------|-------------|
| `toJson` | Marshal a value to JSON string |
| `add` | Integer addition |
| `sub` | Integer subtraction |
| `last` | Returns the last element of a slice |
| `contains` | String contains check |
| `trim` | Trim whitespace |

Example using `toJson` for function calling:

```
{{- if .Functions}}
You have access to the following tools:
{{toJson .Functions}}
{{- end}}
```

## Rendering Templates in Code

The `templates` package exposes `EvaluateTemplateForChat` and `EvaluateTemplateForCompletion`:

```go
package myfeature

import (
    "github.com/mudler/LocalAI/pkg/templates"
    "github.com/mudler/LocalAI/pkg/model"
)

func renderPrompt(cfg *config.BackendConfig, messages []schema.Message) (string, error) {
    // The template manager is usually accessed via the model loader
    tmplData := templates.ChatTemplateData{
        Messages: messages,
    }

    rendered, err := templates.EvaluateTemplateForChat(
        cfg.TemplateConfig.Chat,
        tmplData,
    )
    if err != nil {
        return "", fmt.Errorf("template rendering failed: %w", err)
    }

    return rendered, nil
}
```

## Adding a New Template Format

1. Create `templates/myformat.tmpl`
2. Test it manually with a few message sequences
3. Add it to the gallery model YAML if contributing a model definition

## Debugging Templates

Enable debug logging to see the rendered prompt before it's sent to the backend:

```bash
LOCAL_AI_DEBUG=true ./local-ai
```

You'll see log lines like:
```
DEBU Rendered template  component=templating rendered="[INST] Hello [/INST]"
```

## Common Pitfalls

- **Trailing newlines**: Go templates preserve whitespace. Use `{{-` and `-}}` to trim surrounding whitespace.
- **Missing role handling**: If your template doesn't handle `tool` role messages and the model gets tool results, it will silently drop them. Always add a fallback.
- **Template caching**: Templates are compiled and cached on first use. Restart LocalAI after editing template files during development.
- **Nil function list**: Always guard `{{if .Functions}}` before iterating — the slice may be nil for non-function-calling requests.
