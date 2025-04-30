# A2A Client Implementation

This section covers how to implement and use A2A clients in both Python and JavaScript. A2A clients are responsible for discovering agent capabilities, sending requests, and processing responses.

## Python Client Implementation

The A2A Python client is implemented in the `common/client/client.py` file. It provides a simple interface for interacting with A2A servers.

### Basic Structure

```python
class A2AClient:
    def __init__(self, agent_card: AgentCard = None, url: str = None):
        if agent_card:
            self.url = agent_card.url
        elif url:
            self.url = url
        else:
            raise ValueError("Must provide either agent_card or url")

    async def send_task(self, payload: dict[str, Any]) -> SendTaskResponse:
        request = SendTaskRequest(params=payload)
        return SendTaskResponse(**await self._send_request(request))

    async def send_task_streaming(
        self, payload: dict[str, Any]
    ) -> AsyncIterable[SendTaskStreamingResponse]:
        request = SendTaskStreamingRequest(params=payload)
        with httpx.Client(timeout=None) as client:
            with connect_sse(
                client, "POST", self.url, json=request.model_dump()
            ) as event_source:
                try:
                    for sse in event_source.iter_sse():
                        yield SendTaskStreamingResponse(**json.loads(sse.data))
                except json.JSONDecodeError as e:
                    raise A2AClientJSONError(str(e)) from e
                except httpx.RequestError as e:
                    raise A2AClientHTTPError(400, str(e)) from e

    async def _send_request(self, request: JSONRPCRequest) -> dict[str, Any]:
        async with httpx.AsyncClient() as client:
            try:
                response = await client.post(
                    self.url, json=request.model_dump(), timeout=30
                )
                response.raise_for_status()
                return response.json()
            except httpx.HTTPStatusError as e:
                raise A2AClientHTTPError(e.response.status_code, str(e)) from e
            except json.JSONDecodeError as e:
                raise A2AClientJSONError(str(e)) from e
```

### Key Methods

- **send_task**: Sends a synchronous task request and returns the complete response
- **send_task_streaming**: Sends a streaming task request and yields updates as they arrive
- **get_task**: Retrieves the current state of a task
- **cancel_task**: Attempts to cancel a running task
- **set_task_push_notification**: Configures push notifications for a task
- **get_task_push_notification**: Retrieves the current push notification configuration for a task

### Usage Example

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, Message

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID
    task_id = str(uuid.uuid4())
    
    # Prepare the task parameters
    payload = {
        "id": task_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Convert 100 USD to EUR")]
        )
    }
    
    # Send the task and get the response
    response = await client.send_task(payload)
    
    # Print the response
    print(f"Task ID: {response.result.id}")
    print(f"Status: {response.result.status.state}")
    if response.result.status.message:
        print(f"Response: {response.result.status.message.parts[0].text}")
    
    # Print any artifacts
    if response.result.artifacts:
        for artifact in response.result.artifacts:
            print(f"Artifact: {artifact.name}")
            for part in artifact.parts:
                if part.type == "data":
                    print(f"Data: {part.data}")

asyncio.run(main())
```

### Streaming Example

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, Message

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID
    task_id = str(uuid.uuid4())
    
    # Prepare the task parameters
    payload = {
        "id": task_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Convert 100 USD to EUR")]
        )
    }
    
    # Send the task with streaming and process updates
    async for update in client.send_task_streaming(payload):
        if hasattr(update.result, "status"):
            print(f"Status Update: {update.result.status.state}")
            if update.result.status.message:
                print(f"Message: {update.result.status.message.parts[0].text}")
        elif hasattr(update.result, "artifact"):
            print(f"Artifact Update: {update.result.artifact.name}")

asyncio.run(main())
```

## JavaScript Client Implementation

The A2A JavaScript client is implemented in the `samples/js/src/client/client.ts` file. It provides similar functionality to the Python client but with a TypeScript interface.

### Basic Structure

```typescript
export class A2AClient {
  private baseUrl: string;
  private fetchImpl: typeof fetch;
  private cachedAgentCard: AgentCard | null = null;

  constructor(baseUrl: string, fetchImpl: typeof fetch = fetch) {
    this.baseUrl = baseUrl.endsWith("/") ? baseUrl.slice(0, -1) : baseUrl;
    this.fetchImpl = fetchImpl;
  }

  private async _makeHttpRequest<Req extends A2ARequest>(
    method: Req["method"],
    params: Req["params"],
    acceptHeader: "application/json" | "text/event-stream" = "application/json"
  ): Promise<Response> {
    const requestId = this._generateRequestId();
    const requestBody: JSONRPCRequest = {
      jsonrpc: "2.0",
      id: requestId,
      method: method,
      params: params,
    };

    return this.fetchImpl(`${this.baseUrl}`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Accept: acceptHeader,
      },
      body: JSON.stringify(requestBody),
    });
  }

  // Other methods...
}
```

