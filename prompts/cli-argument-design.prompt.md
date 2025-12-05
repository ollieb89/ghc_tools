# CLI Argument Design Prompt

Design optimal command-line arguments and options for a CLI command.

## Input Requirements

Provide:
- **Command Name**: The command to design arguments for
- **Purpose**: What the command does
- **Required Inputs**: Data the command must receive
- **Optional Behaviors**: Configurable aspects
- **Framework**: `click` | `typer`

## Example Input

```
Command Name: sync
Purpose: Synchronize files between local directory and remote storage
Required Inputs:
  - Source path (local directory)
  - Destination (remote URL or path)
Optional Behaviors:
  - Dry-run mode
  - Delete files not in source
  - Include/exclude patterns
  - Parallel upload count
  - Verbose logging
Framework: typer
```

## Design Principles

### 1. Arguments vs Options

| Use Arguments For | Use Options For |
|-------------------|-----------------|
| Required, positional data | Optional configuration |
| Primary subject of command | Flags that modify behavior |
| Naturally ordered inputs | Named settings |

### 2. Naming Conventions

```
# Good: Clear, discoverable names
--verbose, -v
--output, -o
--dry-run, -n
--force, -f
--quiet, -q
--recursive, -r

# Avoid: Ambiguous or overly short
--d  # Delete? Debug? Directory?
--x  # What does this do?
```

### 3. Type Safety

```python
# Use appropriate types for validation
from pathlib import Path
from typing import Annotated
import typer

# Path that must exist
source: Annotated[Path, typer.Argument(
    exists=True,
    file_okay=False,
    dir_okay=True,
    resolve_path=True,
    help="Source directory to sync from",
)]

# Constrained integer
workers: Annotated[int, typer.Option(
    "--workers", "-w",
    min=1,
    max=32,
    help="Number of parallel workers",
)] = 4

# Choice from predefined values
log_level: Annotated[str, typer.Option(
    "--log-level",
    click_type=click.Choice(["debug", "info", "warning", "error"]),
    help="Set logging verbosity",
)] = "info"
```

## Output Template

### Command Signature (Typer)

```python
@app.command()
def {command_name}(
    # === ARGUMENTS (Required, Positional) ===
    {arg_name}: Annotated[{type}, typer.Argument(
        help="{description}",
        # Add constraints as needed
    )],
    
    # === OPTIONS (Named, Usually Optional) ===
    {option_name}: Annotated[{type}, typer.Option(
        "--{long-name}", "-{short}",
        help="{description}",
    )] = {default},
    
    # === FLAGS (Boolean Options) ===
    {flag_name}: Annotated[bool, typer.Option(
        "--{flag-name}/--no-{flag-name}",
        "-{short}",
        help="{description}",
    )] = False,
) -> None:
    """
    {Command docstring - becomes help text}
    
    Example:
        $ {cli-name} {command} {example-usage}
    """
    pass
```

### Command Signature (Click)

```python
@cli.command()
@click.argument("{arg_name}", type={type})
@click.option(
    "--{option-name}", "-{short}",
    type={type},
    default={default},
    help="{description}",
)
@click.option(
    "--{flag-name}/--no-{flag-name}", "-{short}",
    default=False,
    help="{description}",
)
def {command_name}({arg_name}, {option_name}, {flag_name}):
    """
    {Command docstring}
    
    Example:
        $ {cli-name} {command} {example-usage}
    """
    pass
```

## Example Output

For the sync command input:

```python
from pathlib import Path
from typing import Annotated, Optional
import typer

@app.command()
def sync(
    # Required arguments
    source: Annotated[Path, typer.Argument(
        exists=True,
        file_okay=False,
        dir_okay=True,
        resolve_path=True,
        help="Local directory to sync from",
    )],
    destination: Annotated[str, typer.Argument(
        help="Remote destination (URL or path)",
    )],
    
    # Behavioral options
    dry_run: Annotated[bool, typer.Option(
        "--dry-run", "-n",
        help="Show what would be synced without making changes",
    )] = False,
    delete: Annotated[bool, typer.Option(
        "--delete", "-d",
        help="Delete remote files not present in source",
    )] = False,
    
    # Filtering options
    include: Annotated[Optional[list[str]], typer.Option(
        "--include", "-i",
        help="Glob patterns to include (can be used multiple times)",
    )] = None,
    exclude: Annotated[Optional[list[str]], typer.Option(
        "--exclude", "-e",
        help="Glob patterns to exclude (can be used multiple times)",
    )] = None,
    
    # Performance options
    workers: Annotated[int, typer.Option(
        "--workers", "-w",
        min=1,
        max=32,
        help="Number of parallel upload workers",
    )] = 4,
    
    # Output control
    verbose: Annotated[bool, typer.Option(
        "--verbose", "-v",
        help="Enable verbose output",
    )] = False,
    quiet: Annotated[bool, typer.Option(
        "--quiet", "-q",
        help="Suppress all output except errors",
    )] = False,
) -> None:
    """
    Synchronize files between local directory and remote storage.
    
    Uploads new and modified files from SOURCE to DESTINATION.
    Use --dry-run to preview changes without syncing.
    
    Examples:
        # Basic sync
        $ mytool sync ./data s3://bucket/backup
        
        # Dry run with verbose output
        $ mytool sync ./data s3://bucket/backup --dry-run -v
        
        # Sync only Python files, excluding tests
        $ mytool sync ./src s3://bucket/code -i "*.py" -e "*_test.py"
        
        # Mirror (delete files not in source)
        $ mytool sync ./data s3://bucket/backup --delete
    """
    if verbose and quiet:
        raise typer.BadParameter("Cannot use both --verbose and --quiet")
    
    # Implementation here
```

## Common Patterns

### Mutually Exclusive Options
```python
# Using callback for validation
def validate_output_format(ctx, param, value):
    if ctx.params.get("json_output") and ctx.params.get("csv_output"):
        raise click.BadParameter("Cannot use both --json and --csv")
    return value

@click.option("--json", "json_output", is_flag=True, callback=validate_output_format)
@click.option("--csv", "csv_output", is_flag=True)
```

### Repeatable Options
```python
# Can be used multiple times: -t dev -t prod
tags: Annotated[Optional[list[str]], typer.Option(
    "--tag", "-t",
    help="Tags to apply (can repeat)",
)] = None
```

### File Input with Stdin Support
```python
# Accept filename or "-" for stdin
input_file: Annotated[typer.FileText, typer.Option(
    "--input", "-i",
    help="Input file (use '-' for stdin)",
)] = "-"
```

### Password Prompt
```python
password: Annotated[str, typer.Option(
    "--password", "-p",
    prompt=True,
    hide_input=True,
    confirmation_prompt=True,
    help="Password for authentication",
)]
```

## Help Text Quality Checklist

- [ ] Every argument has a help string
- [ ] Every option has a help string
- [ ] Command has a detailed docstring with examples
- [ ] Default values are mentioned where non-obvious
- [ ] Constraints (min/max, choices) are documented
- [ ] Related options are grouped logically
- [ ] Short flags don't conflict with common conventions
