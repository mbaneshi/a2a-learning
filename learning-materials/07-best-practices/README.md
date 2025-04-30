# A2A Protocol Best Practices

This section outlines best practices for working with the A2A protocol. Following these guidelines will help you build robust, secure, and efficient agent-based applications.

## General Best Practices

### 1. Use Unique Identifiers

Always use globally unique identifiers for tasks and sessions:

```python
import uuid

task_id = str(uuid.uuid4())
session_id = str(uuid.uuid4())
```

This prevents collisions and ensures that each task and session can be uniquely identified.

### 2. Handle Errors Gracefully

Implement comprehensive error handling in both clients and servers:

```python
try:
    response = await client.send_task(payload)
except A2AClientHTTPError as e:
    print(f"HTTP Error: {e.status_code} - {e}")
except A2AClientJSONError as e:
    print(f"JSON Error: {e}")
except Exception as e:
    print(f"Unexpected Error: {e}")
```

For servers, return appropriate error responses:

```python
try:
    # Process request
    return response
except ValidationError as e:
    return JSONRPCResponse(
        id=request.id,
        error=InvalidParamsError(message=str(e))
    )
except Exception as e:
    return JSONRPCResponse(
        id=request.id,
        error=InternalError(message=str(e))
    )
```

### 3. Implement Timeouts

Set appropriate timeouts for network requests:

```python
async with httpx.AsyncClient() as client:
    try:
        response = await client.post(
            self.url, json=request.model_dump(), timeout=30
        )
        response.raise_for_status()
        return response.json()
    except httpx.TimeoutException:
        raise A2AClientHTTPError(408, "Request timed out")
```

For long-running tasks, consider implementing a timeout mechanism in your task manager:

```python
async def process_task_with_timeout(self, task_id, timeout=300):
    try:
        return await asyncio.wait_for(self._process_task(task_id), timeout)
    except asyncio.TimeoutError:
        # Update task status to failed
        await self.update_store(
            task_id,
            TaskStatus(
                state=TaskState.FAILED,
                message=Message(
                    role="agent",
                    parts=[TextPart(text="Task timed out")]
                )
            ),
            None
        )
        raise
```

### 4. Use Proper Logging

Implement comprehensive logging in both clients and servers:

```python
import logging

logger = logging.getLogger(__name__)

class A2AClient:
    def __init__(self, agent_card: AgentCard = None, url: str = None):
        if agent_card:
            self.url = agent_card.url
            logger.info(f"Initialized client with agent card: {agent_card.name}")
        elif url:
            self.url = url
            logger.info(f"Initialized client with URL: {url}")
        else:
            logger.error("Missing required parameters: agent_card or url")
            raise ValueError("Must provide either agent_card or url")

    async def send_task(self, payload: dict[str, Any]) -> SendTaskResponse:
        logger.debug(f"Sending task: {payload['id']}")
        request = SendTaskRequest(params=payload)
        try:
            response = await self._send_request(request)
            logger.debug(f"Received response for task: {payload['id']}")
            return SendTaskResponse(**response)
        except Exception as e:
            logger.error(f"Error sending task {payload['id']}: {e}")
            raise
```

### 5. Validate Input and Output

Always validate input and output data:

```python
from pydantic import ValidationError

def _validate_request(self, request: SendTaskRequest) -> Optional[SendTaskResponse]:
    """Validate the request parameters."""
    try:
        # Check if message has parts
        if not request.params.message.parts:
            return SendTaskResponse(
                id=request.id,
                error=InvalidParamsError(message="Message must have at least one part")
            )
        
        # Check if the first part is text
        if request.params.message.parts[0].type != "text":
            return SendTaskResponse(
                id=request.id,
                error=InvalidParamsError(message="First message part must be text")
            )
        
        return None
    except ValidationError as e:
        return SendTaskResponse(
            id=request.id,
            error=InvalidParamsError(message=str(e))
        )
```

## Client Best Practices

### 1. Implement Retry Logic

For important operations, implement retry logic with exponential backoff:

```python
import random
import time

async def send_task_with_retry(self, payload, max_retries=3, base_delay=1):
    retries = 0
    while retries < max_retries:
        try:
            return await self.send_task(payload)
        except A2AClientHTTPError as e:
            # Only retry on certain status codes
            if e.status_code not in [408, 429, 500, 502, 503, 504]:
                raise
            
            retries += 1
            if retries >= max_retries:
                raise
            
            # Exponential backoff with jitter
            delay = base_delay * (2 ** (retries - 1)) * (0.5 + random.random())
            logger.warning(f"Request failed with {e.status_code}, retrying in {delay:.2f} seconds...")
            await asyncio.sleep(delay)
        except A2AClientJSONError:
            # Don't retry on JSON errors
            raise
```

