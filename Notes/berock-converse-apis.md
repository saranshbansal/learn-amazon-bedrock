# Converse APIs

## Overview

The Amazon Bedrock Converse API enables developers to build sophisticated conversational applications while managing context across multiple interactions. Unlike standard foundation model calls which are stateless, the Converse API provides a structured way to maintain conversation history and context, making it ideal for chatbots, virtual assistants, and other interactive applications.

Converse API also provides a unified way to invoke foundation models, making it easier to swap out models when needed.

## Building Blocks of the Converse API

Now that we understand what the Converse API is and its purpose, let's examine the essential components that make it work. Each component plays a specific role in creating effective conversational experiences, and understanding how they interact is crucial for building robust applications.

From system prompts that shape the model's behavior to tool configurations that enable real-world actions, these building blocks can be combined in various ways to create exactly the conversational experience you need.

Let's break down each component, explore practical examples, and review best practices for implementation.

### System Prompt

A system prompt defines the model's behavior, personality, and operating parameters. It sets the foundation for how the model should interact and what type of responses it should provide.

```python
system_prompt = "You are an AI assistant that helps create playlists. Only return song names and artists."
```

### Message Array

Message arrays store the conversation history as a sequence of interactions between the user and the assistant. Each message includes a role identifier and the content of the message.

```python
messages = [
    {
        "role": "user",
        "content": "Create a playlist of 3 pop songs"
    }
]
```

### Inference Parameters

Inference parameters control how the model generates responses. Different models support different parameters, so it's important to check the model-specific documentation.

Example for Claude models:

```python
inference_config = {
    "temperature": 0.7,
    "max_tokens": 1024,
    "top_p": 0.999,
}
additional_model_fields = {"top_k": top_k}
```

### Invoking the Converse API

```python
response = bedrock.converse(
    modelId="<MODEL_ID_CLAUDE>",
    messages=messages,
    system=system_prompt,
    inferenceConfig=inference_config,
    additionalModelRequestFields=additional_model_fields
)
```

### Tool Configuration

Tool configurations enable the model to interact with external functions.

Here's an example of defining a weather tool and then passing the tool to the Converse API:

```python
weather_tool = {
    "toolSpec": {
        "name": "getWeather",
        "description": "Gets the current weather using latitude and longitude.",
        "inputSchema": {
            "json": {
                "type": "object",
                "properties": {
                    "lat": {"type": "number"},
                    "lon": {"type": "number"}
                },
                "required": ["lat", "lon"]
            }
        }
    }
}

initial_response = bedrock.converse(
    modelId="<MODEL_ID>",
    system="<system_prompt>",
    messages=["<initial_user_msg>"],
    toolConfig={
        "tools": [weather_tool],
        "toolChoice": {"auto": {}}
    },
    inferenceConfig={"temperature": 0.7}
)
```

## Important Implementation Considerations

### Context Window Management

The context window defines how much information the model can process at once, measured in tokens. When building conversational applications, you need to carefully manage this limitation:

- Monitor total tokens in conversation history
- Implement conversation summarization or pruning for long interactions
- Consider model-specific token limits
- Handle cases where context window is exceeded

### Production Best Practices

When deploying conversational applications to production:

- Store conversation histories in DynamoDB or Redis, not in memory
- Implement proper session management to isolate user conversations
- Use session IDs to retrieve relevant conversation history
- Handle rate limits and implement retry logic
- Monitor and log model interactions

### Model-Specific Considerations

Different foundation models have varying capabilities and requirements:

- Some models don't support system prompts (like Titan)
- Parameter support varies between models
- Response formats may differ
- Temperature and other inference parameters may behave differently
- Always check model-specific documentation

## Tool Execution Flow

When using tools with the Converse API, the process follows these steps:

1. Your application sends the user's message and available tools to the model
2. The model determines if a tool is needed and includes a toolUse block in its response
3. Your application must then:
   - Check for the presence of a toolUse block
   - Extract the tool parameters from the response
   - Execute the appropriate tool (like calling a Lambda function)
   - Handle any errors during tool execution
   - Format the tool's result
   - Send the tool's result back to the model with the original conversation history
4. Model generates final response incorporating the tool's output

### Example Flow

```python
import boto3
import json

bedrock = boto3.client(service_name='bedrock-runtime')
lambda_client = boto3.client("lambda")

MODEL_ID_CLAUDE_SONNET = "anthropic.claude-3-sonnet-20240229-v1:0"

# Define toolSpec for lambda function
# Lambda function named getWeather already created
weather_tool = {
    "toolSpec": {
        "name": "getWeather",
        "description": "Gets the current weather using latitude and longitude.",
        "inputSchema": {
            "json": {
                "type": "object",
                "properties": {
                    "lat": {"type": "number"},
                    "lon": {"type": "number"}
                },
                "required": ["lat", "lon"]
            }
        }
    }
}

# Invoke Lambda Function (tool)
def invoke_weather_lambda(input_data):
    response = lambda_client.invoke(
        FunctionName="weather-tool",
        InvocationType="RequestResponse",
        Payload=json.dumps(input_data)
    )
    payload = response["Payload"].read()
    result = json.loads(payload)
    body = result.get("body", "{}")
    return json.loads(body) if isinstance(body, str) else body

initial_user_msg = {
    "role": "user",
    "content": [{"text": "What's the weather like in portland?"}]
}

system_prompt = [
    {"text": """
    You are a helpful assistant that can check the weather using tools when needed.
    If the user inputs a city name that exists in multiple geogrpahic areas, ask the user for clarification.
    """}
]

# Ask model initial question
initial_response = bedrock.converse(
    modelId=MODEL_ID_CLAUDE_SONNET,
    system=system_prompt,
    messages=[initial_user_msg],
    toolConfig={
        "tools": [weather_tool],
        "toolChoice": {"auto": {}}
    },
    inferenceConfig={"temperature": 0.7}
)

# Parse response from Claude to determine if tool is needed
assistant_msg = initial_response["output"]["message"]
content_blocks = assistant_msg["content"]
tool_use_block = next((block for block in content_blocks if "toolUse" in block), None)

if not tool_use_block:
    print("=== Claude Response ===")
    print(content_blocks[0]["text"])
else:
    tool_use = tool_use_block["toolUse"]
    tool_input = tool_use["input"]
    tool_use_id = tool_use["toolUseId"]
    print(tool_use_block)
    print(f"→ Claude requested tool: getWeather with input: {tool_input}")

    # Invoke tool
    tool_output = invoke_weather_lambda(tool_input)
    print(f"← Lambda returned: {tool_output}")

    # Print out tool output
    try:
        summary = f"The weather in {tool_output['location']} is currently {tool_output['temperature']} with {tool_output['condition']}."
    except Exception as e:
        summary = f"Sorry, I couldn't retrieve the weather. ({str(e)})"

    # Construct tool result message
    tool_result_msg = {
        "role": "user",
        "content": [
            {
                "toolResult": {
                    "toolUseId": tool_use_id,
                    "content": [{"text": summary}]
                }
            }
        ]
    }

    # Pass tool result message to model
    final_response = bedrock.converse(
        modelId=MODEL_ID_CLAUDE_SONNET,
        messages=[initial_user_msg, assistant_msg, tool_result_msg],
        toolConfig={
            "tools": [weather_tool],
            "toolChoice": {"auto": {}}
        },
        inferenceConfig={"temperature": 0.7}
    )

    # Print final response
    final_msg = final_response["output"]["message"]["content"][0]["text"]
    print("\n=== Final Agent Response ===")
    print(final_msg)
```