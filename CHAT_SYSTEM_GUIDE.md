# ryOS Chat System Architecture Guide

This document provides a comprehensive overview of the chat system within ryOS, detailing its components, architecture, and integration points. Use this as a reference for understanding how the system works and where to implement new features.

## 🏗️ System Overview

The chat system provides two primary modes:
1. **AI Chat with Ryo** - Direct conversation with Claude-powered AI assistant
2. **Multi-user Chat Rooms** - Real-time messaging between users

### Key Technologies
- **Frontend**: React + TypeScript + Zustand state management
- **Backend**: Vercel Edge Functions + Redis + Pusher
- **AI**: Anthropic Claude with streaming responses
- **Real-time**: Pusher WebSockets for live updates

## 📁 Directory Structure

```
src/apps/chats/                 # Main chat application
├── components/                 # React components
│   ├── ChatsAppComponent.tsx   # Main orchestrator (925 lines)
│   ├── ChatInput.tsx          # Message input + voice (665 lines)
│   ├── ChatMessages.tsx       # Message display (1,369 lines)
│   ├── ChatRoomSidebar.tsx    # Room navigation (221 lines)
│   ├── ChatsMenuBar.tsx       # App menu bar
│   └── CreateRoomDialog.tsx   # Room creation dialog
├── hooks/                     # Custom React hooks
│   ├── useAiChat.ts          # AI chat logic (1,505 lines)
│   ├── useChatRoom.ts        # Room management (501 lines)
│   ├── useRyoChat.ts         # Specialized Ryo chat (192 lines)
│   └── useTokenRefresh.ts    # Auth token management (89 lines)
└── index.tsx                 # App definition & metadata

api/                           # Backend API endpoints
├── chat.ts                   # AI chat endpoint (967 lines)
└── chat-rooms.js            # Room management API (3,771+ lines)

src/stores/useChatsStore.ts   # Global state management (1,634 lines)
src/types/chat.ts            # TypeScript interfaces
src/utils/chat.ts            # Chat utility functions
```

## 🧩 Core Components Deep Dive

### ChatsAppComponent.tsx - Main Orchestrator
**Purpose**: Central component that coordinates all chat functionality

**Key Responsibilities**:
- Manages UI state (dialogs, sidebar visibility)
- Coordinates between AI chat and room chat modes
- Handles authentication flow
- Integrates all sub-components

**Important Hooks Used**:
```typescript
const authResult = useAuth();           // Authentication management
const chatRoomResult = useChatRoom();   // Room functionality
const aiChatResult = useAiChat();       // AI chat functionality
```

**Integration Points**:
- Window management via `WindowFrame`
- Menu bar integration via `ChatsMenuBar`
- Font size controls from global store

### ChatInput.tsx - Message Input Interface
**Purpose**: Handles all user input including text and voice

**Key Features**:
- Text input with history navigation (arrow keys)
- Voice recording with transcription
- Push-to-talk (spacebar when unfocused)
- @mention functionality for rooms
- Rate limiting feedback
- Theme-aware styling

**Voice Integration**:
```typescript
<AudioInputButton
  onTranscriptionComplete={handleTranscriptionComplete}
  onTranscriptionStart={handleTranscriptionStart}
  onRecordingStateChange={handleRecordingStateChange}
/>
```

**Props Interface**:
- `onDirectMessageSubmit` - Direct message sending
- `showNudgeButton` - Enable/disable nudge feature
- `isInChatRoom` - Room vs AI chat mode
- `rateLimitError` - Display rate limit info

### ChatMessages.tsx - Message Display Engine
**Purpose**: Renders all message types with rich formatting

**Key Features**:
- Markdown rendering (bold, italic, links)
- Link previews for URLs
- Tool invocation displays
- Message highlighting during TTS
- User color coding
- Emoji-only message detection
- Admin controls (delete messages)

