# Ollama MCP Server Integration with Cline

This document details the setup, testing, and performance results of the Ollama MCP server integration with Cline, providing a comprehensive guide for using local AI models through the Model Context Protocol.

## Setup Summary

### Installation Process
1. **MCP Documentation Loaded** - Reviewed complete MCP server creation guidelines
2. **Directory Structure Created** - Set up at `/Users/trj/Documents/Cline/MCP`
3. **Repository Cloned** - From `https://github.com/NightTrek/Ollama-mcp`
4. **Dependencies Installed** - Using `npm install` (40 packages installed)
5. **Server Built** - Automatic build process created executable at `build/index.js`
6. **Configuration Updated** - Added to `cline_mcp_settings.json` with proper settings

### Configuration Details
```json
{
  "mcpServers": {
    "github.com/NightTrek/Ollama-mcp": {
      "command": "node",
      "args": ["/Users/trj/Documents/Cline/MCP/Ollama-mcp/build/index.js"],
      "disabled": false,
      "autoApprove": [],
      "env": {
        "OLLAMA_HOST": "http://127.0.0.1:11434"
      }
    }
  }
}
```

## Available Models (21 Total)
- qwen2.5-coder:1.5b, qwen2.5-coder:7b, qwen2.5-coder:3b
- qwen3:30b-a3b, qwen3:0.6b, qwen3:1.7b, qwen3:4b, qwen3:30b, qwen3:latest
- gemma3:1b, gemma3:latest, gemma3n:e2b, gemma3n:e4b
- mannix/jan-nano:ud_q8_k_xl
- alibayram/mimo-7b-rl:latest
- mxbai-embed-large:latest
- phi4-mini-reasoning:3.8b
- granite3.2:8b
- deepseek-r1:8b
- exaone3.5:7.8b
- qwen2.5:latest

## Performance Testing Results

### ✅ Excellent Performance Models

#### 1. **gemma3:1b** - Best Overall Small Model
- **Response Time**: Fast (< 10 seconds)
- **Quality**: Excellent structured responses
- **Use Cases**: General questions, explanations, educational content
- **Example Output**: Detailed explanation of primary colors with proper formatting and follow-up questions
- **Cline Integration**: Perfect - reliable and consistent

#### 2. **gemma3n:e2b** - Superior Reasoning
- **Response Time**: Moderate (10-20 seconds)
- **Quality**: Outstanding explanatory capabilities
- **Use Cases**: Complex explanations, comparisons, educational content
- **Example Output**: Comprehensive AI vs ML explanation with analogies and structured formatting
- **Cline Integration**: Excellent - best for detailed explanations

#### 3. **qwen2.5-coder:1.5b** - Coding Specialist
- **Response Time**: Fast (< 15 seconds)
- **Quality**: Excellent for programming tasks
- **Use Cases**: Code generation, programming explanations, technical documentation
- **Example Output**: Python function with detailed explanation and advanced concepts
- **Cline Integration**: Perfect for development tasks

#### 4. **qwen2.5:latest** - Reliable General Purpose
- **Response Time**: Moderate (15-25 seconds)
- **Quality**: Good conversational responses
- **Use Cases**: General chat, basic math, simple explanations
- **Example Output**: Correct math with friendly conversational style
- **Cline Integration**: Very good for general tasks

#### 5. **granite3.2:8b** - Technical Excellence
- **Response Time**: Moderate (20-30 seconds)
- **Quality**: Excellent technical explanations
- **Use Cases**: Technical documentation, detailed explanations, educational content
- **Example Output**: Comprehensive computer components explanation with proper technical detail
- **Cline Integration**: Excellent for technical tasks

#### 6. **qwen3:0.6b** - Fast Reasoning Specialist
- **Response Time**: Fast (< 30 seconds)
- **Quality**: Good reasoning and conversational responses.
- **Use Cases**: General chat, simple reasoning tasks, and testing the `think` feature.
- **Example Output**: Successfully returns both a final answer and a detailed "thinking" block when prompted.
- **Cline Integration**: Excellent - the best choice for tasks requiring observable reasoning without the long wait times of larger models.

