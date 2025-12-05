# CLI Test Cases Generator Prompt

Generate comprehensive test cases for a Python CLI application.

## Input Requirements

Provide:
- **CLI Module**: Import path to the CLI (e.g., `mypackage.cli`)
- **App Object**: The Typer/Click app object name (e.g., `app`, `cli`)
- **Commands**: List of commands to test with their signatures
- **Framework**: `click` | `typer`

## Example Input

```
CLI Module: file_organizer.cli
App Object: app
Framework: typer
Commands:
  - organize:
      args: [path]
      options: [--dry-run, --verbose, --pattern]
  - preview:
      args: [path]
      options: [--format (text|json)]
  - config:
      subcommands: [show, set, reset]
```

## Test Categories

### 1. Basic Functionality Tests
- Command executes successfully
- Help text displays correctly
- Version flag works

### 2. Argument Validation Tests
- Required arguments are enforced
- Type validation works (paths exist, integers in range)
- Invalid inputs produce clear errors

### 3. Option Tests
- Default values work correctly
- Options modify behavior as expected
- Flags toggle features properly

### 4. Edge Case Tests
- Empty inputs
- Special characters in paths
- Very long inputs
- Unicode handling

### 5. Error Handling Tests
- Missing required arguments
- Invalid option values
- File not found scenarios
- Permission errors

### 6. Output Tests
- Correct exit codes
- Expected output format
- JSON output is valid
- Verbose mode shows extra info

## Output Template

### Test File Structure

