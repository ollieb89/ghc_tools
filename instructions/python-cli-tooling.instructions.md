---
description: 'Python CLI application development best practices using Click, Typer, and Rich'
applyTo: '**/*.py'
---

# Python CLI Development Best Practices

Guidelines for building professional, user-friendly command-line applications in Python.

## CLI Framework Selection

### When to Use Click

Use Click for complex CLIs requiring:
- Explicit control over every aspect
- Complex nested command groups
- Custom parameter types
- Callback-based workflows

### When to Use Typer

Use Typer for simpler CLIs with:
- Type hints-based development
- Automatic help generation
- Less boilerplate code
- Modern Python (3.7+) features

## Project Structure

### Recommended Layout

```
my-cli/
├── pyproject.toml
├── src/
│   └── my_cli/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli.py
│       └── commands/
│           ├── __init__.py
│           └── subcommand.py
└── tests/
    └── test_cli.py
```

### Entry Point Configuration

```toml
# pyproject.toml
[project.scripts]
my-cli = "my_cli.cli:cli"
```

## Command Design

### Argument vs Option

```python
# ❌ BAD: Everything as options
@click.command()
@click.option("--input-file", required=True)
def process(input_file):
    pass

# ✅ GOOD: Required as argument, optional as option
@click.command()
@click.argument("input_file", type=click.Path(exists=True))
@click.option("--output", "-o", default="-", help="Output file [default: stdout]")
def process(input_file, output):
    pass
```

### Option Naming

```python
# ❌ BAD: Inconsistent, unclear
@click.option("--v")
@click.option("--outputDir")
@click.option("-verbose")

# ✅ GOOD: Consistent short/long, clear names
@click.option("--verbose", "-v", is_flag=True)
@click.option("--output-dir", "-o", help="Output directory")
@click.option("--dry-run", "-n", is_flag=True, help="Show what would be done")
```

### Boolean Flags

```python
# ❌ BAD: Requires value
@click.option("--verbose", type=bool, default=False)

# ✅ GOOD: Simple flag
@click.option("--verbose", "-v", is_flag=True, help="Enable verbose output")

# ✅ GOOD: Flag with negation
@click.option("--color/--no-color", default=True, help="Enable/disable colored output")
```

## Help Text

### Command Help

```python
# ❌ BAD: No help, unclear purpose
@click.command()
def sync():
    pass

# ✅ GOOD: Clear help with examples
@click.command()
def sync():
    """Synchronize local files with remote storage.

    Examples:
        my-cli sync ./data
        my-cli sync ./data --dry-run
        my-cli sync ./data --delete-missing
    """
    pass
```

### Option Help

```python
# ❌ BAD: No help text
@click.option("--format")

# ✅ GOOD: Descriptive with default shown
@click.option(
    "--format", "-f",
    type=click.Choice(["json", "yaml", "toml"]),
    default="json",
    help="Output format [default: json]"
)
```

## Input Validation

### File Paths

```python
# ❌ BAD: No validation
@click.argument("path")

# ✅ GOOD: Validated path
@click.argument("path", type=click.Path(exists=True, dir_okay=False, readable=True))

# ✅ GOOD: Optional file with stdin fallback
@click.argument("input", type=click.File("r"), default="-")
```

### Numeric Ranges

```python
# ❌ BAD: Manual validation
@click.option("--port", type=int)
def serve(port):
    if port < 1 or port > 65535:
        raise click.BadParameter("Port must be 1-65535")

# ✅ GOOD: Built-in range
@click.option("--port", type=click.IntRange(1, 65535), default=8000)
```

### Choices

```python
# ❌ BAD: Manual validation
@click.option("--level")
def log(level):
    if level not in ["debug", "info", "warning", "error"]:
        raise ValueError("Invalid level")

# ✅ GOOD: Choice type
@click.option("--level", type=click.Choice(["debug", "info", "warning", "error"]))
```

## Output Formatting

### Status Messages

```python
from rich.console import Console

console = Console()

# ❌ BAD: Plain print
print("Success!")
print("Error: something failed")

# ✅ GOOD: Colored status
console.print("[green]✓[/green] Success!")
console.print("[red]✗[/red] Error: something failed")
console.print("[yellow]⚠[/yellow] Warning: check configuration")
```

### Tables

