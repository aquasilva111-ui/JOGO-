# CI — World Nation Simulator
#
# O token de acesso atual não tem permissão para criar workflows via API
# (escopo `workflow`). Para instalar este arquivo:
#
#   1. No GitHub, entre no repositório → Add file → Create new file
#   2. Caminho: .github/workflows/ci.yml
#   3. Cole o conteúdo abaixo da linha e faça commit na branch kimi/fase-1-fundacao
#
# ---------------------------------------------------------------

name: CI

on:
  push:
    branches: [main, "kimi/**"]
  pull_request:
    branches: [main]

jobs:
  backend:
    name: Backend (pytest)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install -r backend/requirements.txt
      - name: Run simulation tests
        run: PYTHONPATH=backend python3 -m pytest backend/tests -q

  frontend:
    name: Frontend (lint + build)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install dependencies
        working-directory: frontend
        run: npm ci
      - name: Lint
        working-directory: frontend
        run: npm run lint
      - name: Build
        working-directory: frontend
        run: npm run build