### 2. Cache Agent Cards

Cache agent cards to avoid unnecessary requests:

```python
class CachingCardResolver:
    def __init__(self, ttl=3600):  # TTL in seconds
        self.cache = {}
        self.ttl = ttl
    
    async def resolve(self, url: str) -> AgentCard:
        now = time.time()
        
        # Check if we have a cached card that's still valid
        if url in self.cache:
            card, timestamp = self.cache[url]
            if now - timestamp < self.ttl:
                return card
        
        # Fetch a new card
        resolver = A2ACardResolver()
        card = await resolver.resolve(url)
        
        # Cache the card
        self.cache[url] = (card, now)
        
        return card
```

### 3. Handle Streaming Properly

For streaming responses, ensure your code can handle disconnections and reconnections:

```python
async def stream_with_reconnection(self, payload, max_retries=3):
    retries = 0
    while retries < max_retries:
        try:
            async for update in self.send_task_streaming(payload):
                yield update
            break  # Success, exit the loop
        except httpx.ReadError:
            retries += 1
            if retries >= max_retries:
                raise
            
            logger.warning(f"Stream disconnected, retrying ({retries}/{max_retries})...")
            
            # Check the current state before reconnecting
            try:
                response = await self.get_task({"id": payload["id"]})
                if response.result.status.state in [TaskState.COMPLETED, TaskState.FAILED, TaskState.CANCELED]:
                    # Task is already in a terminal state, no need to reconnect
                    yield SendTaskStreamingResponse(
                        id=payload["id"],
                        result=TaskStatusUpdateEvent(
                            id=payload["id"],
                            status=response.result.status,
                            final=True
                        )
                    )
                    break
                
                # Reconnect using resubscribe
                async for update in self.resubscribe_task({"id": payload["id"]}):
                    yield update
                break  # Success, exit the loop
            except Exception as e:
                logger.error(f"Error checking task state: {e}")
                # Continue to retry
```

### 4. Implement Cancellation

Allow users to cancel long-running tasks:

```python
async def cancel_task_if_needed(self, task_id, timeout=30):
    """Cancel a task if it takes too long."""
    try:
        # Start a timer
        start_time = time.time()
        
        # Check the task status periodically
        while time.time() - start_time < timeout:
            response = await self.get_task({"id": task_id})
            if response.result.status.state in [TaskState.COMPLETED, TaskState.FAILED, TaskState.CANCELED]:
                return response
            
            await asyncio.sleep(1)
        
        # If we reach here, the task has timed out
        logger.warning(f"Task {task_id} timed out, attempting to cancel")
        return await self.cancel_task({"id": task_id})
    except Exception as e:
        logger.error(f"Error in cancel_task_if_needed: {e}")
        raise
```

## Server Best Practices

### 1. Use Asynchronous Code

Use async/await for all I/O operations to ensure good performance:

```python
async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
    # Validate the request
    validation_error = self._validate_request(request)
    if validation_error:
        return validation_error
    
    # Store the task
    await self.upsert_task(request.params)
    
    # Update task status to WORKING
    task = await self.update_store(
        request.params.id, TaskStatus(state=TaskState.WORKING), None
    )
    
    # Process the task asynchronously
    try:
        # Extract the query
        query = request.params.message.parts[0].text
        
        # Invoke the agent (make sure this is async)
        agent_response = await self.agent.invoke_async(query, request.params.sessionId)
        
        # Process the response
        # ...
        
        # Update the task with the new status
        task = await self.update_store(request.params.id, status, artifacts)
        
        # Return the response
        return SendTaskResponse(id=request.id, result=task)
    except Exception as e:
        # Handle errors
        # ...
```

### 2. Implement Proper Locking

Use locks to protect shared resources:

```python
async def update_store(self, task_id: str, status: TaskStatus, artifacts: list[Artifact]) -> Task:
    async with self.lock:
        try:
            task = self.tasks[task_id]
        except KeyError:
            logger.error(f"Task {task_id} not found for updating the task")
            raise ValueError(f"Task {task_id} not found")
        
        task.status = status
        
        if status.message is not None:
            task.history.append(status.message)
        
        if artifacts is not None:
            if task.artifacts is None:
                task.artifacts = []
            task.artifacts.extend(artifacts)
        
        return task
```

### 3. Implement Graceful Shutdown

Ensure your server can shut down gracefully:

