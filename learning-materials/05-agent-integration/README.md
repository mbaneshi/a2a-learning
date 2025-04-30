# Integrating A2A with Agent Frameworks

This section explores how to integrate the A2A protocol with various agent frameworks. The A2A repository provides examples for several popular frameworks, demonstrating how to expose their capabilities through the A2A protocol.

## Common Integration Pattern

Regardless of the framework, the integration pattern typically follows these steps:

1. **Create an Agent**: Implement your agent using the framework of your choice
2. **Create a Task Manager**: Implement a custom `TaskManager` that bridges between A2A and your agent
3. **Create an Agent Card**: Define the capabilities and skills of your agent
4. **Set Up the A2A Server**: Configure and start the A2A server with your task manager and agent card

Let's explore how this pattern is applied to different frameworks.

## LangGraph Integration

[LangGraph](https://langchain-ai.github.io/langgraph/) is a library for building stateful, multi-actor applications with LLMs. The A2A repository includes a currency conversion agent built with LangGraph.

### Agent Implementation

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import AIMessage, ToolMessage
import httpx
from typing import Any, Dict, AsyncIterable, Literal
from pydantic import BaseModel

memory = MemorySaver()

@tool
def get_exchange_rate(
    currency_from: str = "USD",
    currency_to: str = "EUR",
    currency_date: str = "latest",
):
    """Use this to get current exchange rate."""
    try:
        response = httpx.get(
            f"https://api.frankfurter.app/{currency_date}",
            params={"from": currency_from, "to": currency_to},
        )
        response.raise_for_status()
        data = response.json()
        if "rates" not in data:
            return {"error": "Invalid API response format."}
        return data
    except Exception as e:
        return {"error": f"API request failed: {e}"}

class ResponseFormat(BaseModel):
    """Respond to the user in this format."""
    status: Literal["input_required", "completed", "error"] = "input_required"
    message: str

class CurrencyAgent:
    SYSTEM_INSTRUCTION = (
        "You are a specialized assistant for currency conversions. "
        "Your sole purpose is to use the 'get_exchange_rate' tool to answer questions about currency exchange rates. "
        # More instructions...
    )
     
    def __init__(self):
        self.model = ChatGoogleGenerativeAI(model="gemini-2.0-flash")
        self.tools = [get_exchange_rate]
        self.graph = create_react_agent(
            self.model, tools=self.tools, checkpointer=memory, 
            prompt=self.SYSTEM_INSTRUCTION, response_format=ResponseFormat
        )

    def invoke(self, query, sessionId) -> str:
        config = {"configurable": {"thread_id": sessionId}}
        self.graph.invoke({"messages": [("user", query)]}, config)        
        return self.get_agent_response(config)

    async def stream(self, query, sessionId) -> AsyncIterable[Dict[str, Any]]:
        inputs = {"messages": [("user", query)]}
        config = {"configurable": {"thread_id": sessionId}}

        for item in self.graph.stream(inputs, config, stream_mode="values"):
            message = item["messages"][-1]
            if isinstance(message, AIMessage) and message.tool_calls:
                yield {
                    "is_task_complete": False,
                    "require_user_input": False,
                    "content": "Looking up the exchange rates...",
                }
            elif isinstance(message, ToolMessage):
                yield {
                    "is_task_complete": False,
                    "require_user_input": False,
                    "content": "Processing the exchange rates..",
                }            
        
        yield self.get_agent_response(config)
        
    def get_agent_response(self, config):
        current_state = self.graph.get_state(config)        
        structured_response = current_state.values.get('structured_response')
        if structured_response and isinstance(structured_response, ResponseFormat): 
            if structured_response.status == "input_required":
                return {
                    "is_task_complete": False,
                    "require_user_input": True,
                    "content": structured_response.message
                }
            elif structured_response.status == "error":
                return {
                    "is_task_complete": False,
                    "require_user_input": True,
                    "content": structured_response.message
                }
            elif structured_response.status == "completed":
                return {
                    "is_task_complete": True,
                    "require_user_input": False,
                    "content": structured_response.message
                }
```

### Task Manager Implementation

```python
class AgentTaskManager(InMemoryTaskManager):
    def __init__(self, agent: CurrencyAgent, notification_sender_auth: PushNotificationSenderAuth):
        super().__init__()
        self.agent = agent
        self.notification_sender_auth = notification_sender_auth

    async def _run_streaming_agent(self, request: SendTaskStreamingRequest):
        task_send_params: TaskSendParams = request.params
        query = self._get_user_query(task_send_params)
        
        if not query:
            return
            
        async for agent_response in self.agent.stream(query, task_send_params.sessionId):
            if agent_response["is_task_complete"]:
                status = TaskStatus(
                    state=TaskState.COMPLETED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text=agent_response["content"])]
                    )
                )
            elif agent_response["require_user_input"]:
                status = TaskStatus(
                    state=TaskState.INPUT_REQUIRED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text=agent_response["content"])]
                    )
                )
            else:
                status = TaskStatus(
                    state=TaskState.WORKING,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text=agent_response["content"])]
                    )
                )
                
            task = await self.update_store(task_send_params.id, status, None)
            
            if await self.has_push_notification_info(task_send_params.id):
                await self.send_task_notification(task)
                
            yield SendTaskStreamingResponse(
                id=request.id,
                result=TaskStatusUpdateEvent(
                    id=task_send_params.id,
                    status=status,
                    final=status.state in [TaskState.COMPLETED, TaskState.FAILED, TaskState.CANCELED]
                )
            )
