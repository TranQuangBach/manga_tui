# docs/architecture.md — manga-tui Architecture Documentation

This document defines the directory structure, data flow, and component interfaces for the manga-tui project, based on the architecture principles outlined in AGENTS.md.

## System Architecture Overview

manga-tui follows a layered architecture pattern that separates concerns while maintaining clear data flow and component interfaces. The architecture is designed to support the accuracy, modularity, and maintainability principles outlined in AGENTS.md.

## Directory Structure

```
manga-tui/
├── cmd/
│   └── tui/
│       └── main.go                    # Entry point and application initialization
├── internal/
│   ├── models/                         # Domain models and business logic
│   │   ├── manga.go                   # Manga domain model
│   │   ├── chapter.go                 # Chapter domain model
│   │   └── reading_progress.go        # Reading progress tracking
│   ├── ui/                             # User interface components
│   │   ├── components/                # UI component library
│   │   │   ├── reader.go              # Main reader component
│   │   │   ├── chapter_list.go        # Chapter navigation component
│   │   │   └── status_bar.go         # Application status component
│   │   ├── layout/                    # Layout management
│   │   │   ├── terminal_manager.go    # Terminal interaction
│   │   │   └── layout_manager.go      # UI layout management
│   │   └── theme/                     # Theme and styling
│   │       └── theme_manager.go       # Theme management
│   ├── storage/                        # Data persistence
│   │   ├── file_storage.go            # File-based storage implementation
│   │   └── backup_manager.go          # Backup and recovery
│   └── utils/                          # Utility functions
│       ├── file_utils.go              # File operation utilities
│       ├── string_utils.go            # String manipulation utilities
│       └── validation_utils.go        # Input validation
├── pkg/                                # Package plugins and extensions
│   ├── reader/                         # Reader implementations
│   │   ├── standard_reader.go         # Standard manga reader
│   │   └── advanced_reader.go         # Advanced features reader
│   └── parser/                         # File format parsers
│       ├── cbr_parser.go              # Comic Book Reader format
│       ├── pdf_parser.go              # PDF format
│       └── cbz_parser.go               # Comic Book Zip format
└── tests/                              # Test suite
    ├── integration/                   # Integration tests
    ├── unit/                          # Unit tests
    └── e2e/                           # End-to-end tests
```

## Component Interfaces and Data Flow

### 1. Core Model Layer

#### Manga Model (`models/manga.go`)
**Purpose**: Represents manga series and metadata

```go
// Interfaces
type MangaRepository interface {
    Save(manga *Manga) error
    FindByID(id string) (*Manga, error)
    FindAll() ([]*Manga, error)
    Delete(id string) error
}

type MangaService interface {
    AddManga(source string) (*Manga, error)
    GetManga(id string) (*Manga, error)
    UpdateReadingProgress(mangaID string, chapterID string, page int) error
}
```

#### Chapter Model (`models/chapter.go`)
**Purpose**: Represents individual manga chapters

```go
// Data Flow: Chapter → Page → Reader
// Operations: Navigate chapters → Load pages → Display pages

type ChapterRepository interface {
    Save(chapter *Chapter) error
    FindByMangaID(mangaID string) ([]*Chapter, error)
    FindByID(chapterID string) (*Chapter, error)
}
```

### 2. Service Layer

#### Business Logic Services

**MangaService** (`services/manga_service.go`)
- **Responsibility**: High-level manga management operations
- **Data Flow**: User input → Service → Repository → Database
- **Interface**: `MangaService`

**ChapterService** (`services/chapter_service.go`)
- **Responsibility**: Chapter navigation and reading progress
- **Data Flow**: UI input → Service → Repository → Cache
- **Interface**: `ChapterService`

### 3. Data Access Layer

#### Storage Components

**FileStorage** (`storage/file_storage.go`)
- **Responsibility**: Persistent storage of manga data
- **Data Flow**: Service → Storage → File System
- **Features**:
  - JSON serialization for metadata
  - Binary storage for manga files
  - Atomic writes for data integrity

**BackupManager** (`storage/backup_manager.go`)
- **Responsibility**: Backup and recovery operations
- **Data Flow**: Manual trigger → Storage → External storage
- **Features**:
  - Incremental backups
  - Version control
  - Corruption detection

