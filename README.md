# Superset MCP

An MCP server that lets Claude (or any MCP client) work with Apache Superset: list and edit dashboards, charts and datasets, run SQL Lab queries, and manage databases and tags.

This is a fork of [aptro/superset-mcp](https://github.com/aptro/superset-mcp) with these changes:

- **Logs in the way the Superset UI does** (session cookie + CSRF), so it works on instances where the JWT API login is disabled, and falls back to JWT/refresh tokens where it is not.
- **Logs in automatically** on the first tool call, using the credentials in `.env`.
- **List tools return everything, not just the first 20 rows.** They walk every page, take `search` / `mine` / `max_results` filters, and report `count`, `returned` and `truncated` so the AI knows when a list was cut short.
- **Compact chart and dataset rows**, so listing a few thousand charts does not flood the AI's context. Use the `get_by_id` tools for full detail.

## Quick start

You need Python 3.10+, [uv](https://docs.astral.sh/uv/) and a Superset user. Upstream was tested on Superset 4.1.1.

```bash
git clone https://github.com/Jaskirat-2004/superset-mcp.git
cd superset-mcp
uv sync
cp .env.example .env   # then edit it
```

`.env`:

```
SUPERSET_BASE_URL=https://your-superset.example.com
SUPERSET_USERNAME=your_username
SUPERSET_PASSWORD=your_password
```

### Claude Code

Run from inside the cloned folder:

```bash
claude mcp add superset -- uv --directory "$(pwd)" run main.py
```

Check it with `claude mcp list`, then ask Claude: *"Who am I logged in as in Superset?"*

### Claude Desktop

Add this to `claude_desktop_config.json` (use the full path to the folder, and the full path to `uv` if Claude Desktop cannot find it):

```json
{
  "mcpServers": {
    "superset": {
      "command": "uv",
      "args": ["--directory", "/full/path/to/superset-mcp", "run", "main.py"]
    }
  }
}
```

### No Superset yet?

Start one locally with Docker (login `admin` / `admin` at http://localhost:8088):

```bash
git clone --branch 4.1.1 --depth 1 https://github.com/apache/superset && \
cd superset && \
docker compose -f docker-compose-image-tag.yml up
```

## Example requests

- "How many dashboards are there, and how many are mine?"
- "List the charts on dashboard 12"
- "Find datasets with 'sales' in the name"
- "Run this SQL on database 1: SELECT COUNT(*) FROM orders"
- "Create a dashboard titled 'Sales Overview'"
- "Tag dashboard 3 with 'Finance'"

## Listing large instances

Every `*_list` tool fetches all pages up to `max_results` (default 500; 50 for query history) and returns:

| Field | Meaning |
|-------|---------|
| `count` | Total rows matching the filters on the server |
| `returned` | Rows included in this response |
| `truncated` | `true` when `returned < count`; narrow the filters or raise `max_results` |
| `result` | The rows |

## Available MCP Tools

This plugin offers the following MCP tools that Claude can use:

### Authentication
- `superset_auth_check_token_validity` - Check if the current access token is valid
- `superset_auth_refresh_token` - Refresh the access token
- `superset_auth_authenticate_user` - Authenticate with Superset

### Dashboards
- `superset_dashboard_list` - List dashboards (all pages; filter by `search`, `mine`)
- `superset_dashboard_get_by_id` - Get a specific dashboard
- `superset_dashboard_create` - Create a new dashboard
- `superset_dashboard_update` - Update an existing dashboard
- `superset_dashboard_delete` - Delete a dashboard

### Charts
- `superset_chart_list` - List charts in compact rows (filter by `search`, `dashboard_id`, `mine`)
- `superset_chart_get_by_id` - Get a specific chart
- `superset_chart_create` - Create a new chart
- `superset_chart_update` - Update an existing chart
- `superset_chart_delete` - Delete a chart

### Databases
- `superset_database_list` - List database connections (filter by `search`)
- `superset_database_get_by_id` - Get a specific database
- `superset_database_create` - Create a new database connection
- `superset_database_get_tables` - List tables in a database
- `superset_database_schemas` - Get schemas for a database
- `superset_database_test_connection` - Test a database connection
- `superset_database_update` - Update an existing database connection
- `superset_database_delete` - Delete a database connection
- `superset_database_get_catalogs` - Get catalogs for a database
- `superset_database_get_connection` - Get database connection information
- `superset_database_get_function_names` - Get function names supported by a database
- `superset_database_get_related_objects` - Get charts and dashboards associated with a database
- `superset_database_validate_sql` - Validate arbitrary SQL against a database
- `superset_database_validate_parameters` - Validate database connection parameters

### Datasets
- `superset_dataset_list` - List datasets in compact rows (filter by `search`, `database_id`, `mine`)
- `superset_dataset_get_by_id` - Get a specific dataset
- `superset_dataset_create` - Create a new dataset

### SQL Lab
- `superset_sqllab_execute_query` - Execute a SQL query
- `superset_sqllab_get_saved_queries` - List saved SQL queries
- `superset_sqllab_format_sql` - Format a SQL query
- `superset_sqllab_get_results` - Get query results
- `superset_sqllab_estimate_query_cost` - Estimate query cost
- `superset_sqllab_export_query_results` - Export query results to CSV
- `superset_sqllab_get_bootstrap_data` - Get SQL Lab bootstrap data

### Queries
- `superset_query_list` - List SQL Lab query history, newest first
- `superset_query_get_by_id` - Get a specific query
- `superset_query_stop` - Stop a running query

### Saved Queries
- `superset_saved_query_get_by_id` - Get a specific saved query
- `superset_saved_query_create` - Create a new saved query

### User Information
- `superset_user_get_current` - Get current user info
- `superset_user_get_roles` - Get user roles

### Activity
- `superset_activity_get_recent` - Get recent activity data

### System
- `superset_menu_get` - Get menu data
- `superset_config_get_base_url` - Get the base URL of the Superset instance

### Tags
- `superset_tag_list` - List tags
- `superset_tag_create` - Create a new tag
- `superset_tag_get_by_id` - Get a specific tag
- `superset_tag_objects` - Get objects associated with tags
- `superset_tag_delete` - Delete a tag
- `superset_tag_object_add` - Add a tag to an object
- `superset_tag_object_remove` - Remove a tag from an object

### Exploration Tools
- `superset_explore_form_data_create` - Create form data for chart exploration
- `superset_explore_form_data_get` - Get form data for chart exploration
- `superset_explore_permalink_create` - Create a permalink for chart exploration
- `superset_explore_permalink_get` - Get a permalink for chart exploration

### Advanced Data Types
- `superset_advanced_data_type_convert` - Convert a value to an advanced data type
- `superset_advanced_data_type_list` - List available advanced data types

## Environment variables

| Variable | Description | Default |
|----------|-------------|---------|
| `SUPERSET_BASE_URL` | URL of your Superset instance | `http://localhost:8088` |
| `SUPERSET_USERNAME` | Superset username | none |
| `SUPERSET_PASSWORD` | Superset password | none |

## Troubleshooting

- **"Not authenticated"**: check `SUPERSET_USERNAME` / `SUPERSET_PASSWORD` in `.env`, and that you can log in to the Superset UI with them.
- **Server does not start in Claude**: run `uv --directory /path/to/superset-mcp run main.py` in a terminal to see the error; Claude may also need the full path to `uv` (`which uv`).
- **A list looks short**: check `truncated` in the response and raise `max_results`.

## Security notes

- Credentials live only in your local `.env`, which is gitignored.
- If JWT login is used, tokens are cached in `.superset_token` and `.superset_refresh_token` in the project folder; both are gitignored.
- The server talks only to your Superset instance. Anything a tool returns is shown to the AI client you use.
- Write tools (create, update, delete, SQL execution) act with your Superset permissions. Use a user whose rights match what you want the AI to be able to do.

## License

MIT, as upstream.
