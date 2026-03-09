# syntax=docker/dockerfile:1

ARG PYTHON_VERSION=3.12

FROM python:${PYTHON_VERSION}-slim AS physical-ai-studio-cpu

ARG GIT_SHA=unknown
ARG APP_UID=1000
ARG APP_GID=1000

RUN groupadd --gid "${APP_GID}" appuser && \
    useradd --uid "${APP_UID}" --gid "${APP_GID}" --create-home appuser

WORKDIR /app
RUN mkdir -p data storage tmp application/backend application/ui && \
    chown -R appuser:appuser /app

USER appuser
EXPOSE 8000
CMD ["sh", "-c", "echo 'CI test container ready' && sleep infinity"]
