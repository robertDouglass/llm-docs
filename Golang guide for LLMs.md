# Model Context Protocol (MCP) Golang Server Implementation Guide

This document provides a comprehensive guide for implementing Model Context Protocol servers in Go, optimized for LLM understanding and code generation.

## Core Server Setup

```go
package main

import (
    "context"
    "fmt"
    "github.com/metoro-io/mcp-golang"
    "github.com/metoro-io/mcp-golang/transport/stdio"
)

func main() {
    // Initialize server with stdio transport
    server := mcp_golang.NewServer(stdio.NewStdioServerTransport())

    // Configure and start server
    if err := server.Serve(); err != nil {
        panic(err)
    }
    
    // Keep server running
    select {}
}
```

## Configuration Structure
```json
{
    "serverInfo": {
        "name": "mcp-server",
        "version": "1.0.0"
    },
    "logging": {
        "file": "/var/log/mcp/server.log",
        "level": "debug",
        "withStderr": false
    },
    "prompts": {
        "file": "/etc/mcp/prompts.yaml"
    },
    "tools": [
        {
            "name": "calculator",
            "description": "Math operations",
            "configuration": {
                "maxValue": 1000
            }
        }
    ]
}
```
## Resource Implementation
```go
// Static Resource
resource := mcp.NewResource(
    "docs://readme",
    "Documentation",
    mcp.WithResourceDescription("Project documentation"),
    mcp.WithMIMEType("text/markdown"),
    mcp.WithAnnotations([]mcp.Role{mcp.RoleAssistant}, 0.8),
)

server.AddResource(resource, func(ctx context.Context, request mcp.ReadResourceRequest) ([]interface{}, error) {
    content := "# Documentation content"
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI: "docs://readme",
                MIMEType: "text/markdown",
            },
            Text: content,
        },
    }, nil
})

// Dynamic Resource
template := mcp.NewResourceTemplate(
    "users://{id}/profile",
    "User Profile",
    mcp.WithTemplateDescription("Get user profile"),
    mcp.WithTemplateMIMEType("application/json"),
    mcp.WithTemplateAnnotations([]mcp.Role{mcp.RoleAssistant}, 0.5),
)

server.AddResourceTemplate(template, func(ctx context.Context, request mcp.ReadResourceRequest) ([]interface{}, error) {
    userID := request.Params.URI
    profile := getUserProfile(userID)
    return []interface{}{
        mcp.TextResourceContents{
            ResourceContents: mcp.ResourceContents{
                URI: fmt.Sprintf("users://%s/profile", userID),
                MIMEType: "application/json",
            },
            Text: profile,
        },
    }, nil
})
```
## Tool Implementation
```go
// Tool argument structure
type CalculateArgs struct {
    Operation string `json:"operation" jsonschema:"required,description=Math operation to perform,enum=add,subtract,multiply,divide"`
    X float64 `json:"x" jsonschema:"required,description=First number"`
    Y float64 `json:"y" jsonschema:"required,description=Second number"`
}

// Tool registration
calculatorTool := mcp.NewTool("calculate",
    mcp.WithDescription("Perform math operations"),
    mcp.WithString("operation",
        mcp.Required(),
        mcp.Description("Operation type"),
        mcp.Enum("add", "subtract", "multiply", "divide"),
    ),
    mcp.WithNumber("x",
        mcp.Required(),
        mcp.Description("First number"),
    ),
    mcp.WithNumber("y",
        mcp.Required(),
        mcp.Description("Second number"),
    ),
)

server.AddTool(calculatorTool, func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    args := request.Params.Arguments
    op := args["operation"].(string)
    x := args["x"].(float64)
    y := args["y"].(float64)
    
    var result float64
    switch op {
        case "add": result = x + y
        case "subtract": result = x - y
        case "multiply": result = x * y
        case "divide":
            if y == 0 {
                return mcp.NewToolResultError("Division by zero"), nil
            }
            result = x / y
    }
    
    return mcp.NewToolResultText(fmt.Sprintf("%.2f", result)), nil
})
```
## Prompt Implementation
```yaml
# prompts.yaml
prompts:
  - name: "code_review"
    description: "Code review assistant"
    arguments:
      - name: "pr_number"
        description: "Pull request number"
        required: true
    prompt: >
      Review pull request {{.pr_number}} and provide detailed feedback.
      Focus on code quality, security, and performance.
```
```go
// Prompt registration
server.AddPrompt(mcp.NewPrompt("code_review",
    mcp.WithPromptDescription("Code review assistant"),
    mcp.WithArgument("pr_number",
        mcp.ArgumentDescription("PR number"),
        mcp.RequiredArgument(),
    ),
), func(ctx context.Context, request mcp.GetPromptRequest) (*mcp.GetPromptResult, error) {
    prNumber := request.Params.Arguments["pr_number"].(string)
    
    return mcp.NewGetPromptResult(
        "Code Review",
        []mcp.PromptMessage{
            mcp.NewPromptMessage(
                mcp.RoleSystem,
                mcp.NewTextContent("You are a code reviewer"),
            ),
            mcp.NewPromptMessage(
                mcp.RoleAssistant,
                mcp.NewEmbeddedResource(mcp.ResourceContents{
                    URI: fmt.Sprintf("git://pulls/%s/diff", prNumber),
                    MIMEType: "text/x-diff",
                }),
            ),
        },
    ), nil
})
```
## Transport Options
```go
// Stdio Transport (Full Feature Support)
transport := stdio.NewStdioServerTransport()
server := mcp_golang.NewServer(transport)

// HTTP Transport (Stateless)
transport := http.NewHTTPTransport("/mcp")
transport.WithAddr(":8080")
server := mcp_golang.NewServer(transport)

// Gin HTTP Transport
transport := http.NewGinTransport()
router := gin.Default()
router.POST("/mcp", transport.Handler())
server := mcp_golang.NewServer(transport)
```
## Error Handling
```go
// Tool error
if err != nil {
    return mcp.NewToolResultError(fmt.Sprintf("Operation failed: %v", err)), nil
}

// Resource error
if err != nil {
    return nil, fmt.Errorf("failed to fetch resource: %v", err)
}

// Prompt error
if err != nil {
    return nil, fmt.Errorf("failed to generate prompt: %v", err)
}
```
## Logging Implementation