### 4. Presentation Layer

#### UI Components

**Reader Component** (`ui/components/reader.go`)
- **Responsibility**: Core reading interface
- **Data Flow**: Model → UI → User Input → Model
- **Interfaces**:
  - `ReaderView` interface for rendering
  - `InputHandler` interface for keyboard events

**ChapterList Component** (`ui/components/chapter_list.go`)
- **Responsibility**: Chapter navigation
- **Data Flow**: Service → UI → User Selection → Service
- **Interfaces**:
  - `Navigable` interface for chapter jumping
  - `Filterable` interface for search/filter

**StatusBar Component** (`ui/components/status_bar.go`)
- **Responsibility**: Application status display
- **Data Flow**: Model → UI → Real-time updates
- **Interfaces**:
  - `Updatable` interface for status changes
  - `ThemeAware` interface for styling

### 5. Infrastructure Layer

#### Terminal Management

**TerminalManager** (`ui/layout/terminal_manager.go`)
- **Responsibility**: Terminal initialization and configuration
- **Data Flow**: Initialization → Configuration → UI Setup
- **Interfaces**:
  - `Terminable` for cleanup
  - `Resizable` for layout adjustments

**LayoutManager** (`ui/layout/layout_manager.go`)
- **Responsibility**: UI layout management
- **Data Flow**: Configuration → Component arrangement → Rendering
- **Interfaces**:
  - `Layoutable` for component positioning
  - `Responsive` for adaptive layouts

## Data Flow Architecture

### Reading Flow
```
1. User opens manga-tui
2. Application loads library from storage
3. Manga list is retrieved via MangaService
4. User selects manga → ChapterService loads chapters
5. User selects chapter → Reader loads pages
6. Pages are displayed via Reader component
7. User navigates pages → Reading progress updated
8. Changes saved via ChapterService → FileStorage
```

### Data Flow Patterns

#### 1. Request-Response Pattern
```go
// Example: Chapter loading flow
func LoadChapter(chapterID string) (*Chapter, error) {
    // Service layer receives request
    chapter, err := chapterService.GetChapter(chapterID)
    if err != nil {
        return nil, err
    }
    
    // Data access layer processes request
    pages, err := reader.LoadPages(chapter)
    if err != nil {
        return nil, err
    }
    
    return chapter, nil
}
```

#### 2. Event-Driven Pattern
```go
// Example: Reading progress updates
type ProgressTracker struct {
    channel chan ProgressEvent
}

func (pt *ProgressTracker) Start() {
    go func() {
        for event := range pt.channel {
            // Handle progress updates
            pt.updateProgress(event)
            pt.saveProgress(event)
        }
    }()
}
```

## Component Communication

### 1. Direct Method Calls
```go
// Between service layer components
func (ms *MangaService) UpdateProgress(mangaID string, chapterID string, page int) error {
    manga, err := ms.repo.FindByID(mangaID)
    if err != nil {
        return err
    }
    
    // Update progress
    if err := manga.UpdateProgress(chapterID, page); err != nil {
        return err
    }
    
    // Persist changes
    return ms.repo.Save(manga)
}
```

### 2. Event Broadcasting
```go
// For UI component updates
type EventBus struct {
    subscribers map[chan interface{}]bool
}

func (eb *EventBus) Publish(event interface{}) {
    for ch := range eb.subscribers {
        ch <- event
    }
}
```

## Technology Stack and Integration Points

### 1. UI Framework
**Bubble Tea** (`cmd/tui/main.go`)
- **Role**: TUI framework
- **Integration**: All UI components use Bubble Tea model pattern
- **Data Flow**: Tea messages → UI components → Services

### 2. File System Integration
**OS File Operations**
- **Role**: Data persistence
- **Integration**: Storage layer uses standard Go file operations
- **Data Flow**: Go file APIs → Storage layer → Business logic

### 3. Error Handling Pipeline
```go
// Consistent error handling across layers
func ProcessUserAction(action UserAction) error {
    // 1. Validate input
    if err := validateAction(action); err != nil {
        return NewValidationError(err.Error())
    }
    
    // 2. Business logic
    if err := executeAction(action); err != nil {
        return NewBusinessError(err.Error())
    }
    
    // 3. Persistence
    if err := saveChanges(); err != nil {
        return NewStorageError(err.Error())
    }
    
    return nil
}
```

