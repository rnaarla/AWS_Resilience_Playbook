# Contributing to AWS Resilience Playbook

Thank you for your interest in contributing! This document provides guidelines for contributing to the AWS Resilience Playbook.

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [How to Contribute](#how-to-contribute)
3. [Development Setup](#development-setup)
4. [Contribution Guidelines](#contribution-guidelines)
5. [Pull Request Process](#pull-request-process)

---

## Code of Conduct

This project adheres to the Contributor Covenant [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code.

---

## How to Contribute

### Reporting Bugs

- Check existing issues first
- Use the bug report template
- Provide detailed reproduction steps
- Include environment details

### Suggesting Enhancements

- Check if enhancement is already suggested
- Use the feature request template
- Explain the use case clearly
- Provide examples if possible

### Documentation Improvements

- Fix typos and grammatical errors
- Clarify confusing sections
- Add missing examples
- Update outdated information

### Code Contributions

- Implementation of new patterns
- Bug fixes
- Test improvements
- Tool enhancements

---

## Development Setup

### Prerequisites

- Git
- Python 3.9+ (for scripts)
- AWS CLI (for examples)
- Markdown linter (markdownlint)

### Setup Steps

```bash
# Clone the repository
git clone https://github.com/rnaarla/AWS_Resilience_Playbook.git
cd AWS_Resilience_Playbook

# Create virtual environment (optional, for Python scripts)
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt  # If exists

# Install pre-commit hooks
pre-commit install  # If configured
```

---

## Contribution Guidelines

### Documentation Standards

- Use clear, concise language
- Follow existing document structure
- Include code examples where appropriate
- Add references for external concepts
- Update table of contents when adding sections

### Code Standards

- Follow language-specific style guides (PEP 8 for Python, etc.)
- Include docstrings for functions
- Add unit tests for new code
- Ensure examples are runnable
- Comment complex logic

### Commit Messages

Use conventional commit format:

```
type(scope): subject

body (optional)

footer (optional)
```

**Types:**

- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Formatting changes
- `refactor`: Code refactoring
- `test`: Test additions/changes
- `chore`: Maintenance tasks

**Examples:**

```
docs(architecture): add control plane deployment guide

feat(chaos): implement new cascading failure scenario

fix(metrics): correct MTTD calculation formula
```

---

## Pull Request Process

### Before Submitting

1. **Update your fork**:
   ```bash
   git fetch upstream
   git checkout main
   git merge upstream/main
   ```

2. **Create a feature branch**:
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes**:
   - Follow contribution guidelines
   - Test your changes
   - Update documentation

4. **Commit your changes**:
   ```bash
   git add .
   git commit -m "type(scope): description"
   ```

5. **Push to your fork**:
   ```bash
   git push origin feature/your-feature-name
   ```

### Submitting the PR

1. Go to the [repository](https://github.com/rnaarla/AWS_Resilience_Playbook)
2. Click "New Pull Request"
3. Select your branch
4. Fill out the PR template:
   - **Title**: Clear, descriptive title
   - **Description**: What and why
   - **Testing**: How you tested
   - **Checklist**: Complete all items

### PR Template

```markdown
## Description
<!-- Describe your changes -->

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Breaking change

## Testing
<!-- How did you test your changes? -->

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Tests added/updated
- [ ] All tests passing
```

### Review Process

1. **Automated checks**: Must pass CI/CD
2. **Peer review**: At least one approval required
3. **Maintainer review**: Final approval from maintainer
4. **Merge**: Squash and merge to main

### After Merge

- Delete your feature branch
- Update your local repository
- Close related issues

---

## Recognition

Contributors will be recognized in:
- GitHub contributors list
- Release notes (for significant contributions)
- README acknowledgments (for major features)

---

## Getting Help

- **Questions**: Open a [Discussion](https://github.com/rnaarla/AWS_Resilience_Playbook/discussions)
- **Issues**: Open an [Issue](https://github.com/rnaarla/AWS_Resilience_Playbook/issues)
- **Security**: See [SECURITY.md](SECURITY.md)

---

## License

By contributing, you agree that your contributions will be licensed under the [CC BY-SA 4.0 License](LICENSE).

---

**Thank you for contributing!** 🎉