```python
import signal

class A2AServer:
    def __init__(self, host="0.0.0.0", port=5000, ...):
        # ...
        self.should_exit = False
        self.active_tasks = set()
    
    def start(self):
        # Set up signal handlers
        signal.signal(signal.SIGINT, self._handle_sigint)
        signal.signal(signal.SIGTERM, self._handle_sigterm)
        
        # ...
        
        import uvicorn
        uvicorn.run(self.app, host=self.host, port=self.port)
    
    def _handle_sigint(self, sig, frame):
        logger.info("Received SIGINT, shutting down gracefully...")
        self._shutdown()
    
    def _handle_sigterm(self, sig, frame):
        logger.info("Received SIGTERM, shutting down gracefully...")
        self._shutdown()
    
    def _shutdown(self):
        self.should_exit = True
        
        # Wait for active tasks to complete
        if self.active_tasks:
            logger.info(f"Waiting for {len(self.active_tasks)} active tasks to complete...")
            # Implement waiting logic
```

### 4. Implement Rate Limiting

Protect your server from abuse with rate limiting:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse
import time

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, requests_per_minute=60):
        super().__init__(app)
        self.requests_per_minute = requests_per_minute
        self.window_size = 60  # seconds
        self.requests = {}
    
    async def dispatch(self, request, call_next):
        # Get client IP
        client_ip = request.client.host
        
        # Get current time
        now = time.time()
        
        # Clean up old requests
        self.requests = {ip: [t for t in times if now - t < self.window_size] 
                         for ip, times in self.requests.items()}
        
        # Check if client has exceeded rate limit
        if client_ip in self.requests and len(self.requests[client_ip]) >= self.requests_per_minute:
            return JSONResponse(
                status_code=429,
                content={
                    "jsonrpc": "2.0",
                    "id": None,
                    "error": {
                        "code": 429,
                        "message": "Too many requests"
                    }
                }
            )
        
        # Add request to tracking
        if client_ip not in self.requests:
            self.requests[client_ip] = []
        self.requests[client_ip].append(now)
        
        # Process the request
        return await call_next(request)
```

### 5. Implement Health Checks

Add health check endpoints to your server:

```python
@app.route("/health", methods=["GET"])
async def health_check(request):
    # Check if the server is healthy
    return JSONResponse({"status": "ok"})

@app.route("/readiness", methods=["GET"])
async def readiness_check(request):
    # Check if the server is ready to accept requests
    # This might include checking database connections, etc.
    return JSONResponse({"status": "ready"})
```

## Security Best Practices

### 1. Implement Authentication

Protect your server with authentication:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class AuthenticationMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, api_key):
        super().__init__(app)
        self.api_key = api_key
    
    async def dispatch(self, request, call_next):
        # Skip authentication for agent card and health check endpoints
        if request.url.path in ["/.well-known/agent.json", "/health", "/readiness"]:
            return await call_next(request)
        
        # Check for API key in headers
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            return JSONResponse(
                status_code=401,
                content={
                    "jsonrpc": "2.0",
                    "id": None,
                    "error": {
                        "code": 401,
                        "message": "Unauthorized"
                    }
                }
            )
        
        # Extract and validate API key
        api_key = auth_header.replace("Bearer ", "")
        if api_key != self.api_key:
            return JSONResponse(
                status_code=401,
                content={
                    "jsonrpc": "2.0",
                    "id": None,
                    "error": {
                        "code": 401,
                        "message": "Invalid API key"
                    }
                }
            )
        
        # Process the request
        return await call_next(request)
```

### 2. Secure Push Notifications

Use JWT for securing push notifications:

```python
from jwcrypto import jwk, jwt
import json
import time

class PushNotificationSenderAuth:
    def __init__(self):
        self.jwk = None
    
    def generate_jwk(self):
        """Generate a new JWK key pair."""
        self.jwk = jwk.JWK.generate(kty="RSA", size=2048)
    
    async def generate_jwt(self) -> str:
        """Generate a JWT token for push notifications."""
        if not self.jwk:
            raise ValueError("JWK not initialized")
        
        # Create the JWT
        token = jwt.JWT(
            header={"alg": "RS256", "typ": "JWT"},
            claims={
                "iss": "a2a-server",
                "iat": int(time.time()),
                "exp": int(time.time()) + 3600,  # 1 hour expiration
            }
        )
        
        # Sign the JWT
        token.make_signed_token(self.jwk)
        
        return token.serialize()
    
    async def handle_jwks_endpoint(self, request):
        """Handle the JWKS endpoint for push notification authentication."""
        if not self.jwk:
            raise ValueError("JWK not initialized")
        
        # Export the public key
        public_jwk = json.loads(self.jwk.export_public())
        
        # Create the JWKS
        jwks = {
            "keys": [public_jwk]
        }
        
        return JSONResponse(jwks)
```

### 3. Validate and Sanitize Input

Always validate and sanitize input data:

