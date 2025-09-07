# Game Architecture Definitions

## The Game description
This multiplayer game is built on an authoritative server architecture using Node.js, Express, and Socket.IO to manage the true game state and enforce rules in real-time. Clients, developed from a single codebase using React Native for both Android and web (via `react-native-web`), connect via Socket.IO to send player actions and receive state updates. To ensure low-latency, responsive gameplay, the state for active game matches is managed entirely in-memory on the server, while a persistent database handles player accounts and other non-session data.

## Git repositories
* multiplayer-back
* multiplayer-front
* shared-types

## The Game Client (Frontend)
* Single codebase with react-native-web
* Shared components and logic
* Build: Metro bundler for both platforms
```
┌─────────────────┐    ┌──────────────────┐
│ React Native    │    │  Authoritative   │
│   Web App       │◄──►│     Server       │
│ (react-native-  │    │                  │
│      web)       │    └──────────────────┘
└─────────────────┘             │
         │                      │
         │         ┌─────────────────┐
         └────────►│ React Native    │
                   │   Android       │
                   │     Client      │
                   └─────────────────┘
```

## The Authoritative Server (Backend)
* It holds the true state of the game, enforces the rules, and communicates changes to all players.
* **Technology**: Node.js with Express and Socket.IO.
    - Why Node.js? It allows you to write your server in JavaScript, the same language as your client. This simplifies development immensely.
    - Why Express? It's a minimal framework for handling basic web server needs (like serving your game files).
    - Why Socket.IO? This is the magic for real-time communication. It simplifies WebSockets, handles disconnections and reconnections gracefully, and makes it easy to create game "rooms" for players.

### Scalability and State Management
While managing game state in-memory is fast, it presents a significant challenge: it creates a single point of failure and a scaling bottleneck.

*   **Problem:** If the Node.js process crashes or the server needs to be rebooted, **all active games will be instantly terminated and their state lost.** Furthermore, a single server can only handle a finite number of concurrent games.

*   **Suggested Solutions:**
    *   **Introduce Redis:** For a more robust solution, use an in-memory data store like Redis to manage game state.
        *   **Fault Tolerance:** If the Node.js process crashes, it can restart and immediately recover the game states from Redis.
        *   **Horizontal Scaling:** With Redis as a shared state store, you can run multiple Node.js server instances behind a load balancer. A player's connection can be routed to any server instance, which can then read/write the relevant game state from the central Redis store. This is crucial for scaling to a large number of players.
    *   **Periodic State Snapshots:** As a lighter alternative, periodically write game states to the persistent database (e.g., every minute). This doesn't prevent data loss on a crash but allows for potential game recovery, albeit not in real-time.

## The Communication Layer
This is the protocol that the client and server use to talk to each other in real-time.

*   **Technology:** WebSockets (managed by Socket.IO).