### Key Methods

- **sendTask**: Sends a synchronous task request and returns the complete response
- **sendTaskSubscribe**: Sends a streaming task request and yields updates as they arrive
- **getTask**: Retrieves the current state of a task
- **cancelTask**: Attempts to cancel a running task
- **setTaskPushNotification**: Configures push notifications for a task
- **getTaskPushNotification**: Retrieves the current push notification configuration for a task
- **getAgentCard**: Retrieves the agent card from the server

### Usage Example

```typescript
import { A2AClient } from "./client";
import { v4 as uuidv4 } from "uuid";

async function main() {
  // Initialize client with agent URL
  const client = new A2AClient("http://localhost:10000");
  
  // Create a unique task ID
  const taskId = uuidv4();
  
  // Prepare the task parameters
  const params = {
    id: taskId,
    message: {
      role: "user",
      parts: [{ type: "text", text: "Convert 100 USD to EUR" }]
    }
  };
  
  try {
    // Send the task and get the response
    const response = await client.sendTask(params);
    
    // Print the response
    console.log(`Task ID: ${response.id}`);
    console.log(`Status: ${response.status.state}`);
    if (response.status.message) {
      console.log(`Response: ${response.status.message.parts[0].text}`);
    }
    
    // Print any artifacts
    if (response.artifacts) {
      for (const artifact of response.artifacts) {
        console.log(`Artifact: ${artifact.name}`);
        for (const part of artifact.parts) {
          if (part.type === "data") {
            console.log(`Data: ${JSON.stringify(part.data)}`);
          }
        }
      }
    }
  } catch (error) {
    console.error("Error:", error);
  }
}

main();
```

### Streaming Example

```typescript
import { A2AClient } from "./client";
import { v4 as uuidv4 } from "uuid";

async function main() {
  // Initialize client with agent URL
  const client = new A2AClient("http://localhost:10000");
  
  // Create a unique task ID
  const taskId = uuidv4();
  
  // Prepare the task parameters
  const params = {
    id: taskId,
    message: {
      role: "user",
      parts: [{ type: "text", text: "Convert 100 USD to EUR" }]
    }
  };
  
  try {
    // Send the task with streaming and process updates
    const stream = client.sendTaskSubscribe(params);
    
    for await (const update of stream) {
      if ("status" in update) {
        console.log(`Status Update: ${update.status.state}`);
        if (update.status.message) {
          console.log(`Message: ${update.status.message.parts[0].text}`);
        }
      } else if ("artifact" in update) {
        console.log(`Artifact Update: ${update.artifact.name}`);
      }
    }
  } catch (error) {
    console.error("Error:", error);
  }
}

main();
```

## Agent Discovery

A2A clients can discover agent capabilities by fetching the agent card from the server:

```python
# Python
from common.client.card_resolver import A2ACardResolver

async def discover_agent(url: str):
    resolver = A2ACardResolver()
    agent_card = await resolver.resolve(url)
    print(f"Agent Name: {agent_card.name}")
    print(f"Agent Description: {agent_card.description}")
    print(f"Agent Skills: {[skill.name for skill in agent_card.skills]}")
    return agent_card
```

```typescript
// JavaScript
import { A2AClient } from "./client";

async function discoverAgent(url: string) {
  const client = new A2AClient(url);
  const agentCard = await client.getAgentCard();
  console.log(`Agent Name: ${agentCard.name}`);
  console.log(`Agent Description: ${agentCard.description}`);
  console.log(`Agent Skills: ${agentCard.skills.map(skill => skill.name).join(", ")}`);
  return agentCard;
}
```

## Error Handling

A2A clients should handle various error conditions:

```python
# Python
try:
    response = await client.send_task(payload)
except A2AClientHTTPError as e:
    print(f"HTTP Error: {e.status_code} - {e}")
except A2AClientJSONError as e:
    print(f"JSON Error: {e}")
except Exception as e:
    print(f"Unexpected Error: {e}")
```

```typescript
// JavaScript
try {
  const response = await client.sendTask(params);
  // Process response
} catch (error) {
  if (error instanceof A2AError) {
    console.error(`A2A Error: ${error.code} - ${error.message}`);
  } else {
    console.error(`Unexpected Error: ${error}`);
  }
}
```

## Best Practices

1. **Generate Unique Task IDs**: Always use a UUID or similar mechanism to generate unique task IDs
2. **Handle Streaming Properly**: For streaming responses, ensure your code can handle disconnections and reconnections
3. **Implement Timeouts**: Set appropriate timeouts for requests, especially for long-running tasks
4. **Validate Responses**: Always validate responses before processing them
5. **Implement Retry Logic**: For important operations, implement retry logic with exponential backoff
6. **Cache Agent Cards**: Cache agent cards to avoid unnecessary requests
7. **Handle Multi-turn Conversations**: Be prepared to handle tasks that enter the `input-required` state
