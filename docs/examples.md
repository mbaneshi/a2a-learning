# A2A Protocol Examples

This section provides practical examples of using the A2A protocol in different scenarios. These examples demonstrate how to use A2A clients and servers to build agent-based applications.

## Example 1: Basic Text Conversation

This example demonstrates a simple text-based conversation between a client and an agent.

### Client Code (Python)

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, Message

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID and session ID
    task_id = str(uuid.uuid4())
    session_id = str(uuid.uuid4())
    
    # Prepare the initial message
    payload = {
        "id": task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Hello, can you help me convert 100 USD to EUR?")]
        )
    }
    
    # Send the task and get the response
    response = await client.send_task(payload)
    
    # Print the response
    print(f"Task ID: {response.result.id}")
    print(f"Status: {response.result.status.state}")
    if response.result.status.message:
        print(f"Agent: {response.result.status.message.parts[0].text}")
    
    # If the agent needs more information, continue the conversation
    if response.result.status.state == "input-required":
        # Prepare the follow-up message
        follow_up_payload = {
            "id": task_id,
            "sessionId": session_id,
            "message": Message(
                role="user",
                parts=[TextPart(text="Yes, please use today's exchange rate.")]
            )
        }
        
        # Send the follow-up message
        follow_up_response = await client.send_task(follow_up_payload)
        
        # Print the follow-up response
        print(f"Status: {follow_up_response.result.status.state}")
        if follow_up_response.result.status.message:
            print(f"Agent: {follow_up_response.result.status.message.parts[0].text}")
        
        # Print any artifacts
        if follow_up_response.result.artifacts:
            for artifact in follow_up_response.result.artifacts:
                print(f"Artifact: {artifact.name}")
                for part in artifact.parts:
                    if part.type == "data":
                        print(f"Data: {part.data}")

asyncio.run(main())
```

### Expected Output

```
Task ID: 550e8400-e29b-41d4-a716-446655440000
Status: input-required
Agent: I'd be happy to help you convert 100 USD to EUR. Do you want me to use today's exchange rate?
Status: completed
Agent: Based on today's exchange rate, 100 USD is approximately 92.34 EUR.
Artifact: conversion-result
Data: {'from': 'USD', 'to': 'EUR', 'rate': 0.9234, 'amount': 100, 'converted': 92.34}
```

## Example 2: Streaming Updates

This example demonstrates how to use streaming to receive real-time updates from an agent.

### Client Code (Python)

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, Message

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID and session ID
    task_id = str(uuid.uuid4())
    session_id = str(uuid.uuid4())
    
    # Prepare the initial message
    payload = {
        "id": task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Generate an image of a sunset over mountains")]
        )
    }
    
    # Send the task with streaming and process updates
    print("Streaming updates:")
    async for update in client.send_task_streaming(payload):
        if hasattr(update.result, "status"):
            print(f"Status: {update.result.status.state}")
            if update.result.status.message:
                print(f"Message: {update.result.status.message.parts[0].text}")
            if update.result.final:
                print("Final update received")
        elif hasattr(update.result, "artifact"):
            print(f"Artifact: {update.result.artifact.name}")
            for part in update.result.artifact.parts:
                if part.type == "file":
                    print(f"File: {part.file.name} ({part.file.mimeType})")
                    # Save the file
                    import base64
                    with open(part.file.name, "wb") as f:
                        f.write(base64.b64decode(part.file.bytes))
                    print(f"Saved file to {part.file.name}")

asyncio.run(main())
```

### Expected Output

```
Streaming updates:
Status: working
Message: I'll generate an image of a sunset over mountains for you. Please wait a moment...
Status: working
Message: Creating your image now...
Artifact: generated-image
File: sunset-mountains.png (image/png)
Saved file to sunset-mountains.png
Status: completed
Message: Here's your image of a sunset over mountains. The warm orange and red hues contrast beautifully with the silhouette of the mountain range.
Final update received
```

## Example 3: File Upload and Processing

This example demonstrates how to upload a file to an agent for processing.

### Client Code (Python)

```python
import asyncio
import uuid
import base64
from common.client.client import A2AClient
from common.types import TextPart, FilePart, FileContent, Message

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID and session ID
    task_id = str(uuid.uuid4())
    session_id = str(uuid.uuid4())
    
    # Read the file and encode it as base64
    with open("document.pdf", "rb") as f:
        file_bytes = base64.b64encode(f.read()).decode("utf-8")
    
    # Prepare the message with file and text
    payload = {
        "id": task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[
                TextPart(text="Please analyze this document and summarize its contents."),
                FilePart(
                    file=FileContent(
                        name="document.pdf",
                        mimeType="application/pdf",
                        bytes=file_bytes
                    )
                )
            ]
        )
    }
    
    # Send the task and get the response
    response = await client.send_task(payload)
    
    # Print the response
    print(f"Task ID: {response.result.id}")
    print(f"Status: {response.result.status.state}")
    if response.result.status.message:
        print(f"Agent: {response.result.status.message.parts[0].text}")
    
    # Print any artifacts
    if response.result.artifacts:
        for artifact in response.result.artifacts:
            print(f"Artifact: {artifact.name}")
            for part in artifact.parts:
                if part.type == "text":
                    print(f"Text: {part.text}")
                elif part.type == "data":
                    print(f"Data: {part.data}")

asyncio.run(main())
```