```python
def _sanitize_text(self, text: str) -> str:
    """Sanitize text input to prevent injection attacks."""
    # Remove control characters
    text = "".join(c for c in text if c.isprintable())
    
    # Limit length
    max_length = 10000
    if len(text) > max_length:
        text = text[:max_length]
    
    return text

def _validate_file(self, file: FileContent) -> bool:
    """Validate file content."""
    # Check file size
    if file.bytes:
        decoded = base64.b64decode(file.bytes)
        if len(decoded) > 10 * 1024 * 1024:  # 10 MB limit
            return False
    
    # Check file type
    allowed_mime_types = [
        "image/jpeg", "image/png", "image/gif",
        "application/pdf", "text/plain",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    ]
    if file.mimeType not in allowed_mime_types:
        return False
    
    return True
```

### 4. Implement Content Security Policy

Add a Content Security Policy to your server:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import Response

class CSPMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)
        
        # Add CSP header
        response.headers["Content-Security-Policy"] = (
            "default-src 'self'; "
            "script-src 'self'; "
            "style-src 'self'; "
            "img-src 'self' data:; "
            "connect-src 'self'; "
            "font-src 'self'; "
            "object-src 'none'; "
            "media-src 'self'; "
            "frame-src 'none'; "
            "form-action 'self'; "
            "base-uri 'self'; "
            "frame-ancestors 'none'"
        )
        
        return response
```

## Performance Best Practices

### 1. Use Connection Pooling

Use connection pooling for HTTP requests:

```python
class A2AClient:
    def __init__(self, agent_card: AgentCard = None, url: str = None):
        # ...
        self.http_client = httpx.AsyncClient(
            timeout=30,
            limits=httpx.Limits(max_keepalive_connections=5, max_connections=10)
        )
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.http_client.aclose()
    
    async def send_task(self, payload: dict[str, Any]) -> SendTaskResponse:
        # ...
        response = await self.http_client.post(self.url, json=request.model_dump())
        # ...
```

### 2. Implement Caching

Cache responses where appropriate:

```python
import functools
import asyncio

def async_lru_cache(maxsize=128, ttl=3600):
    """LRU cache decorator for async functions with TTL."""
    cache = {}
    
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            key = str(args) + str(kwargs)
            now = time.time()
            
            # Clean expired entries
            expired_keys = [k for k, (_, t) in cache.items() if now - t > ttl]
            for k in expired_keys:
                del cache[k]
            
            # Check cache
            if key in cache:
                value, _ = cache[key]
                return value
            
            # Call function
            value = await func(*args, **kwargs)
            
            # Update cache
            if len(cache) >= maxsize:
                # Remove oldest entry
                oldest_key = min(cache.keys(), key=lambda k: cache[k][1])
                del cache[oldest_key]
            
            cache[key] = (value, now)
            
            return value
        
        return wrapper
    
    return decorator

class A2ACardResolver:
    @async_lru_cache(maxsize=100, ttl=3600)
    async def resolve(self, url: str) -> AgentCard:
        # ...
```

### 3. Use Efficient Data Structures

Choose efficient data structures for your use case:

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cache = OrderedDict()
        self.capacity = capacity
    
    def get(self, key):
        if key not in self.cache:
            return None
        
        # Move to end (most recently used)
        self.cache.move_to_end(key)
        
        return self.cache[key]
    
    def put(self, key, value):
        # If key exists, update and move to end
        if key in self.cache:
            self.cache[key] = value
            self.cache.move_to_end(key)
            return
        
        # If at capacity, remove least recently used
        if len(self.cache) >= self.capacity:
            self.cache.popitem(last=False)
        
        # Add new item
        self.cache[key] = value
```

### 4. Implement Pagination

Implement pagination for large responses:

```python
async def get_task_history(self, task_id: str, page: int = 1, page_size: int = 20) -> list[Message]:
    """Get paginated task history."""
    async with self.lock:
        task = self.tasks.get(task_id)
        if task is None:
            return []
        
        # Calculate start and end indices
        start = (page - 1) * page_size
        end = start + page_size
        
        # Return the requested page
        return task.history[start:end]
```

### 5. Use Efficient Serialization

Use efficient serialization for large objects:

```python
def _serialize_file_content(self, file: FileContent) -> dict:
    """Efficiently serialize file content."""
    result = {
        "name": file.name,
        "mimeType": file.mimeType
    }
    
    if file.bytes:
        # For large files, consider storing the bytes separately
        # and including a reference here
        if len(file.bytes) > 1024 * 1024:  # 1 MB
            file_id = str(uuid.uuid4())
            self._store_file_bytes(file_id, file.bytes)
            result["fileId"] = file_id
        else:
            result["bytes"] = file.bytes
    
    if file.uri:
        result["uri"] = file.uri
    
    return result
```

By following these best practices, you can build robust, secure, and efficient agent-based applications using the A2A protocol.
