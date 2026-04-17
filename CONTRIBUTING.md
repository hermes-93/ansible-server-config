# Contributing

## Development Setup

```bash
git clone https://github.com/hermes-93/ansible-server-config.git
cd ansible-server-config
pip install -r requirements-dev.txt
ansible-galaxy collection install -r requirements.yml
```

## Project Structure

- `roles/<name>/defaults/main.yml` — all role variables with defaults
- `roles/<name>/tasks/main.yml` — role tasks
- `roles/<name>/templates/` — Jinja2 templates
- `roles/<name>/handlers/main.yml` — service restart handlers
- `molecule/<scenario>/` — Molecule test scenarios

## Making Changes

1. Fork and create a feature branch: `git checkout -b feature/my-change`
2. Edit role tasks or templates
3. Run tests locally: `molecule test --scenario-name default`
4. Check linting: `yamllint . && ansible-lint --profile production`
5. Open a Pull Request — CI runs automatically

## Adding a New Role

1. Create directory: `roles/<name>/{tasks,defaults,handlers,templates,meta}/`
2. Add a `tasks/main.yml` with tagged tasks
3. Add defaults to `defaults/main.yml`
4. Register a handler if the role manages a service
5. Add role to `molecule/default/converge.yml`
6. Add molecule verify checks to `molecule/default/verify.yml`

## Commit Guidelines

- Present tense, imperative: `add`, `fix`, `remove` (not `added`, `fixed`)
- Scope prefix when helpful: `feat(nginx):`, `fix(security):`, `docs:`
- Keep commits focused — one logical change per commit

## Linting Rules

- **yamllint**: extends `default`, max line 160, commas disabled
- **ansible-lint**: production profile, `var-naming[no-role-prefix]` skipped (domain-prefixed vars are intentional)
