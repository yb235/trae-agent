# Examples and Use Cases

This guide provides real-world examples and use cases for Trae Agent, demonstrating how to solve common software engineering tasks.

## Table of Contents

- [Code Generation](#code-generation)
- [Bug Fixing](#bug-fixing)
- [Testing](#testing)
- [Refactoring](#refactoring)
- [Documentation](#documentation)
- [Code Review](#code-review)
- [Migration](#migration)
- [DevOps](#devops)

## Code Generation

### Example 1: Create a REST API Endpoint

**Task:**
```bash
trae-cli run "Create a REST API endpoint in app.py that handles POST requests to /api/users. It should accept JSON with name and email fields, validate the email format, and return a user ID. Include error handling."
```

**What the agent does:**
1. Reads existing `app.py` structure
2. Creates the `/api/users` endpoint
3. Adds email validation logic
4. Implements error handling
5. Returns appropriate HTTP status codes

**Result:**
```python
@app.route('/api/users', methods=['POST'])
def create_user():
    try:
        data = request.get_json()
        name = data.get('name')
        email = data.get('email')
        
        if not name or not email:
            return jsonify({'error': 'Name and email required'}), 400
        
        # Validate email
        if not re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', email):
            return jsonify({'error': 'Invalid email format'}), 400
        
        user_id = generate_user_id()
        # Save to database
        
        return jsonify({'user_id': user_id}), 201
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### Example 2: Create a Data Processing Pipeline

**Task:**
```bash
trae-cli run "Create a Python module data_pipeline.py with functions to: 1) read CSV files, 2) clean missing values, 3) normalize numeric columns, 4) export to JSON. Use pandas and include type hints."
```

**Result:**
Complete data pipeline with proper error handling, type hints, and documentation.

### Example 3: Generate Database Models

**Task:**
```bash
trae-cli run "Create SQLAlchemy models in models/user.py for User and Profile tables. User has id, username, email, created_at. Profile has id, user_id, bio, avatar_url. Include relationships."
```

### Example 4: Create CLI Tool

**Task:**
```bash
trae-cli run "Create a CLI tool in cli.py using Click that has commands for: create, list, update, delete operations on a todo list. Store todos in a JSON file."
```

## Bug Fixing

### Example 1: Fix Authentication Bug

**Setup:**
```python
# auth.py - has a bug
def authenticate_user(username, password):
    user = db.query(User).filter_by(username=username).first()
    if user.password == password:  # Bug: comparing plain text!
        return user
    return None
```

**Task:**
```bash
trae-cli run "Fix the security bug in auth.py where passwords are compared in plain text. Use bcrypt for password hashing and verification."
```

**What the agent does:**
1. Identifies the security vulnerability
2. Imports bcrypt
3. Updates the function to use bcrypt verification
4. Adds password hashing for new users
5. Creates a migration note

### Example 2: Fix Memory Leak

**Task:**
```bash
trae-cli run "Investigate and fix the memory leak in data_processor.py. The memory usage grows continuously when processing large files."
```

**What the agent does:**
1. Analyzes the code for memory issues
2. Identifies that large data structures aren't being released
3. Implements proper cleanup and generators
4. Adds memory profiling

### Example 3: Fix Race Condition

**Task:**
```bash
trae-cli run "Fix the race condition in cache.py where multiple threads can corrupt the cache. Use appropriate locking mechanisms."
```

### Example 4: Fix Edge Case

**Task:**
```bash
trae-cli run "Fix the bug in calculate_average() that crashes when given an empty list. Handle edge cases properly and add tests."
```

## Testing

### Example 1: Generate Unit Tests

**Task:**
```bash
trae-cli run "Create comprehensive pytest unit tests for the UserService class in services/user_service.py. Aim for 100% code coverage including edge cases."
```

**Result:**
```python
# test_user_service.py
import pytest
from services.user_service import UserService

class TestUserService:
    def test_create_user_success(self):
        service = UserService()
        user = service.create_user("john", "john@example.com")
        assert user.username == "john"
        assert user.email == "john@example.com"
    
    def test_create_user_invalid_email(self):
        service = UserService()
        with pytest.raises(ValueError):
            service.create_user("john", "invalid-email")
    
    def test_create_user_empty_username(self):
        service = UserService()
        with pytest.raises(ValueError):
            service.create_user("", "john@example.com")
    
    # ... more tests
```

### Example 2: Add Integration Tests

**Task:**
```bash
trae-cli run "Create integration tests in tests/integration/test_api.py that test the full workflow: create user -> authenticate -> update profile -> delete user. Use pytest fixtures for database setup."
```

### Example 3: Add Property-Based Tests

**Task:**
```bash
trae-cli run "Add property-based tests using Hypothesis for the sorting algorithm in algorithms.py. Test that it handles all possible inputs correctly."
```

### Example 4: Generate Test Data

**Task:**
```bash
trae-cli run "Create a test fixture generator in tests/fixtures.py that creates realistic test data for User, Post, and Comment models."
```

## Refactoring

### Example 1: Extract Functions

**Task:**
```bash
trae-cli run "Refactor the process_order() function in orders.py. It's too long (200+ lines). Extract separate functions for validation, payment processing, and notification."
```

**What the agent does:**
1. Analyzes the function structure
2. Identifies logical sections
3. Extracts into separate functions
4. Preserves behavior
5. Adds tests to verify

### Example 2: Apply Design Pattern

**Task:**
```bash
trae-cli run "Refactor the payment processing code in payments.py to use the Strategy pattern. Support credit card, PayPal, and cryptocurrency payments."
```

### Example 3: Remove Code Duplication

**Task:**
```bash
trae-cli run "Remove code duplication between UserController and AdminController. Extract common logic into a base controller."
```

### Example 4: Improve Performance

**Task:**
```bash
trae-cli run "Refactor the search function in search.py to improve performance. Current implementation is O(n²), optimize to O(n log n) or better."
```

### Example 5: Modernize Code

**Task:**
```bash
trae-cli run "Update all class-based views in views.py to use modern async/await syntax instead of callback-based approach."
```

## Documentation

### Example 1: Generate API Documentation

**Task:**
```bash
trae-cli run "Generate comprehensive API documentation for all public functions in the api/ directory. Use Google-style docstrings with parameter descriptions, return types, and examples."
```

**Result:**
```python
def create_user(username: str, email: str, password: str) -> User:
    """Create a new user account.
    
    Creates a new user with the provided credentials. The password
    is automatically hashed using bcrypt before storage.
    
    Args:
        username: Unique username for the account. Must be 3-20 characters.
        email: Valid email address for the account.
        password: Plain text password. Must be at least 8 characters.
    
    Returns:
        User: The newly created user object with generated ID.
    
    Raises:
        ValueError: If username is taken or email is invalid.
        DatabaseError: If database operation fails.
    
    Example:
        >>> user = create_user("john", "john@example.com", "securepass123")
        >>> print(user.id)
        42
    """
```

### Example 2: Create README

**Task:**
```bash
trae-cli run "Create a comprehensive README.md for this project. Include: project description, installation instructions, usage examples, configuration, and contributing guidelines."
```

### Example 3: Generate Changelog

**Task:**
```bash
trae-cli run "Generate a CHANGELOG.md based on git commit history. Organize by version and category (Added, Changed, Fixed, Removed)."
```

### Example 4: Create Architecture Documentation

**Task:**
```bash
trae-cli run "Create an ARCHITECTURE.md file documenting the system architecture, key components, data flow, and design decisions."
```

## Code Review

### Example 1: Review Pull Request

**Task:**
```bash
trae-cli run "Review the changes in auth.py and provide feedback on: security issues, code quality, potential bugs, and suggestions for improvement."
```

### Example 2: Check Code Style

**Task:**
```bash
trae-cli run "Review all Python files in the src/ directory for PEP 8 compliance. Fix formatting issues and update code to follow best practices."
```

### Example 3: Security Audit

**Task:**
```bash
trae-cli run "Perform a security audit of the authentication and authorization code. Check for common vulnerabilities like SQL injection, XSS, CSRF, and insecure password storage."
```

### Example 4: Performance Review

**Task:**
```bash
trae-cli run "Review the database queries in models.py. Identify N+1 query problems, missing indexes, and inefficient queries. Suggest optimizations."
```

## Migration

### Example 1: Python 2 to 3

**Task:**
```bash
trae-cli run "Migrate the codebase from Python 2 to Python 3. Update print statements, exception syntax, string/unicode handling, and imports."
```

### Example 2: Update Dependencies

**Task:**
```bash
trae-cli run "Update the codebase to use the latest version of Flask (v3.0). Update deprecated function calls and middleware configuration."
```

### Example 3: Database Schema Migration

**Task:**
```bash
trae-cli run "Create an Alembic migration to add a new 'email_verified' boolean column to the users table. Include upgrade and downgrade functions."
```

### Example 4: API Version Migration

**Task:**
```bash
trae-cli run "Migrate all API endpoints from v1 to v2 format. Update response structure from flat JSON to nested objects with metadata."
```

## DevOps

### Example 1: Create Dockerfile

**Task:**
```bash
trae-cli run "Create a production-ready Dockerfile for this Python Flask application. Use multi-stage builds, minimal base image, non-root user, and health checks."
```

**Result:**
```dockerfile
# Build stage
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Runtime stage
FROM python:3.12-slim
WORKDIR /app

# Create non-root user
RUN useradd -m -u 1000 appuser

# Copy dependencies from builder
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Add local bin to PATH
ENV PATH=/home/appuser/.local/bin:$PATH

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
    CMD python -c "import requests; requests.get('http://localhost:5000/health')"

EXPOSE 5000
CMD ["python", "app.py"]
```

### Example 2: Create CI/CD Pipeline

**Task:**
```bash
trae-cli run "Create a GitHub Actions workflow in .github/workflows/ci.yml that runs tests, linting, and type checking on every push and pull request."
```

### Example 3: Add Monitoring

**Task:**
```bash
trae-cli run "Add Prometheus metrics to the Flask application. Track request count, response time, and error rate. Create a /metrics endpoint."
```

### Example 4: Create Deployment Script

**Task:**
```bash
trae-cli run "Create a deployment script deploy.sh that builds the Docker image, pushes to registry, and deploys to Kubernetes cluster."
```

## Complex Workflows

### Workflow 1: Full Feature Implementation

**Task:**
```bash
trae-cli run "Implement a password reset feature: 1) Add reset token to User model, 2) Create /reset-password endpoint, 3) Add email sending, 4) Create reset form, 5) Write tests for all components."
```

**What the agent does:**
1. Updates database model
2. Creates API endpoints
3. Implements email service
4. Creates frontend form
5. Writes comprehensive tests
6. Updates documentation

### Workflow 2: Performance Optimization

**Task:**
```bash
trae-cli run "Optimize the application performance: 1) Add database indexes for frequently queried columns, 2) Implement Redis caching for user sessions, 3) Add query result caching, 4) Optimize slow queries identified in logs."
```

### Workflow 3: Security Hardening

**Task:**
```bash
trae-cli run "Harden application security: 1) Add rate limiting to API endpoints, 2) Implement CSRF protection, 3) Add input sanitization, 4) Update dependencies with security patches, 5) Add security headers."
```

## Interactive Mode Examples

### Example: Iterative Development

```bash
trae-cli interactive

> Create a User class with name and email
✓ Created User class in models/user.py

> Add a method to validate email format
✓ Added validate_email() method

> Write tests for the User class
✓ Created tests in tests/test_user.py

> Run the tests
✓ All tests passing

> Add password field with hashing
✓ Added password field with bcrypt

> Update tests for password
✓ Updated tests, all passing

> exit
```

## Docker Mode Examples

### Example: Test in Multiple Python Versions

```bash
# Test in Python 3.10
trae-cli run "Run the test suite" --docker-image python:3.10

# Test in Python 3.11
trae-cli run "Run the test suite" --docker-image python:3.11

# Test in Python 3.12
trae-cli run "Run the test suite" --docker-image python:3.12
```

### Example: Isolated Development

```bash
# Each development session in fresh container
trae-cli run "Experiment with new algorithm implementation" \
    --docker-image python:3.12 \
    --working-dir ~/experiments

# Container is removed after, no pollution
```

## Best Practices from Examples

### 1. Be Specific

✓ Good:
```bash
"Add input validation to the signup form. Check email format, password length (min 8), and username uniqueness."
```

✗ Bad:
```bash
"Make the form better"
```

### 2. Include Context

✓ Good:
```bash
"Fix the bug in payment.py where the calculate_tax() function returns incorrect values for Canadian provinces. It should use GST+PST rates."
```

✗ Bad:
```bash
"Fix the tax bug"
```

### 3. Request Verification

✓ Good:
```bash
"Add the logging feature and write tests to verify logs are created correctly."
```

✗ Bad:
```bash
"Add logging"
```

### 4. Break Down Complex Tasks

✓ Good:
```bash
# Task 1
"Create the database schema for the blog feature"

# Task 2
"Implement the blog CRUD API endpoints"

# Task 3
"Add the blog UI components"
```

✗ Bad:
```bash
"Build a complete blog feature with everything"
```

## Tips for Success

### 1. Provide File Context

When referring to specific files:
```bash
"Fix the authentication bug in src/auth/login.py line 42 where it doesn't handle None values"
```

### 2. Specify Standards

When applicable:
```bash
"Add docstrings following Google style guide"
"Format code according to PEP 8"
"Use TypeScript strict mode"
```

### 3. Request Examples

For documentation:
```bash
"Generate API documentation with usage examples for each function"
```

### 4. Include Error Messages

When debugging:
```bash
"Fix the error 'KeyError: user_id' that occurs in process_order() when handling guest checkouts"
```

### 5. Specify Technology

When multiple options exist:
```bash
"Create tests using pytest and pytest-mock"
"Build API with FastAPI (not Flask)"
```

## Related Documentation

- [Getting Started](GETTING_STARTED.md)
- [Workflow Guide](WORKFLOW.md)
- [Tools Documentation](tools.md)
- [API Reference](API_REFERENCE.md)
- [Configuration Guide](CONFIGURATION.md)

## Summary

Trae Agent can handle a wide variety of software engineering tasks:

- **Code Generation**: From single functions to complete modules
- **Bug Fixing**: From syntax errors to complex logic bugs
- **Testing**: Unit, integration, and property-based tests
- **Refactoring**: Code cleanup, pattern application, optimization
- **Documentation**: API docs, READMEs, architecture guides
- **Code Review**: Style, security, performance analysis
- **Migration**: Language versions, frameworks, databases
- **DevOps**: Dockerfiles, CI/CD, deployment scripts

The key to success is providing clear, specific instructions with appropriate context and verification steps.
