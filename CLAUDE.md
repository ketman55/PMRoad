# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Status

This is a fresh repository with no source code implemented yet. The project appears to be named "PMRoad" based on the directory structure.

## Development Setup

Since this is a new repository, the development environment and build system have not been established yet. Common setup tasks will likely include:

- Initializing a package manager (npm, yarn, pip, cargo, etc.)
- Setting up a build system and development scripts
- Configuring linting and formatting tools
- Establishing testing frameworks

### Environment Configuration

This repository uses environment variables for sensitive configuration:

1. Copy `.env.example` to `.env`:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` with your actual credentials:
   - `GITHUB_TOKEN`: Your GitHub personal access token
   - `GITHUB_USERNAME`: Your GitHub username  
   - `GITHUB_EMAIL`: Your GitHub email address

3. The `.env` file is automatically ignored by git for security.

## Project Structure

The project structure has not been established yet. Once source code is added, this section should be updated with:

- Main application entry points
- Directory organization
- Key architectural patterns
- Important configuration files

## Creating Pull Requests

To create pull requests, ensure environment is configured:

1. Set up `.env` file with GitHub credentials (see Environment Configuration above)
2. Use environment variables for authentication:
   ```bash
   export GITHUB_TOKEN=$(grep GITHUB_TOKEN .env | cut -d '=' -f2)
   ```
3. Create and push feature branches for pull requests

## Notes for Claude Code

- This repository is in its initial state
- No specific development commands or build processes have been configured
- Architecture and patterns should be established as the codebase grows
- Update this file as the project structure becomes more defined
- Always reference `.env.example` for required environment variables
- Never commit actual credentials to git repository