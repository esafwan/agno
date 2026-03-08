# Tool Integrations Handbook

Agno ships with **130+ pre-built tool integrations** organized into toolkits. Any Python function can also be a tool via the `@tool` decorator. This handbook provides a deep dive into how tools work, how to create them, and how to handle complex integration scenarios like authentication, sessions, and background jobs.

**Core Directory:** `libs/agno/agno/tools/`

---

## 1. Architecture & Philosophy

Agno's tool system is built on two primary pillars: the `Function` class and the `Toolkit` class.

### The `Function` Class
The `Function` class (`libs/agno/agno/tools/function.py`) is the atomic unit of a tool. It wraps a Python callable and:
- **Auto-generates JSON Schema**: Uses Python type hints and docstrings to build the schema that LLMs use to understand the tool.
- **Handles Execution**: Manages sync/async execution, validation (via Pydantic), and error handling.
- **State Management**: Can inject `agent`, `team`, or `run_context` into the function arguments at runtime.

### The `Toolkit` Class
The `Toolkit` class (`libs/agno/agno/tools/toolkit.py`) groups related functions together. 
- **Connection Management**: Tools that require state (like DB connections) can override `connect()` and `close()`.
- **Instruction Injection**: Toolkits can add specific instructions to the Agent's system prompt.
- **Filtering**: Allows users to include or exclude specific methods from the toolkit.

---

## 2. Creating Custom Tools

### The `@tool` Decorator
The simplest way to create a tool is by decorating a function.

```python
from agno.tools import tool
from agno.agent import Agent

@tool
def get_stock_price(symbol: str) -> str:
    """Get the current stock price for a given symbol.

    Args:
        symbol (str): The stock symbol (e.g., AAPL).

    Returns:
        str: The current price as a string.
    """
    # Implementation logic (e.g., calling an API)
    return f"150.00 USD"

agent = Agent(tools=[get_stock_price])
```

**Key Parameters for `@tool`:**
- `name`: Override the function name.
- `description`: Override the docstring as the tool description.
- `show_result`: If `True`, prints the result in the console.
- `requires_confirmation`: Pauses execution for user approval.
- `cache_results`: Enables in-memory/disk caching.

### Subclassing `Toolkit`
For more complex integrations, subclass `Toolkit`. This is ideal for tools that share configuration (like API keys) or state.

```python
from agno.tools import Toolkit

class MyServiceTools(Toolkit):
    def __init__(self, api_key: str):
        super().__init__(name="my_service")
        self.api_key = api_key
        # Register methods as tools
        self.register(self.fetch_data)

    def fetch_data(self, query: str) -> str:
        """Fetch data from MyService."""
        # Use self.api_key here
        return f"Results for {query}"
```

---

## 3. Authentication & API Integration

Agno follows standardized patterns for handling authentication and API calls.

### API Call Patterns
Most integrations use the `requests` library. See `libs/agno/agno/tools/api.py` for a generic implementation.

**Common Auth Methods:**
1.  **API Keys**: Passed via constructor and added to headers (e.g., `Authorization: Bearer <key>`).
2.  **Basic Auth**: Using `requests.auth.HTTPBasicAuth`.
3.  **OAuth/Tokens**: Handled manually in the `Toolkit.__init__` or injected via environment variables.

### Example: Secure API Toolkit
```python
import requests
from agno.tools import Toolkit

class SecureApiToolkit(Toolkit):
    def __init__(self, api_key: str, base_url: str):
        super().__init__(name="secure_api")
        self.api_key = api_key
        self.base_url = base_url
        self.register(self.call_endpoint)

    def call_endpoint(self, path: str) -> str:
        """Call a secure endpoint."""
        headers = {"X-API-KEY": self.api_key}
        response = requests.get(f"{self.base_url}/{path}", headers=headers)
        return response.text
```

---

## 4. Advanced Features

### Human-in-the-Loop (HITL)
Agno supports blocking tool execution until a human provides input or confirmation.
- **`requires_confirmation=True`**: The agent will wait for a "yes/no" before proceeding.
- **`requires_user_input=True`**: The agent will pause and wait for specific data from the user.

### External Execution
If a tool involves UI interactions or long-running tasks that shouldn't block the backend, use `external_execution=True`. The agent will produce a "paused" state, allowing the frontend to take over.

### Caching
Reduce API costs and latency by caching tool outputs.
```python
@tool(cache_results=True, cache_ttl=3600)  # Cache for 1 hour
def heavy_computation(data: str) -> str:
    ...
```

### Background Jobs & Webhooks
While Agno doesn't have a built-in "scheduler," it integrates with systems like **E2B** or **Airflow**.
- **E2B Tools**: Support `run_background_command` for running code in isolated sandboxes.
- **Webhooks**: Handled at the application level; tools can be used to *trigger* webhooks in external systems.

---

## 5. Best Practices & Nuances

-   **Docstrings are Code**: The LLM *only* knows what your tool does through the docstring. Be explicit about parameters and return values.
-   **Type Hints**: Always use Python type hints. Agno uses these to build the JSON schema (`str`, `int`, `List[str]`, etc.).
-   **Return Strings**: Tools should ideally return strings or JSON-serializable objects.
-   **Error Handling**: Wrap tool logic in `try-except`. Return meaningful error messages to the LLM so it can attempt a "fix" or report the issue.
-   **Security**: Validate paths (use `Toolkit._check_path`) to prevent directory traversal in file-based tools.

---

## 6. Popular Toolkit Quick Reference

| Toolkit | Category | Auth Req |
|---------|----------|----------|
| `DuckDuckGoTools` | Search | No |
| `YFinanceTools` | Finance | No |
| `GithubTools` | DevOps | Personal Access Token |
| `PostgresTools` | Database | Connection String |
| `GmailTools` | Google | OAuth2 / Credentials |
| `SlackTools` | Comm | Bot Token |

For the full list of 130+ tools, see the `agno/tools/` directory.