### Expected Output

```
Task ID: 550e8400-e29b-41d4-a716-446655440000
Status: completed
Agent: I've analyzed the document and here's a summary:

The document is a research paper on climate change impacts, discussing global temperature increases, sea level rise, and mitigation strategies. Key points include:
1. Global temperatures have risen by 1.1°C since pre-industrial times
2. Sea levels are projected to rise by 0.5-1.2m by 2100
3. Recommended mitigation strategies include renewable energy transition and carbon capture

Artifact: document-analysis
Data: {'title': 'Climate Change Impacts', 'page_count': 12, 'topics': ['global warming', 'sea level rise', 'mitigation'], 'key_findings': [...]}
```

## Example 4: Form-Based Interaction

This example demonstrates how to use structured data for form-based interactions.

### Client Code (Python)

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, DataPart, Message, TaskState

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID and session ID
    task_id = str(uuid.uuid4())
    session_id = str(uuid.uuid4())
    
    # Prepare the initial message
    payload = {
        "id": task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="I need to submit a reimbursement request for a business trip")]
        )
    }
    
    # Send the task and get the response
    response = await client.send_task(payload)
    
    # Print the response
    print(f"Task ID: {response.result.id}")
    print(f"Status: {response.result.status.state}")
    if response.result.status.message:
        print(f"Agent: {response.result.status.message.parts[0].text}")
    
    # If the agent needs more information and provides a form, fill it out
    if response.result.status.state == TaskState.INPUT_REQUIRED:
        # Check if there's a form in the message
        form_data = None
        for part in response.result.status.message.parts:
            if part.type == "data" and "form" in part.data:
                form_data = part.data
                break
        
        if form_data:
            print(f"Form received: {form_data}")
            
            # Fill out the form
            completed_form = {
                "amount": 1250.75,
                "currency": "USD",
                "purpose": "Conference attendance",
                "date": "2023-04-15",
                "category": "Travel"
            }
            
            # Send the form back
            form_response_payload = {
                "id": task_id,
                "sessionId": session_id,
                "message": Message(
                    role="user",
                    parts=[
                        TextPart(text="Here's my reimbursement information"),
                        DataPart(data=completed_form)
                    ]
                )
            }
            
            # Send the form response
            form_result = await client.send_task(form_response_payload)
            
            # Print the final response
            print(f"Final Status: {form_result.result.status.state}")
            if form_result.result.status.message:
                print(f"Agent: {form_result.result.status.message.parts[0].text}")
            
            # Print any artifacts
            if form_result.result.artifacts:
                for artifact in form_result.result.artifacts:
                    print(f"Artifact: {artifact.name}")
                    for part in artifact.parts:
                        if part.type == "data":
                            print(f"Data: {part.data}")

asyncio.run(main())
```

### Expected Output

```
Task ID: 550e8400-e29b-41d4-a716-446655440000
Status: input-required
Agent: I'll help you submit a reimbursement request. Please provide the following information:
Form received: {'form': {'fields': [{'name': 'amount', 'type': 'number', 'required': true}, {'name': 'currency', 'type': 'string', 'required': true}, {'name': 'purpose', 'type': 'string', 'required': true}, {'name': 'date', 'type': 'string', 'required': true}, {'name': 'category', 'type': 'string', 'required': true}]}}
Final Status: completed
Agent: Thank you for submitting your reimbursement request. I've processed it and created a request ID for you. Your reimbursement for $1,250.75 USD for Conference attendance on 2023-04-15 has been submitted successfully.
Artifact: reimbursement-receipt
Data: {'request_id': 'REQ-2023-04-15-001', 'status': 'approved', 'amount': 1250.75, 'currency': 'USD', 'purpose': 'Conference attendance', 'date': '2023-04-15', 'category': 'Travel', 'processing_time': '1 business day'}
```

## Example 5: Multi-Agent Orchestration

This example demonstrates how to orchestrate multiple agents using A2A.

### Client Code (Python)

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.client.card_resolver import A2ACardResolver
from common.types import TextPart, Message

async def main():
    # Initialize card resolver
    resolver = A2ACardResolver()
    
    # Discover available agents
    currency_agent_card = await resolver.resolve("http://localhost:10000")
    image_agent_card = await resolver.resolve("http://localhost:10001")
    
    print(f"Found Currency Agent: {currency_agent_card.name}")
    print(f"Found Image Agent: {image_agent_card.name}")
    
    # Initialize clients for each agent
    currency_client = A2AClient(agent_card=currency_agent_card)
    image_client = A2AClient(agent_card=image_agent_card)
    
    # Create a unique session ID
    session_id = str(uuid.uuid4())
    
    # Example 1: Currency conversion
    currency_task_id = str(uuid.uuid4())
    currency_payload = {
        "id": currency_task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Convert 500 USD to JPY")]
        )
    }
    
    print("\nSending currency conversion request...")
    currency_response = await currency_client.send_task(currency_payload)
    
    print(f"Currency Agent Response: {currency_response.result.status.message.parts[0].text}")
    
    # Example 2: Image generation based on currency
    if currency_response.result.artifacts:
        # Extract conversion data
        conversion_data = None
        for artifact in currency_response.result.artifacts:
            for part in artifact.parts:
                if part.type == "data":
                    conversion_data = part.data
                    break
        
        if conversion_data:
            # Generate an image based on the conversion
            image_task_id = str(uuid.uuid4())
            image_payload = {
                "id": image_task_id,
                "sessionId": session_id,
                "message": Message(
                    role="user",
                    parts=[TextPart(text=f"Generate an image representing the exchange of {conversion_data['from']} to {conversion_data['to']} currency")]
                )
            }
            
            print("\nSending image generation request...")
            async for update in image_client.send_task_streaming(image_payload):
                if hasattr(update.result, "status"):
                    if update.result.status.message:
                        print(f"Image Agent: {update.result.status.message.parts[0].text}")
                    if update.result.final:
                        print("Image generation complete")
                elif hasattr(update.result, "artifact"):
                    print(f"Received artifact: {update.result.artifact.name}")

asyncio.run(main())
```

