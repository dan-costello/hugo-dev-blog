---
date: '2025-08-10T13:39:43-04:00'
draft: true
title: 'PydanticAI - my favorite Python library of the summer'
---

#### PydanticAI abstracts away much of the glue code for working with Large Language models - freeing developers to focus on business logic and use cases

Like many tech companies in 2025, my organization is experimenting with ways to leverage Generative AI in our offerings to customers. I am part of a small team working on these ideas - we’ve built our first cloud app, a chatbot using a RAG pipeline to assist users with using our software’s various modules, and are in the middle of making this available to clients. For our next offering, we aimed to explore “tool use” with LLMs, giving the LLM agent the ability to call selected functions to enable it to help users in a more detailed fashion.

Before starting development, I took some time to review the current Python libraries for interfacing with LLMs. There are libraries available from each of the major LLM providers (Anthropic, OpenAI, etc.), but a new library named PydanticAI was now available. When I read that the team behind Pydantic was releasing a new library for interfacing with large language models, I was skeptical.  Pydantic offers a great development experience for type validation in Python, but a LLM library seemed like a big jump for them. After working with the library for a few days, I am impressed with their design choices and wish the library had been available a year ago.

## Switching between Models
When reading through the Pydantic “Getting Started” section, I was very impressed with how they’d provided access to numerous LLMs with minimal configuration.

Providing the optionality to switch between various LLM providers had led to backend complexity in our chat bot application - each LLM provider has their own message structure, parameters, and API endpoints that we had to handle. 

When using Pydantic AI, we can just provide a model string name. After creating the `Agent` with this parameter, we can pass messages, system prompts, and settings in a uniform method.  PydanticAI supports OpenAI and Anthropic direct connections, as well as options such as AWS Bedrock or self-hosted Ollama implementations. 

Here’s an example of simplified code supporting both OpenAI and Anthropic APIs[^note]. Note the different ways of providing system prompts.
```
...
available_models = ['4o', 'sonnet3.7','qwen']

# Hidden: User selects "model" and defines USER_QUERY via CLI

if model == "4o":
    client=OpenAI(api_key="KEY")
    response = client.chat.completions.create(
        model="claude-opus-4-1-20250805", # Anthropic model name
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": USER_QUERY}
        ],
    )
    print(response)
elif model == "sonnet3.7":
    client = anthropic.Anthropic(api_key="my_api_key")
    response = client.messages.create(
        model="claude-3-7-sonnet-20250219",
        system=SYSTEM_PROMPT,
        messages=[
            {"role": "user", "content": USER_QUERY}
        ]
    )
    print(response)
elif model == "qwen":
...



```
The configuration functions continue to expand when providing optionality for self-hosted models or those on AWS Bedrock.

And with Pydantic AI, we can just switch out the model name:
```
from pydantic_ai import Agent

model = 'google-gla:gemini-1.5-flash'
model = 'anthropic:claude-3-7-sonnet-latest'
selected_model = 'bedrock:us.anthropic.claude-3-7-sonnet-20250219-v1:0'

agent = Agent(  
    selected_model,
    system_prompt=SYSTEM_PROMPT,  
)

result = agent.run_sync('Where does "hello world" come from?')  
print(result.output)
```

## Tool Use
The easy method of switching between models was pleasing enough, and I was impressed even more with testing tool use. With most of the LLM SDKs, the tool use section of the agent initiation is a complex nest of JSON.  With Pydantic AI, we can simply add decorator above a function, giving the agent awareness of this tool.  We can also provide context (such as a database connection) to these tools, further empowering our LLM agents.

Here’s a comparison of tool use with the Anthropic SDK compared with Pydantic AI:

### Tool use with Anthropic SDK:
```
import anthropic
import json

client = anthropic.Anthropic()

# Python function to get weather from API
def get_weather(location, unit="fahrenheit"):
    # In reality, you'd call a weather API here
    return {
        "location": location,
        "temperature": "72",
        "unit": unit,
        "condition": "sunny"
    }

# tools schema to describe avaialble functions
tools = [
    {
        "name": "get_weather",
        "description": "Get the current weather in a given location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "The city and state, e.g. San Francisco, CA"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "The unit of temperature to return"
                }
            },
            "required": ["location"]
        }
    }
]

# Send request to external LLM, providing tools schema
message = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "What's the weather like in Boston?"}
    ]
)
```
### Tool use with PydanticAI:
```
from pydantic_ai import Agent, RunContext

agent = Agent('anthropic:claude-3-5-sonnet-latest')

@agent.tool_plain  
def get_weather(location, unit="fahrenheit"):
    """Retrieve temperature and conditions for a given location.""
    return {
        "location": location,
        "temperature": "72",
        "unit": unit,
        "condition": "sunny"
    }


weather = agent.run_sync('What's the weather like today in San Antonio?')
```

The decorator design makes adding tools much more straightforward and fun - we can now focus on what capabilities we want to give our agent, and not be bogged down in ensuring the JSON schema is complete for each available tool.


### Onward 
We are still early in our “tool use” era of LLM development, but it is a joy to work with Pydantic AI so we can focus on the overall program logic and user empowerment.  Using this library doesn't mean we don't need to know how to work with LLM APIs, but it removes much of the "glue code" necessary when working with other offerings.

[^note]: While writing this piece I realized that Anthropic and many other LLM providers provide OpenAI-compatible endpoints, which removes the need for the `if` logic and accomodating the various SDK languages. This would have useful to discover earlier, but good to know!