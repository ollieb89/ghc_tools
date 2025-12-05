---
description: 'Add a new command to an existing CLI project - follows existing patterns, adds tests, updates help'
---

# CLI Command Prompt

Add a new command or subcommand to an existing Python CLI project.

## Input Requirements

Provide the following:
- **CLI Project**: Path to existing CLI project
- **Command Name**: Name of the new command (e.g., `export`, `sync`)
- **Parent Command**: If subcommand, which command it belongs to
- **Description**: What the command does
- **Arguments**: Required positional arguments
- **Options**: Optional flags and parameters
- **Behavior**: Expected functionality

## Example Input

```
CLI Project: apps/cli/
Command Name: export
Parent Command: data (subcommand of `mytool data export`)
Description: Export data to various formats (JSON, CSV, YAML)
Arguments:
  - source: Data source to export (required)
Options:
  --format: Output format (json|csv|yaml), default: json
  --output: Output file path (default: stdout)
  --pretty: Pretty print output (flag)
  --filter: Filter expression
Behavior:
  1. Load data from source
  2. Apply filter if provided
  3. Convert to specified format
  4. Write to output or stdout
```

## Implementation Workflow

### Step 1: Analyze Existing CLI Structure

```bash
# Find the main CLI entry point
grep -r "app = typer.Typer\|@click.group" --include="*.py"

# Understand command organization
tree -P "*.py" --prune apps/cli/src/
```

**Key Questions:**
- Framework: Click or Typer?
- Pattern: Single file or multi-file commands?
- Naming: Snake_case or kebab-case?
- Testing: pytest-click or typer.testing?

### Step 2: Follow Existing Patterns

**Typer Pattern (if existing CLI uses Typer):**

```python
# apps/cli/src/cli/commands/data.py
import typer
from typing import Annotated, Optional
from pathlib import Path
from rich.console import Console

app = typer.Typer(help="Data management commands")
console = Console()

@app.command()
def export(
    source: Annotated[str, typer.Argument(help="Data source to export")],
    format: Annotated[str, typer.Option(
        "--format", "-f",
        help="Output format"
    )] = "json",
    output: Annotated[Optional[Path], typer.Option(
        "--output", "-o",
        help="Output file path (default: stdout)"
    )] = None,
    pretty: Annotated[bool, typer.Option(
        "--pretty", "-p",
        help="Pretty print output"
    )] = False,
    filter: Annotated[Optional[str], typer.Option(
        "--filter",
        help="Filter expression"
    )] = None,
) -> None:
    """Export data to various formats (JSON, CSV, YAML)."""
    try:
        # Load data
        data = load_data(source)
        
        # Apply filter
        if filter:
            data = apply_filter(data, filter)
        
        # Format output
        formatted = format_output(data, format, pretty)
        
        # Write output
        if output:
            output.write_text(formatted)
            console.print(f"[green]✓[/green] Exported to {output}")
        else:
            console.print(formatted)
            
    except DataError as e:
        console.print(f"[red]Error:[/red] {e}")
        raise typer.Exit(code=1)
```

**Click Pattern (if existing CLI uses Click):**

```python
# apps/cli/src/cli/commands/data.py
import click
from pathlib import Path

@click.group()
def data():
    """Data management commands."""
    pass

@data.command()
@click.argument("source")
@click.option("--format", "-f", default="json", 
              type=click.Choice(["json", "csv", "yaml"]),
              help="Output format")
@click.option("--output", "-o", type=click.Path(), 
              help="Output file path")
@click.option("--pretty", "-p", is_flag=True, 
              help="Pretty print output")
@click.option("--filter", help="Filter expression")
def export(source: str, format: str, output: str | None, 
           pretty: bool, filter: str | None) -> None:
    """Export data to various formats (JSON, CSV, YAML)."""
    try:
        data = load_data(source)
        
        if filter:
            data = apply_filter(data, filter)
        
        formatted = format_output(data, format, pretty)
        
        if output:
            Path(output).write_text(formatted)
            click.echo(click.style(f"✓ Exported to {output}", fg="green"))
        else:
            click.echo(formatted)
            
    except DataError as e:
        click.echo(click.style(f"Error: {e}", fg="red"), err=True)
        raise click.Abort()
```

### Step 3: Register the Command

**Typer Registration:**

```python
# apps/cli/src/cli/main.py
from cli.commands import data

app = typer.Typer()
app.add_typer(data.app, name="data")
```

