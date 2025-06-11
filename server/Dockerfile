# Use official Python image as base image
FROM python:3.11-slim-bullseye as base

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    PIP_DEFAULT_TIMEOUT=100 \
    POETRY_VERSION=1.8.2 \
    PYTHONFAULTHANDLER=1 \
    PYTHONHASHSEED=random \
    PYTHONPATH=/app

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    gettext \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Poetry
RUN pip install "poetry==$POETRY_VERSION"

# Copy only requirements to cache them in docker layer
COPY pyproject.toml poetry.lock* ./

# Install Python dependencies using Poetry
RUN poetry config virtualenvs.create false \
    && poetry install --no-interaction --no-ansi --no-root --only main

# Copy project
COPY . .

# Production stage
FROM base as production

# Install production dependencies
RUN poetry install --no-interaction --no-ansi --only main --no-root --no-dev

# Set environment variables for production
ENV DJANGO_SETTINGS_MODULE=server.settings.production \
    PORT=8000 \
    WEB_CONCURRENCY=4

# Create and switch to a non-root user
RUN useradd -m myuser && chown -R myuser:myuser /app
USER myuser

# Collect static files
RUN python manage.py collectstatic --noinput

# Run database migrations
RUN python manage.py migrate --noinput

# Use gunicorn as the production server
CMD gunicorn \
    --bind 0.0.0.0:$PORT \
    --workers $WEB_CONCURRENCY \
    --worker-tmp-dir /dev/shm \
    --timeout 120 \
    --keep-alive 5 \
    --access-logfile - \
    --error-logfile - \
    --log-level=info \
    server.wsgi:application

# Development stage
FROM base as development

# Install development dependencies
RUN poetry install --no-interaction --no-ansi --no-root

# Set environment variables for development
ENV DJANGO_SETTINGS_MODULE=server.settings.development \
    DEBUG=True

# Expose the port the app runs on
EXPOSE 8000

# Run the development server with auto-reload
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
