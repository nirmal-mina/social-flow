# 🚀 **Getting Started Guide - Social Flow Backend Development**

## 🎯 **Quick Start for New Developers**

This guide helps you get started with Social Flow backend development in the correct order, avoiding common pitfalls.

---

## 📋 **Prerequisites Checklist**

Before starting development, ensure you have:
- [ ] Python 3.11+ installed
- [ ] Docker and Docker Compose installed
- [ ] PostgreSQL knowledge (basic)
- [ ] Redis knowledge (basic)
- [ ] Git setup with proper credentials
- [ ] IDE/Editor configured for Python development

---

## 🛠️ **Environment Setup (Step-by-Step)**

### **1. Initial Setup**
```bash
# Clone the repository
git clone https://github.com/nirmal-mina/social-flow.git
cd social-flow/social-flow-backend

# Create virtual environment
python -m venv venv

# Activate virtual environment
# Linux/Mac:
source venv/bin/activate
# Windows:
venv\Scripts\activate

# Verify Python version
python --version  # Should be 3.11+
```

### **2. Install Dependencies**
```bash
# Install production dependencies
pip install -r requirements.txt

# Install development dependencies
pip install -r requirements-dev.txt

# Verify installations
pytest --version
black --version
```

### **3. Environment Configuration**
```bash
# Copy environment template
cp env.example .env

# Edit .env file with your settings
DATABASE_URL=postgresql://user:password@localhost:5432/social_flow_dev
REDIS_URL=redis://localhost:6379/0
JWT_SECRET=your-super-secret-jwt-key-here
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=30
```

### **4. Start Services**
```bash
# Start database and Redis
docker-compose up -d postgres redis

# Wait for services to be ready
docker-compose logs postgres  # Check if ready
docker-compose logs redis     # Check if ready

# Verify services are running
docker-compose ps
```

### **5. Database Setup**
```bash
# Create database tables
alembic upgrade head

# Verify database connection
python -c "from app.core.database import engine; print('Database connected!')"
```

### **6. Verify Setup**
```bash
# Run basic tests
pytest tests/ -v

# Start development server
uvicorn app.main:app --reload --port 8000

# Test API (in another terminal)
curl http://localhost:8000/health
# Should return: {"status": "healthy"}
```

---

## 🎯 **Your First Development Task**

### **Start with Configuration Module**

**Why start here?**
- Foundation for all other modules
- Easy to understand and test
- Required by everything else

### **Step 1: Understand Existing Structure**
```bash
# Explore the app structure
ls -la app/
ls -la app/core/
ls -la tests/

# Check existing configuration
cat app/core/config.py
cat env.example
```

### **Step 2: Write Your First Test**
```python
# tests/unit/test_config.py
import pytest
from app.core.config import Settings

def test_config_loads_defaults():
    """Test that config loads with default values."""
    settings = Settings()
    assert settings.app_name == "Social Flow"
    assert settings.debug is False
    assert settings.jwt_algorithm == "HS256"

def test_config_validates_required_fields():
    """Test that required fields are validated."""
    import os
    # Remove JWT_SECRET if it exists
    jwt_secret = os.environ.pop('JWT_SECRET', None)
    
    with pytest.raises(ValueError):
        Settings()
    
    # Restore JWT_SECRET
    if jwt_secret:
        os.environ['JWT_SECRET'] = jwt_secret
```

### **Step 3: Run and Fix Tests**
```bash
# Run your test
pytest tests/unit/test_config.py -v

# If tests fail, implement the missing functionality
# in app/core/config.py

# Keep running tests until they pass
pytest tests/unit/test_config.py -v --tb=short
```