**Click Registration:**

```python
# apps/cli/src/cli/main.py
from cli.commands.data import data

@click.group()
def cli():
    pass

cli.add_command(data)
```

### Step 4: Add Tests

**Typer Tests:**

```python
# tests/commands/test_data_export.py
import pytest
from typer.testing import CliRunner
from cli.main import app

runner = CliRunner()

class TestDataExport:
    """Tests for the data export command."""
    
    def test_export_json_default(self, sample_data_source):
        """Export with default JSON format."""
        result = runner.invoke(app, ["data", "export", sample_data_source])
        assert result.exit_code == 0
        assert '"name":' in result.stdout  # JSON output
    
    def test_export_csv_format(self, sample_data_source):
        """Export with CSV format."""
        result = runner.invoke(app, [
            "data", "export", sample_data_source,
            "--format", "csv"
        ])
        assert result.exit_code == 0
        assert "name,value" in result.stdout  # CSV header
    
    def test_export_to_file(self, sample_data_source, tmp_path):
        """Export to output file."""
        output_file = tmp_path / "output.json"
        result = runner.invoke(app, [
            "data", "export", sample_data_source,
            "--output", str(output_file)
        ])
        assert result.exit_code == 0
        assert output_file.exists()
        assert "Exported to" in result.stdout
    
    def test_export_pretty_print(self, sample_data_source):
        """Export with pretty printing."""
        result = runner.invoke(app, [
            "data", "export", sample_data_source,
            "--pretty"
        ])
        assert result.exit_code == 0
        assert "\n  " in result.stdout  # Indented output
    
    def test_export_with_filter(self, sample_data_source):
        """Export with filter expression."""
        result = runner.invoke(app, [
            "data", "export", sample_data_source,
            "--filter", "status=active"
        ])
        assert result.exit_code == 0
        # Verify filter applied
    
    def test_export_invalid_source(self):
        """Handle invalid source gracefully."""
        result = runner.invoke(app, ["data", "export", "nonexistent"])
        assert result.exit_code == 1
        assert "Error:" in result.stdout
    
    def test_export_help(self):
        """Verify help text."""
        result = runner.invoke(app, ["data", "export", "--help"])
        assert result.exit_code == 0
        assert "Export data to various formats" in result.stdout
        assert "--format" in result.stdout
        assert "--output" in result.stdout
```

**Click Tests:**

```python
# tests/commands/test_data_export.py
import pytest
from click.testing import CliRunner
from cli.main import cli

runner = CliRunner()

class TestDataExport:
    def test_export_json_default(self, sample_data_source):
        result = runner.invoke(cli, ["data", "export", sample_data_source])
        assert result.exit_code == 0
```

### Step 5: Update Help Text

Ensure help is comprehensive:

```python
@app.command()
def export(
    source: Annotated[str, typer.Argument(
        help="Data source to export. Can be a file path, URL, or database query."
    )],
    format: Annotated[str, typer.Option(
        "--format", "-f",
        help="Output format: json (default), csv, or yaml"
    )] = "json",
    # ... more options
) -> None:
    """
    Export data to various formats (JSON, CSV, YAML).
    
    Examples:
    
        # Export to JSON (default)
        $ mytool data export users.db
        
        # Export to CSV file
        $ mytool data export users.db --format csv --output users.csv
        
        # Export with filter and pretty print
        $ mytool data export users.db --filter "role=admin" --pretty
    """
```

### Step 6: Verify Integration

```bash
# Test help
uv run mytool data export --help

# Test basic functionality
uv run mytool data export test_source

# Test all options
uv run mytool data export test_source --format csv --output test.csv --pretty

# Run tests
uv run pytest tests/commands/test_data_export.py -v
```

## Output Checklist

```
✅ Command Implementation Checklist:
□ Command follows existing CLI patterns
□ All arguments have help text
□ All options have short flags where appropriate
□ Error handling with proper exit codes
□ Progress indicators for long operations
□ Tests cover happy path and error cases
□ Help text includes examples
□ Command registered in main CLI
```

## Related Resources

- **Agent**: `ghc_tools/agents/python-cli-tooling.agent.md`
- **Instructions**: `ghc_tools/instructions/python-cli-tooling.instructions.md`
- **Chatmode**: `ghc_tools/chatmodes/python-cli-tooling.chatmode.md`
- **Scaffold**: `ghc_tools/prompts/cli-scaffold.prompt.md`
