# docs/go-conventions.md — manga-tui Go-Specific Conventions

This document defines Go-specific coding conventions for the manga-tui project, based on the principles outlined in AGENTS.md and adapted for Go development.

## Code Style and Formatting

### 1. File Organization
- **Package Structure**: Organize code into logical packages (e.g., `main`, `ui`, `models`, `utils`)
- **File Naming**: Use snake_case for file names (e.g., `reading_progress.go`, `chapter_manager.go`)
- **Directory Structure**: 
  ```
  /cmd/
    /tui/
      main.go
  /internal/
    /ui/
      /components/
      /layout/
      /theme/
    /models/
      /manga/
      /chapter/
    /utils/
      /file/
      /storage/
  /pkg/
    /reader/
    /parser/
  /tests/
  ```

### 2. Naming Conventions
- **Public Identifiers**: PascalCase (e.g., `MangaReader`, `ChapterManager`)
- **Private Identifiers**: camelCase (e.g., `getReadingProgress`, `saveToFile`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_PAGES`, `DEFAULT_THEME`)
- **Variables**: camelCase (e.g., `chapterCount`, `readingProgress`)
- **Types**: PascalCase (e.g., `MangaFile`, `UIConfig`)

### 3. Code Structure

#### Function Definitions
```go
// Good example - clear purpose and documentation
func LoadMangaFromFile(filePath string) (*MangaLibrary, error) {
    // Implementation
}

// Bad example - unclear purpose
func Xyz(file string) (interface{}, error) {
    // Unclear what this does
}
```

#### Method Definitions
```go
// Good example - receiver method with documentation
type MangaReader struct {
    // Fields
}

// ReadChapter reads a specific chapter and returns its pages
func (mr *MangaReader) ReadChapter(chapterID string) ([]*Page, error) {
    // Implementation
}
```

### 4. Error Handling

#### Go-Specific Patterns
```go
// Good example - explicit error handling
func processFile(path string) error {
    file, err := os.Open(path)
    if err != nil {
        return fmt.Errorf("failed to open file %s: %w", path, err)
    }
    defer file.Close()
    
    // Process file content
    return nil
}
```

#### Custom Error Types
```go
// Define custom error types for better error differentiation
type ReadingError struct {
    ChapterID string
    Message   string
}

func (re *ReadingError) Error() string {
    return fmt.Sprintf("reading error in chapter %s: %s", re.ChapterID, re.Message)
}
```

### 5. Interface Design

```go
// Good example - clear interface definition
type Reader interface {
    // Open initializes the reader with a source
    Open() error
    
    // ReadPage returns the next page content
    ReadPage() (*Page, error)
    
    // Close releases any resources
    Close() error
}
```

### 6. Concurrency Patterns

```go
// Good example - proper goroutine management
func StartBackgroundProcessing(channels <-chan FileInfo) {
    var wg sync.WaitGroup
    
    for file := range channels {
        wg.Add(1)
        go func(f FileInfo) {
            defer wg.Done()
            // Process file
        }(file)
    }
    
    wg.Wait()
}
```

### 7. Testing Conventions

#### Test Structure
```go
// Good example - comprehensive testing
type MangaReaderTest struct {
    // Setup
}

func (mrt *MangaReaderTest) TestReadChapter() {
    // Test implementation
}

func TestIntegration(t *testing.T) {
    // Integration tests
}
```

#### Test Naming
- **Test Functions**: Use descriptive names with `Test` prefix (e.g., `TestLoadMangaFromValidFile`)
- **Benchmarks**: Use `Benchmark` prefix (e.g., `BenchmarkPageProcessing`)
- **Examples**: Use `Example` prefix (e.g., `ExampleReaderInitialization`)

### 8. Documentation

#### GoDoc Standards
```go
// Package documentation
package manga

// Package manga provides functionality for reading and managing manga files.
// It includes features for browsing chapters, tracking reading progress, and
// customizing the reading experience.

// MangaReader represents a reader for manga files
type MangaReader struct {
    // File path to the manga source
    sourcePath string
    
    // Current reading position
    currentPage int
    
    // Chapter information
    chapters []*Chapter
}
```

#### Function Documentation
```go
// LoadManga loads manga data from the specified file path.
// It returns a MangaLibrary containing all chapters or an error if loading fails.
// The function validates the file format and extracts metadata.
func LoadManga(filePath string) (*MangaLibrary, error) {
    // Implementation
}
```

## Dependency Management

### 1. Module Organization
```go
module github.com/yourusername/manga-tui

go 1.21

require (
    golang.org/x/term v0.15.0
    github.com/charmbracelet/bubbletea v0.25.0
)
```

### 2. Import Aliases
```go
// Use import aliases for longer paths to improve readability
import (
    term "golang.org/x/term"
    tea "github.com/charmbracelet/bubbletea"
)
```

## Performance Considerations

### 1. Memory Management
```go
// Good example - efficient memory usage
func processLargeFile(path string) error {
    // Use streaming for large files instead of loading everything into memory
    file, err := os.Open(path)
    if err != nil {
        return err
    }
    defer file.Close()
    
    // Process in chunks
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        // Process each line
    }
    
    return scanner.Err()
}
```

### 2. Caching Strategy
```go
// Implement caching with appropriate invalidation
var cache = make(map[string]*Data)

func GetCachedData(key string) (*Data, error) {
    if data, exists := cache[key]; exists {
        return data, nil
    }
    
    // Load and cache data
    data, err := loadData(key)
    if err != nil {
        return nil, err
    }
    
    cache[key] = data
    return data, nil
}
```

## Security Best Practices

### 1. Input Validation
```go
// Validate file paths to prevent directory traversal
func validateFilePath(path string) error {
    absPath, err := filepath.Abs(path)
    if err != nil {
        return err
    }
    
    // Check if path is within allowed directories
    if !strings.HasPrefix(absPath, allowedDir) {
        return fmt.Errorf("access denied: path outside allowed directory")
    }
    
    return nil
}
```

### 2. Error Messages
```go
// Provide detailed but safe error messages
func readFile(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        if errors.Is(err, os.ErrNotExist) {
            return nil, fmt.Errorf("file not found: %s", filepath.Base(path))
        }
        return nil, fmt.Errorf("failed to read file: %w", err)
    }
    return data, nil
}
```

## Architecture Alignment with AGENTS.md

This Go-specific conventions document supports the general principles outlined in AGENTS.md:

1. **Accuracy over Speed**: Emphasizes correct Go idioms and patterns
2. **No Hallucination**: Provides concrete examples and avoids fictional Go-specific features
3. **Transparency**: Clearly documents Go-specific patterns and their rationale
4. **No Unnecessary Complexity**: Focuses on idiomatic Go without over-engineering
5. **Context Awareness**: Tailors general principles to Go development practices

## Testing and Benchmarking

Following AGENTS.md testing principles:

1. **Unit Tests**: Test individual functions and methods
2. **Integration Tests**: Verify component interactions
3. **Benchmarks**: Measure performance of critical paths
4. **Error Scenario Tests**: Test edge cases and error conditions

## Refactoring Guidelines

When refactoring Go code:

1. **Preserve Interface Contracts**: Don't break public APIs
2. **Use Go's Dependency Injection**: Inject dependencies rather than hard-coding
3. **Follow Go Naming Conventions**: Ensure all identifiers follow the conventions
4. **Update Documentation**: Keep GoDoc comments up to date