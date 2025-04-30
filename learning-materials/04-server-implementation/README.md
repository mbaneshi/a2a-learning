# A2A Server Implementation

This section covers how to implement A2A servers in both Python and JavaScript. A2A servers expose agent functionality through the A2A protocol, handling requests, managing tasks, and sending responses.

## Python Server Implementation

The A2A Python server is implemented in the `common/server/server.py` file. It provides a Starlette-based HTTP server that implements the A2A protocol.

### Basic Structure

```python
class A2AServer:
    def __init__(
        self,
        host="0.0.0.0",
        port=5000,
        endpoint="/",
        agent_card: AgentCard = None,
        task_manager: TaskManager = None,
    ):
        self.host = host
        self.port = port
        self.endpoint = endpoint
        self.task_manager = task_manager
        self.agent_card = agent_card
        self.app = Starlette()
        self.app.add_route(self.endpoint, self._process_request, methods=["POST"])
        self.app.add_route(
            "/.well-known/agent.json", self._get_agent_card, methods=["GET"]
        )

    def start(self):
        if self.agent_card is None:
            raise ValueError("agent_card is not defined")

        if self.task_manager is None:
            raise ValueError("request_handler is not defined")

        import uvicorn
        uvicorn.run(self.app, host=self.host, port=self.port)

    def _get_agent_card(self, request: Request) -> JSONResponse:
        return JSONResponse(self.agent_card.model_dump(exclude_none=True))

    async def _process_request(self, request: Request):
        try:
            body = await request.json()
            json_rpc_request = A2ARequest.validate_python(body)

            if isinstance(json_rpc_request, GetTaskRequest):
                result = await self.task_manager.on_get_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskRequest):
                result = await self.task_manager.on_send_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskStreamingRequest):
                result = await self.task_manager.on_send_task_subscribe(
                    json_rpc_request
                )
            # Other request types...

            return self._create_response(result)
        except Exception as e:
            # Error handling...
```

### Task Manager

The server delegates task processing to a `TaskManager` implementation. The base `TaskManager` class is defined in `common/server/task_manager.py`:

```python
class TaskManager(ABC):
    @abstractmethod
    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        pass

    @abstractmethod
    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        pass

    @abstractmethod
    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        pass

    @abstractmethod
    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass

    @abstractmethod
    async def on_set_task_push_notification(
        self, request: SetTaskPushNotificationRequest
    ) -> SetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_get_task_push_notification(
        self, request: GetTaskPushNotificationRequest
    ) -> GetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_resubscribe_to_task(
        self, request: TaskResubscriptionRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass
```

A default implementation, `InMemoryTaskManager`, is provided that handles task storage and basic operations:

```python
class InMemoryTaskManager(TaskManager):
    def __init__(self):
        self.tasks: dict[str, Task] = {}
        self.push_notification_infos: dict[str, PushNotificationConfig] = {}
        self.lock = asyncio.Lock()
        self.task_sse_subscribers: dict[str, List[asyncio.Queue]] = {}
        self.subscriber_lock = asyncio.Lock()

    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        # Implementation...

    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        # Implementation...

    # Other methods...

    async def upsert_task(self, task_send_params: TaskSendParams) -> Task:
        # Implementation...

    async def update_store(
        self, task_id: str, status: TaskStatus, artifacts: list[Artifact]
    ) -> Task:
        # Implementation...
```

### Creating a Custom Task Manager

To implement a custom agent, you typically extend `InMemoryTaskManager` and override the necessary methods:

```python
class MyAgentTaskManager(InMemoryTaskManager):
    def __init__(self, agent: MyAgent):
        super().__init__()
        self.agent = agent

    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        # Validate the request
        validation_error = self._validate_request(request)
        if validation_error:
            return SendTaskResponse(id=request.id, error=validation_error.error)

        # Store the task
        await self.upsert_task(request.params)
        
        # Update task status to WORKING
        task = await self.update_store(
            request.params.id, TaskStatus(state=TaskState.WORKING), None
        )

        try:
            # Extract the query from the request
            query = request.params.message.parts[0].text
            
            # Invoke the agent
            agent_response = self.agent.invoke(query, request.params.sessionId)
            
            # Process the agent's response
            if agent_response["is_task_complete"]:
                # Task is complete
                status = TaskStatus(
                    state=TaskState.COMPLETED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text=agent_response["content"])]
                    )
                )
            else:
                # Agent needs more information
                status = TaskStatus(
                    state=TaskState.INPUT_REQUIRED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text=agent_response["content"])]
                    )
                )
            
            # Update the task with the new status
            task = await self.update_store(request.params.id, status, None)
            
            # Return the response
            return SendTaskResponse(id=request.id, result=task)
        except Exception as e:
            # Handle errors
            error_status = TaskStatus(
                state=TaskState.FAILED,
                message=Message(
                    role="agent",
                    parts=[TextPart(text=f"Error: {str(e)}")]
                )
            )
            task = await self.update_store(request.params.id, error_status, None)
            return SendTaskResponse(id=request.id, result=task)
```