```go
package main

import (
    "context"
    "os"
    "path/filepath"
    
    "github.com/metoro-io/mcp-golang"
    "github.com/metoro-io/mcp-golang/logging"
    "github.com/sirupsen/logrus"
)

// LogConfig holds logging configuration
type LogConfig struct {
    // File path for log output
    FilePath string `json:"file"`
    
    // Log level (debug, info, warn, error)
    Level string `json:"level" default:"info"`
    
    // Whether to also log to stderr
    WithStderr bool `json:"withStderr" default:"false"`
    
    // Optional: file for protocol debug information
    ProtocolDebugFile string `json:"protocolDebugFile,omitempty"`
}

// setupLogging configures the MCP server logging
func setupLogging(cfg LogConfig) error {
    // Create logger instance
    logger := logrus.New()
    
    // Set log level
    level, err := logrus.ParseLevel(cfg.Level)
    if err != nil {
        return fmt.Errorf("invalid log level %s: %v", cfg.Level, err)
    }
    logger.SetLevel(level)
    
    // Configure log format
    logger.SetFormatter(&logrus.JSONFormatter{
        TimestampFormat: "2006-01-02T15:04:05.000Z07:00",
        FieldMap: logrus.FieldMap{
            logrus.FieldKeyTime:  "timestamp",
            logrus.FieldKeyLevel: "level",
            logrus.FieldKeyMsg:   "message",
        },
    })
    
    // Ensure log directory exists
    if cfg.FilePath != "" {
        logDir := filepath.Dir(cfg.FilePath)
        if err := os.MkdirAll(logDir, 0755); err != nil {
            return fmt.Errorf("failed to create log directory: %v", err)
        }
        
        // Open log file
        file, err := os.OpenFile(cfg.FilePath, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
        if err != nil {
            return fmt.Errorf("failed to open log file: %v", err)
        }
        
        // Configure output
        if cfg.WithStderr {
            logger.SetOutput(io.MultiWriter(file, os.Stderr))
        } else {
            logger.SetOutput(file)
        }
    }
    
    // Configure protocol debugging if enabled
    if cfg.ProtocolDebugFile != "" {
        debugFile, err := os.OpenFile(cfg.ProtocolDebugFile, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
        if err != nil {
            return fmt.Errorf("failed to open protocol debug file: %v", err)
        }
        
        // Set up protocol debug logging
        mcp_golang.SetProtocolDebugWriter(debugFile)
    }
    
    // Set as default logger
    logging.SetLogger(logger)
    
    return nil
}

// Usage in main server setup
func main() {
    // Log configuration from server config
    logConfig := LogConfig{
        FilePath:          "/var/log/mcp/server.log",
        Level:            "debug",
        WithStderr:       true,
        ProtocolDebugFile: "/var/log/mcp/protocol.log",
    }
    
    // Initialize logging
    if err := setupLogging(logConfig); err != nil {
        panic(fmt.Sprintf("Failed to setup logging: %v", err))
    }
    
    // Get logger for use in handlers
    logger := logging.GetLogger()
    
    // Example logging in a tool handler
    server.AddTool(calculatorTool, func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
        logger.WithFields(logrus.Fields{
            "tool":      "calculator",
            "operation": request.Params.Arguments["operation"],
            "x":         request.Params.Arguments["x"],
            "y":         request.Params.Arguments["y"],
        }).Info("Processing calculation request")
        
        // ... handler implementation ...
        
        if err != nil {
            logger.WithError(err).Error("Calculation failed")
            return nil, err
        }
        
        logger.WithField("result", result).Debug("Calculation successful")
        return mcp.NewToolResultText(fmt.Sprintf("%.2f", result)), nil
    })
}
```
## Common logging patterns:
```go
// Info level with fields
logger.WithFields(logrus.Fields{
    "resource": "users",
    "action":   "fetch",
    "userID":   id,
}).Info("Fetching user profile")

// Error logging with error object
logger.WithError(err).Error("Operation failed")

// Debug logging with context
logger.WithContext(ctx).Debug("Processing completed")

// Warning with extra field
logger.WithField("component", "cache").Warn("Cache miss")

// Fatal error (will exit after logging)
logger.Fatal("Failed to initialize server")
```
# Complete Server Example
```go
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/metoro-io/mcp-golang"
    "github.com/metoro-io/mcp-golang/transport/stdio"
)

type CalculateArgs struct {
    Operation string `json:"operation" jsonschema:"required,enum=add,subtract,multiply,divide"`
    X float64 `json:"x" jsonschema:"required"`
    Y float64 `json:"y" jsonschema:"required"`
}

func main() {
    // Initialize server
    server := mcp_golang.NewServer(stdio.NewStdioServerTransport())

    // Add calculator tool
    err := server.RegisterTool("calculate", "Perform math operations", 
        func(args CalculateArgs) (*mcp_golang.ToolResponse, error) {
            var result float64
            switch args.Operation {
                case "add": result = args.X + args.Y
                case "subtract": result = args.X - args.Y
                case "multiply": result = args.X * args.Y
                case "divide":
                    if args.Y == 0 {
                        return mcp.NewToolResultError("Division by zero"), nil
                    }
                    result = args.X / args.Y
            }
            return mcp_golang.NewToolResponse(
                mcp_golang.NewTextContent(fmt.Sprintf("%.2f", result)),
            ), nil
        })
    if err != nil {
        log.Fatalf("Failed to register tool: %v", err)
    }

    // Add documentation resource
    err = server.RegisterResource(
        "docs://api",
        "API Documentation",
        "API documentation resource",
        "text/markdown",
        func() (*mcp_golang.ResourceResponse, error) {
            return mcp_golang.NewResourceResponse(
                mcp_golang.NewTextEmbeddedResource(
                    "docs://api",
                    "# API Documentation\n## Endpoints...",
                    "text/markdown",
                ),
            ), nil
        },
    )
    if err != nil {
        log.Fatalf("Failed to register resource: %v", err)
    }

    // Start server
    if err := server.Serve(); err != nil {
        log.Fatalf("Server error: %v", err)
    }

    // Keep running
    select {}
}
```
# Response Types Summary
## Tool Response
```go
return mcp.NewToolResponse(
    mcp.NewTextContent("Result: 42"),
), nil
```
## Resource Response
```go
return mcp.NewResourceResponse(
    mcp.NewTextEmbeddedResource(
        "uri://resource",
        "Content",
        "application/json",
    ),
), nil
```
## Prompt Response
```go
return mcp.NewGetPromptResult(
    "Title",
    []mcp.PromptMessage{
        mcp.NewPromptMessage(
            mcp.RoleSystem,
            mcp.NewTextContent("System message"),
        ),
        mcp.NewPromptMessage(
            mcp.RoleAssistant,
            mcp.NewTextContent("Assistant message"),
        ),
    },
), nil
```
