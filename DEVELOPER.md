# Cline Developer Guide

This guide provides comprehensive documentation for developing features and contributing to the Cline codebase. Cline is a sophisticated VSCode extension that provides AI-powered coding assistance through a React-based webview frontend and a TypeScript backend.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Development Environment Setup](#development-environment-setup)
- [Feature Development Workflow](#feature-development-workflow)
- [Communication Layer (Protobuf/gRPC)](#communication-layer-protobufgrpc)
- [Backend Development](#backend-development)
- [Frontend Development](#frontend-development)
- [State Management](#state-management)
- [Testing](#testing)
- [Best Practices](#best-practices)
- [Complete Example: Adding a Favorites System](#complete-example-adding-a-favorites-system)
- [Reference Information](#reference-information)

## Architecture Overview

Cline follows a modular architecture with clear separation between frontend and backend components:

### Core Extension (Backend)

- **`src/extension.ts`** - Main entry point and VSCode integration
- **`src/core/controller/`** - State management and task coordination
- **`src/core/task/`** - Task execution and API request handling
- **`src/core/webview/`** - Webview lifecycle management
- **`src/services/`** - Various services (MCP, telemetry, auth, etc.)
- **`src/integrations/`** - Tool implementations and external integrations

### Webview UI (Frontend)

- **`webview-ui/src/App.tsx`** - Main React application
- **`webview-ui/src/components/`** - React components for different views
- **`webview-ui/src/context/`** - State management contexts
- **`webview-ui/src/services/`** - gRPC client services

### Communication

- Uses Protobuf/gRPC for type-safe communication between frontend and backend
- Message passing via VSCode's webview API
- Protocol buffer definitions in the `proto/` directory

## Development Environment Setup

### Prerequisites

- Node.js and npm
- VSCode with extension development setup
- Git for version control

### Getting Started

```bash
# Clone the repository
git clone https://github.com/cline/cline.git
cd cline

# Install all dependencies
npm run install:all

# Start development mode
npm run watch          # Compiles backend in watch mode
npm run dev:webview    # Starts React dev server

# Build for production
npm run package
```

### Key Development Scripts

- `npm run protos` - Compile protobuf definitions
- `npm run check-types` - TypeScript type checking
- `npm run test` - Run tests
- `npm run lint` - Code linting and formatting
- `npm run compile` - Full compilation with type checking
- `npm run build:webview` - Build React frontend

### Development Workflow

1. **Make changes** to TypeScript/React code
2. **Run compilation** with `npm run compile`
3. **Test changes** in VSCode extension host
4. **Debug** using VSCode's built-in debugger
5. **Format code** with `npm run format`

## Feature Development Workflow

### 1. Identify Feature Type

**Backend-only features:** New tools, API integrations, file operations
**Frontend-only features:** UI improvements, new views, styling changes
**Full-stack features:** New functionality requiring both backend logic and UI changes

### 2. Plan Your Implementation

- **Define requirements** and user stories
- **Identify affected components** (backend, frontend, or both)
- **Plan state management** needs
- **Design communication interfaces** if needed
- **Consider testing strategy**

### 3. Implement Step by Step

1. **Backend development** (if needed)
2. **Frontend development** (if needed)
3. **Communication layer** (if full-stack)
4. **Testing and validation**
5. **Documentation updates**

## Communication Layer (Protobuf/gRPC)

When adding features that require frontend-backend communication, use the protobuf system:

### Step 1: Define Protobuf Schema

Create or update a `.proto` file in the `proto/` directory:

```proto
// proto/feature.proto
syntax = "proto3";

import "common.proto";

service FeatureService {
  rpc newFeatureMethod(FeatureRequest) returns (FeatureResponse);
  rpc streamFeatureData(Empty) returns (stream FeatureData);
}

message FeatureRequest {
  string input = 1;
  repeated string options = 2;
}

message FeatureResponse {
  string result = 1;
  int32 status_code = 2;
}

message FeatureData {
  string data = 1;
  int64 timestamp = 2;
}
```

### Step 2: Compile Protobuf Definitions

```bash
npm run protos
```

This generates:
- TypeScript types in `src/generated/`
- Service clients in `webview-ui/src/services/grpc-client/`
- Handler interfaces in `src/core/controller/`

### Step 3: Implement Backend Handler

Create handler in `src/core/controller/[feature]/`:

```typescript
// src/core/controller/feature/featureHandler.ts
import { Controller } from "../index"
import { FeatureRequest, FeatureResponse } from "../../../generated/proto/feature"

export async function handleNewFeature(
  controller: Controller,
  request: FeatureRequest
): Promise<FeatureResponse> {
  try {
    // Implementation logic
    const result = await processFeatureRequest(request.input, request.options)

    return FeatureResponse.create({
      result,
      statusCode: 200
    })
  } catch (error) {
    return FeatureResponse.create({
      result: error.message,
      statusCode: 500
    })
  }
}
```

### Step 4: Register Handler

Add to the appropriate service router or controller:

```typescript
// src/core/controller/feature/index.ts
import { handleNewFeature } from "./featureHandler"

export const featureServiceHandlers = {
  newFeatureMethod: handleNewFeature
}
```

### Step 5: Use in Frontend

```typescript
// webview-ui/src/components/FeatureComponent.tsx
import { FeatureServiceClient } from "../../services/grpc-client"
import { FeatureRequest } from "../../../../shared/proto/feature"

const FeatureComponent = () => {
  const handleFeatureAction = async () => {
    try {
      const response = await FeatureServiceClient.newFeatureMethod(
        FeatureRequest.create({
          input: "user input",
          options: ["option1", "option2"]
        })
      )

      console.log("Feature result:", response.result)
    } catch (error) {
      console.error("Feature error:", error)
    }
  }

  return (
    <button onClick={handleFeatureAction}>
      Execute Feature
    </button>
  )
}
```

## Backend Development

### Adding New Tools

1. **Create tool implementation** in `src/integrations/`:

```typescript
// src/integrations/tools/customTool.ts
import { ClineDefaultTool } from "../../../shared/tools"
import { ToolResponse } from "../../../core/task"

export class CustomTool implements ClineDefaultTool {
  readonly name = "custom_tool"
  readonly description = "Description of what this tool does"

  async execute(args: any, context: any): Promise<ToolResponse> {
    // Tool implementation
    return formatResponse.toolResult("Tool execution result")
  }
}
```

2. **Register tool** in the tool executor:

```typescript
// src/core/task/ToolExecutor.ts
import { CustomTool } from "../../integrations/tools/customTool"

export class ToolExecutor {
  private tools = new Map<string, ClineDefaultTool>()

  constructor() {
    this.registerTool(new CustomTool())
  }

  private registerTool(tool: ClineDefaultTool) {
    this.tools.set(tool.name, tool)
  }
}
```

### Adding New Services

1. **Create service class** in `src/services/`:

```typescript
// src/services/custom/CustomService.ts
export class CustomService {
  private static instance: CustomService

  static getInstance(): CustomService {
    if (!CustomService.instance) {
      CustomService.instance = new CustomService()
    }
    return CustomService.instance
  }

  async performOperation(data: any): Promise<any> {
    // Service implementation
  }
}
```

2. **Initialize in Controller**:

```typescript
// src/core/controller/index.ts
import { CustomService } from "../../services/custom/CustomService"

export class Controller {
  customService: CustomService

  constructor() {
    this.customService = CustomService.getInstance()
  }
}
```

## Frontend Development

### Adding New Views

1. **Create component** in `webview-ui/src/components/`:

```typescript
// webview-ui/src/components/CustomView.tsx
import React from "react"
import { useExtensionState } from "../context/ExtensionStateContext"

const CustomView: React.FC = () => {
  const { state, dispatch } = useExtensionState()

  return (
    <div className="custom-view">
      <h1>Custom View</h1>
      {/* Component content */}
    </div>
  )
}

export default CustomView
```

2. **Add routing** in `App.tsx`:

```typescript
// webview-ui/src/App.tsx
import CustomView from "./components/CustomView"

const AppContent = () => {
  const { showCustomView, hideCustomView } = useExtensionState()

  return (
    <div>
      {showCustomView && <CustomView onDone={hideCustomView} />}
      {/* Other views */}
    </div>
  )
}
```

### State Management

Use the `ExtensionStateContext` for accessing backend state:

```typescript
// webview-ui/src/components/CustomComponent.tsx
import { useExtensionState } from "../context/ExtensionStateContext"

const CustomComponent = () => {
  const { state } = useExtensionState()

  // Access backend state
  const customData = state.customData
  const isLoading = state.isLoading

  return (
    <div>
      {isLoading ? "Loading..." : customData}
    </div>
  )
}
```

## State Management

### Backend State (Controller)

Use the `StateManager` class for persistence:

```typescript
// src/core/controller/index.ts
export class Controller {
  async setCustomSetting(value: any) {
    this.stateManager.setGlobalState("customSetting", value)
    await this.postStateToWebview()
  }

  async getCustomSetting() {
    return this.stateManager.getGlobalStateKey("customSetting")
  }
}
```

### State Types

- **Global State:** Extension-wide settings (API keys, preferences)
- **Workspace State:** Project-specific data
- **Secrets:** Sensitive information (API keys, tokens)

### State Synchronization

State changes automatically sync between backend and frontend through the `postStateToWebview()` method.

## Testing

### Unit Tests

```typescript
// tests/unit/CustomService.test.ts
import { expect } from "chai"
import { CustomService } from "../../src/services/custom/CustomService"

describe("CustomService", () => {
  let service: CustomService

  beforeEach(() => {
    service = CustomService.getInstance()
  })

  it("should perform operation correctly", async () => {
    const result = await service.performOperation({ data: "test" })
    expect(result).to.equal("expected result")
  })
})
```

### Integration Tests

```typescript
// tests/integration/Controller.test.ts
import { Controller } from "../../src/core/controller"

describe("Controller Integration", () => {
  let controller: Controller

  beforeEach(() => {
    controller = new Controller(/* dependencies */)
  })

  it("should handle state changes", async () => {
    await controller.setCustomSetting("test value")
    const value = await controller.getCustomSetting()
    expect(value).to.equal("test value")
  })
})
```

### Running Tests

```bash
# Run all tests
npm run test

# Run unit tests only
npm run test:unit

# Run integration tests
npm run test:integration

# Run with coverage
npm run test:coverage
```

## Best Practices

### Code Organization

- Follow the established directory structure
- Use TypeScript for type safety
- Implement proper error handling
- Add telemetry for feature usage tracking
- Write comprehensive documentation

### Performance

- Minimize bundle size by lazy loading components
- Use efficient state management patterns
- Implement proper caching strategies
- Optimize protobuf message sizes

### Security

- Never store sensitive data in global state
- Use VSCode's secrets storage for API keys
- Validate all user inputs
- Implement proper authentication checks

### Error Handling

```typescript
// Backend error handling
try {
  const result = await riskyOperation()
  return result
} catch (error) {
  console.error("Operation failed:", error)
  throw new Error(`Operation failed: ${error.message}`)
}
```

```typescript
// Frontend error handling
const [error, setError] = useState<string | null>(null)

const handleOperation = async () => {
  try {
    setError(null)
    await performOperation()
  } catch (err) {
    setError(err.message)
  }
}
```

## Complete Example: Adding a Favorites System

Let's implement a complete feature that allows users to favorite tasks:

### 1. Backend Implementation

**Add state management:**

```typescript
// src/core/controller/index.ts
export class Controller {
  async setTaskFavorite(taskId: string, favorite: boolean) {
    const taskHistory = this.stateManager.getGlobalStateKey("taskHistory")
    const task = taskHistory.find(t => t.id === taskId)

    if (task) {
      task.isFavorited = favorite
      this.stateManager.setGlobalState("taskHistory", taskHistory)
      await this.postStateToWebview()
    }
  }

  async getTaskFavorites() {
    const taskHistory = this.stateManager.getGlobalStateKey("taskHistory")
    return taskHistory.filter(task => task.isFavorited)
  }
}
```

**Add protobuf definition:**

```proto
// proto/task.proto
service TaskService {
  rpc setTaskFavorite(SetFavoriteRequest) returns (Empty);
  rpc getTaskFavorites(Empty) returns (TaskList);
}

message SetFavoriteRequest {
  string task_id = 1;
  bool favorite = 2;
}

message TaskList {
  repeated HistoryItem tasks = 1;
}
```

**Implement handlers:**

```typescript
// src/core/controller/task/favoritesHandler.ts
import { Controller } from "../index"
import { SetFavoriteRequest, Empty } from "../../../generated/proto/task"

export async function handleSetTaskFavorite(
  controller: Controller,
  request: SetFavoriteRequest
): Promise<Empty> {
  await controller.setTaskFavorite(request.taskId, request.favorite)
  return Empty.create({})
}
```

### 2. Frontend Implementation

**Add UI component:**

```typescript
// webview-ui/src/components/FavoritesView.tsx
import React from "react"
import { useExtensionState } from "../context/ExtensionStateContext"
import { TaskServiceClient } from "../services/grpc-client"
import { SetFavoriteRequest } from "../../../shared/proto/task"

const FavoritesView: React.FC = () => {
  const { state } = useExtensionState()
  const favorites = state.taskHistory.filter(task => task.isFavorited)

  const toggleFavorite = async (taskId: string, currentlyFavorited: boolean) => {
    try {
      await TaskServiceClient.setTaskFavorite(
        SetFavoriteRequest.create({
          taskId,
          favorite: !currentlyFavorited
        })
      )
    } catch (error) {
      console.error("Failed to toggle favorite:", error)
    }
  }

  return (
    <div className="favorites-view">
      <h2>Favorite Tasks</h2>
      {favorites.map(task => (
        <div key={task.id} className="task-item">
          <span>{task.task}</span>
          <button
            onClick={() => toggleFavorite(task.id, task.isFavorited)}
            className="favorite-button"
          >
            {task.isFavorited ? "★" : "☆"}
          </button>
        </div>
      ))}
    </div>
  )
}

export default FavoritesView
```

### 3. Integration

**Add to main app:**

```typescript
// webview-ui/src/App.tsx
import FavoritesView from "./components/FavoritesView"

const AppContent = () => {
  const { showFavorites } = useExtensionState()

  return (
    <div>
      {showFavorites && <FavoritesView />}
      {/* Other views */}
    </div>
  )
}
```

## Reference Information

### Key Files and Directories

- `src/extension.ts` - Extension entry point
- `src/core/controller/index.ts` - Main controller class
- `src/core/task/index.ts` - Task execution engine
- `webview-ui/src/App.tsx` - React app entry point
- `webview-ui/src/context/ExtensionStateContext.tsx` - Frontend state management
- `proto/` - Protocol buffer definitions
- `src/generated/` - Generated protobuf code

### Common Patterns

**Service Registration:**
```typescript
// Use singleton pattern for services
export class MyService {
  private static instance: MyService

  static getInstance(): MyService {
    if (!MyService.instance) {
      MyService.instance = new MyService()
    }
    return MyService.instance
  }
}
```

**Error Handling:**
```typescript
// Always wrap async operations
try {
  const result = await operation()
  return result
} catch (error) {
  console.error("Operation failed:", error)
  throw error // Re-throw or handle appropriately
}
```

**State Updates:**
```typescript
// Always call postStateToWebview after state changes
await this.stateManager.setGlobalState("key", value)
await this.postStateToWebview()
```

### Development Tips

1. **Use TypeScript** for all new code
2. **Follow existing patterns** in the codebase
3. **Test thoroughly** before submitting PRs
4. **Document your changes** with comments
5. **Use the protobuf system** for frontend-backend communication
6. **Handle errors gracefully** at all levels
7. **Keep components small** and focused on single responsibilities
8. **Use proper state management** patterns

### Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes following the patterns above
4. Add tests for new functionality
5. Update documentation as needed
6. Submit a pull request

For more information, see the main [README.md](README.md) and [CONTRIBUTING.md](CONTRIBUTING.md) files.
