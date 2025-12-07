---
title: "Microsoft Agent Framework - Creating Azure AI Foundry Agents"
date: 2025-10-08
image: "/images/post/maf.png"
description: "Microsoft Agent Framework is the new open-source development kit for creating agents and agentic workflows. This article introduces foundry agents."
categories: ["Microsoft", "Foundry Agents", "Agent Framework"]
type: "featured"
draft: false
---

In an earlier article, we looked at a brief [introduction to the Microsoft Agent Framework](https://ai-engineer.in/post/introduction-to-microsoft-agent-framework/). This framework combines the best parts of AutoGen and Semantic Kernel into a unified framework for building enterprise and production-ready AI agents. It supports different types of agents, and [Microsoft Foundry Agents](https://ai-engineer.in/post/getting-started-with-foundry-agents/) is one among those. All agent types inherit from the common base class `AIAgent` to provide a consistent interface. 

The Foundry agents are created using `AzureAIAgentClient`. Let us start with an example.

```python
import asyncio
from agent_framework.azure import AzureAIAgentClient
from azure.identity.aio import AzureCliCredential

async def main():
    async with (
        AzureCliCredential() as credential,
        AzureAIAgentClient(
            project_endpoint="https://ai-engineer-in.services.ai.azure.com/api/projects/ai-engineer-in",
            model_deployment_name="gpt-5-mini",
            async_credential=credential,
            agent_name="WeatherAgent",
        ).create_agent(
            instructions="You are a weather man. Provide accurate and concise weather information based on user queries.",
        ) as agent,
    ):
        result = await agent.run("What's the weather like in Bengaluru?")
        print(result.text)

asyncio.run(main())
```

You can also supply the `project_endpoint` and `model_deployment_name` as environment variables. 

```shell
export AZURE_AI_PROJECT_ENDPOINT="https://ai-engineer-in.services.ai.azure.com/api/projects/ai-engineer-in"
export AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4o-mini"
```

This example uses the Azure CLI credentials cached on the local system to authenticate with the Foundry. The `create_agent()` method provides a convenient way to create an agent. The `run()` method on the agent instance supplies the prompt to the LLM and retrieves the generated response.

```shell
PS C:\> python .\01-agent-basic-no-env.py
I can’t access live data from here. Do you want a real‑time forecast (I can’t fetch it unless you paste it or allow a tool) or a quick summary of typical/current-season weather for Bengaluru?

Quick summary (typical for early December / dry season):
- Overall: mild, dry and pleasant.
- Daytime highs: about 24–30°C.
- Night/morning lows: about 14–20°C.
- Rain: low chance of rain (post‑monsoon season).
- Wind: light to moderate breezes.
- What to wear: light layers for daytime; a light sweater/jacket for mornings/evenings.

If you want current temperature, wind, humidity, or a 7‑day forecast, tell me and I’ll (a) explain how to get it quickly online or (b) you can paste your current weather output and I’ll interpret it. Which would you like?
```

You can generate a streaming response using the `run_stream()` method. 

```python
print("Agent: ", end="", flush=True)
async for chunk in agent.run_stream("Tell me a short story"):
    if chunk.text:
        print(chunk.text, end="", flush=True)
print()
```

The model that we are using in these examples does not have access to real-time weather information. We can provide the agent with tools to address this. We will use the [OpenWeatherMap API](https://home.openweathermap.org/api_keys) to retrieve the current weather at a given location and generate a response to a user's prompt.

```python
import asyncio
from typing import Annotated
from agent_framework.azure import AzureAIAgentClient
from azure.identity.aio import AzureCliCredential
from pydantic import Field
from dotenv import load_dotenv
import os

load_dotenv()

def get_weather(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    import requests
    import json

    api_key = os.getenv("OPEN_WEATHERMAP_API_KEY")

    base_url = "http://api.openweathermap.org/data/2.5/weather"
    complete_url = f"{base_url}?q={location}&appid={api_key}&units=metric"

    response = requests.get(complete_url)
    data = response.json()

    if data["cod"] != "404":
        main_data = data["main"]
        current_temperature = main_data["temp"]

        return f"Temperature: {current_temperature}°C"
    else:
        return "City not found"

async def main():
    async with (
        AzureCliCredential() as credential,
        AzureAIAgentClient(async_credential=credential).create_agent(
            name="WeatherAgent",
            instructions="You are a helpful weather assistant.",
            tools=get_weather,
            store=True
        ) as agent,
    ):
        result = await agent.run("Given the current climate in Bengaluru, what should I wear?")
        print(result.text)

asyncio.run(main())
```

When we run this, the agent uses the `get_weather` tool to retrieve the weather at the location specified and generates an appropriate response to the prompt.

```shell
PS C:\> python .\03-agent-tool-call.py
Right now it’s about 20.5°C in Bengaluru — mild and slightly cool. Practical dressing tips:

- Base layer: a light long-sleeve shirt, cotton tee with a thin sweater, or a casual button-down.
- Outer layer: carry a light jacket, hoodie, or thin cardigan — easy to remove if it warms up.
- Bottoms: jeans, chinos, or trousers are comfortable; skirts with light tights also work.
- Shoes: closed shoes or sneakers; sandals are ok if you run warm, but a closed pair is more comfortable in the cool.
- Accessories: a light scarf if you feel chilly in the morning/evening. Carry a compact umbrella or light windbreaker if you want to be safe against sudden showers/wind.

If you tell me what you’ll be doing (office, outdoor activity, evening out) or whether you’re sensitive to cold, I can suggest a more specific outfit. Want me to check rain/wind for the next few hours?
```

Besides the function tools, such as the example above, Foundry agents also support hosted tools. This includes a code interpreter tool. The next example demonstrates this.

```python
import asyncio
from agent_framework import HostedCodeInterpreterTool
from agent_framework.azure import AzureAIAgentClient
from azure.identity.aio import AzureCliCredential

from dotenv import load_dotenv  
load_dotenv()

async def main():
    async with (
        AzureCliCredential() as credential,
        AzureAIAgentClient(async_credential=credential).create_agent(
            name="CodingExecutionAgent",
            instructions="You are a helpful assistant that can write and execute Python code.",
            tools=HostedCodeInterpreterTool()
        ) as agent,
    ):
        result = await agent.run("Calculate the 100th prime number.")
        print(result.text)

asyncio.run(main())
```

This program, when executed, generates and executes the code required to answer the prompt.

```shell
PS C:\> python .\04-agent-code-interpreter.py
The 100th prime number is 541.
```

This article is a deep dive into creating Azure AI Foundry agents using the Microsoft Agent Framework. In the next parts of this series, we shall look at other types of agents that we can create using this framework.

{{< notice "info" >}}
  Last updated: 7th December 2025
{{< /notice >}}