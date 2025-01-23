# MCP Server Creation
- Import FastMCP class: `from mcp.server.fastmcp import FastMCP`
- Create server instance: `mcp = FastMCP("Server Name")`
- Optional dependencies: `mcp = FastMCP("Name", dependencies=["pandas", "numpy"])`

# Resource Definition
Resources provide data without computation or side effects.
```python
@mcp.resource("config://app")
def get_config() -> str:
    """Static configuration data"""
    return "Config data"

@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """Dynamic user data with parameters"""
    return f"Profile for {user_id}"
```

# Tool Definition

Tools perform computations and have side effects.

```python
@mcp.tool()
def calculate_bmi(weight_kg: float, height_m: float) -> float:
    """Calculate BMI given weight in kg and height in meters"""
    return weight_kg / (height_m ** 2)

@mcp.tool()
async def fetch_weather(city: str) -> str:
    """Async tool example"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.text
```

# Prompt Definition

Prompts are reusable LLM interaction templates.

```python
@mcp.prompt()
def review_code(code: str) -> str:
    return f"Please review this code:\n\n{code}"

@mcp.prompt()
def debug_error(error: str) -> list[Message]:
    return [
        UserMessage("I'm seeing this error:"),
        UserMessage(error),
        AssistantMessage("I'll help debug that. What have you tried so far?")
    ]
```

# Context Object Usage

```python
@mcp.tool()
async def long_task(files: list[str], ctx: Context) -> str:
    """Tool with progress tracking"""
    for i, file in enumerate(files):
        ctx.info(f"Processing {file}")
        await ctx.report_progress(i, len(files))
        data = await ctx.read_resource(f"file://{file}")
    return "Complete"
```

# MCP Client Usage Summary

## Basic Client Setup
```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="python",
    args=["server.py"],
    env=None
)
```
## Client Session Operations
```python
async with stdio_client(server_params) as (read, write):
    async with ClientSession(read, write) as session:
        # Initialize connection
        await session.initialize()
        
        # List available components
        prompts = await session.list_prompts()
        resources = await session.list_resources()
        tools = await session.list_tools()
        
        # Use components
        prompt = await session.get_prompt("example-prompt", {"arg1": "value"})
        resource = await session.read_resource("file://path")
        result = await session.call_tool("tool-name", {"arg1": "value"})
```
# MCP Core Primitives Summary

## Protocol Primitives
1. Prompts (User-controlled)
   - Interactive templates
   - Invoked by user choice
   - Example: Slash commands, menu options

2. Resources (Application-controlled)
   - Contextual data managed by client
   - Read-only data access
   - Example: File contents, API responses

3. Tools (Model-controlled)
   - Functions for LLM actions
   - Can perform computation
   - Example: API calls, data updates

## Server Capabilities
- prompts: Template management
- resources: Resource exposure
- tools: Tool discovery/execution
- logging: Server logging config
- completion: Argument suggestions

# Running MCP Server Summary
## Development Mode
```bash
mcp dev server.py
mcp dev server.py --with pandas --with numpy
mcp dev server.py --with-editable .
```

## Production Mode
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("App Name")

if __name__ == "__main__":
    mcp.run()
```

# Installation Options

```bash
# Basic install
mcp install server.py

# Named install
mcp install server.py --name "Analytics Server"

# With environment variables
mcp install server.py -e API_KEY=abc123
mcp install server.py -f .env
```

# Basic MCP Server Setup
```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

class CustomServer:
    def __init__(self):
        self.server = Server(
            name="custom-server",
            version="0.1.0" 
        )
        self.setup_handlers()
    
    def setup_handlers(self):
        self.server.setRequestHandler(ListToolsRequestSchema, self.list_tools)
        self.server.setRequestHandler(CallToolRequestSchema, self.call_tool)

    async def run(self):
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(
                read_stream,
                write_stream,
                self.server.create_initialization_options()
            )
```

# Tool Definition Example
```python
@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="example_tool",
            description="Tool description",
            inputSchema={
                "type": "object",
                "properties": {
                    "param1": {
                        "type": "string",
                        "description": "Parameter description"
                    }
                },
                "required": ["param1"]
            }
        )
    ]

@server.call_tool()

async def call_tool(name: str, arguments: dict[str, Any]) -> Sequence[TextContent]:
    if name == "example_tool":
        result = await process_tool(arguments["param1"])
        return [TextContent(type="text", text=json.dumps(result))]
    raise ValueError(f"Unknown tool: {name}")
```

# Error Handling Pattern
```python
from mcp.types import ErrorCode, McpError

async def call_tool(request: Dict[str, Any]) -> Dict[str, Any]:
    try:
        tool_name = request["params"]["name"]
        arguments = request["params"]["arguments"]
        
        if tool_name not in AVAILABLE_TOOLS:
            raise McpError(
                ErrorCode.MethodNotFound,
                f"Unknown tool: {tool_name}"
            )
            
        result = await process_tool(arguments)
        return {"success": True, "data": result}
        
    except Exception as err:
        raise McpError(
            ErrorCode.InternalError,
            str(err)
        ) from err