```

### Server Setup

```python
def main():
    # Parse command-line arguments
    parser = argparse.ArgumentParser(description="Start the Currency Agent server")
    parser.add_argument("--host", type=str, default="0.0.0.0", help="Host to bind to")
    parser.add_argument("--port", type=int, default=10000, help="Port to bind to")
    args = parser.parse_args()
    
    # Define agent capabilities
    capabilities = AgentCapabilities(streaming=True, pushNotifications=True)
    
    # Define agent skills
    skill = AgentSkill(
        id="currency-conversion",
        name="Currency Conversion",
        description="Convert between different currencies and get exchange rates"
    )
    
    # Create agent card
    agent_card = AgentCard(
        name="Currency Agent",
        description="Helps with exchange rates for currencies",
        url=f"http://{args.host}:{args.port}/",
        version="1.0.0",
        defaultInputModes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
        defaultOutputModes=CurrencyAgent.SUPPORTED_CONTENT_TYPES,
        capabilities=capabilities,
        skills=[skill],
    )
    
    # Set up push notification authentication
    notification_sender_auth = PushNotificationSenderAuth()
    notification_sender_auth.generate_jwk()
    
    # Create and start the server
    server = A2AServer(
        agent_card=agent_card,
        task_manager=AgentTaskManager(
            agent=CurrencyAgent(), 
            notification_sender_auth=notification_sender_auth
        ),
        host=args.host,
        port=args.port,
    )
    
    # Add JWKS endpoint for push notification authentication
    server.app.add_route(
        "/.well-known/jwks.json", 
        notification_sender_auth.handle_jwks_endpoint, 
        methods=["GET"]
    )
    
    # Start the server
    server.start()
```

## CrewAI Integration

[CrewAI](https://www.crewai.com/) is a framework for orchestrating role-playing autonomous AI agents. The A2A repository includes an image generation agent built with CrewAI.

### Agent Implementation

```python
class ImageGenerationAgent:
    def __init__(self):
        load_dotenv()
        self.model = LLM(model="gemini-2.0-flash")
        
        self.image_creator_agent = Agent(
            role="Image Creation Expert",
            goal="Generate an image based on the user's text prompt.",
            backstory="You are a digital artist powered by AI.",
            verbose=False,
            allow_delegation=False,
            tools=[generate_image_tool],
            llm=self.model,
        )
        
        self.image_creation_task = Task(
            description=(
                "Receive a user prompt: '{user_prompt}'.\n"
                "Use the 'Image Generator' tool for your image creation."
            ),
            expected_output="The id of the generated image",
            agent=self.image_creator_agent,
        )
        
        self.image_crew = Crew(
            agents=[self.image_creator_agent],
            tasks=[self.image_creation_task],
            process=Process.sequential,
            verbose=False,
        )
    
    def extract_artifact_file_id(self, query):
        # Extract artifact ID from query if present
        try:
            pattern = r'(?:id|artifact-file-id)\s+([0-9a-f]{32})'
            match = re.search(pattern, query)
            return match.group(1) if match else None
        except Exception:
            return None
    
    def invoke(self, query, session_id) -> str:
        """Kickoff CrewAI and return the response."""
        artifact_file_id = self.extract_artifact_file_id(query)
        inputs = {
            "user_prompt": query, 
            "session_id": session_id, 
            "artifact_file_id": artifact_file_id
        }
        response = self.image_crew.kickoff(inputs)
        return response
