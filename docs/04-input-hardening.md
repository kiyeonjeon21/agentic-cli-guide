# 04. Input Hardening Against Hallucinations

## Core principle

> **Agents are not trusted operators.**
>
> Validate agent input the same way you validate user input to a web API.

Agents fail differently than humans — not just typos but **hallucinations**:

| Failure type | Example | Risk |
|--------------|---------|------|
| Path traversal | `../../.ssh/id_rsa` | Escape from intended filesystem roots |
| Query injection in resource IDs | `fileId?fields=name` | Unintended API behavior |
| Double encoding | `%252F` (encoding already-encoded data) | URL parsing bugs |
| Control characters | `\x00`, `\x1b` | Log corruption, terminal escape injection |

## Defenses

### 1. Path traversal

Ensure file paths cannot leave the intended working directory.

```python
import os

def validate_safe_output_dir(path: str, sandbox: str = None) -> str:
    """Restrict output paths within a sandbox directory."""
    if sandbox is None:
        sandbox = os.getcwd()

    # Normalize path (resolve symlinks, .., .)
    canonical = os.path.realpath(os.path.join(sandbox, path))
    sandbox_canonical = os.path.realpath(sandbox)

    if not canonical.startswith(sandbox_canonical + os.sep):
        raise ValueError(
            f"Path traversal detected: '{path}' resolves to '{canonical}' "
            f"which is outside sandbox '{sandbox_canonical}'"
        )

    return canonical
```

```go
func validateSafeOutputDir(path, sandbox string) (string, error) {
    if sandbox == "" {
        sandbox, _ = os.Getwd()
    }

    canonical, err := filepath.EvalSymlinks(filepath.Join(sandbox, path))
    if err != nil {
        return "", fmt.Errorf("invalid path: %w", err)
    }

    sandboxCanonical, _ := filepath.EvalSymlinks(sandbox)
    if !strings.HasPrefix(canonical, sandboxCanonical+string(os.PathSeparator)) {
        return "", fmt.Errorf("path traversal detected: %s escapes %s", path, sandboxCanonical)
    }

    return canonical, nil
}
```

### 2. Reject control characters

Prevent invisible characters from polluting logs or injecting escape sequences.

```python
import re

def reject_control_chars(value: str, field_name: str = "input") -> str:
    """Reject control chars below ASCII 0x20; tab/newline/CR may be allowed by policy."""
    # Allow tab (\t=0x09), newline (\n=0x0a), carriage return (\r=0x0d) if desired
    pattern = re.compile(r'[\x00-\x08\x0b\x0c\x0e-\x1f]')

    if pattern.search(value):
        raise ValueError(
            f"Control character detected in {field_name}. "
            f"Input contains non-printable characters."
        )

    return value
```

### 3. Resource ID injection

Block query parameters or fragments smuggled inside resource IDs.

```python
def validate_resource_name(resource_id: str, field_name: str = "id") -> str:
    """Ensure resource IDs do not contain URL metacharacters."""
    forbidden = {'?', '#', '%', '&', '='}

    found = set(resource_id) & forbidden
    if found:
        raise ValueError(
            f"Invalid characters {found} in {field_name}: '{resource_id}'. "
            f"Resource IDs must not contain URL meta-characters."
        )

    return resource_id
```

### 4. Prevent double encoding

```python
from urllib.parse import quote, unquote

def encode_path_segment(segment: str) -> str:
    """
    Apply percent-encoding at the HTTP layer.
    Detect and reject already-encoded input.
    """
    if segment != unquote(segment):
        raise ValueError(
            f"Input appears to be already percent-encoded: '{segment}'. "
            f"Provide raw (unencoded) values."
        )

    return quote(segment, safe='')
```

## Validation chain

Compose checks into a single pipeline:

```python
def validate_agent_input(
    value: str,
    field_name: str,
    is_path: bool = False,
    is_resource_id: bool = False,
) -> str:
    """Unified validation for agent-provided strings."""

    # 1. Control characters (all string inputs)
    reject_control_chars(value, field_name)

    # 2. Path validation (when value is a filesystem path)
    if is_path:
        value = validate_safe_output_dir(value)

    # 3. Resource ID validation (API identifiers)
    if is_resource_id:
        validate_resource_name(value, field_name)

    return value
```

## Fuzz testing

Exercise adversarial or malformed inputs agents might emit:

```python
FUZZ_INPUTS = [
    # Path traversal
    "../../etc/passwd",
    "../.ssh/id_rsa",
    "..\\..\\windows\\system32",

    # Query injection
    "fileId123?fields=*",
    "docId#section",
    "id&extra=param",

    # Double encoding
    "%2F%2F",
    "%252e%252e%252f",

    # Control characters
    "normal\x00text",
    "hello\x1b[31mred",
    "\r\ninjected-header: value",

    # Unicode tricks
    "file\u200bname",     # zero-width space
    "fi\u0131le",         # dotless i
    "ﬁle",                # fi ligature

    # Empty / boundary
    "",
    " ",
    "a" * 10000,
]
```

## Summary

| Threat | Validation | Applies to |
|--------|------------|------------|
| Path traversal | `validate_safe_output_dir` | File path arguments |
| Control characters | `reject_control_chars` | All string inputs |
| Resource ID injection | `validate_resource_name` | API resource identifiers |
| Double encoding | `encode_path_segment` | URL path segments |
