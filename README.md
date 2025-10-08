# cognito-auth

[![Python](https://img.shields.io/badge/python-3.12%2B-blue)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.2-orange)](pyproject.toml)

Unified authentication and authorisation for AWS Cognito-protected web applications. Supports Streamlit, Dash, FastAPI, and Gradio with minimal configuration.

## Overview

cognito-auth simplifies the integration of AWS Cognito authentication into Python web applications. It handles JWT token verification, user management, and authorization rules across multiple web frameworks, saving developers from implementing complex authentication logic.

The package verifies JWT tokens from AWS Application Load Balancer (ALB) OIDC integration with Cognito User Pools and provides flexible authorization rules based on Cognito groups or specific users.

### Key Features

- **Multi-framework support**: Works with Streamlit, Dash, FastAPI, and Gradio
- **Unified API**: Consistent interface across all supported frameworks
- **Flexible authorization**: Rule-based access control using Cognito groups or individual users
- **JWT verification**: Automatic verification of ALB and Cognito tokens
- **Development mode**: Easy local development without Cognito infrastructure
- **AWS Secrets Manager integration**: Secure configuration management for production

## Installation

### Basic Installation
```bash
pip install cognito-auth
```

### Framework-specific Installation
```bash
pip install cognito-auth[streamlit]  # Streamlit support
pip install cognito-auth[fastapi]    # FastAPI support
pip install cognito-auth[dash]       # Dash support
pip install cognito-auth[gradio]     # Gradio support
pip install cognito-auth[all]        # All frameworks
```

## Quick Start

```python
# For Streamlit apps
from cognito_auth.streamlit import StreamlitAuth 

auth = StreamlitAuth.from_config()
user = auth.get_auth_user()

if user.is_authenticated:
    st.write(f"Welcome, {user.email}!")
```

## Framework Examples

### Streamlit
```python
from cognito_auth.streamlit import StreamlitAuth

auth = StreamlitAuth.from_config()
user = auth.get_auth_user()

if user.is_authenticated:
    st.write(f"Welcome, {user.email}!")
    if "developers" in user.groups:
        st.write("You have developer access.")
```

### FastAPI
```python
from fastapi import Depends, FastAPI
from cognito_auth.fastapi import FastAPIAuth

app = FastAPI()
auth = FastAPIAuth.from_config()

@app.get("/protected")
async def protected_route(user = Depends(auth.get_auth_user)):
    return {"message": f"Hello {user.email}!", "groups": user.groups}
```

### Dash
```python
from dash import Dash, html
from cognito_auth.dash import DashAuth

app = Dash(__name__)
auth = DashAuth.from_config()

@auth.require_auth
def serve_layout():
    user = auth.get_auth_user()
    return html.Div([
        html.H1(f"Welcome, {user.email}!")
    ])

app.layout = serve_layout
```

### Gradio
```python
import gradio as gr
from cognito_auth.gradio import GradioAuth

auth = GradioAuth.from_config()

def greet(request):
    user = auth.get_auth_user(request)
    return f"Hello, {user.email}!"

demo = gr.Interface(fn=greet, inputs=gr.Textbox(), outputs=gr.Textbox())
demo.launch()
```

## Configuration

### Environment Variables

The package uses the following environment variables:

- `COGNITO_AUTH_CONFIG_PATH`: Path to local JSON configuration file
- `COGNITO_AUTH_SECRET_NAME`: AWS Secrets Manager secret name for configuration
- `COGNITO_AUTH_DEV_MODE`: Set to "true" for local development without Cognito

### Configuration Files

Create an `auth-config.json` file:

```json
{
  "allowed_groups": ["developers", "admins", "users"],
  "allowed_users": ["special-user@example.com"],
  "require_all": false
}
```

For development, create a `dev-mock-user.json` file:

```json
{
  "email": "developer@example.com",
  "sub": "12345678-1234-1234-1234-123456789abc",
  "username": "12345678-1234-1234-1234-123456789abc",
  "groups": ["developers", "users"]
}
```

## How It Works

1. AWS ALB (Application Load Balancer) intercepts requests and adds OIDC headers after Cognito authentication
2. Framework-specific auth class extracts headers via framework-specific methods
3. User is instantiated, which triggers TokenVerifier to verify tokens
4. If tokens are valid, Authoriser checks if user meets authorization rules
5. If authorized, user object is returned; otherwise, redirects or raises exception

## Documentation

Full documentation available at [github pages](https://co-cddo.github.io/gds-idea-app-auth).

## Contributing

### Setup

```bash
# Clone and install dependencies
git clone https://github.com/co-cddo/gds-idea-app-auth/
cd cognito-auth
uv sync
```

### Development Commands

- `uv run pytest` - Run tests
- `uv run pytest --cov` - Run tests with coverage
- `uv run ruff check .` - Lint code
- `uv run ruff format .` - Format code
- `uv run mkdocs serve` - Preview documentation locally

### Testing

Tests use pytest with fixtures and mocks. Add tests for new features in `tests/cognito_auth/`.

```bash
# Run specific test file
uv run pytest tests/cognito_auth/test_user.py -v

# Run tests matching pattern
uv run pytest -k test_authoriser
```

### Code Standards

- Python 3.12+
- Type hints where appropriate
- Ruff for linting and formatting (configured in `pyproject.toml`)
- 100% test coverage for new features

### Project Structure

```
src/cognito_auth/        # Main package
├── user.py              # User model
├── token_verifier.py    # JWT verification
├── authoriser.py        # Authorisation rules
└── [framework].py       # Framework integrations

tests/cognito_auth/      # Test suite
docs/                    # MkDocs documentation
```

## Troubleshooting

### Common Issues

#### Token Verification Errors
- Check that your ALB is correctly configured with Cognito User Pool
- Verify that clock sync is correct between your server and AWS

#### Authorization Failures
- Check your auth-config.json rules
- Verify user belongs to the required Cognito groups
- Check if email is in allowed_users list

#### Development Mode Issues
- Ensure COGNITO_AUTH_DEV_MODE=true is set
- Create dev-mock-user.json in the project root

See `CLAUDE.md` for detailed architecture and development guidance.

## License

MIT License - see [LICENSE](LICENSE) for details.