```

### Task Manager Implementation

```python
class AgentTaskManager(InMemoryTaskManager):
    def __init__(
        self,
        agent: ImageGenerationAgent,
        notification_sender_auth: PushNotificationSenderAuth,
    ):
        super().__init__()
        self.agent = agent
        self.notification_sender_auth = notification_sender_auth
    
    def _parse_agent_outcome(
        self, agent_outcome: dict[str, Any]
    ) -> tuple[TaskStatus, list[Artifact]]:
        """Parses the dictionary output from agent.invoke() into A2A TaskStatus and Artifacts."""
        is_task_complete = agent_outcome["is_task_complete"]
        require_user_input = not is_task_complete
        data = agent_outcome.get("data", {})
        text_parts = agent_outcome.get("text_parts", [])
        
        parts = []
        parts.extend(text_parts)
        
        if data:
            parts.append(DataPart(type="data", data=data))
        
        message = Message(role="agent", parts=parts)
        
        if require_user_input:
            status = TaskStatus(state=TaskState.INPUT_REQUIRED, message=message)
        else:
            status = TaskStatus(state=TaskState.COMPLETED, message=message)
        
        artifacts = []
        if "image_id" in agent_outcome:
            image_id = agent_outcome["image_id"]
            if image_id and image_id != -999999999:
                image_data = InMemoryCache.get(agent_outcome["session_id"], image_id)
                if image_data:
                    artifacts.append(
                        Artifact(
                            name="generated-image",
                            parts=[
                                FilePart(
                                    type="file",
                                    file=FileContent(
                                        name=image_data.name,
                                        mimeType="image/png",
                                        bytes=image_data.bytes,
                                    ),
                                )
                            ],
                        )
                    )
        
        return status, artifacts
```

## Google ADK Integration

[Google ADK](https://github.com/google-research/google-adk) is the Agent Development Kit from Google. The A2A repository includes an expense reimbursement agent built with Google ADK.

### Agent Implementation

```python
class ReimbursementAgent:
    """An agent that handles reimbursement requests."""

    SUPPORTED_CONTENT_TYPES = ["text", "text/plain"]

    def __init__(self):
        self._agent = self._build_agent()
        self._user_id = "remote_agent"
        self._runner = Runner(
            app_name=self._agent.name,
            agent=self._agent,
            artifact_service=InMemoryArtifactService(),
            session_service=InMemorySessionService(),
            memory_service=InMemoryMemoryService(),
        )

    def invoke(self, query, session_id) -> str:
        session = self._runner.session_service.get_session(
            app_name=self._agent.name, user_id=self._user_id, session_id=session_id
        )
        content = types.Content(
            role="user", parts=[types.Part.from_text(text=query)]
        )
        if session is None:
            session = self._runner.session_service.create_session(
                app_name=self._agent.name,
                user_id=self._user_id,
                state={},
                session_id=session_id,
            )
        events = list(self._runner.run(
            user_id=self._user_id, session_id=session.id, new_message=content
        ))
        if not events or not events[-1].content or not events[-1].content.parts:
            return ""
        return "\n".join([p.text for p in events[-1].content.parts if p.text])

    async def stream(self, query, session_id) -> AsyncIterable[Dict[str, Any]]:
        session = self._runner.session_service.get_session(
            app_name=self._agent.name, user_id=self._user_id, session_id=session_id
        )
        content = types.Content(
            role="user", parts=[types.Part.from_text(text=query)]
        )
        if session is None:
            session = self._runner.session_service.create_session(
                app_name=self._agent.name,
                user_id=self._user_id,
                state={},
                session_id=session_id,
            )
        async for event in self._runner.run_async(
            user_id=self._user_id, session_id=session.id, new_message=content
        ):
            if event.is_final_response():
                response = ""
                if event.content and event.content.parts:
                    if any(p.text for p in event.content.parts):
                        response = "\n".join([p.text for p in event.content.parts if p.text])
                    elif any(p.function_response for p in event.content.parts):
                        response = next((p.function_response.model_dump() for p in event.content.parts))
                yield {
                    "is_task_complete": True,
                    "require_user_input": False,
                    "content": response,
                }
            elif event.is_awaiting_user_input():
                yield {
                    "is_task_complete": False,
                    "require_user_input": True,
                    "content": event.content.parts[0].text if event.content and event.content.parts else "",
                }
            else:
                yield {
                    "is_task_complete": False,
                    "require_user_input": False,
                    "content": event.content.parts[0].text if event.content and event.content.parts else "",
                }
```

## Genkit Integration (JavaScript)

[Genkit](https://genkit.dev/) is a framework for building AI applications with Google's Gemini models. The A2A repository includes a movie information agent built with Genkit.

### Agent Implementation

```typescript
// genkit.ts
import { googleAI, gemini20Flash } from "@genkit-ai/googleai";
import { genkit } from "genkit";
import { dirname } from "path";
import { fileURLToPath } from "url";

