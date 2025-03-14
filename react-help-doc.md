# WebSocket Integration Guide for React Frontend Developers

This document provides comprehensive guidance for integrating with our backend WebSocket service in your React application.

## Table of Contents

1. [System Overview](#system-overview)
2. [WebSocket Connection](#websocket-connection)
3. [Message Types](#message-types)
4. [Streaming Updates](#streaming-updates)
5. [Error Handling](#error-handling)
6. [React Implementation](#react-implementation)
7. [Complete Example](#complete-example)
8. [Testing](#testing)

## System Overview

Our system provides real-time testing and deployment capabilities for Git repositories through WebSocket communication. The architecture consists of:

- **WebSocket Server**: A Go-based service that handles connections and orchestrates operations
- **Kubernetes Backend**: Isolated namespaces for testing and deployment
- **Streaming Updates**: Real-time progress updates during operations

The workflow is:
1. Connect to WebSocket
2. Set up a namespace (one-time operation)
3. Run tests on specific commits
4. Deploy successful commits to production

![Workflow Diagram](./workflow.png)

## WebSocket Connection

### Connection Details

- **WebSocket URL**: `ws://brain.obimadu.pro/ws` (use wss:// for production)
- **Protocol**: Standard WebSocket (no custom sub-protocols)

### Basic Connection Example

```javascript
const socket = new WebSocket('ws://brain.obimadu.pro/ws');

socket.onopen = () => {
  console.log('Connected to WebSocket server');
};

socket.onclose = (event) => {
  console.log(`Connection closed: ${event.code} ${event.reason}`);
};

socket.onerror = (error) => {
  console.error('WebSocket error:', error);
};

socket.onmessage = (event) => {
  const message = JSON.parse(event.data);
  console.log('Received:', message);
};
```

## Message Types

### Outgoing Messages (Client to Server)

#### 1. Namespace Setup

```javascript
{
  "type": "namespace_setup",
  "projectId": "my-project-id",
  "projectType": "fastapi"  // Currently supported: "fastapi"
}
```

#### 2. Test Request

```javascript
{
  "type": "test_request",
  "projectId": "my-project-id",
  "repoURL": "https://git.example.com/my-repo.git",
  "commitHash": "abc123def456",
  "projectType": "fastapi",
  "testCommand": "pytest tests/",  // Optional, defaults to "pytest tests/"
  "startCommand": "uvicorn main:app --host 0.0.0.0 --port $DEPLOY_PORT"  // Optional
}
```

#### 3. Deploy Request

```javascript
{
  "type": "deploy_request",
  "projectId": "my-project-id",
  "repoURL": "https://git.example.com/my-repo.git",
  "commitHash": "abc123def456",
  "projectType": "fastapi",
  "startCommand": "uvicorn main:app --host 0.0.0.0 --port $DEPLOY_PORT"  // Optional
}
```

### Incoming Messages (Server to Client)

#### 1. Namespace Updates

```javascript
{
  "type": "namespace_update",
  "status": "partial",
  "payload": {
    "update_type": "step",  // Can be: "start", "step", "command", "status", "url", "error", "log", "complete"
    "update_data": {
      // Content varies based on update_type
      "data": {
        "step": "namespace_check",
        "description": "Checking if namespace exists",
        "timestamp": "2023-05-15T12:34:56Z"
      }
    }
  }
}
```

#### 2. Namespace Complete

```javascript
{
  "type": "namespace_complete",
  "status": "success",
  "payload": {
    "message": "Namespace setup completed"
  }
}
```

#### 3. Test Updates

```javascript
{
  "type": "test_update",
  "status": "partial",
  "payload": {
    "update_type": "step",  // Can be: "start", "step", "output", "error", "log", "transition", "complete"
    "update_data": {
      // Content varies based on update_type
      "data": {
        "step": "test_execution",
        "description": "Running tests",
        "timestamp": "2023-05-15T12:34:56Z"
      }
    }
  }
}
```

#### 4. Test Complete

```javascript
{
  "type": "test_complete",
  "status": "success",
  "payload": {
    "message": "Test execution completed"
  }
}
```

#### 5. Deploy Updates

```javascript
{
  "type": "deploy_update",
  "status": "partial",
  "payload": {
    "update_type": "step",  // Can be: "start", "step", "output", "error", "log", "result", "complete"
    "update_data": {
      // Content varies based on update_type
      "data": {
        "step": "deploy_start",
        "description": "Starting deployment",
        "timestamp": "2023-05-15T12:34:56Z"
      }
    }
  }
}
```

#### 6. Deploy Complete

```javascript
{
  "type": "deploy_complete",
  "status": "success",
  "payload": {
    "message": "Deployment completed",
    "url": "http://im-my-project-id.backend.im"  // URL to access the deployed application
  }
}
```

#### 7. Error Response

```javascript
{
  "type": "error",
  "status": "error",
  "payload": {
    "code": "namespace_required",  // Error code
    "message": "Namespace does not exist. Call namespace_setup first"  // Human-readable message
  }
}
```

## Streaming Updates

The server sends streaming updates during long-running operations. These updates follow a consistent pattern:

1. **Operation Start**: Indicates the beginning of an operation
2. **Step Updates**: Progress through defined steps
3. **Output/Log Updates**: Raw output from commands
4. **Completion**: Final status and results

### Update Types

| Update Type | Description | Fields |
|-------------|-------------|--------|
| `start` | Operation beginning | `commit`, `test_cmd`, `namespace`, etc. |
| `step` | Progress milestone | `step` (ID), `description`, `timestamp` |
| `command` | Command execution | `command` (shell command being run) |
| `output` | Command output | `stdout`, `stderr` |
| `error` | Error information | `message`, `step` (optional) |
| `log` | Informational message | `message` |
| `result` | Final operation result | Varies by operation |
| `transition` | Changing between phases | `from`, `to` (e.g., test → deploy) |
| `complete` | Operation finished | Final result data |

## Error Handling

### Common Error Codes

| Error Code | Description | Resolution |
|------------|-------------|------------|
| `invalid_message_format` | Malformed JSON or missing fields | Check message structure |
| `namespace_required` | Namespace doesn't exist | Call namespace_setup first |
| `namespace_check_failed` | Failed to check namespace | Check Kubernetes connectivity |
| `unknown_message_type` | Invalid message type | Use supported message types |

### Error Response Example

```javascript
{
  "type": "error",
  "status": "error",
  "payload": {
    "code": "namespace_required",
    "message": "Namespace does not exist. Call namespace_setup first"
  }
}
```

## React Implementation

### WebSocket Hook

Here's a custom React hook for managing WebSocket connections:

```jsx
// useWebSocket.js
import { useState, useEffect, useCallback, useRef } from 'react';

export const useWebSocket = (url) => {
  const [isConnected, setIsConnected] = useState(false);
  const [messages, setMessages] = useState([]);
  const [error, setError] = useState(null);
  const socketRef = useRef(null);

  // Connect to WebSocket
  useEffect(() => {
    const socket = new WebSocket(url);
    socketRef.current = socket;

    socket.onopen = () => {
      setIsConnected(true);
      setError(null);
    };

    socket.onclose = (event) => {
      setIsConnected(false);
      if (event.code !== 1000) {
        setError(`Connection closed: ${event.code} ${event.reason}`);
      }
    };

    socket.onerror = (event) => {
      setError('WebSocket error');
    };

    socket.onmessage = (event) => {
      const message = JSON.parse(event.data);
      setMessages((prev) => [...prev, message]);
    };

    return () => {
      socket.close();
    };
  }, [url]);

  // Send message to WebSocket
  const sendMessage = useCallback((message) => {
    if (socketRef.current && socketRef.current.readyState === WebSocket.OPEN) {
      socketRef.current.send(JSON.stringify(message));
    } else {
      setError('WebSocket is not connected');
    }
  }, []);

  // Clear messages
  const clearMessages = useCallback(() => {
    setMessages([]);
  }, []);

  return {
    isConnected,
    messages,
    error,
    sendMessage,
    clearMessages
  };
};
```

### Streaming Updates Component

This component handles streaming updates from the server:

```jsx
// StreamingUpdates.js
import React, { useEffect, useState } from 'react';

const StreamingUpdates = ({ messages, operationType }) => {
  const [steps, setSteps] = useState([]);
  const [logs, setLogs] = useState([]);
  const [currentStatus, setCurrentStatus] = useState('waiting');
  const [result, setResult] = useState(null);

  // Filter for specific operation updates
  useEffect(() => {
    if (!messages || !messages.length) return;

    // Filter messages for the current operation type
    const relevantMessages = messages.filter(
      msg => msg.type === `${operationType}_update` || 
             msg.type === `${operationType}_complete` ||
             (msg.type === 'error' && msg.payload?.context === operationType)
    );

    // Process each message
    relevantMessages.forEach(message => {
      if (message.type === `${operationType}_update`) {
        const { update_type, update_data } = message.payload;
        
        if (update_type === 'step') {
          setSteps(prev => [...prev, update_data.data]);
        } else if (update_type === 'log') {
          setLogs(prev => [...prev, update_data.data.message]);
        } else if (update_type === 'error') {
          setCurrentStatus('error');
        } else if (update_type === 'result') {
          setResult(update_data.data);
          setCurrentStatus('complete');
        }
      } else if (message.type === `${operationType}_complete`) {
        setCurrentStatus('complete');
        if (message.payload) {
          setResult(message.payload);
        }
      } else if (message.type === 'error') {
        setCurrentStatus('error');
      }
    });
  }, [messages, operationType]);

  return (
    <div className="streaming-updates">
      <h3>Operation Status: {currentStatus}</h3>
      
      {steps.length > 0 && (
        <div className="steps">
          <h4>Progress Steps</h4>
          <ul>
            {steps.map((step, index) => (
              <li key={index}>
                <span className="timestamp">{step.timestamp}</span>
                <span className="description">{step.description}</span>
                <span className="step-id">({step.step})</span>
              </li>
            ))}
          </ul>
        </div>
      )}
      
      {logs.length > 0 && (
        <div className="logs">
          <h4>Logs</h4>
          <pre>{logs.join('\n')}</pre>
        </div>
      )}
      
      {result && (
        <div className="result">
          <h4>Result</h4>
          <pre>{JSON.stringify(result, null, 2)}</pre>
        </div>
      )}
    </div>
  );
};

export default StreamingUpdates;
```

### Operation Control Component

This component handles sending operation requests:

```jsx
// OperationControl.js
import React, { useState } from 'react';

const OperationControl = ({ isConnected, sendMessage, projectId }) => {
  const [repoURL, setRepoURL] = useState('');
  const [commitHash, setCommitHash] = useState('');
  const [testCommand, setTestCommand] = useState('pytest tests/');
  const [startCommand, setStartCommand] = useState('uvicorn main:app --host 0.0.0.0 --port $DEPLOY_PORT');
  const [projectType, setProjectType] = useState('fastapi');

  const setupNamespace = () => {
    sendMessage({
      type: 'namespace_setup',
      projectId,
      projectType
    });
  };

  const runTests = () => {
    sendMessage({
      type: 'test_request',
      projectId,
      repoURL,
      commitHash,
      projectType,
      testCommand,
      startCommand
    });
  };

  const deployApplication = () => {
    sendMessage({
      type: 'deploy_request',
      projectId,
      repoURL,
      commitHash,
      projectType,
      startCommand
    });
  };

  return (
    <div className="operation-control">
      <h2>Operation Controls</h2>
      
      <div className="control-group">
        <label>
          Project Type:
          <select 
            value={projectType} 
            onChange={(e) => setProjectType(e.target.value)}
          >
            <option value="fastapi">FastAPI</option>
          </select>
        </label>
      </div>
      
      <div className="control-group">
        <label>
          Repository URL:
          <input 
            type="text" 
            value={repoURL} 
            onChange={(e) => setRepoURL(e.target.value)} 
            placeholder="https://git.example.com/repo.git"
          />
        </label>
      </div>
      
      <div className="control-group">
        <label>
          Commit Hash:
          <input 
            type="text" 
            value={commitHash} 
            onChange={(e) => setCommitHash(e.target.value)} 
            placeholder="abc123def456"
          />
        </label>
      </div>
      
      <div className="control-group">
        <label>
          Test Command:
          <input 
            type="text" 
            value={testCommand} 
            onChange={(e) => setTestCommand(e.target.value)} 
          />
        </label>
      </div>
      
      <div className="control-group">
        <label>
          Start Command:
          <input 
            type="text" 
            value={startCommand} 
            onChange={(e) => setStartCommand(e.target.value)} 
          />
        </label>
      </div>
      
      <div className="button-group">
        <button 
          onClick={setupNamespace} 
          disabled={!isConnected || !projectId || !projectType}
        >
          Setup Namespace
        </button>
        
        <button 
          onClick={runTests} 
          disabled={!isConnected || !projectId || !repoURL || !commitHash}
        >
          Run Tests
        </button>
        
        <button 
          onClick={deployApplication} 
          disabled={!isConnected || !projectId || !repoURL || !commitHash}
        >
          Deploy Application
        </button>
      </div>
    </div>
  );
};

export default OperationControl;
```

## Complete Example

Here's a complete React application that integrates all components:

```jsx
// App.js
import React, { useState } from 'react';
import { useWebSocket } from './hooks/useWebSocket';
import OperationControl from './components/OperationControl';
import StreamingUpdates from './components/StreamingUpdates';
import './App.css';

const App = () => {
  const [projectId, setProjectId] = useState('');
  const [activeOperation, setActiveOperation] = useState(null);
  
  const wsUrl = process.env.REACT_APP_WS_URL || 'ws://brain.obimadu.pro/ws';
  const { isConnected, messages, error, sendMessage, clearMessages } = useWebSocket(wsUrl);

  const handleSendMessage = (message) => {
    // Set the active operation based on message type
    const operationType = message.type.split('_')[0]; // namespace, test, or deploy
    setActiveOperation(operationType);
    
    // Clear previous messages when starting a new operation
    clearMessages();
    
    // Send the message
    sendMessage(message);
  };

  return (
    <div className="app">
      <header>
        <h1>Backend.im Testing & Deployment</h1>
        <div className={`connection-status ${isConnected ? 'connected' : 'disconnected'}`}>
          {isConnected ? 'Connected' : 'Disconnected'}
        </div>
      </header>
      
      {error && (
        <div className="error-banner">
          Error: {error}
        </div>
      )}
      
      <main>
        <div className="project-id-container">
          <label>
            Project ID:
            <input 
              type="text" 
              value={projectId} 
              onChange={(e) => setProjectId(e.target.value)} 
              placeholder="my-project-id"
            />
          </label>
        </div>
        
        <div className="control-panel">
          <OperationControl 
            isConnected={isConnected} 
            sendMessage={handleSendMessage} 
            projectId={projectId} 
          />
        </div>
        
        <div className="results-panel">
          {activeOperation && (
            <StreamingUpdates 
              messages={messages} 
              operationType={activeOperation} 
            />
          )}
        </div>
      </main>
    </div>
  );
};

export default App;
```

## Testing

### Local Testing

For local testing, you can use a mock WebSocket server:

```javascript
// mockWebSocketServer.js
const WebSocket = require('ws');

const server = new WebSocket.Server({ port: 8080 });

server.on('connection', (socket) => {
  console.log('Client connected');
  
  socket.on('message', (message) => {
    console.log(`Received: ${message}`);
    
    // Parse the message
    const parsedMessage = JSON.parse(message);
    
    // Mock responses based on message type
    if (parsedMessage.type === 'namespace_setup') {
      // Send a series of updates
      setTimeout(() => {
        socket.send(JSON.stringify({
          type: 'namespace_update',
          status: 'partial',
          payload: {
            update_type: 'start',
            update_data: {
              data: {
                namespace: `im-${parsedMessage.projectId}`,
                project_type: parsedMessage.projectType
              }
            }
          }
        }));
      }, 500);
      
      setTimeout(() => {
        socket.send(JSON.stringify({
          type: 'namespace_update',
          status: 'partial',
          payload: {
            update_type: 'step',
            update_data: {
              data: {
                step: 'namespace_check',
                description: 'Checking if namespace exists',
                timestamp: new Date().toISOString()
              }
            }
          }
        }));
      }, 1000);
      
      // More updates...
      
      setTimeout(() => {
        socket.send(JSON.stringify({
          type: 'namespace_complete',
          status: 'success',
          payload: {
            message: 'Namespace setup completed'
          }
        }));
      }, 3000);
    }
    
    // Similar mock responses for test_request and deploy_request
  });
  
  socket.on('close', () => {
    console.log('Client disconnected');
  });
});

console.log('Mock WebSocket server running on ws://localhost:8080');
```

### Testing with the Real Backend

To test with the real backend:

1. Set the WebSocket URL to the actual backend URL
2. Use valid project IDs and repository URLs
3. Start with namespace setup before running tests or deployments
4. Check the console for detailed message logs

## Conclusion

This integration guide provides all the necessary information to build a React frontend that communicates with our WebSocket-based testing and deployment system. The streaming updates architecture allows for real-time feedback during long-running operations, providing a responsive user experience.

For additional support or questions, please contact the backend team.