**Message Types Supported**:
```typescript
interface ChatMessage {
  role: "user" | "assistant" | "human";
  content: string;
  username?: string;
  timestamp: number;
  parts?: ToolInvocationPart[]; // For AI responses with tools
}
```

**Rendering Features**:
- Segmented text rendering for proper highlighting
- HTML entity decoding
- URL extraction and preview
- Responsive layout with scroll-to-bottom

### ChatRoomSidebar.tsx - Room Navigation
**Purpose**: Provides room selection and management interface

**Features**:
- Room list with unread counts
- Private vs public room sorting
- Admin controls (delete rooms)
- @ryo chat selection
- Real-time user count display

**Room Display Logic**:
```typescript
// Private rooms show member names, public rooms show #channel-name
{room.type === "private" 
  ? getPrivateRoomDisplayName(room, username) 
  : `#${room.name}`}
```

## 🔧 Custom Hooks Architecture

### useAiChat.ts - AI Chat Management
**Purpose**: Manages conversation with Claude AI

**Key Functionality**:
- Streaming AI responses
- Tool calling integration
- Rate limiting handling
- Message history management
- Speech synthesis integration
- Error handling and retry logic

**Tool Integration Example**:
```typescript
// AI can call tools like launching apps, editing text, etc.
tools: {
  launchApp: { /* App launching logic */ },
  textEditSearchReplace: { /* Text editing */ },
  // ... other tools
}
```

### useChatRoom.ts - Room Management
**Purpose**: Handles multi-user chat rooms

**Key Functionality**:
- Room creation/deletion
- Message sending/receiving
- Real-time presence tracking
- Room switching logic
- Pusher event handling

**Real-time Integration**:
```typescript
// Pusher channels for live updates
const channelName = `room-${roomId}`;
pusher.subscribe(channelName);
```

### useRyoChat.ts - Specialized AI Features
**Purpose**: Ryo-specific AI functionality

**Features**:
- Context-aware responses
- System state integration
- Specialized prompt handling

## 🗄️ State Management (useChatsStore.ts)

### Store Structure
```typescript
interface ChatsStoreState {
  // AI Chat
  aiMessages: Message[];
  
  // Authentication
  username: string | null;
  authToken: string | null;
  hasPassword: boolean | null;
  
  // Rooms
  rooms: ChatRoom[];
  currentRoomId: string | null;
  roomMessages: Record<string, ChatMessage[]>;
  unreadCounts: Record<string, number>;
  
  // UI State
  isSidebarVisible: boolean;
  fontSize: number;
  hasEverUsedChats: boolean;
}
```

### Key Patterns
- **Optimistic Updates**: Messages appear immediately, then sync with server
- **Message Caching**: Room messages persist across sessions
- **Authentication Recovery**: Tokens stored in localStorage with encoding
- **Bulk Operations**: Efficient room message fetching

### Important Actions
```typescript
// Room Management
switchRoom(roomId: string | null)
sendMessage(roomId: string, content: string)
createRoom(name: string, type: "public" | "private", members?: string[])

// Authentication
ensureAuthToken()
refreshAuthToken()
logout()

// Message Management
addMessageToRoom(roomId: string, message: ChatMessage)
fetchBulkMessages(roomIds: string[])
```

## 🌐 Backend API Architecture

### chat.ts - AI Chat Endpoint
**Purpose**: Handles AI conversations with Claude

**Key Features**:
- Streaming responses
- Tool calling integration
- System prompt injection
- Rate limiting
- Authentication validation
- Geolocation context

**System State Integration**:
```typescript
interface SystemState {
  apps: { [appId: string]: { isOpen: boolean } };
  username?: string;
  internetExplorer: { url: string; year: string };
  textEdit?: { instances: Array<{ instanceId: string; content: string }> };
  // ... other app states
}
```

### chat-rooms.js - Room Management API
**Purpose**: Manages multi-user chat functionality

**Endpoints**:
- `GET ?action=getRooms` - Fetch available rooms
- `GET ?action=getMessages&roomId=X` - Get room messages
- `POST ?action=createRoom` - Create new room
- `POST ?action=sendMessage` - Send message to room
- `DELETE ?action=deleteRoom&roomId=X` - Delete room

**Authentication Flow**:
```javascript
// Token validation with refresh capability
const validateAuth = async (username, token, requestId, allowExpired = false)
```

**Real-time Events**:
- Pusher integration for live message delivery
- Presence tracking with TTL
- Room count updates

## 🔌 Integration Points for New Features

### Adding New Tool Capabilities
1. **Define tool in chat.ts**:
```typescript
tools: {
  yourNewTool: {
    description: "Description of what your tool does",
    parameters: z.object({
      // Define parameters using Zod schema
    }),
  }
}
```

2. **Handle tool result in ChatMessages.tsx**:
```typescript
case "tool-invocation":
  return <YourToolComponent part={part} />