```
# Transport Implementation
```python
from abc import ABC, abstractmethod
from typing import Self
from mcp.server import Server

class TransportBase(ABC):
    def __init__(self, server: Server) -> None:
        self.server = server

    @abstractmethod 
    async def serve(self, *, raise_exceptions: bool = False) -> None:
        """Start serving the transport"""
        
    @abstractmethod
    async def shutdown(self) -> None:
        """Gracefully shutdown the transport"""

    async def __aenter__(self) -> Self:
        return self
        
    async def __aexit__(self, *exc: object) -> None:
        await self.shutdown()
```
# Initialization Pattern

```python
def main() -> None:
    server = CustomServer()
    logging.basicConfig(level=logging.INFO)
    asyncio.run(server.run())

if __name__ == "__main__":
    main()
```

# Cache Implementation Pattern
```python
from datetime import datetime, timedelta
from collections import OrderedDict
from typing import Optional, Any, Tuple

class Cache:
    def __init__(self, max_size: int = 100, ttl_hours: int = 24):
        self.cache: OrderedDict[str, Tuple[Any, datetime]] = OrderedDict()
        self.max_size = max_size
        self.ttl = timedelta(hours=ttl_hours)

    def get(self, key: str) -> Optional[Any]:
        if key not in self.cache:
            return None
        item, timestamp = self.cache[key]
        if datetime.now() - timestamp > self.ttl:
            del self.cache[key]
            return None
        return item

    def set(self, key: str, value: Any) -> None:
        if len(self.cache) >= self.max_size:
            self.cache.popitem(last=False)
        self.cache[key] = (value, datetime.now())
```

# Logging Pattern

```python
# Basic Setup
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("server-name")

# Operation Logging
class CustomServer:
    async def call_tool(self, name: str, arguments: dict):
        try:
            logger.info(f"Executing tool: {name}")
            logger.debug(f"Tool arguments: {arguments}")
            result = await self.process_tool(name, arguments)
            logger.info(f"Tool execution successful: {name}")
            return result
        except Exception as e:
            logger.error(f"Tool execution failed: {str(e)}", exc_info=True)
            raise

# Cache Logging
class Cache:
    def get(self, key: str):
        if cached := self.cache.get(key):
            logger.info(f"Cache hit for {key}")
            return cached
        logger.debug(f"Cache miss for {key}")
        return None

# Server Status Logging
async def run(self):
    logger.info("Starting MCP server...")
    await self.initialize()
    logger.info(f"Server initialized with capabilities: {self.capabilities}")
    try:
        async with stdio_server() as (read, write):
            logger.info("Server running on stdio transport")
            await self.server.run(read, write)
    except Exception as e:
        logger.critical(f"Server crashed: {str(e)}", exc_info=True)
        raise
    finally:
        logger.info("Server shutting down")
```

# Full server example

```python
from mcp.server.fastmcp import FastMCP
from mcp.types import Context, Message, UserMessage, AssistantMessage
import httpx
import logging
from typing import Any, Optional, Dict, List
from datetime import datetime

# Initialize logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("mcp-server")

# Initialize server
mcp = FastMCP("Example MCP Server", dependencies=["httpx", "pandas", "numpy"])

# Cache implementation
class Cache:
    def __init__(self, max_size: int = 100):
        self._cache: Dict[str, Any] = {}
        self.max_size = max_size
    
    def get(self, key: str) -> Optional[Any]:
        return self._cache.get(key)
    
    def set(self, key: str, value: Any) -> None:
        if len(self._cache) >= self.max_size:
            self._cache.pop(next(iter(self._cache)))
        self._cache[key] = value

cache = Cache()

# Resources
@mcp.resource("config://app")
def get_config() -> dict:
    """Application configuration"""
    return {
        "version": "1.0.0",
        "env": "development",
        "features": ["analytics", "auth"]
    }

@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> dict:
    """Get user profile data"""
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached
    profile = {
        "id": user_id,
        "created_at": datetime.now().isoformat()
    }
    cache.set(f"user:{user_id}", profile)
    return profile

# Tools
@mcp.tool()
async def fetch_data(url: str, ctx: Context) -> str:
    """Fetch data from external URL"""
    logger.info(f"Fetching data from {url}")
    ctx.info(f"Starting request to {url}")
    
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        ctx.info("Request complete")
        return response.text

@mcp.tool()
def process_data(data: List[float]) -> dict:
    """Process numerical data"""
    return {
        "sum": sum(data),
        "len": len(data),
        "mean": sum(data) / len(data) if data else 0
    }

# Prompts
@mcp.prompt()
def analyze_data(data: str) -> List[Message]:
    """Analyze provided data"""
    return [
        UserMessage("Please analyze this data:"),
        UserMessage(data),
        AssistantMessage("I'll help analyze that. What specific aspects interest you?")
    ]

if __name__ == "__main__":
    mcp.run()
```