### **Step 4: Implement Configuration**
```python
# app/core/config.py
from pydantic import BaseSettings, validator
from typing import Optional

class Settings(BaseSettings):
    # App settings
    app_name: str = "Social Flow"
    debug: bool = False
    version: str = "1.0.0"
    
    # Database settings
    database_url: str
    
    # Redis settings
    redis_url: str = "redis://localhost:6379/0"
    
    # JWT settings
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 30
    
    @validator('jwt_secret')
    def jwt_secret_must_not_be_empty(cls, v):
        if not v or len(v.strip()) == 0:
            raise ValueError('JWT_SECRET must not be empty')
        return v
    
    @validator('database_url')
    def database_url_must_be_valid(cls, v):
        if not v.startswith(('postgresql://', 'sqlite://')):
            raise ValueError('DATABASE_URL must be a valid database URL')
        return v
    
    class Config:
        env_file = ".env"
        case_sensitive = False

# Create global settings instance
settings = Settings()
```

### **Step 5: Verify Everything Works**
```bash
# Run tests again
pytest tests/unit/test_config.py -v

# Run all tests to ensure nothing is broken
pytest tests/ -v

# Start the app to verify configuration loads
uvicorn app.main:app --reload --port 8000
```

---

## 🚨 **Common Pitfalls and How to Avoid Them**

### **1. Database Connection Issues**

**Problem:** Database connection fails
```bash
# Error message
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) could not connect to server
```

**Solution:**
```bash
# Check if PostgreSQL is running
docker-compose ps postgres

# Check PostgreSQL logs
docker-compose logs postgres

# Restart if needed
docker-compose restart postgres

# Test connection manually
python -c "
import psycopg2
conn = psycopg2.connect('postgresql://user:password@localhost:5432/social_flow_dev')
print('Connection successful!')
conn.close()
"
```

### **2. Import Errors**

**Problem:** Python can't find your modules
```bash
# Error message
ModuleNotFoundError: No module named 'app.core'
```

**Solution:**
```bash
# Make sure you're in the right directory
pwd  # Should end with /social-flow-backend

# Make sure virtual environment is activated
which python  # Should point to venv/bin/python

# Add current directory to Python path (if needed)
export PYTHONPATH="${PYTHONPATH}:$(pwd)"

# Or run with python -m
python -m pytest tests/
```

### **3. Test Database Issues**

**Problem:** Tests interfere with each other or fail randomly

**Solution:**
```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.models.base import Base

@pytest.fixture(scope="function")  # Create new DB for each test
def db_session():
    # Use in-memory SQLite for tests
    engine = create_engine("sqlite:///:memory:", echo=False)
    Base.metadata.create_all(engine)
    
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    
    yield session
    
    session.close()
```

### **4. Async/Await Issues**

**Problem:** Mixing sync and async code incorrectly

**Incorrect:**
```python
# DON'T do this
def test_async_function():
    result = some_async_function()  # This won't work
    assert result == expected
```

**Correct:**
```python
# DO this
@pytest.mark.asyncio
async def test_async_function():
    result = await some_async_function()
    assert result == expected
```

### **5. Environment Variable Issues**

**Problem:** Environment variables not loading

**Solution:**
```python
# app/core/config.py
from pydantic import BaseSettings
from pathlib import Path

class Settings(BaseSettings):
    class Config:
        env_file = ".env"
        env_file_encoding = 'utf-8'
        case_sensitive = False
    
    # Always provide defaults or make fields optional
    database_url: str = "sqlite:///./app.db"
    debug: bool = False
```

---

## 📝 **Development Workflow Template**

### **For Each New Module, Follow This Pattern:**

#### **1. Plan Phase (5-10 minutes)**
```markdown
## Module: [MODULE_NAME]
### Purpose: What does this module do?
### Dependencies: What other modules does it need?
### Interfaces: What will other modules expect from it?
### Tests: What scenarios need testing?
```

#### **2. Test First (TDD Approach)**
```bash
# Create test file first
touch tests/unit/test_[module_name].py

# Write failing tests
pytest tests/unit/test_[module_name].py -v
# Expected: tests should fail (red)

# Implement minimal code to make tests pass
# Run tests again
pytest tests/unit/test_[module_name].py -v
# Expected: tests should pass (green)

# Refactor if needed
# Run tests to ensure nothing breaks
pytest tests/unit/test_[module_name].py -v
# Expected: tests still pass (green)
```

