# Singleton Locker - Usage Guide

This package implements a file-based singleton pattern that ensures only one instance of your application can run at a time.

## Features

- **File-based locking**: Uses file locks to prevent multiple instances
- **PID tracking**: Writes the current process ID to the lock file
- **Pretty error messages**: Shows informative messages when another instance is running
- **Stale lock detection**: Detects if a lock file is from a dead process
- **Graceful shutdown**: Properly releases locks on exit

## Quick Start

### Basic Usage

```go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    // Create a lock for your application
    lockFile := getLockFilePath("my-unique-app-name")
    lock := NewInstanceLock(lockFile)

    // Try to acquire the lock or exit if another instance is running
    lock.LockOrExit()

    // Ensure the lock is released when the program exits
    defer lock.Unlock()

    fmt.Println("✓ Application started successfully!")

    // Your application logic here
    runYourApplication()
}
```

### Advanced Usage with Error Handling

```go
func main() {
    lockFile := getLockFilePath("my-app")
    lock := NewInstanceLock(lockFile)

    // Manual lock with error handling
    if err := lock.Lock(); err != nil {
        if err == os.ErrExist {
            fmt.Println("Another instance is already running")
            pid := lock.readPIDFromLockFile()
            fmt.Printf("Running instance PID: %d\n", pid)
        } else {
            fmt.Printf("Error acquiring lock: %v\n", err)
        }
        os.Exit(1)
    }
    defer lock.Unlock()

    // Your application code...
}
```

## Example Output

### When another instance is already running:

```
╔════════════════════════════════════════════════════════════╗
║                                                            ║
║          Another instance is already running!              ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝

  Process ID: 12345 (running)
  Lock file: /tmp/go-lock/my-app.lock

  To stop the running instance, use: kill 12345

```

### When lock file is stale (process not running):

```
╔════════════════════════════════════════════════════════════╗
║                                                            ║
║          Another instance is already running!              ║
║                                                            ║
╚════════════════════════════════════════════════════════════╝

  Process ID: 12345 (stale - process not running)
  Lock file: /tmp/go-lock/my-app.lock

  The lock file may be stale. Remove it with: rm /tmp/go-lock/my-app.lock

```

## API Reference

### `NewInstanceLock(lockFilePath string) *InstanceLock`

Creates a new instance lock with the specified lock file path.

**Parameters:**

- `lockFilePath`: Absolute path to the lock file

**Returns:**

- `*InstanceLock`: A new instance lock object

### `(il *InstanceLock) LockOrExit()`

Tries to acquire the lock. If it fails, prints a pretty error message and exits with code 1.

This is the recommended method for most applications as it handles all error cases automatically.

### `(il *InstanceLock) Lock() error`

Tries to acquire the lock and write the current PID to the lock file.

**Returns:**

- `error`: nil on success, os.ErrExist if locked, or other error on failure

### `(il *InstanceLock) Unlock() error`

Releases the lock.

**Returns:**

- `error`: nil on success, error on failure

### `getLockFilePath(appName string) string`

Returns the path to the lock file for the given application name.

**Parameters:**

- `appName`: Name of your application (will be used as the lock file name)

**Returns:**

- `string`: Absolute path to the lock file (in /tmp/go-lock/)

## How It Works

1. **Lock Acquisition**: When your application starts, it tries to acquire an exclusive file lock
2. **PID Writing**: If successful, it writes the current process ID to the lock file
3. **Lock Checking**: If the lock is held by another process, it reads the PID and checks if that process is still running
4. **Pretty Messages**: Displays informative messages to help users understand what's happening
5. **Cleanup**: When your application exits, the lock is automatically released via `defer`

## Testing

To test the singleton behavior:

1. Build the application:

   ```bash
   go build -o singleton
   ```

2. Run the first instance in the background:

   ```bash
   ./singleton &
   ```

3. Try to run a second instance:

   ```bash
   ./singleton
   ```

   You should see the "Another instance is already running" message.

4. Stop the first instance:
   ```bash
   kill <PID>
   ```

## Dependencies

- `github.com/gofrs/flock` - Cross-platform file locking library

Install with:

```bash
go get github.com/gofrs/flock
```

## Platform Support

This implementation works on:

- Linux
- macOS
- Windows

The `flock` library handles platform-specific differences automatically.