export const ai = genkit({
  plugins: [googleAI()],
  model: gemini20Flash,
  promptDir: dirname(fileURLToPath(import.meta.url)),
});

export { z } from "genkit";

// agent.ts
import { ai, z } from "./genkit.js";
import { searchMovies, getMovieDetails } from "./tmdb.js";

// Define the schema for movie search results
const MovieSearchResult = z.object({
  id: z.number(),
  title: z.string(),
  overview: z.string().optional(),
  release_date: z.string().optional(),
  poster_path: z.string().optional(),
});

// Define the schema for movie details
const MovieDetails = z.object({
  id: z.number(),
  title: z.string(),
  overview: z.string(),
  release_date: z.string(),
  runtime: z.number().optional(),
  genres: z.array(z.object({ name: z.string() })),
  vote_average: z.number(),
  poster_path: z.string().optional(),
  backdrop_path: z.string().optional(),
});

// Define tools for the agent
export const movieAgent = ai.agent({
  name: "MovieAgent",
  description: "An agent that can search for movies and provide information about them.",
  tools: {
    searchMovies: {
      description: "Search for movies by title or keywords",
      parameters: z.object({
        query: z.string().describe("The movie title or keywords to search for"),
      }),
      execute: async ({ query }) => {
        const results = await searchMovies(query);
        return results.map((movie) => MovieSearchResult.parse(movie));
      },
    },
    getMovieDetails: {
      description: "Get detailed information about a specific movie",
      parameters: z.object({
        movieId: z.number().describe("The ID of the movie to get details for"),
      }),
      execute: async ({ movieId }) => {
        const details = await getMovieDetails(movieId);
        return MovieDetails.parse(details);
      },
    },
  },
});
```

### Task Handler Implementation

```typescript
import { movieAgent } from "./agent.js";
import { TaskHandler } from "../../server/handler.js";

export const movieAgentHandler: TaskHandler = async function* (context) {
  const { userMessage } = context;
  
  // Extract the user's query
  const query = userMessage.parts[0].text;
  
  // Yield a working status
  yield {
    state: "working",
    message: {
      role: "agent",
      parts: [{ type: "text", text: "Searching for movie information..." }]
    }
  };
  
  try {
    // Run the agent
    const result = await movieAgent.run(query);
    
    // Yield the completed status with the agent's response
    yield {
      state: "completed",
      message: {
        role: "agent",
        parts: [{ type: "text", text: result.text }]
      }
    };
    
    // If the agent found movie data, yield it as an artifact
    if (result.toolResults && result.toolResults.length > 0) {
      yield {
        name: "movie-data",
        parts: [
          {
            type: "data",
            data: result.toolResults.map(tr => tr.result)
          }
        ]
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

### Server Setup

```typescript
import { A2AServer } from "../../server/server.js";
import { movieAgentHandler } from "./handler.js";

// Define the agent card
const movieAgentCard = {
  name: "Movie Information Agent",
  description: "An agent that can search for movies and provide information about them",
  url: "http://localhost:3000/",
  version: "1.0.0",
  capabilities: { streaming: true },
  skills: [
    {
      id: "movie-search",
      name: "Movie Search",
      description: "Search for movies by title or keywords"
    },
    {
      id: "movie-details",
      name: "Movie Details",
      description: "Get detailed information about a specific movie"
    }
  ]
};

// Create and start the server
const server = new A2AServer(movieAgentHandler, {
  card: movieAgentCard,
  basePath: "/",
  cors: true
});

const port = process.env.PORT ? parseInt(process.env.PORT) : 3000;
server.listen(port);
```

## Integration Best Practices

1. **Map Framework States to A2A States**: Ensure your framework's states map correctly to A2A task states
2. **Handle Streaming Properly**: If your framework supports streaming, map it to A2A's streaming capabilities
3. **Manage Session State**: Use the `sessionId` to maintain conversation state between requests
4. **Handle Errors Gracefully**: Catch and handle framework-specific errors, mapping them to appropriate A2A error responses
5. **Support Content Types**: Ensure your integration can handle all content types supported by your framework
6. **Document Skills Clearly**: Clearly document your agent's skills in the agent card
7. **Test Thoroughly**: Test your integration with various request types and edge cases
8. **Monitor Performance**: Monitor the performance of your integration and optimize as needed
9. **Keep Dependencies Updated**: Keep your framework and A2A dependencies updated
10. **Follow Framework Best Practices**: Follow the best practices of your chosen framework

In the next section, we'll explore practical examples of A2A in action.
