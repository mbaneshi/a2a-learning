# A2A Protocol Specification

The A2A protocol is defined using JSON Schema, which provides a clear, machine-readable specification for all protocol structures. This section explores the core components of the protocol specification.

## JSON-RPC Foundation

A2A is built on [JSON-RPC 2.0](https://www.jsonrpc.org/specification), a lightweight remote procedure call protocol using JSON for data encoding. This provides a standardized way to structure requests and responses.

### Basic JSON-RPC Structure

```json
// Request
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "method": "tasks/send",
  "params": {
    // Method-specific parameters
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "result": {
    // Method-specific result
  }
}

// Error Response
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "error": {
    "code": -32600,
    "message": "Invalid Request"
  }
}
```

## Core Data Types

### Message

Messages represent communication between the user and agent:

```json
{
  "role": "user",
  "parts": [
    {
      "type": "text",
      "text": "Hello, can you help me with currency conversion?"
    }
  ],
  "metadata": {
    "conversation_id": "abc123"
  }
}
```

### Parts

Parts are the content elements within messages:

1. **TextPart**: Simple text content
   ```json
   {
     "type": "text",
     "text": "Hello, world!"
   }
   ```

2. **FilePart**: File content (binary data or URI)
   ```json
   {
     "type": "file",
     "file": {
       "name": "image.png",
       "mimeType": "image/png",
       "bytes": "base64-encoded-data"
     }
   }
   ```

3. **DataPart**: Structured data (forms, JSON, etc.)
   ```json
   {
     "type": "data",
     "data": {
       "amount": 100,
       "currency": "USD"
     }
   }
   ```

### Task

A Task represents a unit of work being processed by an agent:

```json
{
  "id": "task-123",
  "sessionId": "session-456",
  "status": {
    "state": "working",
    "message": {
      "role": "agent",
      "parts": [
        {
          "type": "text",
          "text": "I'm working on your request..."
        }
      ]
    },
    "timestamp": "2023-04-01T12:34:56Z"
  },
  "artifacts": [
    {
      "name": "conversion-result",
      "parts": [
        {
          "type": "data",
          "data": {
            "from": "USD",
            "to": "EUR",
            "rate": 0.92,
            "amount": 100,
            "converted": 92
          }
        }
      ]
    }
  ],
  "history": [
    // Array of Message objects representing conversation history
  ],
  "metadata": {
    "custom_field": "value"
  }
}
```

### TaskStatus

TaskStatus represents the current state of a task:

```json
{
  "state": "input-required",
  "message": {
    "role": "agent",
    "parts": [
      {
        "type": "text",
        "text": "What currency would you like to convert to?"
      }
    ]
  },
  "timestamp": "2023-04-01T12:35:00Z"
}
```

### Artifact

Artifacts are outputs produced by an agent during task processing:

```json
{
  "name": "generated-image",
  "description": "An image of a sunset over mountains",
  "parts": [
    {
      "type": "file",
      "file": {
        "name": "sunset.png",
        "mimeType": "image/png",
        "bytes": "base64-encoded-data"
      }
    }
  ],
  "metadata": {
    "generation_params": {
      "prompt": "sunset over mountains",
      "style": "photorealistic"
    }
  }
}
```

## A2A Methods

The A2A protocol defines several methods for task management:

### tasks/send

Sends a task to an agent for processing.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "method": "tasks/send",
  "params": {
    "id": "task-123",
    "sessionId": "session-456",
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "text",
          "text": "Convert 100 USD to EUR"
        }
      ]
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "result": {
    "id": "task-123",
    "sessionId": "session-456",
    "status": {
      "state": "completed",
      "message": {
        "role": "agent",
        "parts": [
          {
            "type": "text",
            "text": "100 USD is approximately 92 EUR based on current exchange rates."
          }
        ]
      },
      "timestamp": "2023-04-01T12:36:00Z"
    },
    "artifacts": [
      {
        "name": "conversion-result",
        "parts": [
          {
            "type": "data",
            "data": {
              "from": "USD",
              "to": "EUR",
              "rate": 0.92,
              "amount": 100,
              "converted": 92
            }
          }
        ]
      }
    ]
  }
}
```

### tasks/sendSubscribe

Similar to `tasks/send` but establishes a streaming connection for real-time updates.

**Request:** Same as `tasks/send`

**Response:** Server-Sent Events (SSE) stream containing:

```
event: data
data: {"id":"task-123","status":{"state":"working","timestamp":"2023-04-01T12:34:56Z"}}

event: data
data: {"id":"task-123","artifact":{"name":"conversion-result","parts":[{"type":"data","data":{"from":"USD","to":"EUR","rate":0.92,"amount":100,"converted":92}}]}}

event: data
data: {"id":"task-123","status":{"state":"completed","message":{"role":"agent","parts":[{"type":"text","text":"100 USD is approximately 92 EUR based on current exchange rates."}]},"timestamp":"2023-04-01T12:36:00Z"},"final":true}
```

### tasks/get

Retrieves the current state of a task.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-456",
  "method": "tasks/get",
  "params": {
    "id": "task-123"
  }
}
```

**Response:** Similar to `tasks/send` response

### tasks/cancel

Attempts to cancel a running task.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-789",
  "method": "tasks/cancel",
  "params": {
    "id": "task-123"
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-789",
  "result": {
    "id": "task-123",
    "status": {
      "state": "canceled",
      "timestamp": "2023-04-01T12:37:00Z"
    }
  }
}
```

### tasks/pushNotification/set

Configures push notifications for a task.

**Request:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-abc",
  "method": "tasks/pushNotification/set",
  "params": {
    "id": "task-123",
    "pushNotification": {
      "url": "https://example.com/webhook",
      "authentication": {
        "schemes": ["bearer"],
        "credentials": "jwt-token"
      }
    }
  }
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "id": "request-abc",
  "result": {
    "id": "task-123",
    "pushNotification": {
      "url": "https://example.com/webhook"
    }
  }
}
```

## Agent Card

The Agent Card is a JSON document that describes an agent's capabilities and is typically served at `/.well-known/agent.json`:

```json
{
  "name": "Currency Conversion Agent",
  "description": "Helps with currency conversion and exchange rates",
  "url": "https://currency-agent.example.com/",
  "version": "1.0.0",
  "provider": {
    "organization": "Example Corp"
  },
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "authentication": {
    "schemes": ["bearer"]
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text", "data"],
  "skills": [
    {
      "id": "currency-conversion",
      "name": "Currency Conversion",
      "description": "Convert between different currencies",
      "examples": [
        "Convert 100 USD to EUR",
        "What's the exchange rate between JPY and GBP?"
      ]
    }
  ]
}
```

## Error Handling

A2A defines standard error codes following the JSON-RPC specification:

- `-32700`: Parse error
- `-32600`: Invalid Request
- `-32601`: Method not found
- `-32602`: Invalid params
- `-32603`: Internal error
- `-32000` to `-32099`: Server error

Custom A2A error codes include:

- `404`: Task not found
- `405`: Task not cancelable
- `406`: Push notifications not supported
