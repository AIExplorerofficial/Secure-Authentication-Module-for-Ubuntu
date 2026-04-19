# Contributing

Thank you for your interest in contributing to this project.

## Reporting Issues

If you find a bug or have a suggestion, please open an issue on GitHub with:
- A clear description of the problem
- Steps to reproduce it
- Your Ubuntu version and any relevant output

## Security Vulnerabilities

Do not open a public issue for security vulnerabilities. Instead, contact the maintainer directly via GitHub.

## Pull Requests

1. Fork the repository
2. Create a new branch (`git checkout -b fix/your-fix`)
3. Make your changes
4. Test that the project still builds with `make` and `make test`
5. Submit a pull request with a clear description of what you changed and why

## Code Style

- Follow the existing C code style in the codebase
- All new C code should compile without warnings under `-Wall -Wextra -Wpedantic`
- Do not introduce hardcoded credentials, bypass flags, or debug shortcuts