```python
"""Tests for {cli_module} CLI commands."""

import json
from pathlib import Path
from unittest.mock import patch, MagicMock

import pytest
from typer.testing import CliRunner  # or: from click.testing import CliRunner

from {cli_module} import {app_object}

runner = CliRunner()


# ============================================================
# Fixtures
# ============================================================

@pytest.fixture
def temp_dir(tmp_path: Path) -> Path:
    """Create a temporary directory with test files."""
    # Create test file structure
    (tmp_path / "file1.txt").write_text("content1")
    (tmp_path / "file2.py").write_text("print('hello')")
    (tmp_path / "subdir").mkdir()
    (tmp_path / "subdir" / "file3.txt").write_text("content3")
    return tmp_path


@pytest.fixture
def mock_api():
    """Mock external API calls."""
    with patch("{cli_module}.api.client") as mock:
        yield mock


# ============================================================
# Basic Functionality Tests
# ============================================================

class TestBasicFunctionality:
    """Test basic CLI functionality."""

    def test_help_displays(self):
        """CLI shows help text with --help flag."""
        result = runner.invoke({app_object}, ["--help"])
        assert result.exit_code == 0
        assert "{cli_name}" in result.stdout
        # Verify main commands are listed
        assert "organize" in result.stdout
        assert "preview" in result.stdout

    def test_version_displays(self):
        """CLI shows version with --version flag."""
        result = runner.invoke({app_object}, ["--version"])
        assert result.exit_code == 0
        assert "0." in result.stdout  # Version number present

    def test_command_help(self):
        """Individual commands show help."""
        result = runner.invoke({app_object}, ["organize", "--help"])
        assert result.exit_code == 0
        assert "--dry-run" in result.stdout


# ============================================================
# Command: organize
# ============================================================

class TestOrganizeCommand:
    """Tests for the 'organize' command."""

    def test_organize_basic(self, temp_dir: Path):
        """organize command runs successfully with valid path."""
        result = runner.invoke({app_object}, ["organize", str(temp_dir)])
        assert result.exit_code == 0

    def test_organize_dry_run(self, temp_dir: Path):
        """--dry-run shows changes without making them."""
        result = runner.invoke({app_object}, [
            "organize", str(temp_dir), "--dry-run"
        ])
        assert result.exit_code == 0
        assert "would" in result.stdout.lower()  # "Would move..."
        # Verify files weren't actually moved
        assert (temp_dir / "file1.txt").exists()

    def test_organize_verbose(self, temp_dir: Path):
        """--verbose shows detailed output."""
        result = runner.invoke({app_object}, [
            "organize", str(temp_dir), "--verbose"
        ])
        assert result.exit_code == 0
        # More output than non-verbose
        assert len(result.stdout) > 50

    def test_organize_with_pattern(self, temp_dir: Path):
        """--pattern filters files correctly."""
        result = runner.invoke({app_object}, [
            "organize", str(temp_dir), "--pattern", "*.txt"
        ])
        assert result.exit_code == 0
        # Should only process .txt files

    def test_organize_missing_path(self):
        """Missing path argument shows error."""
        result = runner.invoke({app_object}, ["organize"])
        assert result.exit_code != 0
        assert "missing" in result.stdout.lower() or "required" in result.stdout.lower()

    def test_organize_invalid_path(self):
        """Non-existent path shows clear error."""
        result = runner.invoke({app_object}, ["organize", "/nonexistent/path"])
        assert result.exit_code != 0
        assert "not exist" in result.stdout.lower() or "invalid" in result.stdout.lower()

    def test_organize_file_instead_of_dir(self, temp_dir: Path):
        """Passing a file instead of directory shows error."""
        file_path = temp_dir / "file1.txt"
        result = runner.invoke({app_object}, ["organize", str(file_path)])
        assert result.exit_code != 0
        assert "directory" in result.stdout.lower()


# ============================================================
# Command: preview
# ============================================================

class TestPreviewCommand:
    """Tests for the 'preview' command."""

    def test_preview_text_format(self, temp_dir: Path):
        """preview shows text output by default."""
        result = runner.invoke({app_object}, ["preview", str(temp_dir)])
        assert result.exit_code == 0
        # Should be human-readable, not JSON
        assert not result.stdout.startswith("{")

    def test_preview_json_format(self, temp_dir: Path):
        """--format json produces valid JSON."""
        result = runner.invoke({app_object}, [
            "preview", str(temp_dir), "--format", "json"
        ])
        assert result.exit_code == 0
        # Verify it's valid JSON
        data = json.loads(result.stdout)
        assert isinstance(data, (dict, list))

    def test_preview_empty_dir(self, tmp_path: Path):
        """preview handles empty directory gracefully."""
        result = runner.invoke({app_object}, ["preview", str(tmp_path)])
        assert result.exit_code == 0
        assert "no files" in result.stdout.lower() or "empty" in result.stdout.lower()


# ============================================================
# Command: config
# ============================================================

class TestConfigCommand:
    """Tests for the 'config' command and subcommands."""

    def test_config_show(self):
        """config show displays current configuration."""
        result = runner.invoke({app_object}, ["config", "show"])
        assert result.exit_code == 0

    def test_config_set_valid(self):
        """config set updates a valid key."""
        result = runner.invoke({app_object}, [
            "config", "set", "default_pattern", "*.txt"
        ])
        assert result.exit_code == 0

    def test_config_set_invalid_key(self):
        """config set rejects invalid keys."""
        result = runner.invoke({app_object}, [
            "config", "set", "nonexistent_key", "value"
        ])
        assert result.exit_code != 0
        assert "invalid" in result.stdout.lower() or "unknown" in result.stdout.lower()

    def test_config_reset(self):
        """config reset restores defaults."""
        result = runner.invoke({app_object}, ["config", "reset"])
        assert result.exit_code == 0
        assert "reset" in result.stdout.lower()


# ============================================================
# Edge Cases
# ============================================================

class TestEdgeCases:
    """Test edge cases and unusual inputs."""

    def test_path_with_spaces(self, tmp_path: Path):
        """Handles paths with spaces correctly."""
        space_dir = tmp_path / "dir with spaces"
        space_dir.mkdir()
        (space_dir / "file.txt").write_text("content")
        
        result = runner.invoke({app_object}, ["organize", str(space_dir)])
        assert result.exit_code == 0

    def test_unicode_filenames(self, tmp_path: Path):
        """Handles unicode in filenames."""
        (tmp_path / "文件.txt").write_text("unicode content")
        (tmp_path / "émoji_🎉.txt").write_text("emoji content")
        
        result = runner.invoke({app_object}, ["preview", str(tmp_path)])
        assert result.exit_code == 0

    def test_symlink_handling(self, tmp_path: Path):
        """Handles symbolic links appropriately."""
        real_file = tmp_path / "real.txt"
        real_file.write_text("content")
        link_file = tmp_path / "link.txt"
        link_file.symlink_to(real_file)
        
        result = runner.invoke({app_object}, ["preview", str(tmp_path)])
        assert result.exit_code == 0

    def test_very_deep_directory(self, tmp_path: Path):
        """Handles deeply nested directories."""
        deep_path = tmp_path
        for i in range(20):
            deep_path = deep_path / f"level{i}"
            deep_path.mkdir()
        (deep_path / "deep_file.txt").write_text("deep content")
        
        result = runner.invoke({app_object}, ["organize", str(tmp_path)])
        assert result.exit_code == 0


# ============================================================
# Error Handling
# ============================================================

class TestErrorHandling:
    """Test error handling and edge cases."""

    def test_permission_error(self, tmp_path: Path, monkeypatch):
        """Shows clear error for permission denied."""
        def raise_permission(*args, **kwargs):
            raise PermissionError("Permission denied")
        
        monkeypatch.setattr("os.listdir", raise_permission)
        
        result = runner.invoke({app_object}, ["organize", str(tmp_path)])
        assert result.exit_code != 0
        assert "permission" in result.stdout.lower()

    def test_keyboard_interrupt(self, temp_dir: Path, monkeypatch):
        """Handles Ctrl+C gracefully."""
        # This test ensures the CLI doesn't crash on interrupt
        # Implementation depends on how interrupts are handled
        pass

    def test_invalid_option_value(self, temp_dir: Path):
        """Invalid option values show helpful errors."""
        result = runner.invoke({app_object}, [
            "preview", str(temp_dir), "--format", "invalid"
        ])
        assert result.exit_code != 0
        assert "invalid" in result.stdout.lower() or "choice" in result.stdout.lower()


# ============================================================
# Exit Codes
# ============================================================

class TestExitCodes:
    """Verify correct exit codes for various scenarios."""

    def test_success_exit_code(self, temp_dir: Path):
        """Successful operations return exit code 0."""
        result = runner.invoke({app_object}, ["preview", str(temp_dir)])
        assert result.exit_code == 0

    def test_error_exit_code(self):
        """Errors return non-zero exit code."""
        result = runner.invoke({app_object}, ["organize", "/nonexistent"])
        assert result.exit_code != 0

    def test_validation_error_exit_code(self):
        """Validation errors return exit code 2."""
        result = runner.invoke({app_object}, ["organize"])  # Missing required arg
        assert result.exit_code == 2  # Typer/Click convention for usage errors
```

## Framework-Specific Notes

### Typer Testing
```python
from typer.testing import CliRunner
runner = CliRunner()

# Invoke the app directly
result = runner.invoke(app, ["command", "--option", "value"])
```

### Click Testing
```python
from click.testing import CliRunner
runner = CliRunner()

# Invoke the CLI group
result = runner.invoke(cli, ["command", "--option", "value"])

# With environment variables
result = runner.invoke(cli, ["command"], env={"API_KEY": "test"})

# With stdin input
result = runner.invoke(cli, ["command"], input="user input\n")
```

## Test Coverage Checklist

- [ ] All commands have basic functionality tests
- [ ] All required arguments tested for presence
- [ ] All options tested for default and explicit values
- [ ] Invalid inputs produce appropriate errors
- [ ] Exit codes are correct for success/failure
- [ ] Help text displays correctly
- [ ] JSON output is valid JSON
- [ ] Edge cases covered (empty, unicode, spaces)
- [ ] Mocks used for external dependencies
- [ ] Fixtures used for file system setup