### Setting Up the Server

To set up and start the server:

```python
def main():
    # Create an agent card
    agent_card = AgentCard(
        name="My Agent",
        description="A sample agent",
        url="http://localhost:5000/",
        version="1.0.0",
        capabilities=AgentCapabilities(streaming=True),
        skills=[
            AgentSkill(
                id="sample-skill",
                name="Sample Skill",
                description="A sample skill"
            )
        ]
    )
    
    # Create a task manager
    task_manager = MyAgentTaskManager(agent=MyAgent())
    
    # Create and start the server
    server = A2AServer(
        agent_card=agent_card,
        task_manager=task_manager,
        host="0.0.0.0",
        port=5000
    )
    
    server.start()

if __name__ == "__main__":
    main()
```

## JavaScript Server Implementation

The A2A JavaScript server is implemented in the `samples/js/src/server/server.ts` file. It provides an Express-based HTTP server that implements the A2A protocol.

### Basic Structure

```typescript
export class A2AServer {
  private taskHandler: TaskHandler;
  private taskStore: TaskStore;
  private corsOptions: CorsOptions | boolean | string;
  private basePath: string;
  private activeCancellations: Set<string> = new Set();
  card: schema.AgentCard;

  constructor(
    taskHandler: TaskHandler,
    options: A2AServerOptions = {}
  ) {
    this.taskHandler = taskHandler;
    this.taskStore = options.taskStore || new InMemoryTaskStore();
    this.corsOptions = options.cors || true;
    this.basePath = options.basePath || "/";
    this.card = options.card || {
      name: "Default A2A Agent",
      description: "A default A2A agent implementation",
      url: "http://localhost:3000/",
      version: "1.0.0",
      capabilities: { streaming: true },
      skills: []
    };
  }

  listen(port: number): Express {
    const app = express();
    
    // Configure middleware
    app.use(cors(this.corsOptions));
    app.use(bodyParser.json());
    
    // Agent card endpoint
    app.get("/.well-known/agent.json", (req, res) => {
      res.json(this.card);
    });
    
    // Main A2A endpoint
    app.post(this.basePath, this.endpoint());
    
    // Error handler
    app.use(this.errorHandler);
    
    // Start listening
    app.listen(port, () => {
      console.log(`A2A Server listening on port ${port} at path ${this.basePath}`);
    });
    
    return app;
  }

  endpoint(): RequestHandler {
    return async (req: Request, res: Response, next: NextFunction) => {
      const requestBody = req.body;
      
      try {
        // Validate request
        if (!this.isValidJsonRpcRequest(requestBody)) {
          throw A2AError.invalidRequest("Invalid JSON-RPC request structure.");
        }
        
        // Route based on method
        switch (requestBody.method) {
          case "tasks/send":
            await this.handleTaskSend(requestBody as schema.SendTaskRequest, res);
            break;
          case "tasks/sendSubscribe":
            await this.handleTaskSendSubscribe(requestBody as schema.SendTaskStreamingRequest, res);
            break;
          // Other methods...
          default:
            throw A2AError.methodNotFound(requestBody.method);
        }
      } catch (error) {
        next(error);
      }
    };
  }

  // Method handlers...
}
```

### Task Handler

The JavaScript implementation uses a `TaskHandler` function to process tasks:

```typescript
export type TaskHandler = (
  context: TaskContext
) => AsyncGenerator<TaskYieldUpdate, schema.Task | void, unknown>;
```

A `TaskContext` provides information about the task:

```typescript
export interface TaskContext {
  task: schema.Task;
  userMessage: schema.Message;
  isCancelled(): boolean;
  history?: schema.Message[];
}
```

### Task Store

The JavaScript implementation uses a `TaskStore` interface to manage task storage:

```typescript
export interface TaskStore {
  save(data: TaskAndHistory): Promise<void>;
  load(taskId: string): Promise<TaskAndHistory | null>;
}
```

Two implementations are provided:
- `InMemoryTaskStore`: Stores tasks in memory
- `FileStore`: Stores tasks on disk

### Creating a Custom Task Handler

To implement a custom agent, you create a task handler function:

