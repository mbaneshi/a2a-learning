# Introduction to the Agent2Agent (A2A) Protocol

## What is A2A?

The Agent2Agent (A2A) protocol is an open protocol initiated by Google designed to enable communication and interoperability between disparate AI agent systems. The core goal is to allow agents built on different frameworks (e.g., LangGraph, CrewAI, Google ADK, Genkit) or by different vendors to discover each other's capabilities, negotiate interaction modes, and collaborate on tasks.

## Why A2A Matters

One of the biggest challenges in enterprise AI adoption is getting agents built on different frameworks and vendors to work together. A2A addresses this challenge by providing:

1. **Standardized Communication**: A common language for agents to communicate regardless of their underlying implementation
2. **Capability Discovery**: Mechanisms for agents to advertise and discover each other's capabilities
3. **Flexible Interaction**: Support for various content types (text, files, structured data)
4. **Security**: Built-in authentication and authorization mechanisms

## Core Concepts

### Agent Card

An Agent Card is a public metadata file (usually at `/.well-known/agent.json`) that describes an agent's capabilities, skills, endpoint URL, and authentication requirements. Clients use this for discovery.

```json
{
  "name": "Currency Agent",
  "description": "Helps with exchange rates for currencies",
  "url": "http://localhost:10000/",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "currency-conversion",
      "name": "Currency Conversion",
      "description": "Convert between different currencies"
    }
  ]
}
```

### A2A Server

An A2A Server is an agent exposing an HTTP endpoint that implements the A2A protocol methods. It receives requests and manages task execution.

### A2A Client

An A2A Client is an application or another agent that consumes A2A services. It sends requests (like `tasks/send`) to an A2A Server's URL.

### Tasks

Tasks are the primary unit of work in A2A. A task represents a request from a client to an agent, along with the agent's response and any artifacts produced during processing.

### Messages

Messages are exchanged between clients and servers as part of tasks. They contain parts (text, files, data) and have a role (user or agent).

### Task States

Tasks progress through various states:
- `submitted`: Initial state when a task is sent
- `working`: The agent is processing the task
- `input-required`: The agent needs more information from the client
- `completed`: The task has been successfully completed
- `canceled`: The task was canceled by the client
- `failed`: The task failed to complete

### Streaming

For long-running tasks, servers supporting the `streaming` capability can use `tasks/sendSubscribe`. The client receives Server-Sent Events (SSE) containing `TaskStatusUpdateEvent` or `TaskArtifactUpdateEvent` messages, providing real-time progress.

### Push Notifications

Servers supporting `pushNotifications` can proactively send task updates to a client-provided webhook URL, configured via `tasks/pushNotification/set`.

## Typical Flow

1. **Discovery**: Client fetches the Agent Card from the server's well-known URL
2. **Initiation**: Client sends a `tasks/send` or `tasks/sendSubscribe` request containing the initial user message and a unique Task ID
3. **Processing**:
   - **(Streaming)**: Server sends SSE events (status updates, artifacts) as the task progresses
   - **(Non-Streaming)**: Server processes the task synchronously and returns the final `Task` object in the response
4. **Interaction (Optional)**: If the task enters `input-required`, the client sends subsequent messages using the same Task ID via `tasks/send` or `tasks/sendSubscribe`
5. **Completion**: The task eventually reaches a terminal state (`completed`, `failed`, `canceled`)

## Key Benefits

- **Framework Agnostic**: Works with any agent framework or implementation
- **Language Agnostic**: Implementations available in Python, JavaScript, and potentially other languages
- **Extensible**: Designed to evolve with new capabilities and interaction modes
- **Enterprise Ready**: Includes authentication, security, and scalability features

In the next sections, we'll dive deeper into the protocol specification, client and server implementations, and how to integrate A2A with various agent frameworks.