#### 7. **qwen3:1.7b** - Reasoning Specialist (with token limits)
- **Response Time**: Moderate (dependent on `num_predict`)
- **Quality**: Good reasoning capabilities.
- **Use Cases**: Reasoning tasks where a limited-length response is acceptable.
- **Example Output**: Successfully returns a thinking block but may time out without `num_predict`.
- **Cline Integration**: Good, but requires using `num_predict` to ensure timely responses.

### ⚠️ Models with Issues

#### Timeout Issues (Model Loading Time)
The primary cause of timeouts appears to be the **initial model loading time** into RAM/VRAM, which can exceed the default 60-second request timeout. This is not necessarily due to the model's `thinking` capability, but rather the time it takes for Ollama to prepare the model for its first inference.

- **mannix/jan-nano:ud_q8_k_xl** - Custom quantized model, requires significant pre-loading time.
- **mannix/jan-nano** - Base model also needs substantial initialization time.
- **alibayram/mimo-7b-rl:latest** - Custom RL model with slow startup.
- **deepseek-r1:8b** - Reasoning model with long processing and loading time.
- **qwen3:0.6b, qwen3:1.7b** - Smaller models that still exhibit loading delays.
- **gemma3n:e4b, gemma3:latest** - Larger models that timeout on complex tasks, likely due to both loading and processing demands.

#### Technical Issues
- **qwen2.5-coder:3b** - Intermittent formatting issues (can sometimes return template tokens like `<|im_start|>`), but is otherwise functional.
- **phi4-mini-reasoning:3.8b** - Incomplete responses (partial answers only)
- **run tool** - Streaming API compatibility issues (`pipeThrough` error)

## Controlling Model Reasoning (Thinking)

The Ollama MCP server now correctly supports controlling model reasoning through a `think` boolean parameter in the `chat_completion` and `run` tools. This feature allows you to enable or disable the model's internal "chain-of-thought" or step-by-step reasoning process.

### How it Works
When you set `"think": true` in an API call, the MCP server instructs Ollama to expose the model's reasoning steps, which are often returned within `<think>...</think>` tags in the response. When `"think": false` (or omitted), the model provides a direct, final answer.

### Verifying Model Support
Not all models support this feature. Before using the `think` parameter, it is crucial to verify that the model's template includes logic for handling it. You can do this using the `show` tool:

```bash
# This will display the model's template
ollama show <model_name> --template
```

Look for template sections that reference `$.Think` or similar variables. For example, the `mannix/jan-nano:latest` and `qwen3:0.6b` models include the following, indicating support:

```
{{- if and $.IsThinkSet (eq $i $lastUserIdx) }}
   {{- if $.Think -}}
      {{- " "}}/think
   {{- else -}}
      {{- " "}}/no_think
   {{- end -}}
{{- end }}
```

Models like `gemma3:1b` and `qwen2.5-coder:1.5b` do not have this logic in their templates and will ignore the `think` parameter.

### Known Issues
- **Timeout with Thinking Models**: Larger models that support thinking, such as `mannix/jan-nano:latest`, can be very slow to load and may time out. For a faster alternative, **`qwen3:0.6b`** is a small model that reliably supports the `think` parameter without significant loading delays.

## MCP Tools Available

### Working Tools
1. **list** - ✅ Lists all available models (tested successfully)
2. **show** - ✅ Shows detailed model information (tested successfully)
3. **chat_completion** - ✅ OpenAI-compatible chat API. Now correctly uses the `/api/chat` endpoint and supports the `think` parameter.
4. **serve** - ✅ Detects if Ollama server is running

### Problematic Tools
1. **run** - ❌ Has known streaming response issues, but correctly passes the `think` parameter to the `/api/generate` endpoint.
2. **pull, push, create, cp, rm** - ⚠️ Not tested (model management tools)

## Cline Integration Assessment

### Excellent Integration (Recommended)
- **gemma3:1b** - Best balance of speed and quality
- **qwen2.5-coder:1.5b** - Perfect for coding tasks
- **gemma3n:e2b** - Best for detailed explanations

### Good Integration
- **qwen2.5:latest** - Reliable for general tasks
- **granite3.2:8b** - Great for technical content