```

### Adding New Message Types
1. **Extend ChatMessage interface** in `src/types/chat.ts`
2. **Update message rendering** in `ChatMessages.tsx`
3. **Add backend validation** in `chat-rooms.js`

### Adding New Chat Commands
1. **Detect command pattern** in `ChatInput.tsx`
2. **Process command** in appropriate hook
3. **Update UI state** accordingly

### Integrating External APIs
**Example: Chart Generation Integration**

1. **Add new tool to chat.ts**:
```typescript
generateChart: {
  description: "Generate charts from data queries",
  parameters: z.object({
    query: z.string(),
    chartType: z.enum(["bar", "line", "pie"]),
  }),
}
```

2. **Create chart component**:
```typescript
// src/components/shared/ChartDisplay.tsx
export function ChartDisplay({ data, type }) {
  // Render chart using your chart library
}
```

3. **Handle in tool invocation**:
```typescript
case "tool-invocation":
  if (ti?.toolName === "generateChart") {
    return <ChartDisplay data={ti.result} type={ti.args.chartType} />
  }
```

## 🎨 Theming and Styling

### Theme Support
The chat system supports multiple OS themes:
- `system7` - Classic Mac OS
- `macosx` - Mac OS X style
- `xp` - Windows XP
- `win98` - Windows 98

### Responsive Design
- Sidebar toggles on mobile
- Font size controls
- Touch-friendly interactions
- Adaptive layouts

### Accessibility
- ARIA labels on interactive elements
- Keyboard navigation support
- Screen reader compatibility
- High contrast support

## 🔒 Security Considerations

### Authentication
- JWT-like token system
- Multi-device support
- Token refresh mechanism
- Grace period for expired tokens

### Input Validation
- Message length limits
- Profanity filtering
- Username validation
- Rate limiting

### API Security
- CORS validation
- Origin checking
- Request sanitization
- Admin privilege checks

## 🚀 Performance Optimizations

### Frontend
- Message virtualization for large chat histories
- Optimistic UI updates
- Debounced API calls
- Lazy loading of components

### Backend
- Redis caching
- Bulk message operations
- Efficient presence tracking
- Rate limiting

### Real-time
- Pusher for WebSocket connections
- Event batching
- Selective channel subscriptions

## 🧪 Testing Considerations

### Component Testing
- Message rendering with various content types
- Input validation and submission
- Room switching functionality
- Authentication flows

### Integration Testing
- API endpoint validation
- Real-time message delivery
- Tool calling functionality
- Error handling scenarios

### Performance Testing
- Large message history handling
- Multiple concurrent users
- Network failure recovery

## 📝 Development Workflow

### Adding New Features
1. Update TypeScript interfaces
2. Implement frontend components
3. Add backend API endpoints
4. Update state management
5. Add real-time integration
6. Test across themes and devices

### Debugging Tools
- Console logging with request IDs
- Redux DevTools for state inspection
- Network tab for API debugging
- Pusher debug console for real-time events

---

This guide should provide a comprehensive understanding of the chat system architecture and serve as a roadmap for extending or integrating new functionality.