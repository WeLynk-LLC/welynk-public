# WeLynk Games SDK

> **COMING SOON**
> The WeLynk Games SDK is currently in **closed beta**. We're working hard to bring game development to the WeLynk platform.
> Sign up for our developer waitlist at [welynk.com/developers](https://welynk.com/developers) to be notified when it becomes available.

---

### A Note on This Documentation

This document is shared publicly to demonstrate the engineering effort and thoughtful design behind WeLynk's game platform. We want to give developers a preview of what's possible and the capabilities we're building.

Since the SDK is not yet publicly available, this documentation intentionally omits internal implementation details, infrastructure specifics, and architectural internals to keep WeLynk safe and secure. When the SDK launches, registered developers will receive comprehensive documentation including additional technical details, debugging guides, and full API specifications.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Core Concepts](#core-concepts)
3. [Turn-Based Games](#turn-based-games)
4. [Word/Puzzle Games](#wordpuzzle-games)
5. [Real-Time Games](#real-time-games)
6. [Physics](#physics)
7. [Spatial Utilities](#spatial-utilities)
8. [Audio](#audio)
9. [Monetization](#monetization)
10. [Production Tips](#production-tips)
11. [API Reference](#api-reference)

---

## Introduction

### What is the WeLynk Games SDK?

The WeLynk Games SDK lets you build multiplayer games that run inside the WeLynk app. You write:

- **Server code** (TypeScript) - Runs on WeLynk's servers, handles game logic
- **Client code** (React/TypeScript) - Runs in players' browsers/apps, renders the UI

The SDK handles the complex parts for you:
- Real-time synchronization across all players
- WebSocket networking
- Player matchmaking and lobbies
- Secure, cheat-resistant architecture
- Audio, economy, and ads integration

### Your First Game: Coin Flip

Let's build the simplest possible game - a coin flip. Two players join, and when both are ready, the server picks a random winner.

#### Game Structure

```
your-game/
├── game.config.json     # Game metadata
├── server/
│   └── index.ts         # Server-side game logic
├── client/
│   └── index.tsx        # React client component
└── shared/
    └── types.ts         # Type definitions shared by server & client
```

#### Define Your Types (`shared/types.ts`)

```typescript
// Every game needs to define its State and Actions

// State is what players see - synced to all clients
export interface CoinFlipState {
  phase: 'waiting' | 'flipping' | 'done';
  winnerId: string | null;
  result: 'heads' | 'tails' | null;
}

// Actions are what players send to the server
export type CoinFlipAction =
  | { type: 'ready' };

// Server-only state (never sent to clients)
export interface CoinFlipServerState {
  readyPlayers: Set<string>;
}
```

#### Server Logic (`server/index.ts`)

```typescript
import type { GameContext } from '@welynk/game-sdk';
import type { CoinFlipState, CoinFlipServerState, CoinFlipAction } from '../shared/types';

export default function(ctx: GameContext<CoinFlipState, CoinFlipServerState>) {
  // Called when the game session is created
  ctx.on('init', () => {
    // Set initial public state (sent to all players)
    ctx.setState({
      phase: 'waiting',
      winnerId: null,
      result: null,
    });

    // Set initial server-only state (never sent to clients)
    ctx.setServerState({
      readyPlayers: new Set(),
    });
  });

  // Called when a player sends an action
  ctx.on('action', (playerId: string, action: CoinFlipAction) => {
    if (action.type === 'ready') {
      // Track who's ready
      const ready = ctx.serverState.readyPlayers;
      ready.add(playerId);

      // If both players are ready, flip the coin!
      if (ready.size >= 2) {
        ctx.setState({ phase: 'flipping' });

        // Use ctx.random for secure, cheat-proof randomness
        const result = ctx.random.choice(['heads', 'tails'] as const);
        const winnerId = ctx.random.choice(ctx.players).id;

        // Set the result after a brief delay
        ctx.timer.start('showResult', 2);
        ctx.setServerState({
          ...ctx.serverState,
          pendingResult: { result, winnerId },
        });
      }
    }
  });

  // Called when a timer fires
  ctx.on('timer', (timerId: string) => {
    if (timerId === 'showResult') {
      const { result, winnerId } = ctx.serverState.pendingResult;
      ctx.setState({
        phase: 'done',
        result,
        winnerId,
      });
    }
  });
}
```

#### Client Component (`client/index.tsx`)

```tsx
import React from 'react';
import { useGame } from '@welynk/game-sdk/react';
import type { CoinFlipState, CoinFlipAction } from '../shared/types';

export default function CoinFlip() {
  const { state, sendAction, myPlayerId, players } = useGame<CoinFlipState>();

  const handleReady = () => {
    sendAction({ type: 'ready' });
  };

  // Find the winner's name
  const winner = state.winnerId
    ? players.find(p => p.id === state.winnerId)
    : null;
  const isWinner = state.winnerId === myPlayerId;

  return (
    <div style={{ textAlign: 'center', padding: 40 }}>
      {state.phase === 'waiting' && (
        <>
          <h1>Coin Flip</h1>
          <p>Players: {players.length}/2</p>
          <button onClick={handleReady}>Ready!</button>
        </>
      )}

      {state.phase === 'flipping' && (
        <h1>Flipping...</h1>
      )}

      {state.phase === 'done' && (
        <>
          <h1>Result: {state.result}</h1>
          <h2>{isWinner ? 'You won!' : `${winner?.username} won!`}</h2>
        </>
      )}
    </div>
  );
}
```

#### Game Configuration (`game.config.json`)

```json
{
  "name": "Coin Flip",
  "description": "A simple coin flip game",
  "version": "1.0.0",
  "player_counts": [2],
  "category": "party",
  "visibility": "public",
  "ageGroupSeparation": false,
  "instructions": "Click Ready when both players have joined. The coin will flip!",
  "thumbnail": "thumbnail.png"
}
```

### Key Concept: Server-Authoritative Design

The server decides all game outcomes, not the client. This is crucial for multiplayer fairness - if the client made decisions, cheaters could modify their app to always win. The SDK provides `ctx.random` for secure randomness that clients cannot predict or manipulate.

---

## Core Concepts

### The Game Context (`ctx`)

Every server-side game receives a `GameContext` object. This is your interface to everything the game needs:

```typescript
export default function(ctx: GameContext<MyState, MyServerState>) {
  // ctx.state        - Read the current public state
  // ctx.setState()   - Update public state (synced to all players)
  // ctx.serverState  - Read server-only state
  // ctx.setServerState() - Update server-only state
  // ctx.players      - Array of all players
  // ctx.random       - Secure random number generator
  // ctx.timer        - Timer management
  // ctx.broadcast()  - Send event to all players
  // ctx.sendToPlayer() - Send event to one player
  // ctx.on()         - Register event handlers
}
```

### State: Public vs Server-Only

**Public State (`ctx.state` / `ctx.setState`):**
- Automatically synced to all connected clients
- Players see this in `useGame().state`
- Keep it minimal - only what clients need to render

**Server-Only State (`ctx.serverState` / `ctx.setServerState`):**
- Never sent to clients
- Use for secrets (hidden cards, answers, internal tracking)
- No network cost

```typescript
// Example: A quiz game
interface QuizState {
  currentQuestion: string;        // Public: everyone sees the question
  timeRemaining: number;          // Public: countdown timer
  scores: Record<string, number>; // Public: leaderboard
}

interface QuizServerState {
  correctAnswer: string;          // Secret: only server knows
  questionsRemaining: string[];   // Secret: upcoming questions
}
```

### Actions: Player Input

Players send **actions** to the server using `sendAction()`:

```typescript
// Client
const { sendAction } = useGame();
sendAction({ type: 'guess', word: 'HELLO' });

// Server
ctx.on('action', (playerId: string, action: MyAction) => {
  if (action.type === 'guess') {
    const { word } = action;
    // Validate and process the guess
  }
});
```

**Always validate actions!** Never trust client data:

```typescript
ctx.on('action', (playerId, action) => {
  // Always validate type
  if (action.type !== 'drop') return ctx.reject('Unknown action');

  // Validate data types
  const { column } = action;
  if (typeof column !== 'number') return ctx.reject('Invalid column');

  // Validate ranges
  if (column < 0 || column >= 7) return ctx.reject('Column out of range');

  // Validate game state
  if (state.currentTurn !== playerId) return ctx.reject('Not your turn');

  // Now it's safe to process
});
```

### Events: Server-to-Client Communication

The server can send events to clients:

```typescript
// Server - send to everyone
ctx.broadcast('scoreUpdate', { playerId: 'abc', newScore: 100 });

// Server - send to one player
ctx.sendToPlayer(playerId, 'privateHint', { letter: 'A' });
```

```typescript
// Client - listen for events
import { useGameEvent } from '@welynk/game-sdk/react';

function Game() {
  useGameEvent('scoreUpdate', (data) => {
    playSound('ding');
    showToast(`${data.playerId} scored!`);
  });

  useGameEvent('privateHint', (data) => {
    setMyHint(data.letter);
  });
}
```

**When to use events vs state:**
- **State**: Persistent data that clients need to render (scores, board, whose turn)
- **Events**: Transient notifications (sound triggers, animations, toasts)

### Player Lifecycle

The SDK tracks players joining, leaving, and reconnecting:

```typescript
ctx.on('playerJoin', (playerId: string) => {
  console.log(`${playerId} joined!`);
});

ctx.on('playerReady', (playerId: string) => {
  // Player's client finished loading
});

ctx.on('playerLeave', (playerId: string) => {
  // Handle player removal
});

ctx.on('playerReconnect', (playerId: string) => {
  // They're back! State will auto-sync
});
```

**Access player info:**

```typescript
ctx.players          // All players: Player[]
ctx.getPlayer(id)    // One player: Player | undefined

interface Player {
  id: string;           // Unique player ID
  username: string;     // Display name
  avatar_url?: string;  // Profile picture
  display_name?: string;
}
```

### Timers

Timers are managed by the platform and survive server restarts:

```typescript
// Start a timer
ctx.timer.start('turnTimer', 30);  // Fires in 30 seconds

// Cancel a timer
ctx.timer.cancel('turnTimer');

// Check remaining time
const remaining = ctx.timer.remaining('turnTimer');

// Handle timer events
ctx.on('timer', (timerId: string) => {
  if (timerId === 'turnTimer') {
    advanceToNextPlayer();
  }
});
```

### Randomness

Use `ctx.random` for all randomness - it's secure and unpredictable:

```typescript
ctx.random.int(1, 6)           // Random integer 1-6 (inclusive)
ctx.random.float()             // Random float 0-1
ctx.random.choice(['a', 'b'])  // Random element from array
ctx.random.shuffle([1, 2, 3])  // Shuffled copy of array
```

---

## Turn-Based Games

Turn-based games are the simplest to implement. Here's a complete Tic-Tac-Toe example.

### Tic-Tac-Toe

**`shared/types.ts`:**

```typescript
export type CellValue = 'X' | 'O' | null;
export type Board = CellValue[][];

export interface TicTacToeState {
  board: Board;
  currentTurn: 'X' | 'O';
  players: { X: string | null; O: string | null };
  winner: 'X' | 'O' | 'draw' | null;
}

export type TicTacToeAction =
  | { type: 'place'; row: number; col: number };
```

**`server/index.ts`:**

```typescript
import type { GameContext } from '@welynk/game-sdk';
import { spatial } from '@welynk/game-sdk';
import type { TicTacToeState, TicTacToeAction, CellValue, Board } from '../shared/types';

const { createGrid, setCell, checkWinAt, isFull } = spatial;

export default function(ctx: GameContext<TicTacToeState>) {
  ctx.on('init', () => {
    const shuffled = ctx.random.shuffle([...ctx.players]);

    ctx.setState({
      board: createGrid<CellValue>(3, 3),
      currentTurn: 'X',
      players: {
        X: shuffled[0]?.id ?? null,
        O: shuffled[1]?.id ?? null,
      },
      winner: null,
    });
  });

  ctx.on('action', (playerId: string, action: TicTacToeAction) => {
    if (action.type !== 'place') {
      return ctx.reject('Unknown action');
    }

    const { row, col } = action;
    const state = ctx.state;

    // Validate game not over
    if (state.winner !== null) {
      return ctx.reject('Game is over');
    }

    // Validate it's this player's turn
    const playerMark = state.players.X === playerId ? 'X' :
                       state.players.O === playerId ? 'O' : null;
    if (playerMark !== state.currentTurn) {
      return ctx.reject('Not your turn');
    }

    // Validate cell is empty
    if (state.board[row]?.[col] !== null) {
      return ctx.reject('Cell already occupied');
    }

    // Make the move
    const newBoard = setCell(state.board, { row, col }, playerMark);

    // Check for winner
    const winResult = checkWinAt(newBoard, 3, { row, col });

    if (winResult.winner !== null) {
      ctx.setState({ board: newBoard, winner: playerMark });
      return;
    }

    // Check for draw
    if (isFull(newBoard)) {
      ctx.setState({ board: newBoard, winner: 'draw' });
      return;
    }

    // Continue game
    ctx.setState({
      board: newBoard,
      currentTurn: playerMark === 'X' ? 'O' : 'X',
    });
  });
}
```

### Turn Management Helpers

```typescript
import {
  nextTurn,
  isPlayerTurn,
  getCurrentPlayer,
  setCurrentPlayer,
  getRandomPlayer,
} from '@welynk/game-sdk';

// Check if it's a player's turn
ctx.on('action', (playerId, action) => {
  if (!isPlayerTurn(ctx, playerId)) {
    return ctx.reject('Not your turn');
  }

  // Process action...
  const nextPlayerId = nextTurn(ctx);
  ctx.setState({ currentPlayer: nextPlayerId });
});

// Start with random player
ctx.on('init', () => {
  const firstPlayer = getRandomPlayer(ctx);
  setCurrentPlayer(ctx, firstPlayer?.id ?? null);
});
```

### Turn Timers

```typescript
const TURN_SECONDS = 30;

function startTurn(playerId: string) {
  ctx.setState({
    currentPlayer: playerId,
    turnEndsAt: Date.now() + TURN_SECONDS * 1000,
  });
  ctx.timer.start('turnTimer', TURN_SECONDS);
}

ctx.on('timer', (timerId) => {
  if (timerId === 'turnTimer') {
    // Time ran out - auto-play or skip
    const availableMoves = getAvailableMoves(ctx.state);
    const randomMove = ctx.random.choice(availableMoves);
    applyMove(ctx.state.currentPlayer, randomMove);
  }
});

ctx.on('action', (playerId, action) => {
  // ... validate and process ...
  ctx.timer.cancel('turnTimer');
  startTurn(nextPlayerId);
});
```

---

## Word/Puzzle Games

Word games often need server-side secret state. Here's a Wordle-style example.

### Wordle Example

The key insight: the target word is stored in `serverState` (secret), while guess results are in `state` (public).

```typescript
interface WordleServerState {
  targetWord: string;  // The secret word - never sent to clients!
}

export default function(ctx: GameContext<WordleState, WordleServerState>) {
  ctx.on('init', () => {
    const targetWord = getRandomWord(ctx.random.choice);
    ctx.setServerState({ targetWord });

    ctx.setState({
      phase: 'playing',
      guesses: [],
      wordLength: targetWord.length,
      maxGuesses: 6,
    });
  });

  ctx.on('action', (playerId, action) => {
    if (action.type !== 'submit_guess') return;

    const guess = action.word.toUpperCase().trim();
    const results = evaluateGuess(guess, ctx.serverState.targetWord);

    // Only reveal results, never the target word
    const isCorrect = results.every(r => r === 'correct');

    if (isCorrect) {
      ctx.setState({
        phase: 'ended',
        guesses: [...ctx.state.guesses, { word: guess, results }],
        winnerId: playerId,
        correctWord: ctx.serverState.targetWord, // Reveal only when game ends
      });
    }
  });
}
```

### Per-Player State Filtering

Sometimes different players should see different data:

```typescript
ctx.setStateFilter((state, playerId) => {
  return {
    ...state,
    deck: [],  // Hide the deck from everyone
    hands: {
      // Each player only sees their own hand
      [playerId]: state.hands[playerId] || [],
    },
  };
});
```

---

## Real-Time Games

Real-time games (Pong, drawing games, racing) need continuous updates. The SDK provides a tick system for smooth gameplay.

### How Real-Time Works

1. The server runs a `tick` event loop for game logic
2. State snapshots are automatically sent to clients
3. Clients render smoothly using interpolation

### Pong Example

```typescript
export default function(ctx: GameContext<PongState>) {
  const world = createWorld({ gravity: 'none' });
  let ball: GameObject;

  ctx.on('init', () => {
    ball = world.addCircle({
      name: 'ball',
      x: 400, y: 300,
      radius: 10,
      bouncy: true,
      fast: true,
    });

    ctx.setState({
      phase: 'waiting',
      ball: { x: 400, y: 300 },
      paddles: { left: 0.5, right: 0.5 },
      scores: { left: 0, right: 0 },
    });
  });

  ctx.on('tick', (deltaTime) => {
    if (ctx.state.phase !== 'playing') return;

    world.step(deltaTime);

    ctx.setState({
      ball: { x: ball.x, y: ball.y },
      tick: ctx.tickNumber,
    });
  });
}
```

### Smooth Rendering (Client)

Use the `<Smooth>` component for smooth rendering:

```tsx
import { useGame, Smooth } from '@welynk/game-sdk/react';

function PongGame() {
  const { state } = useGame<PongState>();

  return (
    <div className="arena">
      <Smooth select={(s: PongState) => ({ x: s.ball.x, y: s.ball.y })}>
        <div className="ball" />
      </Smooth>
    </div>
  );
}
```

### Optimistic Updates

For instant local response:

```tsx
const [paddlePos, setPaddlePos, clearPos] = useOptimisticValue(
  myPaddle?.position ?? 0.5,
  { clearAfterMs: 100 }
);

const handlePointerMove = (e: React.PointerEvent) => {
  const pos = (e.clientY - rect.top) / rect.height;
  setPaddlePos(pos);  // Instant local update
  sendAction({ type: 'setPaddlePosition', position: pos });
};
```

### Input Mode Configuration

```json
{
  "runtime": {
    "inputMode": "streaming"
  }
}
```

| Mode | Use Case |
|------|----------|
| `streaming` | Continuous input (Pong, drawing, racing) |
| `discrete` | Click/tap actions (cards, word games, trivia) |

---

## Physics

The SDK includes a beginner-friendly physics API.

### Creating a Physics World

```typescript
import { createWorld } from '@welynk/game-sdk';

const world = createWorld({ gravity: 'none' });    // Top-down games
const world = createWorld({ gravity: 'normal' });  // Platformers
const world = createWorld({ gravity: 'light' });   // Floaty games
```

### Adding Objects

```typescript
// Circles
const ball = world.addCircle({
  name: 'ball',
  x: 100, y: 100,
  radius: 10,
  bouncy: true,
  movable: true,
  fast: true,       // Enable for fast objects
});

// Rectangles
const paddle = world.addRectangle({
  name: 'paddle',
  x: 50, y: 300,
  width: 80, height: 10,
  movable: false,
  bouncy: true,
});

// Walls
world.addWall({
  name: 'topWall',
  x1: 0, y1: 0,
  x2: 800, y2: 0,
});
```

### Collision Detection

```typescript
world.onCollision((objectA, objectB, info) => {
  console.log(`${objectA.name} hit ${objectB.name}`);

  if (objectA.name === 'ball' && objectB.name === 'paddle') {
    ctx.broadcast('hit', { point: info.point });
  }
});
```

### Moving Objects

```typescript
ball.move(200, 'right');
ball.moveTo(100, 100);
ball.setVelocity(100, -50);
ball.stop();
ball.destroy();
```

### Stepping the Simulation

```typescript
ctx.on('tick', (deltaTime) => {
  world.step(deltaTime);
  ctx.setState({ ball: { x: ball.x, y: ball.y } });
});
```

---

## Spatial Utilities

Grid-based utilities for board games.

### Creating Grids

```typescript
import { spatial } from '@welynk/game-sdk';

const board = spatial.createGrid<string>(3, 3);  // 3x3 grid
const copy = spatial.cloneGrid(board);
```

### Cell Operations

```typescript
const value = spatial.getCell(grid, { row: 1, col: 2 });
const newGrid = spatial.setCell(grid, { row: 1, col: 2 }, 'X');
const isEmpty = spatial.isEmpty(grid, { row: 0, col: 0 });
```

### Win Detection

```typescript
const result = spatial.checkWinAt(grid, 3, { row: 1, col: 1 });
const hasXWon = spatial.hasWon(grid, 'X', 4);
```

### Connect Four Style Drops

```typescript
const result = spatial.dropInDirection(grid, { row: 0, col: 3 }, 'down', 'R');
const columns = spatial.getAvailablePositions(grid, 'down');
```

### Pathfinding

```typescript
const path = spatial.findPath(grid, { row: 0, col: 0 }, { row: 5, col: 5 }, {
  diagonal: true,
  isBlocked: (cell) => cell === 'wall',
});
```

---

## Audio

### Declaring Audio Assets

In `game.config.json`:

```json
{
  "assets": {
    "audio": {
      "music": {
        "background": "assets/music/background.mp3"
      },
      "sfx": {
        "hit": "assets/sounds/hit.wav",
        "score": "assets/sounds/score.wav"
      }
    }
  }
}
```

### Playing Music

```tsx
import { useMusic } from '@welynk/game-sdk/react';

function Game() {
  const music = useMusic('background');

  useEffect(() => {
    music.play();
    return () => music.stop();
  }, []);

  return <button onClick={() => music.pause()}>Pause</button>;
}
```

### Playing Sound Effects

```tsx
import { useAudio } from '@welynk/game-sdk/react';

function Game() {
  const { play } = useAudio('sfx');

  const handleClick = () => {
    play('hit');
  };

  return <button onClick={handleClick}>Click me</button>;
}
```

---

## Monetization

### Lynks (In-Game Economy)

Lynks are WeLynk's platform currency:

```typescript
// Request a purchase
ctx.lynks.requestPurchase(playerId, 10, 'Power-up Pack');

// Handle purchase result
ctx.lynks.onPurchaseResult(async (result) => {
  if (result.success) {
    const verification = await ctx.lynks.verifyReceipt(result.receipt!, result.playerId);
    if (verification.valid) {
      grantPowerUp(result.playerId);
    }
  }
});

// Check balance
const balance = await ctx.lynks.checkBalance(playerId, 10);
```

### Rewarded Ads

```typescript
const result = await ctx.ads.showRewardedAd(playerId, '50 coins');
if (result.success && result.rewarded) {
  grantCoins(playerId, 50);
}
```

### Player Data (Persistent Storage)

```typescript
await ctx.playerData.set(playerId, 'high_score', 1000);
const highScore = await ctx.playerData.get(playerId, 'high_score');
const newScore = await ctx.playerData.increment(playerId, 'wins', 1);
```

---

## Production Tips

### Error Handling

```typescript
ctx.on('action', async (playerId, action) => {
  try {
    await processAction(playerId, action);
  } catch (error) {
    ctx.sendToPlayer(playerId, 'error', {
      message: 'Something went wrong. Please try again.',
    });
  }
});
```

### Safe Area

Handle mobile notches and system UI:

```tsx
import { useSafeArea } from '@welynk/game-sdk/react';

function Game() {
  const safeArea = useSafeArea();

  return (
    <div style={{
      paddingTop: safeArea.top,
      paddingBottom: safeArea.bottom,
    }}>
      {/* Game content */}
    </div>
  );
}
```

### Lobby Controls

```typescript
ctx.lobby.closeRoom();   // Stop accepting players
ctx.lobby.openRoom();    // Accept new players
ctx.lobby.isClosed();    // Check status
```

### Cleanup

```typescript
ctx.on('cleanup', async (signal) => {
  for (const playerId of Object.keys(gameData.playerStats)) {
    await ctx.playerData.set(playerId, 'stats', gameData.playerStats[playerId]);
  }
  signal.done();
});
```

### Performance Tips

1. **Minimize state size** - Only include what clients need to render
2. **Use serverState for secrets** - No network cost
3. **Batch state updates** - Call `setState` once with all changes
4. **Use events for transient data** - Animations, sounds don't need state

---

## API Reference

### Server API (`GameContext`)

#### State Management
```typescript
ctx.state                           // Current public state
ctx.setState({ key: value })        // Update public state
ctx.serverState                     // Server-only state
ctx.setServerState({ key: value })  // Update server-only state
ctx.setStateFilter((state, playerId) => filteredState)
```

#### Players
```typescript
ctx.players                  // All players: Player[]
ctx.getPlayer(playerId)      // Get player by ID
```

#### Communication
```typescript
ctx.broadcast(event, data)
ctx.sendToPlayer(playerId, event, data)
ctx.reject(reason)
```

#### Timer
```typescript
ctx.timer.start(id, seconds)
ctx.timer.cancel(id)
ctx.timer.remaining(id)
```

#### Random
```typescript
ctx.random.int(min, max)
ctx.random.float()
ctx.random.choice(array)
ctx.random.shuffle(array)
```

#### Events
```typescript
ctx.on('init', () => {})
ctx.on('action', (playerId, action) => {})
ctx.on('playerJoin', (playerId) => {})
ctx.on('playerLeave', (playerId) => {})
ctx.on('playerReconnect', (playerId) => {})
ctx.on('timer', (timerId) => {})
ctx.on('tick', (deltaTime, tickNumber) => {})
ctx.on('cleanup', (signal) => { signal.done(); })
```

### Client API

#### Core Hooks
```typescript
const {
  state,
  sendAction,
  requestLeave,
  myPlayerId,
  players,
  gameStatus,
} = useGame<MyState>();

useGameEvent('eventName', (data) => { ... });

const { top, right, bottom, left } = useSafeArea();
```

#### Smooth Rendering
```tsx
<Smooth select={(s) => ({ x: s.ball.x, y: s.ball.y })}>
  <div className="ball" />
</Smooth>

useGameLoop((time, delta) => { ... });
```

#### Audio
```typescript
const music = useMusic('trackId');
music.play();
music.pause();
music.stop();

const { play } = useAudio('sfx');
play('soundId');
```

### game.config.json Schema

```json
{
  "name": "Game Name",
  "description": "Short description",
  "version": "1.0.0",
  "player_counts": [2, 4],
  "category": "board | card | word | drawing | action | social | party | puzzle | trivia | others",
  "visibility": "public | private",
  "ageGroupSeparation": true,
  "instructions": "How to play",
  "thumbnail": "thumbnail.png",
  "tags": ["multiplayer", "strategy"],
  "estimated_duration": "5-10 min",
  "options": {
    "difficulty": {
      "type": "select",
      "label": "Difficulty",
      "default": "normal",
      "choices": [
        { "value": "easy", "label": "Easy" },
        { "value": "normal", "label": "Normal" }
      ]
    }
  },
  "runtime": {
    "inputMode": "discrete | streaming"
  },
  "assets": {
    "audio": {
      "music": { "trackId": "path/to/music.mp3" },
      "sfx": { "soundId": "path/to/sound.wav" }
    }
  }
}
```

---

## Security Best Practices

1. **Never trust client data** - Always validate actions on the server
2. **Use serverState for secrets** - Cards, answers, hidden info
3. **Verify receipts** - Always verify Lynks purchases before granting rewards
4. **Don't expose internal errors** - Return generic error messages to clients

---

*Last updated: February 2026*
