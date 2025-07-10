---
date: '2025-08-10T13:39:43-04:00'
draft: true
title: 'PydanticAI - my favorite Python library of the summer'
---

Like many tech companies in 2025, my organization is experimenting with ways to leverage Generative AI in our offerings to customers. I am part of a small team working on these ideas - we’ve built our first cloud app, a chatbot using a RAG pipeline to assist users with using our software’s command line scripts, and are in the middle of making this available to clients. For our next offering, we aimed to explore “tool use” with LLMs, giving the LLM agent the ability to call selected functions to enable it to help users in a more detailed fashion.

When looking ahead to our next avenues for development, I took some time to review the current development libraries available for use. When I read that the team behind Pydantic was releasing a new library for interfacing with large language models, I was skeptical.  Pydantic offers a great development experience for type validation, but this seemed like a big jump for them

When reading through the Pydantic “Getting Started” section, I was very impressed with how they’d provided access to numerous LLMs with minimal configuration.

Providing this optionality to users had led to complexity in our chat bot application - each LLM provider has their own message structure, parameters, and API endpoints that we had to handle. 

When using Pydantic AI, we just providing a model string name, and we can pass messages, system prompts, and settings in a uniform method.  They support OpenAI, Anthropic direct connections, as well as hosted options such as AWS Bedrock or self-hosted ollama implementations. 

Here’s an example of code before supporting both OpenAI and Anthropic APIs:

And with Pydantic AI, we can just switch out the model name:

This pleased me enough, but there was more coming when testing tool use. With most of the LLM SDKs, the tool use section of the agent initiation is a complex nest of JSON.  With Pydantic AI, I can just throw a decorator above a function, and the agent now has access to this tool.  We can also provide context (such as a database connection) to these tools, further empowering the LLMs.

Here’s a comparison of tool use with the Anthropic SDK compared with Pydantic AI:

We are still early in our “tool use” era of LLM development, but it is a joy to work with Pydantic AI so we can focus on the overall program logic and user empowerment.