### Expected Output

```
Found Currency Agent: Currency Agent
Found Image Agent: Image Generation Agent

Sending currency conversion request...
Currency Agent Response: Based on the current exchange rate, 500 USD is approximately 75,325 JPY.

Sending image generation request...
Image Agent: I'll generate an image representing the exchange of USD to JPY currency.
Image Agent: Creating your image now...
Received artifact: generated-image
Image Agent: Here's your image representing the exchange of USD to JPY currency. The image shows US dollar and Japanese yen symbols with arrows indicating conversion between them.
Image generation complete
```

## Example 6: Push Notifications

This example demonstrates how to set up push notifications for long-running tasks.

### Server Code (Python)

```python
from fastapi import FastAPI, Request
from common.types import TaskStatusUpdateEvent, TaskArtifactUpdateEvent
import uvicorn

app = FastAPI()

@app.post("/webhook")
async def webhook(request: Request):
    data = await request.json()
    
    # Check if it's a status update or artifact update
    if "status" in data:
        event = TaskStatusUpdateEvent(**data)
        print(f"Received status update for task {event.id}")
        print(f"Status: {event.status.state}")
        if event.status.message:
            print(f"Message: {event.status.message.parts[0].text}")
        if event.final:
            print("This is the final update")
    elif "artifact" in data:
        event = TaskArtifactUpdateEvent(**data)
        print(f"Received artifact update for task {event.id}")
        print(f"Artifact: {event.artifact.name}")
    
    return {"status": "received"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Client Code (Python)

```python
import asyncio
import uuid
from common.client.client import A2AClient
from common.types import TextPart, Message, PushNotificationConfig, AuthenticationInfo

async def main():
    # Initialize client with agent URL
    client = A2AClient(url="http://localhost:10000")
    
    # Create a unique task ID and session ID
    task_id = str(uuid.uuid4())
    session_id = str(uuid.uuid4())
    
    # Set up push notification configuration
    push_config = PushNotificationConfig(
        url="http://localhost:8000/webhook",
        authentication=AuthenticationInfo(schemes=["bearer"])
    )
    
    # Prepare the message with push notification config
    payload = {
        "id": task_id,
        "sessionId": session_id,
        "message": Message(
            role="user",
            parts=[TextPart(text="Generate a detailed report on climate change impacts")]
        ),
        "pushNotification": push_config
    }
    
    # Send the task
    response = await client.send_task(payload)
    
    print(f"Task submitted with ID: {response.result.id}")
    print("You will receive push notifications as the task progresses.")
    print("Press Ctrl+C to exit when done.")
    
    # Keep the program running to receive notifications
    while True:
        await asyncio.sleep(1)

asyncio.run(main())
```

### Expected Output (Webhook Server)

```
Received status update for task 550e8400-e29b-41d4-a716-446655440000
Status: working
Message: I'm starting to generate a detailed report on climate change impacts. This may take a few moments...

Received status update for task 550e8400-e29b-41d4-a716-446655440000
Status: working
Message: Researching current data on global temperature increases...

Received artifact update for task 550e8400-e29b-41d4-a716-446655440000
Artifact: temperature-data

Received status update for task 550e8400-e29b-41d4-a716-446655440000
Status: working
Message: Analyzing sea level rise projections...

Received artifact update for task 550e8400-e29b-41d4-a716-446655440000
Artifact: sea-level-data

Received status update for task 550e8400-e29b-41d4-a716-446655440000
Status: completed
Message: I've completed the detailed report on climate change impacts. The report covers global temperature increases, sea level rise, extreme weather events, biodiversity loss, and mitigation strategies.
This is the final update
```

These examples demonstrate the versatility of the A2A protocol in different scenarios. By following these patterns, you can build sophisticated agent-based applications that leverage the full capabilities of the protocol.