### Poor Integration (Not Recommended)
- Custom models (mannix/jan-nano variants)
- Large models that timeout frequently
- Models with formatting issues

## Best Practices for Cline Usage

### Server and Model Verification
After starting or restarting the Ollama MCP server, it is recommended to perform a quick verification to ensure everything is working correctly.

1.  **Check Server Health**: Use the `list` tool to confirm the server is responsive and to see the available models.
    ```json
    {
      "tool": "list"
    }
    ```
2.  **Confirm Basic Model Functionality**: Run a simple test with a known reliable model like `gemma3:1b` to ensure inference is working.
    ```json
    {
      "tool": "chat_completion",
      "model": "gemma3:1b",
      "messages": [
        { "role": "user", "content": "Hello. Just reply 'hi'." }
      ]
    }
    ```
3.  **Investigate Model Capabilities**: Do not assume a model supports advanced features like thinking. Use the `show` tool with various flags (`--template`, `--parameters`, etc.) to look for evidence of specific capabilities before using them.
4.  **Confirm Non-Thinking Mode**: If a model supports thinking, confirm it also works as expected when `think` is set to `false`. This ensures the toggle is fully functional.

### Model Selection Guidelines
1. **For Coding Tasks**: Use `qwen2.5-coder:1.5b`
2. **For General Questions**: Use `gemma3:1b`
3. **For Detailed Explanations**: Use `gemma3n:e2b`
4. **For Technical Content**: Use `granite3.2:8b`
5. **For Simple Math/Chat**: Use `qwen2.5:latest`

### Timeout Management
- Set timeout to 120000ms (2 minutes) for reliable models.
- Avoid complex prompts with larger models.
- Use simpler, more direct questions for better response times.
- For models that are slow to generate responses, use the `num_predict` parameter to limit the output length and prevent timeouts.

### Temperature Settings
- **Coding**: 0.3-0.5 (more deterministic)
- **Creative**: 0.7-0.8 (more varied)
- **Technical**: 0.4-0.6 (balanced)

### Controlling Output Length
You can control the maximum number of tokens in a response by using the `num_predict` parameter. This is useful for generating shorter, more concise answers or for preventing overly long responses. Setting `num_predict` to `-1` will generate tokens until the context is full.

## Usage Examples

### Successful Chat Completion Pattern
This example shows a standard request. To control reasoning, add the `think` parameter.

```javascript
{
  "model": "gemma3:1b",
  "messages": [
    {
      "role": "user",
      "content": "Your question here"
    }
  ],
  "temperature": 0.7,
  "timeout": 120000,
  "think": false, // Set to true for models that support it
  "num_predict": 50 // Optional: Set the max number of tokens
}
```

### Response Format
All successful responses follow OpenAI-compatible format:
```json
{
  "id": "chatcmpl-1751114169242",
  "object": "chat.completion",
  "created": 1751114169,
  "model": "gemma3:1b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Response content here"
      },
      "finish_reason": "stop"
    }
  ]
}
```

## Troubleshooting

### Common Issues
1. **Timeout Errors**: This is often due to model loading time. Try simpler models or pre-load a model by running a simple query in your terminal first (`ollama run model_name`).
2. **Formatting Issues**: Avoid models with known template problems.
3. **Server Not Running**: Use the `serve` tool to check Ollama status.
4. **Model Not Found**: Verify the model name with the `list` tool.

### Performance Optimization
- Use smaller models (1b-3b) for faster responses
- Keep prompts concise and specific
- Avoid creative tasks with technical models
- Pre-load models by running simple queries first


## Conclusion

The Ollama MCP server integration with Cline is highly successful with the right model selection. The top 5 working models provide excellent coverage for most use cases:

1. **gemma3:1b** - Primary recommendation for general use
2. **qwen2.5-coder:1.5b** - Essential for development tasks
3. **gemma3n:e2b** - Best for educational/explanatory content
4. **granite3.2:8b** - Excellent for technical documentation
5. **qwen2.5:latest** - Reliable backup for various tasks

This setup provides a powerful local AI capability within Cline, enabling privacy-focused AI assistance without external API dependencies.



---
*Last Updated: June 28, 2025*
*Tested with Cline v3.18.0 and Ollama MCP Server v0.1.0*