```typescript
const myTaskHandler: TaskHandler = async function* (context: TaskContext) {
  const { task, userMessage } = context;
  
  // Yield a working status
  yield {
    state: "working",
    message: {
      role: "agent",
      parts: [{ type: "text", text: "I'm working on your request..." }]
    }
  };
  
  try {
    // Extract the query
    const query = userMessage.parts[0].text;
    
    // Process the query (replace with your agent logic)
    const response = await processQuery(query);
    
    // Yield the final result
    yield {
      state: "completed",
      message: {
        role: "agent",
        parts: [{ type: "text", text: response.text }]
      }
    };
    
    // If the agent produced data, yield it as an artifact
    if (response.data) {
      yield {
        name: "result-data",
        parts: [{ type: "data", data: response.data }]
      };
    }
  } catch (error) {
    // Handle errors
    yield {
      state: "failed",
      message: {
        role: "agent",
        parts: [{ type: "text", text: `Error: ${error.message}` }]
      }
    };
  }
};
```

### Setting Up the Server

To set up and start the server:

```typescript
import { A2AServer } from "./server";
import { myTaskHandler } from "./my-agent";

// Create an agent card
const agentCard = {
  name: "My Agent",
  description: "A sample agent",
  url: "http://localhost:3000/",
  version: "1.0.0",
  capabilities: { streaming: true },
  skills: [
    {
      id: "sample-skill",
      name: "Sample Skill",
      description: "A sample skill"
    }
  ]
};

// Create and start the server
const server = new A2AServer(myTaskHandler, {
  card: agentCard,
  basePath: "/",
  cors: true
});

server.listen(3000);
```

## Handling Different Content Types

A2A servers should be able to handle different content types:

### Text

```python
# Python
message = Message(
    role="agent",
    parts=[TextPart(text="This is a text response")]
)
```

```typescript
// JavaScript
const message = {
  role: "agent",
  parts: [{ type: "text", text: "This is a text response" }]
};
```

### Files

```python
# Python
with open("image.png", "rb") as f:
    file_bytes = base64.b64encode(f.read()).decode("utf-8")

message = Message(
    role="agent",
    parts=[
        FilePart(
            file=FileContent(
                name="image.png",
                mimeType="image/png",
                bytes=file_bytes
            )
        )
    ]
)
```

```typescript
// JavaScript
import fs from "fs";

const fileBytes = fs.readFileSync("image.png").toString("base64");

const message = {
  role: "agent",
  parts: [
    {
      type: "file",
      file: {
        name: "image.png",
        mimeType: "image/png",
        bytes: fileBytes
      }
    }
  ]
};
```

### Structured Data

```python
# Python
message = Message(
    role="agent",
    parts=[
        DataPart(
            data={
                "from": "USD",
                "to": "EUR",
                "rate": 0.92,
                "amount": 100,
                "converted": 92
            }
        )
    ]
)
```

```typescript
// JavaScript
const message = {
  role: "agent",
  parts: [
    {
      type: "data",
      data: {
        from: "USD",
        to: "EUR",
        rate: 0.92,
        amount: 100,
        converted: 92
      }
    }
  ]
};
```

## Push Notifications

A2A servers can send push notifications to clients:

```python
# Python
async def send_push_notification(task_id: str, update: Union[TaskStatus, Artifact]):
    async with self.lock:
        if task_id not in self.push_notification_infos:
            return
        
        notification_config = self.push_notification_infos[task_id]
        
        # Create the appropriate event
        if isinstance(update, TaskStatus):
            event = TaskStatusUpdateEvent(id=task_id, status=update)
        else:
            event = TaskArtifactUpdateEvent(id=task_id, artifact=update)
        
        # Send the notification
        async with httpx.AsyncClient() as client:
            headers = {}
            if notification_config.authentication:
                if "bearer" in notification_config.authentication.schemes:
                    token = await self.notification_sender_auth.generate_jwt()
                    headers["Authorization"] = f"Bearer {token}"
            
            try:
                response = await client.post(
                    notification_config.url,
                    json=event.model_dump(),
                    headers=headers
                )
                response.raise_for_status()
            except Exception as e:
                logger.error(f"Failed to send push notification: {e}")
```

## Best Practices

1. **Use Asynchronous Code**: Use async/await for all I/O operations to ensure good performance
2. **Implement Proper Locking**: Use locks to protect shared resources
3. **Handle Errors Gracefully**: Catch and handle exceptions, returning appropriate error responses
4. **Validate Requests**: Always validate incoming requests before processing them
5. **Implement Timeouts**: Set appropriate timeouts for agent operations
6. **Support Streaming**: Implement streaming for better user experience with long-running tasks
7. **Implement Cancellation**: Allow tasks to be canceled
8. **Use Proper Logging**: Log important events and errors
9. **Secure Your Server**: Implement authentication and authorization as needed
10. **Test Thoroughly**: Test your server with various request types and edge cases

In the next section, we'll explore how to integrate A2A with various agent frameworks.
