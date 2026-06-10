# Genshin Impact Character Guessing Game - Implementation Plan

## Context

Build a fully offline Genshin Impact character guessing game where:
1. Players select characters (text-only name list or import friend's name list file)
2. Splash art images are shuffled and displayed; player matches each image to a name via dropdown
3. Results show correct/wrong with green/red text

Target: Android APK, Genshin-themed UI, fully offline.

## Tech Stack

- **React 18 + TypeScript + Vite** — web app framework
- **CapacitorJS 6** — wraps web app as native Android APK
- **CSS Modules + CSS Variables** — styling (themable for Genshin look)
- **React Context + useReducer** — state management (no external library needed)

### Prerequisites for building APK
- Node.js 18+

## Project Structure

```
genshin-guess-game/
├── public/
│   └── images/
│       └── characters/          # User places splash art PNGs here
│           ├── placeholder.png
│           ├── hutao.png
│           └── ...
├── src/
│   ├── assets/
│   │   └── fonts/               # Custom Genshin fonts (user-provided)
│   ├── components/
│   │   ├── SelectionScreen/     # Screen 1: character selection
│   │   │   ├── SelectionScreen.tsx
│   │   │   ├── CharacterGrid.tsx
│   │   │   └── ImportExportBar.tsx
│   │   ├── QuizScreen/          # Screen 2: quiz/guessing
│   │   │   ├── QuizScreen.tsx
│   │   │   ├── SplashArtViewer.tsx
│   │   │   ├── AnswerPanel.tsx
│   │   │   ├── CharacterDropdown.tsx
│   │   │   ├── CommentBox.tsx
│   │   │   └── QuestionNavigator.tsx
│   │   ├── ResultScreen/        # Screen 3: results
│   │   │   ├── ResultScreen.tsx
│   │   │   └── ResultRow.tsx
│   │   └── common/
│   │       └── GenshinFrame.tsx # Shared themed wrapper
│   ├── store/
│   │   ├── GameContext.tsx
│   │   ├── gameReducer.ts
│   │   └── types.ts
│   ├── data/
│   │   └── characters.ts       # Built-in character database
│   ├── utils/
│   │   ├── shuffle.ts          # Fisher-Yates shuffle
│   │   └── imagePath.ts        # Character ID → image path
│   ├── App.tsx
│   ├── main.tsx
│   └── global.css
├── capacitor.config.ts
├── index.html
├── package.json
├── tsconfig.json
└── vite.config.ts
```

## Component Tree & Layout

```
App
└── GameProvider (Context + useReducer)
    ├── [Phase: selection] SelectionScreen
    │   ├── CharacterGrid        # Grid of name-only toggleable cards
    │   ├── ImportExportBar      # Import/export .json name list
    │   └── "Start Quiz" button  # Disabled if < 2 selected
    │
    ├── [Phase: quiz] QuizScreen
    │   ├── SplashArtViewer      # Left ~2/3: character splash art image
    │   ├── AnswerPanel          # Right ~1/3, vertical stack:
    │   │   ├── CharacterDropdown # Dropdown of SELECTED character names only
    │   │   └── CommentBox        # Textarea for reasoning
    │   ├── QuestionNavigator    # Bottom: numbered circles (clickable, shows answered state)
    │   └── "Complete" button    # Appears only when ALL questions answered
    │
    └── [Phase: results] ResultScreen
        ├── SplashArtViewer      # Same left ~2/3 layout
        ├── ResultRow            # Fixed text (green=correct / red=wrong + correct answer)
        ├── CommentDisplay       # Shows user's comment
        ├── QuestionNavigator    # Bottom: review navigation
        └── "Play Again" button
```

## State Design (`src/store/types.ts`)

```typescript
interface Character {
  id: string;    // e.g., "hutao" — also used as image filename
  name: string;  // e.g., "胡桃"
}

interface Answer {
  guessedId: string | null;  // null = not yet answered
  comment: string;
}

interface GameState {
  phase: 'selection' | 'quiz' | 'results';
  allCharacters: Character[];           // Built-in full roster (~90 characters)
  selectedIds: string[];                // User's chosen subset
  quizOrder: string[];                  // Shuffled character IDs (question order)
  currentIndex: number;                 // Current question index
  answers: Record<string, Answer>;      // keyed by character ID in quizOrder
}
```

### Actions

- `TOGGLE_CHARACTER { id }` — toggle character in/out of selection
- `IMPORT_LIST { ids: string[] }` — replace selection with imported list
- `START_QUIZ` — shuffle selected, enter quiz phase, init empty answers
- `GO_TO_QUESTION { index }` — navigate to any question
- `SET_ANSWER { id, guessedId, comment }` — record guess and comment
- `COMPLETE_QUIZ` — transition to results phase
- `RESET` — back to selection (preserves selection)

## Key Design Decisions

1. **Dropdown only shows selected characters** (per user requirement) — NOT the full character pool
2. **Question navigation is direct-access number buttons** — user can jump to any question at any time, not just prev/next
3. **"Complete" button only visible when all questions have answered** — checks `Object.values(answers).every(a => a.guessedId !== null)`
4. **Results dropdown replaced by fixed text** — green text if `guessedId === actualId`, red text + correct answer shown below if wrong
5. **Name list file format: JSON**
   ```json
   {
     "version": 1,
     "characters": ["hutao", "zhongli", "ganyu"]
   }
   ```
   Simple array of character IDs. One ID per friend's chosen subset.

## Implementation Sequence

### Step 1: Project Scaffolding
- `npm create vite@latest` with react-ts template
- Install Capacitor: `@capacitor/core @capacitor/cli @capacitor/filesystem @capacitor/android`
- Configure `vite.config.ts` (base: './'), `capacitor.config.ts`
- Create directory structure, placeholder image, `.gitignore`

### Step 2: Data & State Layer
- `src/data/characters.ts` — full character database (all Genshin 5.x characters with id + Chinese name)
- `src/store/types.ts` — all types and action interfaces
- `src/store/gameReducer.ts` — /
- `src/store/GameContext.tsx` — Context provider + custom hooks (`useGame`, `useGameDispatch`)
- `src/utils/shuffle.ts`, `src/utils/imagePath.ts`

### Step 3: Selection Screen
- `CharacterGrid` — grid of cards showing character names, toggleable (selected/unselected visual state)
- `ImportExportBar` — import .json file (file picker), export current selection as .json
- "Start Quiz" button — disabled until ≥2 characters selected
- Wire to TOGGLE_CHARACTER, IMPORT_LIST, START_QUIZ

### Step 4: Quiz Screen (most complex)
- `SplashArtViewer` — displays current question's splash art (left 2/3 of screen), error fallback to placeholder
- `CharacterDropdown` — `<select>` dropdown with only SELECTED character names, shows current answer if set
- `CommentBox` — `<textarea>` for free-text reasoning
- `QuestionNavigator` — row of numbered circles at bottom, clickable to jump to any question; visual states: current (highlighted), answered (checkmark/color), unanswered (dimmed)
- "Complete" button — conditionally rendered when all questions answered
- Wire to GO_TO_QUESTION, SET_ANSWER, COMPLETE_QUIZ

### Step 5: Results Screen
- Reuses `SplashArtViewer` and `QuestionNavigator` layout
- `ResultRow` — fixed text box showing user's guess in green (`#4CAF50`) or red (`#E57373`); if wrong, correct answer displayed below in white/gold
- Shows user's comment below
- Score summary (e.g., "3/5 correct")
- "Play Again" button → RESET

### Step 6: Genshin Impact Theming
- Apply CSS variables for Genshin color palette (gold `#D4A843`, dark bg, cream accents)
- Import and apply user-provided custom fonts via `@font-face`
- Style cards, buttons, dropdown, navigator to match Genshin's ornate UI style
- Add decorative border elements (gold frames, star/constellation motifs) if user provides assets
- Background: dark gradient or user-provided background image


## Verification Plan

1. **Dev mode**: `npm run dev` → open in browser, test all 3 screens
2. **Selection test**: Click characters to toggle, verify count updates, import/export a .json name list
3. **Quiz test**: Start quiz with 5 characters, verify shuffle, answer each question, navigate via number buttons, verify "Complete" button appears only after all answered
4. **Results test**: Submit with some wrong answers, verify green/red color coding, verify correct answer shown for wrong guesses