```python
from rich.table import Table

# ❌ BAD: Manual formatting
for item in items:
    print(f"{item.name}\t{item.size}\t{item.date}")

# ✅ GOOD: Rich table
table = Table(title="Files")
table.add_column("Name", style="cyan")
table.add_column("Size", justify="right")
table.add_column("Date")

for item in items:
    table.add_row(item.name, str(item.size), item.date)

console.print(table)
```

### Progress Bars

```python
from rich.progress import track

# ❌ BAD: No progress indication
for item in items:
    process(item)

# ✅ GOOD: Progress bar
for item in track(items, description="Processing..."):
    process(item)

# ✅ GOOD: Manual progress for more control
from rich.progress import Progress

with Progress() as progress:
    task = progress.add_task("Downloading...", total=100)
    while not done:
        progress.update(task, advance=1)
```

## Error Handling

### User-Friendly Errors

```python
# ❌ BAD: Stack trace for user errors
def load_config(path):
    with open(path) as f:
        return json.load(f)  # JSONDecodeError if invalid

# ✅ GOOD: Catch and explain
def load_config(path):
    try:
        with open(path) as f:
            return json.load(f)
    except FileNotFoundError:
        console.print(f"[red]Error:[/red] Config file not found: {path}")
        raise SystemExit(1)
    except json.JSONDecodeError as e:
        console.print(f"[red]Error:[/red] Invalid JSON in {path}")
        console.print(f"[dim]  Line {e.lineno}: {e.msg}[/dim]")
        raise SystemExit(1)
```

### Exit Codes

```python
# ❌ BAD: Always exit 0 or 1
sys.exit(1)  # What went wrong?

# ✅ GOOD: Meaningful exit codes
EXIT_SUCCESS = 0
EXIT_USAGE_ERROR = 2
EXIT_CONFIG_ERROR = 3
EXIT_RUNTIME_ERROR = 4

if not config_valid:
    console.print("[red]Invalid configuration[/red]")
    raise SystemExit(EXIT_CONFIG_ERROR)
```

## Configuration

### Environment Variables

```python
# ❌ BAD: Manual env var handling
import os
api_key = os.environ.get("API_KEY")

# ✅ GOOD: Click auto_envvar_prefix
@click.group()
@click.option("--api-key", envvar="MYAPP_API_KEY", help="API key [env: MYAPP_API_KEY]")
def cli(api_key):
    pass

# Or with explicit envvar
@click.option("--token", envvar="GITHUB_TOKEN")
```

### Config File Integration

```python
import tomllib
from pathlib import Path

def load_config():
    """Load config from standard locations."""
    paths = [
        Path.cwd() / "myapp.toml",
        Path.home() / ".config" / "myapp" / "config.toml",
    ]
    
    for path in paths:
        if path.exists():
            with open(path, "rb") as f:
                return tomllib.load(f)
    
    return {}
```

## Testing

### Basic CLI Test

```python
from click.testing import CliRunner
from my_cli.cli import cli

def test_version():
    runner = CliRunner()
    result = runner.invoke(cli, ["--version"])
    assert result.exit_code == 0
    assert "1.0.0" in result.output
```

### Command Test with Arguments

```python
def test_create_command():
    runner = CliRunner()
    with runner.isolated_filesystem():
        result = runner.invoke(cli, ["create", "myproject", "--template", "basic"])
        assert result.exit_code == 0
        assert Path("myproject").exists()
```

### Test Error Handling

```python
def test_missing_file_error():
    runner = CliRunner()
    result = runner.invoke(cli, ["process", "nonexistent.txt"])
    assert result.exit_code != 0
    assert "not found" in result.output.lower()
```

## Common Pitfalls

| Problem | Cause | Solution |
|---------|-------|----------|
| Help text not showing | Missing docstring | Add docstring to command function |
| Options not working | Wrong decorator order | `@click.command()` must be last (top) |
| Context not available | Not using `@click.pass_context` | Add decorator and ctx parameter |
| Subcommands not found | Not registered with group | Use `@group.command()` or `group.add_command()` |
| Progress bar flickering | Printing during progress | Use `progress.console.print()` instead |
| Colors not showing | Windows terminal | Install `colorama` or use `--force-terminal` |

## Packaging Checklist

- [ ] pyproject.toml with `[project.scripts]` entry point
- [ ] `__version__` in `__init__.py`
- [ ] `__main__.py` for `python -m` support
- [ ] `--version` flag on main command
- [ ] `--help` automatically provided by Click/Typer
- [ ] README with installation and usage examples
- [ ] Tests with CliRunner covering main paths