*   **Basic Flow:**
    1.  A player makes a move (e.g., clicks a card to play).
    2.  The client sends a small message to the server, like `{ action: 'PLAY_CARD', cardId: 'king_of_hearts' }`.
    3.  The server receives this, validates the move against the game rules (Is it this player's turn? Is that a valid card to play?).
    4.  If the move is valid, the server updates the master game state.
    5.  The server then broadcasts the change to all players in that game room.
    6.  All clients receive the update and re-render their UI to reflect the change.

### Protocol Efficiency: Delta Updates
*   **Problem:** Broadcasting the entire game state on every turn is simple but inefficient. As the game state grows, this consumes unnecessary bandwidth and can slow down updates.

*   **Suggestion: Implement Delta/Diff Updates:** Instead of sending the entire state, calculate and send only what has changed. For example, instead of sending the whole `gameState` object, send a discrete event like `{ event: 'CARD_PLAYED', player: 'p1', card: 'king_of_hearts', nextPlayer: 'p2' }`. The client's state store (Zustand) is then responsible for applying this "patch" to its local state. This drastically reduces payload size. To support this, you should **define a strict event schema** in your `src/types/` directory for all client-to-server and server-to-client events.

### Handling Latency: Client-Side Prediction
*   **Problem:** Network latency can make the game feel sluggish, as a player must wait for a full server round-trip before their UI updates.

*   **Suggestion: Use Optimistic Updates:** This is an advanced technique where the client UI updates *immediately* when a player acts, predicting success.
    *   The client sends the action to the server in the background.
    *   If the server confirms the move (the common case), no further UI change is needed.
    *   If the server *rejects* the move, it sends a rejection message, and the client "rolls back" its UI to the previous state, perhaps showing an error. This makes the game feel instantaneous.

### Robust Reconnections
*   **Problem:** Mobile clients frequently disconnect and reconnect. Socket.IO handles the transport layer, but the application must handle state synchronization.

*   **Suggestions:**
    *   **Full State Sync on Reconnect:** When a client's `connect` event fires (including for reconnections), it should immediately emit a `FETCH_GAME_STATE` event. The server should then reply directly to that client with the *entire current game state* to ensure the player is fully synchronized.
    *   **Server-Side Disconnection Handling:** Define a clear policy for disconnected players. Does the game pause? For how long? Is the player forfeited after 60 seconds? This logic is critical for a smooth user experience.

## Visual Diagram of the Architecture
```
+--------------------------------+      +--------------------------------+
|     Web Client (Browser)       |      |    Native Client (Android)     |
| +----------------------------+ |      | +----------------------------+ |
| |    React App               | |      | |    React App (JS Thread)   | |
| | (react-native-web)         | |      | +----------------------------+ |
| | +------------------------+ | |      |              |                 |
| | | Shared UI Components   | | |      | React Native | Bridge          |
| | +------------------------+ | |      |              v                 |
| | | Socket.IO Client       | | |      | +----------------------------+ |
| | +------------------------+ | |      | |   Native UI Components     | |
| | | Shared Game Logic      | | |      | |   (Rendered by OS)         | |
| | +------------------------+ | |      | +----------------------------+ |
| +----------------------------+ |      +--------------------------------+
+--------------------------------+
                  ^
                  | Real-time Events (WebSockets / Socket.IO)
                  v
+-------------------------------------------------------------+
|                         YOUR SERVER                         |
|                                                             |
| +---------------------------------------------------------+ |
| |                  Node.js Application                    | |
| |                                                         | |
| | +------------------+   +------------------------------+ | |
| | |  Express Server  |   |       Socket.IO Server       | | |
| | | (Serves Web App, |◄──►| (Manages Rooms & Real-time   | | |
| | |    API REST)     |   |          Connections)        | | |
| | +------------------+   +------------------------------+ | |
| |         ^                           ^                   | |
| |         |                           |                   | |
| |         └─────────────┐   ┌─────────┘                   | |
| |                       v   v                             | |
| |                 +-----------------+                     | |
| |                 |   Game Logic    |                     | |
| |                 |  (State Mgmt)   |                     | |
| |                 +-----------------+                     | |
| |                           |                             | |
| |                           v                             | |
| |               +-----------------------+                 | |
| |               | Redis (In-Memory State) |               | |   
| |               +-----------------------+                 | |
| |                           |                             | |
| |                           v                             | |
| |                    +--------------+                     | |
| |                    |   Database   |                     | |
| |                    |  (Accounts,  |                     | |
| |                    |   History)   |                     | |
| |                    +--------------+                     | |
| +---------------------------------------------------------+ |
|                                                             |
+-------------------------------------------------------------+

```

## Frontend Code Structure:
This structure incorporates best practices for testing, configuration, and code sharing.
```
src/
├── __tests__/      # Unit and integration tests.
│   └── game/
│       └── GameRules.test.ts
│
├── api/
│   ├── socket.ts       # Socket.IO client setup, event emitters and listeners.
│   └── httpClient.ts   # For REST API calls (e.g., login, leaderboards).
│
├── assets/
│   ├── fonts/
│   ├── images/
│   └── sounds/
│
├── components/
│   ├── game/         # Components specific to the game screen.
│   │   ├── GameBoard.tsx
│   │   ├── Hand.tsx
│   │   └── Card.tsx
│   └── ui/           # Generic UI components.
│       ├── Button.tsx
│       ├── Modal.tsx
│       └── Spinner.tsx
│
├── config/         # Environment variables and app-wide constants.
│   └── index.ts
│
├── constants/      # Non-configurable game constants (e.g., MAX_PLAYERS).
│   └── gameConstants.ts
│
├── game/           # Core, non-rendering game logic (agnostic to React).
│   ├── GameState.ts
│   ├── Card.ts
│   ├── Player.ts
│   └── GameRules.ts
│
├── hooks/          # Shared custom React hooks.
│   ├── useGameState.ts # Hook to subscribe to game state updates from the store.
│   └── useSocket.ts    # Hook to access the socket instance.
│
├── navigation/     # Navigation logic (e.g., React Navigation stacks).
│   └── AppNavigator.tsx
│
├── platform/       # Platform-specific implementations of shared functionality.
│   ├── Haptics.ts      # Shared interface.
│   ├── web/
│   │   └── Haptics.ts  # Web implementation (might be a no-op).
│   └── native/
│       └── Haptics.ts  # Native implementation using a device API.
│
├── screens/        # Top-level components representing a full screen/view.
│   ├── HomeScreen.tsx
│   ├── LobbyScreen.tsx
│   └── GameScreen.tsx
│
├── state/          # Global state management (Zustand).
│   ├── gameStore.ts  # State slice for the current game match.
│   ├── userStore.ts  # State slice for user authentication and profile.
│   └── index.ts      # Combines stores/reducers.
│
├── styles/         # Global styles, theme definitions, and design tokens.
│   └── theme.ts
│
├── types/          # Shared TypeScript types and interfaces.
│   ├── game.ts       # Types for GameState, Player, Card, etc.
│   └── api.ts        # Types for socket events and API payloads.
│
└── utils/          # Utility and helper functions.
    └── arrayHelpers.ts
```

### Code Structure Best Practices
*   **Monorepo for Shared Code:** For a truly robust system, consider making shared packages (`types/`, `game/`) into a versioned NPM package within a monorepo (using Nx, Turborepo, etc.). Both the client and server would then depend on this package, guaranteeing they use the exact same data structures and game rules.
*   **Pure Game Logic (`src/game/`):** It is critical that code in this directory remains "pure" JavaScript/TypeScript. It must have **zero dependencies on React, React Native, or any UI libraries.** This makes it perfectly portable for use on the server and easy to unit-test.
*   **Constants (`src/constants/`):** This new directory is for game-specific, non-configurable constants like `MAX_PLAYERS = 4` or `ROUND_TIMER_SECONDS = 30`. This separates them from environment-specific configuration and keeps game logic clean.
*   **Unit Testing (`__tests__/`):** The authoritative server's rules engine is the most critical part of the application. The `src/game/` logic should have extensive unit tests to verify its correctness without needing to run the full client-server stack.

## Backend Code Structure: `multiplayer-back`
This structure is designed for a Node.js server using TypeScript. It emphasizes a clear separation of concerns, scalability, and maintainability, directly aligning with the principles outlined in the game plan. It's built to handle HTTP REST APIs, real-time WebSocket communication, and the core authoritative game logic.

The most critical aspect of this design is its reliance on a **shared package** (`@my-game/shared`) for types and pure game logic, which is installed as an npm dependency. This enforces a single source of truth between the client and server.

```
.
├── dist/                     # Compiled JavaScript output from TypeScript.
├── node_modules/             # Project dependencies.
├── src/
│   ├── __tests__/            # Unit and integration tests for the server.
│   │   ├── services/
│   │   │   └── game.service.test.ts
│   │   └── game/
│   │       └── GameManager.test.ts
│   │
│   ├── api/                  # Express REST API layer.
│   │   ├── routes/           # Express router definitions.
│   │   │   ├── auth.routes.ts
│   │   │   └── user.routes.ts
│   │   ├── controllers/      # Route handlers that orchestrate responses.
│   │   │   ├── auth.controller.ts
│   │   │   └── user.controller.ts
│   │   └── middleware/       # Express middleware.
│   │       └── isAuthenticated.ts
│   │
│   ├── config/               # Configuration files and initializers.
│   │   ├── index.ts          # Exports all environment variables.
│   │   ├── database.ts       # Database connection logic (e.g., Prisma, Mongoose).
│   │   └── redis.ts          # Redis client initialization and connection.
│   │
│   ├── game/                 # Server-side game management.
│   │   ├── GameManager.ts    # Singleton/class to manage all active game sessions.
│   │   └── GameSession.ts    # Class representing a single active game instance/room.
│   │
│   ├── models/               # Database schemas/models.
│   │   └── User.ts           # User model definition (e.g., for Prisma or Mongoose).
│   │
│   ├── services/             # Core business logic, decoupled from Express/Socket.IO.
│   │   ├── game.service.ts   # Handles creating, finding, and updating game state in Redis.
│   │   ├── auth.service.ts   # User authentication and account management logic.
│   │   └── socket.service.ts # Manages socket presence and room associations.
│   │
│   ├── socket/               # Socket.IO specific logic.
│   │   ├── index.ts          # Main Socket.IO server setup and event delegation.
│   │   └── handlers/         # Event handlers for different game actions.
│   │       ├── game.handler.ts   # Handles 'PLAY_CARD', 'END_TURN', etc.
│   │       └── lobby.handler.ts  # Handles 'JOIN_LOBBY', 'CREATE_GAME', etc.
│   │
│   ├── utils/                # General utility functions.
│   │   ├── logger.ts
│   │   └── errorHandler.ts
│   │
│   └── server.ts             # Main application entry point.
│
├── .env.example              # Example environment variables.
├── .eslintrc.js              # ESLint configuration.
├── .gitignore
├── Dockerfile                # For containerizing the application.
├── package.json
└── tsconfig.json
```

### Key Concepts Explained

#### `package.json` (Dependency on Shared Code)
This is the cornerstone of the client-server contract. The backend's `package.json` explicitly depends on the shared packages, ensuring type safety and logic consistency.

```json
// package.json
{
  "name": "multiplayer-back",
  "version": "1.0.0",
  "dependencies": {
    "@my-game/shared-types": "1.0.0", // <-- CRITICAL: From the shared-types repo
    "express": "^4.18.2",
    "socket.io": "^4.7.2",
    "redis": "^4.6.10",
    // ... other dependencies like a database ORM, dotenv, etc.
  }
}
```

#### `src/server.ts` - The Entry Point
This file initializes everything. It creates the Express app, attaches the Socket.IO server to it, sets up middleware, registers API routes, and starts listening for connections.

#### `src/api/` vs. `src/socket/` - Separation of Communication
*   **`api/`**: This is for traditional, request-response HTTP communication. It's perfect for actions that don't require real-time updates, such as user login, registration, fetching leaderboards, or viewing player profiles. It follows a standard Model-View-Controller (MVC) like pattern with `routes`, `controllers`, and `middleware`.
*   **`socket/`**: This is the heart of the real-time game. The `index.ts` file configures the main Socket.IO server and acts as a router, delegating incoming events (e.g., `connection`, `disconnect`) to specialized `handlers`. Handlers like `game.handler.ts` contain the logic for what happens when a player sends a `PLAY_CARD` event.

#### `src/services/` - The Brains of the Operation
This is the most important directory for business logic. It is completely decoupled from the transport layers (Express and Socket.IO).
*   `game.service.ts` doesn't know *how* a request came in (HTTP or WebSocket), only that it needs to perform an action like "create a new game" or "apply a player's move to the state stored in Redis."
*   Both `api/controllers/` and `socket/handlers/` will import and use these services. This keeps your code DRY (Don't Repeat Yourself) and makes it easy to test the core logic in isolation.

#### `src/game/` - Server-Side State Orchestration
While the *rules* of the game (e.g., `isCardPlayable(card, gameState)`) come from the shared package, the server needs to manage the *lifecycle* of a game session.
*   **`GameManager.ts`**: A singleton that keeps track of all currently active `GameSession` instances. Its job is to create, find, and destroy sessions.
*   **`GameSession.ts`**: An object representing one game room. It holds the game state (or the ID to fetch it from Redis), manages player connections/disconnections within that room, handles turn timers, and uses the `game.service.ts` to persist state changes.

#### `src/config/` - Connecting to the Outside World
This directory cleanly separates connection and configuration logic.
*   **`redis.ts`**: Exports a connected Redis client instance. The `game.service.ts` will import and use this client to `GET` and `SET` game states. This is the implementation of the "Introduce Redis" solution from the game plan.
*   **`database.ts`**: Exports a connected client for your persistent database (e.g., a Prisma Client or Mongoose connection). The `auth.service.ts` will use this to manage user accounts.
