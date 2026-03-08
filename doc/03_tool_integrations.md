# Tool Integrations Handbook

Agno ships with **130+ pre-built tool integrations** organized into toolkits. Any Python function can also be a tool via the `@tool` decorator. This handbook provides a deep dive into how tools work, how to create them, and how to handle complex integration scenarios like authentication, sessions, and background jobs.

**Core Directory:** `libs/agno/agno/tools/`

---

## Phase 1 Tool Status

| Toolkit | Category | Status | Import Path |
|---------|----------|--------|-------------|
| `SlackTools` | Communication | ✅ Documented | `agno.tools.slack` |
| `DiscordTools` | Communication | ✅ Documented | `agno.tools.discord` |
| `TelegramTools` | Communication | ✅ Documented | `agno.tools.telegram` |
| `JiraTools` | Project Management | ✅ Documented | `agno.tools.jira` |
| `LinearTools` | Project Management | ✅ Documented | `agno.tools.linear` |
| `ClickUpTools` | Project Management | ✅ Documented | `agno.tools.clickup` |
| `TrelloTools` | Project Management | ✅ Documented | `agno.tools.trello` |
| `NotionTools` | Project Management | ✅ Documented | `agno.tools.notion` |
| `ZendeskTools` | Project Management | ✅ Documented | `agno.tools.zendesk` |
| `CalComTools` | Project Management | ✅ Documented | `agno.tools.calcom` |
| `ZoomTools` | Project Management | ✅ Documented | `agno.tools.zoom` |
| `DuckDuckGoTools` | Web & Search | ✅ Documented | `agno.tools.duckduckgo` |
| `TavilyTools` | Web & Search | ✅ Documented | `agno.tools.tavily` |
| `SerperApiTools` | Web & Search | ✅ Documented | `agno.tools.serper` |
| `SerpApiTools` | Web & Search | ✅ Documented | `agno.tools.serpapi` |
| `BraveSearchTools` | Web & Search | ✅ Documented | `agno.tools.bravesearch` |
| `BaiduSearchTools` | Web & Search | ✅ Documented | `agno.tools.baidusearch` |
| `WebSearchTools` | Web & Search | ✅ Documented | `agno.tools.websearch` |
| `HackerNewsTools` | Data Sources | ✅ Documented | `agno.tools.hackernews` |
| `RedditTools` | Data Sources | ✅ Documented | `agno.tools.reddit` |
| `WikipediaTools` | Data Sources | ✅ Documented | `agno.tools.wikipedia` |
| `YoutubeTools` | Data Sources | ✅ Documented | `agno.tools.youtube` |
| `OpenWeatherTools` | Data Sources | ✅ Documented | `agno.tools.openweather` |
| `YFinanceTools` | Financial | ✅ Documented | `agno.tools.yfinance` |
| `ShopifyTools` | Financial | ✅ Documented | `agno.tools.shopify` |
| `SqlTools` | Database | ✅ Documented | `agno.tools.sql` |
| `PostgresTools` | Database | ✅ Documented | `agno.tools.postgres` |
| `PandasTools` | Database | ✅ Documented | `agno.tools.pandas` |
| `CsvTools` | Database | ✅ Documented | `agno.tools.csv_toolkit` |
| `GmailTools` | Google | ✅ Documented | `agno.tools.google.gmail` |
| `GoogleDriveTools` | Google | ✅ Documented | `agno.tools.google.drive` |
| `GoogleSheetsTools` | Google | ✅ Documented | `agno.tools.google.sheets` |
| `GoogleCalendarTools` | Google | ✅ Documented | `agno.tools.google.calendar` |
| `GoogleMapsTools` | Google | ✅ Documented | `agno.tools.google.maps` |
| `AwsSesTools` | AWS | ✅ Documented | `agno.tools.aws_ses` |
| `GithubTools` | GitHub | ✅ Documented | `agno.tools.github` |
| `OpenCVTools` | AI/Media | ✅ Documented | `agno.tools.opencv` |
| `UnsplashTools` | AI/Media | ✅ Documented | `agno.tools.unsplash` |
| `GiphyTools` | AI/Media | ✅ Documented | `agno.tools.giphy` |
| `CodingTools` | Dev/Execution | ✅ Documented | `agno.tools.coding` |
| `DockerTools` | Dev/Execution | ✅ Documented | `agno.tools.docker` |
| `ShellTools` | Dev/Execution | ✅ Documented | `agno.tools.shell` |
| `ParallelTools` | Special | ✅ Documented | `agno.tools.parallel` |
| `UserControlFlowTools` | Special | ✅ Documented | `agno.tools.user_control_flow` |
| `ReasoningTools` | Special | ✅ Documented | `agno.tools.reasoning` |
| `StreamlitComponents` | Special | ✅ Documented | `agno.tools.streamlit` |

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

## 9. Database & SQL

### `SQLTools`
Generic SQL interaction using SQLAlchemy. Supports any database with a valid connection URL.

*   **Import**: `from agno.tools.sql import SQLTools`
*   **Parameters**: `db_url` (e.g., `sqlite:///data.db`, `postgresql://user:pass@host/db`).
*   **Tools**:
    *   `list_tables()`
    *   `describe_table(table_name: str)`
    *   `run_sql_query(query: str, limit: int = 10)`

---

### `PostgresTools`
Specialized toolkit for PostgreSQL using `psycopg`.

*   **Import**: `from agno.tools.postgres import PostgresTools`
*   **Parameters**: `db_name`, `user`, `password`, `host`, `port`.
*   **Tools**:
    *   `show_tables()`
    *   `describe_table(table: str)`
    *   `summarize_table(table: str)`: Provides stats like min/max/avg for numeric columns.
    *   `run_query(query: str)`
    *   `export_table_to_path(table: str, path: str)`

---

### `DuckDbTools`
In-process analytical database. Great for local file analysis (CSV, Parquet, etc.).

*   **Import**: `from agno.tools.duckdb import DuckDbTools`
*   **Tools**:
    *   `run_query(query: str)`
    *   `create_table_from_path(path: str, table: str = None)`
    *   `summarize_table(table: str)`
    *   `load_s3_path_to_table(path: str)`
    *   `create_fts_index(table: str, unique_key: str, input_values: list[str])`

---

### `Neo4jTools`
Graph database interaction using Cypher.

*   **Import**: `from agno.tools.neo4j import Neo4jTools`
*   **Authentication**: `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`.
*   **Tools**:
    *   `list_labels()`: Returns all node labels.
    *   `list_relationship_types()`
    *   `get_schema()`: Returns nodes and relationships visualization.
    *   `run_cypher_query(query: str)`

---

### `GoogleBigQueryTools`
Serverless data warehouse. Requires `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION`.

*   **Import**: `from agno.tools.google.bigquery import GoogleBigQueryTools`
*   **Tools**:
    *   `list_tables()`
    *   `describe_table(table_id: str)`
    *   `run_sql_query(query: str)`

---

### `RedshiftTools`
AWS Data Warehouse. Supports IAM and standard auth.

*   **Import**: `from agno.tools.redshift import RedshiftTools`
*   **Tools**:
    *   `show_tables()`
    *   `describe_table(table: str)`
    *   `summarize_table(table: str)`
    *   `run_query(query: str)`

---

### `PandasTools`
In-memory data manipulation.

*   **Import**: `from agno.tools.pandas import PandasTools`
*   **Tools**:
    *   `create_pandas_dataframe(dataframe_name: str, create_using_function: str, function_parameters: dict)`
    *   `run_dataframe_operation(dataframe_name: str, operation: str, operation_parameters: dict)`: Runs methods like `head()`, `tail()`, `describe()`.

---

### `YFinanceTools`
Real-time and historical stock market data from Yahoo Finance.

*   **Import**: `from agno.tools.yfinance import YFinanceTools`
*   **Tools**:
    *   `get_current_stock_price(symbol: str)`
    *   `get_company_info(symbol: str)`: Returns sector, industry, market cap, etc.
    *   `get_historical_stock_prices(symbol: str, period: str = "1mo", interval: str = "1d")`
    *   `get_stock_fundamentals(symbol: str)`
    *   `get_income_statements(symbol: str)`
    *   `get_key_financial_ratios(symbol: str)`
    *   `get_analyst_recommendations(symbol: str)`
    *   `get_company_news(symbol: str, num_stories: int = 3)`
    *   `get_technical_indicators(symbol: str, period: str = "3mo")`

---

## 10. Developer & Cloud

### `GithubTools`
Comprehensive GitHub interaction. Requires `GITHUB_ACCESS_TOKEN`.

*   **Import**: `from agno.tools.github import GithubTools`
*   **Key Tools**:
    *   `list_repositories()`
    *   `get_repository(repo_name: str)`
    *   `create_issue(repo_name: str, title: str, body: str = None)`
    *   `create_pull_request(repo_name: str, title: str, body: str, head: str, base: str)`
    *   `get_file_content(repo_name: str, path: str)`
    *   `update_file(repo_name: str, path: str, message: str, content: str, sha: str)`
    *   `search_code(query: str)`

---

### `GitlabTools`
Manage Projects, Merge Requests, and Issues on GitLab. Requires `GITLAB_ACCESS_TOKEN`.

*   **Import**: `from agno.tools.gitlab import GitlabTools`
*   **Tools**:
    *   `list_projects(search: str = None)`
    *   `get_project(project_id_or_path: str)`
    *   `list_merge_requests(project_id_or_path: str, state: str = "opened")`
    *   `list_issues(project_id_or_path: str, state: str = "opened")`

---

### `BitbucketTools`
Atlassian Bitbucket integration. Requires `BITBUCKET_USERNAME` and `BITBUCKET_TOKEN`.

*   **Import**: `from agno.tools.bitbucket import BitbucketTools`
*   **Tools**:
    *   `list_repositories(workspace: str)`
    *   `list_all_pull_requests(workspace: str, repo_slug: str)`
    *   `get_pull_request_changes(workspace: str, repo_slug: str, pull_request_id: int)`
    *   `list_repository_commits(workspace: str, repo_slug: str)`

---

### `DockerTools`
Full Docker lifecycle management. Requires access to local Docker socket.

*   **Import**: `from agno.tools.docker import DockerTools`
*   **Tools**:
    *   `run_container(image: str, command: str = None, name: str = None, ports: dict = None)`
    *   `list_containers(all: bool = False)`
    *   `get_container_logs(container_id: str)`
    *   `list_images()`
    *   `pull_image(image_name: str)`
    *   `create_volume(volume_name: str)`

---

### `AWSLambdaTools`
Invoke serverless functions.

*   **Import**: `from agno.tools.aws_lambda import AWSLambdaTools`
*   **Tools**:
    *   `list_functions()`
    *   `invoke_function(function_name: str, payload: str = "{}")`

---

### `AWSSESTool`
Send professional emails using Amazon SES.

*   **Import**: `from agno.tools.aws_ses import AWSSESTool`
*   **Tools**:
    *   `send_email(subject: str, body: str, receiver_email: str)`

---

### `AirflowTools`
Generate and manage Airflow DAGs.

*   **Import**: `from agno.tools.airflow import AirflowTools`
*   **Tools**:
    *   `save_dag_file(contents: str, dag_file: str)`
    *   `read_dag_file(dag_file: str)`

---

## 11. Communication & Productivity

### `GmailTools`
Full Gmail integration for automation. Requires Google Cloud credentials.

*   **Import**: `from agno.tools.google.gmail import GmailTools`
*   **Authentication**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_PROJECT_ID`.
*   **Key Tools**:
    *   `get_latest_emails(count: int)`
    *   `get_unread_emails(count: int)`
    *   `send_email(to: str, subject: str, body: str, attachments: list = None)`
    *   `create_draft_email(to: str, subject: str, body: str)`
    *   `mark_email_as_read(message_id: str)`

---

### `GoogleSheetsTools`
Read and write spreadsheet data. Requires Google Cloud credentials.

*   **Import**: `from agno.tools.google.sheets import GoogleSheetsTools`
*   **Tools**:
    *   `read_sheet(spreadsheet_id: str, spreadsheet_range: str)`
    *   `update_sheet(spreadsheet_id: str, range_name: str, data: list)`
    *   `create_sheet(title: str)`
    *   `create_duplicate_sheet(source_id: str, new_title: str = None)`

---

### `GoogleMapTools`
Location services and place searching. Requires `GOOGLE_MAPS_API_KEY`.

*   **Import**: `from agno.tools.google.maps import GoogleMapTools`
*   **Tools**:
    *   `search_places(query: str)`: Business search.
    *   `get_directions(origin: str, destination: str, mode: str = "driving")`
    *   `geocode_address(address: str)`: Address to Lat/Lng.
    *   `reverse_geocode(lat: float, lng: float)`: Lat/Lng to Address.
    *   `validate_address(address: str)`

---

### `WhatsAppTools`
Send messages via WhatsApp Business API. Requires `WHATSAPP_ACCESS_TOKEN` and `WHATSAPP_PHONE_NUMBER_ID`.

*   **Import**: `from agno.tools.whatsapp import WhatsAppTools`
*   **Tools**:
    *   `send_text_message(text: str, recipient: str)`
    *   `send_template_message(recipient: str, template_name: str, language_code: str = "en_US")`

---

### `DiscordTools`
Interact with Discord channels. Requires `DISCORD_BOT_TOKEN`.

*   **Import**: `from agno.tools.discord import DiscordTools`
*   **Tools**:
    *   `send_message(channel_id: str, message: str)`
    *   `get_channel_messages(channel_id: str, limit: int = 100)`
    *   `list_channels(guild_id: str)`
    *   `delete_message(channel_id: str, message_id: str)`

---

### `FinancialDatasetsTools`
Professional-grade financial data. Requires `FINANCIAL_DATASETS_API_KEY`.

*   **Import**: `from agno.tools.financial_datasets import FinancialDatasetsTools`
*   **Tools**:
    *   `get_income_statements(ticker: str, period: str = "annual", limit: int = 10)`
    *   `get_balance_sheets(ticker: str, period: str = "annual", limit: int = 10)`
    *   `get_cash_flow_statements(ticker: str, period: str = "annual", limit: int = 10)`
    *   `get_company_info(ticker: str)`
    *   `get_stock_prices(ticker: str, interval: str = "1d", limit: int = 100)`
    *   `get_earnings(ticker: str, limit: int = 10)`
    *   `get_insider_trades(ticker: str, limit: int = 50)`
    *   `get_institutional_ownership(ticker: str)`
    *   `get_news(ticker: Optional[str] = None, limit: int = 50)`
    *   `get_sec_filings(ticker: str, form_type: Optional[str] = None, limit: int = 50)`
    *   `get_crypto_prices(symbol: str, interval: str = "1d", limit: int = 100)`
    *   `search_tickers(query: str, limit: int = 10)`

---

### `OpenBBTools`
Integrated investment research. Requires `OPENBB_PAT`.

*   **Import**: `from agno.tools.openbb import OpenBBTools`
*   **Parameters**: `provider` (e.g., "yfinance", "polygon", "fmp").
*   **Tools**:
    *   `get_stock_price(symbol: str)`
    *   `search_company_symbol(company_name: str)`
    *   `get_company_news(symbol: str, num_stories: int = 10)`
    *   `get_company_profile(symbol: str)`
    *   `get_price_targets(symbol: str)`

---

## 8. Business & Productivity

### `GoogleCalendarTools`
Manage your Google Calendar. Requires OAuth `credentials.json` or Environment Variables (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`).

*   **Import**: `from agno.tools.google.calendar import GoogleCalendarTools`
*   **Tools**:
    *   `list_events(limit: int = 10, start_date: str = None)`
    *   `create_event(start_date: str, end_date: str, title: str, description: str = None)`
    *   `update_event(event_id: str, title: str = None, description: str = None)`
    *   `delete_event(event_id: str)`
    *   `find_available_slots(start_date: str, end_date: str, duration_minutes: int = 30)`

---

### `ZoomTools`
Schedule and manage Zoom meetings. Requires `ZOOM_ACCOUNT_ID`, `ZOOM_CLIENT_ID`, and `ZOOM_CLIENT_SECRET`.

*   **Import**: `from agno.tools.zoom import ZoomTools`
*   **Tools**:
    *   `schedule_meeting(topic: str, start_time: str, duration: int, timezone: str = "UTC")`
    *   `list_meetings(user_id: str = "me")`
    *   `get_meeting_recordings(meeting_id: str)`
    *   `delete_meeting(meeting_id: str)`

---

### `SlackTools`
Real-time communication. Requires `SLACK_TOKEN`.

*   **Import**: `from agno.tools.slack import SlackTools`
*   **Tools**:
    *   `send_message(channel: str, text: str)`
    *   `send_message_thread(channel: str, text: str, thread_ts: str)`
    *   `list_channels()`
    *   `get_channel_history(channel: str, limit: int = 100)`
    *   `upload_file(channel: str, content: bytes, filename: str)`
    *   `download_file(file_id: str)`
    *   `search_messages(query: str)`
    *   `list_users()`

---

### `ShopifyTools`
E-commerce data and analytics. Requires `SHOPIFY_SHOP_NAME` and `SHOPIFY_ACCESS_TOKEN`.

*   **Import**: `from agno.tools.shopify import ShopifyTools`
*   **Tools**:
    *   `get_shop_info()`
    *   `get_products(max_results: int = 50)`
    *   `get_orders(max_results: int = 50)`
    *   `get_top_selling_products(limit: int = 10)`
    *   `get_sales_by_date_range(start_date: str, end_date: str)`
    *   `get_order_analytics()`
    *   `get_inventory_levels()`

---

### `ZendeskTools`
Customer support search. Requires `ZENDESK_USERNAME`, `ZENDESK_PASSWORD`, and `ZENDESK_COMPANY_NAME`.

*   **Import**: `from agno.tools.zendesk import ZendeskTools`
*   **Tools**:
    *   `search_zendesk(search_string: str)`: Searches Help Center articles.

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

## 6. Web & Search

### `DuckDuckGoTools`
Convenience wrapper around DuckDuckGo search. No API key required.

*   **Import**: `from agno.tools.duckduckgo import DuckDuckGoTools`
*   **Parameters**:
    *   `enable_search` (bool): Enable web search. Default: `True`.
    *   `enable_news` (bool): Enable news search. Default: `True`.
    *   `modifier` (str): Prepend text to all queries.
    *   `fixed_max_results` (int): Fix the number of results.
    *   `proxy` (str): Proxy URL.
    *   `timeout` (int): Request timeout (default 10s).
    *   `timelimit` (Literal["d", "w", "m", "y"]): Filter results by time.
    *   `region` (str): Region code (e.g., "us-en").
*   **Tools**:
    *   `duckduckgo_search(query: str, max_results: int = 5)`
    *   `duckduckgo_news(query: str, max_results: int = 5)`

---

### `TavilyTools`
AI-optimized search and extraction engine. Requires `TAVILY_API_KEY`.

*   **Import**: `from agno.tools.tavily import TavilyTools`
*   **Parameters**:
    *   `api_key` (str): Tavily API key.
    *   `search_depth` (Literal["basic", "advanced"]): Default: `advanced`.
    *   `max_tokens` (int): Max tokens for response. Default: `6000`.
    *   `include_answer` (bool): Include AI-generated answer. Default: `True`.
*   **Tools**:
    *   `web_search_using_tavily(query: str, max_results: int = 5)`: Returns JSON or Markdown search results.
    *   `extract_url_content(urls: str)`: Scrapes content from a list of URLs.

---

### `ExaTools`
Semantic neural search. Requires `EXA_API_KEY`.

*   **Import**: `from agno.tools.exa import ExaTools`
*   **Parameters**:
    *   `text_length_limit` (int): Character limit per result. Default: `1000`.
    *   `category` (str): Filter by category (e.g., "research paper", "news", "github").
*   **Tools**:
    *   `search_exa(query: str, num_results: int = 5, category: Optional[str] = None)`
    *   `get_contents(urls: list[str])`: Retrieves full page content.
    *   `find_similar(url: str, num_results: int = 5)`: Finds related links.
    *   `exa_answer(query: str)`: Returns an LLM-generated answer based on search.
    *   `research(instructions: str)`: Performs deep multi-step research.

---

### `SerperApiTools` & `SerpApiTools`
Google Search via third-party providers.

*   **Serper**: `from agno.tools.serper import SerperTools`. Env: `SERPER_API_KEY`.
    *   `search_web(query: str)`
    *   `search_news(query: str)`
    *   `search_scholar(query: str)`: Academic search.
    *   `scrape_webpage(url: str)`
*   **SerpApi**: `from agno.tools.serpapi import SerpApiTools`. Env: `SERP_API_KEY`.
    *   `search_google(query: str)`
    *   `search_youtube(query: str)`

---

### `BraveSearchTools`
Privacy-focused search. Requires `BRAVE_API_KEY`.

*   **Import**: `from agno.tools.bravesearch import BraveSearchTools`
*   **Tools**:
    *   `brave_search(query: str, max_results: int = 5, country: str = "US")`

---

### `SearxngTools`
Self-hosted meta-search engine interaction.

*   **Import**: `from agno.tools.searxng import SearxngTools`
*   **Parameters**:
    *   `host` (str): URL of your Searxng instance.
    *   `engines` (List[str]): List of backends to use.
*   **Tools**: `search_web`, `image_search`, `news_search`, `it_search`, `science_search`.

---

### `JinaTools`
Reader and search API that converts web content to clean Markdown/JSON. Requires `JINA_API_KEY`.

*   **Import**: `from agno.tools.jina import JinaReaderTools`
*   **Tools**:
    *   `read_url(url: str)`: Fetches a URL and returns clean content.
    *   `search_query(query: str)`: Searches Jina's index.

---

### `BaiduSearchTools`
Search Baidu (Chinese). Useful for regional data.

*   **Import**: `from agno.tools.baidusearch import BaiduSearchTools`
*   **Tools**:
    *   `baidu_search(query: str, max_results: int = 5, language: str = "zh")`

---

### `WebSearchTools`
Generic meta-search wrapper using `ddgs`.

*   **Import**: `from agno.tools.websearch import WebSearchTools`
*   **Parameters**: `backend` (e.g., "google", "bing", "duckduckgo", "brave").

## 6. Data Sources & Knowledge

### `ArxivTools`
Search and read scientific papers from arXiv.

*   **Import**: `from agno.tools.arxiv import ArxivTools`
*   **Tools**:
    *   `search_arxiv_and_return_articles(query: str, num_articles: int = 10)`
    *   `read_arxiv_papers(id_list: List[str], pages_to_read: int = None)`: Downloads and parses PDF content.

---

### `PubMedTools`
Search biological and biomedical literature.

*   **Import**: `from agno.tools.pubmed import PubmedTools`
*   **Parameters**: `email` (Recommended for NCBI E-utils), `results_expanded` (Boolean for full metadata).
*   **Tools**:
    *   `search_pubmed(query: str, max_results: int = 10)`

---

### `RedditTools`
Interacting with Reddit (Read/Write). Requires `REDDIT_CLIENT_ID` and `REDDIT_CLIENT_SECRET`.

*   **Import**: `from agno.tools.reddit import RedditTools`
*   **Authentication**:
    *   Read-only: Client ID + Secret.
    *   Write-access: Client ID + Secret + `REDDIT_USERNAME` + `REDDIT_PASSWORD`.
*   **Tools**:
    *   `get_top_posts(subreddit: str, time_filter: str = "week", limit: int = 10)`
    *   `get_user_info(username: str)`
    *   `create_post(subreddit: str, title: str, content: str)`
    *   `reply_to_post(post_id: str, content: str)`

---

### `WikipediaTools`
Fetch information from Wikipedia.

*   **Import**: `from agno.tools.wikipedia import WikipediaTools`
*   **Tools**:
    *   `search_wikipedia(query: str)`: Returns a content summary.
    *   `search_wikipedia_and_update_knowledge_base(topic: str)`: Populates an existing Knowledge Base.

---

### `YouTubeTools`
Extract data and captions from YouTube videos.

*   **Import**: `from agno.tools.youtube import YouTubeTools`
*   **Tools**:
    *   `get_youtube_video_data(url: str)`: Returns metadata (title, author).
    *   `get_youtube_video_captions(url: str)`: Full transcript.
    *   `get_video_timestamps(url: str)`: Captions mapped to time markers.

---

### `OpenWeatherTools`
Current weather, forecasts, and pollution data. Requires `OPENWEATHER_API_KEY`.

*   **Import**: `from agno.tools.openweather import OpenWeatherTools`
*   **Tools**:
    *   `get_current_weather(location: str)`
    *   `get_forecast(location: str)`
    *   `get_air_pollution(location: str)`
    *   `geocode_location(location: str)`

---

### `HackerNewsTools`
Get top stories and user details from HN.

*   **Import**: `from agno.tools.hackernews import HackerNewsTools`
*   **Tools**:
    *   `get_top_hackernews_stories(num_stories: int = 10)`
    *   `get_user_details(username: str)`

---

### `WebsiteTools`
Simple URL reader. Can integrate with a Knowledge Base.

*   **Import**: `from agno.tools.website import WebsiteTools`
*   **Tools**:
    *   `read_url(url: str)`: Returns clean text content.
    *   `add_website_to_knowledge(url: str)`: (Requires `Knowledge` object) Adds URL content to vector DB.

---

### `FirecrawlTools`
Advanced scraping/crawling. Requires `FIRECRAWL_API_KEY`.

*   **Import**: `from agno.tools.firecrawl import FirecrawlTools`
*   **Parameters**: `formats` (List of output formats), `limit` (Crawl limit).
*   **Tools**:
    *   `scrape_website(url: str)`
    *   `crawl_website(url: str, limit: int = 10)`
    *   `map_website(url: str)`: Returns a sitemap.
    *   `search_web(query: str)`

---

### `Crawl4aiTools`
Async-optimized crawler with content pruning.

*   **Import**: `from agno.tools.crawl4ai import Crawl4aiTools`
*   **Tools**:
    *   `crawl(url: Union[str, List[str]], search_query: str = None)`: Supports BM25 filtering via `search_query`.

---

### `BrowserbaseTools`
Headless browser in the cloud. Requires `BROWSERBASE_API_KEY` and `BROWSERBASE_PROJECT_ID`.

*   **Import**: `from agno.tools.browserbase import BrowserbaseTools`
*   **Tools**:
    *   `navigate_to(url: str)`
    *   `get_page_content()`: Returns visible text or raw HTML.
    *   `screenshot(path: str)`
    *   `close_session()`

---

### `BrightDataTools`
Premium scraping and structured data feeds. Requires `BRIGHT_DATA_API_KEY`.

*   **Import**: `from agno.tools.brightdata import BrightDataTools`
*   **Tools**:
    *   `scrape_as_markdown(url: str)`
    *   `get_screenshot(url: str)`
    *   `search_engine(query: str, engine: str = "google")`
    *   `web_data_feed(source_type: str, url: str)`: Supports LinkedIn, Amazon, Instagram, etc.

---

### `OxylabsTools`
Enterprise scraping. Requires `OXYLABS_USERNAME` and `OXYLABS_PASSWORD`.

*   **Import**: `from agno.tools.oxylabs import OxylabsTools`
*   **Tools**:
    *   `search_google(query: str)`
    *   `get_amazon_product(asin: str)`
    *   `search_amazon_products(query: str)`
    *   `scrape_website(url: str)`

---

### `ScrapeGraphTools`
AI-powered scraping using LLMs. Requires `SGAI_API_KEY`.

*   **Import**: `from agno.tools.scrapegraph import ScrapeGraphTools`
*   **Tools**:
    *   `smartscraper(url: str, prompt: str)`: Extraction via natural language.
    *   `markdownify(url: str)`
    *   `agentic_crawler(url: str, steps: List[str])`: Interactive actions (clicks, types).
    *   `searchscraper(user_prompt: str)`

---

### `Newspaper4k` & `Trafilatura`
Specialized text extraction toolkits.

*   **Newspaper4k**: `from agno.tools.newspaper4k import Newspaper4kTools`. Best for news articles.
    *   `read_article(url: str)`: Returns author, text, publish date.
*   **Trafilatura**: `from agno.tools.trafilatura import TrafilaturaTools`. High precision text extraction.
    *   `extract_text(url: str)`
    *   `extract_metadata_only(url: str)`
    *   `crawl_website(homepage_url: str)`

---

### `AgentQLTools`
Browser automation via AgentQL. Requires `AGENTQL_API_KEY`.

*   **Import**: `from agno.tools.agentql import AgentQLTools`
*   **Tools**:
    *   `scrape_website(url: str)`
    *   `custom_scrape_website(url: str)`: Uses `agentql_query`.

---

## 8. Popular Toolkit Quick Reference

| Toolkit | Category | Auth Req |
|---------|----------|----------|
| `DuckDuckGoTools` | Search | No |
| `YFinanceTools` | Finance | No |
| `GithubTools` | DevOps | Personal Access Token |
| `PostgresTools` | Database | Connection String |
| `GmailTools` | Google | OAuth2 / Credentials |
| `SlackTools` | Comm | Bot Token |

For the full list of 130+ tools, see the `agno/tools/` directory.
```

## 11. Communication & Productivity (Detailed)

### SlackTools (`agno.tools.slack`)
Detailed integration for Slack workspaces using the `slack-sdk`.

**Authentication**: Requires `SLACK_TOKEN` environment variable.
**Dependencies**: `pip install slack-sdk httpx`

#### Parameters
- `token` (str): Slack API token.
- `output_directory` (str): Local path for file operations.
- `all` (bool): Enable all tools.
- `max_file_size` (int): Default 1GB.

#### Source Code
```python
import base64
import json
from os import getenv
from pathlib import Path
from ssl import SSLContext
from typing import Any, Dict, List, Optional, Union

import httpx

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    from slack_sdk import WebClient
    from slack_sdk.errors import SlackApiError
except ImportError:
    raise ImportError("Slack tools require the `slack_sdk` package. Run `pip install slack-sdk` to install it.")


class SlackTools(Toolkit):
    def __init__(
        self,
        token: Optional[str] = None,
        markdown: bool = True,
        output_directory: Optional[str] = None,
        enable_send_message: bool = True,
        enable_send_message_thread: bool = True,
        enable_list_channels: bool = True,
        enable_get_channel_history: bool = True,
        enable_upload_file: bool = True,
        enable_download_file: bool = True,
        enable_search_messages: bool = False,
        enable_get_thread: bool = False,
        enable_list_users: bool = False,
        enable_get_user_info: bool = False,
        all: bool = False,
        ssl: Optional[SSLContext] = None,
        max_file_size: int = 1_073_741_824,  # 1GB
        thread_message_limit: int = 20,
        **kwargs,
    ):
        """
        Initialize the SlackTools class.

        Args:
            token (str): The Slack API token. Defaults to the SLACK_TOKEN environment variable.
            markdown (bool): Whether to enable Slack markdown formatting. Defaults to True.
            output_directory (str): Optional path to save downloaded/uploaded files locally.
            enable_send_message (bool): Whether to enable the send_message tool. Defaults to True.
            enable_send_message_thread (bool): Whether to enable the send_message_thread tool. Defaults to True.
            enable_list_channels (bool): Whether to enable the list_channels tool. Defaults to True.
            enable_get_channel_history (bool): Whether to enable the get_channel_history tool. Defaults to True.
            enable_upload_file (bool): Whether to enable the upload_file tool. Defaults to True.
            enable_download_file (bool): Whether to enable the download_file tool. Defaults to True.
            enable_search_messages (bool): Whether to enable the search_messages tool. Defaults to False.
            enable_get_thread (bool): Whether to enable the get_thread tool. Defaults to False.
            enable_list_users (bool): Whether to enable the list_users tool. Defaults to False.
            enable_get_user_info (bool): Whether to enable the get_user_info tool. Defaults to False.
            all (bool): Whether to enable all tools. Defaults to False.
            ssl (SSLContext): Optional SSL context for the Slack WebClient. Defaults to None.
            max_file_size (int): Maximum file size in bytes for uploads and downloads. Defaults to 1GB.
            thread_message_limit (int): Maximum number of messages to fetch in get_thread. Defaults to 20.
        """
        _token = token or getenv("SLACK_TOKEN")
        if not _token:
            raise ValueError("SLACK_TOKEN is not set")
        self.token: str = _token
        self.client = WebClient(token=self.token, ssl=ssl)
        self.markdown = markdown
        self.max_file_size = max_file_size
        self.thread_message_limit = thread_message_limit
        self.output_directory = Path(output_directory) if output_directory else None

        if self.output_directory:
            self.output_directory.mkdir(parents=True, exist_ok=True)
            log_debug(f"Uploaded files will be saved to: {self.output_directory}")

        tools: List[Any] = []
        if enable_send_message or all:
            tools.append(self.send_message)
        if enable_send_message_thread or all:
            tools.append(self.send_message_thread)
        if enable_list_channels or all:
            tools.append(self.list_channels)
        if enable_get_channel_history or all:
            tools.append(self.get_channel_history)
        if enable_upload_file or all:
            tools.append(self.upload_file)
        if enable_download_file or all:
            tools.append(self.download_file)
        if enable_search_messages or all:
            tools.append(self.search_messages)
        if enable_get_thread or all:
            tools.append(self.get_thread)
        if enable_list_users or all:
            tools.append(self.list_users)
        if enable_get_user_info or all:
            tools.append(self.get_user_info)

        super().__init__(name="slack", tools=tools, **kwargs)

    def send_message(self, channel: str, text: str) -> str:
        """Send a message to a Slack channel.

        Args:
            channel (str): The channel ID or name to send the message to.
            text (str): The text of the message to send. Supports Slack mrkdwn formatting.

        Returns:
            str: A JSON string containing the response from the Slack API.
        """
        try:
            response = self.client.chat_postMessage(channel=channel, text=text, mrkdwn=self.markdown)
            return json.dumps(response.data)
        except SlackApiError as e:
            logger.error(f"Error sending message: {e}")
            return json.dumps({"error": str(e)})

    def send_message_thread(self, channel: str, text: str, thread_ts: str) -> str:
        """Reply to a message thread in a Slack channel.

        Args:
            channel (str): The channel ID or name where the thread exists.
            text (str): The text of the reply. Supports Slack mrkdwn formatting.
            thread_ts (str): The timestamp of the parent message to reply to.

        Returns:
            str: A JSON string containing the response from the Slack API.
        """
        try:
            response = self.client.chat_postMessage(
                channel=channel, text=text, thread_ts=thread_ts, mrkdwn=self.markdown
            )
            return json.dumps(response.data)
        except SlackApiError as e:
            logger.error(f"Error sending message: {e}")
            return json.dumps({"error": str(e)})

    def list_channels(self) -> str:
        """List all channels in the Slack workspace.

        Returns:
            str: A JSON string containing a list of channels with their IDs and names.
        """
        try:
            response = self.client.conversations_list()
            channels = [{"id": channel["id"], "name": channel["name"]} for channel in response["channels"]]
            return json.dumps(channels)
        except SlackApiError as e:
            logger.error(f"Error listing channels: {e}")
            return json.dumps({"error": str(e)})

    def get_channel_history(self, channel: str, limit: int = 100) -> str:
        """Get the message history of a Slack channel.

        Args:
            channel (str): The channel ID to fetch history from.
            limit (int): The maximum number of messages to fetch. Defaults to 100.

        Returns:
            str: A JSON string containing the channel's message history.
        """
        try:
            response = self.client.conversations_history(channel=channel, limit=limit)
            messages: List[Dict[str, Any]] = [  # type: ignore
                {
                    "text": msg.get("text", ""),
                    "user": "webhook" if msg.get("subtype") == "bot_message" else msg.get("user", "unknown"),
                    "ts": msg.get("ts", ""),
                    "sub_type": msg.get("subtype", "unknown"),
                    "attachments": msg.get("attachments", []) if msg.get("subtype") == "bot_message" else "n/a",
                }
                for msg in response.get("messages", [])
            ]
            return json.dumps(messages)
        except SlackApiError as e:
            logger.error(f"Error getting channel history: {e}")
            return json.dumps({"error": str(e)})

    def _save_file_to_disk(self, content: bytes, filename: str) -> Optional[str]:
        """Save file to disk if output_directory is set. Return file path or None."""
        if not self.output_directory:
            return None

        file_path = self.output_directory / Path(filename).name
        try:
            file_path.write_bytes(content)
            log_debug(f"File saved to: {file_path}")
            return str(file_path)
        except OSError as e:
            logger.warning(f"Failed to save file locally: {e}")
            return None

    def upload_file(
        self,
        channel: str,
        content: Union[str, bytes],
        filename: str,
        title: Optional[str] = None,
        initial_comment: Optional[str] = None,
        thread_ts: Optional[str] = None,
    ) -> str:
        """Upload a file to a Slack channel.

        Args:
            channel (str): The channel ID to upload the file to.
            content (str or bytes): The file content. Text strings will be encoded to bytes.
            filename (str): The name for the uploaded file.
            title (str): An optional title for the file.
            initial_comment (str): An optional message to include with the file upload.
            thread_ts (str): The timestamp of a thread to upload the file into. Optional.

        Returns:
            str: A JSON string containing the response from the Slack API.
        """
        try:
            # Handle both string and bytes content
            if isinstance(content, str):
                content_bytes = content.encode("utf-8")
            else:
                content_bytes = content

            if len(content_bytes) > self.max_file_size:
                limit_mb = self.max_file_size / (1024 * 1024)
                actual_mb = len(content_bytes) / (1024 * 1024)
                return json.dumps(
                    {"error": f"File {filename} ({actual_mb:.1f}MB) exceeds {limit_mb:.0f}MB upload limit"}
                )

            # Save to disk if output_directory is set
            file_path = self._save_file_to_disk(content_bytes, filename)

            response = self.client.files_upload_v2(
                channel=channel,
                content=content_bytes,
                filename=filename,
                title=title,
                initial_comment=initial_comment,
                thread_ts=thread_ts,
            )

            # Copy to avoid mutating the SDK's response object
            result = dict(response.data)
            if file_path:
                result["local_path"] = file_path

            return json.dumps(result)
        except SlackApiError as e:
            logger.error(f"Error uploading file: {e}")
            return json.dumps({"error": str(e)})

    def download_file(self, file_id: str, dest_path: Optional[str] = None) -> str:
        """Download a file from Slack by its file ID.

        Args:
            file_id (str): The Slack file ID to download.
            dest_path (str): An optional destination path to save the file to. Must be within the configured output_directory.

        Returns:
            str: A JSON string containing the file metadata and either the local file path or base64-encoded content.
        """
        try:
            # Get file info from Slack API
            response = self.client.files_info(file=file_id)
            file_info = response["file"]

            url_private = file_info.get("url_private")
            if not url_private:
                return json.dumps({"error": "File URL not available"})

            filename = file_info.get("name", f"file_{file_id}")
            file_size = file_info.get("size", 0)

            if file_size > self.max_file_size:
                limit_mb = self.max_file_size / (1024 * 1024)
                actual_mb = file_size / (1024 * 1024)
                return json.dumps(
                    {"error": f"File {filename} ({actual_mb:.1f}MB) exceeds {limit_mb:.0f}MB download limit"}
                )

            # Download file content
            headers = {"Authorization": f"Bearer {self.token}"}
            download_response = httpx.get(url_private, headers=headers, timeout=30)
            download_response.raise_for_status()
            content = download_response.content

            # Determine where to save
            save_path: Optional[Path] = None
            if dest_path:
                save_path = Path(dest_path).resolve()
                if self.output_directory and not save_path.is_relative_to(self.output_directory.resolve()):
                    return json.dumps({"error": "dest_path must be within the configured output_directory"})
            elif self.output_directory:
                save_path = self.output_directory / Path(filename).name

            result: Dict[str, Any] = {
                "file_id": file_id,
                "filename": filename,
                "size": file_size,
            }

            # Save to disk or return as base64
            if save_path:
                try:
                    save_path.parent.mkdir(parents=True, exist_ok=True)
                    save_path.write_bytes(content)
                    log_debug(f"File downloaded to: {save_path}")
                    result["path"] = str(save_path)
                except OSError as e:
                    logger.warning(f"Failed to save file locally: {e}")
                    # Fall through to return base64
                    result["content_base64"] = base64.b64encode(content).decode("utf-8")
            else:
                result["content_base64"] = base64.b64encode(content).decode("utf-8")

            return json.dumps(result)

        except SlackApiError as e:
            logger.error(f"Error downloading file: {e}")
            return json.dumps({"error": str(e)})
        except httpx.HTTPError as e:
            logger.error(f"Error downloading file content: {e}")
            return json.dumps({"error": f"HTTP error: {str(e)}"})

    def download_file_bytes(self, file_id: str) -> Optional[bytes]:
        """Download file content as raw bytes. For internal use by interfaces."""
        try:
            response = self.client.files_info(file=file_id)
            file_info = response["file"]

            file_size = file_info.get("size", 0)
            if file_size > self.max_file_size:
                limit_mb = self.max_file_size / (1024 * 1024)
                actual_mb = file_size / (1024 * 1024)
                filename = file_info.get("name", file_id)
                logger.error(f"File {filename} ({actual_mb:.1f}MB) exceeds {limit_mb:.0f}MB download limit")
                return None

            url_private = file_info.get("url_private")
            if not url_private:
                return None

            headers = {"Authorization": f"Bearer {self.token}"}
            download_response = httpx.get(url_private, headers=headers, timeout=30)
            download_response.raise_for_status()
            return download_response.content

        except (SlackApiError, httpx.HTTPError) as e:
            logger.error(f"Error downloading file bytes: {e}")
            return None

    def search_messages(self, query: str, limit: int = 20) -> str:
        """Search messages across the Slack workspace.

        Args:
            query (str): The search query. Supports modifiers like from:@user, in:#channel, has:link, before:date, after:date.
            limit (int): The maximum number of results to return. Defaults to 20, max 100.

        Returns:
            str: A JSON string containing the count and list of matching messages with text, user, channel, timestamp, and permalink.
        """
        try:
            response = self.client.search_messages(query=query, count=min(limit, 100))
            matches = response.get("messages", {}).get("matches", [])
            messages = [
                {
                    "text": msg.get("text", ""),
                    "user": msg.get("user", "unknown"),
                    "channel_id": msg.get("channel", {}).get("id", ""),
                    "channel_name": msg.get("channel", {}).get("name", ""),
                    "ts": msg.get("ts", ""),
                    "permalink": msg.get("permalink", ""),
                }
                for msg in matches
            ]
            return json.dumps({"count": len(messages), "messages": messages})
        except SlackApiError as e:
            logger.error(f"Error searching messages: {e}")
            return json.dumps({"error": str(e)})

    def get_thread(self, channel: str, thread_ts: str, limit: int = 20) -> str:
        """Get all messages in a thread by the parent message's timestamp.

        Args:
            channel (str): The channel ID where the thread exists.
            thread_ts (str): The timestamp of the parent message.
            limit (int): The maximum number of replies to fetch. Defaults to 20. Capped by thread_message_limit.

        Returns:
            str: A JSON string containing the thread timestamp, reply count, and list of messages.
        """
        try:
            response = self.client.conversations_replies(
                channel=channel,
                ts=thread_ts,
                limit=min(limit, self.thread_message_limit),
            )
            messages = [
                {
                    "text": msg.get("text", ""),
                    "user": msg.get("user", "unknown"),
                    "ts": msg.get("ts", ""),
                }
                for msg in response.get("messages", [])
            ]
            return json.dumps(
                {
                    "thread_ts": thread_ts,
                    "reply_count": len(messages) - 1,
                    "messages": messages,
                }
            )
        except SlackApiError as e:
            logger.error(f"Error getting thread: {e}")
            return json.dumps({"error": str(e)})

    def list_users(self, limit: int = 100) -> str:
        """List all users in the Slack workspace.

        Args:
            limit (int): The maximum number of users to fetch. Defaults to 100.

        Returns:
            str: A JSON string containing the count and list of users with their ID, name, real name, title, and bot status.
        """
        try:
            response = self.client.users_list(limit=limit)
            users = [
                {
                    "id": member.get("id", ""),
                    "name": member.get("name", ""),
                    "real_name": member.get("profile", {}).get("real_name", ""),
                    "title": member.get("profile", {}).get("title", ""),
                    "is_bot": member.get("is_bot", False),
                }
                for member in response.get("members", [])
                if not member.get("deleted", False)
            ]
            return json.dumps({"count": len(users), "users": users})
        except SlackApiError as e:
            logger.error(f"Error listing users: {e}")
            return json.dumps({"error": str(e)})

    def get_user_info(self, user_id: str) -> str:
        """Get detailed information about a Slack user by their user ID.

        Args:
            user_id (str): The Slack user ID to look up.

        Returns:
            str: A JSON string containing the user's ID, name, real name, email, title, timezone, and bot status.
        """
        try:
            response = self.client.users_info(user=user_id)
            user = response.get("user", {})
            profile = user.get("profile", {})
            return json.dumps(
                {
                    "id": user.get("id", ""),
                    "name": user.get("name", ""),
                    "real_name": profile.get("real_name", ""),
                    "email": profile.get("email", ""),
                    "title": profile.get("title", ""),
                    "tz": user.get("tz", ""),
                    "is_bot": user.get("is_bot", False),
                }
            )
        except SlackApiError as e:
            logger.error(f"Error getting user info: {e}")
            return json.dumps({"error": str(e)})
```

---

### DiscordTools (`agno.tools.discord`)
Interact with Discord servers and channels.

**Authentication**: Requires `DISCORD_BOT_TOKEN` environment variable.
**Dependencies**: `pip install requests`

#### Parameters
- `bot_token` (str): Discord bot token.
- `enable_send_message` (bool): Default True.
- `all` (bool): Enable all tools.

#### Source Code
```python
"""Discord integration tools for interacting with Discord channels and servers."""

import json
from os import getenv
from typing import Any, Dict, List, Optional

import requests

from agno.tools import Toolkit
from agno.utils.log import logger


class DiscordTools(Toolkit):
    def __init__(
        self,
        bot_token: Optional[str] = None,
        enable_send_message: bool = True,
        enable_get_channel_messages: bool = True,
        enable_get_channel_info: bool = True,
        enable_list_channels: bool = True,
        enable_delete_message: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.bot_token = bot_token or getenv("DISCORD_BOT_TOKEN")
        if not self.bot_token:
            logger.error("Discord bot token is required")
            raise ValueError("Discord bot token is required")

        self.base_url = "https://discord.com/api/v10"
        self.headers = {
            "Authorization": f"Bot {self.bot_token}",
            "Content-Type": "application/json",
        }

        tools: List[Any] = []
        if enable_send_message or all:
            tools.append(self.send_message)
        if enable_get_channel_messages or all:
            tools.append(self.get_channel_messages)
        if enable_get_channel_info or all:
            tools.append(self.get_channel_info)
        if enable_list_channels or all:
            tools.append(self.list_channels)
        if enable_delete_message or all:
            tools.append(self.delete_message)

        super().__init__(name="discord", tools=tools, **kwargs)

    def _make_request(self, method: str, endpoint: str, data: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        """Make a request to Discord API."""
        url = f"{self.base_url}{endpoint}"
        response = requests.request(method, url, headers=self.headers, json=data)
        response.raise_for_status()
        return response.json() if response.text else {}

    def send_message(self, channel_id: str, message: str) -> str:
        """
        Send a message to a Discord channel.

        Args:
            channel_id (str): The ID of the channel to send the message to.
            message (str): The text of the message to send.

        Returns:
            str: A success message or error message.
        """
        try:
            data = {"content": message}
            self._make_request("POST", f"/channels/{channel_id}/messages", data)
            return f"Message sent successfully to channel {channel_id}"
        except Exception as e:
            logger.error(f"Error sending message: {e}")
            return f"Error sending message: {str(e)}"

    def get_channel_info(self, channel_id: str) -> str:
        """
        Get information about a Discord channel.

        Args:
            channel_id (str): The ID of the channel to get information about.

        Returns:
            str: A JSON string containing the channel information.
        """
        try:
            response = self._make_request("GET", f"/channels/{channel_id}")
            return json.dumps(response, indent=2)
        except Exception as e:
            logger.error(f"Error getting channel info: {e}")
            return f"Error getting channel info: {str(e)}"

    def list_channels(self, guild_id: str) -> str:
        """
        List all channels in a Discord server.

        Args:
            guild_id (str): The ID of the server to list channels from.

        Returns:
            str: A JSON string containing the list of channels.
        """
        try:
            response = self._make_request("GET", f"/guilds/{guild_id}/channels")
            return json.dumps(response, indent=2)
        except Exception as e:
            logger.error(f"Error listing channels: {e}")
            return f"Error listing channels: {str(e)}"

    def get_channel_messages(self, channel_id: str, limit: int = 100) -> str:
        """
        Get the message history of a Discord channel.

        Args:
            channel_id (str): The ID of the channel to fetch messages from.
            limit (int): The maximum number of messages to fetch. Defaults to 100.

        Returns:
            str: A JSON string containing the channel's message history.
        """
        try:
            response = self._make_request("GET", f"/channels/{channel_id}/messages?limit={limit}")
            return json.dumps(response, indent=2)
        except Exception as e:
            logger.error(f"Error getting messages: {e}")
            return f"Error getting messages: {str(e)}"

    def delete_message(self, channel_id: str, message_id: str) -> str:
        """
        Delete a message from a Discord channel.

        Args:
            channel_id (str): The ID of the channel containing the message.
            message_id (str): The ID of the message to delete.

        Returns:
            str: A success message or error message.
        """
        try:
            self._make_request("DELETE", f"/channels/{channel_id}/messages/{message_id}")
            return f"Message {message_id} deleted successfully from channel {channel_id}"
        except Exception as e:
            logger.error(f"Error deleting message: {e}")
            return f"Error deleting message: {str(e)}"

    @staticmethod
    def get_tool_name() -> str:
        """Get the name of the tool."""
        return "discord"

    @staticmethod
    def get_tool_description() -> str:
        """Get the description of the tool."""
        return "Tool for interacting with Discord channels and servers"

    @staticmethod
    def get_tool_config() -> dict:
        """Get the required configuration for the tool."""
        return {
            "bot_token": {"type": "string", "description": "Discord bot token for authentication", "required": True}
        }
```

---

### TelegramTools (`agno.tools.telegram`)
Send messages via Telegram bots.

**Authentication**: Requires `TELEGRAM_TOKEN` and `chat_id`.
**Dependencies**: `pip install httpx`

#### Parameters
- `chat_id` (Union[str, int]): Target chat ID.
- `token` (str): Bot token.

#### Source Code
```python
from os import getenv
from typing import Any, List, Optional, Union

import httpx

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger


class TelegramTools(Toolkit):
    base_url = "https://api.telegram.org"

    def __init__(
        self,
        chat_id: Union[str, int],
        token: Optional[str] = None,
        enable_send_message: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.token = token or getenv("TELEGRAM_TOKEN")
        if not self.token:
            logger.error("TELEGRAM_TOKEN not set. Please set the TELEGRAM_TOKEN environment variable.")

        self.chat_id = chat_id

        tools: List[Any] = []
        if all or enable_send_message:
            tools.append(self.send_message)

        super().__init__(name="telegram", tools=tools, **kwargs)

    def _call_post_method(self, method, *args, **kwargs):
        return httpx.post(f"{self.base_url}/bot{self.token}/{method}", *args, **kwargs)

    def send_message(self, message: str) -> str:
        """This function sends a message to the chat ID.

        :param message: The message to send.
        :return: The response from the API.
        """
        log_debug(f"Sending telegram message: {message}")
        response = self._call_post_method("sendMessage", json={"chat_id": self.chat_id, "text": message})
        try:
            response.raise_for_status()
            return response.text
        except httpx.HTTPStatusError as e:
            return f"An error occurred: {e}"
```


## 12. Project Management (Detailed)

### JiraTools (`agno.tools.jira`)
Interact with Jira Cloud or Server for issue tracking and project management.

**Authentication**: Requires `JIRA_SERVER_URL`, `JIRA_USERNAME`, and either `JIRA_PASSWORD` or `JIRA_TOKEN`.
**Dependencies**: `pip install jira`

#### Parameters
- `server_url` (str): URL of your Jira instance.
- `username` (str): Email or username.
- `password/token` (str): API token or password.
- `all` (bool): Enable all tools.

#### Source Code
```python
import json
from os import getenv
from typing import Any, List, Optional, cast

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    from jira import JIRA, Issue
except ImportError:
    raise ImportError("`jira` not installed. Please install using `pip install jira`")


class JiraTools(Toolkit):
    def __init__(
        self,
        server_url: Optional[str] = None,
        username: Optional[str] = None,
        password: Optional[str] = None,
        token: Optional[str] = None,
        enable_get_issue: bool = True,
        enable_create_issue: bool = True,
        enable_search_issues: bool = True,
        enable_add_comment: bool = True,
        enable_add_worklog: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.server_url = server_url or getenv("JIRA_SERVER_URL")
        self.username = username or getenv("JIRA_USERNAME")
        self.password = password or getenv("JIRA_PASSWORD")
        self.token = token or getenv("JIRA_TOKEN")

        if not self.server_url:
            raise ValueError("JIRA server URL not provided.")

        # Initialize JIRA client
        if self.token and self.username:
            auth = (self.username, self.token)
        elif self.username and self.password:
            auth = (self.username, self.password)
        else:
            auth = None

        if auth:
            self.jira = JIRA(server=self.server_url, basic_auth=cast(tuple[str, str], auth))
        else:
            self.jira = JIRA(server=self.server_url)

        tools: List[Any] = []
        if enable_get_issue or all:
            tools.append(self.get_issue)
        if enable_create_issue or all:
            tools.append(self.create_issue)
        if enable_search_issues or all:
            tools.append(self.search_issues)
        if enable_add_comment or all:
            tools.append(self.add_comment)
        if enable_add_worklog or all:
            tools.append(self.add_worklog)

        super().__init__(name="jira_tools", tools=tools, **kwargs)

    def get_issue(self, issue_key: str) -> str:
        """
        Retrieves issue details from Jira.

        :param issue_key: The key of the issue to retrieve.
        :return: A JSON string containing issue details.
        """
        try:
            issue = self.jira.issue(issue_key)
            issue = cast(Issue, issue)
            issue_details = {
                "key": issue.key,
                "project": issue.fields.project.key,
                "issuetype": issue.fields.issuetype.name,
                "reporter": issue.fields.reporter.displayName if issue.fields.reporter else "N/A",
                "summary": issue.fields.summary,
                "description": issue.fields.description or "",
            }
            log_debug(f"Issue details retrieved for {issue_key}: {issue_details}")
            return json.dumps(issue_details)
        except Exception as e:
            logger.error(f"Error retrieving issue {issue_key}: {e}")
            return json.dumps({"error": str(e)})

    def create_issue(self, project_key: str, summary: str, description: str, issuetype: str = "Task") -> str:
        """
        Creates a new issue in Jira.

        :param project_key: The key of the project in which to create the issue.
        :param summary: The summary of the issue.
        :param description: The description of the issue.
        :param issuetype: The type of issue to create.
        :return: A JSON string with the new issue's key and URL.
        """
        try:
            issue_dict = {
                "project": {"key": project_key},
                "summary": summary,
                "description": description,
                "issuetype": {"name": issuetype},
            }
            new_issue = self.jira.create_issue(fields=issue_dict)
            issue_url = f"{self.server_url}/browse/{new_issue.key}"
            log_debug(f"Issue created with key: {new_issue.key}")
            return json.dumps({"key": new_issue.key, "url": issue_url})
        except Exception as e:
            logger.error(f"Error creating issue in project {project_key}: {e}")
            return json.dumps({"error": str(e)})

    def search_issues(self, jql_str: str, max_results: int = 50) -> str:
        """
        Searches for issues using a JQL query.

        :param jql_str: The JQL query string.
        :param max_results: Maximum number of results to return.
        :return: A JSON string containing a list of dictionaries with issue details.
        """
        try:
            issues = self.jira.search_issues(jql_str, maxResults=max_results)
            results = []
            for issue in issues:
                issue = cast(Issue, issue)
                issue_details = {
                    "key": issue.key,
                    "summary": issue.fields.summary,
                    "status": issue.fields.status.name,
                    "assignee": issue.fields.assignee.displayName if issue.fields.assignee else "Unassigned",
                }
                results.append(issue_details)
            log_debug(f"Found {len(results)} issues for JQL '{jql_str}'")
            return json.dumps(results)
        except Exception as e:
            logger.error(f"Error searching issues with JQL '{jql_str}': {e}")
            return json.dumps([{"error": str(e)}])

    def add_comment(self, issue_key: str, comment: str) -> str:
        """
        Adds a comment to an issue.

        :param issue_key: The key of the issue.
        :param comment: The comment text.
        :return: A JSON string indicating success or containing an error message.
        """
        try:
            self.jira.add_comment(issue_key, comment)
            log_debug(f"Comment added to issue {issue_key}")
            return json.dumps({"status": "success", "issue_key": issue_key})
        except Exception as e:
            logger.error(f"Error adding comment to issue {issue_key}: {e}")
            return json.dumps({"error": str(e)})

    def add_worklog(self, issue_key: str, time_spent: str, comment: Optional[str] = None) -> str:
        """
        Adds a worklog entry to log time spent on a specific Jira issue.

        :param issue_key: The key of the issue to log work against (e.g., 'PROJ-123').
        :param time_spent: The amount of time spent. Use Jira's format, e.g., '2h', '30m', '1d 4h'.
        :param comment: An optional comment describing the work done.
        :return: A JSON string indicating success or containing an error message.
        """
        try:
            self.jira.add_worklog(issue=issue_key, timeSpent=time_spent, comment=comment)
            log_debug(f"Worklog of '{time_spent}' added to issue {issue_key}")
            return json.dumps({"status": "success", "issue_key": issue_key, "time_spent": time_spent})
        except Exception as e:
            logger.error(f"Error adding worklog to issue {issue_key}: {e}")
            return json.dumps({"error": str(e)})
```

---

### LinearTools (`agno.tools.linear`)
Manage Linear issues, teams, and workflows via GraphQL.

**Authentication**: Requires `LINEAR_API_KEY` (Personal Access Token).
**Dependencies**: `pip install requests`

#### Parameters
- `api_key` (str): Linear API key.

#### Source Code
```python
from os import getenv
from typing import Any, List, Optional

import requests

from agno.tools import Toolkit
from agno.utils.log import log_info, logger


class LinearTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        **kwargs,
    ):
        self.api_key = api_key or getenv("LINEAR_API_KEY")

        if not self.api_key:
            raise ValueError("Linear API key is required")

        self.endpoint = "https://api.linear.app/graphql"
        self.headers = {"Authorization": f"{self.api_key}"}

        tools: List[Any] = [
            self.get_user_details,
            self.get_teams_details,
            self.get_issue_details,
            self.create_issue,
            self.update_issue,
            self.get_user_assigned_issues,
            self.get_workflow_issues,
            self.get_high_priority_issues,
        ]

        super().__init__(name="linear_tools", tools=tools, **kwargs)

    def _execute_query(self, query, variables=None):
        """Helper method to execute GraphQL queries with optional variables."""

        try:
            response = requests.post(self.endpoint, json={"query": query, "variables": variables}, headers=self.headers)
            response.raise_for_status()

            data = response.json()

            if "errors" in data:
                logger.error(f"GraphQL Error: {data['errors']}")
                raise Exception(f"GraphQL Error: {data['errors']}")

            log_info("GraphQL query executed successfully.")
            return data.get("data")

        except requests.exceptions.RequestException as e:
            logger.error(f"Request error: {e}")
            raise

        except Exception as e:
            logger.error(f"Unexpected error: {e}")
            raise

    def get_user_details(self) -> Optional[str]:
        """
        Fetch authenticated user details.
        It will return the user's unique ID, name, and email address from the viewer object in the GraphQL response.

        Returns:
            str or None: A string containing user details like user id, name, and email.

        Raises:
            Exception: If an error occurs during the query execution or data retrieval.
        """

        query = """
        query Me {
          viewer {
            id
            name
            email
          }
        }
        """

        try:
            response = self._execute_query(query)

            if response.get("viewer"):
                user = response["viewer"]
                log_info(
                    f"Retrieved authenticated user details with name: {user['name']}, ID: {user['id']}, Email: {user['email']}"
                )
                return str(user)
            else:
                logger.error("Failed to retrieve the current user details")
                return None

        except Exception as e:
            logger.error(f"Error fetching authenticated user details: {e}")
            raise

    def get_teams_details(self) -> Optional[str]:
        """
        Fetch the list of authenticated teams.
        It will return the unique ID and team name for each team, from the viewer object in the GraphQL response.

        Returns:
            str or None: A dictionary containing team details like team name, id.

        Raises:
            Exception: If an error occurs during the query execution or data retrieval.
        """

        query = """
        query Teams {
          teams {
            nodes {
              id
              name
            }
          }
        }
        """

        try:
            response = self._execute_query(query)

            if response.get("teams"):
                teams = response["teams"]["nodes"]
                log_info(f"Retrieved authenticated team details: {teams}")
                return str(teams)
            else:
                logger.error("Failed to retrieve the current user details")
                return None

        except Exception as e:
            logger.error(f"Error fetching authenticated user details: {e}")
            raise

    def get_issue_details(self, issue_id: str) -> Optional[str]:
        """
        Retrieve details of a specific issue by issue ID.

        Args:
            issue_id (str): The unique identifier of the issue to retrieve.

        Returns:
            str or None: A string containing issue details like issue id, issue title, and issue description.
                  Returns `None` if the issue is not found.

        Raises:
            Exception: If an error occurs during the query execution or data retrieval.
        """

        query = """
        query IssueDetails ($issueId: String!){
        issue(id: $issueId) {
          id
          title
          description
          }
        }
        """
        variables = {"issueId": issue_id}
        try:
            response = self._execute_query(query, variables)

            if response.get("issue"):
                issue = response["issue"]
                log_info(f"Issue '{issue['title']}' retrieved successfully with ID {issue['id']}.")
                return str(issue)
            else:
                logger.error(f"Failed to retrieve issue with ID {issue_id}.")
                return None

        except Exception as e:
            logger.error(f"Error retrieving issue with ID {issue_id}: {e}")
            raise

    def create_issue(
        self,
        title: str,
        description: str,
        team_id: str,
        project_id: Optional[str] = None,
        assignee_id: Optional[str] = None,
    ) -> Optional[str]:
        """
        Create a new issue within a specific project and team.

        Args:
            title (str): The title of the new issue.
            description (str): The description of the new issue.
            team_id (str): The unique identifier of the team in which to create the issue.
            project_id (Optional[str]): The ID of the project (optional).
            assignee_id (Optional[str]): The ID of the assignee (optional).

        Returns:
            str or None: A string containing the created issue's details like issue id and issue title.
                  Returns `None` if the issue creation fails.

        Raises:
            Exception: If an error occurs during the mutation execution or data retrieval.
        """

        query = """
        mutation IssueCreate ($title: String!, $description: String!, $teamId: String!, $projectId: String, $assigneeId: String){
          issueCreate(
            input: { title: $title, description: $description, teamId: $teamId, projectId: $projectId, assigneeId: $assigneeId}
          ) {
            success
            issue {
              id
              title
              url
            }
          }
        }
        """

        variables = {
            "title": title,
            "description": description,
            "teamId": team_id,
        }
        if project_id is not None:
            variables["projectId"] = project_id
        if assignee_id is not None:
            variables["assigneeId"] = assignee_id

        try:
            response = self._execute_query(query, variables)
            log_info(f"Response: {response}")

            if response["issueCreate"]["success"]:
                issue = response["issueCreate"]["issue"]
                log_info(f"Issue '{issue['title']}' created successfully with ID {issue['id']}")
                return str(issue)
            else:
                logger.error("Issue creation failed.")
                return None

        except Exception as e:
            logger.error(f"Error creating issue '{title}' for team ID {team_id}: {e}")
            raise

    def update_issue(self, issue_id: str, title: Optional[str]) -> Optional[str]:
        """
        Update the title or state of a specific issue by issue ID.

        Args:
            issue_id (str): The unique identifier of the issue to update.
            title (str, optional): The new title for the issue. If None, the title remains unchanged.

        Returns:
            str or None: A string containing the updated issue's details with issue id, issue title, and issue state (which includes `id` and `name`).
                  Returns `None` if the update is unsuccessful.

        Raises:
            Exception: If an error occurs during the mutation execution or data retrieval.
        """

        query = """
        mutation IssueUpdate ($issueId: String!, $title: String!){
          issueUpdate(
            id: $issueId,
            input: { title: $title}
          ) {
            success
            issue {
              id
              title
              state {
                id
                name
              }
            }
          }
        }
        """
        variables = {"issueId": issue_id, "title": title}

        try:
            response = self._execute_query(query, variables)

            if response["issueUpdate"]["success"]:
                issue = response["issueUpdate"]["issue"]
                log_info(f"Issue ID {issue_id} updated successfully.")
                return str(issue)
            else:
                logger.error(f"Failed to update issue ID {issue_id}. Success flag was false.")
                return None

        except Exception as e:
            logger.error(f"Error updating issue ID {issue_id}: {e}")
            raise

    def get_user_assigned_issues(self, user_id: str) -> Optional[str]:
        """
        Retrieve issues assigned to a specific user by user ID.

        Args:
            user_id (str): The unique identifier of the user for whom to retrieve assigned issues.

        Returns:
            str or None: A string representing the assigned issues to user id,
            where each issue contains issue details (e.g., `id`, `title`).
            Returns None if the user or issues cannot be retrieved.

        Raises:
            Exception: If an error occurs while querying for the user's assigned issues.
        """

        query = """
        query UserAssignedIssues($userId: String!) {
        user(id: $userId) {
          id
          name
          assignedIssues {
            nodes {
              id
              title
              }
            }
          }
        }
        """
        variables = {"userId": user_id}

        try:
            response = self._execute_query(query, variables)

            if response.get("user"):
                user = response["user"]
                issues = user["assignedIssues"]["nodes"]
                log_info(f"Retrieved {len(issues)} issues assigned to user '{user['name']}' (ID: {user['id']}).")
                return str(issues)
            else:
                logger.error("Failed to retrieve user or issues.")
                return None

        except Exception as e:
            logger.error(f"Error retrieving issues for user ID {user_id}: {e}")
            raise

    def get_workflow_issues(self, workflow_id: str) -> Optional[str]:
        """
        Retrieve issues within a specific workflow state by workflow ID.

        Args:
            workflow_id (str): The unique identifier of the workflow state to retrieve issues from.

        Returns:
            str or None: A string representing the issues within the specified workflow state,
            where each issue contains details of an issue (e.g., `title`).
            Returns None if no issues are found or if the workflow state cannot be retrieved.

        Raises:
            Exception: If an error occurs while querying issues for the specified workflow state.
        """

        query = """
        query WorkflowStateIssues($workflowId: String!) {
        workflowState(id: $workflowId) {
          issues {
            nodes {
              title
              }
            }
          }
        }
        """
        variables = {"workflowId": workflow_id}
        try:
            response = self._execute_query(query, variables)

            if response.get("workflowState"):
                issues = response["workflowState"]["issues"]["nodes"]
                log_info(f"Retrieved {len(issues)} issues in workflow state ID {workflow_id}.")
                return str(issues)
            else:
                logger.error("Failed to retrieve issues for the specified workflow state.")
                return None

        except Exception as e:
            logger.error(f"Error retrieving issues for workflow state ID {workflow_id}: {e}")
            raise

    def get_high_priority_issues(self) -> Optional[str]:
        """
        Retrieve issues with a high priority (priority <= 2).

        Returns:
            str or None: A str representing high-priority issues, where it
            contains details of an issue (e.g., `id`, `title`, `priority`).
            Returns None if no issues are retrieved.

        Raises:
            Exception: If an error occurs during the query process.
        """

        query = """
        query HighPriorityIssues {
        issues(filter: {
          priority: { lte: 2 }
        }) {
          nodes {
            id
            title
            priority
            }
          }
        }
        """
        try:
            response = self._execute_query(query)

            if response.get("issues"):
                high_priority_issues = response["issues"]["nodes"]
                log_info(f"Retrieved {len(high_priority_issues)} high-priority issues.")
                return str(high_priority_issues)
            else:
                logger.error("Failed to retrieve high-priority issues.")
                return None

        except Exception as e:
            logger.error(f"Error retrieving high-priority issues: {e}")
            raise
```

---

### ClickUpTools (`agno.tools.clickup`)
Lifecycle management for ClickUp tasks and spaces.

**Authentication**: Requires `CLICKUP_API_KEY` and `MASTER_SPACE_ID`.
**Dependencies**: `pip install requests`

#### Parameters
- `api_key` (str): ClickUp API token.
- `master_space_id` (str): Team/Workspace ID.

#### Source Code
```python
import json
import re
from os import getenv
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    import requests
except ImportError:
    raise ImportError("`requests` not installed. Please install using `pip install requests`")


class ClickUpTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        master_space_id: Optional[str] = None,
        **kwargs,
    ):
        self.api_key = api_key or getenv("CLICKUP_API_KEY")
        self.master_space_id = master_space_id or getenv("MASTER_SPACE_ID")
        self.base_url = "https://api.clickup.com/api/v2"
        self.headers = {"Authorization": self.api_key}

        if not self.api_key:
            raise ValueError("CLICKUP_API_KEY not set. Please set the CLICKUP_API_KEY environment variable.")
        if not self.master_space_id:
            raise ValueError("MASTER_SPACE_ID not set. Please set the MASTER_SPACE_ID environment variable.")

        tools: List[Any] = [
            self.list_tasks,
            self.create_task,
            self.get_task,
            self.update_task,
            self.delete_task,
            self.list_spaces,
            self.list_lists,
        ]

        super().__init__(name="clickup", tools=tools, **kwargs)

    def _make_request(
        self, method: str, endpoint: str, params: Optional[Dict] = None, data: Optional[Dict] = None
    ) -> Dict[str, Any]:
        """Make a request to the ClickUp API."""
        url = f"{self.base_url}/{endpoint}"
        try:
            response = requests.request(method=method, url=url, headers=self.headers, params=params, json=data)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.error(f"Error making request to {url}: {e}")
            return {"error": str(e)}

    def _find_by_name(self, items: List[Dict[str, Any]], name: str) -> Optional[Dict[str, Any]]:
        """Find an item in a list by name using exact match or regex pattern.

        Args:
            items: List of items to search through
            name: Name to search for

        Returns:
            Matching item or None if not found
        """
        if not name:
            return items[0] if items else None

        pattern = re.compile(name, re.IGNORECASE)
        for item in items:
            # Try exact match first (case-insensitive)
            if item["name"].lower() == name.lower():
                return item
            # Then try regex pattern match
            if pattern.search(item["name"]):
                return item
        return None

    def _get_space(self, space_name: str) -> Dict[str, Any]:
        """Get space information by name."""
        spaces = self._make_request("GET", f"team/{self.master_space_id}/space")
        if "error" in spaces:
            return spaces

        spaces_list = spaces.get("spaces", [])
        if not spaces_list:
            return {"error": "No spaces found"}

        space = self._find_by_name(spaces_list, space_name)
        if not space:
            return {"error": f"Space '{space_name}' not found"}
        return space

    def _get_list(self, space_id: str, list_name: str) -> Dict[str, Any]:
        """Get list information by name."""
        lists = self._make_request("GET", f"space/{space_id}/list")
        if "error" in lists:
            return lists

        lists_data = lists.get("lists", [])
        if not lists_data:
            return {"error": "No lists found in space"}

        list_item = self._find_by_name(lists_data, list_name)
        if not list_item:
            return {"error": f"List '{list_name}' not found"}
        return list_item

    def _get_tasks(self, list_id: str) -> List[Dict[str, Any]]:
        """Get tasks in a list, optionally filtered by name."""
        tasks = self._make_request("GET", f"list/{list_id}/task")
        if "error" in tasks:
            return []

        tasks_data = tasks.get("tasks", [])
        return tasks_data

    def list_tasks(self, space_name: str) -> str:
        """List all tasks in a space.

        Args:
            space_name (str): Name of the space to list tasks from

        Returns:
            str: JSON string containing tasks
        """
        # Get space
        space = self._get_space(space_name)
        if "error" in space:
            return json.dumps(space, indent=2)

        # Get lists
        lists = self._make_request("GET", f"space/{space['id']}/list")
        lists_data = lists.get("lists", [])
        if not lists_data:
            return json.dumps({"error": f"No lists found in space '{space_name}'"}, indent=2)

        # Get tasks from all lists
        all_tasks = []
        for list_info in lists_data:
            tasks = self._get_tasks(list_info["id"])
            for task in tasks:
                task["list_name"] = list_info["name"]  # Add list name for context
            all_tasks.extend(tasks)

        return json.dumps({"tasks": all_tasks}, indent=2)

    def create_task(self, space_name: str, task_name: str, task_description: str) -> str:
        """Create a new task in a space.

        Args:
            space_name (str): Name of the space to create task in
            task_name (str): Name of the task
            task_description (str): Description of the task

        Returns:
            str: JSON string containing created task details
        """
        # Get space
        space = self._get_space(space_name)
        if "error" in space:
            return json.dumps(space, indent=2)

        # Get first list in space
        response = self._make_request("GET", f"space/{space['id']}/list")
        log_debug(f"Lists: {response}")
        lists_data = response.get("lists", [])
        if not lists_data:
            return json.dumps({"error": f"No lists found in space '{space_name}'"}, indent=2)

        list_info = lists_data[0]  # Use first list

        # Create task
        data = {"name": task_name, "description": task_description}

        task = self._make_request("POST", f"list/{list_info['id']}/task", data=data)
        return json.dumps(task, indent=2)

    def list_spaces(self) -> str:
        """List all spaces in the workspace.

        Returns:
            str: JSON string containing list of spaces
        """
        spaces = self._make_request("GET", f"team/{self.master_space_id}/space")
        return json.dumps(spaces, indent=2)

    def list_lists(self, space_name: str) -> str:
        """List all lists in a space.

        Args:
            space_name (str): Name of the space to list lists from

        Returns:
            str: JSON string containing list of lists
        """
        # Get space
        space = self._get_space(space_name)
        if "error" in space:
            return json.dumps(space, indent=2)

        # Get lists
        lists = self._make_request("GET", f"space/{space['id']}/list")
        return json.dumps(lists, indent=2)

    def get_task(self, task_id: str) -> str:
        """Get details of a specific task.

        Args:
            task_id (str): The ID of the task

        Returns:
            str: JSON string containing task details
        """
        task = self._make_request("GET", f"task/{task_id}")
        return json.dumps(task, indent=2)

    def update_task(self, task_id: str, **kwargs) -> str:
        """Update a specific task.

        Args:
            task_id (str): The ID of the task
            **kwargs: Task fields to update (name, description, status, priority, etc.)

        Returns:
            str: JSON string containing updated task details
        """
        task = self._make_request("PUT", f"task/{task_id}", data=kwargs)
        return json.dumps(task, indent=2)

    def delete_task(self, task_id: str) -> str:
        """Delete a specific task.

        Args:
            task_id (str): The ID of the task

        Returns:
            str: JSON string containing deletion status
        """
        result = self._make_request("DELETE", f"task/{task_id}")
        if "error" not in result:
            result = {"success": True, "message": f"Task {task_id} deleted successfully"}
        return json.dumps(result, indent=2)
```

---

### TrelloTools (`agno.tools.trello`)
Manage Trello boards, lists, and cards.

**Authentication**: Requires `TRELLO_API_KEY`, `TRELLO_API_SECRET`, and `TRELLO_TOKEN`.
**Dependencies**: `pip install py-trello`

#### Parameters
- `api_key` (str): Trello API key.
- `api_secret` (str): Trello API secret.
- `token` (str): Trello user token.

#### Source Code
```python
import json
from os import getenv
from typing import Any, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info, logger

try:
    from trello import TrelloClient  # type: ignore
except ImportError:
    raise ImportError("`py-trello` not installed.")


class TrelloTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        api_secret: Optional[str] = None,
        token: Optional[str] = None,
        **kwargs,
    ):
        self.api_key = api_key or getenv("TRELLO_API_KEY")
        self.api_secret = api_secret or getenv("TRELLO_API_SECRET")
        self.token = token or getenv("TRELLO_TOKEN")

        if not all([self.api_key, self.api_secret, self.token]):
            logger.warning("Missing Trello credentials")

        try:
            self.client = TrelloClient(api_key=self.api_key, api_secret=self.api_secret, token=self.token)
        except Exception as e:
            logger.error(f"Error initializing Trello client: {e}")
            self.client = None

        tools: List[Any] = [
            self.create_card,
            self.get_board_lists,
            self.move_card,
            self.get_cards,
            self.create_board,
            self.create_list,
            self.list_boards,
        ]

        super().__init__(name="trello", tools=tools, **kwargs)

    def create_card(self, board_id: str, list_name: str, card_title: str, description: str = "") -> str:
        """
        Create a new card in the specified board and list.

        Args:
            board_id (str): ID of the board to create the card in
            list_name (str): Name of the list to add the card to
            card_title (str): Title of the card
            description (str): Description of the card

        Returns:
            str: JSON string containing card details or error message
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_info(f"Creating card {card_title}")

            board = self.client.get_board(board_id)
            target_list = None

            for lst in board.list_lists():
                if lst.name.lower() == list_name.lower():
                    target_list = lst
                    break

            if not target_list:
                return f"List '{list_name}' not found on board"

            card = target_list.add_card(name=card_title, desc=description)

            return json.dumps({"id": card.id, "name": card.name, "url": card.url, "list": list_name})

        except Exception as e:
            return f"Error creating card: {e}"

    def get_board_lists(self, board_id: str) -> str:
        """
        Get all lists on a board.

        Args:
            board_id (str): ID of the board

        Returns:
            str: JSON string containing lists information
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_debug(f"Getting lists for board {board_id}")

            board = self.client.get_board(board_id)
            lists = board.list_lists()

            lists_info = [{"id": lst.id, "name": lst.name, "cards_count": len(lst.list_cards())} for lst in lists]

            return json.dumps({"lists": lists_info})

        except Exception as e:
            return f"Error getting board lists: {e}"

    def move_card(self, card_id: str, list_id: str) -> str:
        """
        Move a card to a different list.

        Args:
            card_id (str): ID of the card to move
            list_id (str): ID of the destination list

        Returns:
            str: JSON string containing result of the operation
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_debug(f"Moving card {card_id} to list {list_id}")

            card = self.client.get_card(card_id)
            card.change_list(list_id)

            return json.dumps({"success": True, "card_id": card_id, "new_list_id": list_id})

        except Exception as e:
            return f"Error moving card: {e}"

    def get_cards(self, list_id: str) -> str:
        """
        Get all cards in a list.

        Args:
            list_id (str): ID of the list

        Returns:
            str: JSON string containing cards information
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_debug(f"Getting cards for list {list_id}")

            trello_list = self.client.get_list(list_id)
            cards = trello_list.list_cards()

            cards_info = [
                {
                    "id": card.id,
                    "name": card.name,
                    "description": card.description,
                    "url": card.url,
                    "labels": [label.name for label in card.labels],
                }
                for card in cards
            ]

            return json.dumps({"cards": cards_info})

        except Exception as e:
            return f"Error getting cards: {e}"

    def create_board(self, name: str, default_lists: bool = False) -> str:
        """
        Create a new Trello board.

        Args:
            name (str): Name of the board
            default_lists (bool): Whether the default lists should be created

        Returns:
            str: JSON string containing board details or error message
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_info(f"Creating board {name}")

            board = self.client.add_board(board_name=name, default_lists=default_lists)

            return json.dumps(
                {
                    "id": board.id,
                    "name": board.name,
                    "url": board.url,
                }
            )

        except Exception as e:
            return f"Error creating board: {e}"

    def create_list(self, board_id: str, list_name: str, pos: str = "bottom") -> str:
        """
        Create a new list on a specified board.

        Args:
            board_id (str): ID of the board to create the list in
            list_name (str): Name of the new list
            pos (str): Position of the list - 'top', 'bottom', or a positive number

        Returns:
            str: JSON string containing list details or error message
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_info(f"Creating list {list_name}")

            board = self.client.get_board(board_id)
            new_list = board.add_list(name=list_name, pos=pos)

            return json.dumps(
                {
                    "id": new_list.id,
                    "name": new_list.name,
                    "pos": new_list.pos,
                    "board_id": board_id,
                }
            )

        except Exception as e:
            return f"Error creating list: {e}"

    def list_boards(self, board_filter: str = "all") -> str:
        """
        Get a list of all boards for the authenticated user.

        Args:
            board_filter (str): Filter for boards. Options: 'all', 'open', 'closed',
                              'organization', 'public', 'starred'. Defaults to 'all'.

        Returns:
            str: JSON string containing list of boards
        """
        try:
            if not self.client:
                return "Trello client not initialized"

            log_debug(f"Listing boards with filter: {board_filter}")

            boards = self.client.list_boards(board_filter=board_filter)

            boards_list = []
            for board in boards:
                board_data = {
                    "id": board.id,
                    "name": board.name,
                    "description": getattr(board, "description", ""),
                    "url": board.url,
                    "closed": board.closed,
                    "starred": getattr(board, "starred", False),
                    "organization": getattr(board, "idOrganization", None),
                }
                boards_list.append(board_data)

            return json.dumps(
                {
                    "filter_used": board_filter,
                    "total_boards": len(boards_list),
                    "boards": boards_list,
                }
            )

        except Exception as e:
            return f"Error listing boards: {e}"
```

---

### NotionTools (`agno.tools.notion`)
Create and manage Notion pages and databases.

**Authentication**: Requires `NOTION_API_KEY` (Integration Token) and `NOTION_DATABASE_ID`.
**Dependencies**: `pip install notion-client httpx`

#### Parameters
- `api_key` (str): Notion integration token.
- `database_id` (str): Targeted database ID.

#### Source Code
```python
import json
import os
from typing import Any, Dict, List, Optional, cast

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    from notion_client import Client
except ImportError:
    raise ImportError("`notion-client` not installed. Please install using `pip install notion-client`")


class NotionTools(Toolkit):
    """
    Notion toolkit for creating and managing Notion pages.

    Args:
        api_key (Optional[str]): Notion API key (integration token). If not provided, uses NOTION_API_KEY env var.
        database_id (Optional[str]): The ID of the database to work with. If not provided, uses NOTION_DATABASE_ID env var.
        enable_create_page (bool): Enable creating pages. Default is True.
        enable_update_page (bool): Enable updating pages. Default is True.
        enable_search_pages (bool): Enable searching pages. Default is True.
        all (bool): Enable all tools. Overrides individual flags when True. Default is False.
    """

    def __init__(
        self,
        api_key: Optional[str] = None,
        database_id: Optional[str] = None,
        enable_create_page: bool = True,
        enable_update_page: bool = True,
        enable_search_pages: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.api_key = api_key or os.getenv("NOTION_API_KEY")
        self.database_id = database_id or os.getenv("NOTION_DATABASE_ID")

        if not self.api_key:
            raise ValueError(
                "Notion API key is required. Either pass api_key parameter or set NOTION_API_KEY environment variable."
            )
        if not self.database_id:
            raise ValueError(
                "Notion database ID is required. Either pass database_id parameter or set NOTION_DATABASE_ID environment variable."
            )

        self.client = Client(auth=self.api_key)

        tools: List[Any] = []
        if all or enable_create_page:
            tools.append(self.create_page)
        if all or enable_update_page:
            tools.append(self.update_page)
        if all or enable_search_pages:
            tools.append(self.search_pages)

        super().__init__(name="notion_tools", tools=tools, **kwargs)

    def create_page(self, title: str, tag: str, content: str) -> str:
        """Create a new page in the Notion database with a title, tag, and content.

        Args:
            title (str): The title of the page
            tag (str): The tag/category for the page (e.g., travel, tech, general-blogs, fashion, documents)
            content (str): The content to add to the page

        Returns:
            str: JSON string with page creation details
        """
        try:
            log_debug(f"Creating Notion page with title: {title}, tag: {tag}")

            # Create the page in the database
            new_page = cast(
                Dict[str, Any],
                self.client.pages.create(
                    parent={"database_id": self.database_id},
                    properties={"Name": {"title": [{"text": {"content": title}}]}, "Tag": {"select": {"name": tag}}},
                    children=[
                        {
                            "object": "block",
                            "type": "paragraph",
                            "paragraph": {"rich_text": [{"type": "text", "text": {"content": content}}]},
                        }
                    ],
                ),
            )

            result = {"success": True, "page_id": new_page["id"], "url": new_page["url"], "title": title, "tag": tag}
            return json.dumps(result, indent=2)

        except Exception as e:
            logger.exception(e)
            return json.dumps({"success": False, "error": str(e)})

    def update_page(self, page_id: str, content: str) -> str:
        """Add content to an existing Notion page.

        Args:
            page_id (str): The ID of the page to update
            content (str): The content to append to the page

        Returns:
            str: JSON string with update status
        """
        try:
            log_debug(f"Updating Notion page: {page_id}")

            # Append content to the page
            self.client.blocks.children.append(
                block_id=page_id,
                children=[
                    {
                        "object": "block",
                        "type": "paragraph",
                        "paragraph": {"rich_text": [{"type": "text", "text": {"content": content}}]},
                    }
                ],
            )

            result = {"success": True, "page_id": page_id, "message": "Content added successfully"}
            return json.dumps(result, indent=2)

        except Exception as e:
            logger.exception(e)
            return json.dumps({"success": False, "error": str(e)})

    def search_pages(self, tag: str) -> str:
        """Search for pages in the database by tag.

        Args:
            tag (str): The tag to search for

        Returns:
            str: JSON string with list of matching pages
        """
        try:
            log_debug(f"Searching for pages with tag: {tag}")

            import httpx

            headers = {
                "Authorization": f"Bearer {self.api_key}",
                "Notion-Version": "2022-06-28",
                "Content-Type": "application/json",
            }

            payload = {"filter": {"property": "Tag", "select": {"equals": tag}}}

            # The SDK client does not support the query method
            response = httpx.post(
                f"https://api.notion.com/v1/databases/{self.database_id}/query",
                headers=headers,
                json=payload,
                timeout=30.0,
            )

            if response.status_code != 200:
                return json.dumps(
                    {
                        "success": False,
                        "error": f"API request failed with status {response.status_code}",
                        "message": response.text,
                    }
                )

            data = response.json()
            pages = []

            for page in data.get("results", []):
                try:
                    page_title = "Untitled"
                    if page.get("properties", {}).get("Name", {}).get("title"):
                        page_title = page["properties"]["Name"]["title"][0]["text"]["content"]

                    page_tag = None
                    if page.get("properties", {}).get("Tag", {}).get("select"):
                        page_tag = page["properties"]["Tag"]["select"]["name"]

                    page_info = {
                        "page_id": page["id"],
                        "title": page_title,
                        "tag": page_tag,
                        "url": page.get("url", ""),
                    }
                    pages.append(page_info)
                except Exception as page_error:
                    log_debug(f"Error parsing page: {page_error}")
                    continue

            result = {"success": True, "count": len(pages), "pages": pages}
            return json.dumps(result, indent=2)

        except Exception as e:
            logger.exception(e)
            return json.dumps(
                {
                    "success": False,
                    "error": str(e),
                    "message": "Failed to search pages. Make sure the database is shared with the integration and has a 'Tag' property.",
                }
            )
```

---

### ZendeskTools (`agno.tools.zendesk`)
Search Help Center articles in Zendesk.

**Authentication**: Requires `ZENDESK_USERNAME`, `ZENDESK_PASSWORD`, and `ZENDESK_COMPANY_NAME`.
**Dependencies**: `pip install requests`

#### Parameters
- `username` (str): Zendesk email.
- `password` (str): Zendesk API token or password.
- `company_name` (str): Subdomain (e.g., 'acme' in acme.zendesk.com).

#### Source Code
```python
import json
import re
from os import getenv
from typing import Any, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    import requests
except ImportError:
    raise ImportError("`requests` not installed. Please install using `pip install requests`.")


class ZendeskTools(Toolkit):
    """
    A toolkit class for interacting with the Zendesk API to search articles.
    It requires authentication details and the company name to configure the API access.
    """

    def __init__(
        self,
        username: Optional[str] = None,
        password: Optional[str] = None,
        company_name: Optional[str] = None,
        enable_search_zendesk: bool = True,
        all: bool = False,
        **kwargs,
    ):
        """
        Initializes the ZendeskTools class with necessary authentication details
        and registers the search_zendesk method.

        Parameters:
        username (str): The username for Zendesk API authentication.
        password (str): The password for Zendesk API authentication.
        company_name (str): The company name to form the base URL for API requests.
        enable_search_zendesk (bool): Whether to enable the search functionality.
        all (bool): Enable all functions.
        """
        self.username = username or getenv("ZENDESK_USERNAME")
        self.password = password or getenv("ZENDESK_PASSWORD")
        self.company_name = company_name or getenv("ZENDESK_COMPANY_NAME")

        if not self.username or not self.password or not self.company_name:
            logger.error("Username, password, or company name not provided.")

        tools: List[Any] = []
        if all or enable_search_zendesk:
            tools.append(self.search_zendesk)

        super().__init__(name="zendesk_tools", tools=tools, **kwargs)

    def search_zendesk(self, search_string: str) -> str:
        """
        Searches for articles in Zendesk Help Center that match the given search string.

        Parameters:
        search_string (str): The search query to look for in Zendesk articles.

        Returns:
        str: A JSON-formatted string containing the list of articles without HTML tags.

        Raises:
        ConnectionError: If the API request fails due to connection-related issues.
        """

        if not self.username or not self.password or not self.company_name:
            return "Username, password, or company name not provided."

        log_debug(f"Searching Zendesk for: {search_string}")

        auth = (self.username, self.password)
        url = f"https://{self.company_name}.zendesk.com/api/v2/help_center/articles/search.json?query={search_string}"
        try:
            response = requests.get(url, auth=auth)
            response.raise_for_status()
            clean = re.compile("<.*?>")
            articles = [re.sub(clean, "", article["body"]) for article in response.json()["results"]]
            return json.dumps(articles)
        except requests.RequestException as e:
            raise ConnectionError(f"API request failed: {e}")
```

---

### CalComTools (`agno.tools.calcom`)
Manage bookings and availability on Cal.com.

**Authentication**: Requires `CALCOM_API_KEY` and `CALCOM_EVENT_TYPE_ID`.
**Dependencies**: `pip install requests pytz`

#### Parameters
- `api_key` (str): Cal.com API key.
- `event_type_id` (int): Default event type ID for bookings.
- `user_timezone` (str): IANA timezone (e.g., 'America/New_York').

#### Source Code
```python
from datetime import datetime
from os import getenv
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import logger

try:
    import pytz
    import requests
except ImportError:
    raise ImportError("`requests` and `pytz` not installed. Please install using `pip install requests pytz`")


class CalComTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        event_type_id: Optional[int] = None,
        user_timezone: Optional[str] = None,
        # Enable flags for <6 functions
        enable_get_available_slots: bool = True,
        enable_create_booking: bool = True,
        enable_get_upcoming_bookings: bool = True,
        enable_reschedule_booking: bool = True,
        enable_cancel_booking: bool = True,
        all: bool = False,
        **kwargs,
    ):
        """Initialize the Cal.com toolkit.

        Args:
            api_key: Cal.com API key
            event_type_id: Default event type ID for bookings
            user_timezone: User's timezone in IANA format (e.g., 'Asia/Kolkata')
        """

        # Get credentials from environment if not provided
        self.api_key = api_key or getenv("CALCOM_API_KEY")
        event_type_str = getenv("CALCOM_EVENT_TYPE_ID")
        if event_type_id is not None:
            self.event_type_id = int(event_type_id)
        else:
            self.event_type_id = int(event_type_str) if event_type_str is not None else 0

        if not self.api_key:
            logger.error("CALCOM_API_KEY not set. Please set the CALCOM_API_KEY environment variable.")
        if not self.event_type_id:
            logger.error("CALCOM_EVENT_TYPE_ID not set. Please set the CALCOM_EVENT_TYPE_ID environment variable.")

        self.user_timezone = user_timezone or "America/New_York"

        tools: List[Any] = []
        if all or enable_get_available_slots:
            tools.append(self.get_available_slots)
        if all or enable_create_booking:
            tools.append(self.create_booking)
        if all or enable_get_upcoming_bookings:
            tools.append(self.get_upcoming_bookings)
        if all or enable_reschedule_booking:
            tools.append(self.reschedule_booking)
        if all or enable_cancel_booking:
            tools.append(self.cancel_booking)

        super().__init__(name="calcom", tools=tools, **kwargs)

    def _convert_to_user_timezone(self, utc_time: str) -> str:
        """Convert UTC time to user's timezone.

        Args:
            utc_time: UTC time string
            user_timezone: User's timezone (e.g., 'Asia/Kolkata')

        Returns:
            str: Formatted time in user's timezone
        """
        utc_dt = datetime.fromisoformat(utc_time.replace("Z", "+00:00"))
        user_tz = pytz.timezone(self.user_timezone)
        user_dt = utc_dt.astimezone(user_tz)
        return user_dt.strftime("%Y-%m-%d %H:%M %Z")

    def _get_headers(self, api_version: str = "2024-08-13") -> Dict[str, str]:
        """Get headers for Cal.com API requests.

        Args:
            api_version: Cal.com API version

        Returns:
            Dict[str, str]: Headers dictionary
        """
        return {
            "Authorization": f"Bearer {self.api_key}",
            "cal-api-version": api_version,
            "Content-Type": "application/json",
        }

    def get_available_slots(
        self,
        start_date: str,
        end_date: str,
    ) -> str:
        """Get available time slots for booking.

        Args:
            start_date: Start date in YYYY-MM-DD format
            end_date: End date in YYYY-MM-DD format
            user_timezone: User's timezone
            event_type_id: Optional specific event type ID

        Returns:
            str: Available slots or error message
        """
        try:
            url = "https://api.cal.com/v2/slots/available"
            querystring = {
                "startTime": f"{start_date}T00:00:00Z",
                "endTime": f"{end_date}T23:59:59Z",
                "eventTypeId": str(self.event_type_id),
            }

            response = requests.get(url, headers=self._get_headers(), params=querystring)  # type: ignore
            if response.status_code == 200:
                slots = response.json()["data"]["slots"]
                available_slots = []
                for date, times in slots.items():
                    for slot in times:
                        user_time = self._convert_to_user_timezone(slot["time"])
                        available_slots.append(user_time)
                return f"Available slots: {', '.join(available_slots)}"
            return f"Failed to fetch slots: {response.text}"
        except Exception as e:
            logger.error(f"Error fetching available slots: {e}")
            return f"Error: {str(e)}"

    def create_booking(
        self,
        start_time: str,
        name: str,
        email: str,
    ) -> str:
        """Create a new booking.

        Args:
            start_time: Start time in YYYY-MM-DDTHH:MM:SSZ format
            name: Attendee's name
            email: Attendee's email

        Returns:
            str: Booking confirmation or error message
        """
        try:
            url = "https://api.cal.com/v2/bookings"
            start_time = datetime.fromisoformat(start_time).astimezone(pytz.utc).isoformat(timespec="seconds")
            payload = {
                "start": start_time,
                "eventTypeId": self.event_type_id,
                "attendee": {"name": name, "email": email, "timeZone": self.user_timezone},
            }

            response = requests.post(url, json=payload, headers=self._get_headers())
            if response.status_code == 201:
                booking_data = response.json()["data"]
                user_time = self._convert_to_user_timezone(booking_data["start"])
                return f"Booking created successfully for {user_time}. Booking uid: {booking_data['uid']}"
            return f"Failed to create booking: {response.text}"
        except Exception as e:
            logger.error(f"Error creating booking: {e}")
            return f"Error: {str(e)}"

    def get_upcoming_bookings(self, email: Optional[str] = None) -> str:
        """Get all upcoming bookings for an attendee.

        Args:
            email (str): Attendee's email [Optional]

        Returns:
            str: List of upcoming bookings or error message
        """
        try:
            url = "https://api.cal.com/v2/bookings"
            querystring = {"status": "upcoming"}
            if email:
                querystring["attendeeEmail"] = email

            response = requests.get(url, headers=self._get_headers(), params=querystring)
            if response.status_code == 200:
                bookings = response.json()["data"]
                if not bookings:
                    return "No upcoming bookings found."

                booking_info = []
                for booking in bookings:
                    user_time = self._convert_to_user_timezone(booking["start"])
                    booking_info.append(
                        f"uid: {booking['uid']}, Title: {booking['title']}, Time: {user_time}, Status: {booking['status']}"
                    )
                return "Upcoming bookings:\n" + "\n".join(booking_info)
            return f"Failed to fetch bookings: {response.text}"
        except Exception as e:
            logger.error(f"Error fetching upcoming bookings: {e}")
            return f"Error: {str(e)}"

    def reschedule_booking(
        self,
        booking_uid: str,
        new_start_time: str,
        reason: str,
    ) -> str:
        """Reschedule an existing booking.

        Args:
            booking_uid: Booking UID to reschedule
            new_start_time: New start time in YYYY-MM-DDTHH:MM:SSZ format
            reason: Reason for rescheduling
            user_timezone: User's timezone

        Returns:
            str: Rescheduling confirmation or error message
        """
        try:
            url = f"https://api.cal.com/v2/bookings/{booking_uid}/reschedule"
            new_start_time = datetime.fromisoformat(new_start_time).astimezone(pytz.utc).isoformat(timespec="seconds")
            payload = {"start": new_start_time, "reschedulingReason": reason}

            response = requests.post(url, json=payload, headers=self._get_headers())
            if response.status_code == 201:
                booking_data = response.json()["data"]
                user_time = self._convert_to_user_timezone(booking_data["start"])
                return f"Booking rescheduled to {user_time}. New booking uid: {booking_data['uid']}"
            return f"Failed to reschedule booking: {response.text}"
        except Exception as e:
            logger.error(f"Error rescheduling booking: {e}")
            return f"Error: {str(e)}"

    def cancel_booking(self, booking_uid: str, reason: str) -> str:
        """Cancel an existing booking.

        Args:
            booking_uid: Booking UID to cancel
            reason: Reason for cancellation

        Returns:
            str: Cancellation confirmation or error message
        """
        try:
            url = f"https://api.cal.com/v2/bookings/{booking_uid}/cancel"
            payload = {"cancellationReason": reason}

            response = requests.post(url, json=payload, headers=self._get_headers())
            if response.status_code == 200:
                return "Booking cancelled successfully."
            return f"Failed to cancel booking: {response.text}"
        except Exception as e:
            logger.error(f"Error cancelling booking: {e}")
            return f"Error: {str(e)}"
```

---

### ZoomTools (`agno.tools.zoom`)
Schedule and manage Zoom meetings using Server-to-Server OAuth.

**Authentication**: Requires `ZOOM_ACCOUNT_ID`, `ZOOM_CLIENT_ID`, and `ZOOM_CLIENT_SECRET`.
**Dependencies**: `pip install requests`

#### Parameters
- `account_id` (str): Zoom account ID.
- `client_id` (str): OAuth client ID.
- `client_secret` (str): OAuth client secret.

#### Source Code
```python
import json
from base64 import b64encode
from datetime import datetime, timedelta
from os import getenv
from typing import Any, List, Optional

import requests

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info, logger


class ZoomTools(Toolkit):
    def __init__(
        self,
        account_id: Optional[str] = None,
        client_id: Optional[str] = None,
        client_secret: Optional[str] = None,
        **kwargs,
    ):
        """
        Initialize the ZoomTool.

        Args:
            account_id (str): The Zoom account ID for authentication. If not provided, will use ZOOM_ACCOUNT_ID env var.
            client_id (str): The client ID for authentication. If not provided, will use ZOOM_CLIENT_ID env var.
            client_secret (str): The client secret for authentication. If not provided, will use ZOOM_CLIENT_SECRET env var.
            name (str): The name of the tool. Defaults to "zoom_tool".
        """
        # Get credentials from env vars if not provided
        self.account_id = account_id or getenv("ZOOM_ACCOUNT_ID")
        self.client_id = client_id or getenv("ZOOM_CLIENT_ID")
        self.client_secret = client_secret or getenv("ZOOM_CLIENT_SECRET")
        self.__access_token = None  # Made private
        self.__token_expiry = None  # Track token expiration

        if not self.account_id or not self.client_id or not self.client_secret:
            logger.error(
                "ZOOM_ACCOUNT_ID, ZOOM_CLIENT_ID, and ZOOM_CLIENT_SECRET must be set either through parameters or environment variables."
            )

        tools: List[Any] = [
            self.get_access_token,
            self.schedule_meeting,
            self.get_upcoming_meetings,
            self.list_meetings,
            self.get_meeting_recordings,
            self.delete_meeting,
            self.get_meeting,
        ]

        super().__init__(name="zoom_tool", tools=tools, **kwargs)

    def get_access_token(self) -> str:
        """
        Get a valid access token, refreshing if necessary using Zoom's Server-to-Server OAuth.

        Returns:
            str: The current access token or empty string if token generation fails.
        """
        # Check if we have a valid token
        if self.__access_token and self.__token_expiry and datetime.now() < self.__token_expiry:
            return self.__access_token

        # Generate new token
        try:
            headers = {
                "Content-Type": "application/x-www-form-urlencoded",
            }

            # Create base64 encoded auth string
            auth_string = b64encode(f"{self.client_id}:{self.client_secret}".encode()).decode()
            headers["Authorization"] = f"Basic {auth_string}"

            data = {
                "grant_type": "account_credentials",
                "account_id": self.account_id,
            }

            response = requests.post("https://zoom.us/oauth/token", headers=headers, data=data)
            response.raise_for_status()

            token_data = response.json()
            self.__access_token = token_data["access_token"]
            # Set expiry time slightly before actual expiry to ensure token validity
            self.__token_expiry = datetime.now() + timedelta(seconds=token_data["expires_in"] - 60)  # type: ignore

            log_debug("Successfully generated new Zoom access token")
            return self.__access_token  # type: ignore

        except requests.RequestException as e:
            logger.error(f"Failed to generate Zoom access token: {e}")
            self.__access_token = None
            self.__token_expiry = None
            return ""

    def schedule_meeting(self, topic: str, start_time: str, duration: int, timezone: str = "UTC") -> str:
        """
        Schedule a new Zoom meeting.

        Args:
            topic (str): The topic or title of the meeting.
            start_time (str): The start time of the meeting in ISO 8601 format.
            duration (int): The duration of the meeting in minutes.
            timezone (str): The timezone for the meeting (e.g., "America/New_York", "Asia/Tokyo").

        Returns:
            A JSON-formatted string containing the response from Zoom API with the scheduled meeting details,
            or an error message if the scheduling fails.
        """
        log_debug(f"Attempting to schedule meeting: {topic} in timezone: {timezone}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = "https://api.zoom.us/v2/users/me/meetings"
        headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
        data = {
            "topic": topic,
            "type": 2,
            "start_time": start_time,
            "duration": duration,
            "timezone": timezone,
            "settings": {
                "host_video": True,
                "participant_video": True,
                "join_before_host": False,
                "mute_upon_entry": False,
                "watermark": True,
                "audio": "voip",
                "auto_recording": "none",
            },
        }

        try:
            response = requests.post(url, json=data, headers=headers)
            response.raise_for_status()
            meeting_info = response.json()

            result = {
                "message": "Meeting scheduled successfully!",
                "meeting_id": meeting_info["id"],
                "topic": meeting_info["topic"],
                "start_time": meeting_info["start_time"],
                "duration": meeting_info["duration"],
                "join_url": meeting_info["join_url"],
            }
            log_info(f"Meeting scheduled successfully. ID: {meeting_info['id']}")
            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error scheduling meeting: {e}")
            return json.dumps({"error": str(e)})

    def get_upcoming_meetings(self, user_id: str = "me") -> str:
        """
        Get a list of upcoming meetings for a specified user.

        Args:
            user_id (str): The user ID or 'me' for the authenticated user. Defaults to 'me'.

        Returns:
            A JSON-formatted string containing the upcoming meetings information,
            or an error message if the request fails.
        """
        log_debug(f"Fetching upcoming meetings for user: {user_id}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = f"https://api.zoom.us/v2/users/{user_id}/meetings"
        headers = {"Authorization": f"Bearer {token}"}
        params = {"type": "upcoming", "page_size": str(30)}

        try:
            response = requests.get(url, headers=headers, params=params)  # type: ignore
            response.raise_for_status()
            meetings = response.json()

            result = {"message": "Upcoming meetings retrieved successfully", "meetings": meetings.get("meetings", [])}
            log_info(f"Retrieved {len(result['meetings'])} upcoming meetings")
            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error fetching upcoming meetings: {e}")
            return json.dumps({"error": str(e)})

    def list_meetings(self, user_id: str = "me", type: str = "scheduled") -> str:
        """
        List all meetings for a specified user.

        Args:
            user_id (str): The user ID or 'me' for the authenticated user. Defaults to 'me'.
            type (str): The type of meetings to return. Options are:
                       "scheduled" - All valid scheduled meetings
                       "live" - All live meetings
                       "upcoming" - All upcoming meetings
                       "previous" - All previous meetings
                       Defaults to "scheduled".

        Returns:
            A JSON-formatted string containing the meetings information,
            or an error message if the request fails.
        """
        log_debug(f"Fetching meetings for user: {user_id}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = f"https://api.zoom.us/v2/users/{user_id}/meetings"
        headers = {"Authorization": f"Bearer {token}"}
        params = {"type": type}

        try:
            response = requests.get(url, headers=headers, params=params)
            response.raise_for_status()
            meetings = response.json()

            result = {
                "message": "Meetings retrieved successfully",
                "page_count": meetings.get("page_count", 0),
                "page_number": meetings.get("page_number", 1),
                "page_size": meetings.get("page_size", 30),
                "total_records": meetings.get("total_records", 0),
                "meetings": meetings.get("meetings", []),
            }
            log_info(f"Retrieved {len(result['meetings'])} meetings")
            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error fetching meetings: {e}")
            return json.dumps({"error": str(e)})

    def get_meeting_recordings(
        self, meeting_id: str, include_download_token: bool = False, token_ttl: Optional[int] = None
    ) -> str:
        """
        Get all recordings for a specific meeting.

        Args:
            meeting_id (str): The meeting ID or UUID to get recordings for.
            include_download_token (bool): Whether to include download access token in response.
            token_ttl (int, optional): Time to live for download token in seconds (max 604800).

        Returns:
            A JSON-formatted string containing the meeting recordings information,
            or an error message if the request fails.
        """
        log_debug(f"Fetching recordings for meeting: {meeting_id}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = f"https://api.zoom.us/v2/meetings/{meeting_id}/recordings"
        headers = {"Authorization": f"Bearer {token}"}

        # Build query parameters
        params = {}
        if include_download_token:
            params["include_fields"] = "download_access_token"
            if token_ttl is not None:
                if 0 <= token_ttl <= 604800:
                    params["ttl"] = str(token_ttl)  # Convert to string if necessary
                else:
                    logger.warning("Invalid TTL value. Must be between 0 and 604800 seconds.")

        try:
            response = requests.get(url, headers=headers, params=params)
            response.raise_for_status()
            recordings = response.json()

            result = {
                "message": "Meeting recordings retrieved successfully",
                "meeting_id": str(recordings.get("id", "")),
                "uuid": recordings.get("uuid", ""),
                "host_id": recordings.get("host_id", ""),
                "topic": recordings.get("topic", ""),
                "start_time": recordings.get("start_time", ""),
                "duration": recordings.get("duration", 0),
                "total_size": recordings.get("total_size", 0),
                "recording_count": recordings.get("recording_count", 0),
                "recording_files": recordings.get("recording_files", []),
            }

            log_info(f"Retrieved {result['recording_count']} recording files")
            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error fetching meeting recordings: {e}")
            return json.dumps({"error": str(e)})

    def delete_meeting(self, meeting_id: str, schedule_for_reminder: bool = True) -> str:
        """
        Delete a scheduled Zoom meeting.

        Args:
            meeting_id (str): The ID of the meeting to delete
            schedule_for_reminder (bool): Send cancellation email to registrants.
                                          Defaults to True.

        Returns:
            A JSON-formatted string containing the response status,
            or an error message if the deletion fails.
        """
        log_debug(f"Attempting to delete meeting: {meeting_id}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = f"https://api.zoom.us/v2/meetings/{meeting_id}"
        headers = {"Authorization": f"Bearer {token}"}
        params = {"schedule_for_reminder": schedule_for_reminder}

        try:
            response = requests.delete(url, headers=headers, params=params)
            response.raise_for_status()

            # Zoom returns 204 No Content for successful deletion
            if response.status_code == 204:
                result = {"message": "Meeting deleted successfully!", "meeting_id": meeting_id}
                log_info(f"Meeting {meeting_id} deleted successfully")
            else:
                result = response.json()

            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error deleting meeting: {e}")
            return json.dumps({"error": str(e)})

    def get_meeting(self, meeting_id: str) -> str:
        """
        Get the details of a specific Zoom meeting.

        Args:
            meeting_id (str): The ID of the meeting to retrieve

        Returns:
            A JSON-formatted string containing the meeting details,
            or an error message if the request fails.
        """
        log_debug(f"Fetching details for meeting: {meeting_id}")
        token = self.get_access_token()
        if not token:
            logger.error("Unable to obtain access token.")
            return json.dumps({"error": "Failed to obtain access token"})

        url = f"https://api.zoom.us/v2/meetings/{meeting_id}"
        headers = {"Authorization": f"Bearer {token}"}

        try:
            response = requests.get(url, headers=headers)
            response.raise_for_status()
            meeting_info = response.json()

            result = {
                "message": "Meeting details retrieved successfully",
                "meeting_id": str(meeting_info.get("id", "")),
                "topic": meeting_info.get("topic", ""),
                "type": meeting_info.get("type", ""),
                "start_time": meeting_info.get("start_time", ""),
                "duration": meeting_info.get("duration", 0),
                "timezone": meeting_info.get("timezone", ""),
                "created_at": meeting_info.get("created_at", ""),
                "join_url": meeting_info.get("join_url", ""),
                "settings": meeting_info.get("settings", {}),
            }

            log_info(f"Retrieved details for meeting ID: {meeting_id}")
            return json.dumps(result, indent=2)
        except requests.RequestException as e:
            logger.error(f"Error fetching meeting details: {e}")
            return json.dumps({"error": str(e)})

    def instructions(self) -> str:
        """
        Provide instructions for using the ZoomTool.

        Returns:
            A string containing instructions on how to use the ZoomTool.
        """
        return "Use this tool to schedule and manage Zoom meetings. You can schedule meetings by providing a topic, start time, and duration."
```


## 13. Web & Search (Detailed)

### DuckDuckGoTools (`agno.tools.duckduckgo`)
Privacy-focused web and news search using `ddgs`. No API key required.

**Authentication**: None.
**Dependencies**: `pip install ddgs`

#### Parameters
- `enable_search` (bool): Default True.
- `enable_news` (bool): Default True.
- `region` (str): e.g., 'us-en'.
- `timelimit` (str): 'd', 'w', 'm', 'y'.

#### Source Code
```python
from typing import Literal, Optional

from agno.tools.websearch import WebSearchTools


class DuckDuckGoTools(WebSearchTools):
    """
    DuckDuckGoTools is a convenience wrapper around WebSearchTools with the backend
    defaulting to "duckduckgo".

    Args:
        enable_search (bool): Enable web search function.
        enable_news (bool): Enable news search function.
        modifier (Optional[str]): A modifier to be prepended to search queries.
        fixed_max_results (Optional[int]): A fixed number of maximum results.
        proxy (Optional[str]): Proxy to be used for requests.
        timeout (Optional[int]): The maximum number of seconds to wait for a response.
        verify_ssl (bool): Whether to verify SSL certificates.
        timelimit (Optional[str]): Time limit for search results. Valid values:
            "d" (day), "w" (week), "m" (month), "y" (year).
        region (Optional[str]): Region for search results (e.g., "us-en", "uk-en", "ru-ru").
        backend (Optional[str]): Backend to use for searching (e.g., "api", "html", "lite").
            Defaults to "duckduckgo".
    """

    def __init__(
        self,
        enable_search: bool = True,
        enable_news: bool = True,
        modifier: Optional[str] = None,
        fixed_max_results: Optional[int] = None,
        proxy: Optional[str] = None,
        timeout: Optional[int] = 10,
        verify_ssl: bool = True,
        timelimit: Optional[Literal["d", "w", "m", "y"]] = None,
        region: Optional[str] = None,
        backend: Optional[str] = None,
        **kwargs,
    ):
        super().__init__(
            enable_search=enable_search,
            enable_news=enable_news,
            backend=backend or "duckduckgo",
            modifier=modifier,
            fixed_max_results=fixed_max_results,
            proxy=proxy,
            timeout=timeout,
            verify_ssl=verify_ssl,
            timelimit=timelimit,
            region=region,
            **kwargs,
        )

        # Backward compatibility aliases for old method names
        self.duckduckgo_search = self.web_search
        self.duckduckgo_news = self.search_news
```

---

### TavilyTools (`agno.tools.tavily`)
AI-optimized search and content extraction.

**Authentication**: Requires `TAVILY_API_KEY`.
**Dependencies**: `pip install tavily-python`

#### Parameters
- `search_depth` (str): 'basic' or 'advanced'.
- `max_tokens` (int): Max results size.
- `include_answer` (bool): Include AI generated summary.

#### Source Code
```python
import json
from os import getenv
from typing import Any, Dict, List, Literal, Optional

from agno.tools import Toolkit
from agno.utils.log import logger

try:
    from tavily import TavilyClient
except ImportError:
    raise ImportError("`tavily-python` not installed. Please install using `pip install tavily-python`")


class TavilyTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        api_base_url: Optional[str] = None,
        enable_search: bool = True,
        enable_search_context: bool = False,
        enable_extract: bool = False,
        all: bool = False,
        max_tokens: int = 6000,
        include_answer: bool = True,
        search_depth: Literal["basic", "advanced"] = "advanced",
        extract_depth: Literal["basic", "advanced"] = "basic",
        include_images: bool = False,
        include_favicon: bool = False,
        extract_timeout: Optional[int] = None,
        extract_format: Literal["markdown", "text"] = "markdown",
        format: Literal["json", "markdown"] = "markdown",
        **kwargs,
    ):
        """Initialize TavilyTools with search and extract capabilities.

        Args:
            api_key: Tavily API key. If not provided, will use TAVILY_API_KEY env var.
            api_base_url: Tavily API base URL. If not provided, will use TAVILY_API_BASE_URL env var. Defaults to None. If None - will use https://api.tavily.com.
            enable_search: Enable web search functionality. Defaults to True.
            enable_search_context: Use search context mode instead of regular search. Defaults to False.
            enable_extract: Enable URL content extraction functionality. Defaults to False.
            all: Enable all available tools. Defaults to False.
            max_tokens: Maximum tokens for search results. Defaults to 6000.
            include_answer: Include AI-generated answer in search results. Defaults to True.
            search_depth: Search depth level - basic (1 credit) or advanced (2 credits). Defaults to "advanced".
            extract_depth: Extract depth level - basic (1 credit/5 URLs) or advanced (2 credits/5 URLs). Defaults to "basic".
            include_images: Include images in extracted content. Defaults to False.
            include_favicon: Include favicon in extracted content. Defaults to False.
            extract_timeout: Timeout in seconds for extraction requests. Defaults to None.
            extract_format: Output format for extracted content - markdown or text. Defaults to "markdown".
            format: Output format for search results - json or markdown. Defaults to "markdown".
            **kwargs: Additional arguments passed to Toolkit.
        """
        self.api_key = api_key or getenv("TAVILY_API_KEY")
        if not self.api_key:
            logger.error("TAVILY_API_KEY not provided")
        self.api_base_url = api_base_url or getenv("TAVILY_API_BASE_URL")

        self.client: TavilyClient = TavilyClient(api_key=self.api_key, api_base_url=self.api_base_url)
        self.search_depth: Literal["basic", "advanced"] = search_depth
        self.extract_depth: Literal["basic", "advanced"] = extract_depth
        self.max_tokens: int = max_tokens
        self.include_answer: bool = include_answer
        self.include_images: bool = include_images
        self.include_favicon: bool = include_favicon
        self.extract_timeout: Optional[int] = extract_timeout
        self.extract_format: Literal["markdown", "text"] = extract_format
        self.format: Literal["json", "markdown"] = format

        tools: List[Any] = []

        if enable_search or all:
            if enable_search_context:
                tools.append(self.web_search_with_tavily)
            else:
                tools.append(self.web_search_using_tavily)

        if enable_extract or all:
            tools.append(self.extract_url_content)

        super().__init__(name="tavily_tools", tools=tools, **kwargs)

    def web_search_using_tavily(self, query: str, max_results: int = 5) -> str:
        """Use this function to search the web for a given query.
        This function uses the Tavily API to provide realtime online information about the query.

        Args:
            query (str): Query to search for.
            max_results (int): Maximum number of results to return. Defaults to 5.

        Returns:
            str: JSON string of results related to the query.
        """

        response = self.client.search(
            query=query, search_depth=self.search_depth, include_answer=self.include_answer, max_results=max_results
        )

        clean_response: Dict[str, Any] = {"query": query}
        if "answer" in response:
            clean_response["answer"] = response["answer"]

        clean_results = []
        current_token_count = len(json.dumps(clean_response))
        for result in response.get("results", []):
            _result = {
                "title": result["title"],
                "url": result["url"],
                "content": result["content"],
                "score": result["score"],
            }
            current_token_count += len(json.dumps(_result))
            if current_token_count > self.max_tokens:
                break
            clean_results.append(_result)
        clean_response["results"] = clean_results

        if self.format == "json":
            return json.dumps(clean_response) if clean_response else "No results found."
        elif self.format == "markdown":
            _markdown = ""
            _markdown += f"# {query}\n\n"
            if "answer" in clean_response:
                _markdown += "### Summary\n"
                _markdown += f"{clean_response.get('answer')}\n\n"
            for result in clean_response["results"]:
                _markdown += f"### [{result['title']}]({result['url']})\n"
                _markdown += f"{result['content']}\n\n"
            return _markdown

    def web_search_with_tavily(self, query: str) -> str:
        """Use this function to search the web for a given query.
        This function uses the Tavily API to provide realtime online information about the query.

        Args:
            query (str): Query to search for.

        Returns:
            str: JSON string of results related to the query.
        """

        return self.client.get_search_context(query=query, search_depth=self.search_depth, max_tokens=self.max_tokens)

    def extract_url_content(self, urls: str) -> str:
        """Extract content from one or more URLs using Tavily's Extract API.
        This function retrieves the main content from web pages in markdown or text format.

        Args:
            urls (str): Single URL or multiple comma-separated URLs to extract content from.
                       Example: "https://example.com" or "https://example.com,https://another.com"

        Returns:
            str: Extracted content in the specified format (markdown or text).
                 For multiple URLs, returns combined content with URL headers.
                 Failed extractions are noted in the output.
        """
        # Parse URLs - handle both single and comma-separated multiple URLs
        url_list = [url.strip() for url in urls.split(",") if url.strip()]

        if not url_list:
            return "Error: No valid URLs provided."

        try:
            # Prepare extract parameters
            extract_params: Dict[str, Any] = {
                "urls": url_list,
                "depth": self.extract_depth,
            }

            # Add optional parameters if specified
            if self.include_images:
                extract_params["include_images"] = True
            if self.include_favicon:
                extract_params["include_favicon"] = True
            if self.extract_timeout is not None:
                extract_params["timeout"] = self.extract_timeout

            # Call Tavily Extract API
            response = self.client.extract(**extract_params)

            # Process response based on format preference
            if not response or "results" not in response:
                return "Error: No content could be extracted from the provided URL(s)."

            results = response.get("results", [])
            if not results:
                return "Error: No content could be extracted from the provided URL(s)."

            # Format output
            if self.extract_format == "markdown":
                return self._format_extract_markdown(results)
            elif self.extract_format == "text":
                return self._format_extract_text(results)
            else:
                # Fallback to JSON if format is unrecognized
                return json.dumps(results, indent=2)

        except Exception as e:
            logger.error(f"Error extracting content from URLs: {e}")
            return f"Error extracting content: {str(e)}"

    def _format_extract_markdown(self, results: List[Dict[str, Any]]) -> str:
        """Format extraction results as markdown.

        Args:
            results: List of extraction result dictionaries from Tavily API.

        Returns:
            str: Formatted markdown content.
        """
        output = []

        for result in results:
            url = result.get("url", "Unknown URL")
            raw_content = result.get("raw_content", "")
            failed_reason = result.get("failed_reason")

            if failed_reason:
                output.append(f"## {url}\n\n **Extraction Failed**: {failed_reason}\n\n")
            elif raw_content:
                output.append(f"## {url}\n\n{raw_content}\n\n")
            else:
                output.append(f"## {url}\n\n*No content available*\n\n")

        return "".join(output) if output else "No content extracted."

    def _format_extract_text(self, results: List[Dict[str, Any]]) -> str:
        """Format extraction results as plain text.

        Args:
            results: List of extraction result dictionaries from Tavily API.

        Returns:
            str: Formatted plain text content.
        """
        output = []

        for result in results:
            url = result.get("url", "Unknown URL")
            raw_content = result.get("raw_content", "")
            failed_reason = result.get("failed_reason")

            output.append(f"URL: {url}")
            output.append("-" * 80)

            if failed_reason:
                output.append(f"EXTRACTION FAILED: {failed_reason}")
            elif raw_content:
                output.append(raw_content)
            else:
                output.append("No content available")

            output.append("\n")

        return "\n".join(output) if output else "No content extracted."
```

---

### SerperTools (`agno.tools.serper`)
Google Search, News, Scholar, and Scrape API via Serper.dev.

**Authentication**: Requires `SERPER_API_KEY`.
**Dependencies**: `pip install requests`

#### Parameters
- `location` (str): Google location code.
- `language` (str): Language code.
- `num_results` (int): Number of results.

#### Source Code
```python
import json
from os import getenv
from typing import Any, Dict, List, Optional

import requests

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_error, log_warning


class SerperTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        location: str = "us",
        language: str = "en",
        num_results: int = 10,
        date_range: Optional[str] = None,
        enable_search: bool = True,
        enable_search_news: bool = True,
        enable_search_scholar: bool = True,
        enable_scrape_webpage: bool = True,
        all: bool = False,
        **kwargs,
    ):
        """
        Initialize the SerperTools.

        Args:
            api_key Optional[str]: The Serper API key.
            location Optional[str]: The Google location code for search results.
            language Optional[str]: The language code for search results.
            num_results Optional[int]: The number of search results to retrieve.
            date_range Optional[str]: Default date range filter for searches.
        """
        self.api_key = api_key or getenv("SERPER_API_KEY")
        if not self.api_key:
            log_debug("No Serper API key provided")

        self.location = location
        self.language = language
        self.num_results = num_results
        self.date_range = date_range

        tools: List[Any] = []
        if all or enable_search:
            tools.append(self.search_web)
        if all or enable_search_news:
            tools.append(self.search_news)
        if all or enable_search_scholar:
            tools.append(self.search_scholar)
        if all or enable_scrape_webpage:
            tools.append(self.scrape_webpage)

        super().__init__(name="serper_tools", tools=tools, **kwargs)

    def _make_request(self, endpoint: str, params: Dict[str, Any]) -> Dict[str, Any]:
        """
        Makes a request to the Serper API.

        Args:
            endpoint (str): The API endpoint
            params (Dict[str, Any]): Request parameters

        Returns:
            Dict[str, Any]: Search response
        """
        try:
            if not self.api_key:
                log_error("No Serper API key provided")
                return {"success": False, "error": "Please provide a Serper API key"}

            url = f"https://google.serper.dev/{endpoint}"
            if endpoint == "scrape":
                url = "https://scrape.serper.dev"

            headers = {"X-API-KEY": self.api_key, "Content-Type": "application/json"}

            # Add optional parameters
            if self.date_range:
                params["tbs"] = self.date_range
            if self.location:
                params["gl"] = self.location

            if self.language:
                params["hl"] = self.language

            payload = json.dumps(params)

            log_debug(f"Making request to {url} with params: {params}")
            response = requests.request("POST", url, headers=headers, data=payload)
            response.raise_for_status()

            log_debug(f"Successfully received response from {endpoint} endpoint")
            return {"success": True, "data": response.json(), "raw_response": response.text}
        except Exception as e:
            log_error(f"Serper API error: {str(e)}")
            return {"success": False, "error": str(e)}

    def search_web(
        self,
        query: str,
        num_results: Optional[int] = None,
    ) -> str:
        """
        Searches Google for the provided query using the Serper API.

        Args:
            query (str): The search query to search for on Google.
            num_results (int, optional): Number of search results to retrieve.

        Returns:
            str: A JSON-formatted string containing the search results or an error message if the search fails.
        """
        try:
            if not query:
                return json.dumps({"error": "Please provide a query to search for"}, indent=2)

            log_debug(f"Searching Google for: {query}")

            params = {
                "q": query,
                "num": num_results or self.num_results,
            }

            result = self._make_request("search", params)

            if result["success"]:
                log_debug(f"Successfully found Google search results for query: {query}")
                return result["raw_response"]
            else:
                log_error(f"Error searching Google for query {query}: {result['error']}")
                return json.dumps({"error": result["error"]}, indent=2)

        except Exception as e:
            log_error(f"Unexpected error searching Google for query {query}: {e}")
            return json.dumps({"error": f"An unexpected error occurred: {str(e)}"}, indent=2)

    def search_news(
        self,
        query: str,
        num_results: Optional[int] = None,
    ) -> str:
        """
        Searches for news articles using the Serper News API.

        Args:
            query (str): The search query for news articles.
            num_results (int, optional): Number of news results to retrieve.

        Returns:
            str: A JSON-formatted string containing the news search results or an error message.
        """
        try:
            if not query:
                return json.dumps({"error": "Please provide a query to search for news"}, indent=2)

            log_debug(f"Searching news for: {query}")

            params = {
                "q": query,
                "num": num_results or self.num_results,
            }

            result = self._make_request("news", params)

            if result["success"]:
                log_debug(f"Successfully found {num_results or self.num_results} news articles for query: {query}")
                return result["raw_response"]
            else:
                log_error(f"Error searching news for query {query}: {result['error']}")
                return json.dumps({"error": result["error"]}, indent=2)

        except Exception as e:
            log_error(f"Unexpected error searching news for query {query}: {e}")
            return json.dumps({"error": f"An unexpected error occurred: {str(e)}"}, indent=2)

    def search_scholar(
        self,
        query: str,
        num_results: Optional[int] = None,
    ) -> str:
        """
        Searches for academic papers using Google Scholar via Serper API.

        Args:
            query (str): The search query for academic papers.
            num_results (int, optional): Number of academic papers to retrieve.

        Returns:
            str: A JSON-formatted string containing the scholar search results or an error message.
        """
        try:
            if not query:
                return json.dumps({"error": "Please provide a query to search for academic papers"}, indent=2)

            log_debug(f"Searching scholar for: {query}")

            params = {
                "q": query,
                "num": num_results or self.num_results,
            }

            result = self._make_request("scholar", params)

            if result["success"]:
                log_debug(f"Successfully found academic papers for query: {query}")
                return result["raw_response"]
            else:
                log_error(f"Error searching scholar for query {query}: {result['error']}")
                return json.dumps({"error": result["error"]}, indent=2)

        except Exception as e:
            log_error(f"Unexpected error searching scholar for query {query}: {e}")
            return json.dumps({"error": f"An unexpected error occurred: {str(e)}"}, indent=2)

    def scrape_webpage(
        self,
        url: str,
        markdown: bool = False,
    ) -> str:
        """
        Scrapes and extracts content from a webpage using the Serper scraping API.

        Args:
            url (str): The URL of the webpage to scrape.
            markdown (bool, optional): Return content in markdown format (default: False).

        Returns:
            str: A JSON-formatted string containing the scraped webpage content or an error message.
        """
        try:
            if not url:
                log_warning("No URL provided to scrape")
                return json.dumps({"error": "Please provide a URL to scrape"}, indent=2)

            log_debug(f"Scraping webpage: {url}")

            params = {
                "url": url,
                "includeMarkdown": markdown,
            }

            result = self._make_request("scrape", params)

            if result["success"]:
                log_debug(f"Successfully scraped webpage: {url}")
                return result["raw_response"]
            else:
                log_error(f"Error scraping webpage {url}: {result['error']}")
                return json.dumps({"error": result["error"]}, indent=2)

        except Exception as e:
            log_error(f"Unexpected error scraping webpage {url}: {e}")
            return json.dumps({"error": f"An unexpected error occurred: {str(e)}"}, indent=2)
```

---

### SerpApiTools (`agno.tools.serpapi`)
Search Google and YouTube using SerpApi.

**Authentication**: Requires `SERP_API_KEY`.
**Dependencies**: `pip install google-search-results`

#### Parameters
- `api_key` (str): SerpApi key.
- `enable_search_youtube` (bool): Default False.

#### Source Code
```python
import json
from os import getenv
from typing import Any, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_info, logger

try:
    import serpapi
except ImportError:
    raise ImportError("`google-search-results` not installed.")


class SerpApiTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        enable_search_google: bool = True,
        enable_search_youtube: bool = False,
        all: bool = False,
        **kwargs,
    ):
        self.api_key = api_key or getenv("SERP_API_KEY")
        if not self.api_key:
            logger.warning("No Serpapi API key provided")

        tools: List[Any] = []
        if all or enable_search_google:
            tools.append(self.search_google)
        if all or enable_search_youtube:
            tools.append(self.search_youtube)

        super().__init__(name="serpapi_tools", tools=tools, **kwargs)

    def search_google(self, query: str, num_results: int = 10) -> str:
        """
        Search Google using the Serpapi API. Returns the search results.

        Args:
            query(str): The query to search for.
            num_results(int): The number of results to return.

        Returns:
            str: The search results from Google.
                Keys:
                    - 'search_results': List of organic search results.
                    - 'recipes_results': List of recipes search results.
                    - 'shopping_results': List of shopping search results.
                    - 'knowledge_graph': The knowledge graph.
                    - 'related_questions': List of related questions.
        """

        try:
            if not self.api_key:
                return "Please provide an API key"
            if not query:
                return "Please provide a query to search for"

            log_info(f"Searching Google for: {query}")

            params = {"q": query, "api_key": self.api_key, "num": num_results}

            search = serpapi.GoogleSearch(params)
            results = search.get_dict()

            filtered_results = {
                "search_results": results.get("organic_results", ""),
                "recipes_results": results.get("recipes_results", ""),
                "shopping_results": results.get("shopping_results", ""),
                "knowledge_graph": results.get("knowledge_graph", ""),
                "related_questions": results.get("related_questions", ""),
            }

            return json.dumps(filtered_results)

        except Exception as e:
            return f"Error searching for the query {query}: {e}"

    def search_youtube(self, query: str) -> str:
        """
        Search Youtube using the Serpapi API. Returns the search results.

        Args:
            query(str): The query to search for.

        Returns:
            str: The video search results from Youtube.
                Keys:
                    - 'video_results': List of video results.
                    - 'movie_results': List of movie results.
                    - 'channel_results': List of channel results.
        """

        try:
            if not self.api_key:
                return "Please provide an API key"
            if not query:
                return "Please provide a query to search for"

            log_info(f"Searching Youtube for: {query}")

            params = {"search_query": query, "api_key": self.api_key}

            search = serpapi.YoutubeSearch(params)
            results = search.get_dict()

            filtered_results = {
                "video_results": results.get("video_results", ""),
                "movie_results": results.get("movie_results", ""),
                "channel_results": results.get("channel_results", ""),
            }

            return json.dumps(filtered_results)

        except Exception as e:
            return f"Error searching for the query {query}: {e}"
```

---

### BraveSearchTools (`agno.tools.bravesearch`)
Search the Brave index.

**Authentication**: Requires `BRAVE_API_KEY`.
**Dependencies**: `pip install brave-search`

#### Parameters
- `api_key` (str): Brave API key.
- `country` (str): Default 'US'.

#### Source Code
```python
import json
from os import getenv
from typing import Optional

from agno.tools import Toolkit
from agno.utils.log import log_info

try:
    from brave import Brave
except ImportError:
    raise ImportError("`brave-search` not installed. Please install using `pip install brave-search`")


class BraveSearchTools(Toolkit):
    """
    BraveSearch is a toolkit for searching Brave easily.

    Args:
        api_key (str, optional): Brave API key. If not provided, will use BRAVE_API_KEY environment variable.
        fixed_max_results (Optional[int]): A fixed number of maximum results.
        fixed_language (Optional[str]): A fixed language for the search results.
    """

    def __init__(
        self,
        api_key: Optional[str] = None,
        fixed_max_results: Optional[int] = None,
        fixed_language: Optional[str] = None,
        enable_brave_search: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.api_key = api_key or getenv("BRAVE_API_KEY")
        if not self.api_key:
            raise ValueError("BRAVE_API_KEY is required. Please set the BRAVE_API_KEY environment variable.")

        self.fixed_max_results = fixed_max_results
        self.fixed_language = fixed_language

        self.brave_client = Brave(api_key=self.api_key)

        tools = []
        if all or enable_brave_search:
            tools.append(self.brave_search)

        super().__init__(
            name="brave_search",
            tools=tools,
            **kwargs,
        )

    def brave_search(
        self,
        query: str,
        max_results: int = 5,
        country: str = "US",
        search_lang: str = "en",
    ) -> str:
        """
        Search Brave for the specified query and return the results.

        Args:
            query (str): The query to search for.
            max_results (int, optional): The maximum number of results to return. Default is 5.
            country (str, optional): The country code for search results. Default is "US".
            search_lang (str, optional): The language of the search results. Default is "en".
        Returns:
            str: A JSON formatted string containing the search results.
        """
        final_max_results = self.fixed_max_results if self.fixed_max_results is not None else max_results
        final_search_lang = self.fixed_language if self.fixed_language is not None else search_lang

        if not query:
            return json.dumps({"error": "Please provide a query to search for"})

        log_info(f"Searching Brave for: {query}")

        search_params = {
            "q": query,
            "count": final_max_results,
            "country": country,
            "search_lang": final_search_lang,
            "result_filter": "web",
        }

        search_results = self.brave_client.search(**search_params)

        filtered_results = {
            "web_results": [],
            "query": query,
            "total_results": 0,
        }

        if hasattr(search_results, "web") and search_results.web:
            web_results = []
            for result in search_results.web.results:
                web_result = {
                    "title": result.title,
                    "url": str(result.url),
                    "description": result.description,
                }
                web_results.append(web_result)
            filtered_results["web_results"] = web_results
            filtered_results["total_results"] = len(web_results)

        return json.dumps(filtered_results, indent=2)
```

---

### BaiduSearchTools (`agno.tools.baidusearch`)
Search the Baidu index (Chinese).

**Authentication**: Requires `baidusearch` library.
**Dependencies**: `pip install baidusearch pycountry`

#### Parameters
- `language` (str): Default 'zh'.
- `max_results` (int): Default 5.

#### Source Code
```python
import json
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug

try:
    from baidusearch.baidusearch import search  # type: ignore
except ImportError:
    raise ImportError("`baidusearch` not installed. Please install using `pip install baidusearch`")

try:
    from pycountry import pycountry
except ImportError:
    raise ImportError("`pycountry` not installed. Please install using `pip install pycountry`")


class BaiduSearchTools(Toolkit):
    """
    BaiduSearch is a toolkit for searching Baidu easily.

    Args:
        fixed_max_results (Optional[int]): A fixed number of maximum results.
        fixed_language (Optional[str]): A fixed language for the search results.
        headers (Optional[Any]): Headers to be used in the search request.
        proxy (Optional[str]): Proxy to be used in the search request.
        debug (Optional[bool]): Enable debug output.
    """

    def __init__(
        self,
        fixed_max_results: Optional[int] = None,
        fixed_language: Optional[str] = None,
        headers: Optional[Any] = None,
        proxy: Optional[str] = None,
        timeout: Optional[int] = 10,
        debug: Optional[bool] = False,
        enable_baidu_search: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.fixed_max_results = fixed_max_results
        self.fixed_language = fixed_language
        self.headers = headers
        self.proxy = proxy
        self.timeout = timeout
        self.debug = debug

        tools = []
        if all or enable_baidu_search:
            tools.append(self.baidu_search)

        super().__init__(name="baidusearch", tools=tools, **kwargs)

    def baidu_search(self, query: str, max_results: int = 5, language: str = "zh") -> str:
        """Execute Baidu search and return results

        Args:
            query (str): Search keyword
            max_results (int, optional): Maximum number of results to return, default 5
            language (str, optional): Search language, default Chinese

        Returns:
            str: A JSON formatted string containing the search results.
        """
        max_results = self.fixed_max_results or max_results
        language = self.fixed_language or language

        if len(language) != 2:
            try:
                language = pycountry.languages.lookup(language).alpha_2
            except LookupError:
                language = "zh"

        log_debug(f"Searching Baidu [{language}] for: {query}")

        results = search(keyword=query, num_results=max_results)

        res: List[Dict[str, str]] = []
        for idx, item in enumerate(results, 1):
            res.append(
                {
                    "title": item.get("title", ""),
                    "url": item.get("url", ""),
                    "abstract": item.get("abstract", ""),
                    "rank": str(idx),
                }
            )
        return json.dumps(res, indent=2)
```

---

### WebSearchTools (`agno.tools.websearch`)
Generic meta-search toolkit using `ddgs`. Supports multiple backends.

**Authentication**: None.
**Dependencies**: `pip install ddgs`

#### Parameters
- `backend` (str): 'auto', 'duckduckgo', 'google', 'bing', etc.
- `modifier` (str): Text to prepend to queries.

#### Source Code
```python
import json
from typing import Any, List, Literal, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug

try:
    from ddgs import DDGS
except ImportError:
    raise ImportError("`ddgs` not installed. Please install using `pip install ddgs`")

# Valid timelimit values for search filtering
VALID_TIMELIMITS = frozenset({"d", "w", "m", "y"})


class WebSearchTools(Toolkit):
    """
    Toolkit for searching the web. Uses the meta-search library DDGS.
    Multiple search backends (e.g. google, bing, duckduckgo) are available.

    Args:
        enable_search (bool): Enable web search function.
        enable_news (bool): Enable news search function.
        backend (str): The backend to use for searching. Defaults to "auto" which
            automatically selects available backends. Other options include:
            "duckduckgo", "google", "bing", "brave", "yandex", "yahoo", etc.
        modifier (Optional[str]): A modifier to be prepended to search queries.
        fixed_max_results (Optional[int]): A fixed number of maximum results.
        proxy (Optional[str]): Proxy to be used for requests.
        timeout (Optional[int]): The maximum number of seconds to wait for a response.
        verify_ssl (bool): Whether to verify SSL certificates.
        timelimit (Optional[str]): Time limit for search results. Valid values:
            "d" (day), "w" (week), "m" (month), "y" (year).
        region (Optional[str]): Region for search results (e.g., "us-en", "uk-en", "ru-ru").
    """

    def __init__(
        self,
        enable_search: bool = True,
        enable_news: bool = True,
        backend: str = "auto",
        modifier: Optional[str] = None,
        fixed_max_results: Optional[int] = None,
        proxy: Optional[str] = None,
        timeout: Optional[int] = 10,
        verify_ssl: bool = True,
        timelimit: Optional[Literal["d", "w", "m", "y"]] = None,
        region: Optional[str] = None,
        **kwargs,
    ):
        # Validate timelimit parameter
        if timelimit is not None and timelimit not in VALID_TIMELIMITS:
            raise ValueError(
                f"Invalid timelimit '{timelimit}'. Must be one of: 'd' (day), 'w' (week), 'm' (month), 'y' (year)."
            )

        self.proxy: Optional[str] = proxy
        self.timeout: Optional[int] = timeout
        self.fixed_max_results: Optional[int] = fixed_max_results
        self.modifier: Optional[str] = modifier
        self.verify_ssl: bool = verify_ssl
        self.backend: str = backend
        self.timelimit: Optional[Literal["d", "w", "m", "y"]] = timelimit
        self.region: Optional[str] = region

        tools: List[Any] = []
        if enable_search:
            tools.append(self.web_search)
        if enable_news:
            tools.append(self.search_news)

        super().__init__(name="websearch", tools=tools, **kwargs)

    def web_search(self, query: str, max_results: int = 5) -> str:
        """Use this function to search the web for a query.

        Args:
            query(str): The query to search for.
            max_results (optional, default=5): The maximum number of results to return.

        Returns:
            The search results from the web.
        """
        actual_max_results = self.fixed_max_results or max_results
        search_query = f"{self.modifier} {query}" if self.modifier else query

        log_debug(f"Searching web for: {search_query} using backend: {self.backend}")
        search_kwargs: dict = {
            "query": search_query,
            "max_results": actual_max_results,
            "backend": self.backend,
        }
        if self.timelimit is not None:
            search_kwargs["timelimit"] = self.timelimit
        if self.region is not None:
            search_kwargs["region"] = self.region
        with DDGS(proxy=self.proxy, timeout=self.timeout, verify=self.verify_ssl) as ddgs:
            results = ddgs.text(**search_kwargs)

        return json.dumps(results, indent=2)

    def search_news(self, query: str, max_results: int = 5) -> str:
        """Use this function to get the latest news from the web.

        Args:
            query(str): The query to search for.
            max_results (optional, default=5): The maximum number of results to return.

        Returns:
            The latest news from the web.
        """
        actual_max_results = self.fixed_max_results or max_results

        log_debug(f"Searching web news for: {query} using backend: {self.backend}")
        search_kwargs: dict = {
            "query": query,
            "max_results": actual_max_results,
            "backend": self.backend,
        }
        if self.timelimit is not None:
            search_kwargs["timelimit"] = self.timelimit
        if self.region is not None:
            search_kwargs["region"] = self.region
        with DDGS(proxy=self.proxy, timeout=self.timeout, verify=self.verify_ssl) as ddgs:
            results = ddgs.news(**search_kwargs)

        return json.dumps(results, indent=2)
```


## 14. Data Sources & Knowledge (Detailed)

### HackerNewsTools (`agno.tools.hackernews`)
Get top stories and user details from Hacker News.

**Authentication**: None.
**Dependencies**: `pip install httpx`

#### Parameters
- `enable_get_top_stories` (bool): Default True.
- `enable_get_user_details` (bool): Default True.

#### Source Code
```python
import json
from typing import Any, List

import httpx

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger


class HackerNewsTools(Toolkit):
    """
    HackerNews is a tool for getting top stories from Hacker News.

    Args:
        enable_get_top_stories (bool): Enable getting top stories from Hacker News. Default is True.
        enable_get_user_details (bool): Enable getting user details from Hacker News. Default is True.
        all (bool): Enable all tools. Overrides individual flags when True. Default is False.
    """

    def __init__(
        self, enable_get_top_stories: bool = True, enable_get_user_details: bool = True, all: bool = False, **kwargs
    ):
        tools: List[Any] = []
        if all or enable_get_top_stories:
            tools.append(self.get_top_hackernews_stories)
        if all or enable_get_user_details:
            tools.append(self.get_user_details)

        super().__init__(name="hackers_news", tools=tools, **kwargs)

    def get_top_hackernews_stories(self, num_stories: int = 10) -> str:
        """Use this function to get top stories from Hacker News.

        Args:
            num_stories (int): Number of stories to return. Defaults to 10.

        Returns:
            str: JSON string of top stories.
        """

        log_debug(f"Getting top {num_stories} stories from Hacker News")
        # Fetch top story IDs
        response = httpx.get("https://hacker-news.firebaseio.com/v0/topstories.json")
        story_ids = response.json()

        # Fetch story details
        stories = []
        for story_id in story_ids[:num_stories]:
            story_response = httpx.get(f"https://hacker-news.firebaseio.com/v0/item/{story_id}.json")
            story = story_response.json()
            story["username"] = story["by"]
            stories.append(story)
        return json.dumps(stories)

    def get_user_details(self, username: str) -> str:
        """Use this function to get the details of a Hacker News user using their username.

        Args:
            username (str): Username of the user to get details for.

        Returns:
            str: JSON string of the user details.
        """

        try:
            log_debug(f"Getting details for user: {username}")
            user = httpx.get(f"https://hacker-news.firebaseio.com/v0/user/{username}.json").json()
            user_details = {
                "id": user.get("user_id"),
                "karma": user.get("karma"),
                "about": user.get("about"),
                "total_items_submitted": len(user.get("submitted", [])),
            }
            return json.dumps(user_details)
        except Exception as e:
            logger.exception(e)
            return f"Error getting user details: {e}"
```

---

### RedditTools (`agno.tools.reddit`)
Interact with Reddit for posts, subreddits, and user info.

**Authentication**: Requires `REDDIT_CLIENT_ID`, `REDDIT_CLIENT_SECRET`. Optional for write: `REDDIT_USERNAME`, `REDDIT_PASSWORD`.
**Dependencies**: `pip install praw`

#### Parameters
- `client_id` (str): Reddit client ID.
- `client_secret` (str): Reddit client secret.
- `user_agent` (str): Default 'RedditTools v1.0'.

#### Source Code
```python
import json
from os import getenv
from typing import Callable, Dict, List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info, logger

try:
    import praw  # type: ignore
except ImportError:
    raise ImportError("praw` not installed. Please install using `pip install praw`")


class RedditTools(Toolkit):
    def __init__(
        self,
        reddit_instance: Optional[praw.Reddit] = None,
        client_id: Optional[str] = None,
        client_secret: Optional[str] = None,
        user_agent: Optional[str] = None,
        username: Optional[str] = None,
        password: Optional[str] = None,
        **kwargs,
    ):
        if reddit_instance is not None:
            log_info("Using provided Reddit instance")
            self.reddit = reddit_instance
        else:
            # Get credentials from environment variables if not provided
            self.client_id = client_id or getenv("REDDIT_CLIENT_ID")
            self.client_secret = client_secret or getenv("REDDIT_CLIENT_SECRET")
            self.user_agent = user_agent or getenv("REDDIT_USER_AGENT", "RedditTools v1.0")
            self.username = username or getenv("REDDIT_USERNAME")
            self.password = password or getenv("REDDIT_PASSWORD")

            self.reddit = None
            # Check if we have all required credentials
            if all([self.client_id, self.client_secret]):
                # Initialize with read-only access if no user credentials
                if not all([self.username, self.password]):
                    log_info("Initializing Reddit client with read-only access")
                    self.reddit = praw.Reddit(
                        client_id=self.client_id,
                        client_secret=self.client_secret,
                        user_agent=self.user_agent,
                    )
                # Initialize with user authentication if credentials provided
                else:
                    log_info(f"Initializing Reddit client with user authentication for u/{self.username}")
                    self.reddit = praw.Reddit(
                        client_id=self.client_id,
                        client_secret=self.client_secret,
                        user_agent=self.user_agent,
                        username=self.username,
                        password=self.password,
                    )
            else:
                logger.warning("Missing Reddit API credentials")

        tools: List[Callable] = [
            self.get_user_info,
            self.get_top_posts,
            self.get_subreddit_info,
            self.get_trending_subreddits,
            self.get_subreddit_stats,
            self.create_post,
            self.reply_to_post,
            self.reply_to_comment,
        ]

        super().__init__(name="reddit", tools=tools, **kwargs)

    def _check_user_auth(self) -> bool:
        """
        Check if user authentication is available for actions that require it.
        Returns:
            bool: True if user is authenticated, False otherwise
        """
        if not self.reddit:
            logger.error("Reddit client not initialized")
            return False

        if not all([self.username, self.password]):
            logger.error("User authentication required. Please provide username and password.")
            return False

        try:
            # Verify authentication by checking if we can get the authenticated user
            self.reddit.user.me()
            return True
        except Exception as e:
            logger.error(f"Authentication error: {e}")
            return False

    def get_user_info(self, username: str) -> str:
        """Get information about a Reddit user."""
        if not self.reddit:
            return "Please provide Reddit API credentials"

        try:
            log_info(f"Getting info for u/{username}")

            user = self.reddit.redditor(username)
            info: Dict[str, Union[str, int, bool, float]] = {
                "name": user.name,
                "comment_karma": user.comment_karma,
                "link_karma": user.link_karma,
                "is_mod": user.is_mod,
                "is_gold": user.is_gold,
                "is_employee": user.is_employee,
                "created_utc": user.created_utc,
            }

            return json.dumps(info)

        except Exception as e:
            return f"Error getting user info: {e}"

    def get_top_posts(self, subreddit: str, time_filter: str = "week", limit: int = 10) -> str:
        """
        Get top posts from a subreddit for a specific time period.
        Args:
            subreddit (str): Name of the subreddit.
            time_filter (str): Time period to filter posts.
            limit (int): Number of posts to fetch.
        Returns:
            str: JSON string containing top posts.
        """
        if not self.reddit:
            return "Please provide Reddit API credentials"

        try:
            log_debug(f"Getting top posts from r/{subreddit}")
            posts = self.reddit.subreddit(subreddit).top(time_filter=time_filter, limit=limit)
            top_posts: List[Dict[str, Union[str, int, float]]] = [
                {
                    "id": post.id,
                    "title": post.title,
                    "score": post.score,
                    "url": post.url,
                    "selftext": post.selftext,
                    "author": str(post.author),
                    "permalink": post.permalink,
                    "created_utc": post.created_utc,
                    "subreddit": str(post.subreddit),
                    "subreddit_name_prefixed": post.subreddit_name_prefixed,
                }
                for post in posts
            ]
            return json.dumps({"top_posts": top_posts})
        except Exception as e:
            return f"Error getting top posts: {e}"

    def get_subreddit_info(self, subreddit_name: str) -> str:
        """
        Get information about a specific subreddit.
        Args:
            subreddit_name (str): Name of the subreddit.
        Returns:
            str: JSON string containing subreddit information.
        """
        if not self.reddit:
            return "Please provide Reddit API credentials"

        try:
            log_info(f"Getting info for r/{subreddit_name}")

            subreddit = self.reddit.subreddit(subreddit_name)
            flairs = [flair["text"] for flair in subreddit.flair.link_templates]
            info: Dict[str, Union[str, int, bool, float, List[str]]] = {
                "display_name": subreddit.display_name,
                "title": subreddit.title,
                "description": subreddit.description,
                "subscribers": subreddit.subscribers,
                "created_utc": subreddit.created_utc,
                "over18": subreddit.over18,
                "available_flairs": flairs,
                "public_description": subreddit.public_description,
                "url": subreddit.url,
            }

            return json.dumps(info)

        except Exception as e:
            return f"Error getting subreddit info: {e}"

    def get_trending_subreddits(self) -> str:
        """Get currently trending subreddits."""
        if not self.reddit:
            return "Please provide Reddit API credentials"

        try:
            log_debug("Getting trending subreddits")
            popular_subreddits = self.reddit.subreddits.popular(limit=5)
            trending: List[str] = [subreddit.display_name for subreddit in popular_subreddits]
            return json.dumps({"trending_subreddits": trending})
        except Exception as e:
            return f"Error getting trending subreddits: {e}"

    def get_subreddit_stats(self, subreddit: str) -> str:
        """
        Get statistics about a subreddit.
        Args:
            subreddit (str): Name of the subreddit.
        Returns:
            str: JSON string containing subreddit statistics
        """
        if not self.reddit:
            return "Please provide Reddit API credentials"

        try:
            log_debug(f"Getting stats for r/{subreddit}")
            sub = self.reddit.subreddit(subreddit)
            stats: Dict[str, Union[str, int, bool, float]] = {
                "display_name": sub.display_name,
                "subscribers": sub.subscribers,
                "active_users": sub.active_user_count,
                "description": sub.description,
                "created_utc": sub.created_utc,
                "over18": sub.over18,
                "public_description": sub.public_description,
            }
            return json.dumps({"subreddit_stats": stats})
        except Exception as e:
            return f"Error getting subreddit stats: {e}"

    def create_post(
        self,
        subreddit: str,
        title: str,
        content: str,
        flair: Optional[str] = None,
        is_self: bool = True,
    ) -> str:
        """
        Create a new post in a subreddit.

        Args:
            subreddit (str): Name of the subreddit to post in.
            title (str): Title of the post.
            content (str): Content of the post (text for self posts, URL for link posts).
            flair (Optional[str]): Flair to add to the post. Must be an available flair in the subreddit.
            is_self (bool): Whether this is a self (text) post (True) or link post (False).
        Returns:
            str: JSON string containing the created post information.
        """
        if not self.reddit:
            return "Please provide Reddit API credentials"

        if not self._check_user_auth():
            return "User authentication required for posting. Please provide username and password."

        try:
            log_info(f"Creating post in r/{subreddit}")

            subreddit_obj = self.reddit.subreddit(subreddit)

            if flair:
                available_flairs = [f["text"] for f in subreddit_obj.flair.link_templates]
                if flair not in available_flairs:
                    return f"Invalid flair. Available flairs: {', '.join(available_flairs)}"

            if is_self:
                submission = subreddit_obj.submit(
                    title=title,
                    selftext=content,
                    flair_id=flair,
                )
            else:
                submission = subreddit_obj.submit(
                    title=title,
                    url=content,
                    flair_id=flair,
                )
            log_info(f"Post created: {submission.permalink}")

            post_info: Dict[str, Union[str, int, float]] = {
                "id": submission.id,
                "title": submission.title,
                "url": submission.url,
                "permalink": submission.permalink,
                "created_utc": submission.created_utc,
                "author": str(submission.author),
                "flair": submission.link_flair_text,
            }

            return json.dumps({"post": post_info})

        except Exception as e:
            return f"Error creating post: {e}"

    def reply_to_post(self, post_id: str, content: str, subreddit: Optional[str] = None) -> str:
        """
        Post a reply to an existing Reddit post or comment.

        Args:
            post_id (str): The ID of the post or comment to reply to.
                          Can be a full URL, permalink, or just the ID.
            content (str): The content of the reply.
            subreddit (Optional[str]): The subreddit name if known.
                                     This helps with error handling and validation.

        Returns:
            str: JSON string containing information about the created reply.
        """
        if not self.reddit:
            logger.error("Reddit instance not initialized")
            return "Please provide Reddit API credentials"

        if not self._check_user_auth():
            logger.error("User authentication failed")
            return "User authentication required for posting replies. Please provide username and password."

        try:
            log_debug(f"Creating reply to post {post_id}")

            # Clean up the post_id if it's a full URL or permalink
            if "/" in post_id:
                # Extract the actual ID from the URL/permalink
                original_id = post_id
                post_id = post_id.split("/")[-1]
                log_debug(f"Extracted post ID {post_id} from {original_id}")

            # Verify post exists
            if not self._check_post_exists(post_id):
                error_msg = f"Post with ID {post_id} does not exist or is not accessible"
                logger.error(error_msg)
                return error_msg

            # Get the submission object
            submission = self.reddit.submission(id=post_id)

            log_debug(
                f"Post details: Title: {submission.title}, Author: {submission.author}, Subreddit: {submission.subreddit.display_name}"
            )

            # If subreddit was provided, verify we're in the right place
            if subreddit and submission.subreddit.display_name.lower() != subreddit.lower():
                error_msg = f"Error: Post ID belongs to r/{submission.subreddit.display_name}, not r/{subreddit}"
                logger.error(error_msg)
                return error_msg

            # Create the reply
            log_debug(f"Attempting to post reply with content length: {len(content)}")
            reply = submission.reply(body=content)

            # Prepare the response information
            reply_info: Dict[str, Union[str, int, float]] = {
                "id": reply.id,
                "body": reply.body,
                "score": reply.score,
                "permalink": reply.permalink,
                "created_utc": reply.created_utc,
                "author": str(reply.author),
                "parent_id": reply.parent_id,
                "submission_id": submission.id,
                "subreddit": str(reply.subreddit),
            }

            log_debug(f"Reply created successfully: {reply.permalink}")
            return json.dumps({"reply": reply_info})

        except praw.exceptions.RedditAPIException as api_error:
            # Handle specific Reddit API errors
            error_messages = [f"{error.error_type}: {error.message}" for error in api_error.items]
            error_msg = f"Reddit API Error: {'; '.join(error_messages)}"
            logger.error(error_msg)
            return error_msg

        except Exception as e:
            error_msg = f"Error creating reply: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def reply_to_comment(self, comment_id: str, content: str, subreddit: Optional[str] = None) -> str:
        """
        Post a reply to an existing Reddit comment.

        Args:
            comment_id (str): The ID of the comment to reply to.
                            Can be a full URL, permalink, or just the ID.
            content (str): The content of the reply.
            subreddit (Optional[str]): The subreddit name if known.
                                     This helps with error handling and validation.

        Returns:
            str: JSON string containing information about the created reply.
        """
        if not self.reddit:
            logger.error("Reddit instance not initialized")
            return "Please provide Reddit API credentials"

        if not self._check_user_auth():
            logger.error("User authentication failed")
            return "User authentication required for posting replies. Please provide username and password."

        try:
            log_debug(f"Creating reply to comment {comment_id}")

            # Clean up the comment_id if it's a full URL or permalink
            if "/" in comment_id:
                original_id = comment_id
                comment_id = comment_id.split("/")[-1]
                log_info(f"Extracted comment ID {comment_id} from {original_id}")

            # Get the comment object
            comment = self.reddit.comment(id=comment_id)

            log_debug(f"Comment details: Author: {comment.author}, Subreddit: {comment.subreddit.display_name}")

            # If subreddit was provided, verify we're in the right place
            if subreddit and comment.subreddit.display_name.lower() != subreddit.lower():
                error_msg = f"Error: Comment ID belongs to r/{comment.subreddit.display_name}, not r/{subreddit}"
                logger.error(error_msg)
                return error_msg

            # Create the reply
            log_debug(f"Attempting to post reply with content length: {len(content)}")
            reply = comment.reply(body=content)

            # Prepare the response information
            reply_info: Dict[str, Union[str, int, float]] = {
                "id": reply.id,
                "body": reply.body,
                "score": reply.score,
                "permalink": reply.permalink,
                "created_utc": reply.created_utc,
                "author": str(reply.author),
                "parent_id": reply.parent_id,
                "submission_id": comment.submission.id,
                "subreddit": str(reply.subreddit),
            }

            log_debug(f"Reply created successfully: {reply.permalink}")
            return json.dumps({"reply": reply_info})

        except praw.exceptions.RedditAPIException as api_error:
            # Handle specific Reddit API errors
            error_messages = [f"{error.error_type}: {error.message}" for error in api_error.items]
            error_msg = f"Reddit API Error: {'; '.join(error_messages)}"
            logger.error(error_msg)
            return error_msg

        except Exception as e:
            error_msg = f"Error creating reply: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def _check_post_exists(self, post_id: str) -> bool:
        """
        Verify that a post exists and is accessible.

        Args:
            post_id (str): The ID of the post to check

        Returns:
            bool: True if post exists and is accessible, False otherwise
        """
        try:
            submission = self.reddit.submission(id=post_id)
            # Try to access some attributes to verify the post exists
            _ = submission.title
            _ = submission.author
            return True
        except Exception as e:
            logger.error(f"Error checking post existence: {str(e)}")
            return False
```

---

### WikipediaTools (`agno.tools.wikipedia`)
Search and retrieve summaries from Wikipedia.

**Authentication**: None.
**Dependencies**: `pip install wikipedia`

#### Parameters
- `knowledge` (Knowledge): Optional Agno Knowledge object to update base.

#### Source Code
```python
import json
from typing import List, Optional

from agno.knowledge.document import Document
from agno.knowledge.knowledge import Knowledge
from agno.knowledge.reader.wikipedia_reader import WikipediaReader
from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info


class WikipediaTools(Toolkit):
    def __init__(
        self,
        knowledge: Optional[Knowledge] = None,
        all: bool = False,
        **kwargs,
    ):
        tools = []

        self.knowledge: Optional[Knowledge] = knowledge
        if self.knowledge is not None and isinstance(self.knowledge, Knowledge):
            tools.append(self.search_wikipedia_and_update_knowledge_base)
        else:
            tools.append(self.search_wikipedia)  # type: ignore

        super().__init__(name="wikipedia_tools", tools=tools, **kwargs)

    def search_wikipedia_and_update_knowledge_base(self, topic: str) -> str:
        """This function searches wikipedia for a topic, adds the results to the knowledge base and returns them.

        USE THIS FUNCTION TO GET INFORMATION WHICH DOES NOT EXIST.

        :param topic: The topic to search Wikipedia and add to knowledge base.
        :return: Relevant documents from Wikipedia knowledge base.
        """

        if self.knowledge is None:
            return "Knowledge not provided"

        log_debug(f"Adding to knowledge: {topic}")
        self.knowledge.insert(
            topics=[topic],
            reader=WikipediaReader(),
        )
        log_debug(f"Searching knowledge: {topic}")
        relevant_docs: List[Document] = self.knowledge.search(query=topic)
        return json.dumps([doc.to_dict() for doc in relevant_docs])

    def search_wikipedia(self, query: str) -> str:
        """Searches Wikipedia for a query.

        :param query: The query to search for.
        :return: Relevant documents from wikipedia.
        """
        try:
            import wikipedia  # noqa: F401
        except ImportError:
            raise ImportError(
                "The `wikipedia` package is not installed. Please install it via `pip install wikipedia`."
            )

        log_info(f"Searching wikipedia for: {query}")
        return json.dumps(Document(name=query, content=wikipedia.summary(query)).to_dict())
```

---

### YouTubeTools (`agno.tools.youtube`)
Retrieve video data, captions, and timestamps from YouTube.

**Authentication**: None (uses public APIs/scrapers).
**Dependencies**: `pip install youtube_transcript_api`

#### Parameters
- `languages` (List[str]): List of preferred caption languages.
- `proxies` (Dict): Optional proxy settings.

#### Source Code
```python
import json
from typing import Any, Dict, List, Optional
from urllib.parse import parse_qs, urlencode, urlparse
from urllib.request import urlopen

from agno.tools import Toolkit
from agno.utils.log import log_debug

try:
    from youtube_transcript_api import YouTubeTranscriptApi
except ImportError:
    raise ImportError(
        "`youtube_transcript_api` not installed. Please install using `pip install youtube_transcript_api`"
    )


class YouTubeTools(Toolkit):
    def __init__(
        self,
        enable_get_video_captions: bool = True,
        enable_get_video_data: bool = True,
        enable_get_video_timestamps: bool = True,
        all: bool = False,
        languages: Optional[List[str]] = None,
        proxies: Optional[Dict[str, Any]] = None,
        **kwargs,
    ):
        self.languages: Optional[List[str]] = languages
        self.proxies: Optional[Dict[str, Any]] = proxies

        tools: List[Any] = []
        if all or enable_get_video_captions:
            tools.append(self.get_youtube_video_captions)
        if all or enable_get_video_data:
            tools.append(self.get_youtube_video_data)
        if all or enable_get_video_timestamps:
            tools.append(self.get_video_timestamps)

        super().__init__(name="youtube_tools", tools=tools, **kwargs)

    def get_youtube_video_id(self, url: str) -> Optional[str]:
        """Function to get the video ID from a YouTube URL.

        Args:
            url: The URL of the YouTube video.

        Returns:
            str: The video ID of the YouTube video.
        """
        parsed_url = urlparse(url)
        hostname = parsed_url.hostname

        if hostname == "youtu.be":
            return parsed_url.path[1:]
        if hostname in ("www.youtube.com", "youtube.com"):
            if parsed_url.path == "/watch":
                query_params = parse_qs(parsed_url.query)
                return query_params.get("v", [None])[0]
            if parsed_url.path.startswith("/embed/"):
                return parsed_url.path.split("/")[2]
            if parsed_url.path.startswith("/v/"):
                return parsed_url.path.split("/")[2]
        return None

    def get_youtube_video_data(self, url: str) -> str:
        """Function to get video data from a YouTube URL.
        Data returned includes {title, author_name, author_url, type, height, width, version, provider_name, provider_url, thumbnail_url}

        Args:
            url: The URL of the YouTube video.

        Returns:
            str: JSON data of the YouTube video.
        """
        if not url:
            return "No URL provided"

        log_debug(f"Getting video data for youtube video: {url}")

        try:
            video_id = self.get_youtube_video_id(url)
        except Exception:
            return "Error getting video ID from URL, please provide a valid YouTube url"

        try:
            params = {"format": "json", "url": f"https://www.youtube.com/watch?v={video_id}"}
            url = "https://www.youtube.com/oembed"
            query_string = urlencode(params)
            url = url + "?" + query_string

            with urlopen(url) as response:
                response_text = response.read()
                video_data = json.loads(response_text.decode())
                clean_data = {
                    "title": video_data.get("title"),
                    "author_name": video_data.get("author_name"),
                    "author_url": video_data.get("author_url"),
                    "type": video_data.get("type"),
                    "height": video_data.get("height"),
                    "width": video_data.get("width"),
                    "version": video_data.get("version"),
                    "provider_name": video_data.get("provider_name"),
                    "provider_url": video_data.get("provider_url"),
                    "thumbnail_url": video_data.get("thumbnail_url"),
                }
                return json.dumps(clean_data, indent=4)
        except Exception as e:
            return f"Error getting video data: {e}"

    def get_youtube_video_captions(self, url: str) -> str:
        """Use this function to get captions from a YouTube video.

        Args:
            url: The URL of the YouTube video.

        Returns:
            str: The captions of the YouTube video.
        """
        if not url:
            return "No URL provided"

        log_debug(f"Getting captions for youtube video: {url}")

        try:
            video_id = self.get_youtube_video_id(url)
        except Exception:
            return "Error getting video ID from URL, please provide a valid YouTube url"

        try:
            captions = None
            kwargs: Dict = {}
            if self.languages:
                kwargs["languages"] = self.languages or ["en"]
            if self.proxies:
                kwargs["proxies"] = self.proxies
            if video_id is not None:
                captions = YouTubeTranscriptApi().fetch(video_id, **kwargs)
            else:
                return "No video ID found"
            if captions:
                return " ".join(line.text for line in captions)
            return "No captions found for video"
        except Exception as e:
            # log_info(f"Error getting captions for video {video_id}: {e}")
            return f"Error getting captions for video: {e}"

    def get_video_timestamps(self, url: str) -> str:
        """Generate timestamps for a YouTube video based on captions.

        Args:
            url: The URL of the YouTube video.

        Returns:
            str: Timestamps and summaries for the video.
        """
        if not url:
            return "No URL provided"

        log_debug(f"Getting timestamps for youtube video: {url}")

        try:
            video_id = self.get_youtube_video_id(url)
        except Exception:
            return "Error getting video ID from URL, please provide a valid YouTube url"

        if video_id is None:
            return "No video ID found"

        try:
            kwargs: Dict = {}
            if self.languages:
                kwargs["languages"] = self.languages or ["en"]
            if self.proxies:
                kwargs["proxies"] = self.proxies

            captions = YouTubeTranscriptApi().fetch(video_id, **kwargs)
            timestamps = []
            for line in captions:
                start = int(line.start)
                minutes, seconds = divmod(start, 60)
                timestamps.append(f"{minutes}:{seconds:02d} - {line.text}")
            return "\n".join(timestamps)
        except Exception as e:
            return f"Error generating timestamps: {e}"
```

---

### OpenWeatherTools (`agno.tools.openweather`)
Access real-time weather, forecasts, and air pollution data.

**Authentication**: Requires `OPENWEATHER_API_KEY`.
**Dependencies**: `pip install requests`

#### Parameters
- `units` (str): 'metric', 'imperial', 'standard'.
- `enable_current_weather` (bool): Default True.

#### Source Code
```python
import json
from os import getenv
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_info, logger

try:
    import requests
except ImportError:
    raise ImportError("`requests` not installed. Please install using `pip install requests`")


class OpenWeatherTools(Toolkit):
    """
    OpenWeather is a toolkit for accessing weather data from OpenWeatherMap API.

    Args:
        api_key (Optional[str]): OpenWeatherMap API key. If not provided, will try to get from OPENWEATHER_API_KEY env var.
        units (str): Units of measurement. Options are 'standard', 'metric', and 'imperial'. Default is 'metric'.
        enable_current_weather (bool): Enable current weather function. Default is True.
        enable_forecast (bool): Enable forecast function. Default is True.
        enable_air_pollution (bool): Enable air pollution function. Default is True.
        enable_geocoding (bool): Enable geocoding function. Default is True.
        all (bool): Enable all functions. Default is False.
    """

    def __init__(
        self,
        api_key: Optional[str] = None,
        units: str = "metric",
        enable_current_weather: bool = True,
        enable_forecast: bool = True,
        enable_air_pollution: bool = True,
        enable_geocoding: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.api_key = api_key or getenv("OPENWEATHER_API_KEY")
        if not self.api_key:
            raise ValueError(
                "OpenWeather API key is required. Provide it as an argument or set the OPENWEATHER_API_KEY environment variable."
            )

        self.units = units
        self.base_url = "https://api.openweathermap.org/data/2.5"
        self.geo_url = "https://api.openweathermap.org/geo/1.0"

        tools: List[Any] = []
        if enable_current_weather or all:
            tools.append(self.get_current_weather)
        if enable_forecast or all:
            tools.append(self.get_forecast)
        if enable_air_pollution or all:
            tools.append(self.get_air_pollution)
        if enable_geocoding or all:
            tools.append(self.geocode_location)

        super().__init__(name="openweather_tools", tools=tools, **kwargs)

    def _make_request(self, url: str, params: Dict) -> Dict:
        """Make a request to the OpenWeatherMap API.

        Args:
            url (str): The API endpoint URL.
            params (Dict): Query parameters for the request.

        Returns:
            Dict: The JSON response from the API.
        """
        try:
            params["appid"] = self.api_key
            response = requests.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            logger.error(f"Error making request to {url}: {e}")
            return {"error": str(e)}

    def geocode_location(self, location: str, limit: int = 1) -> str:
        """Convert a location name to geographic coordinates.

        Args:
            location (str): The name of the city, e.g., "London", "Paris", "New York".
            limit (int): Maximum number of location results. Default is 1.

        Returns:
            str: JSON string containing location data with coordinates.
        """
        try:
            log_info(f"Geocoding location: {location}")
            url = f"{self.geo_url}/direct"
            params = {"q": location, "limit": limit}

            result = self._make_request(url, params)

            if "error" in result:
                return json.dumps(result)

            if not result:
                return json.dumps({"error": f"No location found for '{location}'"})

            return json.dumps(result, indent=2)
        except Exception as e:
            logger.error(f"Error geocoding location: {e}")
            return json.dumps({"error": str(e)})

    def get_current_weather(self, location: str) -> str:
        """Get current weather data for a location.

        Args:
            location (str): The name of the city, e.g., "London", "Paris", "New York".

        Returns:
            str: JSON string containing current weather data.
        """
        try:
            log_info(f"Getting current weather for: {location}")

            # First geocode the location to get coordinates
            geocode_result = json.loads(self.geocode_location(location))
            if "error" in geocode_result:
                return json.dumps(geocode_result)

            if not geocode_result:
                return json.dumps({"error": f"No location found for '{location}'"})

            # Get the first location result
            loc_data = geocode_result[0]
            lat, lon = loc_data["lat"], loc_data["lon"]

            # Get current weather using coordinates
            url = f"{self.base_url}/weather"
            params = {"lat": lat, "lon": lon, "units": self.units}

            result = self._make_request(url, params)

            # Add the location name to the result
            if "error" not in result:
                result["location_name"] = loc_data.get("name", location)
                result["country"] = loc_data.get("country", "")

            return json.dumps(result, indent=2)
        except Exception as e:
            logger.error(f"Error getting current weather: {e}")
            return json.dumps({"error": str(e)})

    def get_forecast(self, location: str, days: int = 5) -> str:
        """Get weather forecast for a location.

        Args:
            location (str): The name of the city, e.g., "London", "Paris", "New York".
            days (int): Number of days for forecast (max 5). Default is 5.

        Returns:
            str: JSON string containing forecast data.
        """
        try:
            log_info(f"Getting {days}-day forecast for: {location}")

            # First geocode the location to get coordinates
            geocode_result = json.loads(self.geocode_location(location))
            if "error" in geocode_result:
                return json.dumps(geocode_result)

            if not geocode_result:
                return json.dumps({"error": f"No location found for '{location}'"})

            # Get the first location result
            loc_data = geocode_result[0]
            lat, lon = loc_data["lat"], loc_data["lon"]

            # Get forecast using coordinates
            url = f"{self.base_url}/forecast"
            params = {
                "lat": lat,
                "lon": lon,
                "units": self.units,
                # Each day has 8 3-hour forecasts, max 5 days (40 entries)
                "cnt": min(days * 8, 40),
            }

            result = self._make_request(url, params)

            # Add the location name to the result
            if "error" not in result:
                result["location_name"] = loc_data.get("name", location)
                result["country"] = loc_data.get("country", "")

            return json.dumps(result, indent=2)
        except Exception as e:
            logger.error(f"Error getting forecast: {e}")
            return json.dumps({"error": str(e)})

    def get_air_pollution(self, location: str) -> str:
        """Get current air pollution data for a location.

        Args:
            location (str): The name of the city, e.g., "London", "Paris", "New York".

        Returns:
            str: JSON string containing air pollution data.
        """
        try:
            log_info(f"Getting air pollution data for: {location}")

            # First geocode the location to get coordinates
            geocode_result = json.loads(self.geocode_location(location))
            if "error" in geocode_result:
                return json.dumps(geocode_result)

            if not geocode_result:
                return json.dumps({"error": f"No location found for '{location}'"})

            # Get the first location result
            loc_data = geocode_result[0]
            lat, lon = loc_data["lat"], loc_data["lon"]

            # Get air pollution data using coordinates
            url = f"{self.base_url}/air_pollution"
            params = {"lat": lat, "lon": lon}

            result = self._make_request(url, params)

            # Add the location name to the result
            if "error" not in result:
                result["location_name"] = loc_data.get("name", location)
                result["country"] = loc_data.get("country", "")

            return json.dumps(result, indent=2)
        except Exception as e:
            logger.error(f"Error getting air pollution data: {e}")
            return json.dumps({"error": str(e)})
```


## 15. Financial & Business (Detailed)

### YFinanceTools (`agno.tools.yfinance`)
Get real-time and historical financial data from Yahoo Finance.

**Authentication**: None.
**Dependencies**: `pip install yfinance`

#### Parameters
- `enable_stock_price` (bool): Default True.
- `enable_company_info` (bool): Default False.
- `all` (bool): Default False.

#### Source Code
```python
import json
from typing import Any, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug

try:
    import yfinance as yf
except ImportError:
    raise ImportError("`yfinance` not installed. Please install using `pip install yfinance`.")


class YFinanceTools(Toolkit):
    """
    YFinanceTools is a toolkit for getting financial data from Yahoo Finance.

    Args:
        enable_stock_price (bool): Enable the get_current_stock_price tool. Default: True.
        enable_company_info (bool): Enable the get_company_info tool. Default: False.
        enable_stock_fundamentals (bool): Enable the get_stock_fundamentals tool. Default: False.
        enable_income_statements (bool): Enable the get_income_statements tool. Default: False.
        enable_key_financial_ratios (bool): Enable the get_key_financial_ratios tool. Default: False.
        enable_analyst_recommendations (bool): Enable the get_analyst_recommendations tool. Default: False.
        enable_company_news (bool): Enable the get_company_news tool. Default: False.
        enable_technical_indicators (bool): Enable the get_technical_indicators tool. Default: False.
        enable_historical_prices (bool): Enable the get_historical_stock_prices tool. Default: False.
        all (bool): Enable all tools. Overrides individual flags when True. Default: False.
        session (Optional[Any]): Optional session for yfinance requests.
    """

    def __init__(
        self,
        enable_stock_price: bool = True,
        enable_company_info: bool = False,
        enable_stock_fundamentals: bool = False,
        enable_income_statements: bool = False,
        enable_key_financial_ratios: bool = False,
        enable_analyst_recommendations: bool = False,
        enable_company_news: bool = False,
        enable_technical_indicators: bool = False,
        enable_historical_prices: bool = False,
        all: bool = False,
        session: Optional[Any] = None,
        **kwargs,
    ):
        self.session = session

        tools: List[Any] = []
        if all or enable_stock_price:
            tools.append(self.get_current_stock_price)
        if all or enable_company_info:
            tools.append(self.get_company_info)
        if all or enable_stock_fundamentals:
            tools.append(self.get_stock_fundamentals)
        if all or enable_income_statements:
            tools.append(self.get_income_statements)
        if all or enable_key_financial_ratios:
            tools.append(self.get_key_financial_ratios)
        if all or enable_analyst_recommendations:
            tools.append(self.get_analyst_recommendations)
        if all or enable_company_news:
            tools.append(self.get_company_news)
        if all or enable_technical_indicators:
            tools.append(self.get_technical_indicators)
        if all or enable_historical_prices:
            tools.append(self.get_historical_stock_prices)

        super().__init__(name="yfinance_tools", tools=tools, **kwargs)

    def get_current_stock_price(self, symbol: str) -> str:
        """
        Use this function to get the current stock price for a given symbol.

        Args:
            symbol (str): The stock symbol.

        Returns:
            str: The current stock price or error message.
        """
        try:
            log_debug(f"Fetching current price for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            # Use "regularMarketPrice" for regular market hours, or "currentPrice" for pre/post market
            current_price = stock.info.get("regularMarketPrice", stock.info.get("currentPrice"))
            return f"{current_price:.4f}" if current_price else f"Could not fetch current price for {symbol}"
        except Exception as e:
            return f"Error fetching current price for {symbol}: {e}"

    def get_company_info(self, symbol: str) -> str:
        """Use this function to get company information and overview for a given stock symbol.

        Args:
            symbol (str): The stock symbol.

        Returns:
            str: JSON containing company profile and overview.
        """
        try:
            company_info_full = yf.Ticker(symbol, session=self.session).info
            if company_info_full is None:
                return f"Could not fetch company info for {symbol}"

            log_debug(f"Fetching company info for {symbol}")

            company_info_cleaned = {
                "Name": company_info_full.get("shortName"),
                "Symbol": company_info_full.get("symbol"),
                "Current Stock Price": f"{company_info_full.get('regularMarketPrice', company_info_full.get('currentPrice'))} {company_info_full.get('currency', 'USD')}",
                "Market Cap": f"{company_info_full.get('marketCap', company_info_full.get('enterpriseValue'))} {company_info_full.get('currency', 'USD')}",
                "Sector": company_info_full.get("sector"),
                "Industry": company_info_full.get("industry"),
                "Address": company_info_full.get("address1"),
                "City": company_info_full.get("city"),
                "State": company_info_full.get("state"),
                "Zip": company_info_full.get("zip"),
                "Country": company_info_full.get("country"),
                "EPS": company_info_full.get("trailingEps"),
                "P/E Ratio": company_info_full.get("trailingPE"),
                "52 Week Low": company_info_full.get("fiftyTwoWeekLow"),
                "52 Week High": company_info_full.get("fiftyTwoWeekHigh"),
                "50 Day Average": company_info_full.get("fiftyDayAverage"),
                "200 Day Average": company_info_full.get("twoHundredDayAverage"),
                "Website": company_info_full.get("website"),
                "Summary": company_info_full.get("longBusinessSummary"),
                "Analyst Recommendation": company_info_full.get("recommendationKey"),
                "Number Of Analyst Opinions": company_info_full.get("numberOfAnalystOpinions"),
                "Employees": company_info_full.get("fullTimeEmployees"),
                "Total Cash": company_info_full.get("totalCash"),
                "Free Cash flow": company_info_full.get("freeCashflow"),
                "Operating Cash flow": company_info_full.get("operatingCashflow"),
                "EBITDA": company_info_full.get("ebitda"),
                "Revenue Growth": company_info_full.get("revenueGrowth"),
                "Gross Margins": company_info_full.get("grossMargins"),
                "Ebitda Margins": company_info_full.get("ebitdaMargins"),
            }
            return json.dumps(company_info_cleaned, indent=2)
        except Exception as e:
            return f"Error fetching company profile for {symbol}: {e}"

    def get_historical_stock_prices(self, symbol: str, period: str = "1mo", interval: str = "1d") -> str:
        """
        Use this function to get the historical stock price for a given symbol.

        Args:
            symbol (str): The stock symbol.
            period (str): The period for which to retrieve historical prices. Defaults to "1mo".
                        Valid periods: 1d,5d,1mo,3mo,6mo,1y,2y,5y,10y,ytd,max
            interval (str): The interval between data points. Defaults to "1d".
                        Valid intervals: 1d,5d,1wk,1mo,3mo

        Returns:
          str: The current stock price or error message.
        """
        try:
            log_debug(f"Fetching historical prices for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            historical_price = stock.history(period=period, interval=interval)
            return historical_price.to_json(orient="index")
        except Exception as e:
            return f"Error fetching historical prices for {symbol}: {e}"

    def get_stock_fundamentals(self, symbol: str) -> str:
        """Use this function to get fundamental data for a given stock symbol yfinance API.

        Args:
            symbol (str): The stock symbol.

        Returns:
            str: A JSON string containing fundamental data or an error message.
                Keys:
                    - 'symbol': The stock symbol.
                    - 'company_name': The long name of the company.
                    - 'sector': The sector to which the company belongs.
                    - 'industry': The industry to which the company belongs.
                    - 'market_cap': The market capitalization of the company.
                    - 'pe_ratio': The forward price-to-earnings ratio.
                    - 'pb_ratio': The price-to-book ratio.
                    - 'dividend_yield': The dividend yield.
                    - 'eps': The trailing earnings per share.
                    - 'beta': The beta value of the stock.
                    - '52_week_high': The 52-week high price of the stock.
                    - '52_week_low': The 52-week low price of the stock.
        """
        try:
            log_debug(f"Fetching fundamentals for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            info = stock.info
            fundamentals = {
                "symbol": symbol,
                "company_name": info.get("longName", ""),
                "sector": info.get("sector", ""),
                "industry": info.get("industry", ""),
                "market_cap": info.get("marketCap", "N/A"),
                "pe_ratio": info.get("forwardPE", "N/A"),
                "pb_ratio": info.get("priceToBook", "N/A"),
                "dividend_yield": info.get("dividendYield", "N/A"),
                "eps": info.get("trailingEps", "N/A"),
                "beta": info.get("beta", "N/A"),
                "52_week_high": info.get("fiftyTwoWeekHigh", "N/A"),
                "52_week_low": info.get("fiftyTwoWeekLow", "N/A"),
            }
            return json.dumps(fundamentals, indent=2)
        except Exception as e:
            return f"Error getting fundamentals for {symbol}: {e}"

    def get_income_statements(self, symbol: str) -> str:
        """Use this function to get income statements for a given stock symbol.

        Args:
            symbol (str): The stock symbol.

        Returns:
            dict: JSON containing income statements or an empty dictionary.
        """
        try:
            log_debug(f"Fetching income statements for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            financials = stock.financials
            return financials.to_json(orient="index")
        except Exception as e:
            return f"Error fetching income statements for {symbol}: {e}"

    def get_key_financial_ratios(self, symbol: str) -> str:
        """Use this function to get key financial ratios for a given stock symbol.

        Args:
            symbol (str): The stock symbol.

        Returns:
            dict: JSON containing key financial ratios.
        """
        try:
            log_debug(f"Fetching key financial ratios for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            key_ratios = stock.info
            return json.dumps(key_ratios, indent=2)
        except Exception as e:
            return f"Error fetching key financial ratios for {symbol}: {e}"

    def get_analyst_recommendations(self, symbol: str) -> str:
        """Use this function to get analyst recommendations for a given stock symbol.

        Args:
            symbol (str): The stock symbol.

        Returns:
            str: JSON containing analyst recommendations.
        """
        try:
            log_debug(f"Fetching analyst recommendations for {symbol}")
            stock = yf.Ticker(symbol, session=self.session)
            recommendations = stock.recommendations
            return recommendations.to_json(orient="index")
        except Exception as e:
            return f"Error fetching analyst recommendations for {symbol}: {e}"

    def get_company_news(self, symbol: str, num_stories: int = 3) -> str:
        """Use this function to get company news and press releases for a given stock symbol.

        Args:
            symbol (str): The stock symbol.
            num_stories (int): The number of news stories to return. Defaults to 3.

        Returns:
            str: JSON containing company news and press releases.
        """
        try:
            log_debug(f"Fetching company news for {symbol}")
            news = yf.Ticker(symbol, session=self.session).news
            return json.dumps(news[:num_stories], indent=2)
        except Exception as e:
            return f"Error fetching company news for {symbol}: {e}"

    def get_technical_indicators(self, symbol: str, period: str = "3mo") -> str:
        """Use this function to get technical indicators for a given stock symbol.

        Args:
            symbol (str): The stock symbol.
            period (str): The time period for which to retrieve technical indicators.
                Valid periods: 1d, 5d, 1mo, 3mo, 6mo, 1y, 2y, 5y, 10y, ytd, max. Defaults to 3mo.

        Returns:
            str: JSON containing technical indicators.
        """
        try:
            log_debug(f"Fetching technical indicators for {symbol}")
            indicators = yf.Ticker(symbol, session=self.session).history(period=period)
            return indicators.to_json(orient="index")
        except Exception as e:
            return f"Error fetching technical indicators for {symbol}: {e}"
```

---

### ShopifyTools (`agno.tools.shopify`)
Analyze sales, products, and customers using the Shopify Admin GraphQL API.

**Authentication**: Requires `SHOPIFY_SHOP_NAME` and `SHOPIFY_ACCESS_TOKEN`.
**Dependencies**: `pip install httpx`

#### Parameters
- `shop_name` (str): Store name (subdomain).
- `access_token` (str): API access token.
- `api_version` (str): Default '2025-10'.

#### Source Code
```python
"""
Shopify Toolkit for Agno SDK

A toolkit for analyzing sales data, product performance, and customer insights using the Shopify Admin GraphQL API.
Requires a valid Shopify access token with appropriate scopes.

Required scopes:
- read_orders (for order and sales data)
- read_products (for product information)
- read_customers (for customer insights)
- read_analytics (for analytics data)
"""

import json
from collections import Counter
from datetime import datetime, timedelta
from itertools import combinations
from os import getenv
from typing import Any, Dict, List, Optional

import httpx

from agno.tools import Toolkit
from agno.utils.log import log_debug


class ShopifyTools(Toolkit):
    """
    Shopify toolkit for analyzing sales data and product performance.

    Args:
        shop_name: Your Shopify store name (e.g., 'my-store' from my-store.myshopify.com).
        access_token: Shopify Admin API access token with required scopes.
        api_version: Shopify API version (default: '2025-10').
        timeout: Request timeout in seconds.
    """

    def __init__(
        self,
        shop_name: Optional[str] = None,
        access_token: Optional[str] = None,
        api_version: str = "2025-10",
        timeout: int = 30,
        **kwargs,
    ):
        self.shop_name = shop_name or getenv("SHOPIFY_SHOP_NAME")
        self.access_token = access_token or getenv("SHOPIFY_ACCESS_TOKEN")
        self.api_version = api_version
        self.timeout = timeout
        self.base_url = f"https://{self.shop_name}.myshopify.com/admin/api/{self.api_version}/graphql.json"

        tools: List[Any] = [
            self.get_shop_info,
            self.get_products,
            self.get_orders,
            self.get_top_selling_products,
            self.get_products_bought_together,
            self.get_sales_by_date_range,
            self.get_order_analytics,
            self.get_product_sales_breakdown,
            self.get_customer_order_history,
            self.get_inventory_levels,
            self.get_low_stock_products,
            self.get_sales_trends,
            self.get_average_order_value,
            self.get_repeat_customers,
        ]

        super().__init__(name="shopify", tools=tools, **kwargs)

    def _make_graphql_request(self, query: str, variables: Optional[Dict] = None) -> Dict:
        """Make an authenticated GraphQL request to the Shopify Admin API."""
        headers = {
            "X-Shopify-Access-Token": self.access_token or "",
            "Content-Type": "application/json",
        }

        body: Dict[str, Any] = {"query": query}
        if variables:
            body["variables"] = variables

        with httpx.Client(timeout=self.timeout) as client:
            response = client.post(
                self.base_url,
                headers=headers,
                json=body,
            )

            try:
                result = response.json()
                if "errors" in result:
                    return {"error": result["errors"]}
                return result.get("data", {})
            except json.JSONDecodeError:
                return {"error": f"Failed to parse response: {response.text}"}

    def get_shop_info(self) -> str:
        """Get basic information about the Shopify store.

        Returns:
            JSON string containing shop name, email, currency, and other details.
        """
        log_debug("Fetching Shopify shop info")

        query = """
        query {
            shop {
                name
                email
                currencyCode
                primaryDomain {
                    url
                }
                billingAddress {
                    country
                    city
                }
                plan {
                    displayName
                }
            }
        }
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        return json.dumps(result.get("shop", {}), indent=2)

    def get_products(
        self,
        max_results: int = 50,
        status: Optional[str] = None,
    ) -> str:
        """Get products from the store.

        Args:
            max_results: Maximum number of products to return (default 50, max 250).
            status: Filter by status - 'ACTIVE', 'ARCHIVED', or 'DRAFT' (optional).

        Returns:
            JSON string containing list of products with id, title, status, variants, and pricing.
        """
        log_debug(f"Fetching products: max_results={max_results}, status={status}")

        query_filter = ""
        if status:
            query_filter = f', query: "status:{status}"'

        query = f"""
        query {{
            products(first: {min(max_results, 250)}{query_filter}) {{
                edges {{
                    node {{
                        id
                        title
                        status
                        totalInventory
                        createdAt
                        updatedAt
                        priceRangeV2 {{
                            minVariantPrice {{
                                amount
                                currencyCode
                            }}
                            maxVariantPrice {{
                                amount
                                currencyCode
                            }}
                        }}
                        variants(first: 10) {{
                            edges {{
                                node {{
                                    id
                                    title
                                    sku
                                    price
                                    inventoryQuantity
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        products = []
        for edge in result.get("products", {}).get("edges", []):
            node = edge["node"]
            products.append(
                {
                    "id": node["id"],
                    "title": node["title"],
                    "status": node["status"],
                    "total_inventory": node.get("totalInventory"),
                    "created_at": node.get("createdAt"),
                    "price_range": {
                        "min": node.get("priceRangeV2", {}).get("minVariantPrice", {}).get("amount"),
                        "max": node.get("priceRangeV2", {}).get("maxVariantPrice", {}).get("amount"),
                        "currency": node.get("priceRangeV2", {}).get("minVariantPrice", {}).get("currencyCode"),
                    },
                    "variants": [
                        {
                            "id": v["node"]["id"],
                            "title": v["node"]["title"],
                            "sku": v["node"].get("sku"),
                            "price": v["node"]["price"],
                            "inventory": v["node"].get("inventoryQuantity"),
                        }
                        for v in node.get("variants", {}).get("edges", [])
                    ],
                }
            )

        return json.dumps(products, indent=2)

    def get_orders(
        self,
        max_results: int = 50,
        status: Optional[str] = None,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Get recent orders from the store.

        Args:
            max_results: Maximum number of orders to return (default 50, max 250).
            status: Filter by financial status - 'paid', 'pending', 'refunded' (optional).
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing list of orders with id, total, customer, and line items.
        """
        log_debug(
            f"Fetching orders: max_results={max_results}, status={status}, created_after={created_after}, created_before={created_before}"
        )

        query_parts = []
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        if status:
            query_parts.append(f"financial_status:{status}")

        query_filter = " AND ".join(query_parts) if query_parts else ""
        query_param = f', query: "{query_filter}"' if query_filter else ""

        query = f"""
        query {{
            orders(first: {min(max_results, 250)}{query_param}, sortKey: CREATED_AT, reverse: true) {{
                edges {{
                    node {{
                        id
                        name
                        createdAt
                        displayFinancialStatus
                        displayFulfillmentStatus
                        totalPriceSet {{
                            shopMoney {{
                                amount
                                currencyCode
                            }}
                        }}
                        subtotalPriceSet {{
                            shopMoney {{
                                amount
                            }}
                        }}
                        customer {{
                            id
                            email
                            firstName
                            lastName
                        }}
                        lineItems(first: 50) {{
                            edges {{
                                node {{
                                    id
                                    title
                                    quantity
                                    variant {{
                                        id
                                        sku
                                    }}
                                    originalUnitPriceSet {{
                                        shopMoney {{
                                            amount
                                        }}
                                    }}
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        orders = []
        for edge in result.get("orders", {}).get("edges", []):
            node = edge["node"]
            customer = node.get("customer") or {}
            orders.append(
                {
                    "id": node["id"],
                    "name": node["name"],
                    "created_at": node["createdAt"],
                    "financial_status": node.get("displayFinancialStatus"),
                    "fulfillment_status": node.get("displayFulfillmentStatus"),
                    "total": node.get("totalPriceSet", {}).get("shopMoney", {}).get("amount"),
                    "subtotal": node.get("subtotalPriceSet", {}).get("shopMoney", {}).get("amount"),
                    "currency": node.get("totalPriceSet", {}).get("shopMoney", {}).get("currencyCode"),
                    "customer": {
                        "id": customer.get("id"),
                        "email": customer.get("email"),
                        "name": f"{customer.get('firstName', '')} {customer.get('lastName', '')}".strip(),
                    }
                    if customer
                    else None,
                    "line_items": [
                        {
                            "id": item["node"]["id"],
                            "title": item["node"]["title"],
                            "quantity": item["node"]["quantity"],
                            "unit_price": item["node"]
                            .get("originalUnitPriceSet", {})
                            .get("shopMoney", {})
                            .get("amount"),
                            "sku": item["node"].get("variant", {}).get("sku") if item["node"].get("variant") else None,
                        }
                        for item in node.get("lineItems", {}).get("edges", [])
                    ],
                }
            )

        return json.dumps(orders, indent=2)

    def get_top_selling_products(
        self,
        limit: int = 10,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Get the top selling products by quantity sold.

        Analyzes order data to find which products sell the most.

        Args:
            limit: Number of top products to return (default 10).
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing ranked list of top products with sales data.
        """
        log_debug(
            f"Calculating top selling products: limit={limit}, created_after={created_after}, created_before={created_before}"
        )

        query_parts = ["financial_status:paid"]
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts)

        query = f"""
        query {{
            orders(first: 250, query: "{query_filter}", sortKey: CREATED_AT) {{
                edges {{
                    node {{
                        lineItems(first: 100) {{
                            edges {{
                                node {{
                                    title
                                    quantity
                                    variant {{
                                        id
                                        product {{
                                            id
                                            title
                                        }}
                                    }}
                                    originalUnitPriceSet {{
                                        shopMoney {{
                                            amount
                                        }}
                                    }}
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        # Aggregate sales by product
        product_sales: Dict[str, Dict[str, Any]] = {}

        for order_edge in result.get("orders", {}).get("edges", []):
            for item_edge in order_edge["node"].get("lineItems", {}).get("edges", []):
                item = item_edge["node"]
                variant = item.get("variant")
                if not variant or not variant.get("product"):
                    continue

                product_id = variant["product"]["id"]
                product_title = variant["product"]["title"]
                quantity = item["quantity"]
                unit_price = float(item.get("originalUnitPriceSet", {}).get("shopMoney", {}).get("amount", 0))

                if product_id not in product_sales:
                    product_sales[product_id] = {
                        "id": product_id,
                        "title": product_title,
                        "total_quantity": 0,
                        "total_revenue": 0.0,
                        "order_count": 0,
                    }

                product_sales[product_id]["total_quantity"] += quantity
                product_sales[product_id]["total_revenue"] += quantity * unit_price
                product_sales[product_id]["order_count"] += 1

        # Sort by quantity and limit
        sorted_products = sorted(product_sales.values(), key=lambda x: x["total_quantity"], reverse=True)[:limit]

        # Add ranking
        for i, product in enumerate(sorted_products):
            product["rank"] = i + 1
            product["total_revenue"] = round(product["total_revenue"], 2)

        return json.dumps(sorted_products, indent=2)

    def get_products_bought_together(
        self,
        min_occurrences: int = 2,
        limit: int = 20,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Find products that are frequently bought together.

        Analyzes orders to find product pairs that appear together most often.
        Useful for cross-selling and bundle recommendations.

        Args:
            min_occurrences: Minimum times a pair must appear together (default 2).
            limit: Number of product pairs to return (default 20).
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing ranked list of product pairs with co-occurrence count.
        """
        log_debug(f"Finding products bought together: created_after={created_after}, created_before={created_before}")

        query_parts = ["financial_status:paid"]
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts)

        query = f"""
        query {{
            orders(first: 250, query: "{query_filter}") {{
                edges {{
                    node {{
                        lineItems(first: 100) {{
                            edges {{
                                node {{
                                    variant {{
                                        product {{
                                            id
                                            title
                                        }}
                                    }}
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        # Count co-occurrences
        pair_counter: Counter = Counter()
        product_info: Dict[str, str] = {}

        for order_edge in result.get("orders", {}).get("edges", []):
            products_in_order = []
            for item_edge in order_edge["node"].get("lineItems", {}).get("edges", []):
                variant = item_edge["node"].get("variant")
                if variant and variant.get("product"):
                    product_id = variant["product"]["id"]
                    product_title = variant["product"]["title"]
                    products_in_order.append(product_id)
                    product_info[product_id] = product_title

            # Get unique products in this order and count pairs
            unique_products = list(set(products_in_order))
            if len(unique_products) >= 2:
                for pair in combinations(sorted(unique_products), 2):
                    pair_counter[pair] += 1

        # Filter by minimum occurrences and sort
        frequent_pairs = [
            {
                "product_1": {
                    "id": pair[0],
                    "title": product_info.get(pair[0], "Unknown"),
                },
                "product_2": {
                    "id": pair[1],
                    "title": product_info.get(pair[1], "Unknown"),
                },
                "times_bought_together": count,
            }
            for pair, count in pair_counter.most_common(limit)
            if count >= min_occurrences
        ]

        return json.dumps(frequent_pairs, indent=2)

    def get_sales_by_date_range(
        self,
        start_date: str,
        end_date: str,
    ) -> str:
        """Get sales summary for a specific date range.

        Args:
            start_date: Start date in YYYY-MM-DD format.
            end_date: End date in YYYY-MM-DD format.

        Returns:
            JSON string containing total revenue, order count, and daily breakdown.
        """
        log_debug(f"Fetching sales for date range: {start_date} to {end_date}")

        query = f"""
        query {{
            orders(first: 250, query: "created_at:>={start_date} AND created_at:<={end_date} AND financial_status:paid", sortKey: CREATED_AT) {{
                edges {{
                    node {{
                        createdAt
                        totalPriceSet {{
                            shopMoney {{
                                amount
                                currencyCode
                            }}
                        }}
                        lineItems(first: 100) {{
                            edges {{
                                node {{
                                    quantity
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        # Aggregate data
        total_revenue = 0.0
        total_orders = 0
        total_items = 0
        daily_breakdown: Dict[str, Dict[str, Any]] = {}
        currency = None

        for order_edge in result.get("orders", {}).get("edges", []):
            order = order_edge["node"]
            amount = float(order.get("totalPriceSet", {}).get("shopMoney", {}).get("amount", 0))
            currency = order.get("totalPriceSet", {}).get("shopMoney", {}).get("currencyCode")
            date = order["createdAt"][:10]  # Extract date part

            items_in_order = sum(item["node"]["quantity"] for item in order.get("lineItems", {}).get("edges", []))

            total_revenue += amount
            total_orders += 1
            total_items += items_in_order

            if date not in daily_breakdown:
                daily_breakdown[date] = {"date": date, "revenue": 0.0, "orders": 0, "items": 0}
            daily_breakdown[date]["revenue"] += amount
            daily_breakdown[date]["orders"] += 1
            daily_breakdown[date]["items"] += items_in_order

        # Sort daily breakdown by date
        sorted_daily = sorted(daily_breakdown.values(), key=lambda x: x["date"])
        for day in sorted_daily:
            day["revenue"] = round(day["revenue"], 2)

        summary = {
            "period": {"start": start_date, "end": end_date},
            "total_revenue": round(total_revenue, 2),
            "total_orders": total_orders,
            "total_items_sold": total_items,
            "average_order_value": round(total_revenue / total_orders, 2) if total_orders > 0 else 0,
            "currency": currency,
            "daily_breakdown": sorted_daily,
        }

        return json.dumps(summary, indent=2)

    def get_order_analytics(
        self,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Get comprehensive order analytics for a time period.

        Provides metrics like total orders, revenue, average order value,
        fulfillment rates, and more.

        Args:
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing various order metrics and statistics.
        """
        log_debug(f"Generating order analytics: created_after={created_after}, created_before={created_before}")

        query_parts = []
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts) if query_parts else ""
        query_param = f', query: "{query_filter}"' if query_filter else ""

        query = f"""
        query {{
            orders(first: 250{query_param}, sortKey: CREATED_AT) {{
                edges {{
                    node {{
                        displayFinancialStatus
                        displayFulfillmentStatus
                        totalPriceSet {{
                            shopMoney {{
                                amount
                                currencyCode
                            }}
                        }}
                        subtotalPriceSet {{
                            shopMoney {{
                                amount
                            }}
                        }}
                        totalShippingPriceSet {{
                            shopMoney {{
                                amount
                            }}
                        }}
                        totalTaxSet {{
                            shopMoney {{
                                amount
                            }}
                        }}
                        lineItems(first: 100) {{
                            edges {{
                                node {{
                                    quantity
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        orders = result.get("orders", {}).get("edges", [])

        if not orders:
            return json.dumps({"message": "No orders found in the specified period"}, indent=2)

        # Calculate metrics
        total_orders = len(orders)
        total_revenue = 0.0
        total_subtotal = 0.0
        total_shipping = 0.0
        total_tax = 0.0
        total_items = 0
        currency = None

        financial_status_counts: Counter = Counter()
        fulfillment_status_counts: Counter = Counter()
        order_values: List[float] = []

        for order_edge in orders:
            order = order_edge["node"]

            amount = float(order.get("totalPriceSet", {}).get("shopMoney", {}).get("amount", 0))
            currency = order.get("totalPriceSet", {}).get("shopMoney", {}).get("currencyCode")

            total_revenue += amount
            total_subtotal += float(order.get("subtotalPriceSet", {}).get("shopMoney", {}).get("amount", 0))
            total_shipping += float(order.get("totalShippingPriceSet", {}).get("shopMoney", {}).get("amount", 0))
            total_tax += float(order.get("totalTaxSet", {}).get("shopMoney", {}).get("amount", 0))

            order_values.append(amount)

            items = sum(item["node"]["quantity"] for item in order.get("lineItems", {}).get("edges", []))
            total_items += items

            financial_status_counts[order.get("displayFinancialStatus", "UNKNOWN")] += 1
            fulfillment_status_counts[order.get("displayFulfillmentStatus", "UNKNOWN")] += 1

        avg_order_value = total_revenue / total_orders if total_orders > 0 else 0
        avg_items_per_order = total_items / total_orders if total_orders > 0 else 0

        analytics = {
            "period": {
                "created_after": created_after,
                "created_before": created_before,
            },
            "total_orders": total_orders,
            "total_revenue": round(total_revenue, 2),
            "total_subtotal": round(total_subtotal, 2),
            "total_shipping": round(total_shipping, 2),
            "total_tax": round(total_tax, 2),
            "currency": currency,
            "average_order_value": round(avg_order_value, 2),
            "total_items_sold": total_items,
            "average_items_per_order": round(avg_items_per_order, 2),
            "min_order_value": round(min(order_values), 2) if order_values else 0,
            "max_order_value": round(max(order_values), 2) if order_values else 0,
            "financial_status_breakdown": dict(financial_status_counts),
            "fulfillment_status_breakdown": dict(fulfillment_status_counts),
        }

        return json.dumps(analytics, indent=2)

    def get_product_sales_breakdown(
        self,
        product_id: str,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Get detailed sales breakdown for a specific product.

        Args:
            product_id: The Shopify product ID (gid://shopify/Product/xxx or just the number).
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing product sales data including quantity, revenue, and variant breakdown.
        """
        log_debug(f"Fetching sales breakdown for product: {product_id}")

        # Normalize product ID
        if not product_id.startswith("gid://"):
            product_id = f"gid://shopify/Product/{product_id}"

        query_parts = ["financial_status:paid"]
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts)

        query = f"""
        query {{
            orders(first: 250, query: "{query_filter}") {{
                edges {{
                    node {{
                        createdAt
                        lineItems(first: 100) {{
                            edges {{
                                node {{
                                    title
                                    quantity
                                    variant {{
                                        id
                                        title
                                        sku
                                        product {{
                                            id
                                            title
                                        }}
                                    }}
                                    originalUnitPriceSet {{
                                        shopMoney {{
                                            amount
                                            currencyCode
                                        }}
                                    }}
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        # Filter for specific product and aggregate
        product_title = None
        total_quantity = 0
        total_revenue = 0.0
        order_count = 0
        currency = None
        variant_breakdown: Dict[str, Dict[str, Any]] = {}
        daily_sales: Dict[str, Dict[str, Any]] = {}

        for order_edge in result.get("orders", {}).get("edges", []):
            order = order_edge["node"]
            order_date = order["createdAt"][:10]
            found_in_order = False

            for item_edge in order.get("lineItems", {}).get("edges", []):
                item = item_edge["node"]
                variant = item.get("variant")
                if not variant or not variant.get("product"):
                    continue

                if variant["product"]["id"] == product_id:
                    product_title = variant["product"]["title"]
                    quantity = item["quantity"]
                    unit_price = float(item.get("originalUnitPriceSet", {}).get("shopMoney", {}).get("amount", 0))
                    currency = item.get("originalUnitPriceSet", {}).get("shopMoney", {}).get("currencyCode")
                    variant_id = variant["id"]
                    variant_title = variant.get("title", "Default")

                    total_quantity += quantity
                    total_revenue += quantity * unit_price
                    found_in_order = True

                    # Variant breakdown
                    if variant_id not in variant_breakdown:
                        variant_breakdown[variant_id] = {
                            "variant_id": variant_id,
                            "variant_title": variant_title,
                            "sku": variant.get("sku"),
                            "quantity": 0,
                            "revenue": 0.0,
                        }
                    variant_breakdown[variant_id]["quantity"] += quantity
                    variant_breakdown[variant_id]["revenue"] += quantity * unit_price

                    # Daily breakdown
                    if order_date not in daily_sales:
                        daily_sales[order_date] = {"date": order_date, "quantity": 0, "revenue": 0.0}
                    daily_sales[order_date]["quantity"] += quantity
                    daily_sales[order_date]["revenue"] += quantity * unit_price

            if found_in_order:
                order_count += 1

        if product_title is None:
            return json.dumps({"error": "Product not found in any orders during this period"}, indent=2)

        # Format variants and daily data
        for v in variant_breakdown.values():
            v["revenue"] = round(v["revenue"], 2)

        sorted_daily = sorted(daily_sales.values(), key=lambda x: x["date"])
        for d in sorted_daily:
            d["revenue"] = round(d["revenue"], 2)

        breakdown = {
            "product_id": product_id,
            "product_title": product_title,
            "period": {
                "created_after": created_after,
                "created_before": created_before,
            },
            "total_quantity_sold": total_quantity,
            "total_revenue": round(total_revenue, 2),
            "order_count": order_count,
            "currency": currency,
            "average_units_per_order": round(total_quantity / order_count, 2) if order_count > 0 else 0,
            "variant_breakdown": list(variant_breakdown.values()),
            "daily_sales": sorted_daily,
        }

        return json.dumps(breakdown, indent=2)

    def get_customer_order_history(
        self,
        customer_email: str,
        max_orders: int = 50,
    ) -> str:
        """Get order history for a specific customer.

        Args:
            customer_email: The customer's email address.
            max_orders: Maximum number of orders to return (default 50).

        Returns:
            JSON string containing customer info and their order history.
        """
        log_debug(f"Fetching order history for customer: {customer_email}")

        query = f"""
        query {{
            orders(first: {min(max_orders, 250)}, query: "email:{customer_email}", sortKey: CREATED_AT, reverse: true) {{
                edges {{
                    node {{
                        id
                        name
                        createdAt
                        displayFinancialStatus
                        displayFulfillmentStatus
                        totalPriceSet {{
                            shopMoney {{
                                amount
                                currencyCode
                            }}
                        }}
                        customer {{
                            id
                            firstName
                            lastName
                            numberOfOrders
                            amountSpent {{
                                amount
                                currencyCode
                            }}
                        }}
                        lineItems(first: 20) {{
                            edges {{
                                node {{
                                    title
                                    quantity
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        orders = result.get("orders", {}).get("edges", [])

        if not orders:
            return json.dumps({"message": f"No orders found for {customer_email}"}, indent=2)

        # Get customer info from first order
        first_customer = orders[0]["node"].get("customer", {}) if orders else {}

        customer_info = {
            "email": customer_email,
            "id": first_customer.get("id"),
            "name": f"{first_customer.get('firstName', '')} {first_customer.get('lastName', '')}".strip(),
            "total_orders": first_customer.get("numberOfOrders"),
            "total_spent": first_customer.get("amountSpent", {}).get("amount"),
            "currency": first_customer.get("amountSpent", {}).get("currencyCode"),
        }

        order_list = []
        for order_edge in orders:
            order = order_edge["node"]
            order_list.append(
                {
                    "id": order["id"],
                    "name": order["name"],
                    "created_at": order["createdAt"],
                    "financial_status": order.get("displayFinancialStatus"),
                    "fulfillment_status": order.get("displayFulfillmentStatus"),
                    "total": order.get("totalPriceSet", {}).get("shopMoney", {}).get("amount"),
                    "items": [
                        {"title": item["node"]["title"], "quantity": item["node"]["quantity"]}
                        for item in order.get("lineItems", {}).get("edges", [])
                    ],
                }
            )

        response = {
            "customer": customer_info,
            "orders": order_list,
        }

        return json.dumps(response, indent=2)

    def get_inventory_levels(
        self,
        max_results: int = 100,
    ) -> str:
        """Get current inventory levels for all products.

        Args:
            max_results: Maximum number of products to return (default 100).

        Returns:
            JSON string containing products with their inventory quantities.
        """
        log_debug(f"Fetching inventory levels: max_results={max_results}")

        query = f"""
        query {{
            products(first: {min(max_results, 250)}, query: "status:ACTIVE") {{
                edges {{
                    node {{
                        id
                        title
                        totalInventory
                        tracksInventory
                        variants(first: 50) {{
                            edges {{
                                node {{
                                    id
                                    title
                                    sku
                                    inventoryQuantity
                                    inventoryPolicy
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        products = []
        for edge in result.get("products", {}).get("edges", []):
            node = edge["node"]
            products.append(
                {
                    "id": node["id"],
                    "title": node["title"],
                    "total_inventory": node.get("totalInventory"),
                    "tracks_inventory": node.get("tracksInventory"),
                    "variants": [
                        {
                            "id": v["node"]["id"],
                            "title": v["node"]["title"],
                            "sku": v["node"].get("sku"),
                            "inventory_quantity": v["node"].get("inventoryQuantity"),
                            "inventory_policy": v["node"].get("inventoryPolicy"),
                        }
                        for v in node.get("variants", {}).get("edges", [])
                    ],
                }
            )

        return json.dumps(products, indent=2)

    def get_low_stock_products(
        self,
        threshold: int = 10,
        max_results: int = 50,
    ) -> str:
        """Get products that are running low on stock.

        Args:
            threshold: Inventory level below which a product is considered low stock (default 10).
            max_results: Maximum number of products to return (default 50).

        Returns:
            JSON string containing low stock products sorted by inventory level.
        """
        log_debug(f"Finding low stock products: threshold={threshold}")

        query = """
        query {{
            products(first: 250, query: "status:ACTIVE") {{
                edges {{
                    node {{
                        id
                        title
                        totalInventory
                        variants(first: 50) {{
                            edges {{
                                node {{
                                    id
                                    title
                                    sku
                                    inventoryQuantity
                                }}
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        low_stock = []
        for edge in result.get("products", {}).get("edges", []):
            node = edge["node"]
            total_inv = node.get("totalInventory", 0)

            if total_inv is not None and total_inv <= threshold:
                low_stock_variants = [
                    {
                        "id": v["node"]["id"],
                        "title": v["node"]["title"],
                        "sku": v["node"].get("sku"),
                        "inventory_quantity": v["node"].get("inventoryQuantity"),
                    }
                    for v in node.get("variants", {}).get("edges", [])
                    if v["node"].get("inventoryQuantity") is not None and v["node"]["inventoryQuantity"] <= threshold
                ]

                if low_stock_variants:
                    low_stock.append(
                        {
                            "id": node["id"],
                            "title": node["title"],
                            "total_inventory": total_inv,
                            "low_stock_variants": low_stock_variants,
                        }
                    )

        # Sort by total inventory (lowest first)
        low_stock.sort(key=lambda x: x["total_inventory"])

        return json.dumps(low_stock[:max_results], indent=2)

    def get_sales_trends(
        self,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
        compare_previous_period: bool = True,
    ) -> str:
        """Get sales trends comparing current period to previous period.

        Args:
            created_after: Start date for analysis period (YYYY-MM-DD format, optional).
            created_before: End date for analysis period (YYYY-MM-DD format, optional).
            compare_previous_period: Whether to compare with previous period of same length (default True).

        Returns:
            JSON string containing current period metrics and comparison with previous period.
        """
        log_debug(f"Calculating sales trends: created_after={created_after}, created_before={created_before}")

        # Use provided dates or default to last 30 days
        now = datetime.now()
        if created_before:
            current_end_dt = datetime.strptime(created_before, "%Y-%m-%d")
        else:
            current_end_dt = now
        current_end = current_end_dt.strftime("%Y-%m-%d")

        if created_after:
            current_start_dt = datetime.strptime(created_after, "%Y-%m-%d")
        else:
            current_start_dt = current_end_dt - timedelta(days=30)
        current_start = current_start_dt.strftime("%Y-%m-%d")

        # Calculate previous period of same length
        period_days = (current_end_dt - current_start_dt).days
        previous_end_dt = current_start_dt - timedelta(days=1)
        previous_start_dt = previous_end_dt - timedelta(days=period_days)
        previous_start = previous_start_dt.strftime("%Y-%m-%d")
        previous_end = previous_end_dt.strftime("%Y-%m-%d")

        def fetch_period_data(start: str, end: str) -> Dict[str, Any]:
            query = f"""
            query {{
                orders(first: 250, query: "created_at:>={start} AND created_at:<{end} AND financial_status:paid") {{
                    edges {{
                        node {{
                            totalPriceSet {{
                                shopMoney {{
                                    amount
                                    currencyCode
                                }}
                            }}
                            lineItems(first: 100) {{
                                edges {{
                                    node {{
                                        quantity
                                    }}
                                }}
                            }}
                        }}
                    }}
                }}
            }}
            """
            result = self._make_graphql_request(query)
            if "error" in result:
                return {"error": result["error"]}

            orders = result.get("orders", {}).get("edges", [])
            total_revenue = sum(
                float(o["node"].get("totalPriceSet", {}).get("shopMoney", {}).get("amount", 0)) for o in orders
            )
            total_items = sum(
                sum(item["node"]["quantity"] for item in o["node"].get("lineItems", {}).get("edges", []))
                for o in orders
            )
            currency = (
                orders[0]["node"].get("totalPriceSet", {}).get("shopMoney", {}).get("currencyCode") if orders else None
            )

            return {
                "total_orders": len(orders),
                "total_revenue": round(total_revenue, 2),
                "total_items_sold": total_items,
                "average_order_value": round(total_revenue / len(orders), 2) if orders else 0,
                "currency": currency,
            }

        current_data = fetch_period_data(current_start, current_end)
        if "error" in current_data:
            return json.dumps(current_data, indent=2)

        result = {
            "current_period": {
                "start": current_start,
                "end": current_end,
                **current_data,
            }
        }

        if compare_previous_period:
            previous_data = fetch_period_data(previous_start, previous_end)
            if "error" not in previous_data:
                result["previous_period"] = {
                    "start": previous_start,
                    "end": previous_end,
                    **previous_data,
                }

                # Calculate changes
                if previous_data["total_revenue"] > 0:
                    revenue_change = (
                        (current_data["total_revenue"] - previous_data["total_revenue"])
                        / previous_data["total_revenue"]
                    ) * 100
                else:
                    revenue_change = 100 if current_data["total_revenue"] > 0 else 0

                if previous_data["total_orders"] > 0:
                    orders_change = (
                        (current_data["total_orders"] - previous_data["total_orders"]) / previous_data["total_orders"]
                    ) * 100
                else:
                    orders_change = 100 if current_data["total_orders"] > 0 else 0

                result["comparison"] = {
                    "revenue_change_percent": round(revenue_change, 2),
                    "orders_change_percent": round(orders_change, 2),
                    "revenue_trend": "up" if revenue_change > 0 else ("down" if revenue_change < 0 else "flat"),
                    "orders_trend": "up" if orders_change > 0 else ("down" if orders_change < 0 else "flat"),
                }

        return json.dumps(result, indent=2)

    def get_average_order_value(
        self,
        group_by: str = "day",
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Get average order value over time.

        Args:
            group_by: How to group data - 'day', 'week', or 'month' (default 'day').
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing average order value trends.
        """
        log_debug(
            f"Calculating AOV: group_by={group_by}, created_after={created_after}, created_before={created_before}"
        )

        query_parts = ["financial_status:paid"]
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts)

        query = f"""
        query {{
            orders(first: 250, query: "{query_filter}", sortKey: CREATED_AT) {{
                edges {{
                    node {{
                        createdAt
                        totalPriceSet {{
                            shopMoney {{
                                amount
                                currencyCode
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        orders = result.get("orders", {}).get("edges", [])

        if not orders:
            return json.dumps({"message": "No orders found in the specified period"}, indent=2)

        # Group orders
        grouped: Dict[str, List[float]] = {}
        currency = None

        for order_edge in orders:
            order = order_edge["node"]
            created_at = order["createdAt"][:10]
            amount = float(order.get("totalPriceSet", {}).get("shopMoney", {}).get("amount", 0))
            currency = order.get("totalPriceSet", {}).get("shopMoney", {}).get("currencyCode")

            if group_by == "week":
                # Get ISO week
                date_obj = datetime.strptime(created_at, "%Y-%m-%d")
                key = f"{date_obj.isocalendar()[0]}-W{date_obj.isocalendar()[1]:02d}"
            elif group_by == "month":
                key = created_at[:7]  # YYYY-MM
            else:
                key = created_at

            if key not in grouped:
                grouped[key] = []
            grouped[key].append(amount)

        # Calculate averages
        aov_data = [
            {
                "period": key,
                "order_count": len(values),
                "total_revenue": round(sum(values), 2),
                "average_order_value": round(sum(values) / len(values), 2),
            }
            for key, values in sorted(grouped.items())
        ]

        overall_avg = sum(o["total_revenue"] for o in aov_data) / sum(o["order_count"] for o in aov_data)  # type: ignore

        response = {
            "period": {
                "created_after": created_after,
                "created_before": created_before,
            },
            "group_by": group_by,
            "overall_average_order_value": round(overall_avg, 2),
            "currency": currency,
            "breakdown": aov_data,
        }

        return json.dumps(response, indent=2)

    def get_repeat_customers(
        self,
        min_orders: int = 2,
        limit: int = 50,
        created_after: Optional[str] = None,
        created_before: Optional[str] = None,
    ) -> str:
        """Find customers who have made multiple purchases.

        Args:
            min_orders: Minimum number of orders to qualify as repeat customer (default 2).
            limit: Maximum number of customers to return (default 50).
            created_after: Only include orders created after this date (YYYY-MM-DD format, optional).
            created_before: Only include orders created before this date (YYYY-MM-DD format, optional).

        Returns:
            JSON string containing repeat customers with their order counts and total spend.
        """
        log_debug(
            f"Finding repeat customers: min_orders={min_orders}, created_after={created_after}, created_before={created_before}"
        )

        query_parts = ["financial_status:paid"]
        if created_after:
            query_parts.append(f"created_at:>={created_after}")
        if created_before:
            query_parts.append(f"created_at:<={created_before}")
        query_filter = " AND ".join(query_parts)

        query = f"""
        query {{
            orders(first: 250, query: "{query_filter}") {{
                edges {{
                    node {{
                        customer {{
                            id
                            email
                            firstName
                            lastName
                            numberOfOrders
                            amountSpent {{
                                amount
                                currencyCode
                            }}
                        }}
                        totalPriceSet {{
                            shopMoney {{
                                amount
                            }}
                        }}
                    }}
                }}
            }}
        }}
        """

        result = self._make_graphql_request(query)

        if "error" in result:
            return json.dumps(result, indent=2)

        # Aggregate by customer
        customer_data: Dict[str, Dict[str, Any]] = {}

        for order_edge in result.get("orders", {}).get("edges", []):
            customer = order_edge["node"].get("customer")
            if not customer or not customer.get("id"):
                continue

            customer_id = customer["id"]
            order_amount = float(order_edge["node"].get("totalPriceSet", {}).get("shopMoney", {}).get("amount", 0))

            if customer_id not in customer_data:
                customer_data[customer_id] = {
                    "id": customer_id,
                    "email": customer.get("email"),
                    "name": f"{customer.get('firstName', '')} {customer.get('lastName', '')}".strip(),
                    "total_orders_all_time": customer.get("numberOfOrders"),
                    "total_spent_all_time": customer.get("amountSpent", {}).get("amount"),
                    "currency": customer.get("amountSpent", {}).get("currencyCode"),
                    "orders_in_period": 0,
                    "spent_in_period": 0.0,
                }

            customer_data[customer_id]["orders_in_period"] += 1
            customer_data[customer_id]["spent_in_period"] += order_amount

        # Filter repeat customers and sort
        repeat_customers = [
            {**c, "spent_in_period": round(c["spent_in_period"], 2)}
            for c in customer_data.values()
            if c["orders_in_period"] >= min_orders
        ]

        repeat_customers.sort(key=lambda x: x["orders_in_period"], reverse=True)

        response = {
            "period": {
                "created_after": created_after,
                "created_before": created_before,
            },
            "min_orders_threshold": min_orders,
            "repeat_customer_count": len(repeat_customers),
            "customers": repeat_customers[:limit],
        }

        return json.dumps(response, indent=2)
```


## 16. Database & SQL (Detailed)

### SqlTools (`agno.tools.sql`)
Interface with SQL databases using SQLAlchemy. Supports various dialects (SQLite, MySQL, Oracle, etc.).

**Authentication**: Requires `db_url` or individual connection parameters (`user`, `password`, `host`, `port`, `dialect`).
**Dependencies**: `pip install sqlalchemy`

#### Parameters
- `db_url` (str): SQLAlchemy database URL.
- `tables` (Dict): Optional map of tables to expose.
- `enable_list_tables` (bool): Default True.

#### Source Code
```python
import json
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    from sqlalchemy import Engine, create_engine
    from sqlalchemy.inspection import inspect
    from sqlalchemy.orm import Session, sessionmaker
    from sqlalchemy.sql.expression import text
except ImportError:
    raise ImportError("`sqlalchemy` not installed")


class SQLTools(Toolkit):
    def __init__(
        self,
        db_url: Optional[str] = None,
        db_engine: Optional[Engine] = None,
        user: Optional[str] = None,
        password: Optional[str] = None,
        host: Optional[str] = None,
        port: Optional[int] = None,
        schema: Optional[str] = None,
        dialect: Optional[str] = None,
        tables: Optional[Dict[str, Any]] = None,
        enable_list_tables: bool = True,
        enable_describe_table: bool = True,
        enable_run_sql_query: bool = True,
        all: bool = False,
        **kwargs,
    ):
        # Get the database engine
        _engine: Optional[Engine] = db_engine
        if _engine is None and db_url is not None:
            _engine = create_engine(db_url)
        elif user and password and host and port and dialect:
            if schema is not None:
                _engine = create_engine(f"{dialect}://{user}:{password}@{host}:{port}/{schema}")
            else:
                _engine = create_engine(f"{dialect}://{user}:{password}@{host}:{port}")

        if _engine is None:
            raise ValueError("Could not build the database connection")

        # Database connection
        self.db_engine: Engine = _engine
        self.Session: sessionmaker[Session] = sessionmaker(bind=self.db_engine)

        self.schema = schema

        # Tables this toolkit can access
        self.tables: Optional[Dict[str, Any]] = tables

        tools: List[Any] = []
        if enable_list_tables or all:
            tools.append(self.list_tables)
        if enable_describe_table or all:
            tools.append(self.describe_table)
        if enable_run_sql_query or all:
            tools.append(self.run_sql_query)

        super().__init__(name="sql_tools", tools=tools, **kwargs)

    def list_tables(self) -> str:
        """Use this function to get a list of table names in the database.

        Returns:
            str: list of tables in the database.
        """
        if self.tables is not None:
            return json.dumps(self.tables)

        try:
            log_debug("listing tables in the database")
            inspector = inspect(self.db_engine)
            if self.schema:
                table_names = inspector.get_table_names(schema=self.schema)
            else:
                table_names = inspector.get_table_names()
            log_debug(f"table_names: {table_names}")
            return json.dumps(table_names)
        except Exception as e:
            logger.error(f"Error getting tables: {e}")
            return f"Error getting tables: {e}"

    def describe_table(self, table_name: str) -> str:
        """Use this function to describe a table.

        Args:
            table_name (str): The name of the table to get the schema for.

        Returns:
            str: schema of a table
        """

        try:
            log_debug(f"Describing table: {table_name}")
            inspector = inspect(self.db_engine)
            table_schema = inspector.get_columns(table_name, schema=self.schema)
            return json.dumps(
                [
                    {"name": column["name"], "type": str(column["type"]), "nullable": column["nullable"]}
                    for column in table_schema
                ]
            )
        except Exception as e:
            logger.error(f"Error getting table schema: {e}")
            return f"Error getting table schema: {e}"

    def run_sql_query(self, query: str, limit: Optional[int] = 10) -> str:
        """Use this function to run a SQL query and return the result.

        Args:
            query (str): The query to run.
            limit (int, optional): The number of rows to return. Defaults to 10. Use `None` to show all results.
        Returns:
            str: Result of the SQL query.
        Notes:
            - The result may be empty if the query does not return any data.
        """

        try:
            return json.dumps(self.run_sql(sql=query, limit=limit), default=str)
        except Exception as e:
            logger.error(f"Error running query: {e}")
            return f"Error running query: {e}"

    def run_sql(self, sql: str, limit: Optional[int] = None) -> List[dict]:
        """Internal function to run a sql query.

        Args:
            sql (str): The sql query to run.
            limit (int, optional): The number of rows to return. Defaults to None.

        Returns:
            List[dict]: The result of the query.
        """
        log_debug(f"Running sql |\n{sql}")

        with self.Session() as sess, sess.begin():
            result = sess.execute(text(sql))

            # Check if the operation has returned rows.
            try:
                if limit:
                    rows = result.fetchmany(limit)
                else:
                    rows = result.fetchall()
                return [row._asdict() for row in rows]
            except Exception as e:
                logger.error(f"Error while executing SQL: {e}")
                return []
```

---

### PostgresTools (`agno.tools.postgres`)
Optimized toolkit for PostgreSQL using `psycopg`. Supports schema inspection and CSV export.

**Authentication**: Connection parameters or existing `psycopg` connection.
**Dependencies**: `pip install psycopg-binary`

#### Parameters
- `db_name` (str): Database name.
- `user` (str): Username.
- `table_schema` (str): Default 'public'.

#### Source Code
```python
import csv
from typing import Any, Dict, List, Optional

try:
    import psycopg
    from psycopg import sql
    from psycopg.connection import Connection as PgConnection
    from psycopg.rows import DictRow, dict_row
except ImportError:
    raise ImportError("`psycopg` not installed. Please install using `pip install 'psycopg-binary'`.")

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_error


class PostgresTools(Toolkit):
    """
    A toolkit for interacting with PostgreSQL databases.

    Args:
        connection (Optional[PgConnection[DictRow]]): Existing database connection to reuse.
        db_name (Optional[str]): Database name to connect to.
        user (Optional[str]): Username for authentication.
        password (Optional[str]): Password for authentication.
        host (Optional[str]): PostgreSQL server hostname.
        port (Optional[int]): PostgreSQL server port number.
        table_schema (str): Default schema for table operations. Default is "public".
    """

    _requires_connect: bool = True

    def __init__(
        self,
        connection: Optional[PgConnection[DictRow]] = None,
        db_name: Optional[str] = None,
        user: Optional[str] = None,
        password: Optional[str] = None,
        host: Optional[str] = None,
        port: Optional[int] = None,
        table_schema: str = "public",
        **kwargs,
    ):
        self._connection: Optional[PgConnection[DictRow]] = connection
        self.db_name: Optional[str] = db_name
        self.user: Optional[str] = user
        self.password: Optional[str] = password
        self.host: Optional[str] = host
        self.port: Optional[int] = port
        self.table_schema: str = table_schema

        tools: List[Any] = [
            self.show_tables,
            self.describe_table,
            self.summarize_table,
            self.inspect_query,
            self.run_query,
            self.export_table_to_path,
        ]

        super().__init__(name="postgres_tools", tools=tools, **kwargs)

    def connect(self) -> PgConnection[DictRow]:
        """
        Establish a connection to the PostgreSQL database.

        Returns:
            The database connection object.
        """
        if self._connection is not None and not self._connection.closed:
            log_debug("Connection already established, reusing existing connection")
            return self._connection

        log_debug("Establishing new PostgreSQL connection.")
        connection_kwargs: Dict[str, Any] = {"row_factory": dict_row}
        if self.db_name:
            connection_kwargs["dbname"] = self.db_name
        if self.user:
            connection_kwargs["user"] = self.user
        if self.password:
            connection_kwargs["password"] = self.password
        if self.host:
            connection_kwargs["host"] = self.host
        if self.port:
            connection_kwargs["port"] = self.port

        connection_kwargs["options"] = f"-c search_path={self.table_schema}"

        self._connection = psycopg.connect(**connection_kwargs)
        self._connection.read_only = True
        return self._connection

    def close(self) -> None:
        """Closes the database connection if it's open."""
        if self._connection and not self._connection.closed:
            log_debug("Closing PostgreSQL connection.")
            self._connection.close()
            self._connection = None

    @property
    def is_connected(self) -> bool:
        """Check if a connection is currently established."""
        return self._connection is not None and not self._connection.closed

    def _ensure_connection(self) -> PgConnection[DictRow]:
        """
        Ensure a connection exists, creating one if necessary.

        Returns:
            The database connection object.
        """
        if not self.is_connected:
            return self.connect()
        return self._connection  # type: ignore

    def __enter__(self):
        return self.connect()

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.is_connected:
            self.close()

    def _execute_query(self, query: str, params: Optional[tuple] = None) -> str:
        try:
            connection = self._ensure_connection()
            with connection.cursor() as cursor:
                log_debug("Running PostgreSQL query")
                cursor.execute(query, params)

                if cursor.description is None:
                    return cursor.statusmessage or "Query executed successfully with no output."

                columns = [desc[0] for desc in cursor.description]
                rows = cursor.fetchall()

                if not rows:
                    return f"Query returned no results.\nColumns: {', '.join(columns)}"

                header = ",".join(columns)
                data_rows = [",".join(map(str, row.values())) for row in rows]
                return f"{header}\n" + "\n".join(data_rows)

        except psycopg.Error as e:
            log_error(f"Database error: {e}")
            if self._connection and not self._connection.closed:
                self._connection.rollback()
            return f"Error executing query: {e}"
        except Exception as e:
            log_error(f"An unexpected error occurred: {e}")
            return f"An unexpected error occurred: {e}"

    def show_tables(self) -> str:
        """Lists all tables in the configured schema."""

        stmt = "SELECT table_name FROM information_schema.tables WHERE table_schema = %s;"
        return self._execute_query(stmt, (self.table_schema,))

    def describe_table(self, table: str) -> str:
        """
        Provides the schema (column name, data type, is nullable) for a given table.

        Args:
            table: The name of the table to describe.

        Returns:
            A string describing the table's columns and data types.
        """
        stmt = """
            SELECT column_name, data_type, is_nullable
            FROM information_schema.columns
            WHERE table_schema = %s AND table_name = %s;
        """
        return self._execute_query(stmt, (self.table_schema, table))

    def summarize_table(self, table: str) -> str:
        """
        Computes and returns key summary statistics for a table's columns.

        Args:
            table: The name of the table to summarize.

        Returns:
            A string containing a summary of the table.
        """
        try:
            connection = self._ensure_connection()
            with connection.cursor() as cursor:
                # First, get column information using a parameterized query
                schema_query = """
                    SELECT column_name, data_type
                    FROM information_schema.columns
                    WHERE table_schema = %s AND table_name = %s;
                """
                cursor.execute(schema_query, (self.table_schema, table))
                columns = cursor.fetchall()
                if not columns:
                    return f"Error: Table '{table}' not found in schema '{self.table_schema}'."

                summary_parts = [f"Summary for table: {table}\n"]
                table_identifier = sql.Identifier(self.table_schema, table)

                for col in columns:
                    col_name, data_type = col["column_name"], col["data_type"]
                    col_identifier = sql.Identifier(col_name)

                    query = None
                    if any(
                        t in data_type for t in ["integer", "numeric", "real", "double precision", "bigint", "smallint"]
                    ):
                        query = sql.SQL("""
                            SELECT
                                COUNT(*) AS total_rows,
                                COUNT({col}) AS non_null_rows,
                                MIN({col}) AS min,
                                MAX({col}) AS max,
                                AVG({col}) AS average,
                                STDDEV({col}) AS std_deviation
                            FROM {tbl};
                        """).format(col=col_identifier, tbl=table_identifier)
                    elif any(t in data_type for t in ["char", "text", "uuid"]):
                        query = sql.SQL("""
                            SELECT
                                COUNT(*) AS total_rows,
                                COUNT({col}) AS non_null_rows,
                                COUNT(DISTINCT {col}) AS unique_values,
                                AVG(LENGTH({col}::text)) as avg_length
                            FROM {tbl};
                        """).format(col=col_identifier, tbl=table_identifier)

                    if query:
                        cursor.execute(query)
                        stats = cursor.fetchone()
                        summary_parts.append(f"\n--- Column: {col_name} (Type: {data_type}) ---")
                        if stats is not None:
                            for key, value in stats.items():
                                val_str = (
                                    f"{value:.2f}" if isinstance(value, float) and value is not None else str(value)
                                )
                                summary_parts.append(f"  {key}: {val_str}")
                        else:
                            summary_parts.append("  No statistics available")

                return "\n".join(summary_parts)

        except psycopg.Error as e:
            return f"Error summarizing table: {e}"

    def inspect_query(self, query: str) -> str:
        """
        Shows the execution plan for a SQL query (using EXPLAIN).

        :param query: The SQL query to inspect.
        :return: The query's execution plan.
        """
        return self._execute_query(f"EXPLAIN {query}")

    def export_table_to_path(self, table: str, path: str) -> str:
        """
        Exports a table's data to a local CSV file.

        :param table: The name of the table to export.
        :param path: The local file path to save the file.
        :return: A confirmation message with the file path.
        """
        log_debug(f"Exporting Table {table} as CSV to local path {path}")

        table_identifier = sql.Identifier(self.table_schema, table)
        stmt = sql.SQL("SELECT * FROM {tbl};").format(tbl=table_identifier)

        try:
            connection = self._ensure_connection()
            with connection.cursor() as cursor:
                cursor.execute(stmt)

                if cursor.description is None:
                    return f"Error: Query returned no description for table '{table}'."

                columns = [desc[0] for desc in cursor.description]

                with open(path, "w", newline="", encoding="utf-8") as f:
                    writer = csv.writer(f)
                    writer.writerow(columns)
                    writer.writerows(row.values() for row in cursor)

            return f"Successfully exported table '{table}' to '{path}'."
        except (psycopg.Error, IOError) as e:
            if self._connection and not self._connection.closed:
                self._connection.rollback()
            return f"Error exporting table: {e}"

    def run_query(self, query: str) -> str:
        """
        Runs a read-only SQL query and returns the result.

        :param query: The SQL query to run.
        :return: The query result as a formatted string.
        """
        return self._execute_query(query)
```

---

### PandasTools (`agno.tools.pandas`)
Create and operate on Pandas DataFrames.

**Authentication**: None.
**Dependencies**: `pip install pandas`

#### Parameters
- `enable_create_pandas_dataframe` (bool): Default True.
- `enable_run_dataframe_operation` (bool): Default True.

#### Source Code
```python
from typing import Any, Dict, List

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    import pandas as pd
except ImportError:
    raise ImportError("`pandas` not installed. Please install using `pip install pandas`.")


class PandasTools(Toolkit):
    def __init__(
        self,
        enable_create_pandas_dataframe: bool = True,
        enable_run_dataframe_operation: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.dataframes: Dict[str, pd.DataFrame] = {}

        tools: List[Any] = []
        if all or enable_create_pandas_dataframe:
            tools.append(self.create_pandas_dataframe)
        if all or enable_run_dataframe_operation:
            tools.append(self.run_dataframe_operation)

        super().__init__(name="pandas_tools", tools=tools, **kwargs)

    def create_pandas_dataframe(
        self, dataframe_name: str, create_using_function: str, function_parameters: Dict[str, Any]
    ) -> str:
        """Creates a pandas dataframe named `dataframe_name` by running a function `create_using_function` with the parameters `function_parameters`.
        Returns the created dataframe name as a string if successful, otherwise returns an error message.

        For Example:
        - To create a dataframe `csv_data` by reading a CSV file, use: {"dataframe_name": "csv_data", "create_using_function": "read_csv", "function_parameters": {"filepath_or_buffer": "data.csv"}}
        - To create a dataframe `csv_data` by reading a JSON file, use: {"dataframe_name": "json_data", "create_using_function": "read_json", "function_parameters": {"path_or_buf": "data.json"}}

        :param dataframe_name: The name of the dataframe to create.
        :param create_using_function: The function to use to create the dataframe.
        :param function_parameters: The parameters to pass to the function.
        :return: The name of the created dataframe if successful, otherwise an error message.
        """
        try:
            log_debug(f"Creating dataframe: {dataframe_name}")
            log_debug(f"Using function: {create_using_function}")
            log_debug(f"With parameters: {function_parameters}")

            if dataframe_name in self.dataframes:
                return f"Dataframe already exists: {dataframe_name}"

            # Create the dataframe
            dataframe = getattr(pd, create_using_function)(**function_parameters)
            if dataframe is None:
                return f"Error creating dataframe: {dataframe_name}"
            if not isinstance(dataframe, pd.DataFrame):
                return f"Error creating dataframe: {dataframe_name}"
            if dataframe.empty:
                return f"Dataframe is empty: {dataframe_name}"
            self.dataframes[dataframe_name] = dataframe
            log_debug(f"Created dataframe: {dataframe_name}")
            return dataframe_name
        except Exception as e:
            logger.error(f"Error creating dataframe: {e}")
            return f"Error creating dataframe: {e}"

    def run_dataframe_operation(self, dataframe_name: str, operation: str, operation_parameters: Dict[str, Any]) -> str:
        """Runs an operation `operation` on a dataframe `dataframe_name` with the parameters `operation_parameters`.
        Returns the result of the operation as a string if successful, otherwise returns an error message.

        For Example:
        - To get the first 5 rows of a dataframe `csv_data`, use: {"dataframe_name": "csv_data", "operation": "head", "operation_parameters": {"n": 5}}
        - To get the last 5 rows of a dataframe `csv_data`, use: {"dataframe_name": "csv_data", "operation": "tail", "operation_parameters": {"n": 5}}

        :param dataframe_name: The name of the dataframe to run the operation on.
        :param operation: The operation to run on the dataframe.
        :param operation_parameters: The parameters to pass to the operation.
        :return: The result of the operation if successful, otherwise an error message.
        """
        try:
            log_debug(f"Running operation: {operation}")
            log_debug(f"On dataframe: {dataframe_name}")
            log_debug(f"With parameters: {operation_parameters}")

            # Get the dataframe
            dataframe = self.dataframes.get(dataframe_name)

            # Run the operation
            result = getattr(dataframe, operation)(**operation_parameters)

            log_debug(f"Ran operation: {operation}")
            try:
                try:
                    return result.to_string()
                except AttributeError:
                    return str(result)
            except Exception:
                return "Operation ran successfully"
        except Exception as e:
            logger.error(f"Error running operation: {e}")
            return f"Error running operation: {e}"
```

---

### CsvTools (`agno.tools.csv_toolkit`)
Read, list, and query CSV files. Uses `duckdb` for SQL querying over CSVs.

**Authentication**: None.
**Dependencies**: `pip install duckdb` (for querying).

#### Parameters
- `csvs` (List[str/Path]): List of CSV file paths.
- `row_limit` (int): Optional limit for reading.

#### Source Code
```python
import csv
import json
from pathlib import Path
from typing import Any, Dict, List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info, logger


class CsvTools(Toolkit):
    def __init__(
        self,
        csvs: Optional[List[Union[str, Path]]] = None,
        row_limit: Optional[int] = None,
        duckdb_connection: Optional[Any] = None,
        duckdb_kwargs: Optional[Dict[str, Any]] = None,
        enable_read_csv_file: bool = True,
        enable_list_csv_files: bool = True,
        enable_get_columns: bool = True,
        enable_query_csv_file: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.csvs: List[Path] = []
        if csvs:
            for _csv in csvs:
                if isinstance(_csv, str):
                    self.csvs.append(Path(_csv))
                elif isinstance(_csv, Path):
                    self.csvs.append(_csv)
                else:
                    raise ValueError(f"Invalid csv file: {_csv}")
        self.row_limit = row_limit
        self.duckdb_connection: Optional[Any] = duckdb_connection
        self.duckdb_kwargs: Optional[Dict[str, Any]] = duckdb_kwargs

        tools: List[Any] = []
        if all or enable_read_csv_file:
            tools.append(self.read_csv_file)
        if all or enable_list_csv_files:
            tools.append(self.list_csv_files)
        if all or enable_get_columns:
            tools.append(self.get_columns)
        if all or enable_query_csv_file:
            try:
                import duckdb  # noqa: F401

                tools.append(self.query_csv_file)
            except ImportError:
                logger.warning("`duckdb` not installed. Query functionality disabled.")

        super().__init__(name="csv_tools", tools=tools, **kwargs)

    def list_csv_files(self) -> str:
        """Returns a list of available csv files

        Returns:
            str: List of available csv files
        """
        return json.dumps([_csv.stem for _csv in self.csvs])

    def read_csv_file(self, csv_name: str, row_limit: Optional[int] = None) -> str:
        """Use this function to read the contents of a csv file `name` without the extension.

        Args:
            csv_name (str): The name of the csv file to read without the extension.
            row_limit (Optional[int]): The number of rows to return. None returns all rows. Defaults to None.

        Returns:
            str: The contents of the csv file if successful, otherwise returns an error message.
        """
        try:
            if csv_name not in [_csv.stem for _csv in self.csvs]:
                return f"File: {csv_name} not found, please use one of {self.list_csv_files()}"

            log_info(f"Reading file: {csv_name}")
            file_path = [_csv for _csv in self.csvs if _csv.stem == csv_name][0]

            # Read the csv file
            csv_data = []
            _row_limit = row_limit or self.row_limit
            with open(str(file_path), newline="") as csvfile:
                reader = csv.DictReader(csvfile)
                if _row_limit is not None:
                    csv_data = [row for row in reader][:_row_limit]
                else:
                    csv_data = [row for row in reader]
            return json.dumps(csv_data)
        except Exception as e:
            logger.error(f"Error reading csv: {e}")
            return f"Error reading csv: {e}"

    def get_columns(self, csv_name: str) -> str:
        """Use this function to get the columns of the csv file `csv_name` without the extension.

        Args:
            csv_name (str): The name of the csv file to get the columns from without the extension.

        Returns:
            str: The columns of the csv file if successful, otherwise returns an error message.
        """
        try:
            if csv_name not in [_csv.stem for _csv in self.csvs]:
                return f"File: {csv_name} not found, please use one of {self.list_csv_files()}"

            log_info(f"Reading columns from file: {csv_name}")
            file_path = [_csv for _csv in self.csvs if _csv.stem == csv_name][0]

            # Get the columns of the csv file
            with open(str(file_path), newline="") as csvfile:
                reader = csv.DictReader(csvfile)
                columns = reader.fieldnames

            return json.dumps(columns)
        except Exception as e:
            logger.error(f"Error getting columns: {e}")
            return f"Error getting columns: {e}"

    def query_csv_file(self, csv_name: str, sql_query: str) -> str:
        """Use this function to run a SQL query on csv file `csv_name` without the extension.
        The Table name is the name of the csv file without the extension.
        The SQL Query should be a valid DuckDB SQL query.
        Always wrap column names with double quotes if they contain spaces or special characters
        Remember to escape the quotes in th e JSON string (use \")
        Use single quotes for string values

        Args:
            csv_name (str): The name of the csv file to query
            sql_query (str): The SQL Query to run on the csv file.

        Returns:
            str: The query results if successful, otherwise returns an error message.
        """
        try:
            import duckdb

            if csv_name not in [_csv.stem for _csv in self.csvs]:
                return f"File: {csv_name} not found, please use one of {self.list_csv_files()}"

            # Load the csv file into duckdb
            log_info(f"Loading csv file: {csv_name}")
            file_path = [_csv for _csv in self.csvs if _csv.stem == csv_name][0]

            # Create duckdb connection
            con = self.duckdb_connection
            if not self.duckdb_connection:
                con = duckdb.connect(**(self.duckdb_kwargs or {}))
            if con is None:
                logger.error("Error connecting to DuckDB")
                return "Error connecting to DuckDB, please check the connection."

            # Create a table from the csv file
            con.execute(
                f"CREATE TABLE {csv_name} AS SELECT * FROM read_csv('{file_path}', ignore_errors=false, auto_detect=true)"
            )

            # -*- Format the SQL Query
            # Remove backticks
            formatted_sql = sql_query.replace("`", "")
            # If there are multiple statements, only run the first one
            formatted_sql = formatted_sql.split(";")[0]
            # -*- Run the SQL Query
            log_info(f"Running query: {formatted_sql}")
            query_result = con.sql(formatted_sql)
            result_output = "No output"
            if query_result is not None:
                try:
                    results_as_python_objects = query_result.fetchall()
                    result_rows = []
                    for row in results_as_python_objects:
                        if len(row) == 1:
                            result_rows.append(str(row[0]))
                        else:
                            result_rows.append(",".join(str(x) for x in row))

                    result_data = "\n".join(result_rows)
                    result_output = ",".join(query_result.columns) + "\n" + result_data
                except AttributeError:
                    result_output = str(query_result)

            log_debug(f"Query result: {result_output}")
            return result_output
        except Exception as e:
            logger.error(f"Error querying csv: {e}")
            return f"Error querying csv: {e}"
```


## 17. Google Cloud (Detailed)

### GmailTools (`agno.tools.google.gmail`)
Read, compose, and manage Gmail messages and threads.

**Authentication**: OAuth2 (Client ID/Secret) or Service Account with domain-wide delegation.
**Environment Variables**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_PROJECT_ID`.
**Dependencies**: `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`

#### Parameters
- `creds_path` (str): Path to credentials JSON.
- `include_html` (bool): Default False.
- `max_batch_size` (int): Default 10.

#### Source Code
```python
"""
Gmail Toolkit for interacting with Gmail API

Required Environment Variables:
-----------------------------
- GOOGLE_CLIENT_ID: Google OAuth client ID
- GOOGLE_CLIENT_SECRET: Google OAuth client secret
- GOOGLE_PROJECT_ID: Google Cloud project ID
- GOOGLE_REDIRECT_URI: Google OAuth redirect URI (default: http://localhost)

How to Get These Credentials:
---------------------------
1. Go to Google Cloud Console (https://console.cloud.google.com)
2. Create a new project or select an existing one
3. Enable the Gmail API:
   - Go to "APIs & Services" > "Enable APIs and Services"
   - Search for "Gmail API"
   - Click "Enable"

4. Create OAuth 2.0 credentials:
   - Go to "APIs & Services" > "Credentials"
   - Click "Create Credentials" > "OAuth client ID"
   - Go through the OAuth consent screen setup
   - Give it a name and click "Create"
   - You'll receive:
     * Client ID (GOOGLE_CLIENT_ID)
     * Client Secret (GOOGLE_CLIENT_SECRET)
   - The Project ID (GOOGLE_PROJECT_ID) is visible in the project dropdown at the top of the page

5. Add auth redirect URI:
   - Go to https://console.cloud.google.com/auth/clients and add the redirect URI as http://127.0.0.1/

6. Set up environment variables:
   Create a .envrc file in your project root with:
   ```
   export GOOGLE_CLIENT_ID=your_client_id_here
   export GOOGLE_CLIENT_SECRET=your_client_secret_here
   export GOOGLE_PROJECT_ID=your_project_id_here
   export GOOGLE_REDIRECT_URI=http://127.0.0.1/  # Default value
   ```

Note: The first time you run the application, it will open a browser window for OAuth authentication.
A token.json file will be created to store the authentication credentials for future use.

Service Account Authentication (Alternative):
---------------------------------------------
For server/bot deployments where no browser is available, use a Google service account
with domain-wide delegation instead of OAuth.

1. Create a service account in Google Cloud Console > "IAM & Admin" > "Service Accounts"
2. Download the JSON key file
3. In Google Workspace Admin Console, go to Security > API Controls > Domain-wide Delegation
4. Add the service account's client_id with the Gmail scopes your agent needs
5. Set environment variables:
   ```
   export GOOGLE_SERVICE_ACCOUNT_FILE=/path/to/service-account-key.json
   export GOOGLE_DELEGATED_USER=user@yourdomain.com
   ```

When service_account_path (or GOOGLE_SERVICE_ACCOUNT_FILE) is set, OAuth is skipped entirely.
The delegated_user specifies which mailbox the service account will access.
"""

import base64
import json
import mimetypes
import re
import tempfile
import textwrap
from datetime import datetime, timedelta
from functools import wraps
from os import getenv
from pathlib import Path
from typing import Any, Callable, Dict, List, Optional, Tuple, Union

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_error

try:
    from email.mime.application import MIMEApplication
    from email.mime.multipart import MIMEMultipart
    from email.mime.text import MIMEText

    from google.auth.transport.requests import Request
    from google.oauth2.credentials import Credentials
    from google.oauth2.service_account import Credentials as ServiceAccountCredentials
    from google_auth_oauthlib.flow import InstalledAppFlow
    from googleapiclient.discovery import build
    from googleapiclient.errors import HttpError
except ImportError:
    raise ImportError(
        "Google client library for Python not found , install it using `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`"
    )


def authenticate(func):
    """Decorator to ensure authentication before executing a function."""

    @wraps(func)
    def wrapper(self, *args, **kwargs):
        try:
            if not self.creds or not self.creds.valid:
                self._auth()
            if not self.service:
                self.service = build("gmail", "v1", credentials=self.creds)
        except Exception as e:
            log_error(f"Gmail authentication failed: {e}")
            return json.dumps({"error": f"Gmail authentication failed: {e}"})
        return func(self, *args, **kwargs)

    return wrapper


def validate_email(email: str) -> bool:
    """Validate email format."""
    email = email.strip()
    pattern = r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
    return bool(re.match(pattern, email))


GMAIL_QUERY_INSTRUCTIONS = textwrap.dedent("""\
    You have access to Gmail tools for reading, composing, and organizing emails.

    ## Gmail Query Syntax
    Use these operators in search and context query parameters:
    - `from:user@example.com` / `to:user@example.com` — filter by sender/recipient
    - `subject:"meeting notes"` — filter by subject
    - `is:unread` / `is:starred` / `is:important` — filter by status
    - `has:attachment` — emails with attachments
    - `newer_than:7d` / `older_than:1m` — relative date (d=days, m=months, y=years)
    - `after:2024/01/01` / `before:2024/12/31` — absolute date range
    - `label:work` — filter by label
    - `from:me` — emails sent by the user
    - Combine with spaces (AND): `from:me newer_than:7d has:attachment`""")


class GmailTools(Toolkit):
    # Default scopes for Gmail API access
    DEFAULT_SCOPES = [
        "https://www.googleapis.com/auth/gmail.readonly",
        "https://www.googleapis.com/auth/gmail.modify",
        "https://www.googleapis.com/auth/gmail.compose",
    ]

    def __init__(
        self,
        creds: Optional[Union[Credentials, ServiceAccountCredentials]] = None,
        credentials_path: Optional[str] = None,
        token_path: Optional[str] = None,
        service_account_path: Optional[str] = None,
        delegated_user: Optional[str] = None,
        scopes: Optional[List[str]] = None,
        port: Optional[int] = None,
        login_hint: Optional[str] = None,
        include_html: bool = False,
        max_body_length: Optional[int] = None,
        attachment_dir: Optional[str] = None,
        # Reading
        get_latest_emails: bool = True,
        get_emails_from_user: bool = True,
        get_unread_emails: bool = True,
        get_starred_emails: bool = True,
        get_emails_by_context: bool = True,
        get_emails_by_date: bool = True,
        get_emails_by_thread: bool = True,
        search_emails: bool = True,
        # Management
        mark_email_as_read: bool = True,
        mark_email_as_unread: bool = True,
        star_email: bool = True,
        unstar_email: bool = True,
        archive_email: bool = False,
        # Composing
        create_draft_email: bool = True,
        send_email: bool = True,
        send_email_reply: bool = True,
        # Labels
        list_custom_labels: bool = True,
        apply_label: bool = True,
        remove_label: bool = True,
        delete_custom_label: bool = True,
        # Thread & message tools
        get_message: bool = True,
        get_thread: bool = True,
        search_threads: bool = True,
        modify_thread_labels: bool = False,
        trash_thread: bool = False,
        get_draft: bool = True,
        list_drafts: bool = True,
        send_draft: bool = False,
        update_draft: bool = True,
        list_labels: bool = False,
        modify_message_labels: bool = False,
        trash_message: bool = False,
        download_attachment: bool = False,
        max_batch_size: int = 10,
        instructions: Optional[str] = None,
        add_instructions: bool = True,
        **kwargs,
    ):
        """Initialize GmailTools and authenticate with Gmail API

        Args:
            creds (Optional[Union[Credentials, ServiceAccountCredentials]]): Pre-fetched credentials. Use this to skip a new auth flow. Defaults to None.
            credentials_path (Optional[str]): Path to credentials file. Defaults to None.
            token_path (Optional[str]): Path to token file. Defaults to None.
            service_account_path (Optional[str]): Path to a service account JSON key file. When provided (or GOOGLE_SERVICE_ACCOUNT_FILE env var is set), service account auth is used instead of OAuth. Requires delegated_user for Gmail.
            delegated_user (Optional[str]): Email of the user to impersonate via domain-wide delegation. Required when using service account auth. Can also be set via GOOGLE_DELEGATED_USER env var.
            scopes (Optional[List[str]]): Custom OAuth scopes. If None, uses DEFAULT_SCOPES.
            port (Optional[int]): Port to use for OAuth authentication. Defaults to None.
            login_hint (Optional[str]): Email to pre-select in the OAuth consent screen. Defaults to None.
            include_html (bool): If True, return raw HTML body instead of stripping tags. Defaults to False.
            max_body_length (Optional[int]): Truncate message bodies to this length. Defaults to None (no truncation).
            attachment_dir (Optional[str]): Directory to save downloaded attachments. Defaults to a temp directory.
            max_batch_size (int): Max items per Gmail API batch request. Maximum 100 (Gmail API limit). Defaults to 10.
            instructions (Optional[str]): Custom instructions for the toolkit. If None, uses DEFAULT_INSTRUCTIONS.
            add_instructions (bool): Whether to inject toolkit instructions into the agent system prompt. Defaults to True.
        """
        if instructions is None:
            self.instructions = GMAIL_QUERY_INSTRUCTIONS
        else:
            self.instructions = instructions

        self.creds = creds
        self.credentials_path = credentials_path
        self.token_path = token_path
        self.service_account_path = service_account_path
        self.delegated_user = delegated_user
        self.service = None
        self.scopes = scopes or self.DEFAULT_SCOPES
        self.port = port
        self.login_hint = login_hint
        self.include_html = include_html
        self.max_body_length = max_body_length
        self.attachment_dir = attachment_dir
        # Gmail API allows max 100 items per batch request
        self.max_batch_size = max(min(max_batch_size, 100), 1)
        self._temp_dir: Optional[tempfile.TemporaryDirectory] = None
        self._label_cache: Optional[Dict[str, str]] = None

        # When include_tools is specified, expose the full catalog and let
        # Toolkit's whitelist filter select the requested tools.
        if kwargs.get("include_tools"):
            tools = self._all_tools()
        else:
            tools: List[Any] = []  # type: ignore
            # Reading emails
            if get_latest_emails:
                tools.append(self.get_latest_emails)
            if get_emails_from_user:
                tools.append(self.get_emails_from_user)
            if get_unread_emails:
                tools.append(self.get_unread_emails)
            if get_starred_emails:
                tools.append(self.get_starred_emails)
            if get_emails_by_context:
                tools.append(self.get_emails_by_context)
            if get_emails_by_date:
                tools.append(self.get_emails_by_date)
            if get_emails_by_thread:
                tools.append(self.get_emails_by_thread)
            if search_emails:
                tools.append(self.search_emails)
            # Email management
            if mark_email_as_read:
                tools.append(self.mark_email_as_read)
            if mark_email_as_unread:
                tools.append(self.mark_email_as_unread)
            if star_email:
                tools.append(self.star_email)
            if unstar_email:
                tools.append(self.unstar_email)
            if archive_email:
                tools.append(self.archive_email)
            # Composing emails
            if create_draft_email:
                tools.append(self.create_draft_email)
            if send_email:
                tools.append(self.send_email)
            if send_email_reply:
                tools.append(self.send_email_reply)
            # Label management
            if list_custom_labels:
                tools.append(self.list_custom_labels)
            if apply_label:
                tools.append(self.apply_label)
            if remove_label:
                tools.append(self.remove_label)
            if delete_custom_label:
                tools.append(self.delete_custom_label)
            # New tools
            if get_message:
                tools.append(self.get_message)
            if get_thread:
                tools.append(self.get_thread)
            if search_threads:
                tools.append(self.search_threads)
            if modify_thread_labels:
                tools.append(self.modify_thread_labels)
            if trash_thread:
                tools.append(self.trash_thread)
            if get_draft:
                tools.append(self.get_draft)
            if list_drafts:
                tools.append(self.list_drafts)
            if send_draft:
                tools.append(self.send_draft)
            if update_draft:
                tools.append(self.update_draft)
            if list_labels:
                tools.append(self.list_labels)
            if modify_message_labels:
                tools.append(self.modify_message_labels)
            if trash_message:
                tools.append(self.trash_message)
            if download_attachment:
                tools.append(self.download_attachment)

        super().__init__(
            name="gmail_tools",
            tools=tools,
            instructions=self.instructions,
            add_instructions=add_instructions,
            **kwargs,
        )

        # Validate that required scopes are present for requested operations (only check registered functions)
        compose_tools = {"create_draft_email", "send_email", "send_email_reply", "send_draft", "update_draft"}
        if any(t in self.functions for t in compose_tools):
            if "https://www.googleapis.com/auth/gmail.compose" not in self.scopes:
                raise ValueError(
                    "The scope https://www.googleapis.com/auth/gmail.compose is required for email composition operations"
                )

        read_operations = {
            "get_latest_emails",
            "get_emails_from_user",
            "get_unread_emails",
            "get_starred_emails",
            "get_emails_by_context",
            "get_emails_by_date",
            "get_emails_by_thread",
            "search_emails",
            "list_custom_labels",
            "get_message",
            "get_thread",
            "search_threads",
            "list_labels",
            "get_draft",
            "list_drafts",
            "download_attachment",
        }
        if any(op in self.functions for op in read_operations):
            read_scope = "https://www.googleapis.com/auth/gmail.readonly"
            write_scope = "https://www.googleapis.com/auth/gmail.modify"
            if read_scope not in self.scopes and write_scope not in self.scopes:
                raise ValueError(f"The scope {read_scope} is required for email reading operations")

        modify_operations = {
            "mark_email_as_read",
            "mark_email_as_unread",
            "star_email",
            "unstar_email",
            "archive_email",
            "apply_label",
            "remove_label",
            "delete_custom_label",
            "modify_message_labels",
            "modify_thread_labels",
            "trash_message",
            "trash_thread",
        }
        if any(op in self.functions for op in modify_operations):
            modify_scope = "https://www.googleapis.com/auth/gmail.modify"
            if modify_scope not in self.scopes:
                raise ValueError(f"The scope {modify_scope} is required for email modification operations")

    def _all_tools(self) -> list:
        return [
            self.get_latest_emails,
            self.get_emails_from_user,
            self.get_unread_emails,
            self.get_starred_emails,
            self.get_emails_by_context,
            self.get_emails_by_date,
            self.get_emails_by_thread,
            self.search_emails,
            self.send_email,
            self.send_email_reply,
            self.create_draft_email,
            self.mark_email_as_read,
            self.mark_email_as_unread,
            self.star_email,
            self.unstar_email,
            self.archive_email,
            self.list_custom_labels,
            self.apply_label,
            self.remove_label,
            self.delete_custom_label,
            self.get_message,
            self.get_thread,
            self.search_threads,
            self.modify_thread_labels,
            self.trash_thread,
            self.get_draft,
            self.list_drafts,
            self.send_draft,
            self.update_draft,
            self.list_labels,
            self.modify_message_labels,
            self.trash_message,
            self.download_attachment,
        ]

    def _auth(self) -> None:
        """Authenticate with Gmail API using service account (priority) or OAuth flow."""
        if self.creds and self.creds.valid:
            return

        # Service account authentication takes priority over OAuth
        service_account_path = self.service_account_path or getenv("GOOGLE_SERVICE_ACCOUNT_FILE")
        if service_account_path:
            delegated_user = self.delegated_user or getenv("GOOGLE_DELEGATED_USER")
            if not delegated_user:
                raise ValueError(
                    "delegated_user is required for Gmail service account authentication. "
                    "Gmail service accounts must impersonate a user via domain-wide delegation. "
                    "Provide delegated_user as a parameter or set GOOGLE_DELEGATED_USER env var."
                )
            self.creds = ServiceAccountCredentials.from_service_account_file(
                service_account_path,
                scopes=self.scopes,
                subject=delegated_user,
            )
            # Eagerly fetch token so creds.valid=True and @authenticate won't re-enter _auth
            self.creds.refresh(Request())
            return

        # OAuth flow
        token_file = Path(self.token_path or "token.json")
        creds_file = Path(self.credentials_path or "credentials.json")

        if token_file.exists():
            try:
                self.creds = Credentials.from_authorized_user_file(str(token_file), self.scopes)
            except ValueError:
                # Token file missing refresh_token — fall through to re-auth
                self.creds = None

        if self.creds and self.creds.expired and self.creds.refresh_token:  # type: ignore[union-attr]
            try:
                self.creds.refresh(Request())
            except Exception:
                # Refresh token revoked or expired — fall through to re-auth
                self.creds = None

        if not self.creds or not self.creds.valid:
            client_config = {
                "installed": {
                    "client_id": getenv("GOOGLE_CLIENT_ID"),
                    "client_secret": getenv("GOOGLE_CLIENT_SECRET"),
                    "project_id": getenv("GOOGLE_PROJECT_ID"),
                    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                    "token_uri": "https://oauth2.googleapis.com/token",
                    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
                    "redirect_uris": [getenv("GOOGLE_REDIRECT_URI", "http://localhost")],
                }
            }
            if creds_file.exists():
                flow = InstalledAppFlow.from_client_secrets_file(str(creds_file), self.scopes)
            else:
                flow = InstalledAppFlow.from_client_config(client_config, self.scopes)
            # prompt=consent forces Google to return a refresh_token every time
            oauth_kwargs: Dict[str, Any] = {"prompt": "consent"}
            if self.login_hint:
                oauth_kwargs["login_hint"] = self.login_hint
            self.creds = flow.run_local_server(port=self.port, **oauth_kwargs)

        # Save the credentials for future use
        if self.creds and self.creds.valid:
            token_file.write_text(self.creds.to_json())  # type: ignore[union-attr]
            log_debug("Gmail credentials saved")

    def _format_emails(self, emails: List[dict]) -> str:
        """Format list of email dictionaries into a readable string"""
        if not emails:
            return "No emails found"

        formatted_emails = []
        for email in emails:
            formatted_email = (
                f"From: {email['from']}\n"
                f"Subject: {email['subject']}\n"
                f"Date: {email['date']}\n"
                f"Body: {email['body']}\n"
                f"Message ID: {email['id']}\n"
                f"In-Reply-To: {email['in-reply-to']}\n"
                f"References: {email['references']}\n"
                f"Thread ID: {email['thread_id']}\n"
                "----------------------------------------"
            )
            formatted_emails.append(formatted_email)

        return "\n\n".join(formatted_emails)

    @authenticate
    def get_latest_emails(self, count: int) -> str:
        """
        Get the latest X emails from the user's inbox.

        Args:
            count (int): Number of latest emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            results = self.service.users().messages().list(userId="me", maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving latest emails: {error}"
        except Exception as error:
            return f"Unexpected error retrieving latest emails: {type(error).__name__}: {error}"

    @authenticate
    def get_emails_from_user(self, user: str, count: int) -> str:
        """
        Get X number of emails from a specific user (name or email).

        Args:
            user (str): Name or email address of the sender
            count (int): Maximum number of emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            query = f"from:{user}" if "@" in user else f"from:{user}*"
            results = self.service.users().messages().list(userId="me", q=query, maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving emails from {user}: {error}"
        except Exception as error:
            return f"Unexpected error retrieving emails from {user}: {type(error).__name__}: {error}"

    @authenticate
    def get_unread_emails(self, count: int) -> str:
        """
        Get the X number of latest unread emails from the user's inbox.

        Args:
            count (int): Maximum number of unread emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            results = self.service.users().messages().list(userId="me", q="is:unread", maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving unread emails: {error}"
        except Exception as error:
            return f"Unexpected error retrieving unread emails: {type(error).__name__}: {error}"

    @authenticate
    def get_emails_by_thread(self, thread_id: str) -> str:
        """
        Retrieve all emails from a specific thread.

        Args:
            thread_id (str): The ID of the email thread.

        Returns:
            str: Formatted string containing email thread details.
        """
        try:
            thread = self.service.users().threads().get(userId="me", id=thread_id).execute()  # type: ignore
            messages = thread.get("messages", [])
            emails = self._get_message_details(messages)
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving emails from thread {thread_id}: {error}"
        except Exception as error:
            return f"Unexpected error retrieving emails from thread {thread_id}: {type(error).__name__}: {error}"

    @authenticate
    def get_starred_emails(self, count: int) -> str:
        """
        Get X number of starred emails from the user's inbox.

        Args:
            count (int): Maximum number of starred emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            results = self.service.users().messages().list(userId="me", q="is:starred", maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving starred emails: {error}"
        except Exception as error:
            return f"Unexpected error retrieving starred emails: {type(error).__name__}: {error}"

    @authenticate
    def get_emails_by_context(self, context: str, count: int) -> str:
        """
        Get X number of emails matching a specific context or search term.

        Args:
            context (str): Search term or context to match in emails
            count (int): Maximum number of emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            results = self.service.users().messages().list(userId="me", q=context, maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving emails by context '{context}': {error}"
        except Exception as error:
            return f"Unexpected error retrieving emails by context '{context}': {type(error).__name__}: {error}"

    @authenticate
    def get_emails_by_date(
        self, start_date: str, range_in_days: Optional[int] = None, num_emails: Optional[int] = 10
    ) -> str:
        """Get emails from a date or date range.

        Args:
            start_date (str): Start date in YYYY/MM/DD format (e.g. "2026/03/01").
            range_in_days (Optional[int]): Number of days to include in the range (default: None, meaning all emails after start_date).
            num_emails (Optional[int]): Maximum number of emails to retrieve (default: 10).

        Returns:
            str: Formatted string containing email details.
        """
        try:
            start_date_dt = datetime.strptime(start_date, "%Y/%m/%d")
            if range_in_days:
                end_date = start_date_dt + timedelta(days=range_in_days)
                query = f"after:{start_date} before:{end_date.strftime('%Y/%m/%d')}"
            else:
                query = f"after:{start_date}"

            results = self.service.users().messages().list(userId="me", q=query, maxResults=num_emails).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving emails by date: {error}"
        except Exception as error:
            return f"Unexpected error retrieving emails by date: {type(error).__name__}: {error}"

    @authenticate
    def create_draft_email(
        self,
        to: str,
        subject: str,
        body: str,
        cc: Optional[str] = None,
        bcc: Optional[str] = None,
        attachments: Optional[Union[str, List[str]]] = None,
        thread_id: Optional[str] = None,
        message_id: Optional[str] = None,
    ) -> str:
        """
        Create and save a draft email. To reply to a thread, provide thread_id and message_id.
        to, cc and bcc are comma separated string of email ids.

        Args:
            to (str): Comma separated string of recipient email addresses
            subject (str): Email subject
            body (str): Email body content
            cc (Optional[str]): Comma separated string of CC email addresses (optional)
            bcc (Optional[str]): Comma separated string of BCC email addresses (optional)
            attachments (Optional[Union[str, List[str]]]): File path(s) for attachments (optional)
            thread_id (Optional[str]): Thread ID to reply to (optional, makes this a reply draft)
            message_id (Optional[str]): Message ID being replied to (optional, used with thread_id)

        Returns:
            str: Stringified dictionary containing draft email details including id
        """
        try:
            self._validate_email_params(to, subject, body)

            # Process attachments
            attachment_files = []
            if attachments:
                if isinstance(attachments, str):
                    attachment_files = [attachments]
                else:
                    attachment_files = attachments

                for file_path in attachment_files:
                    if not Path(file_path).exists():
                        return f"Error: Attachment file not found: {file_path}"

            if thread_id and not subject.lower().startswith("re:"):
                subject = f"Re: {subject}"

            message = self._create_message(
                to.split(","),
                subject,
                body,
                cc.split(",") if cc else None,
                bcc=bcc.split(",") if bcc else None,
                thread_id=thread_id,
                message_id=message_id,
                attachments=attachment_files,
            )
            draft = {"message": message}
            draft = self.service.users().drafts().create(userId="me", body=draft).execute()  # type: ignore
            return json.dumps(draft)
        except HttpError as error:
            return f"HTTP Error creating draft: {error}"
        except Exception as error:
            return f"Error creating draft: {type(error).__name__}: {error}"

    @authenticate
    def send_email(
        self,
        to: str,
        subject: str,
        body: str,
        cc: Optional[str] = None,
        bcc: Optional[str] = None,
        attachments: Optional[Union[str, List[str]]] = None,
        thread_id: Optional[str] = None,
        message_id: Optional[str] = None,
    ) -> str:
        """
        Send an email immediately. To reply to a thread, provide thread_id and message_id.
        to, cc and bcc are comma separated string of email ids.

        Args:
            to (str): Comma separated string of recipient email addresses
            subject (str): Email subject
            body (str): Email body content
            cc (Optional[str]): Comma separated string of CC email addresses (optional)
            bcc (Optional[str]): Comma separated string of BCC email addresses (optional)
            attachments (Optional[Union[str, List[str]]]): File path(s) for attachments (optional)
            thread_id (Optional[str]): Thread ID to reply to (optional, makes this a reply)
            message_id (Optional[str]): Message ID being replied to (optional, used with thread_id)

        Returns:
            str: Stringified dictionary containing sent email details including id
        """
        try:
            self._validate_email_params(to, subject, body)

            # Process attachments
            attachment_files = []
            if attachments:
                if isinstance(attachments, str):
                    attachment_files = [attachments]
                else:
                    attachment_files = attachments

                for file_path in attachment_files:
                    if not Path(file_path).exists():
                        return f"Error: Attachment file not found: {file_path}"

            if thread_id and not subject.lower().startswith("re:"):
                subject = f"Re: {subject}"

            message = self._create_message(
                to.split(","),
                subject,
                body,
                cc.split(",") if cc else None,
                bcc=bcc.split(",") if bcc else None,
                thread_id=thread_id,
                message_id=message_id,
                attachments=attachment_files,
            )
            message = self.service.users().messages().send(userId="me", body=message).execute()  # type: ignore
            return json.dumps(message)
        except HttpError as error:
            return f"HTTP Error sending email: {error}"
        except Exception as error:
            return f"Error sending email: {type(error).__name__}: {error}"

    @authenticate
    def send_email_reply(
        self,
        thread_id: str,
        message_id: str,
        to: str,
        subject: str,
        body: str,
        cc: Optional[str] = None,
        attachments: Optional[Union[str, List[str]]] = None,
    ) -> str:
        """
        Respond to an existing email thread.

        Args:
            thread_id (str): The ID of the email thread to reply to.
            message_id (str): The ID of the email being replied to.
            to (str): Comma-separated recipient email addresses.
            subject (str): Email subject (prefixed with "Re:" if not already).
            body (str): Email body content.
            cc (Optional[str]): Comma-separated CC email addresses (optional).
            attachments (Optional[Union[str, List[str]]]): File path(s) for attachments (optional)

        Returns:
            str: Stringified dictionary containing sent email details including id.
        """
        try:
            self._validate_email_params(to, subject, body)

            if not subject.lower().startswith("re:"):
                subject = f"Re: {subject}"

            # Process attachments
            attachment_files = []
            if attachments:
                if isinstance(attachments, str):
                    attachment_files = [attachments]
                else:
                    attachment_files = attachments

                for file_path in attachment_files:
                    if not Path(file_path).exists():
                        return f"Error: Attachment file not found: {file_path}"

            message = self._create_message(
                to.split(","),
                subject,
                body,
                cc=cc.split(",") if cc else None,
                thread_id=thread_id,
                message_id=message_id,
                attachments=attachment_files,
            )
            message = self.service.users().messages().send(userId="me", body=message).execute()  # type: ignore
            return json.dumps(message)
        except HttpError as error:
            return f"HTTP Error sending reply: {error}"
        except Exception as error:
            return f"Error sending reply: {type(error).__name__}: {error}"

    @authenticate
    def search_emails(self, query: str, count: int) -> str:
        """
        Get X number of emails based on a given natural text query.
        Searches in to, from, cc, subject and email body contents.

        Args:
            query (str): Natural language query to search for
            count (int): Number of emails to retrieve

        Returns:
            str: Formatted string containing email details
        """
        try:
            results = self.service.users().messages().list(userId="me", q=query, maxResults=count).execute()  # type: ignore
            emails = self._get_message_details(results.get("messages", []))
            return self._format_emails(emails)
        except HttpError as error:
            return f"Error retrieving emails with query '{query}': {error}"
        except Exception as error:
            return f"Unexpected error retrieving emails with query '{query}': {type(error).__name__}: {error}"

    @authenticate
    def mark_email_as_read(self, message_id: str) -> str:
        """
        Mark a specific email as read by removing the 'UNREAD' label.
        This is crucial for long polling scenarios to prevent processing the same email multiple times.

        Args:
            message_id (str): The ID of the message to mark as read

        Returns:
            str: Success message or error description
        """
        try:
            # Remove the UNREAD label to mark the email as read
            modify_request = {"removeLabelIds": ["UNREAD"]}

            self.service.users().messages().modify(userId="me", id=message_id, body=modify_request).execute()  # type: ignore

            return f"Successfully marked email {message_id} as read. Labels removed: UNREAD"

        except HttpError as error:
            return f"HTTP Error marking email {message_id} as read: {error}"
        except Exception as error:
            return f"Error marking email {message_id} as read: {type(error).__name__}: {error}"

    @authenticate
    def mark_email_as_unread(self, message_id: str) -> str:
        """
        Mark a specific email as unread by adding the 'UNREAD' label.
        This is useful for flagging emails that need attention or re-processing.

        Args:
            message_id (str): The ID of the message to mark as unread

        Returns:
            str: Success message or error description
        """
        try:
            # Add the UNREAD label to mark the email as unread
            modify_request = {"addLabelIds": ["UNREAD"]}

            self.service.users().messages().modify(userId="me", id=message_id, body=modify_request).execute()  # type: ignore

            return f"Successfully marked email {message_id} as unread. Labels added: UNREAD"

        except HttpError as error:
            return f"HTTP Error marking email {message_id} as unread: {error}"
        except Exception as error:
            return f"Error marking email {message_id} as unread: {type(error).__name__}: {error}"

    @authenticate
    def star_email(self, message_id: str) -> str:
        """Add a star to an email message.

        Args:
            message_id (str): The ID of the message to star.

        Returns:
            str: Success message or error description.
        """
        try:
            modify_request = {"addLabelIds": ["STARRED"]}
            self.service.users().messages().modify(userId="me", id=message_id, body=modify_request).execute()  # type: ignore
            return f"Successfully starred email {message_id}"
        except HttpError as error:
            return f"HTTP Error starring email {message_id}: {error}"
        except Exception as error:
            return f"Error starring email {message_id}: {type(error).__name__}: {error}"

    @authenticate
    def unstar_email(self, message_id: str) -> str:
        """Remove the star from an email message.

        Args:
            message_id (str): The ID of the message to unstar.

        Returns:
            str: Success message or error description.
        """
        try:
            modify_request = {"removeLabelIds": ["STARRED"]}
            self.service.users().messages().modify(userId="me", id=message_id, body=modify_request).execute()  # type: ignore
            return f"Successfully unstarred email {message_id}"
        except HttpError as error:
            return f"HTTP Error unstarring email {message_id}: {error}"
        except Exception as error:
            return f"Error unstarring email {message_id}: {type(error).__name__}: {error}"

    @authenticate
    def archive_email(self, message_id: str) -> str:
        """Archive an email by removing it from the inbox. The email is NOT deleted and can still be found via search.

        Args:
            message_id (str): The ID of the message to archive.

        Returns:
            str: Success message or error description.
        """
        try:
            modify_request = {"removeLabelIds": ["INBOX"]}
            self.service.users().messages().modify(userId="me", id=message_id, body=modify_request).execute()  # type: ignore
            return f"Successfully archived email {message_id}"
        except HttpError as error:
            return f"HTTP Error archiving email {message_id}: {error}"
        except Exception as error:
            return f"Error archiving email {message_id}: {type(error).__name__}: {error}"

    @authenticate
    def list_custom_labels(self) -> str:
        """
        List only user-created custom labels (filters out system labels) in a numbered format.

        Returns:
            str: A numbered list of custom labels only
        """
        try:
            results = self.service.users().labels().list(userId="me").execute()  # type: ignore
            labels = results.get("labels", [])

            # Filter out only user-created labels
            custom_labels = [label["name"] for label in labels if label.get("type") == "user"]

            if not custom_labels:
                return "No custom labels found.\nCreate labels using apply_label function!"

            # Create numbered list
            numbered_labels = [f"{i}. {name}" for i, name in enumerate(custom_labels, 1)]
            return f"Your Custom Labels ({len(custom_labels)} total):\n\n" + "\n".join(numbered_labels)

        except HttpError as e:
            return f"Error fetching labels: {e}"
        except Exception as e:
            return f"Unexpected error: {type(e).__name__}: {e}"

    @authenticate
    def apply_label(self, context: str, label_name: str, count: int = 10) -> str:
        """
        Find emails matching a context (search query) and apply a label, creating it if necessary.

        Args:
            context (str): Gmail search query (e.g., 'is:unread category:promotions')
            label_name (str): Name of the label to apply
            count (int): Maximum number of emails to process
        Returns:
            str: Summary of labeled emails
        """
        try:
            # Fetch messages matching context
            results = self.service.users().messages().list(userId="me", q=context, maxResults=count).execute()  # type: ignore

            messages = results.get("messages", [])
            if not messages:
                return f"No emails found matching: '{context}'"

            # Populate cache if needed, then check existence
            self._resolve_label_ids([label_name])
            label_id = self._label_cache.get(label_name.lower())  # type: ignore[union-attr]
            if not label_id:
                label = (
                    self.service.users()  # type: ignore
                    .labels()
                    .create(
                        userId="me",
                        body={"name": label_name, "labelListVisibility": "labelShow", "messageListVisibility": "show"},
                    )
                    .execute()
                )
                label_id = label["id"]
                # New label created — invalidate cache
                self._label_cache = None

            # Apply label to all matching messages
            for msg in messages:
                self.service.users().messages().modify(  # type: ignore
                    userId="me", id=msg["id"], body={"addLabelIds": [label_id]}
                ).execute()  # type: ignore

            return f"Applied label '{label_name}' to {len(messages)} emails matching '{context}'."

        except HttpError as e:
            return f"Error applying label '{label_name}': {e}"
        except Exception as e:
            return f"Unexpected error: {type(e).__name__}: {e}"

    @authenticate
    def remove_label(self, context: str, label_name: str, count: int = 10) -> str:
        """
        Remove a label from emails matching a context (search query).

        Args:
            context (str): Gmail search query (e.g., 'is:unread category:promotions')
            label_name (str): Name of the label to remove
            count (int): Maximum number of emails to process
        Returns:
            str: Summary of emails with label removed
        """
        try:
            # Populate cache if needed, then check existence
            self._resolve_label_ids([label_name])
            label_id = self._label_cache.get(label_name.lower())  # type: ignore[union-attr]
            if not label_id:
                return f"Label '{label_name}' not found."

            # Fetch messages matching context that have this label
            results = (
                self.service.users()  # type: ignore
                .messages()
                .list(userId="me", q=f"{context} label:{label_name}", maxResults=count)
                .execute()
            )

            messages = results.get("messages", [])
            if not messages:
                return f"No emails found matching: '{context}' with label '{label_name}'"

            # Remove label from all matching messages
            removed_count = 0
            for msg in messages:
                self.service.users().messages().modify(  # type: ignore
                    userId="me", id=msg["id"], body={"removeLabelIds": [label_id]}
                ).execute()  # type: ignore
                removed_count += 1

            return f"Removed label '{label_name}' from {removed_count} emails matching '{context}'."

        except HttpError as e:
            return f"Error removing label '{label_name}': {e}"
        except Exception as e:
            return f"Unexpected error: {type(e).__name__}: {e}"

    @authenticate
    def delete_custom_label(self, label_name: str, confirm: bool = False) -> str:
        """
        Delete a custom label (with safety confirmation).

        Args:
            label_name (str): Name of the label to delete
            confirm (bool): Must be True to actually delete the label
        Returns:
            str: Confirmation message or warning
        """
        if not confirm:
            return f"LABEL DELETION REQUIRES CONFIRMATION. This will permanently delete the label '{label_name}' from all emails. Set confirm=True to proceed."

        try:
            # Get all labels to find the target label
            labels = self.service.users().labels().list(userId="me").execute().get("labels", [])  # type: ignore
            target_label = None

            for label in labels:
                if label["name"].lower() == label_name.lower():
                    target_label = label
                    break

            if not target_label:
                return f"Label '{label_name}' not found."

            # Check if it's a system label using the type field
            if target_label.get("type") != "user":
                return f"Cannot delete system label '{label_name}'. Only user-created labels can be deleted."

            # Delete the label
            self.service.users().labels().delete(userId="me", id=target_label["id"]).execute()  # type: ignore
            self._label_cache = None

            return f"Successfully deleted label '{label_name}'. This label has been removed from all emails."

        except HttpError as e:
            return f"Error deleting label '{label_name}': {e}"
        except Exception as e:
            return f"Unexpected error: {type(e).__name__}: {e}"

    def _validate_email_params(self, to: str, subject: str, body: str) -> None:
        """Validate email parameters."""
        if not to:
            raise ValueError("Recipient email cannot be empty")

        # Validate each email in the comma-separated list
        for email in to.split(","):
            if not validate_email(email.strip()):
                raise ValueError(f"Invalid recipient email format: {email}")

        if not subject or not subject.strip():
            raise ValueError("Subject cannot be empty")

        if body is None:
            raise ValueError("Email body cannot be None")

    def _create_message(
        self,
        to: List[str],
        subject: str,
        body: str,
        cc: Optional[List[str]] = None,
        bcc: Optional[List[str]] = None,
        thread_id: Optional[str] = None,
        message_id: Optional[str] = None,
        attachments: Optional[List[str]] = None,
    ) -> dict:
        """Build a base64-encoded MIME message dict ready for the Gmail API send/draft endpoints."""
        body = body.replace("\\n", "\n").replace("\n", "<br>")

        # Create multipart message if attachments exist, otherwise simple text message
        message: Union[MIMEMultipart, MIMEText]
        if attachments:
            message = MIMEMultipart()

            # Add the text body
            text_part = MIMEText(body, "html")
            message.attach(text_part)

            # Add attachments
            for file_path in attachments:
                file_path_obj = Path(file_path)
                if not file_path_obj.exists():
                    continue

                # Guess the content type based on the file extension
                content_type, encoding = mimetypes.guess_type(file_path)
                if content_type is None or encoding is not None:
                    content_type = "application/octet-stream"

                _, sub_type = content_type.split("/", 1)

                # Read file and create attachment
                with open(file_path, "rb") as file:
                    attachment_data = file.read()

                attachment = MIMEApplication(attachment_data, _subtype=sub_type)
                attachment.add_header("Content-Disposition", "attachment", filename=file_path_obj.name)
                message.attach(attachment)
        else:
            message = MIMEText(body, "html")

        # Set headers
        message["to"] = ", ".join(to)
        message["from"] = "me"
        message["subject"] = subject

        if cc:
            message["Cc"] = ", ".join(cc)
        if bcc:
            message["Bcc"] = ", ".join(bcc)

        # Add reply headers if this is a response
        if thread_id and message_id:
            message["In-Reply-To"] = message_id
            message["References"] = message_id

        raw_message = base64.urlsafe_b64encode(message.as_bytes()).decode()
        email_data = {"raw": raw_message}

        if thread_id:
            email_data["threadId"] = thread_id

        return email_data

    def _get_message_details(self, messages: List[dict]) -> List[dict]:
        """Get details for list of messages"""
        details = []
        for msg in messages:
            msg_data = self.service.users().messages().get(userId="me", id=msg["id"], format="full").execute()  # type: ignore
            details.append(
                {
                    "id": msg_data["id"],
                    "thread_id": msg_data.get("threadId"),
                    "subject": next(
                        (header["value"] for header in msg_data["payload"]["headers"] if header["name"] == "Subject"),
                        None,
                    ),
                    "from": next(
                        (header["value"] for header in msg_data["payload"]["headers"] if header["name"] == "From"), None
                    ),
                    "date": next(
                        (header["value"] for header in msg_data["payload"]["headers"] if header["name"] == "Date"), None
                    ),
                    "in-reply-to": next(
                        (
                            header["value"]
                            for header in msg_data["payload"]["headers"]
                            if header["name"] == "In-Reply-To"
                        ),
                        None,
                    ),
                    "references": next(
                        (
                            header["value"]
                            for header in msg_data["payload"]["headers"]
                            if header["name"] == "References"
                        ),
                        None,
                    ),
                    "body": self._get_message_body(msg_data),
                }
            )
        return details

    def _get_message_body(self, msg_data: dict) -> str:
        """Extract message body from message data"""
        body = ""
        attachments = []
        try:
            if "parts" in msg_data["payload"]:
                for part in msg_data["payload"]["parts"]:
                    if part["mimeType"] == "text/plain":
                        if "data" in part["body"]:
                            body = base64.urlsafe_b64decode(part["body"]["data"]).decode()
                    elif "filename" in part:
                        attachments.append(part["filename"])
            elif "body" in msg_data["payload"] and "data" in msg_data["payload"]["body"]:
                body = base64.urlsafe_b64decode(msg_data["payload"]["body"]["data"]).decode()
        except Exception:
            return "Unable to decode message body"

        if attachments:
            return f"{body}\n\nAttachments: {', '.join(attachments)}"
        return body

    def _decode_body_data(self, data: str) -> str:
        """Decode a base64url-encoded Gmail body part to text."""
        try:
            raw_bytes = base64.urlsafe_b64decode(data)
        except Exception:
            return ""
        try:
            return raw_bytes.decode("utf-8")
        except UnicodeDecodeError:
            return raw_bytes.decode("latin-1")

    def _resolve_label_ids(self, label_names: List[str]) -> List[str]:
        """Convert label names to Gmail label IDs. Falls back to raw name for system labels like INBOX."""
        if self._label_cache is None:
            labels = self.service.users().labels().list(userId="me").execute().get("labels", [])  # type: ignore
            self._label_cache = {lbl["name"].lower(): lbl["id"] for lbl in labels}
        return [self._label_cache.get(name.lower(), name) for name in label_names]

    def _batch_get(
        self,
        ids: List[str],
        request_builder: Callable,
    ) -> List[Dict]:
        """Execute multiple Gmail API requests in batched HTTP calls.

        Args:
            ids: List of resource IDs to fetch.
            request_builder: Callable that takes an ID and returns a Gmail API request object.

        Returns:
            List of API response dicts (or error dicts for failed requests).
        """
        service = self.service
        results: List[Dict] = []

        def callback(request_id: str, response: Any, exception: Any) -> None:
            if exception:
                log_error(f"Batch request {request_id} failed: {exception}")
                results.append({"id": request_id, "error": str(exception)})
            else:
                results.append(response)

        for i in range(0, len(ids), self.max_batch_size):
            chunk = ids[i : i + self.max_batch_size]
            batch = service.new_batch_http_request(callback=callback)  # type: ignore
            for item_id in chunk:
                batch.add(request_builder(item_id), request_id=item_id)
            batch.execute()
        return results

    def _download_attachment_file(self, message_id: str, attachment_id: str, filename: str) -> str:
        """Download a Gmail attachment to disk and return the local file path."""
        service = self.service
        att = (
            service.users().messages().attachments().get(userId="me", messageId=message_id, id=attachment_id).execute()  # type: ignore
        )
        data = base64.urlsafe_b64decode(att["data"])
        if self.attachment_dir:
            dest_dir = Path(self.attachment_dir)
        else:
            if self._temp_dir is None:
                self._temp_dir = tempfile.TemporaryDirectory()
            dest_dir = Path(self._temp_dir.name)
        dest_dir.mkdir(parents=True, exist_ok=True)
        # Strip directory components to prevent path traversal from sender-controlled filenames
        safe_name = Path(filename).name or "attachment"
        file_path = dest_dir / safe_name
        file_path.write_bytes(data)
        log_debug(f"Downloaded attachment: {file_path}")
        return str(file_path)

    def _format_message(self, msg_data: Dict, include_body: bool = True) -> Dict[str, Any]:
        """Convert a raw Gmail API message into a clean dict with id, subject, from, to, date, body, and attachments."""
        raw_headers = msg_data.get("payload", {}).get("headers", [])
        headers = {h["name"].lower(): h["value"] for h in raw_headers}
        result: Dict[str, Any] = {
            "id": msg_data["id"],
            "threadId": msg_data.get("threadId"),
            "labelIds": msg_data.get("labelIds", []),
            "snippet": msg_data.get("snippet", ""),
            "subject": headers.get("subject"),
            "from": headers.get("from"),
            "to": headers.get("to"),
            "date": headers.get("date"),
            "cc": headers.get("cc"),
            "inReplyTo": headers.get("in-reply-to"),
            "references": headers.get("references"),
        }
        if include_body and "payload" in msg_data:
            body, attachments = self._extract_body(msg_data["payload"])
            result["body"] = body
            if attachments:
                result["attachments"] = attachments
        return result

    def _extract_body(self, payload: Dict) -> Tuple[str, List[Dict]]:
        """Extract text body and attachment metadata from a Gmail message payload.

        Handles multipart MIME, prefers text/plain over text/html, strips HTML tags
        unless include_html is set, and truncates to max_body_length.

        Returns:
            Tuple of (body_text, list_of_attachment_dicts).
        """
        mime_type = payload.get("mimeType", "")

        if "parts" not in payload:
            data = payload.get("body", {}).get("data")
            if not data:
                return "", []
            text = self._decode_body_data(data)
            if "html" in mime_type and not self.include_html:
                text = re.sub(r"<[^>]+>", "", text)
                text = "\n".join(s for s in (line.strip() for line in text.splitlines()) if s)
            if self.max_body_length and len(text) > self.max_body_length:
                text = text[: self.max_body_length] + "... [truncated]"
            return text, []

        plain_parts: List[str] = []
        html_parts: List[str] = []
        attachments: List[Dict] = []

        for part in payload["parts"]:
            part_mime = part.get("mimeType", "")

            if part_mime.startswith("multipart/"):
                sub_body, sub_att = self._extract_body(part)
                if sub_body:
                    plain_parts.append(sub_body)
                attachments.extend(sub_att)
                continue

            part_body = part.get("body", {})
            if part_body.get("attachmentId"):
                attachments.append(
                    {
                        "filename": part.get("filename", "unknown"),
                        "mimeType": part_mime,
                        "size": part_body.get("size", 0),
                        "attachmentId": part_body["attachmentId"],
                    }
                )
                continue

            data = part_body.get("data")
            if not data:
                continue

            if part_mime == "text/plain":
                plain_parts.append(self._decode_body_data(data))
            elif part_mime == "text/html":
                html_parts.append(self._decode_body_data(data))

        if plain_parts:
            body = "\n".join(plain_parts)
        elif html_parts:
            html = "\n".join(html_parts)
            if self.include_html:
                body = html
            else:
                body = re.sub(r"<[^>]+>", "", html)
                body = "\n".join(s for s in (line.strip() for line in body.splitlines()) if s)
        else:
            body = ""

        if self.max_body_length and len(body) > self.max_body_length:
            body = body[: self.max_body_length] + "... [truncated]"
        return body, attachments

    # -- New tools ----------------------------------------------------------------

    @authenticate
    def get_message(self, message_id: str, download_attachments: bool = False) -> str:
        """Get a single email message by its ID with full content including headers, body, and attachment metadata.

        Args:
            message_id: The Gmail message ID.
            download_attachments: If True, download attachments to disk and include file paths in the response.

        Returns:
            JSON string with message content including id, threadId, subject, from, to, date, body, and attachments.
        """
        try:
            service = self.service
            raw = service.users().messages().get(userId="me", id=message_id, format="full").execute()  # type: ignore
            result = self._format_message(raw)
            if download_attachments and result.get("attachments"):
                for att in result["attachments"]:
                    if att.get("attachmentId"):
                        att["localPath"] = self._download_attachment_file(
                            message_id, att["attachmentId"], att["filename"]
                        )
            return json.dumps(result)
        except HttpError as e:
            log_error(f"Failed to get message {message_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def get_thread(self, thread_id: str) -> str:
        """Get all messages in a Gmail thread as structured JSON.

        Args:
            thread_id: The Gmail thread ID.

        Returns:
            JSON string with thread metadata and all messages in chronological order.
        """
        try:
            service = self.service
            thread = service.users().threads().get(userId="me", id=thread_id).execute()  # type: ignore
            messages = [self._format_message(m) for m in thread.get("messages", [])]
            return json.dumps(
                {
                    "threadId": thread_id,
                    "messages": messages,
                    "messageCount": len(messages),
                }
            )
        except HttpError as e:
            log_error(f"Failed to get thread {thread_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def search_threads(self, query: str, count: int = 10) -> str:
        """Search Gmail threads using Gmail query syntax. Returns thread IDs and snippets, not full message content.

        Args:
            query: Gmail search query string. Supports all Gmail operators like from:, to:, subject:, is:unread, etc.
            count: Maximum number of threads to return (default 10, max 500).

        Returns:
            JSON string with list of matching threads with their IDs and snippets.
        """
        try:
            service = self.service
            max_results = min(count, 500)
            results = service.users().threads().list(userId="me", q=query, maxResults=max_results).execute()  # type: ignore
            threads = results.get("threads", [])
            return json.dumps(
                {
                    "threads": threads,
                    "resultSizeEstimate": results.get("resultSizeEstimate", len(threads)),
                }
            )
        except HttpError as e:
            log_error(f"Thread search failed: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def modify_thread_labels(
        self,
        thread_id: str,
        add_labels: Optional[str] = None,
        remove_labels: Optional[str] = None,
    ) -> str:
        """Add or remove labels from an entire thread (all messages in the conversation).

        Args:
            thread_id: The Gmail thread ID.
            add_labels: Comma-separated label names to add (e.g. 'STARRED,Important').
            remove_labels: Comma-separated label names to remove (e.g. 'UNREAD,INBOX').

        Returns:
            JSON string with updated thread label state.
        """
        try:
            body: Dict[str, List[str]] = {}
            if add_labels:
                names = [n.strip() for n in add_labels.split(",") if n.strip()]
                body["addLabelIds"] = self._resolve_label_ids(names)
            if remove_labels:
                names = [n.strip() for n in remove_labels.split(",") if n.strip()]
                body["removeLabelIds"] = self._resolve_label_ids(names)

            if not body:
                return json.dumps({"error": "Must specify add_labels or remove_labels"})

            service = self.service
            result = service.users().threads().modify(userId="me", id=thread_id, body=body).execute()  # type: ignore
            return json.dumps({"threadId": result["id"], "labelIds": result.get("labelIds", [])})
        except HttpError as e:
            log_error(f"Failed to modify labels on thread {thread_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def trash_thread(self, thread_id: str) -> str:
        """Move an entire thread to the trash. All messages in the conversation will be trashed.

        Args:
            thread_id: The Gmail thread ID to trash.

        Returns:
            JSON string confirming the thread was trashed.
        """
        try:
            service = self.service
            service.users().threads().trash(userId="me", id=thread_id).execute()  # type: ignore
            return json.dumps({"threadId": thread_id, "action": "trashed"})
        except HttpError as e:
            log_error(f"Failed to trash thread {thread_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def get_draft(self, draft_id: str) -> str:
        """Get a draft email by its ID with full message content.

        Args:
            draft_id: The Gmail draft ID.

        Returns:
            JSON string with draft ID and full message details.
        """
        try:
            service = self.service
            draft = service.users().drafts().get(userId="me", id=draft_id, format="full").execute()  # type: ignore
            msg_data = draft.get("message", {})
            return json.dumps(
                {
                    "draftId": draft["id"],
                    "message": self._format_message(msg_data) if msg_data else {},
                }
            )
        except HttpError as e:
            log_error(f"Failed to get draft {draft_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def list_drafts(self, count: int = 10) -> str:
        """List draft emails in the mailbox.

        Args:
            count: Maximum number of drafts to return (default 10, max 500).

        Returns:
            JSON string with list of draft IDs and estimated total count.
        """
        try:
            service = self.service
            max_results = min(count, 500)
            results = service.users().drafts().list(userId="me", maxResults=max_results).execute()  # type: ignore
            drafts = results.get("drafts", [])
            return json.dumps({"drafts": drafts, "resultSizeEstimate": results.get("resultSizeEstimate", len(drafts))})
        except HttpError as e:
            log_error(f"Failed to list drafts: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def send_draft(self, draft_id: str) -> str:
        """Send an existing draft email.

        Args:
            draft_id: The Gmail draft ID to send.

        Returns:
            JSON string with sent message ID, thread ID, and labels.
        """
        try:
            service = self.service
            result = service.users().drafts().send(userId="me", body={"id": draft_id}).execute()  # type: ignore
            return json.dumps(
                {
                    "id": result.get("id"),
                    "threadId": result.get("threadId"),
                    "labelIds": result.get("labelIds", []),
                }
            )
        except HttpError as e:
            log_error(f"Failed to send draft {draft_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def update_draft(
        self,
        draft_id: str,
        to: str,
        subject: str,
        body: str,
        cc: Optional[str] = None,
        bcc: Optional[str] = None,
        attachments: Optional[Union[str, List[str]]] = None,
        thread_id: Optional[str] = None,
        message_id: Optional[str] = None,
    ) -> str:
        """Replace the content of an existing draft email.

        Args:
            draft_id: The Gmail draft ID to update.
            to: Comma-separated recipient email addresses.
            subject: Email subject line.
            body: Email body content.
            cc: Comma-separated CC email addresses (optional).
            bcc: Comma-separated BCC email addresses (optional).
            attachments: File path(s) for attachments (optional).
            thread_id: Thread ID for reply drafts (optional).
            message_id: Message ID being replied to, used with thread_id (optional).

        Returns:
            JSON string with updated draft ID.
        """
        try:
            self._validate_email_params(to, subject, body)
            attachment_files: List[str] = []
            if attachments:
                attachment_files = [attachments] if isinstance(attachments, str) else list(attachments)
                for fp in attachment_files:
                    p = Path(fp)
                    try:
                        size = p.stat().st_size
                    except FileNotFoundError:
                        raise ValueError(f"Attachment file not found: {fp}")
                    if size > 25 * 1024 * 1024:
                        raise ValueError(f"Attachment exceeds 25MB limit: {fp}")

            if thread_id and not subject.lower().startswith("re:"):
                subject = f"Re: {subject}"

            mime = self._create_message(
                to=[t.strip() for t in to.split(",")],
                subject=subject,
                body=body,
                cc=[c.strip() for c in cc.split(",")] if cc else None,
                bcc=[b.strip() for b in bcc.split(",")] if bcc else None,
                thread_id=thread_id,
                message_id=message_id,
                attachments=attachment_files or None,
            )

            service = self.service
            result = service.users().drafts().update(userId="me", id=draft_id, body={"message": mime}).execute()  # type: ignore
            return json.dumps({"draftId": result["id"]})
        except HttpError as e:
            log_error(f"Failed to update draft {draft_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Failed to update draft {draft_id}: {e}")
            return json.dumps({"error": f"{type(e).__name__}: {e}"})

    @authenticate
    def list_labels(self) -> str:
        """List all Gmail labels (system and custom) with message and thread counts.

        Returns:
            JSON string with list of label objects including name, type, and message/thread counts.
        """
        try:
            service = self.service
            results = service.users().labels().list(userId="me").execute()  # type: ignore
            labels = results.get("labels", [])

            label_ids = [lbl["id"] for lbl in labels]
            detailed_labels = self._batch_get(label_ids, lambda lid: service.users().labels().get(userId="me", id=lid))  # type: ignore
            formatted = []
            for detail in detailed_labels:
                if "error" in detail:
                    continue
                formatted.append(
                    {
                        "id": detail["id"],
                        "name": detail["name"],
                        "type": detail.get("type", "system"),
                        "messagesTotal": detail.get("messagesTotal", 0),
                        "messagesUnread": detail.get("messagesUnread", 0),
                        "threadsTotal": detail.get("threadsTotal", 0),
                        "threadsUnread": detail.get("threadsUnread", 0),
                    }
                )
            return json.dumps({"labels": formatted, "count": len(formatted)})
        except HttpError as e:
            log_error(f"Failed to list labels: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def modify_message_labels(
        self,
        message_id: str,
        add_labels: Optional[str] = None,
        remove_labels: Optional[str] = None,
    ) -> str:
        """Add or remove labels from a single message. Use for marking read/unread, starring, categorizing, etc.
        For example: add_labels="STARRED" or remove_labels="UNREAD" to mark as read.

        Args:
            message_id: The Gmail message ID.
            add_labels: Comma-separated label names to add (e.g. 'STARRED,Work').
            remove_labels: Comma-separated label names to remove (e.g. 'UNREAD,INBOX').

        Returns:
            JSON string with updated message label state.
        """
        try:
            body: Dict[str, List[str]] = {}
            if add_labels:
                names = [n.strip() for n in add_labels.split(",") if n.strip()]
                body["addLabelIds"] = self._resolve_label_ids(names)
            if remove_labels:
                names = [n.strip() for n in remove_labels.split(",") if n.strip()]
                body["removeLabelIds"] = self._resolve_label_ids(names)

            if not body:
                return json.dumps({"error": "Must specify add_labels or remove_labels"})

            service = self.service
            result = service.users().messages().modify(userId="me", id=message_id, body=body).execute()  # type: ignore
            return json.dumps({"id": result["id"], "labelIds": result.get("labelIds", [])})
        except HttpError as e:
            log_error(f"Failed to modify labels on message {message_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def trash_message(self, message_id: str, undo: bool = False) -> str:
        """Move a message to trash, or restore it from trash with undo=True.

        Args:
            message_id: The Gmail message ID.
            undo: If True, restore the message from trash instead of trashing it.

        Returns:
            JSON string confirming the action.
        """
        try:
            service = self.service
            if undo:
                service.users().messages().untrash(userId="me", id=message_id).execute()  # type: ignore
                return json.dumps({"id": message_id, "action": "untrashed"})
            else:
                service.users().messages().trash(userId="me", id=message_id).execute()  # type: ignore
                return json.dumps({"id": message_id, "action": "trashed"})
        except HttpError as e:
            action_name = "untrash" if undo else "trash"
            log_error(f"Failed to {action_name} message {message_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})

    @authenticate
    def download_attachment(self, message_id: str, attachment_id: str, filename: str) -> str:
        """Download an email attachment to disk. Use get_message first to find attachment IDs.

        Args:
            message_id: The Gmail message ID containing the attachment.
            attachment_id: The attachment ID from the message's attachment metadata.
            filename: The filename to save the attachment as.

        Returns:
            JSON string with the local file path where the attachment was saved.
        """
        try:
            local_path = self._download_attachment_file(message_id, attachment_id, filename)
            return json.dumps({"localPath": local_path, "filename": filename, "messageId": message_id})
        except HttpError as e:
            log_error(f"Failed to download attachment from {message_id}: {e}")
            return json.dumps({"error": f"Gmail API error: {e}"})
        except Exception as e:
            log_error(f"Unexpected error: {e}")
            return json.dumps({"error": f"Unexpected error: {type(e).__name__}: {e}"})
```

---

### GoogleDriveTools (`agno.tools.google.drive`)
Manage files and folders on Google Drive.

**Authentication**: OAuth2.
**Environment Variables**: `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_PROJECT_ID`, `GOOGLE_CLOUD_QUOTA_PROJECT_ID`.
**Dependencies**: `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`

#### Parameters
- `auth_port` (int): Default 5050.
- `list_files` (bool): Default True.

#### Source Code
```python
"""
Google Drive API integration for file management and sharing.


This module provides functions to interact with Google Drive, including listing,
uploading, and downloading files.
It uses the Google Drive API and handles authentication via OAuth2.

Required Environment Variables:
-----------------------------
- GOOGLE_CLIENT_ID: Google OAuth client ID
- GOOGLE_CLIENT_SECRET: Google OAuth client secret
- GOOGLE_PROJECT_ID: Google Cloud project ID
- GOOGLE_REDIRECT_URI: Google OAuth redirect URI (default: http://localhost)
- GOOGLE_CLOUD_QUOTA_PROJECT_ID: Google Cloud quota project ID

How to Get These Credentials:
---------------------------
1. Go to Google Cloud Console (https://console.cloud.google.com)
2. Create a new project or select an existing one
3. Enable the Google Drive API:
   - Go to "APIs & Services" > "Enable APIs and Services"
   - Search for "Google Drive API"
   - Click "Enable"

4. Create OAuth 2.0 credentials:
   - Go to "APIs & Services" > "Credentials"
   - Click "Create Credentials" > "OAuth client ID"
   - Enable the OAuth Consent Screen if you haven't already
   - After enabling the Consent Screen, click on "Create Credentials" > "OAuth client ID"
   - You'll receive:
     * Client ID (GOOGLE_CLIENT_ID)
     * Client Secret (GOOGLE_CLIENT_SECRET)
   - The Project ID (GOOGLE_PROJECT_ID) is visible in the project dropdown at the top of the page

5. Add auth redirect URI:
   - Go to https://console.cloud.google.com/auth/clients
   - Add `http://localhost:5050` as a recognized redirect URI OR with http://localhost:{PORT_NUMBER}


6. Set up environment variables:
   Create a .envrc file in your project root with:
   ``
   export GOOGLE_CLIENT_ID=your_client_id_here
   export GOOGLE_CLIENT_SECRET=your_client_secret_here
   export GOOGLE_PROJECT_ID=your_project_id_here
   export GOOGLE_REDIRECT_URI=http://localhost/  # Default value
   export GOOGLE_AUTHENTICATION_PORT=5050  # Port for OAuth redirect
   export GOOGLE_CLOUD_QUOTA_PROJECT_ID=your_quota_project_id_here
   ``

---

Remember to install the dependencies using `pip install google google-auth-oauthlib`

Important Points to Note :
1. The first time you run the application, it will open a browser window for OAuth authentication.
2. A token.json file will be created to store the authentication credentials for future use.

You can customize the authentication port by setting the `GOOGLE_AUTHENTICATION_PORT` environment variable.
This will be used in the `run_local_server` method for OAuth authentication.

"""

import mimetypes
from functools import wraps
from os import getenv
from pathlib import Path
from typing import Any, List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import log_error

try:
    from google.auth.transport.requests import Request
    from google.oauth2.credentials import Credentials
    from google_auth_oauthlib.flow import InstalledAppFlow
    from googleapiclient.discovery import Resource, build
    from googleapiclient.http import MediaFileUpload, MediaIoBaseDownload
except ImportError:
    raise ImportError(
        "Google client library for Python not found , install it using `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`"
    )


def authenticate(func):
    """Decorator to ensure authentication before executing a function."""

    @wraps(func)
    def wrapper(self, *args, **kwargs):
        if not self.creds or not self.creds.valid:
            self._auth()
        if not self.service:
            # Set quota project on credentials if available
            creds_to_use = self.creds
            if hasattr(self, "quota_project_id") and self.quota_project_id:
                creds_to_use = self.creds.with_quota_project(self.quota_project_id)
            self.service = build("drive", "v3", credentials=creds_to_use)
        return func(self, *args, **kwargs)

    return wrapper


class GoogleDriveTools(Toolkit):
    # Default scopes for Google Drive API access
    DEFAULT_SCOPES = ["https://www.googleapis.com/auth/drive.file", "https://www.googleapis.com/auth/drive.readonly"]

    def __init__(
        self,
        auth_port: Optional[int] = 5050,
        creds: Optional[Credentials] = None,
        scopes: Optional[List[str]] = None,
        creds_path: Optional[str] = None,
        token_path: Optional[str] = None,
        quota_project_id: Optional[str] = None,
        list_files: bool = True,
        upload_file: bool = False,
        download_file: bool = False,
        **kwargs,
    ):
        self.creds: Optional[Credentials] = creds
        self.service: Optional[Resource] = None
        self.credentials_path = creds_path
        self.token_path = token_path
        self.scopes = scopes or []
        self.scopes.extend(self.DEFAULT_SCOPES)

        self.quota_project_id = quota_project_id or getenv("GOOGLE_CLOUD_QUOTA_PROJECT_ID")
        if not self.quota_project_id:
            raise ValueError("GOOGLE_CLOUD_QUOTA_PROJECT_ID is not set")

        self.auth_port: int = int(getenv("GOOGLE_AUTH_PORT", str(auth_port)))
        if not self.auth_port:
            raise ValueError("GOOGLE_AUTH_PORT is not set")

        tools: List[Any] = []
        if list_files:
            tools.append(self.list_files)
        if upload_file:
            tools.append(self.upload_file)
        if download_file:
            tools.append(self.download_file)
        super().__init__(name="google_drive_tools", tools=tools, **kwargs)
        if not self.scopes:
            # Add read permission by default
            self.scopes.append(self.DEFAULT_SCOPES[1])  # 'drive.readonly'
            # Add write permission if allow_update is True
            if getattr(self, "allow_update", False):
                self.scopes.append(self.DEFAULT_SCOPES[0])  # 'drive.file'

    def _auth(self):
        """
        Authenticate and set up the Google Drive API client.
        This method checks if credentials are valid and refreshes or requests them if needed.
        """
        if self.creds and self.creds.valid:
            # Already authenticated
            return

        token_file = Path(self.token_path or "token.json")
        creds_file = Path(self.credentials_path or "credentials.json")

        if token_file.exists():
            self.creds = Credentials.from_authorized_user_file(str(token_file), self.scopes)
        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:
                self.creds.refresh(Request())
            else:
                client_config = {
                    "installed": {
                        "client_id": getenv("GOOGLE_CLIENT_ID"),
                        "client_secret": getenv("GOOGLE_CLIENT_SECRET"),
                        "project_id": getenv("GOOGLE_PROJECT_ID"),
                        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                        "token_uri": "https://oauth2.googleapis.com/token",
                        "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
                        "redirect_uris": [getenv("GOOGLE_REDIRECT_URI", "http://localhost")],
                    }
                }
                # File based authentication
                if creds_file.exists():
                    flow = InstalledAppFlow.from_client_secrets_file(str(creds_file), self.scopes)
                else:
                    flow = InstalledAppFlow.from_client_config(client_config, self.scopes)
                # Opens up a browser window for OAuth authentication
                self.creds = flow.run_local_server(port=self.auth_port)  # type: ignore

            token_file.write_text(self.creds.to_json()) if self.creds else None

    @authenticate
    def list_files(self, query: Optional[str] = None, page_size: int = 10) -> List[dict]:
        """
        List files in your Google Drive.

        Args:
            query (Optional[str]): Optional search query to filter files (see Google Drive API docs).
            page_size (int): Maximum number of files to return.

        Returns:
            List[dict]: List of file metadata dictionaries.
        """
        if not self.service:
            raise ValueError("Google Drive service is not initialized. Please authenticate first.")
        try:
            results = (
                self.service.files()  # type: ignore
                .list(q=query, pageSize=page_size, fields="nextPageToken, files(id, name, mimeType, modifiedTime)")
                .execute()
            )
            items = results.get("files", [])
            return items
        except Exception as error:
            log_error(f"Could not list files: {error}")
            return []

    @authenticate
    def upload_file(self, file_path: Union[str, Path], mime_type: Optional[str] = None) -> Optional[dict]:
        """
        Upload a file to your Google Drive.

        Args:
            file_path (Union[str, Path]): Path to the file you want to upload.
            mime_type (Optional[str]): MIME type of the file. If not provided, it will be guessed.

        Returns:
            Optional[dict]: Metadata of the uploaded file, or None if upload failed.
        """
        if not self.service:
            raise ValueError("Google Drive service is not initialized. Please authenticate first.")
        file_path = Path(file_path)
        if not file_path.exists() or not file_path.is_file():
            raise ValueError(f"The file '{file_path}' does not exist or is not a file.")
        if mime_type is None:
            mime_type, _ = mimetypes.guess_type(file_path.as_posix())
            if mime_type is None:
                mime_type = "application/octet-stream"  # Default MIME type

        file_metadata = {"name": file_path.name}
        media = MediaFileUpload(file_path.as_posix(), mimetype=mime_type)

        try:
            uploaded_file = (
                self.service.files()  # type: ignore
                .create(body=file_metadata, media_body=media, fields="id, name, mimeType, modifiedTime")
                .execute()
            )
            return uploaded_file
        except Exception as error:
            log_error(f"Could not upload file '{file_path}': {error}")
            return None

    @authenticate
    def download_file(self, file_id: str, dest_path: Union[str, Path]) -> Optional[Path]:
        """
        Download a file from your Google Drive.

        Args:
            file_id (str): The ID of the file you want to download.
            dest_path (Union[str, Path]): Where to save the downloaded file.

        Returns:
            Optional[Path]: The path to the downloaded file, or None if download failed.
        """
        if not self.service:
            raise ValueError("Google Drive service is not initialized. Please authenticate first.")
        dest_path = Path(dest_path)
        try:
            request = self.service.files().get_media(fileId=file_id)  # type: ignore
            with open(dest_path, "wb") as fh:
                downloader = MediaIoBaseDownload(fh, request)
                done = False
                while not done:
                    status, done = downloader.next_chunk()
                    print(f"Download progress: {int(status.progress() * 100)}%.")
            return dest_path
        except Exception as error:
            log_error(f"Could not download file '{file_id}': {error}")
            return None
```

---

### GoogleSheetsTools (`agno.tools.google.sheets`)
Read and write data to Google Sheets.

**Authentication**: OAuth2 or Service Account.
**Dependencies**: `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`

#### Parameters
- `spreadsheet_id` (str): Target sheet ID.
- `read_sheet` (bool): Default True.

#### Source Code
```python
"""
Google Sheets Toolset for interacting with Sheets API

Required Environment Variables:
-----------------------------
- GOOGLE_CLIENT_ID: Google OAuth client ID
- GOOGLE_CLIENT_SECRET: Google OAuth client secret
- GOOGLE_PROJECT_ID: Google Cloud project ID
- GOOGLE_REDIRECT_URI: Google OAuth redirect URI (default: http://localhost)

How to Get These Credentials:
---------------------------
1. Go to Google Cloud Console (https://console.cloud.google.com)
2. Create a new project or select an existing one
3. Enable the Google Sheets API:
   - Go to "APIs & Services" > "Enable APIs and Services"
   - Search for "Google Sheets API"
   - Click "Enable"

4. Create OAuth 2.0 credentials:
   - Go to "APIs & Services" > "Credentials"
   - Click "Create Credentials" > "OAuth client ID"
   - Go through the OAuth consent screen setup
   - Give it a name and click "Create"
   - You'll receive:
     * Client ID (GOOGLE_CLIENT_ID)
     * Client Secret (GOOGLE_CLIENT_SECRET)
   - The Project ID (GOOGLE_PROJECT_ID) is visible in the project dropdown at the top of the page

5. Set up environment variables:
   Create a .envrc file in your project root with:
   ```
   export GOOGLE_CLIENT_ID=your_client_id_here
   export GOOGLE_CLIENT_SECRET=your_client_secret_here
   export GOOGLE_PROJECT_ID=your_project_id_here
   export GOOGLE_REDIRECT_URI=http://localhost  # Default value
   ```

Alternatively, follow the instructions in the Google Sheets API Quickstart guide:
1: Steps: https://developers.google.com/sheets/api/quickstart/python
2: Save the credentials.json file to the root of the project or update the path in the GoogleSheetsTools class

Note: The first time you run the application, it will open a browser window for OAuth authentication.
A token.json file will be created to store the authentication credentials for future use.
"""

import json
from functools import wraps
from os import getenv
from pathlib import Path
from typing import Any, List, Optional, Union

from agno.tools import Toolkit

try:
    from google.auth.transport.requests import Request
    from google.oauth2.credentials import Credentials
    from google.oauth2.service_account import Credentials as ServiceAccountCredentials
    from google_auth_oauthlib.flow import InstalledAppFlow
    from googleapiclient.discovery import Resource, build
except ImportError:
    raise ImportError(
        "`google-api-python-client` `google-auth-httplib2` `google-auth-oauthlib` not installed. Please install using `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`"
    )


def authenticate(func):
    """Decorator to ensure authentication before executing a function."""

    @wraps(func)
    def wrapper(self, *args, **kwargs):
        if not self.creds or not self.creds.valid:
            self._auth()
        if not self.service:
            self.service = build("sheets", "v4", credentials=self.creds)
        return func(self, *args, **kwargs)

    return wrapper


class GoogleSheetsTools(Toolkit):
    # Default scopes for Google Sheets API access
    DEFAULT_SCOPES = {
        "read": "https://www.googleapis.com/auth/spreadsheets.readonly",
        "write": "https://www.googleapis.com/auth/spreadsheets",
    }

    service: Optional[Resource]

    def __init__(
        self,
        scopes: Optional[List[str]] = None,
        spreadsheet_id: Optional[str] = None,
        spreadsheet_range: Optional[str] = None,
        creds: Optional[Union[Credentials, ServiceAccountCredentials]] = None,
        creds_path: Optional[str] = None,
        token_path: Optional[str] = None,
        service_account_path: Optional[str] = None,
        oauth_port: int = 0,
        read_sheet: bool = True,
        create_sheet: bool = False,
        update_sheet: bool = False,
        create_duplicate_sheet: bool = False,
        # Backward compat aliases (deprecated)
        enable_read_sheet: Optional[bool] = None,
        enable_create_sheet: Optional[bool] = None,
        enable_update_sheet: Optional[bool] = None,
        enable_create_duplicate_sheet: Optional[bool] = None,
        all: bool = False,
        **kwargs,
    ):
        """Initialize GoogleSheetsTools with the specified configuration.

        Args:
            scopes (Optional[List[str]]): Custom OAuth scopes. If None, uses write scope by default.
            spreadsheet_id (Optional[str]): ID of the target spreadsheet.
            spreadsheet_range (Optional[str]): Range within the spreadsheet.
            creds (Optional[Credentials | ServiceAccountCredentials]): Pre-existing credentials.
            creds_path (Optional[str]): Path to credentials file.
            token_path (Optional[str]): Path to token file.
            service_account_path (Optional[str]): Path to a service account file.
            oauth_port (int): Port to use for OAuth authentication. Defaults to 0.
            read_sheet (bool): Enable reading from a sheet.
            create_sheet (bool): Enable creating a sheet.
            update_sheet (bool): Enable updating a sheet.
            create_duplicate_sheet (bool): Enable creating a duplicate sheet.
            enable_read_sheet (Optional[bool]): Deprecated alias for read_sheet.
            enable_create_sheet (Optional[bool]): Deprecated alias for create_sheet.
            enable_update_sheet (Optional[bool]): Deprecated alias for update_sheet.
            enable_create_duplicate_sheet (Optional[bool]): Deprecated alias for create_duplicate_sheet.
            all (bool): Enable all tools.
        """
        # Resolve deprecated aliases: explicit deprecated flag overrides new flag
        _read_sheet = enable_read_sheet if enable_read_sheet is not None else read_sheet
        _create_sheet = enable_create_sheet if enable_create_sheet is not None else create_sheet
        _update_sheet = enable_update_sheet if enable_update_sheet is not None else update_sheet
        _create_duplicate_sheet = (
            enable_create_duplicate_sheet if enable_create_duplicate_sheet is not None else create_duplicate_sheet
        )

        self.spreadsheet_id = spreadsheet_id
        self.spreadsheet_range = spreadsheet_range
        self.creds = creds
        self.credentials_path = creds_path
        self.token_path = token_path
        self.oauth_port = oauth_port
        self.service: Optional[Resource] = None
        self.service_account_path = service_account_path

        # Determine required scopes based on operations if no custom scopes provided
        if scopes is None:
            self.scopes = []
            if _read_sheet:
                self.scopes.append(self.DEFAULT_SCOPES["read"])
            if _create_sheet or _update_sheet or _create_duplicate_sheet:
                self.scopes.append(self.DEFAULT_SCOPES["write"])
            # Remove duplicates while preserving order
            self.scopes = list(dict.fromkeys(self.scopes))
        else:
            self.scopes = scopes
            # Validate that required scopes are present for requested operations
            if (_create_sheet or _update_sheet or _create_duplicate_sheet) and self.DEFAULT_SCOPES[
                "write"
            ] not in self.scopes:
                raise ValueError(f"The scope {self.DEFAULT_SCOPES['write']} is required for write operations")
            if (
                _read_sheet
                and self.DEFAULT_SCOPES["read"] not in self.scopes
                and self.DEFAULT_SCOPES["write"] not in self.scopes
            ):
                raise ValueError(
                    f"Either {self.DEFAULT_SCOPES['read']} or {self.DEFAULT_SCOPES['write']} is required for read operations"
                )

        tools: List[Any] = []
        if all or _read_sheet:
            tools.append(self.read_sheet)
        if all or _create_sheet:
            tools.append(self.create_sheet)
        if all or _update_sheet:
            tools.append(self.update_sheet)
        if all or _create_duplicate_sheet:
            tools.append(self.create_duplicate_sheet)

        super().__init__(name="google_sheets_tools", tools=tools, **kwargs)

    def _auth(self) -> None:
        """
        Authenticate with Google Sheets API
        """
        if self.creds and self.creds.valid:
            return

        service_account_path = self.service_account_path or getenv("GOOGLE_SERVICE_ACCOUNT_FILE")

        if service_account_path:
            self.creds = ServiceAccountCredentials.from_service_account_file(
                service_account_path,
                scopes=self.scopes,
            )
            if self.creds and self.creds.expired:
                self.creds.refresh(Request())
            return

        token_file = Path(self.token_path or "token.json")
        creds_file = Path(self.credentials_path or "credentials.json")

        if token_file.exists():
            self.creds = Credentials.from_authorized_user_file(str(token_file), self.scopes)

        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:  # type: ignore
                self.creds.refresh(Request())
            else:
                client_config = {
                    "installed": {
                        "client_id": getenv("GOOGLE_CLIENT_ID"),
                        "client_secret": getenv("GOOGLE_CLIENT_SECRET"),
                        "project_id": getenv("GOOGLE_PROJECT_ID"),
                        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                        "token_uri": "https://oauth2.googleapis.com/token",
                        "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
                        "redirect_uris": [getenv("GOOGLE_REDIRECT_URI", "http://localhost")],
                    }
                }
                # File based authentication
                if creds_file.exists():
                    flow = InstalledAppFlow.from_client_secrets_file(str(creds_file), self.scopes)
                else:
                    flow = InstalledAppFlow.from_client_config(client_config, self.scopes)
                # Opens up a browser window for OAuth authentication
                self.creds = flow.run_local_server(port=self.oauth_port)
            token_file.write_text(self.creds.to_json()) if self.creds else None  # type: ignore

    @authenticate
    def read_sheet(self, spreadsheet_id: Optional[str] = None, spreadsheet_range: Optional[str] = None) -> str:
        """
        Read values from a Google Sheet. Prioritizes instance attributes over method parameters.

        Args:
            spreadsheet_id: Fallback spreadsheet ID if instance attribute is None
            spreadsheet_range: Fallback range if instance attribute is None

        Returns:
            JSON of list of rows, where each row is a list of values
        """
        if not self.creds:
            return "Not authenticated. Call auth() first."

        # Prioritize instance attributes
        sheet_id = self.spreadsheet_id or spreadsheet_id
        sheet_range = self.spreadsheet_range or spreadsheet_range

        if not sheet_id or not sheet_range:
            return "Spreadsheet ID and range must be provided either in constructor or method call"

        try:
            result = self.service.spreadsheets().values().get(spreadsheetId=sheet_id, range=sheet_range).execute()  # type: ignore
            return json.dumps(result.get("values", []))

        except Exception as e:
            return f"Error reading Google Sheet: {e}"

    @authenticate
    def create_sheet(self, title: str) -> str:
        """
        Create a Google Sheet with a given title.

        Args:
            title: The title of the Google Sheet

        Returns:
            The ID of the created Google Sheet
        """
        if not self.creds:
            return "Not authenticated. Call auth() first."

        try:
            spreadsheet = {"properties": {"title": title}}

            spreadsheet = self.service.spreadsheets().create(body=spreadsheet, fields="spreadsheetId").execute()  # type: ignore
            spreadsheet_id = spreadsheet.get("spreadsheetId")

            return f"Spreadsheet created: https://docs.google.com/spreadsheets/d/{spreadsheet_id}"

        except Exception as e:
            return f"Error creating Google Sheet: {e}"

    @authenticate
    def update_sheet(
        self, data: List[List[Any]], spreadsheet_id: Optional[str] = None, range_name: Optional[str] = None
    ) -> str:
        """Updates a Google Sheet with the provided data.

        Note: This function can overwrite existing data in the sheet.
        User needs to ensure that the provided range correctly matches the data that needs to be updated.

        Args:
            data: The data to update the sheet with
            spreadsheet_id: The ID of the Google Sheet
            range_name: The range of the Google Sheet to update

        Returns:
            A message indicating the success or failure of the operation
        """
        if not self.creds:
            return "Not authenticated. Call auth() first."

        try:
            # Define the request body
            body = {"values": data}

            # Update the sheet
            self.service.spreadsheets().values().update(  # type: ignore
                spreadsheetId=spreadsheet_id,
                range=range_name,
                valueInputOption="RAW",
                body=body,
            ).execute()

            return f"Sheet updated successfully: {spreadsheet_id}"

        except Exception as e:
            return f"Error updating Google Sheet: {e}"

    @authenticate
    def create_duplicate_sheet(
        self, source_id: str, new_title: Optional[str] = None, copy_permissions: bool = True
    ) -> str:
        """Duplicate a Google Spreadsheet using the Google Drive API's copy feature.
        This ensures an exact duplicate including formatting and data.

        Note: Make sure your credentials include the drive scope 'https://www.googleapis.com/auth/drive'

        Args:
            source_id: The ID of the source spreadsheet.
            new_title: Optional new title for the duplicated spreadsheet. If not provided, the source title will be used.
            copy_permissions: Whether to copy the permissions from the source spreadsheet. Defaults to True.

        Returns:
            A link to the duplicated spreadsheet.
        """
        if not self.creds:
            return "Not authenticated. Call auth() first."

        if not self.service:
            return "Service not initialized"

        try:
            # Ensure the drive scope is included
            if "https://www.googleapis.com/auth/drive" not in self.scopes:
                self.scopes.append("https://www.googleapis.com/auth/drive")
                self._auth()  # Re-authenticate with updated scopes

            drive_service = build("drive", "v3", credentials=self.creds)

            # Use new_title if provided, otherwise fetch the title from the source spreadsheet
            if not new_title:
                source_sheet = self.service.spreadsheets().get(spreadsheetId=source_id).execute()
                new_title = source_sheet["properties"]["title"]

            body = {"name": new_title}
            new_file = drive_service.files().copy(fileId=source_id, body=body).execute()
            new_spreadsheet_id = new_file.get("id")

            # Copy permissions if requested
            if copy_permissions:
                # Get permissions from source file
                source_permissions = (
                    drive_service.permissions()
                    .list(fileId=source_id, fields="permissions(emailAddress,role,type)")
                    .execute()
                    .get("permissions", [])
                )

                # Apply each permission to the new file
                for permission in source_permissions:
                    # Skip the owner permission as it can't be transferred
                    if permission.get("role") == "owner":
                        continue

                    drive_service.permissions().create(
                        fileId=new_spreadsheet_id,
                        body={
                            "role": permission.get("role"),
                            "type": permission.get("type"),
                            "emailAddress": permission.get("emailAddress"),
                        },
                    ).execute()

            return f"Spreadsheet duplicated successfully: https://docs.google.com/spreadsheets/d/{new_spreadsheet_id}"
        except Exception as e:
            return f"Error duplicating spreadsheet via Drive API: {e}"
```

---

### GoogleCalendarTools (`agno.tools.google.calendar`)
Manage events and check availability in Google Calendar.

**Authentication**: OAuth2.
**Dependencies**: `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`

#### Parameters
- `calendar_id` (str): Default 'primary'.
- `allow_update` (bool): Default False.

#### Source Code
```python
import datetime
import json
import uuid
from functools import wraps
from os import getenv
from pathlib import Path
from typing import Any, Dict, List, Optional, cast

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_error, log_info

try:
    from google.auth.transport.requests import Request
    from google.oauth2.credentials import Credentials
    from google_auth_oauthlib.flow import InstalledAppFlow
    from googleapiclient.discovery import Resource, build
    from googleapiclient.errors import HttpError

except ImportError:
    raise ImportError(
        "Google client libraries not found, Please install using `pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib`"
    )

SCOPES = ["https://www.googleapis.com/auth/calendar"]


def authenticate(func):
    """Decorator to ensure authentication before executing the method."""

    @wraps(func)
    def wrapper(self, *args, **kwargs):
        try:
            if not self.creds or not self.creds.valid:
                self._auth()
            if not self.service:
                self.service = build("calendar", "v3", credentials=self.creds)
        except Exception as e:
            log_error(f"An error occurred: {e}")
        return func(self, *args, **kwargs)

    return wrapper


class GoogleCalendarTools(Toolkit):
    # Default scopes for Google Calendar API access
    DEFAULT_SCOPES = {
        "read": "https://www.googleapis.com/auth/calendar.readonly",
        "write": "https://www.googleapis.com/auth/calendar",
    }

    service: Optional[Resource]

    def __init__(
        self,
        scopes: Optional[List[str]] = None,
        credentials_path: Optional[str] = None,
        token_path: Optional[str] = "token.json",
        access_token: Optional[str] = None,
        calendar_id: str = "primary",
        oauth_port: int = 8080,
        allow_update: bool = False,
        list_events: bool = True,
        create_event: bool = True,
        update_event: bool = True,
        delete_event: bool = True,
        fetch_all_events: bool = True,
        find_available_slots: bool = True,
        list_calendars: bool = True,
        **kwargs,
    ):
        self.creds: Optional[Credentials] = None
        self.service: Optional[Resource] = None
        self.calendar_id: str = calendar_id
        self.oauth_port: int = oauth_port
        self.access_token = access_token
        self.credentials_path = credentials_path
        self.token_path = token_path
        self.allow_update = allow_update
        self.scopes = scopes or []

        tools: List[Any] = []
        if list_events:
            tools.append(self.list_events)
        if create_event:
            tools.append(self.create_event)
        if update_event:
            tools.append(self.update_event)
        if delete_event:
            tools.append(self.delete_event)
        if fetch_all_events:
            tools.append(self.fetch_all_events)
        if find_available_slots:
            tools.append(self.find_available_slots)
        if list_calendars:
            tools.append(self.list_calendars)

        super().__init__(
            name="google_calendar_tools",
            tools=tools,
            **kwargs,
        )
        if not self.scopes:
            # Add read permission by default
            self.scopes.append(self.DEFAULT_SCOPES["read"])
            # Add write permission if allow_update is True
            if self.allow_update:
                self.scopes.append(self.DEFAULT_SCOPES["write"])

        # Validate that required scopes are present for requested operations
        if self.allow_update and self.DEFAULT_SCOPES["write"] not in self.scopes:
            raise ValueError(f"The scope {self.DEFAULT_SCOPES['write']} is required for write operations")
        if self.DEFAULT_SCOPES["read"] not in self.scopes and self.DEFAULT_SCOPES["write"] not in self.scopes:
            raise ValueError(
                f"Either {self.DEFAULT_SCOPES['read']} or {self.DEFAULT_SCOPES['write']} is required for read operations"
            )

    def _auth(self) -> None:
        """
        Authenticate with Google Calendar API
        """
        if self.creds and self.creds.valid:
            return

        token_file = Path(self.token_path or "token.json")
        creds_file = Path(self.credentials_path or "credentials.json")

        if token_file.exists():
            self.creds = Credentials.from_authorized_user_file(str(token_file), self.DEFAULT_SCOPES)

        if not self.creds or not self.creds.valid:
            if self.creds and self.creds.expired and self.creds.refresh_token:
                self.creds.refresh(Request())
            else:
                client_config = {
                    "installed": {
                        "client_id": getenv("GOOGLE_CLIENT_ID"),
                        "client_secret": getenv("GOOGLE_CLIENT_SECRET"),
                        "project_id": getenv("GOOGLE_PROJECT_ID"),
                        "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                        "token_uri": "https://oauth2.googleapis.com/token",
                        "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
                        "redirect_uris": [getenv("GOOGLE_REDIRECT_URI", "http://localhost")],
                    }
                }
                # File based authentication
                if creds_file.exists():
                    flow = InstalledAppFlow.from_client_secrets_file(str(creds_file), self.scopes)
                else:
                    flow = InstalledAppFlow.from_client_config(client_config, self.scopes)
                # Opens up a browser window for OAuth authentication
                self.creds = flow.run_local_server(port=self.oauth_port)

        if self.creds:
            token_file.write_text(self.creds.to_json())
            log_debug("Successfully authenticated with Google Calendar API.")
            log_info(f"Token file path: {token_file}")

    @authenticate
    def list_events(self, limit: int = 10, start_date: Optional[str] = None) -> str:
        """
        List upcoming events from the user's Google Calendar.

        Args:
            limit (Optional[int]): Number of events to return, default value is 10
            start_date (Optional[str]): The start date to return events from in ISO format (YYYY-MM-DDTHH:MM:SS)

        Returns:
            str: JSON string containing the Google Calendar events or error message
        """
        if start_date is None:
            start_date = datetime.datetime.now(datetime.timezone.utc).isoformat()
            log_debug(f"No start date provided, using current datetime: {start_date}")
        elif isinstance(start_date, str):
            try:
                start_date = datetime.datetime.fromisoformat(start_date).strftime("%Y-%m-%dT%H:%M:%S.%fZ")
            except ValueError:
                return json.dumps(
                    {"error": f"Invalid date format: {start_date}. Use ISO format (YYYY-MM-DDTHH:MM:SS)."}
                )

        try:
            service = cast(Resource, self.service)

            events_result = (
                service.events()
                .list(
                    calendarId=self.calendar_id,
                    timeMin=start_date,
                    maxResults=limit,
                    singleEvents=True,
                    orderBy="startTime",
                )
                .execute()
            )
            events = events_result.get("items", [])
            if not events:
                return json.dumps({"message": "No upcoming events found."})
            return json.dumps(events)
        except HttpError as error:
            log_error(f"An error occurred: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def create_event(
        self,
        start_date: str,
        end_date: str,
        title: Optional[str] = None,
        description: Optional[str] = None,
        location: Optional[str] = None,
        timezone: Optional[str] = "UTC",
        attendees: Optional[List[str]] = None,
        add_google_meet_link: Optional[bool] = False,
        notify_attendees: Optional[bool] = False,
    ) -> str:
        """
        Create a new event in the Google Calendar.

        Args:
            start_date (str): Start date and time of the event in ISO format (YYYY-MM-DDTHH:MM:SS)
            end_date (str): End date and time of the event in ISO format (YYYY-MM-DDTHH:MM:SS)
            title (Optional[str]): Title/summary of the event
            description (Optional[str]): Detailed description of the event
            location (Optional[str]): Location of the event
            timezone (Optional[str]): Timezone for the event (default: UTC)
            attendees (Optional[List[str]]): List of email addresses of the attendees
            add_google_meet_link (Optional[bool]): Whether to add a Google Meet video link to the event
            notify_attendees (Optional[bool]): Whether to send email notifications to attendees (default: False)

        Returns:
            str: JSON string containing the created Google Calendar event or error message
        """
        try:
            # Format attendees if provided
            attendees_list = [{"email": attendee} for attendee in attendees] if attendees else []

            # Convert ISO string to datetime and format as required
            try:
                start_time = datetime.datetime.fromisoformat(start_date).strftime("%Y-%m-%dT%H:%M:%S")
                end_time = datetime.datetime.fromisoformat(end_date).strftime("%Y-%m-%dT%H:%M:%S")
            except ValueError:
                return json.dumps({"error": "Invalid datetime format. Use ISO format (YYYY-MM-DDTHH:MM:SS)."})

            # Create event dictionary
            event: Dict[str, Any] = {
                "summary": title,
                "location": location,
                "description": description,
                "start": {"dateTime": start_time, "timeZone": timezone},
                "end": {"dateTime": end_time, "timeZone": timezone},
                "attendees": attendees_list,
            }

            # Add Google Meet link if requested
            if add_google_meet_link:
                event["conferenceData"] = {
                    "createRequest": {"requestId": str(uuid.uuid4()), "conferenceSolutionKey": {"type": "hangoutsMeet"}}
                }

            # Remove None values
            event = {k: v for k, v in event.items() if v is not None}

            # Determine sendUpdates value based on notify_attendees parameter
            send_updates = "all" if notify_attendees and attendees else "none"

            service = cast(Resource, self.service)

            event_result = (
                service.events()
                .insert(
                    calendarId=self.calendar_id,
                    body=event,
                    conferenceDataVersion=1 if add_google_meet_link else 0,
                    sendUpdates=send_updates,
                )
                .execute()
            )
            log_debug(f"Event created successfully in calendar {self.calendar_id}. Event ID: {event_result['id']}")
            return json.dumps(event_result)
        except HttpError as error:
            log_error(f"An error occurred: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def update_event(
        self,
        event_id: str,
        title: Optional[str] = None,
        description: Optional[str] = None,
        location: Optional[str] = None,
        start_date: Optional[str] = None,
        end_date: Optional[str] = None,
        timezone: Optional[str] = None,
        attendees: Optional[List[str]] = None,
        notify_attendees: Optional[bool] = False,
    ) -> str:
        """
        Update an existing event in the Google Calendar.

        Args:
            event_id (str): ID of the event to update
            title (Optional[str]): New title/summary of the event
            description (Optional[str]): New description of the event
            location (Optional[str]): New location of the event
            start_date (Optional[str]): New start date and time in ISO format (YYYY-MM-DDTHH:MM:SS)
            end_date (Optional[str]): New end date and time in ISO format (YYYY-MM-DDTHH:MM:SS)
            timezone (Optional[str]): New timezone for the event
            attendees (Optional[List[str]]): Updated list of attendee email addresses
            notify_attendees (Optional[bool]): Whether to send email notifications to attendees (default: False)

        Returns:
            str: JSON string containing the updated Google Calendar event or error message
        """
        try:
            service = cast(Resource, self.service)

            # First get the existing event to preserve its structure
            event = service.events().get(calendarId=self.calendar_id, eventId=event_id).execute()

            # Update only the fields that are provided
            if title is not None:
                event["summary"] = title
            if description is not None:
                event["description"] = description
            if location is not None:
                event["location"] = location
            if attendees is not None:
                event["attendees"] = [{"email": attendee} for attendee in attendees]

            # Handle datetime updates
            if start_date:
                try:
                    start_time = datetime.datetime.fromisoformat(start_date).strftime("%Y-%m-%dT%H:%M:%S")
                    event["start"]["dateTime"] = start_time
                    if timezone:
                        event["start"]["timeZone"] = timezone
                except ValueError:
                    return json.dumps({"error": f"Invalid start datetime format: {start_date}. Use ISO format."})

            if end_date:
                try:
                    end_time = datetime.datetime.fromisoformat(end_date).strftime("%Y-%m-%dT%H:%M:%S")
                    event["end"]["dateTime"] = end_time
                    if timezone:
                        event["end"]["timeZone"] = timezone
                except ValueError:
                    return json.dumps({"error": f"Invalid end datetime format: {end_date}. Use ISO format."})

            # Determine sendUpdates value based on notify_attendees parameter
            send_updates = "all" if notify_attendees and attendees else "none"

            # Update the event

            updated_event = (
                service.events()
                .update(calendarId=self.calendar_id, eventId=event_id, body=event, sendUpdates=send_updates)
                .execute()
            )

            log_debug(f"Event {event_id} updated successfully.")
            return json.dumps(updated_event)
        except HttpError as error:
            log_error(f"An error occurred while updating event: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def delete_event(self, event_id: str, notify_attendees: Optional[bool] = True) -> str:
        """
        Delete an event from the Google Calendar.

        Args:
            event_id (str): ID of the event to delete
            notify_attendees (Optional[bool]): Whether to send email notifications to attendees (default: False)

        Returns:
            str: JSON string containing success message or error message
        """
        try:
            # Determine sendUpdates value based on notify_attendees parameter
            send_updates = "all" if notify_attendees else "none"

            service = cast(Resource, self.service)

            service.events().delete(calendarId=self.calendar_id, eventId=event_id, sendUpdates=send_updates).execute()

            log_debug(f"Event {event_id} deleted successfully.")
            return json.dumps({"success": True, "message": f"Event {event_id} deleted successfully."})
        except HttpError as error:
            log_error(f"An error occurred while deleting event: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def fetch_all_events(
        self,
        max_results: int = 10,
        start_date: Optional[str] = None,
        end_date: Optional[str] = None,
    ) -> str:
        """
        Fetch all Google Calendar events in a given date range.

        Args:
            start_date (Optional[str]): The minimum date to include events from in ISO format (YYYY-MM-DDTHH:MM:SS).
            end_date (Optional[str]): The maximum date to include events up to in ISO format (YYYY-MM-DDTHH:MM:SS).

        Returns:
            str: JSON string containing all Google Calendar events or error message
        """
        try:
            service = cast(Resource, self.service)

            params = {
                "calendarId": self.calendar_id,
                "maxResults": min(max_results, 100),
                "singleEvents": True,
                "orderBy": "startTime",
            }

            # Set time parameters if provided
            if start_date:
                # Accept both string and already formatted ISO strings
                if isinstance(start_date, str):
                    try:
                        # Try to parse and reformat to ensure proper timezone format
                        dt = datetime.datetime.fromisoformat(start_date)
                        if dt.tzinfo is None:
                            dt = dt.replace(tzinfo=datetime.timezone.utc)
                        params["timeMin"] = dt.isoformat()
                    except ValueError:
                        # If it's already a valid ISO string, use it directly
                        params["timeMin"] = start_date
                else:
                    params["timeMin"] = start_date

            if end_date:
                # Accept both string and already formatted ISO strings
                if isinstance(end_date, str):
                    try:
                        # Try to parse and reformat to ensure proper timezone format
                        dt = datetime.datetime.fromisoformat(end_date)
                        if dt.tzinfo is None:
                            dt = dt.replace(tzinfo=datetime.timezone.utc)
                        params["timeMax"] = dt.isoformat()
                    except ValueError:
                        # If it's already a valid ISO string, use it directly
                        params["timeMax"] = end_date
                else:
                    params["timeMax"] = end_date

            # Handle pagination
            all_events = []
            page_token = None

            while True:
                if page_token:
                    params["pageToken"] = page_token

                events_result = service.events().list(**params).execute()
                all_events.extend(events_result.get("items", []))

                page_token = events_result.get("nextPageToken")
                if not page_token:
                    break

            log_debug(f"Fetched {len(all_events)} events from calendar: {self.calendar_id}")

            if not all_events:
                return json.dumps({"message": "No events found."})
            return json.dumps(all_events)
        except HttpError as error:
            log_error(f"An error occurred while fetching events: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def find_available_slots(
        self,
        start_date: str,
        end_date: str,
        duration_minutes: int = 30,
    ) -> str:
        """
        Find available time slots within a date range.

        This method fetches your actual calendar events to determine busy periods,
        then finds available slots within standard working hours (9 AM - 5 PM).

        Args:
            start_date (str): Start date to search from in ISO format (YYYY-MM-DD)
            end_date (str): End date to search to in ISO format (YYYY-MM-DD)
            duration_minutes (int): Length of the desired slot in minutes (default: 30 minutes)

        Returns:
            str: JSON string containing available Google Calendar time slots or error message
        """
        try:
            start_dt = datetime.datetime.fromisoformat(start_date)
            end_dt = datetime.datetime.fromisoformat(end_date)
            # Ensure dates are timezone-aware (use UTC if no timezone specified)
            if start_dt.tzinfo is None:
                start_dt = start_dt.replace(tzinfo=datetime.timezone.utc)
            if end_dt.tzinfo is None:
                end_dt = end_dt.replace(tzinfo=datetime.timezone.utc)

            # Get working hours from user settings
            working_hours_json = self._get_working_hours()
            working_hours_data = json.loads(working_hours_json)

            if "error" not in working_hours_data:
                working_hours_start = working_hours_data["start_hour"]
                working_hours_end = working_hours_data["end_hour"]
                timezone = working_hours_data["timezone"]
                locale = working_hours_data["locale"]
                log_debug(
                    f"Using working hours from settings: {working_hours_start}:00-{working_hours_end}:00 ({locale})"
                )
            else:
                # Fallback defaults
                working_hours_start, working_hours_end = 9, 17
                timezone = "UTC"
                locale = "en"
                log_debug("Using default working hours: 9:00-17:00")

            # Fetch actual calendar events to determine busy periods
            events_json = self.fetch_all_events(start_date=start_date, end_date=end_date)
            events_data = json.loads(events_json)

            if "error" in events_data:
                return json.dumps({"error": events_data["error"]})

            events = events_data if isinstance(events_data, list) else events_data.get("items", [])

            # Extract busy periods from actual calendar events
            busy_periods = []
            for event in events:
                # Skip all-day events and transparent events
                if event.get("transparency") == "transparent":
                    continue

                start_info = event.get("start", {})
                end_info = event.get("end", {})

                # Only process timed events (not all-day)
                if "dateTime" in start_info and "dateTime" in end_info:
                    try:
                        start_time = datetime.datetime.fromisoformat(start_info["dateTime"].replace("Z", "+00:00"))
                        end_time = datetime.datetime.fromisoformat(end_info["dateTime"].replace("Z", "+00:00"))
                        busy_periods.append((start_time, end_time))
                    except (ValueError, KeyError) as e:
                        log_debug(f"Skipping invalid event: {e}")
                        continue

            # Generate available slots within working hours
            available_slots = []
            current_date = start_dt.replace(hour=working_hours_start, minute=0, second=0, microsecond=0)
            end_search = end_dt.replace(hour=working_hours_end, minute=0, second=0, microsecond=0)

            while current_date <= end_search:
                # Skip weekends if not in working hours
                if current_date.weekday() >= 5:  # Saturday=5, Sunday=6
                    current_date = (current_date + datetime.timedelta(days=1)).replace(
                        hour=working_hours_start, minute=0, second=0, microsecond=0
                    )
                    continue

                slot_end = current_date + datetime.timedelta(minutes=duration_minutes)

                # Check if this slot conflicts with any busy period
                is_available = True
                for busy_start, busy_end in busy_periods:
                    if not (slot_end <= busy_start or current_date >= busy_end):
                        is_available = False
                        break

                # Only add slots within working hours
                if is_available and slot_end.hour <= working_hours_end:
                    available_slots.append({"start": current_date.isoformat(), "end": slot_end.isoformat()})

                # Move to next slot (30-minute intervals)
                current_date += datetime.timedelta(minutes=30)

                # Skip to next day at working hours start if past working hours end
                if current_date.hour >= working_hours_end:
                    current_date = (current_date + datetime.timedelta(days=1)).replace(
                        hour=working_hours_start, minute=0, second=0, microsecond=0
                    )

            result = {
                "available_slots": available_slots,
                "duration_minutes": duration_minutes,
                "working_hours": {"start": f"{working_hours_start:02d}:00", "end": f"{working_hours_end:02d}:00"},
                "timezone": timezone,
                "locale": locale,
                "events_analyzed": len(busy_periods),
            }

            log_debug(f"Found {len(available_slots)} available slots")
            return json.dumps(result)

        except Exception as e:
            log_error(f"An error occurred while finding available slots: {e}")
            return json.dumps({"error": f"An error occurred: {str(e)}"})

    @authenticate
    def _get_working_hours(self) -> str:
        """
        Get working hours based on user's calendar settings and locale.

        Returns:
            str: JSON string containing working hours information
        """
        try:
            # Get all user settings
            settings_result = self.service.settings().list().execute()  # type: ignore
            settings = settings_result.get("items", [])

            # Process settings into a more usable format
            user_prefs = {}
            for setting in settings:
                user_prefs[setting["id"]] = setting["value"]

            # Extract relevant settings
            timezone = user_prefs.get("timezone", "UTC")
            locale = user_prefs.get("locale", "en")
            week_start = int(user_prefs.get("weekStart", "0"))  # 0=Sunday, 1=Monday, 6=Saturday
            hide_weekends = user_prefs.get("hideWeekends", "false") == "true"

            # Determine working hours based on locale/culture
            if locale.startswith(("es", "it", "pt")):  # Spain, Italy, Portugal
                start_hour, end_hour = 9, 18
            elif locale.startswith(("de", "nl", "dk", "se", "no")):  # Northern Europe
                start_hour, end_hour = 8, 17
            elif locale.startswith(("ja", "ko")):  # East Asia
                start_hour, end_hour = 9, 18
            else:  # Default US/International
                start_hour, end_hour = 9, 17

            working_hours = {
                "start_hour": start_hour,
                "end_hour": end_hour,
                "start_time": f"{start_hour:02d}:00",
                "end_time": f"{end_hour:02d}:00",
                "timezone": timezone,
                "locale": locale,
                "week_start": week_start,
                "hide_weekends": hide_weekends,
            }

            log_debug(f"Working hours for locale {locale}: {start_hour}:00-{end_hour}:00")
            return json.dumps(working_hours)

        except HttpError as error:
            log_error(f"An error occurred while getting working hours: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})

    @authenticate
    def list_calendars(self) -> str:
        """
        List all available Google Calendars for the authenticated user.

        Returns:
            str: JSON string containing available calendars with their IDs and names
        """
        try:
            calendar_list = self.service.calendarList().list().execute()  # type: ignore
            calendars = calendar_list.get("items", [])

            all_calendars = []
            for calendar in calendars:
                calendar_info = {
                    "id": calendar.get("id"),
                    "name": calendar.get("summary", "Unnamed Calendar"),
                    "description": calendar.get("description", ""),
                    "primary": calendar.get("primary", False),
                    "access_role": calendar.get("accessRole", "unknown"),
                    "color": calendar.get("backgroundColor", "#ffffff"),
                }
                all_calendars.append(calendar_info)

            log_debug(f"Found {len(all_calendars)} calendars for user")
            return json.dumps(
                {
                    "calendars": all_calendars,
                    "current_default": self.calendar_id,
                }
            )

        except HttpError as error:
            log_error(f"An error occurred while listing calendars: {error}")
            return json.dumps({"error": f"An error occurred: {error}"})
```

---

### GoogleMapTools (`agno.tools.google.maps`)
Search for places and get directions using Google Maps.

**Authentication**: API Key.
**Environment Variables**: `GOOGLE_MAPS_API_KEY`.
**Dependencies**: `pip install googlemaps google-maps-places`

#### Parameters
- `search_places` (bool): Default True.
- `get_directions` (bool): Default True.

#### Source Code
```python
"""
This module provides tools for searching business information using the Google Maps API.

Prerequisites:
- Set the environment variable `GOOGLE_MAPS_API_KEY` with your Google Maps API key.
  You can obtain the API key from the Google Cloud Console:
  https://console.cloud.google.com/projectselector2/google/maps-apis/credentials

- You also need to activate the Address Validation API for your project.
  https://console.developers.google.com/apis/api/addressvalidation.googleapis.com

"""

import json
from datetime import datetime
from os import getenv
from typing import Any, List, Optional

from agno.tools import Toolkit

try:
    import googlemaps
    from google.maps import places_v1
except ImportError:
    print("Error importing googlemaps. Please install the package using `pip install googlemaps google-maps-places`.")


class GoogleMapTools(Toolkit):
    def __init__(
        self,
        key: Optional[str] = None,
        search_places: bool = True,
        get_directions: bool = True,
        validate_address: bool = True,
        geocode_address: bool = True,
        reverse_geocode: bool = True,
        get_distance_matrix: bool = True,
        get_elevation: bool = True,
        get_timezone: bool = True,
        **kwargs,
    ):
        self.api_key = key or getenv("GOOGLE_MAPS_API_KEY")
        if not self.api_key:
            raise ValueError("GOOGLE_MAPS_API_KEY is not set in the environment variables.")
        self.client = googlemaps.Client(key=self.api_key)

        self.places_client = places_v1.PlacesClient()

        tools: List[Any] = []
        if search_places:
            tools.append(self.search_places)
        if get_directions:
            tools.append(self.get_directions)
        if validate_address:
            tools.append(self.validate_address)
        if geocode_address:
            tools.append(self.geocode_address)
        if reverse_geocode:
            tools.append(self.reverse_geocode)
        if get_distance_matrix:
            tools.append(self.get_distance_matrix)
        if get_elevation:
            tools.append(self.get_elevation)
        if get_timezone:
            tools.append(self.get_timezone)

        super().__init__(name="google_maps", tools=tools, **kwargs)

    def search_places(self, query: str) -> str:
        """
        Search for places using Google Maps Places API.
        This tool takes a search query and returns detailed place information.

        Args:
            query (str): The query string to search for using Google Maps Search API. (e.g., "dental clinics in Noida")

        Returns:
            Stringified list of dictionaries containing business information like name, address, phone, website, rating, and reviews etc.
        """
        try:
            # Perform places search
            request = places_v1.SearchTextRequest(
                text_query=query,
            )
            response = self.places_client.search_text(request=request, metadata=[("x-goog-fieldmask", "*")])

            places = []
            for place in response.places:
                place_info = {
                    "name": place.display_name.text,
                    "address": place.formatted_address,
                    "rating": place.rating,
                    "reviews": [{"text": review.text.text, "rating": review.rating} for review in place.reviews],
                    "place_id": place.id,
                    "phone": place.international_phone_number,
                    "website": place.website_uri,
                    "hours": [description for description in place.regular_opening_hours.weekday_descriptions],
                }

                places.append(place_info)

            return json.dumps(places)

        except Exception as e:
            print(f"Error searching Google Maps: {str(e)}")
            return str([])

    def get_directions(
        self,
        origin: str,
        destination: str,
        mode: str = "driving",
        departure_time: Optional[datetime] = None,
        avoid: Optional[List[str]] = None,
    ) -> str:
        """
        Get directions between two locations using Google Maps Directions API.

        Args:
            origin (str): Starting point address or coordinates
            destination (str): Destination address or coordinates
            mode (str, optional): Travel mode. Options: "driving", "walking", "bicycling", "transit". Defaults to "driving"
            departure_time (datetime, optional): Desired departure time for transit directions
            avoid (List[str], optional): Features to avoid: "tolls", "highways", "ferries"

        Returns:
            str: Stringified dictionary containing route information including steps, distance, duration, etc.
        """
        try:
            result = self.client.directions(origin, destination, mode=mode, departure_time=departure_time, avoid=avoid)
            return str(result)
        except Exception as e:
            print(f"Error getting directions: {str(e)}")
            return str([])

    def validate_address(
        self, address: str, region_code: str = "US", locality: Optional[str] = None, enable_usps_cass: bool = False
    ) -> str:
        """
        Validate an address using Google Maps Address Validation API.

        Args:
            address (str): The address to validate
            region_code (str): The region code (e.g., "US" for United States)
            locality (str, optional): The locality (city) to help with validation
            enable_usps_cass (bool): Whether to enable USPS CASS validation for US addresses

        Returns:
            str: Stringified dictionary containing address validation results
        """
        try:
            result = self.client.addressvalidation(
                [address], regionCode=region_code, locality=locality, enableUspsCass=enable_usps_cass
            )
            return str(result)
        except Exception as e:
            print(f"Error validating address: {str(e)}")
            return str({})

    def geocode_address(self, address: str, region: Optional[str] = None) -> str:
        """
        Convert an address into geographic coordinates using Google Maps Geocoding API.

        Args:
            address (str): The address to geocode
            region (str, optional): The region code to bias results

        Returns:
            str: Stringified list of dictionaries containing location information
        """
        try:
            result = self.client.geocode(address, region=region)
            return str(result)
        except Exception as e:
            print(f"Error geocoding address: {str(e)}")
            return str([])

    def reverse_geocode(
        self, lat: float, lng: float, result_type: Optional[List[str]] = None, location_type: Optional[List[str]] = None
    ) -> str:
        """
        Convert geographic coordinates into an address using Google Maps Reverse Geocoding API.

        Args:
            lat (float): Latitude
            lng (float): Longitude
            result_type (List[str], optional): Array of address types to filter results
            location_type (List[str], optional): Array of location types to filter results

        Returns:
            str: Stringified list of dictionaries containing address information
        """
        try:
            result = self.client.reverse_geocode((lat, lng), result_type=result_type, location_type=location_type)
            return str(result)
        except Exception as e:
            print(f"Error reverse geocoding: {str(e)}")
            return str([])

    def get_distance_matrix(
        self,
        origins: List[str],
        destinations: List[str],
        mode: str = "driving",
        departure_time: Optional[datetime] = None,
        avoid: Optional[List[str]] = None,
    ) -> str:
        """
        Calculate distance and time for a matrix of origins and destinations.

        Args:
            origins (List[str]): List of addresses or coordinates
            destinations (List[str]): List of addresses or coordinates
            mode (str, optional): Travel mode. Options: "driving", "walking", "bicycling", "transit"
            departure_time (datetime, optional): Desired departure time
            avoid (List[str], optional): Features to avoid: "tolls", "highways", "ferries"

        Returns:
            str: Stringified dictionary containing distance and duration information
        """
        try:
            result = self.client.distance_matrix(
                origins, destinations, mode=mode, departure_time=departure_time, avoid=avoid
            )
            return str(result)
        except Exception as e:
            print(f"Error getting distance matrix: {str(e)}")
            return str({})

    def get_elevation(self, lat: float, lng: float) -> str:
        """
        Get the elevation for a specific location using Google Maps Elevation API.

        Args:
            lat (float): Latitude
            lng (float): Longitude

        Returns:
            str: Stringified dictionary containing elevation data
        """
        try:
            result = self.client.elevation((lat, lng))
            return str(result)
        except Exception as e:
            print(f"Error getting elevation: {str(e)}")
            return str([])

    def get_timezone(self, lat: float, lng: float, timestamp: Optional[datetime] = None) -> str:
        """
        Get timezone information for a location using Google Maps Time Zone API.

        Args:
            lat (float): Latitude
            lng (float): Longitude
            timestamp (datetime, optional): The timestamp to use for timezone calculation

        Returns:
            str: Stringified dictionary containing timezone information
        """
        try:
            if timestamp is None:
                timestamp = datetime.now()

            result = self.client.timezone(location=(lat, lng), timestamp=timestamp)
            return str(result)
        except Exception as e:
            print(f"Error getting timezone: {str(e)}")
            return str({})
```


## 18. AWS (Detailed)

### AwsSesTools (`agno.tools.aws_ses`)
Send emails using Amazon Simple Email Service (SES).

**Authentication**: AWS credentials (via boto3).
**Dependencies**: `pip install boto3`

#### Parameters
- `sender_email` (str): Verified SES sender email.
- `sender_name` (str): Display name for the sender.
- `region_name` (str): Default 'us-east-1'.

#### Source Code
```python
from typing import Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug

try:
    import boto3
except ImportError:
    raise ImportError("boto3 is required for AWSSESTool. Please install it using `pip install boto3`.")


class AWSSESTool(Toolkit):
    def __init__(
        self,
        sender_email: Optional[str] = None,
        sender_name: Optional[str] = None,
        region_name: str = "us-east-1",
        enable_send_email: bool = True,
        all: bool = False,
        **kwargs,
    ):
        tools = []
        if all or enable_send_email:
            tools.append(self.send_email)
        super().__init__(name="aws_ses_tool", tools=tools, **kwargs)
        self.client = boto3.client("ses", region_name=region_name)
        self.sender_email = sender_email
        self.sender_name = sender_name

    def send_email(self, subject: str, body: str, receiver_email: str) -> str:
        """
        Use this tool to send an email using AWS SES.

        Args: subject: The subject of the email
                body: The body of the email
                receiver_email: The email address of the receiver
        """
        if not self.client:
            raise Exception("AWS SES client not initialized. Please check the configuration.")
        if not subject:
            return "Email subject cannot be empty."
        if not body:
            return "Email body cannot be empty."
        try:
            response = self.client.send_email(
                Source=f"{self.sender_name} <{self.sender_email}>",
                Destination={
                    "ToAddresses": [receiver_email],
                },
                Message={
                    "Body": {
                        "Text": {
                            "Charset": "UTF-8",
                            "Data": body,
                        },
                    },
                    "Subject": {
                        "Charset": "UTF-8",
                        "Data": subject,
                    },
                },
            )
            log_debug(f"Email sent with message ID: {response['MessageId']}")
            return "Email sent successfully!"
        except Exception as e:
            raise Exception(f"Failed to send email: {e}")
```

---

## 19. GitHub / Version Control (Detailed)

### GithubTools (`agno.tools.github`)
Comprehensive toolkit for interacting with GitHub repositories, issues, pull requests, and more.

**Authentication**: Requires `GITHUB_ACCESS_TOKEN`.
**Dependencies**: `pip install pygithub`

#### Parameters
- `access_token` (str): Personal Access Token or Fine-grained token.
- `base_url` (str): Base URL for GitHub Enterprise (optional).

#### Source Code
> [!NOTE]
> The GitHub toolkit is extensive. Below are the core implementation details and available tools.

```python
import json
from os import getenv
from typing import Any, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger

try:
    from github import Auth, Github, GithubException

except ImportError:
    raise ImportError("`PyGithub` not installed. Please install using `pip install pygithub`")


class GithubTools(Toolkit):
    def __init__(
        self,
        access_token: Optional[str] = None,
        base_url: Optional[str] = None,
        **kwargs,
    ):
        self.access_token = access_token or getenv("GITHUB_ACCESS_TOKEN")
        self.base_url = base_url

        self.g = self.authenticate()

        tools: List[Any] = [
            self.search_repositories,
            self.list_repositories,
            self.get_repository,
            self.get_pull_request,
            self.get_pull_request_changes,
            self.create_issue,
            self.create_repository,
            self.delete_repository,
            self.list_branches,
            self.get_repository_languages,
            self.get_pull_request_count,
            self.get_repository_stars,
            self.get_pull_requests,
            self.get_pull_request_comments,
            self.create_pull_request_comment,
            self.edit_pull_request_comment,
            self.get_pull_request_with_details,
            self.get_repository_with_stats,
            self.list_issues,
            self.get_issue,
            self.comment_on_issue,
            self.close_issue,
            self.reopen_issue,
            self.assign_issue,
            self.label_issue,
            self.list_issue_comments,
            self.edit_issue,
            self.create_pull_request,
            self.create_file,
            self.get_file_content,
            self.update_file,
            self.delete_file,
            self.get_directory_content,
            self.get_branch_content,
            self.create_branch,
            self.set_default_branch,
            self.search_code,
            self.search_issues_and_prs,
            self.create_review_request,
        ]

        super().__init__(name="github", tools=tools, **kwargs)

    def authenticate(self):
        """Authenticate with GitHub using the provided access token."""
        if not self.access_token:  # Fixes lint type error
            raise ValueError("GitHub access token is required")

        auth = Auth.Token(self.access_token)
        if self.base_url:
            log_debug(f"Authenticating with GitHub Enterprise at {self.base_url}")
            return Github(base_url=self.base_url, auth=auth)
        else:
            log_debug("Authenticating with public GitHub")
            return Github(auth=auth)

    def search_repositories(
        self,
        query: str,
        sort: str = "stars",
        order: str = "desc",
        page: int = 1,
        per_page: int = 30,
    ) -> str:
        """Search for repositories on GitHub.

        Note: GitHub's Search API has a maximum limit of 1000 results per query.

        Args:
            query (str): The search query keywords.
            sort (str, optional): The field to sort results by. Can be 'stars', 'forks', or 'updated'. Defaults to 'stars'.
            order (str, optional): The order of results. Can be 'asc' or 'desc'. Defaults to 'desc'.
            page (int, optional): Page number of results to return, counting from 1. Defaults to 1.
            per_page (int, optional): Number of results per page. Max 100. Defaults to 30.

        Returns:
            A JSON-formatted string containing a list of repositories matching the search query.
        """
        log_debug(f"Searching repositories with query: '{query}', page: {page}, per_page: {per_page}")
        try:
            # Ensure per_page doesn't exceed GitHub's max of 100
            per_page = min(per_page, 100)

            repositories = self.g.search_repositories(query=query, sort=sort, order=order)

            # Get the specified page of results
            repo_list = []
            for repo in repositories.get_page(page - 1):
                repo_info = {
                    "full_name": repo.full_name,
                    "description": repo.description,
                    "url": repo.html_url,
                    "stars": repo.stargazers_count,
                    "forks": repo.forks_count,
                    "language": repo.language,
                }
                repo_list.append(repo_info)

                if len(repo_list) >= per_page:
                    break

            return json.dumps(repo_list, indent=2)

        except GithubException as e:
            logger.error(f"Error searching repositories: {e}")
            return json.dumps({"error": str(e)})

    def list_repositories(self) -> str:
        """List all repositories for the authenticated user.

        Returns:
            A JSON-formatted string containing a list of repository names.
        """
        log_debug("Listing repositories")
        try:
            repos = self.g.get_user().get_repos()
            repo_names = [repo.full_name for repo in repos]
            return json.dumps(repo_names, indent=2)
        except GithubException as e:
            logger.error(f"Error listing repositories: {e}")
            return json.dumps({"error": str(e)})

    def create_repository(
        self,
        name: str,
        private: bool = False,
        description: Optional[str] = None,
        auto_init: bool = False,
        organization: Optional[str] = None,
    ) -> str:
        """Create a new repository on GitHub.

        Args:
            name (str): The name of the repository.
            private (bool, optional): Whether the repository is private. Defaults to False.
            description (str, optional): A short description of the repository.
            auto_init (bool, optional): Whether to initialize the repository with a README. Defaults to False.
            organization (str, optional): Name of organization to create repo in. If None, creates in user account.

        Returns:
            A JSON-formatted string containing the created repository details.
        """
        log_debug(f"Creating repository: {name}")
        try:
            description = description if description is not None else ""

            if organization:
                log_debug(f"Creating in organization: {organization}")
                org = self.g.get_organization(organization)
                repo = org.create_repo(
                    name=name,
                    private=private,
                    description=description,
                    auto_init=auto_init,
                )
            else:
                repo = self.g.get_user().create_repo(
                    name=name,
                    private=private,
                    description=description,
                    auto_init=auto_init,
                )

            repo_info = {
                "name": repo.full_name,
                "url": repo.html_url,
                "private": repo.private,
                "description": repo.description,
            }
            return json.dumps(repo_info, indent=2)
        except GithubException as e:
            logger.error(f"Error creating repository: {e}")
            return json.dumps({"error": str(e)})

    def get_repository(self, repo_name: str) -> str:
        """Get details of a specific repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').

        Returns:
            A JSON-formatted string containing repository details.
        """
        log_debug(f"Getting repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            repo_info = {
                "name": repo.full_name,
                "description": repo.description,
                "url": repo.html_url,
                "stars": repo.stargazers_count,
                "forks": repo.forks_count,
                "open_issues": repo.open_issues_count,
                "language": repo.language,
                "license": repo.license.name if repo.license else None,
                "default_branch": repo.default_branch,
            }
            return json.dumps(repo_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting repository: {e}")
            return json.dumps({"error": str(e)})

    def get_repository_languages(self, repo_name: str) -> str:
        """Get the languages used in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').

        Returns:
            A JSON-formatted string containing the list of languages.
        """
        log_debug(f"Getting languages for repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            languages = repo.get_languages()
            return json.dumps(languages, indent=2)
        except GithubException as e:
            logger.error(f"Error getting repository languages: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_request_count(
        self,
        repo_name: str,
        state: str = "all",
        author: Optional[str] = None,
        base: Optional[str] = None,
        head: Optional[str] = None,
    ) -> str:
        """Get the count of pull requests for a repository based on query parameters.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            state (str, optional): The state of the PRs to count ('open', 'closed', 'all'). Defaults to 'all'.
            author (str, optional): Filter PRs by author username.
            base (str, optional): Filter PRs by base branch name.
            head (str, optional): Filter PRs by head branch name.

        Returns:
            A JSON-formatted string containing the count of pull requests.
        """
        log_debug(f"Counting pull requests for repository: {repo_name} with state: {state}")
        try:
            repo = self.g.get_repo(repo_name)
            pulls = repo.get_pulls(state=state, base=base, head=head)

            # If author is specified, filter the results
            if author:
                # If we need to filter by author and state, make sure both conditions are met
                if state != "all":
                    count = sum(1 for pr in pulls if pr.user.login == author and pr.state == state)
                else:
                    count = sum(1 for pr in pulls if pr.user.login == author)
            else:
                count = pulls.totalCount

            return json.dumps({"count": count}, indent=2)
        except GithubException as e:
            logger.error(f"Error counting pull requests: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_request(self, repo_name: str, pr_number: int) -> str:
        """Get details of a specific pull request.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.


        Returns:
            A JSON-formatted string containing pull request details.
        """
        log_debug(f"Getting pull request #{pr_number} for repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)
            pr_info = {
                "number": pr.number,
                "title": pr.title,
                "user": pr.user.login,
                "body": pr.body,
                "created_at": pr.created_at.isoformat(),
                "updated_at": pr.updated_at.isoformat(),
                "state": pr.state,
                "merged": pr.is_merged(),
                "mergeable": pr.mergeable,
                "url": pr.html_url,
            }
            return json.dumps(pr_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting pull request: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_request_changes(self, repo_name: str, pr_number: int) -> str:
        """Get the changes (files modified) in a pull request.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.

        Returns:
            A JSON-formatted string containing the list of changed files.
        """
        log_debug(f"Getting changes for pull request #{pr_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)
            files = pr.get_files()
            changes = []
            for file in files:
                file_info = {
                    "filename": file.filename,
                    "status": file.status,
                    "additions": file.additions,
                    "deletions": file.deletions,
                    "changes": file.changes,
                    "raw_url": file.raw_url,
                    "blob_url": file.blob_url,
                    "patch": file.patch,
                }
                changes.append(file_info)
            return json.dumps(changes, indent=2)
        except GithubException as e:
            logger.error(f"Error getting pull request changes: {e}")
            return json.dumps({"error": str(e)})

    def create_issue(self, repo_name: str, title: str, body: Optional[str] = None) -> str:
        """Create an issue in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            title (str): The title of the issue.
            body (str, optional): The body content of the issue.

        Returns:
            A JSON-formatted string containing the created issue details.
        """
        log_debug(f"Creating issue in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.create_issue(title=title, body=body)  # type: ignore
            issue_info = {
                "id": issue.id,
                "number": issue.number,
                "title": issue.title,
                "body": issue.body,
                "url": issue.html_url,
                "state": issue.state,
                "created_at": issue.created_at.isoformat(),
                "user": issue.user.login,
            }
            return json.dumps(issue_info, indent=2)
        except GithubException as e:
            logger.error(f"Error creating issue: {e}")
            return json.dumps({"error": str(e)})

    def list_issues(self, repo_name: str, state: str = "open", page: int = 1, per_page: int = 20) -> str:
        """List issues for a repository with pagination.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            state (str, optional): The state of issues to list ('open', 'closed', 'all'). Defaults to 'open'.
            page (int, optional): Page number of results to return, counting from 1. Defaults to 1.
            per_page (int, optional): Number of results per page. Defaults to 20.
        Returns:
            A JSON-formatted string containing a list of issues with pagination metadata.
        """
        log_debug(f"Listing issues for repository: {repo_name} with state: {state}, page: {page}, per_page: {per_page}")
        try:
            repo = self.g.get_repo(repo_name)

            issues = repo.get_issues(state=state)

            # Filter out pull requests after fetching issues
            total_issues = 0
            all_issues = []
            for issue in issues:
                if not issue.pull_request:
                    all_issues.append(issue)
                    total_issues += 1

            # Calculate pagination metadata
            total_pages = (total_issues + per_page - 1) // per_page

            # Validate page number
            if page < 1:
                page = 1
            elif page > total_pages and total_pages > 0:
                page = total_pages

            # Get the specified page of results
            issue_list = []
            page_start = (page - 1) * per_page
            page_end = page_start + per_page

            for i in range(page_start, min(page_end, total_issues)):
                if i < len(all_issues):
                    issue = all_issues[i]
                    issue_info = {
                        "number": issue.number,
                        "title": issue.title,
                        "user": issue.user.login,
                        "created_at": issue.created_at.isoformat(),
                        "state": issue.state,
                        "url": issue.html_url,
                    }
                    issue_list.append(issue_info)

            meta = {"current_page": page, "per_page": per_page, "total_items": total_issues, "total_pages": total_pages}

            response = {"data": issue_list, "meta": meta}

            return json.dumps(response, indent=2)
        except GithubException as e:
            logger.error(f"Error listing issues: {e}")
            return json.dumps({"error": str(e)})

    def get_issue(self, repo_name: str, issue_number: int) -> str:
        """Get details of a specific issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.

        Returns:
            A JSON-formatted string containing issue details.
        """
        log_debug(f"Getting issue #{issue_number} for repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue_info = {
                "number": issue.number,
                "title": issue.title,
                "body": issue.body,
                "user": issue.user.login,
                "state": issue.state,
                "created_at": issue.created_at.isoformat(),
                "updated_at": issue.updated_at.isoformat(),
                "url": issue.html_url,
                "assignees": [assignee.login for assignee in issue.assignees],
                "labels": [label.name for label in issue.labels],
            }
            return json.dumps(issue_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting issue: {e}")
            return json.dumps({"error": str(e)})

    def comment_on_issue(self, repo_name: str, issue_number: int, comment_body: str) -> str:
        """Add a comment to an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.
            comment_body (str): The content of the comment.

        Returns:
            A JSON-formatted string containing the comment details.
        """
        log_debug(f"Adding comment to issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            comment = issue.create_comment(body=comment_body)
            comment_info = {
                "id": comment.id,
                "body": comment.body,
                "user": comment.user.login,
                "created_at": comment.created_at.isoformat(),
                "url": comment.html_url,
            }
            return json.dumps(comment_info, indent=2)
        except GithubException as e:
            logger.error(f"Error commenting on issue: {e}")
            return json.dumps({"error": str(e)})

    def close_issue(self, repo_name: str, issue_number: int) -> str:
        """Close an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.

        Returns:
            A JSON-formatted string confirming the issue is closed.
        """
        log_debug(f"Closing issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue.edit(state="closed")
            return json.dumps({"message": f"Issue #{issue_number} closed."}, indent=2)
        except GithubException as e:
            logger.error(f"Error closing issue: {e}")
            return json.dumps({"error": str(e)})

    def reopen_issue(self, repo_name: str, issue_number: int) -> str:
        """Reopen a closed issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.

        Returns:
            A JSON-formatted string confirming the issue is reopened.
        """
        log_debug(f"Reopening issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue.edit(state="open")
            return json.dumps({"message": f"Issue #{issue_number} reopened."}, indent=2)
        except GithubException as e:
            logger.error(f"Error reopening issue: {e}")
            return json.dumps({"error": str(e)})

    def assign_issue(self, repo_name: str, issue_number: int, assignees: List[str]) -> str:
        """Assign users to an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.
            assignees (List[str]): A list of usernames to assign.

        Returns:
            A JSON-formatted string confirming the assignees.
        """
        log_debug(f"Assigning users to issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue.edit(assignees=assignees)
            return json.dumps({"message": f"Issue #{issue_number} assigned to {assignees}."}, indent=2)
        except GithubException as e:
            logger.error(f"Error assigning issue: {e}")
            return json.dumps({"error": str(e)})

    def label_issue(self, repo_name: str, issue_number: int, labels: List[str]) -> str:
        """Add labels to an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.
            labels (List[str]): A list of label names to add.

        Returns:
            A JSON-formatted string confirming the labels.
        """
        log_debug(f"Labeling issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue.edit(labels=labels)
            return json.dumps(
                {"message": f"Labels {labels} added to issue #{issue_number}."},
                indent=2,
            )
        except GithubException as e:
            logger.error(f"Error labeling issue: {e}")
            return json.dumps({"error": str(e)})

    def list_issue_comments(self, repo_name: str, issue_number: int) -> str:
        """List comments on an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.

        Returns:
            A JSON-formatted string containing a list of comments.
        """
        log_debug(f"Listing comments for issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            comments = issue.get_comments()
            comment_list = []
            for comment in comments:
                comment_info = {
                    "id": comment.id,
                    "user": comment.user.login,
                    "body": comment.body,
                    "created_at": comment.created_at.isoformat(),
                    "url": comment.html_url,
                }
                comment_list.append(comment_info)
            return json.dumps(comment_list, indent=2)
        except GithubException as e:
            logger.error(f"Error listing issue comments: {e}")
            return json.dumps({"error": str(e)})

    def edit_issue(
        self,
        repo_name: str,
        issue_number: int,
        title: Optional[str] = None,
        body: Optional[str] = None,
    ) -> str:
        """Edit the title or body of an issue.

        Args:
            repo_name (str): The full name of the repository.
            issue_number (int): The number of the issue.
            title (str, optional): The new title for the issue.
            body (str, optional): The new body content for the issue.

        Returns:
            A JSON-formatted string confirming the issue has been updated.
        """
        log_debug(f"Editing issue #{issue_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            issue = repo.get_issue(number=issue_number)
            issue.edit(title=title, body=body)  # type: ignore
            return json.dumps({"message": f"Issue #{issue_number} updated."}, indent=2)
        except GithubException as e:
            logger.error(f"Error editing issue: {e}")
            return json.dumps({"error": str(e)})

    def delete_repository(self, repo_name: str) -> str:
        """Delete a repository (requires admin permissions).

            Args:
                repo_name (str): The full name of the repository to delete (e.g., 'owner/repo').

        Returns:
            A JSON-formatted string with success message or error.
        """
        log_debug(f"Deleting repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            repo.delete()
            return json.dumps({"message": f"Repository {repo_name} deleted successfully"}, indent=2)
        except GithubException as e:
            logger.error(f"Error deleting repository: {e}")
            return json.dumps({"error": str(e)})

    def list_branches(self, repo_name: str) -> str:
        """List all branches in a repository.

        Args:
            repo_name (str): Full repository name (e.g., 'owner/repo').

        Returns:
            JSON list of branch names.
        """
        try:
            repo = self.g.get_repo(repo_name)
            branches = [branch.name for branch in repo.get_branches()]
            return json.dumps(branches, indent=2)
        except GithubException as e:
            logger.error(f"Error listing branches: {e}")
            return json.dumps({"error": str(e)})

    def get_repository_stars(self, repo_name: str) -> str:
        """Get the number of stars for a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').

        Returns:
            A JSON-formatted string containing the star count.
        """
        log_debug(f"Getting star count for repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            return json.dumps({"stars": repo.stargazers_count}, indent=2)
        except GithubException as e:
            logger.error(f"Error getting repository stars: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_requests(
        self,
        repo_name: str,
        state: str = "open",
        sort: str = "created",
        direction: str = "desc",
        limit: int = 50,
    ) -> str:
        """Get pull requests matching query parameters.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            state (str, optional): State of the PRs to retrieve. Can be 'open', 'closed', or 'all'. Defaults to 'open'.
            sort (str, optional): What to sort results by. Can be 'created', 'updated', 'popularity', 'long-running'. Defaults to 'created'.
            direction (str, optional): The direction of the sort. Can be 'asc' or 'desc'. Defaults to 'desc'.
            limit (int, optional): The maximum number of pull requests to return. Defaults to 20.

        Returns:
            A JSON-formatted string containing a list of pull requests.
        """
        try:
            repo = self.g.get_repo(repo_name)
            pulls = repo.get_pulls(state=state, sort=sort, direction=direction)

            pr_list = []
            for pr in pulls[:limit]:
                pr_info = {
                    "number": pr.number,
                    "title": pr.title,
                    "user": pr.user.login,
                    "created_at": pr.created_at.isoformat(),
                    "updated_at": pr.updated_at.isoformat(),
                    "state": pr.state,
                    "url": pr.html_url,
                }
                pr_list.append(pr_info)

            return json.dumps(pr_list, indent=2)
        except GithubException as e:
            logger.error(f"Error getting pull requests by query: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_request_comments(self, repo_name: str, pr_number: int, include_issue_comments: bool = True) -> str:
        """Get all comments on a pull request.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.
            include_issue_comments (bool, optional): Whether to include general PR comments. Defaults to True.

        Returns:
            A JSON-formatted string containing a list of pull request comments.
        """
        log_debug(f"Getting comments for pull request #{pr_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)

            comment_list = []

            # Get review comments (comments on specific lines of code)
            review_comments = pr.get_comments()
            for comment in review_comments:
                comment_info = {
                    "id": comment.id,
                    "body": comment.body,
                    "user": comment.user.login,
                    "created_at": comment.created_at.isoformat(),
                    "updated_at": comment.updated_at.isoformat(),
                    "path": comment.path,
                    "position": comment.position,
                    "commit_id": comment.commit_id,
                    "url": comment.html_url,
                    "type": "review_comment",
                }
                comment_list.append(comment_info)

            # Get general issue comments if requested
            if include_issue_comments:
                issue_comments = pr.get_issue_comments()
                for comment in issue_comments:
                    comment_info = {
                        "id": comment.id,
                        "body": comment.body,
                        "user": comment.user.login,
                        "created_at": comment.created_at.isoformat(),
                        "updated_at": comment.updated_at.isoformat(),
                        "url": comment.html_url,
                        "type": "issue_comment",
                    }
                    comment_list.append(comment_info)

            # Sort all comments by creation date
            comment_list.sort(key=lambda x: x["created_at"], reverse=True)

            return json.dumps(comment_list, indent=2)
        except GithubException as e:
            logger.error(f"Error getting pull request comments: {e}")
            return json.dumps({"error": str(e)})

    def create_pull_request_comment(
        self,
        repo_name: str,
        pr_number: int,
        body: str,
        commit_id: str,
        path: str,
        position: int,
    ) -> str:
        """Create a comment on a specific line of a specific file in a pull request.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.
            body (str): The text of the comment.
            commit_id (str): The SHA of the commit to comment on.
            path (str): The relative path to the file to comment on.
            position (int): The line index in the diff to comment on.

        Returns:
            A JSON-formatted string containing the created comment details.
        """
        log_debug(f"Creating comment on pull request #{pr_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)
            commit = repo.get_commit(commit_id)
            comment = pr.create_comment(body, commit, path, position)

            comment_info = {
                "id": comment.id,
                "body": comment.body,
                "user": comment.user.login,
                "created_at": comment.created_at.isoformat(),
                "path": comment.path,
                "position": comment.position,
                "commit_id": comment.commit_id,
                "url": comment.html_url,
            }

            return json.dumps(comment_info, indent=2)
        except GithubException as e:
            logger.error(f"Error creating pull request comment: {e}")
            return json.dumps({"error": str(e)})

    def edit_pull_request_comment(self, repo_name: str, comment_id: int, body: str) -> str:
        """Edit an existing pull request comment.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            comment_id (int): The id of the comment to edit.
            body (str): The new text of the comment.

        Returns:
            A JSON-formatted string containing the updated comment details.
        """
        log_debug(f"Editing comment #{comment_id} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            comments = repo.get_pulls_comments()
            comment = None
            for comment in comments:
                if comment.id == comment_id:
                    comment.edit(body)

            if not comment:
                return f"Could not find comment #{comment_id} in repository: {repo_name}"

            comment_info = {
                "id": comment.id,
                "body": comment.body,
                "user": comment.user.login,
                "updated_at": comment.updated_at.isoformat(),
                "path": comment.path,
                "position": comment.position,
                "commit_id": comment.commit_id,
                "url": comment.html_url,
            }

            return json.dumps(comment_info, indent=2)
        except GithubException as e:
            logger.error(f"Error editing pull request comment: {e}")
            return json.dumps({"error": str(e)})

    def get_pull_request_with_details(self, repo_name: str, pr_number: int) -> str:
        """Get comprehensive details of a pull request including comments, labels, and metadata.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.

        Returns:
            A JSON-formatted string containing detailed pull request information.
        """
        log_debug(f"Getting comprehensive details for PR #{pr_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)

            # Get review comments
            review_comments = []
            for comment in pr.get_comments():
                review_comments.append(
                    {
                        "id": comment.id,
                        "body": comment.body,
                        "user": comment.user.login,
                        "created_at": comment.created_at.isoformat(),
                        "path": comment.path,
                        "position": comment.position,
                        "commit_id": comment.commit_id,
                        "url": comment.html_url,
                        "type": "review_comment",
                    }
                )

            # Get issue comments
            issue_comments = []
            for comment in pr.get_issue_comments():
                issue_comments.append(
                    {
                        "id": comment.id,
                        "body": comment.body,
                        "user": comment.user.login,
                        "created_at": comment.created_at.isoformat(),
                        "url": comment.html_url,
                        "type": "issue_comment",
                    }
                )

            # Get commit data
            commits = []
            for commit in pr.get_commits():
                commit_info = {
                    "sha": commit.sha,
                    "message": commit.commit.message,
                    "author": (commit.commit.author.name if commit.commit.author else "Unknown"),
                    "date": (commit.commit.author.date.isoformat() if commit.commit.author else None),
                    "url": commit.html_url,
                }
                commits.append(commit_info)

            # Get files changed
            files_changed = []
            for file in pr.get_files():
                file_info = {
                    "filename": file.filename,
                    "status": file.status,
                    "additions": file.additions,
                    "deletions": file.deletions,
                    "changes": file.changes,
                    "patch": file.patch,
                }
                files_changed.append(file_info)

            # Combine all comments and sort by creation date
            all_comments = review_comments + issue_comments
            all_comments.sort(key=lambda x: x["created_at"], reverse=True)

            # Get basic PR info
            pr_info = {
                "number": pr.number,
                "title": pr.title,
                "user": pr.user.login,
                "state": pr.state,
                "created_at": pr.created_at.isoformat(),
                "updated_at": pr.updated_at.isoformat(),
                "html_url": pr.html_url,
                "body": pr.body,
                "base": pr.base.ref,
                "head": pr.head.ref,
                "merged": pr.is_merged(),
                "mergeable": pr.mergeable,
                "additions": pr.additions,
                "deletions": pr.deletions,
                "changed_files": pr.changed_files,
                "labels": [label.name for label in pr.labels],
                "comments_count": {
                    "review_comments": len(review_comments),
                    "issue_comments": len(issue_comments),
                    "total": len(all_comments),
                },
                "comments": all_comments,
                "commits": commits,
                "files_changed": files_changed,
            }

            return json.dumps(pr_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting pull request details: {e}")
            return json.dumps({"error": str(e)})

    def get_repository_with_stats(self, repo_name: str) -> str:
        """Get comprehensive repository information including statistics.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').

        Returns:
            A JSON-formatted string containing detailed repository information and statistics.
        """
        log_debug(f"Getting detailed info for repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Helper function to safely convert values to primitive types
            def safe_value(val):
                if hasattr(val, "isoformat"):
                    return val.isoformat()
                elif isinstance(val, (int, float, bool, str)) or val is None:
                    return val
                else:
                    return str(val)

            # Get basic repo info
            repo_info = {
                "id": int(repo.id),
                "name": str(repo.name),
                "full_name": str(repo.full_name),
                "owner": str(repo.owner.login),
                "description": str(repo.description) if repo.description else None,
                "html_url": str(repo.html_url),
                "homepage": str(repo.homepage) if repo.homepage else None,
                "language": str(repo.language) if repo.language else None,
                "created_at": safe_value(repo.created_at),
                "updated_at": safe_value(repo.updated_at),
                "pushed_at": safe_value(repo.pushed_at),
                "size": int(repo.size),
                "stargazers_count": int(repo.stargazers_count),
                "watchers_count": int(repo.watchers_count),
                "forks_count": int(repo.forks_count),
                "open_issues_count": int(repo.open_issues_count),
                "default_branch": str(repo.default_branch),
                "topics": [str(topic) for topic in repo.get_topics()],
                "license": (str(repo.license.name) if repo.license and hasattr(repo.license, "name") else None),
                "private": bool(repo.private),
                "archived": bool(repo.archived),
            }

            # Get languages
            repo_info["languages"] = {str(lang): int(count) for lang, count in repo.get_languages().items()}

            # Calculate actual open issues (GitHub's count includes PRs)
            try:
                open_issues_count = 0
                for issue in repo.get_issues(state="open"):
                    if not issue.pull_request:
                        open_issues_count += 1
                repo_info["actual_open_issues"] = open_issues_count
            except Exception as e:
                log_debug(f"Error getting actual open issues: {e}")
                repo_info["actual_open_issues"] = None

            # Get open pull requests count
            try:
                open_prs = repo.get_pulls(state="open")
                repo_info["open_pr_count"] = int(open_prs.totalCount)
            except Exception as e:
                log_debug(f"Error getting open PRs count: {e}")
                repo_info["open_pr_count"] = None

            # Get recent open PRs
            try:
                open_prs_list = []
                open_prs = repo.get_pulls(state="open")

                # Use a simple for loop approach instead of trying to slice first
                count = 0
                for pr in open_prs:
                    if count >= 10:
                        break
                    try:
                        # Ensure all fields are primitives, not Mock objects
                        pr_data = {
                            "number": int(pr.number),
                            "title": str(pr.title),
                            "user": str(pr.user.login),
                            "created_at": safe_value(pr.created_at),
                            "updated_at": safe_value(pr.updated_at),
                            "url": str(pr.html_url),
                            "base": str(pr.base.ref),
                            "head": str(pr.head.ref),
                            "comment_count": int(pr.comments),
                        }
                        open_prs_list.append(pr_data)
                        count += 1
                    except Exception as e:
                        log_debug(f"Error processing individual PR: {e}")

                repo_info["recent_open_prs"] = open_prs_list
            except Exception as e:
                log_debug(f"Error getting recent open PRs: {e}")
                repo_info["recent_open_prs"] = []

            # Calculate PR metrics
            try:
                # Get a sample of PRs for statistics
                all_prs_list = []
                all_prs = repo.get_pulls(state="all", sort="created", direction="desc")

                pr_count = 0
                for pr in all_prs:
                    if pr_count >= 100:  # Limit to 100 PRs
                        break
                    all_prs_list.append(pr)
                    pr_count += 1

                # Calculate basic metrics
                merged_prs = []
                for pr in all_prs_list:
                    is_merged = pr.is_merged()
                    if is_merged:
                        merged_prs.append(pr)

                # Compute merge time for merged PRs (in hours)
                merge_times = []
                for pr in merged_prs:
                    if pr.merged_at and pr.created_at:
                        merge_time = (pr.merged_at - pr.created_at).total_seconds() / 3600
                        merge_times.append(merge_time)

                pr_metrics = {
                    "total_prs": len(all_prs_list),
                    "merged_prs": len(merged_prs),
                    "acceptance_rate": ((len(merged_prs) / len(all_prs_list) * 100) if len(all_prs_list) > 0 else 0),
                    "avg_time_to_merge": (sum(merge_times) / len(merge_times) if merge_times else None),
                }
                repo_info["pr_metrics"] = pr_metrics
            except Exception as e:
                log_debug(f"Error calculating PR metrics: {e}")
                repo_info["pr_metrics"] = None

            # Get contributors
            try:
                contributors: list[dict] = []
                for contributor in repo.get_contributors():
                    if len(contributors) >= 20:  # Limit to top 20
                        break
                    contributors.append(
                        {
                            "login": str(contributor.login),
                            "contributions": int(contributor.contributions),
                            "url": str(contributor.html_url),
                        }
                    )
                repo_info["contributors"] = contributors
            except Exception as e:
                log_debug(f"Error getting contributors: {e}")
                repo_info["contributors"] = []

            return json.dumps(repo_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting repository stats: {e}")
            return json.dumps({"error": str(e)})

    def create_pull_request(
        self,
        repo_name: str,
        title: str,
        body: str,
        head: str,
        base: str,
        draft: bool = False,
        maintainer_can_modify: bool = True,
    ) -> str:
        """Create a new pull request in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            title (str): The title of the pull request.
            body (str): The body text of the pull request.
            head (str): The name of the branch where your changes are implemented.
            base (str): The name of the branch you want the changes pulled into.
            draft (bool, optional): Whether the pull request is a draft. Defaults to False.
            maintainer_can_modify (bool, optional): Whether maintainers can modify the PR. Defaults to True.

        Returns:
            A JSON-formatted string containing the created pull request details.
        """
        log_debug(f"Creating pull request in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.create_pull(
                title=title,
                body=body,
                head=head,
                base=base,
                draft=draft,
                maintainer_can_modify=maintainer_can_modify,
            )

            pr_info = {
                "number": pr.number,
                "title": pr.title,
                "body": pr.body,
                "user": pr.user.login,
                "state": pr.state,
                "created_at": pr.created_at.isoformat(),
                "html_url": pr.html_url,
                "base": pr.base.ref,
                "head": pr.head.ref,
                "mergeable": pr.mergeable,
            }

            return json.dumps(pr_info, indent=2)
        except GithubException as e:
            logger.error(f"Error creating pull request: {e}")
            return json.dumps({"error": str(e)})

    def create_review_request(
        self,
        repo_name: str,
        pr_number: int,
        reviewers: List[str],
        team_reviewers: Optional[List[str]] = None,
    ) -> str:
        """Create a review request for a pull request.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            pr_number (int): The number of the pull request.
            reviewers (List[str]): List of user logins that will be requested to review.
            team_reviewers (List[str], optional): List of team slugs that will be requested to review. Defaults to None.

        Returns:
            A JSON-formatted string with the success message or error.
        """
        log_debug(f"Creating review request for PR #{pr_number} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)
            pr = repo.get_pull(pr_number)
            pr.create_review_request(reviewers=reviewers, team_reviewers=team_reviewers or [])

            return json.dumps(
                {
                    "message": f"Review request created for PR #{pr_number}",
                    "requested_reviewers": reviewers,
                    "requested_team_reviewers": team_reviewers or [],
                },
                indent=2,
            )
        except GithubException as e:
            logger.error(f"Error creating review request: {e}")
            return json.dumps({"error": str(e)})

    def create_file(
        self,
        repo_name: str,
        path: str,
        content: str,
        message: str,
        branch: Optional[str] = None,
    ) -> str:
        """Create a new file in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            path (str): The path to the file in the repository.
            content (str): The content of the file.
            message (str): The commit message.
            branch (str, optional): The branch to commit to. Defaults to repository's default branch.

        Returns:
            A JSON-formatted string containing the file creation result.
        """
        log_debug(f"Creating file {path} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Convert string content to bytes
            content_bytes = content.encode("utf-8")

            # Create the file
            result = repo.create_file(path=path, message=message, content=content_bytes, branch=branch)

            # Extract relevant information
            file_info = {
                "path": result["content"].path,  # type: ignore
                "sha": result["content"].sha,
                "url": result["content"].html_url,
                "commit": {
                    "sha": result["commit"].sha,
                    "message": result["commit"].commit.message
                    if result["commit"].commit
                    else result["commit"]._rawData["message"],
                    "url": result["commit"].html_url,
                },
            }

            return json.dumps(file_info, indent=2)
        except (GithubException, AssertionError) as e:
            logger.error(f"Error creating file: {e}")
            return json.dumps({"error": str(e)})

    def get_file_content(self, repo_name: str, path: str, ref: Optional[str] = None) -> str:
        """Get the content of a file in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            path (str): The path to the file in the repository.
            ref (str, optional): The name of the commit/branch/tag. Defaults to the repository's default branch.

        Returns:
            A JSON-formatted string containing the file content and metadata.
        """
        log_debug(f"Getting content of file {path} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Conditionally call get_contents based on ref
            if ref is not None:
                file_content = repo.get_contents(path, ref=ref)
            else:
                file_content = repo.get_contents(path)

            # If it's a list (directory), raise an error
            if isinstance(file_content, list):
                return json.dumps({"error": f"{path} is a directory, not a file"})

            # Decode content
            try:
                decoded_content = file_content.decoded_content.decode("utf-8")
            except UnicodeDecodeError:
                decoded_content = "Binary file (content not displayed)"
            except Exception as e:
                log_debug(f"Error decoding file content: {e}")
                decoded_content = "Binary file (content not displayed)"

            # Make sure we don't try to display binary content
            if isinstance(decoded_content, str) and (
                "\x00" in decoded_content or sum(1 for c in decoded_content[:1000] if not (32 <= ord(c) <= 126)) > 200
            ):
                decoded_content = "Binary file (content not displayed)"

            # Create response
            content_info = {
                "name": file_content.name,
                "path": file_content.path,
                "sha": file_content.sha,
                "size": file_content.size,
                "type": file_content.type,
                "url": file_content.html_url,
                "content": decoded_content,
            }

            return json.dumps(content_info, indent=2)
        except GithubException as e:
            logger.error(f"Error getting file content: {e}")
            return json.dumps({"error": str(e)})

    def update_file(
        self,
        repo_name: str,
        path: str,
        content: str,
        message: str,
        sha: str,
        branch: Optional[str] = None,
    ) -> str:
        """Update an existing file in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            path (str): The path to the file in the repository.
            content (str): The new content of the file.
            message (str): The commit message.
            sha (str): The blob SHA of the file being replaced.
            branch (str, optional): The branch to commit to. Defaults to repository's default branch.

        Returns:
            A JSON-formatted string containing the file update result.
        """
        log_debug(f"Updating file {path} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Convert string content to bytes
            content_bytes = content.encode("utf-8")

            # Update the file
            result = repo.update_file(
                path=path,
                message=message,
                content=content_bytes,
                sha=sha,
                branch=branch,
            )

            # Extract relevant information
            file_info = {
                "path": result["content"].path,
                "sha": result["content"].sha,
                "url": result["content"].html_url,
                "commit": {
                    "sha": result["commit"].sha,
                    "message": result["commit"].commit.message,
                    "url": result["commit"].html_url,
                },
            }

            return json.dumps(file_info, indent=2)
        except GithubException as e:
            logger.error(f"Error updating file: {e}")
            return json.dumps({"error": str(e)})

    def delete_file(
        self,
        repo_name: str,
        path: str,
        message: str,
        sha: str,
        branch: Optional[str] = None,
    ) -> str:
        """Delete a file from a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            path (str): The path to the file in the repository.
            message (str): The commit message.
            sha (str): The blob SHA of the file being deleted.
            branch (str, optional): The branch to commit to. Defaults to repository's default branch.

        Returns:
            A JSON-formatted string containing the file deletion result.
        """
        log_debug(f"Deleting file {path} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Delete the file
            result = repo.delete_file(path=path, message=message, sha=sha, branch=branch)

            # Extract relevant information
            commit_info = {
                "message": f"File {path} deleted successfully",
                "commit": {
                    "sha": result["commit"].sha,
                    "message": result["commit"].commit.message,
                    "url": result["commit"].html_url,
                },
            }

            return json.dumps(commit_info, indent=2)
        except GithubException as e:
            logger.error(f"Error deleting file: {e}")
            return json.dumps({"error": str(e)})

    def get_directory_content(self, repo_name: str, path: str, ref: Optional[str] = None) -> str:
        """Get the contents of a directory in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            path (str): The path to the directory in the repository. Use empty string for root.
            ref (str, optional): The name of the commit/branch/tag. Defaults to repository's default branch.

        Returns:
            A JSON-formatted string containing a list of directory contents.
        """
        log_debug(f"Getting contents of directory {path} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Conditionally call get_contents based on ref
            if ref is not None:
                contents = repo.get_contents(path, ref=ref)
            else:
                contents = repo.get_contents(path)

            # If it's not a list, it's a file not a directory
            if not isinstance(contents, list):
                return json.dumps({"error": f"{path} is a file, not a directory"})

            # Process directory contents
            items = []
            for content in contents:
                item = {
                    "name": content.name,
                    "path": content.path,
                    "type": content.type,
                    "size": content.size,
                    "sha": content.sha,
                    "url": content.html_url,
                    "download_url": content.download_url,
                }
                items.append(item)

            # Sort by type (directories first) and then by name
            items.sort(key=lambda x: (x["type"] != "dir", x["name"].lower()))

            return json.dumps(items, indent=2)
        except GithubException as e:
            logger.error(f"Error getting directory contents: {e}")
            return json.dumps({"error": str(e)})

    def get_branch_content(self, repo_name: str, branch: str = "main") -> str:
        """Get the root directory content of a specific branch.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            branch (str, optional): The branch name. Defaults to "main".

        Returns:
            A JSON-formatted string containing a list of branch contents.
        """
        log_debug(f"Getting contents of branch {branch} in repository: {repo_name}")
        try:
            # This is just a convenience function that uses get_directory_content with empty path
            return self.get_directory_content(repo_name=repo_name, path="", ref=branch)
        except GithubException as e:
            logger.error(f"Error getting branch contents: {e}")
            return json.dumps({"error": str(e)})

    def create_branch(self, repo_name: str, branch_name: str, source_branch: Optional[str] = None) -> str:
        """Create a new branch in a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            branch_name (str): The name of the new branch.
            source_branch (str, optional): The source branch to create from. Defaults to repository's default branch.

        Returns:
            A JSON-formatted string containing information about the created branch.
        """
        log_debug(f"Creating branch {branch_name} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Get the source branch or default branch if not specified
            if source_branch is None:
                source_branch = repo.default_branch

            # Get the SHA of the latest commit on the source branch
            source_branch_ref = repo.get_git_ref(f"heads/{source_branch}")
            sha = source_branch_ref.object.sha

            # Create the new branch
            new_branch = repo.create_git_ref(f"refs/heads/{branch_name}", sha)

            branch_info = {
                "name": branch_name,
                "sha": new_branch.object.sha,
                "url": new_branch.url.replace("api.github.com/repos", "github.com").replace("git/refs/heads", "tree"),
            }

            return json.dumps(branch_info, indent=2)
        except GithubException as e:
            logger.error(f"Error creating branch: {e}")
            return json.dumps({"error": str(e)})

    def set_default_branch(self, repo_name: str, branch_name: str) -> str:
        """Set the default branch for a repository.

        Args:
            repo_name (str): The full name of the repository (e.g., 'owner/repo').
            branch_name (str): The name of the branch to set as default.

        Returns:
            A JSON-formatted string with success message or error.
        """
        log_debug(f"Setting default branch to {branch_name} in repository: {repo_name}")
        try:
            repo = self.g.get_repo(repo_name)

            # Check if the branch exists by looking at all branches
            branches = [branch.name for branch in repo.get_branches()]
            if branch_name not in branches:
                return json.dumps({"error": f"Branch '{branch_name}' does not exist"})

            # Set the default branch
            repo.edit(default_branch=branch_name)

            return json.dumps(
                {
                    "message": f"Default branch changed to {branch_name}",
                    "repository": repo_name,
                    "default_branch": branch_name,
                },
                indent=2,
            )
        except GithubException as e:
            logger.error(f"Error setting default branch: {e}")
            return json.dumps({"error": str(e)})

    def search_code(
        self,
        query: str,
        language: Optional[str] = None,
        repo: Optional[str] = None,
        user: Optional[str] = None,
        path: Optional[str] = None,
        filename: Optional[str] = None,
    ) -> str:
        """Search for code in GitHub repositories.

        Args:
            query (str): The search query.
            language (str, optional): Filter by language. Defaults to None.
            repo (str, optional): Filter by repository (e.g., 'owner/repo'). Defaults to None.
            user (str, optional): Filter by user or organization. Defaults to None.
            path (str, optional): Filter by file path. Defaults to None.
            filename (str, optional): Filter by filename. Defaults to None.

        Returns:
            A JSON-formatted string containing the search results.
        """
        log_debug(f"Searching code with query: {query}")
        try:
            search_query = query

            # Add filters to the query if provided
            if language:
                search_query += f" language:{language}"
            if repo:
                search_query += f" repo:{repo}"
            if user:
                search_query += f" user:{user}"
            if path:
                search_query += f" path:{path}"
            if filename:
                search_query += f" filename:{filename}"

            # Perform the search
            log_debug(f"Final search query: {search_query}")
            code_results = self.g.search_code(search_query)

            results: list[dict] = []
            limit = 60
            max_pages = 2  # GitHub returns 30 items per page, so 2 pages covers our limit
            page_index = 0

            while len(results) < limit and page_index < max_pages:
                # Fetch one page of results from GitHub API
                page_items = code_results.get_page(page_index)

                # Stop if no more results available
                if not page_items:
                    break

                # Process each code result in the current page
                for code in page_items:
                    code_info = {
                        "repository": code.repository.full_name,
                        "path": code.path,
                        "name": code.name,
                        "sha": code.sha,
                        "html_url": code.html_url,
                        "git_url": code.git_url,
                        "score": code.score,
                    }
                    results.append(code_info)
                page_index += 1

            # Return search results
            return json.dumps(
                {
                    "query": search_query,
                    "total_count": code_results.totalCount,
                    "results_count": len(results),
                    "results": results,
                },
                indent=2,
            )
        except GithubException as e:
            logger.error(f"Error searching code: {e}")
            return json.dumps({"error": str(e)})

    def search_issues_and_prs(
        self,
        query: str,
        state: Optional[str] = None,
        type_filter: Optional[str] = None,
        repo: Optional[str] = None,
        user: Optional[str] = None,
        label: Optional[str] = None,
        sort: str = "created",
        order: str = "desc",
        page: int = 1,
        per_page: int = 30,
    ) -> str:
        """Search for issues and pull requests on GitHub.

        Args:
            query (str): The search query.
            state (str, optional): Filter by state ('open', 'closed'). Defaults to None.
            type_filter (str, optional): Filter by type ('issue', 'pr'). Defaults to None.
            repo (str, optional): Filter by repository (e.g., 'owner/repo'). Defaults to None.
            user (str, optional): Filter by user or organization. Defaults to None.
            label (str, optional): Filter by label. Defaults to None.
            sort (str, optional): Sort results by ('created', 'updated', 'comments'). Defaults to "created".
            order (str, optional): Sort order ('asc', 'desc'). Defaults to "desc".
            page (int, optional): Page number for pagination. Defaults to 1.
            per_page (int, optional): Number of results per page. Defaults to 30.

        Returns:
            A JSON-formatted string containing the search results.
        """
        log_debug(f"Searching issues and PRs with query: {query}")
        try:
            search_query = query

            # Add filters to the query if provided
            if state:
                search_query += f" state:{state}"
            if type_filter == "issue":
                search_query += " is:issue"
            elif type_filter == "pr":
                search_query += " is:pr"
            if repo:
                search_query += f" repo:{repo}"
            if user:
                search_query += f" user:{user}"
            if label:
                search_query += f" label:{label}"

            # Perform the search
            log_debug(f"Final search query: {search_query}")
            issue_results = self.g.search_issues(search_query, sort=sort, order=order)

            # Process results
            per_page = min(per_page, 100)  # Ensure per_page doesn't exceed 100
            results = []

            try:
                # Get the specific page of results
                page_items = issue_results.get_page(page - 1)

                for issue in page_items:
                    issue_info = {
                        "number": issue.number,
                        "title": issue.title,
                        "repository": issue.repository.full_name,
                        "state": issue.state,
                        "created_at": issue.created_at.isoformat(),
                        "updated_at": issue.updated_at.isoformat(),
                        "html_url": issue.html_url,
                        "user": issue.user.login,
                        "is_pull_request": hasattr(issue, "pull_request") and issue.pull_request is not None,
                        "comments": issue.comments,
                        "labels": [label.name for label in issue.labels],
                    }
                    results.append(issue_info)

                    if len(results) >= per_page:
                        break
            except IndexError:
                # Page is out of range
                pass

            # Return search results
            return json.dumps(
                {
                    "query": search_query,
                    "total_count": issue_results.totalCount,
                    "page": page,
                    "per_page": per_page,
                    "results_count": len(results),
                    "results": results,
                },
                indent=2,
            )
        except GithubException as e:
            logger.error(f"Error searching issues and PRs: {e}")
            return json.dumps({"error": str(e)})
```


## 20. AI / Media Generation (Detailed)

### OpenCVTools (`agno.tools.opencv`)
Capture images and videos from the webcam using OpenCV.

**Dependencies**: `pip install opencv-python`
**Notes**: Requires camera permissions for the terminal/process.

#### Parameters
- `show_preview` (bool): Show live camera preview window.
- `enable_capture_image` (bool): Default True.
- `enable_capture_video` (bool): Default True.

#### Source Code
```python
import time
from pathlib import Path
from typing import Callable, List
from uuid import uuid4

from agno.agent import Agent
from agno.media import Image, Video
from agno.tools import Toolkit
from agno.tools.function import ToolResult
from agno.utils.log import log_debug, log_error, log_info

try:
    import cv2
except ImportError:
    raise ImportError("`opencv-python` package not found. Please install it with `pip install opencv-python`")


class OpenCVTools(Toolkit):
    """Tools for capturing images and videos from the webcam using OpenCV"""

    def __init__(
        self,
        show_preview=False,
        enable_capture_image: bool = True,
        enable_capture_video: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.show_preview = show_preview

        tools: List[Callable] = []
        if all or enable_capture_image:
            tools.append(self.capture_image)
        if all or enable_capture_video:
            tools.append(self.capture_video)

        super().__init__(
            name="opencv_tools",
            tools=tools,
            **kwargs,
        )

    def capture_image(
        self,
        agent: Agent,
        prompt: str = "Webcam capture",
    ) -> ToolResult:
        """Capture an image from the webcam.

        Args:
            prompt (str): Description of the image capture. Defaults to "Webcam capture".

        Returns:
            ToolResult: A ToolResult containing the captured image or error message.
        """
        try:
            log_debug("Initializing webcam for image capture...")
            cam = cv2.VideoCapture(0)

            if not cam.isOpened():
                cam = cv2.VideoCapture(0, cv2.CAP_AVFOUNDATION)  # macOS
            if not cam.isOpened():
                cam = cv2.VideoCapture(0, cv2.CAP_DSHOW)  # Windows
            if not cam.isOpened():
                cam = cv2.VideoCapture(0, cv2.CAP_V4L2)  # Linux

            if not cam.isOpened():
                error_msg = "Could not open webcam. Please ensure your terminal has camera permissions and the camera is not being used by another application."
                log_error(error_msg)
                return ToolResult(content=error_msg)

            try:
                cam.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)
                cam.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)
                cam.set(cv2.CAP_PROP_FPS, 30)

                log_debug("Camera initialized successfully")
                captured_frame = None

                if self.show_preview:
                    log_info("Live preview started. Press 'c' to capture image, 'q' to quit.")

                    while True:
                        ret, frame = cam.read()
                        if not ret:
                            error_msg = "Failed to read frame from webcam"
                            log_error(error_msg)
                            return ToolResult(content=error_msg)

                        cv2.imshow('Camera Preview - Press "c" to capture, "q" to quit', frame)

                        key = cv2.waitKey(1) & 0xFF
                        if key == ord("c"):
                            captured_frame = frame.copy()
                            log_info("Image captured!")
                            break
                        elif key == ord("q"):
                            log_info("Capture cancelled by user")
                            return ToolResult(content="Image capture cancelled by user")
                else:
                    ret, captured_frame = cam.read()
                    if not ret:
                        error_msg = "Failed to capture image from webcam"
                        log_error(error_msg)
                        return ToolResult(content=error_msg)

                if captured_frame is None:
                    error_msg = "No frame captured"
                    log_error(error_msg)
                    return ToolResult(content=error_msg)

                success, encoded_image = cv2.imencode(".png", captured_frame)

                if not success:
                    error_msg = "Failed to encode captured image"
                    log_error(error_msg)
                    return ToolResult(content=error_msg)

                image_bytes = encoded_image.tobytes()
                media_id = str(uuid4())

                # Create ImageArtifact with raw bytes (not base64 encoded)
                image_artifact = Image(
                    id=media_id,
                    content=image_bytes,  # Store as raw bytes
                    original_prompt=prompt,
                    mime_type="image/png",
                )

                log_debug(f"Successfully captured and attached image {media_id}")
                return ToolResult(
                    content="Image captured successfully",
                    images=[image_artifact],
                )

            finally:
                # Release the camera and close windows
                cam.release()
                cv2.destroyAllWindows()
                log_debug("Camera resources released")

        except Exception as e:
            error_msg = f"Error capturing image: {str(e)}"
            log_error(error_msg)
            return ToolResult(content=error_msg)

    def capture_video(
        self,
        agent: Agent,
        duration: int = 10,
        prompt: str = "Webcam video capture",
    ) -> ToolResult:
        """Capture a video from the webcam.

        Args:
            duration (int): Duration in seconds to record video. Defaults to 10 seconds.
            prompt (str): Description of the video capture. Defaults to "Webcam video capture".

        Returns:
            ToolResult: A ToolResult containing the captured video or error message.
        """
        try:
            log_debug("Initializing webcam for video capture...")
            cap = cv2.VideoCapture(0)

            # Try different backends for better compatibility
            if not cap.isOpened():
                cap = cv2.VideoCapture(0, cv2.CAP_AVFOUNDATION)  # macOS
            if not cap.isOpened():
                cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)  # Windows
            if not cap.isOpened():
                cap = cv2.VideoCapture(0, cv2.CAP_V4L2)  # Linux

            if not cap.isOpened():
                error_msg = "Could not open webcam. Please ensure your terminal has camera permissions and the camera is not being used by another application."
                log_error(error_msg)
                return ToolResult(content=error_msg)

            try:
                frame_width = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
                frame_height = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))
                actual_fps = cap.get(cv2.CAP_PROP_FPS)

                # Use actual FPS or default to 30 if detection fails
                if actual_fps <= 0 or actual_fps > 60:
                    actual_fps = 30.0

                log_debug(f"Video properties: {frame_width}x{frame_height} at {actual_fps} FPS")

                # Try different codecs in order of preference for compatibility
                codecs_to_try = [
                    ("avc1", "H.264"),  # Most compatible
                    ("mp4v", "MPEG-4"),  # Fallback
                    ("XVID", "Xvid"),  # Another fallback
                ]

                import os
                import tempfile

                with tempfile.NamedTemporaryFile(suffix=".mp4", delete=False) as temp_file:
                    temp_filepath = temp_file.name

                out = None
                successful_codec = None

                for codec_fourcc, codec_name in codecs_to_try:
                    try:
                        fourcc = getattr(cv2, "VideoWriter_fourcc")(*codec_fourcc)
                        out = cv2.VideoWriter(temp_filepath, fourcc, actual_fps, (frame_width, frame_height))

                        if out.isOpened():
                            successful_codec = codec_name
                            log_debug(f"Successfully initialized video writer with {codec_name} codec")
                            break
                        else:
                            out.release()
                            out = None
                    except Exception as e:
                        log_debug(f"Failed to initialize {codec_name} codec: {e}")
                        if out:
                            out.release()
                            out = None

                if not out or not out.isOpened():
                    error_msg = "Failed to initialize video writer with any codec"
                    log_error(error_msg)
                    return ToolResult(content=error_msg)

                start_time = time.time()
                frame_count = 0

                if self.show_preview:
                    log_info(f"Recording {duration}s video with live preview using {successful_codec} codec...")
                else:
                    log_info(f"Recording {duration}s video using {successful_codec} codec...")

                while True:
                    ret, frame = cap.read()

                    if not ret:
                        error_msg = "Failed to capture video frame"
                        log_error(error_msg)
                        return ToolResult(content=error_msg)

                    # Write the frame to the output file
                    out.write(frame)
                    frame_count += 1

                    # Show live preview if enabled
                    if self.show_preview:
                        # Add recording indicator
                        elapsed = time.time() - start_time
                        remaining = max(0, duration - elapsed)

                        # Draw recording info on frame
                        display_frame = frame.copy()
                        cv2.putText(
                            display_frame,
                            f"REC {remaining:.1f}s",
                            (10, 30),
                            cv2.FONT_HERSHEY_SIMPLEX,
                            1,
                            (0, 0, 255),
                            2,
                        )
                        cv2.circle(display_frame, (30, 60), 10, (0, 0, 255), -1)  # Red dot

                        cv2.imshow(f"Recording Video - {remaining:.1f}s remaining", display_frame)
                        cv2.waitKey(1)

                    # Check if recording duration is reached
                    if time.time() - start_time >= duration:
                        break

                # Release video writer
                out.release()

                # Verify the file was created and has content
                temp_path = Path(temp_filepath)
                if not temp_path.exists() or temp_path.stat().st_size == 0:
                    error_msg = "Video file was not created or is empty"
                    log_error(error_msg)
                    return ToolResult(content=error_msg)

                # Read the video file and encode to base64
                with open(temp_filepath, "rb") as video_file:
                    video_bytes = video_file.read()

                # Clean up temporary file
                os.unlink(temp_filepath)

                media_id = str(uuid4())

                # Create VideoArtifact with base64 encoded content
                video_artifact = Video(
                    id=media_id,
                    content=video_bytes,
                    original_prompt=prompt,
                    mime_type="video/mp4",
                )

                actual_duration = time.time() - start_time
                log_debug(
                    f"Successfully captured and attached video {media_id} ({actual_duration:.1f}s, {frame_count} frames)"
                )

                return ToolResult(
                    content=f"Video captured successfully and attached as artifact {media_id} ({actual_duration:.1f}s, {frame_count} frames, {successful_codec} codec)",
                    videos=[video_artifact],
                )

            finally:
                if "cap" in locals():
                    cap.release()
                cv2.destroyAllWindows()
                log_debug("Video capture resources released")

        except Exception as e:
            error_msg = f"Error capturing video: {str(e)}"
            log_error(error_msg)
            return ToolResult(content=error_msg)
```

---

### UnsplashTools (`agno.tools.unsplash`)
Search and retrieve high-quality, royalty-free images from Unsplash.

**Authentication**: Requires `UNSPLASH_ACCESS_KEY`.
**External URL**: [Unsplash Developers](https://unsplash.com/developers)

#### Parameters
- `access_key` (str): Unsplash API access key.
- `enable_search_photos` (bool): Default True.

#### Source Code
```python
"""Unsplash Tools for searching and retrieving high-quality, royalty-free images.

This toolkit provides AI agents with the ability to search for and retrieve images
from Unsplash, a popular platform with over 4.3 million high-quality photos.

Get your free API key at: https://unsplash.com/developers
"""

import json
from os import getenv
from typing import Any, Dict, List, Optional
from urllib.parse import urlencode
from urllib.request import Request, urlopen

from agno.tools import Toolkit
from agno.utils.log import log_debug, logger


class UnsplashTools(Toolkit):
    """A toolkit for searching and retrieving images from Unsplash.

    Unsplash provides access to over 4.3 million high-quality, royalty-free images
    that can be used for various purposes. This toolkit enables AI agents to:
    - Search for photos by keywords
    - Get detailed information about specific photos
    - Retrieve random photos with optional filters
    - Track downloads (required by Unsplash API guidelines)

    Example:
        ```python
        from agno.agent import Agent
        from agno.models.openai import OpenAIChat
        from agno.tools.unsplash import UnsplashTools

        agent = Agent(
            model=OpenAIChat(id="gpt-4o"),
            tools=[UnsplashTools()],
        )
        agent.print_response("Find me 3 photos of mountains at sunset")
        ```
    """

    def __init__(
        self,
        access_key: Optional[str] = None,
        enable_search_photos: bool = True,
        enable_get_photo: bool = True,
        enable_get_random_photo: bool = True,
        enable_download_photo: bool = False,
        all: bool = False,
        **kwargs: Any,
    ):
        """Initialize the Unsplash toolkit.

        Args:
            access_key: Unsplash API access key. If not provided, will look for
                UNSPLASH_ACCESS_KEY environment variable.
            enable_search_photos: Enable the search_photos tool. Default: True.
            enable_get_photo: Enable the get_photo tool. Default: True.
            enable_get_random_photo: Enable the get_random_photo tool. Default: True.
            enable_download_photo: Enable the download_photo tool. Default: False.
            all: Enable all tools. Default: False.
            **kwargs: Additional arguments passed to the Toolkit base class.
        """
        self.access_key = access_key or getenv("UNSPLASH_ACCESS_KEY")
        if not self.access_key:
            logger.warning("No Unsplash API key provided. Set UNSPLASH_ACCESS_KEY environment variable.")

        self.base_url = "https://api.unsplash.com"

        tools: List[Any] = []
        if all or enable_search_photos:
            tools.append(self.search_photos)
        if all or enable_get_photo:
            tools.append(self.get_photo)
        if all or enable_get_random_photo:
            tools.append(self.get_random_photo)
        if all or enable_download_photo:
            tools.append(self.download_photo)

        super().__init__(name="unsplash_tools", tools=tools, **kwargs)

    def _make_request(self, endpoint: str, params: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
        """Make an authenticated request to the Unsplash API.

        Args:
            endpoint: API endpoint path (e.g., "/search/photos").
            params: Optional query parameters.

        Returns:
            JSON response as a dictionary.

        Raises:
            Exception: If the API request fails.
        """
        url = f"{self.base_url}{endpoint}"
        if params:
            url = f"{url}?{urlencode(params)}"

        headers = {
            "Authorization": f"Client-ID {self.access_key}",
            "Accept-Version": "v1",
        }

        request = Request(url, headers=headers)
        with urlopen(request) as response:
            return json.loads(response.read().decode())

    def _format_photo(self, photo: Dict[str, Any]) -> Dict[str, Any]:
        """Format photo data into a clean, consistent structure.

        Args:
            photo: Raw photo data from Unsplash API.

        Returns:
            Formatted photo dictionary with essential fields.
        """
        return {
            "id": photo.get("id"),
            "description": photo.get("description") or photo.get("alt_description"),
            "width": photo.get("width"),
            "height": photo.get("height"),
            "color": photo.get("color"),
            "created_at": photo.get("created_at"),
            "urls": {
                "raw": photo.get("urls", {}).get("raw"),
                "full": photo.get("urls", {}).get("full"),
                "regular": photo.get("urls", {}).get("regular"),
                "small": photo.get("urls", {}).get("small"),
                "thumb": photo.get("urls", {}).get("thumb"),
            },
            "author": {
                "name": photo.get("user", {}).get("name"),
                "username": photo.get("user", {}).get("username"),
                "profile_url": photo.get("user", {}).get("links", {}).get("html"),
            },
            "links": {
                "html": photo.get("links", {}).get("html"),
                "download": photo.get("links", {}).get("download"),
            },
            "likes": photo.get("likes"),
            "tags": [tag.get("title") for tag in photo.get("tags", [])[:5] if tag.get("title")],
        }

    def search_photos(
        self,
        query: str,
        per_page: int = 10,
        page: int = 1,
        orientation: Optional[str] = None,
        color: Optional[str] = None,
    ) -> str:
        """Search for photos on Unsplash by keyword.

        Args:
            query: The search query string (e.g., "mountain sunset", "office workspace").
            per_page: Number of results per page (1-30). Default: 10.
            page: Page number to retrieve. Default: 1.
            orientation: Filter by orientation: "landscape", "portrait", or "squarish".
            color: Filter by color: "black_and_white", "black", "white", "yellow",
                "orange", "red", "purple", "magenta", "green", "teal", "blue".

        Returns:
            JSON string containing search results with photo details including
            URLs, author information, and metadata.
        """
        if not self.access_key:
            return "Error: No Unsplash API key provided. Set UNSPLASH_ACCESS_KEY environment variable."

        if not query:
            return "Error: Please provide a search query."

        log_debug(f"Searching Unsplash for: {query}")

        try:
            params: Dict[str, Any] = {
                "query": query,
                "per_page": min(max(1, per_page), 30),
                "page": max(1, page),
            }

            if orientation and orientation in ["landscape", "portrait", "squarish"]:
                params["orientation"] = orientation

            if color:
                valid_colors = [
                    "black_and_white",
                    "black",
                    "white",
                    "yellow",
                    "orange",
                    "red",
                    "purple",
                    "magenta",
                    "green",
                    "teal",
                    "blue",
                ]
                if color in valid_colors:
                    params["color"] = color

            response = self._make_request("/search/photos", params)

            results = {
                "total": response.get("total", 0),
                "total_pages": response.get("total_pages", 0),
                "photos": [self._format_photo(photo) for photo in response.get("results", [])],
            }

            return json.dumps(results, indent=2)

        except Exception as e:
            return f"Error searching Unsplash: {e}"

    def get_photo(self, photo_id: str) -> str:
        """Get detailed information about a specific photo.

        Args:
            photo_id: The unique identifier of the photo.

        Returns:
            JSON string containing detailed photo information including
            URLs, author, metadata, EXIF data, and location if available.
        """
        if not self.access_key:
            return "Error: No Unsplash API key provided. Set UNSPLASH_ACCESS_KEY environment variable."

        if not photo_id:
            return "Error: Please provide a photo ID."

        log_debug(f"Getting Unsplash photo: {photo_id}")

        try:
            photo = self._make_request(f"/photos/{photo_id}")

            result = self._format_photo(photo)

            # Add extra details available for single photo requests
            if photo.get("exif"):
                result["exif"] = {
                    "make": photo["exif"].get("make"),
                    "model": photo["exif"].get("model"),
                    "aperture": photo["exif"].get("aperture"),
                    "exposure_time": photo["exif"].get("exposure_time"),
                    "focal_length": photo["exif"].get("focal_length"),
                    "iso": photo["exif"].get("iso"),
                }

            if photo.get("location"):
                result["location"] = {
                    "name": photo["location"].get("name"),
                    "city": photo["location"].get("city"),
                    "country": photo["location"].get("country"),
                }

            result["views"] = photo.get("views")
            result["downloads"] = photo.get("downloads")

            return json.dumps(result, indent=2)

        except Exception as e:
            return f"Error getting photo: {e}"

    def get_random_photo(
        self,
        query: Optional[str] = None,
        orientation: Optional[str] = None,
        count: int = 1,
    ) -> str:
        """Get random photo(s) from Unsplash.

        Args:
            query: Optional search query to filter random photos.
            orientation: Filter by orientation: "landscape", "portrait", or "squarish".
            count: Number of random photos to return (1-30). Default: 1.

        Returns:
            JSON string containing random photo(s) data.
        """
        if not self.access_key:
            return "Error: No Unsplash API key provided. Set UNSPLASH_ACCESS_KEY environment variable."

        log_debug(f"Getting random Unsplash photo (query={query})")

        try:
            params: Dict[str, Any] = {
                "count": min(max(1, count), 30),
            }

            if query:
                params["query"] = query

            if orientation and orientation in ["landscape", "portrait", "squarish"]:
                params["orientation"] = orientation

            response = self._make_request("/photos/random", params)

            # Response is a list when count > 1, single object when count = 1
            if isinstance(response, list):
                photos = [self._format_photo(photo) for photo in response]
            else:
                photos = [self._format_photo(response)]

            return json.dumps({"photos": photos}, indent=2)

        except Exception as e:
            return f"Error getting random photo: {e}"

    def download_photo(self, photo_id: str) -> str:
        """Trigger a download event for a photo.

        This is required by the Unsplash API guidelines when a photo is downloaded
        or used. It helps photographers track the usage of their work.

        Args:
            photo_id: The unique identifier of the photo being downloaded.

        Returns:
            JSON string with the download URL.
        """
        if not self.access_key:
            return "Error: No Unsplash API key provided. Set UNSPLASH_ACCESS_KEY environment variable."

        if not photo_id:
            return "Error: Please provide a photo ID."

        log_debug(f"Tracking download for Unsplash photo: {photo_id}")

        try:
            response = self._make_request(f"/photos/{photo_id}/download")

            return json.dumps(
                {
                    "photo_id": photo_id,
                    "download_url": response.get("url"),
                },
                indent=2,
            )

        except Exception as e:
            return f"Error tracking download: {e}"
```

---

### GiphyTools (`agno.tools.giphy`)
Search for and retrieve GIFs from Giphy.

**Authentication**: Requires `GIPHY_API_KEY`.
**Dependencies**: `pip install httpx`

#### Parameters
- `api_key` (str): Giphy API key.
- `limit` (int): Number of GIFs to return. Default 1.

#### Source Code
```python
import uuid
from os import getenv
from typing import Any, List, Optional, Union

import httpx

from agno.agent import Agent
from agno.media import Image
from agno.team.team import Team
from agno.tools import Toolkit
from agno.tools.function import ToolResult
from agno.utils.log import logger


class GiphyTools(Toolkit):
    def __init__(
        self,
        api_key: Optional[str] = None,
        limit: int = 1,
        enable_search_gifs: bool = True,
        all: bool = False,
        **kwargs,
    ):
        """Initialize Giphy tools.

        Args:
            api_key: Giphy API key. Defaults to GIPHY_API_KEY environment variable.
            limit: Number of GIFs to return. Defaults to 1.
            enable_search_gifs: Whether to enable GIF search functionality. Defaults to True.
            all: Enable all functions. Defaults to False.
        """
        self.api_key = api_key or getenv("GIPHY_API_KEY")
        if not self.api_key:
            logger.error("No Giphy API key provided")

        self.limit: int = limit

        tools: List[Any] = []
        if all or enable_search_gifs:
            tools.append(self.search_gifs)

        super().__init__(name="giphy_tools", tools=tools, **kwargs)

    def search_gifs(self, agent: Union[Agent, Team], query: str) -> ToolResult:
        """Find a GIPHY gif

        Args:
            query (str): A text description of the required gif.

        Returns:
            ToolResult: Contains the found GIF images or error message.
        """

        base_url = "https://api.giphy.com/v1/gifs/search"
        params = {
            "api_key": self.api_key,
            "q": query,
            "limit": self.limit,
        }

        try:
            response = httpx.get(base_url, params=params)
            response.raise_for_status()

            # Extract the GIF URLs
            data = response.json()
            gif_urls = []
            image_artifacts = []

            for gif in data.get("data", []):
                images = gif.get("images", {})
                original_image = images["original"]

                media_id = str(uuid.uuid4())
                gif_url = original_image["url"]
                alt_text = gif["alt_text"]
                gif_urls.append(gif_url)

                # Create ImageArtifact for the GIF
                image_artifact = Image(id=media_id, url=gif_url, alt_text=alt_text, revised_prompt=query)
                image_artifacts.append(image_artifact)

            if image_artifacts:
                return ToolResult(content=f"Found {len(gif_urls)} GIF(s): {gif_urls}", images=image_artifacts)
            else:
                return ToolResult(content="No gifs found")

        except httpx.HTTPStatusError as e:
            logger.error(f"HTTP error occurred: {e.response.status_code} - {e.response.text}")
            return ToolResult(content=f"HTTP error occurred: {e.response.status_code}")
        except Exception as e:
            logger.error(f"An error occurred: {e}")
            return ToolResult(content=f"An error occurred: {e}")
```

---

## 21. Code Execution & Development (Detailed)

### CodingTools (`agno.tools.coding`)
A powerful toolkit for coding agents to read, edit, write files and run shell commands.

**Security Warning**: Can run arbitrary shell commands. Human supervision is recommended.
**Allowed Commands**: `python`, `pytest`, `pip`, `ls`, `git`, etc.

#### Parameters
- `base_dir` (str): Root directory for operations.
- `restrict_to_base_dir` (bool): Prevent escaping the base directory. Default True.

#### Source Code
```python
import functools
import shlex
import subprocess
import tempfile
from pathlib import Path
from textwrap import dedent
from typing import Any, List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import log_error, log_info, logger


@functools.lru_cache(maxsize=None)
def _warn_coding_tools() -> None:
    logger.warning("CodingTools can run arbitrary shell commands, please provide human supervision.")


class CodingTools(Toolkit):
    """A minimal, powerful toolkit for coding agents.

    Provides four core tools (read, edit, write, shell) and three optional
    exploration tools (grep, find, ls). With these primitives, an agent can
    perform any file operation, run tests, use git, install packages, search
    codebases, and more.

    Inspired by the Pi coding agent's philosophy: a small number of composable
    tools is more powerful than many specialized ones.
    """

    DEFAULT_ALLOWED_COMMANDS: List[str] = [
        "python",
        "python3",
        "pytest",
        "pip",
        "pip3",
        "cat",
        "head",
        "tail",
        "wc",
        "ls",
        "find",
        "grep",
        "mkdir",
        "rm",
        "mv",
        "cp",
        "touch",
        "echo",
        "printf",
        "git",
        "chmod",
        "diff",
        "sort",
        "uniq",
        "tr",
        "cut",
    ]

    DEFAULT_INSTRUCTIONS = dedent("""\
        You have access to coding tools: read_file, edit_file, write_file, and run_shell.
        With these tools, you can perform any coding task including reading code, making edits,
        creating files, running tests, using git, installing packages, and searching codebases.

        ## Tool Usage Guidelines

        **read_file** - Read files with line numbers. Use offset and limit to paginate large files.
        - Always read a file before editing it to understand its current contents.
        - Use the line numbers in the output to understand the file structure.

        **edit_file** - Make precise edits using exact text matching (find and replace).
        - The old_text must match exactly one location in the file, including whitespace and indentation.
        - Include enough surrounding context in old_text to ensure a unique match.
        - Prefer small, focused edits over rewriting entire files.
        - If an edit fails due to multiple matches, include more surrounding lines in old_text.

        **write_file** - Create new files or overwrite existing ones entirely.
        - Use this for creating new files. For modifying existing files, prefer edit_file.
        - Parent directories are created automatically.

        **run_shell** - Execute shell commands with timeout protection.
        - Use this for: running tests, git operations, installing packages, searching files (grep/find),
          checking system state, compiling code, and any other command-line task.
        - Commands run from the base directory.
        - Output is truncated if too long; the full output is saved to a temp file.

        ## Best Practices
        - Read before editing: always read_file before edit_file to see current contents.
        - Make small, incremental edits rather than rewriting entire files.
        - Run tests after making changes to verify correctness.\
    """)

    EXPLORATION_INSTRUCTIONS = dedent("""\

        **grep** - Search file contents for a pattern with line numbers.
        - Use for finding code patterns, function definitions, imports, etc.
        - Supports regex patterns and case-insensitive search.
        - Use the include parameter to filter by file type (e.g. "*.py").

        **find** - Search for files by glob pattern.
        - Use for discovering files in the project structure.
        - Supports recursive patterns like "**/*.py".

        **ls** - List directory contents.
        - Use for quick directory exploration.
        - Directories are shown with a trailing /.\
    """)

    def __init__(
        self,
        base_dir: Optional[Union[Path, str]] = None,
        restrict_to_base_dir: bool = True,
        max_lines: int = 2000,
        max_bytes: int = 50_000,
        shell_timeout: int = 120,
        enable_read_file: bool = True,
        enable_edit_file: bool = True,
        enable_write_file: bool = True,
        enable_run_shell: bool = True,
        enable_grep: bool = False,
        enable_find: bool = False,
        enable_ls: bool = False,
        instructions: Optional[str] = None,
        add_instructions: bool = True,
        all: bool = False,
        allowed_commands: Optional[List[str]] = None,
        **kwargs: Any,
    ):
        """Initialize CodingTools.

        Args:
            base_dir: Root directory for file operations. Defaults to cwd.
            restrict_to_base_dir: If True, file and shell operations cannot escape base_dir.
            max_lines: Maximum lines to return before truncating (default 2000).
            max_bytes: Maximum bytes to return before truncating (default 50KB).
            shell_timeout: Timeout in seconds for shell commands (default 120).
            enable_read_file: Enable the read_file tool.
            enable_edit_file: Enable the edit_file tool.
            enable_write_file: Enable the write_file tool.
            enable_run_shell: Enable the run_shell tool.
            enable_grep: Enable the grep tool (disabled by default).
            enable_find: Enable the find tool (disabled by default).
            enable_ls: Enable the ls tool (disabled by default).
            instructions: Custom instructions for the LLM. Uses defaults if None.
            add_instructions: Whether to add instructions to the agent's system message.
            all: Enable all tools regardless of individual flags.
            allowed_commands: List of allowed shell command names when restrict_to_base_dir is True.
                Defaults to DEFAULT_ALLOWED_COMMANDS. Set to None explicitly after init to disable.
        """
        self.base_dir: Path = Path(base_dir).resolve() if base_dir else Path.cwd().resolve()
        self.restrict_to_base_dir = restrict_to_base_dir
        self.allowed_commands: Optional[List[str]] = (
            allowed_commands if allowed_commands is not None else self.DEFAULT_ALLOWED_COMMANDS
        )
        self.max_lines = max_lines
        self.max_bytes = max_bytes
        self.shell_timeout = shell_timeout
        self._temp_files: List[str] = []

        import atexit

        atexit.register(self._cleanup_temp_files)

        has_exploration = all or enable_grep or enable_find or enable_ls

        if instructions is None:
            resolved_instructions = self.DEFAULT_INSTRUCTIONS
            if has_exploration:
                resolved_instructions += self.EXPLORATION_INSTRUCTIONS
        else:
            resolved_instructions = instructions

        tools: List[Any] = []
        if all or enable_read_file:
            tools.append(self.read_file)
        if all or enable_edit_file:
            tools.append(self.edit_file)
        if all or enable_write_file:
            tools.append(self.write_file)
        if all or enable_run_shell:
            tools.append(self.run_shell)
        if all or enable_grep:
            tools.append(self.grep)
        if all or enable_find:
            tools.append(self.find)
        if all or enable_ls:
            tools.append(self.ls)

        super().__init__(
            name="coding_tools",
            tools=tools,
            instructions=resolved_instructions,
            add_instructions=add_instructions,
            **kwargs,
        )

    def _truncate_output(self, text: str) -> tuple:
        """Truncate text to configured limits.

        Returns:
            Tuple of (possibly truncated text, was_truncated, total_line_count).
        """
        lines = text.split("\n")
        total_lines = len(lines)
        was_truncated = False

        if total_lines > self.max_lines:
            lines = lines[: self.max_lines]
            was_truncated = True

        result = "\n".join(lines)

        if len(result.encode("utf-8", errors="replace")) > self.max_bytes:
            # Truncate by bytes: find last complete line within limit
            truncated_lines = []
            current_bytes = 0
            for line in lines:
                line_bytes = len((line + "\n").encode("utf-8", errors="replace"))
                if current_bytes + line_bytes > self.max_bytes:
                    break
                truncated_lines.append(line)
                current_bytes += line_bytes
            result = "\n".join(truncated_lines)
            was_truncated = True

        return result, was_truncated, total_lines

    def _cleanup_temp_files(self) -> None:
        """Remove temporary files created during shell output truncation."""
        for path in self._temp_files:
            try:
                Path(path).unlink(missing_ok=True)
            except OSError:
                pass
        self._temp_files.clear()

    # Shell operators that enable command chaining or substitution
    _DANGEROUS_PATTERNS: List[str] = ["&&", "||", ";", "|", "$(", "`", ">", ">>", "<"]

    def _check_command(self, command: str) -> Optional[str]:
        """Check if a shell command is safe to execute.

        When restrict_to_base_dir is True, this method:
        1. Blocks shell metacharacters that enable chaining/substitution.
        2. Validates the command name against the allowed_commands list (if set).
        3. Checks that path-like tokens don't escape the base directory.

        Returns an error message if a violation is found, None if safe.
        """
        if not self.restrict_to_base_dir:
            return None

        # Block shell operators that enable chaining/substitution
        for pattern in self._DANGEROUS_PATTERNS:
            if pattern in command:
                return f"Error: Shell operator '{pattern}' is not allowed in restricted mode."

        try:
            tokens = shlex.split(command)
        except ValueError:
            return "Error: Could not parse shell command."

        # Validate command against allowlist
        if self.allowed_commands is not None and tokens:
            cmd = tokens[0]
            cmd_base = Path(cmd).name  # Handle /usr/bin/python -> python
            if cmd_base not in self.allowed_commands:
                return f"Error: Command '{cmd_base}' is not in the allowed commands list."

        for i, token in enumerate(tokens):
            # Skip the command itself (already validated by allowlist above)
            if i == 0:
                continue
            # Skip flags
            if token.startswith("-"):
                continue

            # Check tokens that look like paths
            if "/" in token or token == "..":
                try:
                    # Resolve relative to base_dir
                    if token.startswith("/"):
                        resolved = Path(token).resolve()
                    else:
                        resolved = (self.base_dir / token).resolve()

                    # Check if resolved path is within base_dir
                    try:
                        resolved.relative_to(self.base_dir)
                    except ValueError:
                        return f"Error: Command references path outside base directory: {token}"
                except (OSError, RuntimeError):
                    continue

        return None

    def read_file(self, file_path: str, offset: int = 0, limit: Optional[int] = None) -> str:
        """Read the contents of a file with line numbers.

        Returns file contents with line numbers prefixed to each line. Supports
        pagination via offset and limit for large files. Output is truncated if
        it exceeds the configured limits.

        :param file_path: Path to the file to read (relative to base_dir or absolute).
        :param offset: Line number to start reading from (0-indexed). Default 0.
        :param limit: Maximum number of lines to read. Defaults to max_lines setting.
        :return: File contents with line numbers, or an error message.
        """
        try:
            safe, resolved_path = self._check_path(file_path, self.base_dir, self.restrict_to_base_dir)
            if not safe:
                return f"Error: Path '{file_path}' is outside the allowed base directory"

            if not resolved_path.exists():
                return f"Error: File not found: {file_path}"

            if not resolved_path.is_file():
                return f"Error: Not a file: {file_path}"

            # Detect binary files
            try:
                with open(resolved_path, "rb") as f:
                    chunk = f.read(8192)
                    if b"\x00" in chunk:
                        return f"Error: Binary file detected: {file_path}"
            except Exception:
                pass

            contents = resolved_path.read_text(encoding="utf-8", errors="replace")

            if not contents:
                return f"File is empty: {file_path}"

            lines = contents.split("\n")
            total_lines = len(lines)

            # Apply offset and limit
            effective_limit = limit if limit is not None else self.max_lines
            selected_lines = lines[offset : offset + effective_limit]

            # Format with line numbers
            # Calculate width for line number alignment
            max_line_num = offset + len(selected_lines)
            num_width = max(len(str(max_line_num)), 4)

            formatted_lines = []
            for i, line in enumerate(selected_lines):
                line_num = offset + i + 1  # 1-based
                formatted_lines.append(f"{line_num:>{num_width}} | {line}")

            output = "\n".join(formatted_lines)

            # Apply truncation
            output, was_truncated, _ = self._truncate_output(output)

            # Add summary footer
            shown_start = offset + 1
            shown_end = offset + len(selected_lines)
            if was_truncated or shown_end < total_lines or offset > 0:
                output += f"\n[Showing lines {shown_start}-{shown_end} of {total_lines} total]"

            return output

        except UnicodeDecodeError:
            return f"Error: Cannot decode file as text: {file_path}"
        except PermissionError:
            return f"Error: Permission denied: {file_path}"
        except Exception as e:
            log_error(f"Error reading file: {e}")
            return f"Error reading file: {e}"

    def edit_file(self, file_path: str, old_text: str, new_text: str) -> str:
        """Edit a file by replacing an exact text match with new text.

        The old_text must match exactly one location in the file. If it matches
        zero or multiple locations, the edit is rejected with an error message.
        Returns a unified diff showing the change.

        :param file_path: Path to the file to edit (relative to base_dir or absolute).
        :param old_text: The exact text to find and replace. Must match uniquely.
        :param new_text: The text to replace old_text with.
        :return: A unified diff of the change, or an error message.
        """
        try:
            safe, resolved_path = self._check_path(file_path, self.base_dir, self.restrict_to_base_dir)
            if not safe:
                return f"Error: Path '{file_path}' is outside the allowed base directory"

            if not resolved_path.exists():
                return f"Error: File not found: {file_path}"

            if not resolved_path.is_file():
                return f"Error: Not a file: {file_path}"

            if not old_text:
                return "Error: old_text cannot be empty"

            if old_text == new_text:
                return "No changes needed: old_text and new_text are identical"

            contents = resolved_path.read_text(encoding="utf-8")

            # Count occurrences
            count = contents.count(old_text)

            if count == 0:
                return (
                    f"Error: old_text not found in {file_path}. "
                    "Make sure the text matches exactly (including whitespace and indentation)."
                )

            if count > 1:
                return (
                    f"Error: old_text matches {count} locations in {file_path}. "
                    "Provide more surrounding context to make the match unique."
                )

            # Perform the replacement
            new_contents = contents.replace(old_text, new_text, 1)

            # Write the file
            resolved_path.write_text(new_contents, encoding="utf-8")

            # Generate unified diff
            import difflib

            old_lines = contents.splitlines(keepends=True)
            new_lines = new_contents.splitlines(keepends=True)

            diff = difflib.unified_diff(
                old_lines,
                new_lines,
                fromfile=f"a/{file_path}",
                tofile=f"b/{file_path}",
                n=3,
            )
            diff_output = "".join(diff)

            if not diff_output:
                return "Edit applied but no visible diff generated"

            # Truncate if needed
            diff_output, was_truncated, total_lines = self._truncate_output(diff_output)
            if was_truncated:
                diff_output += f"\n[Diff truncated: {total_lines} lines total]"

            log_info(f"Edited {file_path}")
            return diff_output

        except PermissionError:
            return f"Error: Permission denied: {file_path}"
        except Exception as e:
            log_error(f"Error editing file: {e}")
            return f"Error editing file: {e}"

    def write_file(self, file_path: str, contents: str) -> str:
        """Create or overwrite a file with the given contents.

        Parent directories are created automatically if they do not exist.

        :param file_path: Path to the file to write (relative to base_dir or absolute).
        :param contents: The full contents to write to the file.
        :return: A success message with the file path, or an error message.
        """
        try:
            safe, resolved_path = self._check_path(file_path, self.base_dir, self.restrict_to_base_dir)
            if not safe:
                return f"Error: Path '{file_path}' is outside the allowed base directory"

            # Create parent directories
            if not resolved_path.parent.exists():
                resolved_path.parent.mkdir(parents=True, exist_ok=True)

            resolved_path.write_text(contents, encoding="utf-8")

            line_count = len(contents.split("\n"))
            log_info(f"Wrote {file_path}")
            return f"Wrote {line_count} lines to {file_path}"

        except PermissionError:
            return f"Error: Permission denied: {file_path}"
        except Exception as e:
            log_error(f"Error writing file: {e}")
            return f"Error writing file: {e}"

    def run_shell(self, command: str, timeout: Optional[int] = None) -> str:
        """Execute a shell command and return its output.

        Runs the command as a string via the system shell. Output (stdout + stderr)
        is truncated if it exceeds the configured limits. When output is truncated,
        the full output is saved to a temporary file and its path is included in
        the response.

        :param command: The shell command to execute as a single string.
        :param timeout: Timeout in seconds. Defaults to the toolkit's shell_timeout.
        :return: Command output (stdout and stderr combined), or an error message.
        """
        try:
            _warn_coding_tools()
            log_info(f"Running shell command: {command}")

            # Check for path escapes in command
            path_error = self._check_command(command)
            if path_error:
                return path_error

            effective_timeout = timeout if timeout is not None else self.shell_timeout

            result = subprocess.run(
                command,
                shell=True,
                capture_output=True,
                text=True,
                timeout=effective_timeout,
                cwd=str(self.base_dir),
            )

            # Combine stdout and stderr
            output = result.stdout
            if result.stderr:
                output += result.stderr

            header = f"Exit code: {result.returncode}\n"

            # Apply truncation
            truncated_output, was_truncated, total_lines = self._truncate_output(output)

            if was_truncated:
                # Save full output to temp file
                tmp = tempfile.NamedTemporaryFile(
                    mode="w",
                    delete=False,
                    suffix=".txt",
                    prefix="coding_tools_",
                )
                tmp.write(output)
                tmp.close()
                self._temp_files.append(tmp.name)
                truncated_output += f"\n[Output truncated: {total_lines} lines total. Full output saved to: {tmp.name}]"

            return header + truncated_output

        except subprocess.TimeoutExpired:
            effective_timeout = timeout if timeout is not None else self.shell_timeout
            return f"Error: Command timed out after {effective_timeout} seconds"
        except Exception as e:
            log_error(f"Error running shell command: {e}")
            return f"Error running shell command: {e}"

    def grep(
        self,
        pattern: str,
        path: Optional[str] = None,
        ignore_case: bool = False,
        include: Optional[str] = None,
        context: int = 0,
        limit: int = 100,
    ) -> str:
        """Search file contents for a pattern.

        Returns matching lines with file paths and line numbers. Respects
        .gitignore when using grep -r. Output is truncated if it exceeds limits.

        :param pattern: Search pattern (regex by default).
        :param path: Directory or file to search in (default: base directory).
        :param ignore_case: Case-insensitive search (default: False).
        :param include: Filter files by glob pattern, e.g. '*.py'.
        :param context: Number of lines to show before and after each match (default: 0).
        :param limit: Maximum number of matches to return (default: 100).
        :return: Matching lines with file paths and line numbers, or an error message.
        """
        try:
            if not pattern:
                return "Error: Pattern cannot be empty"

            # Resolve search path
            if path:
                safe, resolved_path = self._check_path(path, self.base_dir, self.restrict_to_base_dir)
                if not safe:
                    return f"Error: Path '{path}' is outside the allowed base directory"
            else:
                resolved_path = self.base_dir

            if not resolved_path.exists():
                return f"Error: Path not found: {path or '.'}"

            # Build grep command
            cmd = ["grep", "-rn"]
            if ignore_case:
                cmd.append("-i")
            if context > 0:
                cmd.extend(["-C", str(context)])
            if include:
                cmd.extend(["--include", include])

            cmd.append(pattern)
            cmd.append(str(resolved_path))

            result = subprocess.run(
                cmd,
                capture_output=True,
                text=True,
                timeout=30,
                cwd=str(self.base_dir),
            )

            output = result.stdout
            if not output:
                if result.returncode == 1:
                    return f"No matches found for pattern: {pattern}"
                if result.stderr:
                    return f"Error: {result.stderr.strip()}"
                return f"No matches found for pattern: {pattern}"

            # Make paths relative to base_dir
            base_str = str(self.base_dir) + "/"
            output = output.replace(base_str, "")

            # Enforce global match limit
            output_lines = output.split("\n")
            if len(output_lines) > limit:
                output = "\n".join(output_lines[:limit])
                output += f"\n[Results limited to {limit} matches]"

            # Apply truncation
            output, was_truncated, total_lines = self._truncate_output(output)
            if was_truncated:
                output += f"\n[Output truncated: {total_lines} lines total]"

            return output

        except subprocess.TimeoutExpired:
            return "Error: grep timed out after 30 seconds"
        except FileNotFoundError:
            return "Error: grep command not found. Install grep to use this tool."
        except Exception as e:
            log_error(f"Error running grep: {e}")
            return f"Error running grep: {e}"

    def find(self, pattern: str, path: Optional[str] = None, limit: int = 500) -> str:
        """Search for files by glob pattern.

        Returns matching file paths relative to the search directory.

        :param pattern: Glob pattern to match files, e.g. '*.py', '**/*.json'.
        :param path: Directory to search in (default: base directory).
        :param limit: Maximum number of results (default: 500).
        :return: Matching file paths, one per line, or an error message.
        """
        try:
            if not pattern:
                return "Error: Pattern cannot be empty"

            # Resolve search path
            if path:
                safe, resolved_path = self._check_path(path, self.base_dir, self.restrict_to_base_dir)
                if not safe:
                    return f"Error: Path '{path}' is outside the allowed base directory"
            else:
                resolved_path = self.base_dir

            if not resolved_path.exists():
                return f"Error: Path not found: {path or '.'}"

            if not resolved_path.is_dir():
                return f"Error: Not a directory: {path}"

            # Use pathlib glob
            matches = []
            for match in resolved_path.glob(pattern):
                try:
                    rel_path = match.relative_to(self.base_dir)
                    suffix = "/" if match.is_dir() else ""
                    matches.append(str(rel_path) + suffix)
                except ValueError:
                    continue  # Skip paths outside base_dir

                if len(matches) >= limit:
                    break

            if not matches:
                return f"No files found matching pattern: {pattern}"

            result = "\n".join(sorted(matches))

            footer = ""
            if len(matches) >= limit:
                footer = f"\n[Results limited to {limit} entries]"

            return result + footer

        except Exception as e:
            log_error(f"Error finding files: {e}")
            return f"Error finding files: {e}"

    def ls(self, path: Optional[str] = None, limit: int = 500) -> str:
        """List directory contents.

        Returns entries sorted alphabetically with '/' suffix for directories.
        Includes dotfiles.

        :param path: Directory to list (default: base directory).
        :param limit: Maximum number of entries to return (default: 500).
        :return: Directory listing, one entry per line, or an error message.
        """
        try:
            # Resolve path
            if path:
                safe, resolved_path = self._check_path(path, self.base_dir, self.restrict_to_base_dir)
                if not safe:
                    return f"Error: Path '{path}' is outside the allowed base directory"
            else:
                resolved_path = self.base_dir

            if not resolved_path.exists():
                return f"Error: Path not found: {path or '.'}"

            if not resolved_path.is_dir():
                return f"Error: Not a directory: {path}"

            entries = []
            for entry in sorted(resolved_path.iterdir(), key=lambda p: p.name.lower()):
                suffix = "/" if entry.is_dir() else ""
                entries.append(entry.name + suffix)
                if len(entries) >= limit:
                    break

            if not entries:
                return f"Directory is empty: {path or '.'}"

            result = "\n".join(entries)

            if len(entries) >= limit:
                result += f"\n[Listing limited to {limit} entries]"

            return result

        except PermissionError:
            return f"Error: Permission denied: {path or '.'}"
        except Exception as e:
            log_error(f"Error listing directory: {e}")
            return f"Error listing directory: {e}"
```

---

### DockerTools (`agno.tools.docker`)
Manage Docker containers, images, volumes, and networks.

**Dependencies**: `pip install docker`
**Notes**: Requires Docker daemon to be running.

#### Parameters
- Supports all standard Docker operations (run, exec, logs, pull, build, etc.)

#### Source Code
```python
import json
import os
import sys
from typing import Any, Dict, List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import logger

if sys.version_info >= (3, 12):
    # Apply more comprehensive monkey patch for Python 3.12 compatibility
    try:
        import inspect

        from docker import auth

        # Create a more comprehensive patched version that ignores any unknown parameters
        original_load_config = auth.load_config

        def patched_load_config(*args, **kwargs):
            # Get the original function's parameters
            try:
                sig = inspect.signature(original_load_config)
                # Filter out any kwargs that aren't in the signature
                valid_kwargs = {k: v for k, v in kwargs.items() if k in sig.parameters}
                return original_load_config(*args, **valid_kwargs)
            except Exception as e:
                logger.warning(f"Error in patched_load_config: {e}")
                return {}

        # Replace the original function with our patched version
        auth.load_config = patched_load_config

        # Add the missing get_config_header function
        if not hasattr(auth, "get_config_header"):

            def get_config_header(client, registry=None):
                """
                Replacement for missing get_config_header function.
                Returns empty auth headers to avoid authentication errors.
                """
                return {}

            # Add the function to the auth module
            auth.get_config_header = get_config_header
            logger.info("Added missing get_config_header function for Docker auth compatibility")

        logger.info("Applied comprehensive compatibility patch for Docker client on Python 3.12")
    except Exception as e:
        logger.warning(f"Failed to apply Docker client compatibility patch: {e}")

try:
    import docker
    from docker.errors import DockerException, ImageNotFound
except ImportError:
    raise ImportError("The `docker` package is not installed. Please install it via `pip install docker`.")


class DockerTools(Toolkit):
    def __init__(
        self,
        **kwargs,
    ):
        self._check_docker_availability()

        try:
            os.environ["DOCKER_CONFIG"] = ""

            if hasattr(self, "socket_path"):
                socket_url = f"unix://{self.socket_path}"
                self.client = docker.DockerClient(base_url=socket_url)
            else:
                self.client = docker.DockerClient()

            self.client.ping()
            logger.info("Successfully connected to Docker daemon")
        except Exception as e:
            logger.error(f"Error connecting to Docker: {e}")

        tools: List[Any] = [
            # Container management
            self.list_containers,
            self.start_container,
            self.stop_container,
            self.remove_container,
            self.get_container_logs,
            self.inspect_container,
            self.run_container,
            self.exec_in_container,
            # Image management
            self.list_images,
            self.pull_image,
            self.remove_image,
            self.build_image,
            self.tag_image,
            self.inspect_image,
            # Volume management
            self.list_volumes,
            self.create_volume,
            self.remove_volume,
            self.inspect_volume,
            # Network management
            self.list_networks,
            self.create_network,
            self.remove_network,
            self.inspect_network,
            self.connect_container_to_network,
            self.disconnect_container_from_network,
        ]

        super().__init__(name="docker_tools", tools=tools, **kwargs)

    def _check_docker_availability(self):
        """Check if Docker socket exists and is accessible."""
        # Common Docker socket paths
        socket_paths = [
            # Linux/macOS
            "/var/run/docker.sock",
            # macOS Docker Desktop
            os.path.expanduser("~/.docker/run/docker.sock"),
            # macOS newer versions
            os.path.join(os.path.expanduser("~"), ".docker", "desktop", "docker.sock"),
            # macOS alternative
            os.path.expanduser("~/Library/Containers/com.docker.docker/Data/docker.sock"),
            # Windows
            os.path.join("\\", "\\", ".", "pipe", "docker_engine"),
        ]

        # Check if any socket exists
        socket_exists = any(os.path.exists(path) for path in socket_paths)
        if not socket_exists:
            logger.error("Docker socket not found. Is Docker installed and running?")
            raise ValueError(
                "Docker socket not found. Please make sure Docker is installed and running.\n"
                "On macOS: Start Docker Desktop application.\n"
                "On Linux: Run 'sudo systemctl start docker'."
            )

        # Find the first available socket path
        for path in socket_paths:
            if os.path.exists(path):
                logger.info(f"Found Docker socket at {path}")
                self.socket_path = path
                return

    def list_containers(self, all: bool = False) -> str:
        """
        List Docker containers.

        Args:
            all (bool): If True, show all containers (default shows just running).

        Returns:
            str: A JSON string containing the list of containers.
        """
        try:
            containers = self.client.containers.list(all=all)
            container_list = []

            for container in containers:
                # Handle cases where container image might not have tags
                image_info = container.image.tags[0] if container.image.tags else container.image.id

                container_list.append(
                    {
                        "id": container.id,
                        "name": container.name,
                        "image": image_info,
                        "status": container.status,
                        "created": container.attrs.get("Created"),
                        "ports": container.ports,
                        "labels": container.labels,
                    }
                )

            return json.dumps(container_list, indent=2)
        except DockerException as e:
            error_msg = f"Error listing containers: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def start_container(self, container_id: str) -> str:
        """
        Start a Docker container.

        Args:
            container_id (str): The ID or name of the container to start.

        Returns:
            str: A success message or error message.
        """
        try:
            container = self.client.containers.get(container_id)
            container.start()
            return f"Container {container_id} started successfully"
        except DockerException as e:
            error_msg = f"Error starting container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def stop_container(self, container_id: str, timeout: int = 10) -> str:
        """
        Stop a Docker container.

        Args:
            container_id (str): The ID or name of the container to stop.
            timeout (int): Timeout in seconds to wait for container to stop.

        Returns:
            str: A success message or error message.
        """
        try:
            container = self.client.containers.get(container_id)
            container.stop(timeout=timeout)
            return f"Container {container_id} stopped successfully"
        except DockerException as e:
            error_msg = f"Error stopping container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def remove_container(self, container_id: str, force: bool = False, volumes: bool = False) -> str:
        """
        Remove a Docker container.

        Args:
            container_id (str): The ID or name of the container to remove.
            force (bool): If True, force the removal of a running container.
            volumes (bool): If True, remove anonymous volumes associated with the container.

        Returns:
            str: A success message or error message.
        """
        try:
            container = self.client.containers.get(container_id)
            container.remove(force=force, v=volumes)
            return f"Container {container_id} removed successfully"
        except DockerException as e:
            error_msg = f"Error removing container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def get_container_logs(self, container_id: str, tail: int = 100, stream: bool = False) -> str:
        """
        Get logs from a Docker container.

        Args:
            container_id (str): The ID or name of the container.
            tail (int): Number of lines to show from the end of the logs.
            stream (bool): If True, return a generator that yields log lines.

        Returns:
            str: The container logs or an error message.
        """
        try:
            container = self.client.containers.get(container_id)
            logs = container.logs(tail=tail, stream=stream)
            if isinstance(logs, bytes):
                return logs.decode("utf-8", errors="replace")
            # If streaming, we can't meaningfully return this as a string
            if stream:
                return "Logs are being streamed. This function returns data when stream=False."
            return "No logs found"
        except DockerException as e:
            error_msg = f"Error getting container logs: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def inspect_container(self, container_id: str) -> str:
        """
        Inspect a Docker container.

        Args:
            container_id (str): The ID or name of the container.

        Returns:
            str: A JSON string containing detailed information about the container.
        """
        try:
            container = self.client.containers.get(container_id)
            return json.dumps(container.attrs, indent=2)
        except DockerException as e:
            error_msg = f"Error inspecting container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def run_container(
        self,
        image: str,
        command: Optional[str] = None,
        name: Optional[str] = None,
        detach: bool = True,
        ports: Optional[Dict[str, Union[str, int]]] = None,  # Updated type hint
        volumes: Optional[Dict[str, Dict[str, str]]] = None,
        environment: Optional[Dict[str, str]] = None,
        network: Optional[str] = None,
    ) -> str:
        """
        Run a Docker container.

        Args:
            image (str): The image to run.
            command (str, optional): The command to run in the container.
            name (str, optional): A name for the container.
            detach (bool): Run container in the background.
            ports (dict, optional): Port mappings {'container_port/protocol': host_port}.
            volumes (dict, optional): Volume mappings.
            environment (dict, optional): Environment variables.
            network (str, optional): Network to connect the container to.

        Returns:
            str: Container ID or error message.
        """
        try:
            # Fix port mapping: convert integer values to strings
            if ports:
                fixed_ports = {}
                for container_port, host_port in ports.items():
                    if isinstance(host_port, int):
                        host_port = str(host_port)
                    fixed_ports[container_port] = host_port
            else:
                fixed_ports = None

            container = self.client.containers.run(
                image=image,
                command=command,
                name=name,
                detach=detach,
                ports=fixed_ports,  # Use the fixed ports
                volumes=volumes,
                environment=environment,
                network=network,
            )
            return f"Container started with ID: {container.id}"
        except DockerException as e:
            error_msg = f"Error running container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def exec_in_container(self, container_id: str, command: str) -> str:
        """
        Execute a command in a running container.

        Args:
            container_id (str): The ID or name of the container.
            command (str): The command to execute.

        Returns:
            str: Command output or error message.
        """
        try:
            container = self.client.containers.get(container_id)
            exit_code, output = container.exec_run(command)
            if isinstance(output, bytes):
                output_str = output.decode("utf-8", errors="replace")
            else:
                output_str = str(output)

            if exit_code == 0:
                return output_str
            else:
                return f"Command failed with exit code {exit_code}: {output_str}"
        except DockerException as e:
            error_msg = f"Error executing command in container: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def list_images(self) -> str:
        """
        List Docker images.

        Returns:
            str: A JSON string containing the list of images.
        """
        try:
            images = self.client.images.list()
            image_list = []

            for image in images:
                image_list.append(
                    {
                        "id": image.id,
                        "tags": image.tags,
                        "created": image.attrs.get("Created"),
                        "size": image.attrs.get("Size"),
                        "labels": image.labels,
                    }
                )

            return json.dumps(image_list, indent=2)
        except DockerException as e:
            error_msg = f"Error listing images: {str(e)}"
            logger.error(error_msg)
            return error_msg  # type: ignore

    def pull_image(self, image_name: str, tag: str = "latest") -> str:
        """
        Pull a Docker image.

        Args:
            image_name (str): The name of the image to pull.
            tag (str): The tag to pull.

        Returns:
            str: A success message or error message.
        """
        try:
            logger.info(f"Starting to pull image {image_name}:{tag}")
            for line in self.client.api.pull(image_name, tag=tag, stream=True, decode=True):
                if "progress" in line:
                    logger.info(f"Pulling {image_name}:{tag} - {line.get('progress', '')}")
                elif "status" in line:
                    logger.info(f"Pull status: {line.get('status', '')}")

            logger.info(f"Successfully pulled image {image_name}:{tag}")
            return f"Image {image_name}:{tag} pulled successfully"
        except Exception as e:
            error_msg = f"Error pulling image: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def remove_image(self, image_id: str, force: bool = False) -> str:
        """
        Remove a Docker image.

        Args:
            image_id (str): The ID or name of the image to remove.
            force (bool): If True, force removal of the image.

        Returns:
            str: A success message or error message.
        """
        try:
            self.client.images.remove(image_id, force=force)
            return f"Image {image_id} removed successfully"
        except ImageNotFound:
            return f"Image {image_id} not found"
        except DockerException as e:
            error_msg = f"Error removing image: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def build_image(self, path: str, tag: str, dockerfile: str = "Dockerfile", rm: bool = True) -> str:
        """
        Build a Docker image from a Dockerfile.

        Args:
            path (str): Path to the directory containing the Dockerfile.
            tag (str): Tag to apply to the built image.
            dockerfile (str): Name of the Dockerfile.
            rm (bool): Remove intermediate containers.

        Returns:
            str: A success message or error message.
        """
        try:
            image, logs = self.client.images.build(path=path, tag=tag, dockerfile=dockerfile, rm=rm)
            return f"Image built successfully with ID: {image.id}"
        except DockerException as e:
            error_msg = f"Error building image: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def tag_image(self, image_id: str, repository: str, tag: Optional[str] = None) -> str:
        """
        Tag a Docker image.

        Args:
            image_id (str): The ID or name of the image to tag.
            repository (str): The repository to tag in.
            tag (str, optional): The tag name.

        Returns:
            str: A success message or error message.
        """
        try:
            image = self.client.images.get(image_id)
            image.tag(repository, tag=tag)
            return f"Image {image_id} tagged as {repository}:{tag or 'latest'}"
        except DockerException as e:
            error_msg = f"Error tagging image: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def inspect_image(self, image_id: str) -> str:
        """
        Inspect a Docker image.

        Args:
            image_id (str): The ID or name of the image.

        Returns:
            str: A JSON string containing detailed information about the image.
        """
        try:
            image = self.client.images.get(image_id)
            return json.dumps(image.attrs, indent=2)
        except DockerException as e:
            error_msg = f"Error inspecting image: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def list_volumes(self) -> str:
        """
        List Docker volumes.

        Returns:
            str: A JSON string containing the list of volumes.
        """
        try:
            volumes = self.client.volumes.list()
            volume_list = []

            for volume in volumes:
                volume_list.append(
                    {
                        "name": volume.name,
                        "driver": volume.attrs.get("Driver"),
                        "mountpoint": volume.attrs.get("Mountpoint"),
                        "created": volume.attrs.get("CreatedAt"),
                        "labels": volume.attrs.get("Labels", {}),
                    }
                )

            return json.dumps(volume_list, indent=2)
        except DockerException as e:
            error_msg = f"Error listing volumes: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def create_volume(self, volume_name: str, driver: str = "local", labels: Optional[Dict[str, str]] = None) -> str:
        """
        Create a Docker volume.

        Args:
            volume_name (str): The name of the volume to create.
            driver (str): The volume driver to use.
            labels (dict, optional): Labels to apply to the volume.

        Returns:
            str: A success message or error message.
        """
        try:
            self.client.volumes.create(name=volume_name, driver=driver, labels=labels)
            return f"Volume {volume_name} created successfully"
        except DockerException as e:
            error_msg = f"Error creating volume: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def remove_volume(self, volume_name: str, force: bool = False) -> str:
        """
        Remove a Docker volume.

        Args:
            volume_name (str): The name of the volume to remove.
            force (bool): Force removal of the volume.

        Returns:
            str: A success message or error message.
        """
        try:
            volume = self.client.volumes.get(volume_name)
            volume.remove(force=force)
            return f"Volume {volume_name} removed successfully"
        except DockerException as e:
            error_msg = f"Error removing volume: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def inspect_volume(self, volume_name: str) -> str:
        """
        Inspect a Docker volume.

        Args:
            volume_name (str): The name of the volume.

        Returns:
            str: A JSON string containing detailed information about the volume.
        """
        try:
            volume = self.client.volumes.get(volume_name)
            return json.dumps(volume.attrs, indent=2)
        except DockerException as e:
            error_msg = f"Error inspecting volume: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def list_networks(self) -> str:
        """
        List Docker networks.

        Returns:
            str: A JSON string containing the list of networks.
        """
        try:
            networks = self.client.networks.list()
            network_list = []

            for network in networks:
                network_list.append(
                    {
                        "id": network.id,
                        "name": network.name,
                        "driver": network.attrs.get("Driver"),
                        "scope": network.attrs.get("Scope"),
                        "created": network.attrs.get("Created"),
                        "internal": network.attrs.get("Internal", False),
                        "containers": list(network.attrs.get("Containers", {}).keys()),
                    }
                )

            return json.dumps(network_list, indent=2)
        except DockerException as e:
            error_msg = f"Error listing networks: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def create_network(
        self, network_name: str, driver: str = "bridge", internal: bool = False, labels: Optional[Dict[str, str]] = None
    ) -> str:
        """
        Create a Docker network.

        Args:
            network_name (str): The name of the network to create.
            driver (str): The network driver to use.
            internal (bool): If True, create an internal network.
            labels (dict, optional): Labels to apply to the network.

        Returns:
            str: A success message or error message.
        """
        try:
            network = self.client.networks.create(name=network_name, driver=driver, internal=internal, labels=labels)
            return f"Network {network_name} created successfully with ID: {network.id}"
        except DockerException as e:
            error_msg = f"Error creating network: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def remove_network(self, network_name: str) -> str:
        """
        Remove a Docker network.

        Args:
            network_name (str): The name of the network to remove.

        Returns:
            str: A success message or error message.
        """
        try:
            network = self.client.networks.get(network_name)
            network.remove()
            return f"Network {network_name} removed successfully"
        except DockerException as e:
            error_msg = f"Error removing network: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def inspect_network(self, network_name: str) -> str:
        """
        Inspect a Docker network.

        Args:
            network_name (str): The name of the network.

        Returns:
            str: A JSON string containing detailed information about the network.
        """
        try:
            network = self.client.networks.get(network_name)
            return json.dumps(network.attrs, indent=2)
        except DockerException as e:
            error_msg = f"Error inspecting network: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def connect_container_to_network(self, container_id: str, network_name: str) -> str:
        """
        Connect a container to a network.

        Args:
            container_id (str): The ID or name of the container.
            network_name (str): The name of the network.

        Returns:
            str: A success message or error message.
        """
        try:
            network = self.client.networks.get(network_name)
            network.connect(container_id)
            return f"Container {container_id} connected to network {network_name}"
        except DockerException as e:
            error_msg = f"Error connecting container to network: {str(e)}"
            logger.error(error_msg)
            return error_msg

    def disconnect_container_from_network(self, container_id: str, network_name: str) -> str:
        """
        Disconnect a container from a network.

        Args:
            container_id (str): The ID or name of the container.
            network_name (str): The name of the network.

        Returns:
            str: A success message or error message.
        """
        try:
            network = self.client.networks.get(network_name)
            network.disconnect(container_id)
            return f"Container {container_id} disconnected from network {network_name}"
        except DockerException as e:
            error_msg = f"Error disconnecting container from network: {str(e)}"
            logger.error(error_msg)
            return error_msg
```

---

### ShellTools (`agno.tools.shell`)
Run shell commands and capture output.

#### Parameters
- `base_dir` (str): Directory to run commands in.

#### Source Code
```python
from pathlib import Path
from typing import List, Optional, Union

from agno.tools import Toolkit
from agno.utils.log import log_debug, log_info, logger


class ShellTools(Toolkit):
    def __init__(
        self,
        base_dir: Optional[Union[Path, str]] = None,
        enable_run_shell_command: bool = True,
        all: bool = False,
        **kwargs,
    ):
        self.base_dir: Optional[Path] = None
        if base_dir is not None:
            self.base_dir = Path(base_dir) if isinstance(base_dir, str) else base_dir

        tools = []
        if all or enable_run_shell_command:
            tools.append(self.run_shell_command)

        super().__init__(name="shell_tools", tools=tools, **kwargs)

    def run_shell_command(self, args: List[str], tail: int = 100) -> str:
        """Runs a shell command and returns the output or error.

        Args:
            args (List[str]): The command to run as a list of strings.
            tail (int): The number of lines to return from the output.

        Returns:
            str: The output of the command.
        """
        import subprocess

        try:
            log_info(f"Running shell command: {args}")
            result = subprocess.run(
                args,
                capture_output=True,
                text=True,
                cwd=str(self.base_dir) if self.base_dir else None,
            )
            log_debug(f"Result: {result}")
            log_debug(f"Return code: {result.returncode}")
            if result.returncode != 0:
                return f"Error: {result.stderr}"
            return "\n".join(result.stdout.split("\n")[-tail:])
        except Exception as e:
            logger.warning(f"Failed to run shell command: {e}")
            return f"Error: {e}"
```


## 22. Special Purpose (Detailed)

### ParallelTools (`agno.tools.parallel`)
Access Parallel's AI-optimized web search and content extraction APIs.

**Authentication**: Requires `PARALLEL_API_KEY`.
**Dependencies**: `pip install parallel-web`

#### Parameters
- `max_results` (int): Default 10.
- `max_chars_per_result` (int): Default 10000.

#### Source Code
```python
import json
from os import getenv
from typing import Any, Dict, List, Optional

from agno.tools import Toolkit
from agno.utils.log import log_error

try:
    from parallel import Parallel as ParallelClient
except ImportError:
    raise ImportError("`parallel-web` not installed. Please install using `pip install parallel-web`")


class CustomJSONEncoder(json.JSONEncoder):
    """Custom JSON encoder that handles non-serializable types by converting them to strings."""

    def default(self, obj):
        try:
            return super().default(obj)
        except TypeError:
            return str(obj)


class ParallelTools(Toolkit):
    """
    ParallelTools provides access to Parallel's web search and extraction APIs.

    Parallel offers powerful APIs optimized for AI agents:
    - Search API: AI-optimized web search that returns relevant excerpts tailored for LLMs
    - Extract API: Extract content from specific URLs in clean markdown format, handling JavaScript-heavy pages and PDFs

    Args:
        api_key (Optional[str]): Parallel API key. If not provided, will use PARALLEL_API_KEY environment variable.
        enable_search (bool): Enable Search API functionality. Default is True.
        enable_extract (bool): Enable Extract API functionality. Default is True.
        all (bool): Enable all tools. Overrides individual flags when True. Default is False.
        max_results (int): Default maximum number of results for search operations. Default is 10.
        max_chars_per_result (int): Default maximum characters per result for search operations. Default is 10000.
        beta_version (str): Beta API version header. Default is "search-extract-2025-10-10".
        mode (Optional[str]): Default search mode. Options: "one-shot" or "agentic". Default is None.
        include_domains (Optional[List[str]]): Default domains to restrict results to. Default is None.
        exclude_domains (Optional[List[str]]): Default domains to exclude from results. Default is None.
        max_age_seconds (Optional[int]): Default cache age threshold (minimum 600). Default is None.
        disable_cache_fallback (Optional[bool]): Default cache fallback behavior. Default is None.
    """

    def __init__(
        self,
        api_key: Optional[str] = None,
        enable_search: bool = True,
        enable_extract: bool = True,
        all: bool = False,
        max_results: int = 10,
        max_chars_per_result: int = 10000,
        beta_version: str = "search-extract-2025-10-10",
        mode: Optional[str] = None,
        include_domains: Optional[List[str]] = None,
        exclude_domains: Optional[List[str]] = None,
        max_age_seconds: Optional[int] = None,
        disable_cache_fallback: Optional[bool] = None,
        **kwargs,
    ):
        self.api_key: Optional[str] = api_key or getenv("PARALLEL_API_KEY")
        if not self.api_key:
            log_error("PARALLEL_API_KEY not set. Please set the PARALLEL_API_KEY environment variable.")

        self.max_results = max_results
        self.max_chars_per_result = max_chars_per_result
        self.beta_version = beta_version
        self.mode = mode
        self.include_domains = include_domains
        self.exclude_domains = exclude_domains
        self.max_age_seconds = max_age_seconds
        self.disable_cache_fallback = disable_cache_fallback

        self.parallel_client = ParallelClient(
            api_key=self.api_key, default_headers={"parallel-beta": self.beta_version}
        )

        tools: List[Any] = []
        if all or enable_search:
            tools.append(self.parallel_search)
        if all or enable_extract:
            tools.append(self.parallel_extract)

        super().__init__(name="parallel_tools", tools=tools, **kwargs)

    def parallel_search(
        self,
        objective: Optional[str] = None,
        search_queries: Optional[List[str]] = None,
        max_results: Optional[int] = None,
        max_chars_per_result: Optional[int] = None,
    ) -> str:
        """Use this function to search the web using Parallel's Search API with a natural language objective.
        You must provide at least one of objective or search_queries.

        Args:
            objective (Optional[str]): Natural-language description of what the web search is trying to find.
            search_queries (Optional[List[str]]): Traditional keyword queries with optional search operators.
            max_results (Optional[int]): Upper bound on results returned. Overrides constructor default.
            max_chars_per_result (Optional[int]): Upper bound on total characters per url for excerpts.

        Returns:
            str: A JSON formatted string containing the search results with URLs, titles, publish dates, and relevant excerpts.
        """
        try:
            if not objective and not search_queries:
                return json.dumps({"error": "Please provide at least one of: objective or search_queries"}, indent=2)

            # Use instance defaults if not provided
            final_max_results = max_results if max_results is not None else self.max_results

            search_params: Dict[str, Any] = {
                "max_results": final_max_results,
            }

            # Add objective if provided
            if objective:
                search_params["objective"] = objective

            # Add search_queries if provided
            if search_queries:
                search_params["search_queries"] = search_queries

            # Add mode from constructor default
            if self.mode:
                search_params["mode"] = self.mode

            # Add excerpts configuration
            excerpts_config: Dict[str, Any] = {}
            final_max_chars = max_chars_per_result if max_chars_per_result is not None else self.max_chars_per_result
            if final_max_chars is not None:
                excerpts_config["max_chars_per_result"] = final_max_chars

            if excerpts_config:
                search_params["excerpts"] = excerpts_config

            # Add source_policy from constructor defaults
            source_policy: Dict[str, Any] = {}
            if self.include_domains:
                source_policy["include_domains"] = self.include_domains
            if self.exclude_domains:
                source_policy["exclude_domains"] = self.exclude_domains

            if source_policy:
                search_params["source_policy"] = source_policy

            # Add fetch_policy from constructor defaults
            fetch_policy: Dict[str, Any] = {}
            if self.max_age_seconds is not None:
                fetch_policy["max_age_seconds"] = self.max_age_seconds
            if self.disable_cache_fallback is not None:
                fetch_policy["disable_cache_fallback"] = self.disable_cache_fallback

            if fetch_policy:
                search_params["fetch_policy"] = fetch_policy

            search_result = self.parallel_client.beta.search(**search_params)

            # Use model_dump() if available, otherwise convert to dict
            try:
                if hasattr(search_result, "model_dump"):
                    return json.dumps(search_result.model_dump(), cls=CustomJSONEncoder)
            except Exception:
                pass

            # Manually format the results
            formatted_results: Dict[str, Any] = {
                "search_id": getattr(search_result, "search_id", ""),
                "results": [],
            }

            if hasattr(search_result, "results") and search_result.results:
                results_list: List[Dict[str, Any]] = []
                for result in search_result.results:
                    formatted_result: Dict[str, Any] = {
                        "title": getattr(result, "title", ""),
                        "url": getattr(result, "url", ""),
                        "publish_date": getattr(result, "publish_date", ""),
                        "excerpt": getattr(result, "excerpt", ""),
                    }
                    results_list.append(formatted_result)
                formatted_results["results"] = results_list

            if hasattr(search_result, "warnings"):
                formatted_results["warnings"] = search_result.warnings

            if hasattr(search_result, "usage"):
                formatted_results["usage"] = search_result.usage

            return json.dumps(formatted_results, cls=CustomJSONEncoder, indent=2)

        except Exception as e:
            log_error(f"Error searching Parallel for objective '{objective}': {e}")
            return json.dumps({"error": f"Search failed: {str(e)}"}, indent=2)

    def parallel_extract(
        self,
        urls: List[str],
        objective: Optional[str] = None,
        search_queries: Optional[List[str]] = None,
        excerpts: bool = True,
        max_chars_per_excerpt: Optional[int] = None,
        full_content: bool = False,
        max_chars_for_full_content: Optional[int] = None,
    ) -> str:
        """Use this function to extract content from specific URLs using Parallel's Extract API.

        Args:
            urls (List[str]): List of public URLs to extract content from.
            objective (Optional[str]): Search focus to guide content extraction.
            search_queries (Optional[List[str]]): Keywords for targeting relevant content.
            excerpts (bool): Include relevant text snippets.
            max_chars_per_excerpt (Optional[int]): Upper bound on total characters per url. Only used when excerpts is True.
            full_content (bool): Include complete page text.
            max_chars_for_full_content (Optional[int]): Limit on characters per url. Only used when full_content is True.

        Returns:
            str: A JSON formatted string containing extracted content with titles, publish dates, excerpts and/or full content.
        """
        try:
            if not urls:
                return json.dumps({"error": "Please provide at least one URL to extract"}, indent=2)

            extract_params: Dict[str, Any] = {
                "urls": urls,
            }

            # Add objective if provided
            if objective:
                extract_params["objective"] = objective

            # Add search_queries if provided
            if search_queries:
                extract_params["search_queries"] = search_queries

            # Add excerpts configuration
            if excerpts and max_chars_per_excerpt is not None:
                extract_params["excerpts"] = {"max_chars_per_result": max_chars_per_excerpt}
            else:
                extract_params["excerpts"] = excerpts

            # Add full_content configuration
            if full_content and max_chars_for_full_content is not None:
                extract_params["full_content"] = {"max_chars_per_result": max_chars_for_full_content}
            else:
                extract_params["full_content"] = full_content

            # Add fetch_policy from constructor defaults
            fetch_policy: Dict[str, Any] = {}
            if self.max_age_seconds is not None:
                fetch_policy["max_age_seconds"] = self.max_age_seconds
            if self.disable_cache_fallback is not None:
                fetch_policy["disable_cache_fallback"] = self.disable_cache_fallback

            if fetch_policy:
                extract_params["fetch_policy"] = fetch_policy

            extract_result = self.parallel_client.beta.extract(**extract_params)

            # Use model_dump() if available, otherwise convert to dict
            try:
                if hasattr(extract_result, "model_dump"):
                    return json.dumps(extract_result.model_dump(), cls=CustomJSONEncoder)
            except Exception:
                pass

            # Manually format the results
            formatted_results: Dict[str, Any] = {
                "extract_id": getattr(extract_result, "extract_id", ""),
                "results": [],
                "errors": [],
            }

            if hasattr(extract_result, "results") and extract_result.results:
                results_list: List[Dict[str, Any]] = []
                for result in extract_result.results:
                    formatted_result: Dict[str, Any] = {
                        "url": getattr(result, "url", ""),
                        "title": getattr(result, "title", ""),
                        "publish_date": getattr(result, "publish_date", ""),
                    }

                    if excerpts and hasattr(result, "excerpts"):
                        formatted_result["excerpts"] = result.excerpts

                    if full_content and hasattr(result, "full_content"):
                        formatted_result["full_content"] = result.full_content

                    results_list.append(formatted_result)
                formatted_results["results"] = results_list

            if hasattr(extract_result, "errors") and extract_result.errors:
                formatted_results["errors"] = extract_result.errors

            if hasattr(extract_result, "warnings"):
                formatted_results["warnings"] = extract_result.warnings

            if hasattr(extract_result, "usage"):
                formatted_results["usage"] = extract_result.usage

            return json.dumps(formatted_results, cls=CustomJSONEncoder, indent=2)

        except Exception as e:
            log_error(f"Error extracting from Parallel: {e}")
            return json.dumps({"error": f"Extract failed: {str(e)}"}, indent=2)
```

---

### UserControlFlowTools (`agno.tools.user_control_flow`)
Interrupt agent execution to request input from the user.

#### Parameters
- `user_input_fields` (list[dict]): Define fields (name, type, description) for the user to fill.

#### Source Code
```python
from textwrap import dedent
from typing import Optional

from agno.tools import Toolkit


class UserControlFlowTools(Toolkit):
    def __init__(
        self,
        instructions: Optional[str] = None,
        add_instructions: bool = True,
        enable_get_user_input: bool = True,
        all: bool = False,
        **kwargs,
    ):
        """A toolkit that provides the ability for the agent to interrupt the agent run and interact with the user."""

        if instructions is None:
            self.instructions = self.DEFAULT_INSTRUCTIONS
        else:
            self.instructions = instructions

        tools = []
        if all or enable_get_user_input:
            tools.append(self.get_user_input)

        super().__init__(
            name="user_control_flow_tools",
            instructions=self.instructions,
            add_instructions=add_instructions,
            tools=tools,
            **kwargs,
        )

    def get_user_input(self, user_input_fields: list[dict]) -> str:
        """Use this tool to get user input for the given fields. Provide all the fields that you require the user to fill in, as if they were filling in a form.

        Args:
            user_input_fields (list[dict[str, str]]): A list of dictionaries, each containing the following keys:
                - field_name: The name of the field to get input for.
                - field_type: The type of the field to get input for. Only valid python types are supported (e.g. str, int, float, bool, list, dict, etc.).
                - field_description: A description of the field to get input for.

        """
        # Nothing needs to be executed here, the agent logic will interrupt the run and wait for the user input
        return "User input received"

    # --------------------------------------------------------------------------------
    # Default instructions
    # --------------------------------------------------------------------------------

    DEFAULT_INSTRUCTIONS = dedent(
        """\
        You have access to the `get_user_input` tool to get user input for the given fields.

        1. **Get User Input**:
            - Purpose: When you have call a tool/function where you don't have enough information, don't say you can't do it, just use the `get_user_input` tool to get the information you need from the user.
            - Usage: Call `get_user_input` with the fields you require the user to fill in for you to continue your task.

        ## IMPORTANT GUIDELINES
        - **Don't respond and ask the user for information.** Just use the `get_user_input` tool to get the information you need from the user.
        - **Don't make up information you don't have.** If you don't have the information, use the `get_user_input` tool to get the information you need from the user.
        - **Include only the required fields.** Include only the required fields in the `user_input_fields` parameter of the `get_user_input` tool. Don't include fields you already have the information for.
        - **Provide a clear and concise description of the field.** Clearly describe the field in the `field_description` parameter of the `user_input_fields` parameter of the `get_user_input` tool.
        - **Provide a type for the field.** Fill the `field_type` parameter of the `user_input_fields` parameter of the `get_user_input` tool with the type of the field.

        ## INPUT VALIDATION AND CONVERSION
        - **Boolean fields**: Only explicit positive responses are considered True:
          * True values: 'true', 'yes', 'y', '1', 'on', 't', 'True', 'YES', 'Y', 'T'
          * False values: Everything else including 'false', 'no', 'n', '0', 'off', 'f', empty strings, unanswered fields, or any other input
          * **CRITICAL**: Empty/unanswered fields should be treated as False (not selected)
        - **Users can leave fields unanswered.** Empty responses are valid and should be treated as False for boolean fields.
        - **NEVER ask for the same field twice.** Once you receive ANY user input for a field (including empty strings), accept it and move on.
        - **DO NOT validate or re-request input.** Accept whatever the user provides and convert it appropriately.
        - **Proceed with only the fields that were explicitly answered as True.** Skip or ignore fields that are False/unanswered.
        - **Complete the task immediately after receiving all user inputs, do not ask for confirmation or re-validation.**
        """
    )
```

---

### ReasoningTools (`agno.tools.reasoning`)
Step-by-step reasoning tools (Think and Analyze) to structure the agent's internal monologue.

#### Parameters
- `enable_think` (bool): Default True.
- `enable_analyze` (bool): Default True.

#### Source Code
```python
from textwrap import dedent
from typing import Any, List, Optional

from agno.reasoning.step import NextAction, ReasoningStep
from agno.run import RunContext
from agno.tools import Toolkit
from agno.utils.log import log_debug, log_error


class ReasoningTools(Toolkit):
    def __init__(
        self,
        enable_think: bool = True,
        enable_analyze: bool = True,
        all: bool = False,
        instructions: Optional[str] = None,
        add_instructions: bool = False,
        add_few_shot: bool = False,
        few_shot_examples: Optional[str] = None,
        **kwargs,
    ):
        """A toolkit that provides step-by-step reasoning tools: Think and Analyze."""

        # Add instructions for using this toolkit
        if instructions is None:
            self.instructions = "<reasoning_instructions>\n" + self.DEFAULT_INSTRUCTIONS
            if add_few_shot:
                if few_shot_examples is not None:
                    self.instructions += "\n" + few_shot_examples
                else:
                    self.instructions += "\n" + self.FEW_SHOT_EXAMPLES
            self.instructions += "\n</reasoning_instructions>\n"
        else:
            self.instructions = instructions

        tools: List[Any] = []
        # Prefer new flags; fallback to legacy ones
        if all or enable_think:
            tools.append(self.think)
        if all or enable_analyze:
            tools.append(self.analyze)

        super().__init__(
            name="reasoning_tools",
            instructions=self.instructions,
            add_instructions=add_instructions,
            tools=tools,
            **kwargs,
        )

    def think(
        self,
        run_context: RunContext,
        title: str,
        thought: str,
        action: Optional[str] = None,
        confidence: float = 0.8,
    ) -> str:
        """Use this tool as a scratchpad to reason about the question and work through it step-by-step.
        This tool will help you break down complex problems into logical steps and track the reasoning process.
        You can call it as many times as needed. These internal thoughts are never revealed to the user.

        Args:
            title: A concise title for this step
            thought: Your detailed thought for this step
            action: What you'll do based on this thought
            confidence: How confident you are about this thought (0.0 to 1.0)

        Returns:
            A list of previous thoughts and the new thought
        """
        try:
            log_debug(f"Thought about {title}")

            # Create a reasoning step
            reasoning_step = ReasoningStep(
                title=title,
                reasoning=thought,
                action=action,
                next_action=NextAction.CONTINUE,
                confidence=confidence,
            )

            current_run_id = run_context.run_id

            # Add this step to the Agent's session state
            if run_context.session_state is None:
                run_context.session_state = {}
            if "reasoning_steps" not in run_context.session_state:
                run_context.session_state["reasoning_steps"] = {}
            if current_run_id not in run_context.session_state["reasoning_steps"]:
                run_context.session_state["reasoning_steps"][current_run_id] = []
            run_context.session_state["reasoning_steps"][current_run_id].append(reasoning_step.model_dump_json())

            # Return all previous reasoning_steps and the new reasoning_step
            if (
                "reasoning_steps" in run_context.session_state
                and current_run_id in run_context.session_state["reasoning_steps"]
            ):
                formatted_reasoning_steps = ""
                for i, step in enumerate(run_context.session_state["reasoning_steps"][current_run_id], 1):
                    step_parsed = ReasoningStep.model_validate_json(step)
                    step_str = dedent(f"""\
Step {i}:
Title: {step_parsed.title}
Reasoning: {step_parsed.reasoning}
Action: {step_parsed.action}
Confidence: {step_parsed.confidence}
""")
                    formatted_reasoning_steps += step_str + "\n"
                return formatted_reasoning_steps.strip()
            return reasoning_step.model_dump_json()
        except Exception as e:
            log_error(f"Error recording thought: {e}")
            return f"Error recording thought: {e}"

    def analyze(
        self,
        run_context: RunContext,
        title: str,
        result: str,
        analysis: str,
        next_action: str = "continue",
        confidence: float = 0.8,
    ) -> str:
        """Use this tool to analyze results from a reasoning step and determine next actions.

        Args:
            title: A concise title for this analysis step
            result: The outcome of the previous action
            analysis: Your analysis of the results
            next_action: What to do next ("continue", "validate", or "final_answer")
            confidence: How confident you are in this analysis (0.0 to 1.0)

        Returns:
            A list of previous thoughts and the new analysis
        """
        try:
            log_debug(f"Analyzed {title}")

            # Map string next_action to enum
            next_action_enum = NextAction.CONTINUE
            if next_action.lower() == "validate":
                next_action_enum = NextAction.VALIDATE
            elif next_action.lower() in ["final", "final_answer", "finalize"]:
                next_action_enum = NextAction.FINAL_ANSWER

            # Create a reasoning step for the analysis
            reasoning_step = ReasoningStep(
                title=title,
                result=result,
                reasoning=analysis,
                next_action=next_action_enum,
                confidence=confidence,
            )

            current_run_id = run_context.run_id
            # Add this step to the Agent's session state
            if run_context.session_state is None:
                run_context.session_state = {}
            if "reasoning_steps" not in run_context.session_state:
                run_context.session_state["reasoning_steps"] = {}
            if current_run_id not in run_context.session_state["reasoning_steps"]:
                run_context.session_state["reasoning_steps"][current_run_id] = []
            run_context.session_state["reasoning_steps"][current_run_id].append(reasoning_step.model_dump_json())

            # Return all previous reasoning_steps and the new reasoning_step
            if (
                "reasoning_steps" in run_context.session_state
                and current_run_id in run_context.session_state["reasoning_steps"]
            ):
                formatted_reasoning_steps = ""
                for i, step in enumerate(run_context.session_state["reasoning_steps"][current_run_id], 1):
                    step_parsed = ReasoningStep.model_validate_json(step)
                    step_str = dedent(f"""\
Step {i}:
Title: {step_parsed.title}
Reasoning: {step_parsed.reasoning}
Action: {step_parsed.action}
Confidence: {step_parsed.confidence}
""")
                    formatted_reasoning_steps += step_str + "\n"
                return formatted_reasoning_steps.strip()
            return reasoning_step.model_dump_json()
        except Exception as e:
            log_error(f"Error recording analysis: {e}")
            return f"Error recording analysis: {e}"

    # --------------------------------------------------------------------------------
    # Default instructions and few-shot examples
    # --------------------------------------------------------------------------------

    DEFAULT_INSTRUCTIONS = dedent(
        """\
        You have access to the `think` and `analyze` tools to work through problems step-by-step and structure your thought process. You must ALWAYS `think` before making tool calls or generating a response.

        1. **Think** (scratchpad):
            - Purpose: Use the `think` tool as a scratchpad to break down complex problems, outline steps, and decide on immediate actions within your reasoning flow. Use this to structure your internal monologue.
            - Usage: Call `think` before making tool calls or generating a response. Explain your reasoning and specify the intended action (e.g., "make a tool call", "perform calculation", "ask clarifying question").

        2. **Analyze** (evaluation):
            - Purpose: Evaluate the result of a think step or a set of tool calls. Assess if the result is expected, sufficient, or requires further investigation.
            - Usage: Call `analyze` after a set of tool calls. Determine the `next_action` based on your analysis: `continue` (more reasoning needed), `validate` (seek external confirmation/validation if possible), or `final_answer` (ready to conclude).
            - Explain your reasoning highlighting whether the result is correct/sufficient.

        ## IMPORTANT GUIDELINES
        - **Always Think First:** You MUST use the `think` tool before making tool calls or generating a response.
        - **Iterate to Solve:** Use the `think` and `analyze` tools iteratively to build a clear reasoning path. The typical flow is `Think` -> [`Tool Calls` if needed] -> [`Analyze` if needed] -> ... -> `final_answer`. Repeat this cycle until you reach a satisfactory conclusion.
        - **Make multiple tool calls in parallel:** After a `think` step, you can make multiple tool calls in parallel.
        - **Keep Thoughts Internal:** The reasoning steps (thoughts and analyses) are for your internal process only. Do not share them directly with the user.
        - **Conclude Clearly:** When your analysis determines the `next_action` is `final_answer`, provide a concise and accurate final answer to the user."""
    )

    FEW_SHOT_EXAMPLES = dedent(
        """
        Below are examples demonstrating how to use the `think` and `analyze` tools.

        ### Examples

        **Example 1: Simple Fact Retrieval**

        *User Request:* How many continents are there on Earth?

        *Agent's Internal Process:*

        ```tool_call
        think(
          title="Understand Request",
          thought="The user wants to know the standard number of continents on Earth. This is a common piece of knowledge.",
          action="Recall or verify the number of continents.",
          confidence=0.95
        )
        ```
        *--(Agent internally recalls the fact)--*
        ```tool_call
        analyze(
          title="Evaluate Fact",
          result="Standard geographical models list 7 continents: Africa, Antarctica, Asia, Australia, Europe, North America, South America.",
          analysis="The recalled information directly answers the user's question accurately.",
          next_action="final_answer",
          confidence=1.0
        )
        ```

        *Agent's Final Answer to User:*
        There are 7 continents on Earth: Africa, Antarctica, Asia, Australia, Europe, North America, and South America.

        **Example 2: Multi-Step Information Gathering**

        *User Request:* What is the capital of France and its current population?

        *Agent's Internal Process:*

        ```tool_call
        think(
          title="Plan Information Retrieval",
          thought="The user needs two pieces of information: the capital of France and its current population. I should use external tools (like search) to find the most up-to-date and accurate information.",
          action="First, search for the capital of France.",
          confidence=0.95
        )
        ```

        *Perform multiple tool calls in parallel*
        *--(Tool call 1: search(query="capital of France"))--*
        *--(Tool call 2: search(query="population of Paris current"))--*
        *--(Tool Result 1: "Paris")--*
        *--(Tool Result 2: "Approximately 2.1 million (city proper, estimate for early 2024)")--*

        ```tool_call
        analyze(
          title="Analyze Capital Search Result",
          result="The search result indicates Paris is the capital of France.",
          analysis="This provides the first piece of requested information. Now I need to find the population of Paris.",
          next_action="continue",
          confidence=1.0
        )
        ```
        ```tool_call
        analyze(
          title="Analyze Population Search Result",
          result="The search provided an estimated population figure for Paris.",
          analysis="I now have both the capital and its estimated population. I can provide the final answer.",
          next_action="final_answer",
          confidence=0.9
        )
        ```

        *Agent's Final Answer to User:*
        The capital of France is Paris. Its estimated population (city proper) is approximately 2.1 million as of early 2024."""
    )
```

---

### StreamlitComponents (`agno.tools.streamlit`)
Interact with Streamlit components and UI elements.

#### Source Code
```python
from os import environ, getenv
from typing import Optional

try:
    import streamlit as st
except ImportError:
    raise ImportError("`streamlit` library not installed. Please install using `pip install streamlit`")


def get_username_sidebar() -> Optional[str]:
    """Sidebar component to get username"""

    # Get username from user if not in session state
    if "username" not in st.session_state:
        username_input_container = st.sidebar.empty()
        username = username_input_container.text_input(":technologist: Enter username")
        if username != "":
            st.session_state["username"] = username
            username_input_container.empty()

    # Get username from session state
    username = st.session_state.get("username")  # type: ignore
    return username


def reload_button_sidebar(text: str = "Reload Session", **kwargs) -> None:
    """Sidebar component to show reload button"""

    if st.sidebar.button(text, **kwargs):
        st.session_state.clear()
        st.rerun()


def check_password(password_env_var: str = "APP_PASSWORD") -> bool:
    """Component to check if a password entered by the user is correct.
    To use this component, set the environment variable `APP_PASSWORD`.

    Args:
        password_env_var (str, optional): The environment variable to use for the password. Defaults to "APP_PASSWORD".

    Returns:
        bool: `True` if the user had the correct password.
    """

    app_password = getenv(password_env_var)
    if app_password is None:
        return True

    def check_first_run_password():
        """Checks whether a password entered on the first run is correct."""

        if "first_run_password" in st.session_state:
            password_to_check = st.session_state["first_run_password"]
            if password_to_check == app_password:
                st.session_state["password_correct"] = True
                # don't store password
                del st.session_state["first_run_password"]
            else:
                st.session_state["password_correct"] = False

    def check_updated_password():
        """Checks whether an updated password is correct."""

        if "updated_password" in st.session_state:
            password_to_check = st.session_state["updated_password"]
            if password_to_check == app_password:
                st.session_state["password_correct"] = True
                # don't store password
                del st.session_state["updated_password"]
            else:
                st.session_state["password_correct"] = False

    # First run, show input for password.
    if "password_correct" not in st.session_state:
        st.text_input(
            "Password",
            type="password",
            on_change=check_first_run_password,
            key="first_run_password",
        )
        return False
    # Password incorrect, show input for updated password + error.
    elif not st.session_state["password_correct"]:
        st.text_input(
            "Password",
            type="password",
            on_change=check_updated_password,
            key="updated_password",
        )
        st.error("😕 Password incorrect")
        return False
    # Password correct.
    else:
        return True


def get_openai_key_sidebar() -> Optional[str]:
    """Sidebar component to get OpenAI API key"""

    # Get OpenAI API key from environment variable
    openai_key: Optional[str] = getenv("OPENAI_API_KEY")
    # If not found, get it from user input
    if openai_key is None or openai_key == "" or openai_key == "sk-***":
        api_key = st.sidebar.text_input("OpenAI API key", placeholder="sk-***", key="api_key")
        if api_key != "sk-***" or api_key != "" or api_key is not None:
            openai_key = api_key

    # Store it in session state and environment variable
    if openai_key is not None and openai_key != "":
        st.session_state["OPENAI_API_KEY"] = openai_key
        environ["OPENAI_API_KEY"] = openai_key

    return openai_key
```