#### **3. Implementation Pattern**
```python
# Always follow this structure:

# 1. Imports (standard library first, then third-party, then local)
import os
from typing import Optional, List

from sqlalchemy import Column, Integer, String
from pydantic import BaseModel

from app.core.config import settings
from app.models.base import Base

# 2. Type definitions and constants
DEFAULT_PAGE_SIZE = 20
MAX_PAGE_SIZE = 100

# 3. Main class or functions
class SomeService:
    def __init__(self, dependency):
        self.dependency = dependency
    
    def public_method(self):
        """Public method with docstring."""
        return self._private_method()
    
    def _private_method(self):
        """Private method with docstring."""
        pass

# 4. Factory functions or module initialization
def get_service() -> SomeService:
    return SomeService(dependency)
```

#### **4. Integration Testing**
```bash
# After unit tests pass, run integration tests
pytest tests/integration/test_[module_name].py -v

# Run all tests to ensure no regressions
pytest tests/ -v

# Check test coverage
pytest --cov=app tests/ --cov-report=html
# Open htmlcov/index.html to see coverage report
```

---

## 🎯 **Success Metrics for Each Module**

### **Before Moving to Next Module, Ensure:**
- [ ] **All tests pass** (100% test success rate)
- [ ] **Code coverage > 80%** for the module
- [ ] **No linting errors** (`black`, `isort`, `flake8`)
- [ ] **Type checking passes** (`mypy app/[module]`)
- [ ] **Documentation updated** (docstrings and README if needed)
- [ ] **Integration tests pass** (if applicable)
- [ ] **Performance is acceptable** (if applicable)

### **Quality Gates Checklist:**
```bash
# Code Quality
black --check app/[module]
isort --check app/[module]
flake8 app/[module]
mypy app/[module]

# Testing
pytest tests/unit/test_[module] -v --cov=app/[module] --cov-report=term-missing

# Integration (if applicable)
pytest tests/integration/test_[module] -v

# All tests still pass
pytest tests/ -v
```

---

## 🔧 **Useful Development Commands**

### **Daily Development Workflow:**
```bash
# Start your day
git pull origin main
docker-compose up -d postgres redis
source venv/bin/activate

# Before coding
pytest tests/ -v  # Ensure everything works

# While coding (run frequently)
pytest tests/unit/test_[current_module].py -v

# Before committing
black app/ tests/
isort app/ tests/
flake8 app/ tests/
pytest tests/ -v

# Commit
git add .
git commit -m "feat: implement [module] with tests"
git push origin feature/[module-name]
```

### **Debugging Commands:**
```bash
# Debug failing tests
pytest tests/unit/test_[module].py -v -s --tb=long

# Debug with breakpoints
pytest tests/unit/test_[module].py -v -s --pdb

# Check database state
python -c "
from app.core.database import SessionLocal
from app.models import User
with SessionLocal() as db:
    users = db.query(User).all()
    print(f'Found {len(users)} users')
"

# Check Redis state
redis-cli -h localhost -p 6379
> keys *
> get some_key
```

### **Performance Testing:**
```bash
# Basic performance test
python -m timeit "import app.services.user_service; print('Import time')"

# Memory usage
python -c "
import tracemalloc
tracemalloc.start()
# Your code here
current, peak = tracemalloc.get_traced_memory()
print(f'Current memory usage: {current / 1024 / 1024:.1f} MB')
print(f'Peak memory usage: {peak / 1024 / 1024:.1f} MB')
"
```

---

## 📚 **Learning Resources**

### **FastAPI Specific:**
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [SQLAlchemy ORM Tutorial](https://docs.sqlalchemy.org/en/14/orm/tutorial.html)
- [Pydantic Documentation](https://pydantic-docs.helpmanual.io/)

### **Testing:**
- [Pytest Documentation](https://docs.pytest.org/)
- [Testing FastAPI Apps](https://fastapi.tiangolo.com/tutorial/testing/)

### **Database:**
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Alembic Documentation](https://alembic.sqlalchemy.org/)

---

**Remember:** Start small, test everything, and build incrementally. The order matters - stick to it, and you'll have a solid, maintainable backend!