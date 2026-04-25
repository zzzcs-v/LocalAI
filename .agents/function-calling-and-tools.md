# Function Calling and Tools

This guide covers how LocalAI handles function calling (tools) in chat completions, including how to parse tool definitions, dispatch calls, and return results.

## Overview

Function calling allows models to request execution of external tools during a chat completion. LocalAI supports both the OpenAI `tools` format and the legacy `functions` format.

The flow is:
1. Client sends a chat request with `tools` or `functions` defined
2. The model generates a response requesting a tool call
3. LocalAI parses the tool call from the model output
4. The response is returned to the client with `tool_calls` populated
5. The client executes the tool and sends the result back in a follow-up message

## Request Structures

```go
// Tool definition as sent by the client
type Tool struct {
    Type     string       `json:"type"`     // always "function"
    Function FunctionDef  `json:"function"`
}

type FunctionDef struct {
    Name        string          `json:"name"`
    Description string          `json:"description"`
    Parameters  json.RawMessage `json:"parameters"` // JSON Schema
}

// Tool call returned by the model
type ToolCall struct {
    ID       string           `json:"id"`
    Type     string           `json:"type"`
    Function FunctionCallResult `json:"function"`
}

type FunctionCallResult struct {
    Name      string `json:"name"`
    Arguments string `json:"arguments"` // JSON string
}
```

## Enabling Function Calling in a Config

In your model YAML config, set the grammar or template to guide the model toward structured output:

```yaml
name: my-model
parameters:
  model: my-model.gguf

# Use a grammar to enforce JSON tool-call output
function:
  grammar_type: json
  # Optional: override the prompt template used when tools are present
  no_action_function_name: "none"
  no_action_description: "If no tool is needed, respond normally."

# Template used when tools are injected into the prompt
template:
  chat_message: |
    {{- if .FunctionCall }}
    You must respond with a JSON object matching this schema:
    {"name": "<function_name>", "arguments": {<args>}}
    {{- end }}
```

## Grammar-Based Enforcement

For llama.cpp backends, LocalAI can generate a BNF grammar from the provided tool schemas to constrain model output:

```go
// GenerateToolGrammar builds a llama.cpp BNF grammar from a list of tool definitions.
// This ensures the model can only output valid JSON matching one of the tool schemas.
func GenerateToolGrammar(tools []Tool) (string, error) {
    // Each tool's parameters field is a JSON Schema.
    // We combine them into a union grammar so the model picks one.
    schemas := make([]map[string]interface{}, 0, len(tools))
    for _, t := range tools {
        var schema map[string]interface{}
        if err := json.Unmarshal(t.Function.Parameters, &schema); err != nil {
            return "", fmt.Errorf("tool %q has invalid parameters schema: %w", t.Function.Name, err)
        }
        // Wrap with the function name so output is self-identifying
        schemas = append(schemas, map[string]interface{}{
            "type": "object",
            "properties": map[string]interface{}{
                "name":      map[string]interface{}{"type": "string", "const": t.Function.Name},
                "arguments": schema,
            },
            "required": []string{"name", "arguments"},
        })
    }
    return jsonSchemaToGrammar(schemas)
}
```

## Parsing Tool Calls from Model Output

After generation, LocalAI inspects the raw model output to extract tool calls:

```go
// ParseToolCalls attempts to extract tool call JSON from raw model output.
// It handles both bare JSON objects and markdown code-fenced JSON.
func ParseToolCalls(raw string) ([]ToolCall, error) {
    raw = strings.TrimSpace(raw)

    // Strip markdown code fences if present
    if strings.HasPrefix(raw, "```") {
        lines := strings.SplitN(raw, "\n", 2)
        if len(lines) == 2 {
            raw = strings.TrimSuffix(lines[1], "```")
            raw = strings.TrimSpace(raw)
        }
    }

    var obj map[string]json.RawMessage
    if err := json.Unmarshal([]byte(raw), &obj); err != nil {
        return nil, fmt.Errorf("model output is not valid JSON: %w", err)
    }

    nameRaw, ok := obj["name"]
    if !ok {
        return nil, errors.New("tool call JSON missing 'name' field")
    }

    argsRaw, ok := obj["arguments"]
    if !ok {
        return nil, errors.New("tool call JSON missing 'arguments' field")
    }

    var name string
    if err := json.Unmarshal(nameRaw, &name); err != nil {
        return nil, fmt.Errorf("could not parse tool name: %w", err)
    }

    call := ToolCall{
        ID:   generateToolCallID(),
        Type: "function",
        Function: FunctionCallResult{
            Name:      name,
            Arguments: string(argsRaw),
        },
    }
    return []ToolCall{call}, nil
}

func generateToolCallID() string {
    return fmt.Sprintf("call_%s", uuid.New().String()[:8])
}
```

## Injecting Tools into the Prompt

When the backend does not natively support tool calling, LocalAI serializes the tool definitions into the system prompt:

```go
// InjectToolsIntoSystemPrompt appends a tool manifest to the system message.
// Used for backends that lack native function-calling support.
func InjectToolsIntoSystemPrompt(systemPrompt string, tools []Tool) (string, error) {
    if len(tools) == 0 {
        return systemPrompt, nil
    }

    defs := make([]map[string]interface{}, 0, len(tools))
    for _, t := range tools {
        var params interface{}
        _ = json.Unmarshal(t.Function.Parameters, &params)
        defs = append(defs, map[string]interface{}{
            "name":        t.Function.Name,
            "description": t.Function.Description,
            "parameters":  params,
        })
    }

    toolJSON, err := json.MarshalIndent(defs, "", "  ")
    if err != nil {
        return "", err
    }

    injection := fmt.Sprintf(
        "\n\nYou have access to the following tools:\n```json\n%s\n```\n"+
            "To call a tool, respond ONLY with a JSON object: "+
            `{"name": "<tool_name>", "arguments": {<args>}}`,
        string(toolJSON),
    )
    return systemPrompt + injection, nil
}
```

## Tool Choice

The `tool_choice` field in the request controls which tool (if any) the model must use:

| Value | Behaviour |
|---|---|
| `"none"` | Model must not call any tool |
| `"auto"` (default) | Model decides whether to call a tool |
| `{"type": "function", "function": {"name": "my_fn"}}` | Model must call `my_fn` |

LocalAI enforces `tool_choice` by adjusting the grammar or by appending a directive to the system prompt.

## Legacy `functions` Compatibility

Requests using the older `functions` / `function_call` fields are transparently converted to the `tools` format before processing:

```go
func normalizeFunctionCalling(req *ChatCompletionRequest) {
    if len(req.Functions) > 0 && len(req.Tools) == 0 {
        for _, fn := range req.Functions {
            req.Tools = append(req.Tools, Tool{
                Type:     "function",
                Function: fn,
            })
        }
    }
    if req.FunctionCall != nil && req.ToolChoice == nil {
        req.ToolChoice = req.FunctionCall
    }
}
```

## Testing

See `.agents/testing-and-mocking.md` for general patterns. For function calling specifically:

```go
func TestParseToolCalls(t *testing.T) {
    raw := `{"name": "get_weather", "arguments": {"location": "London"}}`
    calls, err := ParseToolCalls(raw)
    require.NoError(t, err)
    require.Len(t, calls, 1)
    assert.Equal(t, "get_weather", calls[0].Function.Name)
}
```

## See Also

- `.agents/prompt-templating.md` — template syntax used when injecting tools
- `.agents/request-lifecycle.md` — where tool parsing fits in the overall request flow
- `.agents/configuration-and-settings.md` — `function:` block in model YAML