## Scalability Considerations

### 1. Horizontal Scaling (Future)
```go
// Plugin architecture for future expansion
// Each plugin implements specific interfaces
// Example plugin interface
type MangaPlugin interface {
    Name() string
    Initialize(config PluginConfig) error
    GetFeatures() []string
}
```

### 2. Performance Optimization
```go
// Caching strategy for frequently accessed data
cache := make(map[string]*CachedData)

func GetCachedChapter(chapterID string) (*Chapter, error) {
    if data, exists := cache[chapterID]; exists {
        return data.chapter, nil
    }
    
    // Load and cache
    chapter, err := loadChapter(chapterID)
    if err != nil {
        return nil, err
    }
    
    cache[chapterID] = &CachedData{
        chapter: chapter,
        timestamp: time.Now(),
    }
    
    return chapter, nil
}
```

## Security Architecture

### 1. Defense in Depth
```go
// Multiple security layers
func SafeFileOperation(path string) error {
    // Layer 1: Input validation
    if err := validatePath(path); err != nil {
        return err
    }
    
    // Layer 2: Permission checks
    if err := checkPermissions(path); err != nil {
        return err
    }
    
    // Layer 3: Path traversal protection
    if err := preventPathTraversal(path); err != nil {
        return err
    }
    
    // Layer 4: File operation
    return performFileOperation(path)
}
```

### 2. Data Protection
```go
// Encryption for sensitive data
func encryptSensitiveData(data []byte) ([]byte, error) {
    // Use AES-256 encryption
    // Store encryption keys securely
    return nil, nil // Implementation details
}
```

## Testing Architecture

### 1. Layered Testing Strategy
```go
// Test organization by layer
type TestSuite struct {
    unitTests      *UnitTestSuite
    integrationTests *IntegrationTestSuite
    e2eTests       *E2ETestSuite
}

func (ts *TestSuite) RunAll() {
    ts.unitTests.Run()
    ts.integrationTests.Run()
    ts.e2eTests.Run()
}
```

### 2. Mock Dependencies
```go
// Interface-based testing
type MockMangaRepository struct {
    savedManga []*Manga
    findFunc   func(id string) (*Manga, error)
}

func (mr *MockMangaRepository) Save(manga *Manga) error {
    mr.savedManga = append(mr.savedManga, manga)
    return nil
}
```

## Maintenance and Extensibility

### 1. Interface Segregation
```go
// Single responsibility principle
// Separate interfaces for different concerns
type ReaderInterface interface {
    ReadPage() (*Page, error)
}

type ProgressInterface interface {
    SaveProgress(progress *Progress) error
    GetProgress(chapterID string) (*Progress, error)
}
```

### 2. Dependency Injection
```go
// Loose coupling for easier testing and maintenance
type Application struct {
    mangaService MangaService
    chapterService ChapterService
    readerComponent UIComponent
}

func NewApplication() *Application {
    repo := NewFileMangaRepository()
    mangaService := NewMangaService(repo)
    
    // Other services and components...
    
    return &Application{
        mangaService: mangaService,
        // ...
    }
}
```

## Architecture Principles Alignment

This architecture document supports the general principles outlined in AGENTS.md:

1. **Accuracy over Speed**: Clear interfaces and defined contracts ensure correctness
2. **No Hallucination**: Practical, implementable architecture based on real patterns
3. **Transparency**: Explicit data flow and component interfaces
4. **No Unnecessary Complexity**: Simple, focused components with single responsibilities
5. **Context Awareness**: Architecture designed for terminal-based manga reading
6. **No Premature Execution**: Well-defined interfaces for future development

## Future Evolution Considerations

### 1. Plugin Architecture
- External extensions without modifying core code
- Interface-based plugin loading
- Version management for plugins

### 2. Performance Optimization
- Lazy loading for large manga volumes
- Intelligent caching strategies
- Memory-efficient data structures

### 3. Enhanced Features
- Cloud synchronization
- Advanced reading modes
- Annotation and note support
- Multi-format parsing improvements