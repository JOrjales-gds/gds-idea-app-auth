# gds-idea-app-auth

A unified authentication and authorization library for GDS IDEA web applications using AWS Cognito and Application Load Balancer.

## Overview

This library provides a consistent authentication and authorization mechanism for web applications in the GDS IDEA ecosystem. It handles token verification, user management, and authorization rules across multiple web frameworks including:

- Streamlit
- Dash
- FastAPI
- Gradio

The library verifies and processes tokens from AWS Application Load Balancer (ALB) and AWS Cognito, extracting user information and applying configurable authorization rules based on email domains, Cognito groups, or specific user lists.

### Key Features

- **Multi-framework support**: Works with Streamlit, Dash, FastAPI, and Gradio
- **Token verification**: Validates ALB and Cognito JWT tokens
- **Flexible authorization**: Filter by email domains, Cognito groups, or specific users
- **Configuration options**: Load from S3, Parameter Store, or Streamlit secrets
- **User object**: Easy access to user details like email, groups, and claims

## Installation

### Prerequisites

- Python 3.13+
- A web application using one of the supported frameworks
- AWS Cognito User Pool configured with an Application Load Balancer

### Install from PyPI

```bash
pip install cognito-auth
```

### Install from source

```bash
git clone https://github.com/JOrjales-gds/gds-idea-app-auth.git
cd gds-idea-app-auth
pip install -e .
```

## Quick Start

### Streamlit Example

```python
import streamlit as st
from cognito_auth.guard import AuthGuard

# Create a guard that allows users with specific email domains
guard = AuthGuard(allowed_domains=['digital.cabinet-office.gov.uk'])

# Apply the guard and get the authenticated user
user = guard.streamlit()

# Now you can use the user information in your app
st.write(f"Welcome {user.email}!")
st.write(f"Groups: {', '.join(user.access_claims.get('cognito:groups', []))}")
```

### FastAPI Example

```python
from fastapi import FastAPI, Depends
from cognito_auth.guard import AuthGuard

app = FastAPI()

# Create a guard with multiple authorization options
guard = AuthGuard(
    allowed_domains=['digital.cabinet-office.gov.uk'],
    allowed_groups=['Admins', 'Analysts'],
    require_all=False  # User needs to match ANY rule (domain OR group)
)

@app.get("/")
def home(user = Depends(guard.fastapi())):
    return {
        "message": f"Hello {user.email}!",
        "groups": user.access_claims.get("cognito:groups", []),
        "username": user.username
    }
```

### Dash Example

```python
from dash import Dash, html, callback, Output, Input
from cognito_auth.guard import AuthGuard

guard = AuthGuard(allowed_domains=['digital.cabinet-office.gov.uk'])

@guard.dash
def create_app():
    app = Dash(__name__)
    
    app.layout = html.Div([
        html.H1("Protected Dashboard"),
        html.Div(id='user-info')
    ])
    
    @app.callback(Output('user-info', 'children'), Input('user-info', 'id'))
    def show_user(_):
        user = guard.get_current_user_dash()
        return f"Logged in as: {user.email}"
    
    return app

app = create_app()
server = app.server
```

### Gradio Example

```python
import gradio as gr
from cognito_auth.guard import AuthGuard

guard = AuthGuard(allowed_domains=['digital.cabinet-office.gov.uk'])

def process(text, request: gr.Request):
    user = guard.get_current_user_gradio(request)
    if not user:
        return "Not authenticated"
    return f"Hello {user.email}! You said: {text}"

with gr.Blocks() as demo:
    gr.Markdown("# Protected App")
    text_input = gr.Textbox(label="Enter text")
    output = gr.Textbox(label="Response")
    submit = gr.Button("Submit")
    submit.click(process, inputs=[text_input], outputs=output)

demo.launch()
```

## Configuration

### Authorization Options

You can configure the `AuthGuard` with various authorization rules:

```python
guard = AuthGuard(
    allowed_domains=['digital.cabinet-office.gov.uk', 'example.org'],  # Allow specific email domains
    allowed_groups=['Admins', 'Analysts'],  # Allow specific Cognito groups
    allowed_users=['user1@example.com', 'user2@example.com'],  # Allow specific users
    require_all=False,  # If True, user must match ALL rules; if False, ANY rule
    redirect_url="https://gds-idea.click/401.html",  # Where to redirect unauthorized users
    region="eu-west-2"  # AWS region
)
```

### Loading Configuration from External Sources

```python
# From S3
guard = AuthGuard.from_s3('my-config-bucket', 'auth-config.json')

# From AWS Parameter Store
guard = AuthGuard.from_parameter_store('/my-app/auth-config')

# From Streamlit secrets
guard = AuthGuard.from_secrets()
```

## API Reference

### AuthGuard Class

#### `AuthGuard(authorizer=None, allowed_domains=None, allowed_groups=None, allowed_users=None, redirect_url="https://gds-idea.click/401.html", region="eu-west-2", require_all=False)`

Creates a new authentication guard with the specified authorization rules.

**Parameters:**
- `authorizer` (Optional[Authorizer]): Pre-configured Authorizer instance
- `allowed_domains` (Optional[list[str]]): List of allowed email domains
- `allowed_groups` (Optional[list[str]]): List of allowed Cognito groups
- `allowed_users` (Optional[list[str]]): List of allowed usernames or emails
- `redirect_url` (str): Where to redirect on auth failure
- `region` (str): AWS region
- `require_all` (bool): If True, ALL rules must pass. If False, ANY rule passes.

#### Class Methods

- `from_s3(bucket: str, key: str, **kwargs)`: Create guard with authorizer from S3.
- `from_secrets(**kwargs)`: Create guard with authorizer from Streamlit secrets.
- `from_parameter_store(parameter_prefix: str, **kwargs)`: Create guard with authorizer from AWS Parameter Store.

#### Framework-Specific Methods

- `streamlit() -> User`: Protect Streamlit app and return authenticated user.
- `dash(func: Callable) -> Callable`: Decorator for protecting Dash apps.
- `fastapi()`: Dependency for protecting FastAPI routes.
- `get_current_user_dash() -> Optional[User]`: Get current user in Dash callback.
- `get_current_user_gradio(request) -> Optional[User]`: Get current user in Gradio function.
- `gradio_middleware()`: Middleware for FastAPI+Gradio apps.

### User Class

#### `User(oidc_data_header: str, access_token_header: str, region: str, verify_tokens: bool = True)`

Represents an authenticated user from AWS ALB + Cognito.

**Properties:**
- `is_authenticated` (bool): Whether the user is authenticated.
- `sub` (str): User's subject identifier (unique user ID).
- `username` (str): User's username.
- `email` (str): User's email address.
- `email_domain` (str): Domain part of the user's email.
- `email_verified` (bool): Whether the user's email has been verified.
- `exp` (Optional[datetime]): Token expiration time.
- `issuer` (str): Token issuer (Cognito User Pool).
- `oidc_claims` (dict): All claims from x-amzn-oidc-data token.
- `access_claims` (dict): All claims from x-amzn-oidc-accesstoken token.

## Development

### Setup Development Environment

```bash
git clone https://github.com/JOrjales-gds/gds-idea-app-auth.git
cd gds-idea-app-auth
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install -e .
pip install -r requirements-dev.txt  # if available
```

### Running Tests

```bash
pytest
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Contact

- Issues: [GitHub Issues](https://github.com/JOrjales-gds/gds-idea-app-auth/issues)
- Maintainer: [Jorge Orjales](jorge.orjales@digital.cabinet-office.gov.uk)