<script lang="ts">
  type Card = {
    id: number;
    name: string;
    flipped: boolean;
    matched: boolean;
  };

  type GameMode = 'menu' | 'single' | 'multi';
  type Theme = 'dark' | 'light';
  type Difficulty = 'easy' | 'medium' | 'hard' | 'expert';
  type MenuStep = 'mode' | 'difficulty';
  type CardStyle = 'classic' | 'dots' | 'waves' | 'stripes';

  const cardStyleOrder: CardStyle[] = ['classic', 'dots', 'waves', 'stripes'];
  const cardStyleLabels: Record<CardStyle, string> = {
    classic: 'Klassisch',
    dots: 'Punkte',
    waves: 'Wellen',
    stripes: 'Streifen',
  };

  type CardSet = 'food' | 'animals' | 'space' | 'symbols';
  const cardSetOrder: CardSet[] = ['food', 'animals', 'space', 'symbols'];
  const cardSetLabels: Record<CardSet, string> = {
    food: 'Essen',
    animals: 'Tiere',
    space: 'Weltraum',
    symbols: 'Symbole',
  };
  const cardSetEmojis: Record<CardSet, string[]> = {
    food: ['🍎','🍌','🍕','🍔','🍩','🍪','🍰','🍓','🍇','🥐','🧀','🌮','🥗','🍜','🍦','🧁','🍫','🍿','🥑','🍒','🥨','🌭','🥪','🧇'],
    animals: ['🐱','🐶','🐰','🐼','🦊','🐙','🦋','🐢','🦄','🐝','🦁','🐸','🐧','🐨','🦉','🐬','🦓','🐘','🦒','🦏','🦈','🦂','🦖','🦔'],
    space: ['🚀','🌟','⭐','🌙','☄️','🪐','🌌','🌠','🌍','🌕','🌞','🛸','👽','🔭','🛰️','🌑','🌖','🌗','🌘','💫','🌑','✨','🌟','☄️'],
    symbols: ['💎','🎯','🍀','🔥','⚡','🌈','👑','🎵','🎲','🎈','🎨','🔮','⚔️','🛡️','🎭','🗝️','🏆','💖','✨','🌀','💠','🎪','🔔','🌟'],
  };

  const difficultyConfig: Record<Difficulty, {
    pairs: number; cols: number; label: string; icon: string; desc: string;
    aiForget: number; aiMistake: number; aiMisremember: number;
  }> = {
    easy:   { pairs: 8,  cols: 4, label: 'Leicht',   icon: '🌱', desc: '8 Paare',  aiForget: 0.5,  aiMistake: 0.4,  aiMisremember: 0.25 },
    medium: { pairs: 12, cols: 6, label: 'Mittel',   icon: '⚡', desc: '12 Paare', aiForget: 0.35, aiMistake: 0.3,  aiMisremember: 0.2  },
    hard:   { pairs: 18, cols: 6, label: 'Schwer',   icon: '🔥', desc: '18 Paare', aiForget: 0.25, aiMistake: 0.2,  aiMisremember: 0.15 },
    expert: { pairs: 24, cols: 8, label: 'Experte', icon: '💀', desc: '24 Paare', aiForget: 0.15, aiMistake: 0.1,  aiMisremember: 0.1  },
  };

  function createCards(pairCount: number = 12): Card[] {
    const pool = cardSetEmojis[cardSet];
    const selected = pool.slice(0, pairCount);
    const duplicated = [...selected, ...selected];
    return duplicated
      .map((name, index) => ({ id: index, name, flipped: false, matched: false }))
      .sort(() => Math.random() - 0.5);
  }

  const STORAGE_KEY = 'memory-game-state';
  const THEME_KEY = 'memory-game-theme';
  const CARDSTYLE_KEY = 'memory-game-cardstyle';
  const CARDSET_KEY = 'memory-game-cardset';

  type SavedState = {
    gameMode: GameMode;
    difficulty: Difficulty;
    cards: Card[];
    currentPlayer: number;
    scores: Record<number, number>;
    gameOver: boolean;
    winner: string;
    moves: number;
    aiMemory: [number, string][];
  };

  function loadState(): SavedState | null {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (!raw) return null;
      const parsed = JSON.parse(raw) as SavedState;
      if (!parsed.gameMode || !Array.isArray(parsed.cards)) return null;
      return parsed;
    } catch {
      return null;
    }
  }

  function saveState() {
    if (gameMode === 'menu' || gameOver) return;
    const state: SavedState = {
      gameMode,
      difficulty,
      cards: cards.map((c) => ({ ...c, flipped: false })),
      currentPlayer,
      scores: { ...scores },
      gameOver,
      winner,
      moves,
      aiMemory: Array.from(aiMemory.entries()),
    };
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    } catch {
      // ignore
    }
  }

  function clearState() {
    try {
      localStorage.removeItem(STORAGE_KEY);
    } catch {
      // ignore
    }
  }

  function loadTheme(): Theme {
    try {
      const t = localStorage.getItem(THEME_KEY);
      if (t === 'light' || t === 'dark') return t;
    } catch {
      // ignore
    }
    return 'dark';
  }

  function loadCardStyle(): CardStyle {
    try {
      const s = localStorage.getItem(CARDSTYLE_KEY) as CardStyle | null;
      if (s && cardStyleOrder.includes(s)) return s;
    } catch {
      // ignore
    }
    return 'classic';
  }

  function loadCardSet(): CardSet {
    try {
      const s = localStorage.getItem(CARDSET_KEY) as CardSet | null;
      if (s && cardSetOrder.includes(s)) return s;
    } catch {
      // ignore
    }
    return 'food';
  }

  const saved = loadState();

  let gameMode = $state<GameMode>(saved?.gameMode ?? 'menu');
  let difficulty = $state<Difficulty>(saved?.difficulty ?? 'medium');
  let menuStep = $state<MenuStep>('mode');
  let pendingMode = $state<GameMode>('menu');
  let cards = $state<Card[]>(saved?.cards ?? createCards(difficultyConfig[difficulty].pairs));
  let selectedCards = $state<Card[]>([]);
  let isChecking = $state(false);

  let currentPlayer = $state(saved?.currentPlayer ?? 1);
  let scores = $state<Record<number, number>>(saved?.scores ?? { 1: 0, 2: 0 });
  let gameOver = $state(saved?.gameOver ?? false);
  let winner = $state<string>(saved?.winner ?? '');
  let moves = $state(saved?.moves ?? 0);

  let theme = $state<Theme>(loadTheme());
  let cardStyle = $state<CardStyle>(loadCardStyle());
  let cardSet = $state<CardSet>(loadCardSet());

  function toggleTheme() {
    theme = theme === 'dark' ? 'light' : 'dark';
    try {
      localStorage.setItem(THEME_KEY, theme);
    } catch {
      // ignore
    }
  }

  function toggleCardStyle() {
    const idx = cardStyleOrder.indexOf(cardStyle);
    cardStyle = cardStyleOrder[(idx + 1) % cardStyleOrder.length];
    try {
      localStorage.setItem(CARDSTYLE_KEY, cardStyle);
    } catch {
      // ignore
    }
  }

  function toggleCardSet() {
    const idx = cardSetOrder.indexOf(cardSet);
    cardSet = cardSetOrder[(idx + 1) % cardSetOrder.length];
    if (gameMode !== 'menu') restartGame();
    try {
      localStorage.setItem(CARDSET_KEY, cardSet);
    } catch {
      // ignore
    }
  }

  // Computer memory: map of card id -> emoji name for cards it has seen
  let aiMemory = $state<Map<number, string>>(
    saved ? new Map(saved.aiMemory) : new Map()
  );

  let player1Label = $derived(gameMode === 'single' ? 'Du' : 'Spieler 1');
  let player2Label = $derived(gameMode === 'single' ? 'Computer' : 'Spieler 2');

  let aiForgetChance = $derived(difficultyConfig[difficulty].aiForget);
  let aiMisrememberChance = $derived(difficultyConfig[difficulty].aiMisremember);
  let aiMistakeChance = $derived(difficultyConfig[difficulty].aiMistake);

  function startGame(mode: GameMode) {
    gameMode = mode;
    menuStep = 'mode';
    restartGame();
  }

  function selectMode(mode: GameMode) {
    pendingMode = mode;
    menuStep = 'difficulty';
  }

  function startWithDifficulty(d: Difficulty) {
    difficulty = d;
    startGame(pendingMode);
  }

  function flipCard(card: Card) {
    if (isChecking || card.flipped || card.matched) return;

    card.flipped = true;
    selectedCards = [...selectedCards, card];

    // Computer learns every card that gets flipped, but sometimes forgets or misremembers
    if (gameMode === 'single') {
      const roll = Math.random();
      if (roll < aiForgetChance) {
        // forget it entirely
      } else if (roll < aiForgetChance + aiMisrememberChance) {
        aiMemory.set(card.id, cards[Math.floor(Math.random() * cards.length)].name);
        aiMemory = new Map(aiMemory);
      } else {
        aiMemory.set(card.id, card.name);
        aiMemory = new Map(aiMemory);
      }
    }

    if (selectedCards.length === 2) {
      isChecking = true;
      moves++;

      if (selectedCards[0].name === selectedCards[1].name) {
        selectedCards.forEach((c) => (c.matched = true));
        scores[currentPlayer]++;
        selectedCards = [];
        isChecking = false;

        if (cards.every((c) => c.matched)) {
          gameOver = true;
          winner =
            scores[1] > scores[2]
              ? player1Label
              : scores[1] < scores[2]
                ? player2Label
                : 'Unentschieden';
        } else if (gameMode === 'single' && currentPlayer === 2) {
          // Computer matched — goes again
          setTimeout(() => computerTurn(), 500);
        }
      } else {
        setTimeout(() => {
          selectedCards.forEach((c) => (c.flipped = false));
          selectedCards = [];
          currentPlayer = currentPlayer === 1 ? 2 : 1;
          isChecking = false;

          if (gameMode === 'single' && currentPlayer === 2 && !gameOver) {
            setTimeout(() => computerTurn(), 500);
          }
        }, 800);
      }
    }
  }

  function computerTurn() {
    if (gameOver || currentPlayer !== 2) return;

    const unmatched = cards.filter((c) => !c.matched && !c.flipped);
    if (unmatched.length < 2) return;

    // Occasionally lose a random remembered card between turns
    if (aiMemory.size > 0 && Math.random() < aiForgetChance) {
      const keys = Array.from(aiMemory.keys());
      aiMemory.delete(keys[Math.floor(Math.random() * keys.length)]);
      aiMemory = new Map(aiMemory);
    }

    // Group all known unmatched cards by emoji
    const knownByEmoji = new Map<string, Card[]>();
    for (const c of unmatched) {
      const emoji = aiMemory.get(c.id);
      if (!emoji) continue;
      if (!knownByEmoji.has(emoji)) knownByEmoji.set(emoji, []);
      knownByEmoji.get(emoji)!.push(c);
    }

    // Find a guaranteed pair from memory
    let knownPair: [Card, Card] | null = null;
    for (const group of knownByEmoji.values()) {
      if (group.length >= 2) {
        knownPair = [group[0], group[1]];
        break;
      }
    }

    // Sometimes the AI ignores a known pair and plays randomly (makes a mistake)
    if (knownPair && Math.random() < aiMistakeChance) {
      knownPair = null;
    }

    if (knownPair) {
      flipCard(knownPair[0]);
      setTimeout(() => flipCard(knownPair![1]), 450);
      return;
    }

    // No known pair (or chose to ignore one) — flip an unknown card first
    const unknownCards = unmatched.filter((c) => !aiMemory.has(c.id));
    const firstCard = unknownCards.length > 0
      ? unknownCards[Math.floor(Math.random() * unknownCards.length)]
      : unmatched[Math.floor(Math.random() * unmatched.length)];

    flipCard(firstCard);

    // After learning the first card, check if we now know its match
    const firstEmoji = aiMemory.get(firstCard.id);
    const remaining = cards.filter((c) => !c.matched && !c.flipped && c.id !== firstCard.id);
    const knownMatch = firstEmoji
      ? remaining.find((c) => aiMemory.get(c.id) === firstEmoji)
      : undefined;
    // Sometimes ignore a known match after first flip too
    const useKnownMatch = knownMatch && Math.random() >= aiMistakeChance;
    const secondCard = useKnownMatch
      ? knownMatch!
      : remaining[Math.floor(Math.random() * remaining.length)];

    setTimeout(() => flipCard(secondCard), 450);
  }

  function restartGame() {
    cards = createCards(difficultyConfig[difficulty].pairs);
    selectedCards = [];
    currentPlayer = 1;
    scores = { 1: 0, 2: 0 };
    isChecking = false;
    gameOver = false;
    winner = '';
    moves = 0;
    aiMemory = new Map();
    clearState();
  }

  function backToMenu() {
    gameMode = 'menu';
    menuStep = 'mode';
    restartGame();
  }

  let matchedCount = $derived(cards.filter((c) => c.matched).length);
  let totalPairs = $derived(difficultyConfig[difficulty].pairs);
  let gridCols = $derived(difficultyConfig[difficulty].cols);
  let difficultyLabel = $derived(difficultyConfig[difficulty].label);
  let isComputerTurn = $derived(gameMode === 'single' && currentPlayer === 2 && !gameOver && !isChecking);

  const confettiEmojis = ['🎉', '🎊', '✨', '⭐', '🌟', '💫', '🏆', '🥳'];
  const confettiColors = ['#6ee7b7', '#34d399', '#60a5fa', '#2563eb', '#fbbf24', '#f59e0b', '#f472b6', '#e879f9'];

  type Confetti = { id: number; x: number; delay: number; duration: number; rotate: number; size: number; color: string; emoji: string | null; };

  let confettiPieces = $state<Confetti[]>([]);

  function launchConfetti() {
    const pieces: Confetti[] = [];
    for (let i = 0; i < 80; i++) {
      const useEmoji = Math.random() > 0.5;
      pieces.push({
        id: i,
        x: Math.random() * 100,
        delay: Math.random() * 0.5,
        duration: 2 + Math.random() * 2,
        rotate: Math.random() * 720 - 360,
        size: 8 + Math.random() * 12,
        color: confettiColors[Math.floor(Math.random() * confettiColors.length)],
        emoji: useEmoji ? confettiEmojis[Math.floor(Math.random() * confettiEmojis.length)] : null,
      });
    }
    confettiPieces = pieces;
  }

  $effect(() => {
    if (gameOver) {
      launchConfetti();
      clearState();
    } else {
      confettiPieces = [];
    }
  });

  // Persist game state to localStorage whenever it changes
  $effect(() => {
    saveState();
  });

  // If we restored a single-player game and it's the computer's turn, resume
  $effect(() => {
    if (
      gameMode === 'single' &&
      currentPlayer === 2 &&
      !gameOver &&
      !isChecking &&
      selectedCards.length === 0
    ) {
      const timer = setTimeout(() => computerTurn(), 600);
      return () => clearTimeout(timer);
    }
  });
</script>

<div class="app" data-card-style={cardStyle} class:theme-dark={theme === 'dark'} class:theme-light={theme === 'light'}>
  <div class="toggle-group">
    <button class="theme-toggle" onclick={toggleTheme} aria-label="Design umschalten" title="Design umschalten">
      <span class="theme-icon" class:show-sun={theme === 'dark'}>
        <span class="icon-sun">☀️</span>
        <span class="icon-moon">🌙</span>
      </span>
    </button>
    <button class="style-toggle" onclick={toggleCardStyle} aria-label="Kartendesign umschalten" title="Kartendesign: {cardStyleLabels[cardStyle]}">
      <span class="style-toggle-inner">🎴</span>
      <span class="style-toggle-label">{cardStyleLabels[cardStyle]}</span>
    </button>
    <button class="style-toggle" onclick={toggleCardSet} aria-label="Kartensymbole umschalten" title="Symbole: {cardSetLabels[cardSet]}">
      <span class="style-toggle-inner">🎨</span>
      <span class="style-toggle-label">{cardSetLabels[cardSet]}</span>
    </button>
  </div>

  <div class="bg-orb orb-1"></div>
  <div class="bg-orb orb-2"></div>

{#if gameMode === 'menu'}
  <div class="menu-container">
    <header class="menu-header">
      <h1 class="title">Memory</h1>
      <p class="subtitle">Finde alle Paare</p>
    </header>

    {#if menuStep === 'mode'}
      <div class="menu-cards">
        <button class="menu-card single" onclick={() => selectMode('single')}>
          <div class="menu-icon">🤖</div>
          <h2 class="menu-title">Einzelspieler</h2>
          <p class="menu-desc">Spiele gegen den Computer</p>
        </button>

        <button class="menu-card multi" onclick={() => selectMode('multi')}>
          <div class="menu-icon">👥</div>
          <h2 class="menu-title">Mehrspieler</h2>
          <p class="menu-desc">Spieler 1 gegen Spieler 2</p>
        </button>
      </div>
    {:else}
      <div class="difficulty-section">
        <div class="difficulty-top">
          <button class="back-btn" onclick={() => menuStep = 'mode'} aria-label="Zurück">
            <span class="back-arrow">←</span> Zurück
          </button>
        </div>
        <header class="difficulty-header">
          <h2 class="difficulty-heading">Schwierigkeit wählen</h2>
          <p class="difficulty-sub">Wie viele Paare traust du dir zu?</p>
        </header>
        <div class="difficulty-cards">
          {#each Object.entries(difficultyConfig) as [key, cfg] (key)}
            <button class="difficulty-card {key}" onclick={() => startWithDifficulty(key as Difficulty)}>
              <div class="difficulty-card-glow {key}"></div>
              <div class="difficulty-icon">{cfg.icon}</div>
              <h3 class="difficulty-title">{cfg.label}</h3>
              <p class="difficulty-desc">{cfg.desc}</p>
              <div class="difficulty-bar">
                <span class="difficulty-bar-fill {key}"></span>
              </div>
            </button>
          {/each}
        </div>
      </div>
    {/if}
  </div>
{:else}
  <div class="game-container" class:game-over-active={gameOver}>
    {#if gameOver}
      <div class="confetti-layer">
        {#each confettiPieces as piece (piece.id)}
          <div
            class="confetti"
            style="
              left: {piece.x}%;
              animation-delay: {piece.delay}s;
              animation-duration: {piece.duration}s;
              --rotate: {piece.rotate}deg;
              --size: {piece.size}px;
              --color: {piece.color};
            "
          >
            {#if piece.emoji}
              <span class="confetti-emoji">{piece.emoji}</span>
            {:else}
              <span class="confetti-shape"></span>
            {/if}
          </div>
        {/each}
      </div>

      <div class="winner-overlay" onclick={restartGame} onkeydown={(e) => e.key === 'Enter' && restartGame()} role="button" tabindex="0">
        <div class="winner-card">
          <div class="winner-trophy">{winner === 'Unentschieden' ? '🤝' : '🏆'}</div>
          <h2 class="winner-title">{winner === 'Unentschieden' ? 'Unentschieden!' : winner}</h2>
          <p class="winner-subtitle">{winner === 'Unentschieden' ? 'Beide gleich stark!' : 'gewinnt das Spiel!'}</p>
          <div class="winner-scores">
            <div class="winner-score-item">
              <span class="winner-score-num">{scores[1]}</span>
              <span class="winner-score-label">{player1Label}</span>
            </div>
            <div class="winner-score-divider"></div>
            <div class="winner-score-item">
              <span class="winner-score-num">{scores[2]}</span>
              <span class="winner-score-label">{player2Label}</span>
            </div>
          </div>
          <p class="winner-moves">{moves} Züge gesamt</p>
          <div class="winner-buttons">
            <button class="winner-restart" onclick={restartGame}>Nochmal spielen</button>
            <button class="winner-menu" onclick={backToMenu}>Zum Menü</button>
          </div>
        </div>
      </div>
    {/if}

    <header class="game-header">
      <h1 class="title">Memory</h1>
      <p class="subtitle">{gameMode === 'single' ? 'Einzelspieler' : 'Mehrspieler'} · {difficultyLabel}</p>
    </header>

    <div class="play-area">
      <aside class="sidebar">
        <div class="scoreboard">
          <div class="player" class:active={currentPlayer === 1 && !gameOver}>
            <div class="player-avatar p1">{gameMode === 'single' ? '👤' : '1'}</div>
            <div class="player-info">
              <span class="player-label">{player1Label}</span>
              <span class="player-score">{scores[1]}</span>
            </div>
          </div>
          <div class="player" class:active={currentPlayer === 2 && !gameOver}>
            <div class="player-avatar p2">{gameMode === 'single' ? '🤖' : '2'}</div>
            <div class="player-info">
              <span class="player-label">{player2Label}</span>
              <span class="player-score">{scores[2]}</span>
            </div>
          </div>
        </div>

        <div class="stats">
          <div class="stat">
            <span class="stat-value">{moves}</span>
            <span class="stat-label">Züge</span>
          </div>
          <div class="stat">
            <span class="stat-value">{matchedCount}/{totalPairs}</span>
            <span class="stat-label">Paare</span>
          </div>
        </div>

        <div class="status-box">
          {#if gameOver}
            <span class="game-over">
              {winner === 'Unentschieden' ? 'Unentschieden!' : winner + ' gewinnt!'}
            </span>
          {:else if isComputerTurn}
            <span class="turn-text">
              <span class="turn-dot p2"></span>
              Computer denkt nach...
            </span>
          {:else}
            <span class="turn-text">
              <span class="turn-dot p{currentPlayer}"></span>
              {currentPlayer === 1 ? player1Label : player2Label} ist dran
            </span>
          {/if}
        </div>

        <button class="restart-btn" onclick={restartGame}>
          <span>Neues Spiel</span>
        </button>
        <button class="menu-btn" onclick={backToMenu}>
          <span>Zum Menü</span>
        </button>
      </aside>

      <div class="grid {difficulty}">
        {#each cards as card (card.id)}
          <button
            class="card"
            class:flipped={card.flipped}
            class:matched={card.matched}
            onclick={() => flipCard(card)}
            disabled={card.matched || (gameMode === 'single' && currentPlayer === 2)}
            aria-label={card.flipped || card.matched ? card.name : 'Verdeckte Karte'}
          >
            <div class="card-inner">
              <div class="card-back">
                <div class="card-back-pattern"></div>
                <div class="card-back-overlay"></div>
              </div>
              <div class="card-front">
                <span class="emoji">{card.name}</span>
              </div>
            </div>
          </button>
        {/each}
      </div>
    </div>
  </div>
{/if}
</div>

<style>
  /* ── Theme Variables ── */
  .theme-dark {
    --bg-top: #1a1c2e;
    --bg-bottom: #0f1019;
    --text-primary: #f0f1f8;
    --text-secondary: #c2c5e0;
    --text-muted: #8b90b0;
    --text-dim: #6b7090;
    --surface: rgba(255, 255, 255, 0.04);
    --surface-2: rgba(255, 255, 255, 0.03);
    --surface-hover: rgba(255, 255, 255, 0.08);
    --border: rgba(255, 255, 255, 0.07);
    --border-soft: rgba(255, 255, 255, 0.05);
    --border-hover: rgba(255, 255, 255, 0.14);
    --card-back-bg: linear-gradient(145deg, #252840, #1a1c2e);
    --card-back-border: rgba(255, 255, 255, 0.08);
    --card-front-bg: linear-gradient(145deg, #2a2d48, #1e2138);
    --menu-card-bg: linear-gradient(145deg, #1e2138, #161828);
    --overlay-bg: rgba(15, 16, 25, 0.82);
    --winner-card-bg: linear-gradient(145deg, #1e2138, #161828);
    --winner-border: rgba(110, 231, 183, 0.25);
    --winner-shadow: 0 0 60px rgba(110, 231, 183, 0.15), 0 20px 60px rgba(0, 0, 0, 0.5);
    --title-gradient: linear-gradient(135deg, #6ee7b7 0%, #38bdf8 50%, #818cf8 100%);
    --toggle-bg: rgba(255, 255, 255, 0.06);
    --toggle-border: rgba(255, 255, 255, 0.1);
    --pattern-stroke: rgba(110, 231, 183, 0.25);
    --pattern-green: rgba(110, 231, 183, 0.12);
    --pattern-blue: rgba(96, 165, 250, 0.12);
    --card-front-border: rgba(110, 231, 183, 0.3);
    --card-hover-border: rgba(110, 231, 183, 0.35);
    --card-glow: rgba(110, 231, 183, 0.12);
    --orb-1: rgba(110, 231, 183, 0.08);
    --orb-2: rgba(96, 165, 250, 0.07);
    --shadow-card: 0 4px 24px rgba(0, 0, 0, 0.25);
    --divider: rgba(255, 255, 255, 0.1);
  }

  .theme-light {
    --bg-top: #eef0f7;
    --bg-bottom: #dce0ec;
    --text-primary: #1a1c2e;
    --text-secondary: #3a3d52;
    --text-muted: #6b7090;
    --text-dim: #9499b0;
    --surface: rgba(255, 255, 255, 0.65);
    --surface-2: rgba(255, 255, 255, 0.5);
    --surface-hover: rgba(255, 255, 255, 0.85);
    --border: rgba(0, 0, 0, 0.08);
    --border-soft: rgba(0, 0, 0, 0.06);
    --border-hover: rgba(0, 0, 0, 0.16);
    --card-back-bg: linear-gradient(145deg, #ffffff, #e6e9f2);
    --card-back-border: rgba(0, 0, 0, 0.1);
    --card-front-bg: linear-gradient(145deg, #ffffff, #eef1f8);
    --menu-card-bg: linear-gradient(145deg, #ffffff, #eef1f8);
    --overlay-bg: rgba(200, 206, 222, 0.82);
    --winner-card-bg: linear-gradient(145deg, #ffffff, #eef1f8);
    --winner-border: rgba(52, 211, 153, 0.3);
    --winner-shadow: 0 0 60px rgba(52, 211, 153, 0.12), 0 20px 60px rgba(0, 0, 0, 0.12);
    --title-gradient: linear-gradient(135deg, #059669 0%, #2563eb 50%, #4f46e5 100%);
    --toggle-bg: rgba(255, 255, 255, 0.7);
    --toggle-border: rgba(0, 0, 0, 0.1);
    --pattern-stroke: rgba(52, 211, 153, 0.35);
    --pattern-green: rgba(52, 211, 153, 0.12);
    --pattern-blue: rgba(96, 165, 250, 0.12);
    --card-front-border: rgba(52, 211, 153, 0.4);
    --card-hover-border: rgba(52, 211, 153, 0.45);
    --card-glow: rgba(52, 211, 153, 0.1);
    --orb-1: rgba(52, 211, 153, 0.1);
    --orb-2: rgba(96, 165, 250, 0.09);
    --shadow-card: 0 4px 24px rgba(0, 0, 0, 0.08);
    --divider: rgba(0, 0, 0, 0.1);
  }

  /* ── App Shell ── */
  .app {
    position: relative;
    min-height: 100svh;
    background: radial-gradient(ellipse at top, var(--bg-top) 0%, var(--bg-bottom) 100%);
    transition: background 0.4s ease;
  }

  .bg-orb {
    position: fixed;
    border-radius: 50%;
    filter: blur(80px);
    pointer-events: none;
    z-index: 0;
    animation: orbFloat 20s ease-in-out infinite;
  }

  .orb-1 {
    width: 400px;
    height: 400px;
    top: -100px;
    left: -100px;
    background: var(--orb-1);
  }

  .orb-2 {
    width: 350px;
    height: 350px;
    bottom: -100px;
    right: -80px;
    background: var(--orb-2);
    animation-delay: -10s;
  }

  @keyframes orbFloat {
    0%, 100% { transform: translate(0, 0); }
    50% { transform: translate(40px, 30px); }
  }

  /* ── Toggle Group ── */
  .toggle-group {
    position: fixed;
    top: 18px;
    right: 18px;
    z-index: 300;
    display: flex;
    flex-direction: column;
    gap: 8px;
    align-items: flex-end;
  }

  .style-toggle {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    height: 44px;
    padding: 0 14px 0 12px;
    border-radius: 22px;
    border: 1.5px solid var(--toggle-border);
    background: var(--toggle-bg);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    cursor: pointer;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    font-family: inherit;
  }

  .style-toggle:hover {
    transform: scale(1.05);
    border-color: rgba(96, 165, 250, 0.4);
    box-shadow: 0 4px 16px rgba(96, 165, 250, 0.12);
  }

  .style-toggle-inner {
    font-size: 1.1rem;
    line-height: 1;
    transition: transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .style-toggle:hover .style-toggle-inner {
    transform: rotate(15deg) scale(1.1);
  }

  .style-toggle-label {
    font-size: 0.78rem;
    font-weight: 600;
    color: var(--text-secondary);
    letter-spacing: 0.3px;
    white-space: nowrap;
  }

  /* ── Theme Toggle ── */
  .theme-toggle {
    width: 44px;
    height: 44px;
    border-radius: 50%;
    border: 1.5px solid var(--toggle-border);
    background: var(--toggle-bg);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .theme-toggle:hover {
    transform: scale(1.1);
    border-color: rgba(110, 231, 183, 0.4);
    box-shadow: 0 4px 16px var(--card-glow);
  }

  .theme-icon {
    position: relative;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.3rem;
    line-height: 1;
  }

  .icon-sun, .icon-moon {
    position: absolute;
    transition: opacity 0.3s ease, transform 0.3s ease;
  }

  .theme-icon.show-sun .icon-sun {
    opacity: 1;
    transform: rotate(0deg) scale(1);
  }
  .theme-icon.show-sun .icon-moon {
    opacity: 0;
    transform: rotate(180deg) scale(0);
  }

  .theme-icon:not(.show-sun) .icon-sun {
    opacity: 0;
    transform: rotate(-180deg) scale(0);
  }
  .theme-icon:not(.show-sun) .icon-moon {
    opacity: 1;
    transform: rotate(0deg) scale(1);
  }

  /* ── Shared Typography ── */
  .title {
    font-size: 2.2rem;
    font-weight: 800;
    margin: 0;
    letter-spacing: -1px;
    text-transform: uppercase;
    background: var(--title-gradient);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
    color: var(--text-primary);
  }

  .subtitle {
    font-size: 0.85rem;
    color: var(--text-dim);
    margin: 2px 0 0;
    letter-spacing: 1px;
    text-transform: uppercase;
  }

  /* ── Menu Screen ── */
  .menu-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 40px;
    min-height: 100svh;
    box-sizing: border-box;
    padding: 24px;
    position: relative;
    z-index: 1;
  }

  .menu-header {
    text-align: center;
  }

  .menu-header .title {
    font-size: 3.2rem;
  }

  .menu-cards {
    display: flex;
    gap: 24px;
    flex-wrap: wrap;
    justify-content: center;
  }

  .menu-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    width: 250px;
    padding: 40px 28px;
    border-radius: 22px;
    background: var(--menu-card-bg);
    border: 1.5px solid var(--border);
    cursor: pointer;
    transition: all 0.35s cubic-bezier(0.4, 0, 0.2, 1);
    color: var(--text-secondary);
    box-shadow: var(--shadow-card);
  }

  .menu-card:hover {
    transform: translateY(-8px);
    box-shadow: 0 16px 48px rgba(0, 0, 0, 0.2);
  }

  .menu-card.single:hover {
    border-color: rgba(96, 165, 250, 0.4);
    box-shadow: 0 16px 48px rgba(96, 165, 250, 0.15);
  }

  .menu-card.multi:hover {
    border-color: rgba(110, 231, 183, 0.4);
    box-shadow: 0 16px 48px rgba(110, 231, 183, 0.15);
  }

  .menu-icon {
    font-size: 3.2rem;
    line-height: 1;
    margin-bottom: 6px;
    filter: drop-shadow(0 4px 12px rgba(0, 0, 0, 0.2));
    transition: transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  .menu-card:hover .menu-icon {
    transform: scale(1.15) translateY(-4px);
  }

  .menu-title {
    font-size: 1.35rem;
    font-weight: 700;
    margin: 0;
    color: var(--text-primary);
  }

  .menu-desc {
    font-size: 0.88rem;
    color: var(--text-muted);
    margin: 0;
    text-align: center;
  }

  /* ── Difficulty Selection ── */
  .difficulty-section {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0;
    width: 100%;
    max-width: 720px;
    animation: fadeIn 0.35s ease;
  }

  .difficulty-top {
    width: 100%;
    display: flex;
    justify-content: flex-start;
  }

  .back-btn {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 8px 18px;
    font-size: 0.85rem;
    font-weight: 600;
    color: var(--text-muted);
    background: var(--surface);
    border: 1.5px solid var(--border);
    border-radius: 10px;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .back-arrow {
    font-size: 1rem;
    line-height: 1;
  }

  .back-btn:hover {
    color: var(--text-primary);
    background: var(--surface-hover);
    border-color: var(--border-hover);
    transform: translateX(-3px);
  }

  .difficulty-header {
    text-align: center;
    margin: 28px 0 36px;
  }

  .difficulty-heading {
    font-size: 1.6rem;
    font-weight: 800;
    margin: 0;
    color: var(--text-primary);
    letter-spacing: -0.5px;
  }

  .difficulty-sub {
    font-size: 0.88rem;
    color: var(--text-muted);
    margin: 6px 0 0;
    letter-spacing: 0.3px;
  }

  .difficulty-cards {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 16px;
    width: 100%;
  }

  .difficulty-card {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 10px;
    padding: 32px 16px 24px;
    border-radius: 18px;
    background: var(--menu-card-bg);
    border: 2px solid var(--border);
    cursor: pointer;
    transition: all 0.35s cubic-bezier(0.4, 0, 0.2, 1);
    color: var(--text-secondary);
    box-shadow: var(--shadow-card);
    overflow: hidden;
  }

  .difficulty-card-glow {
    position: absolute;
    top: -40%;
    left: 50%;
    transform: translateX(-50%);
    width: 120%;
    height: 80%;
    border-radius: 50%;
    filter: blur(30px);
    opacity: 0;
    transition: opacity 0.35s ease;
    pointer-events: none;
  }

  .difficulty-card-glow.easy   { background: rgba(52, 211, 153, 0.25); }
  .difficulty-card-glow.medium { background: rgba(251, 191, 36, 0.25); }
  .difficulty-card-glow.hard   { background: rgba(249, 115, 22, 0.25); }
  .difficulty-card-glow.expert { background: rgba(248, 113, 113, 0.25); }

  .difficulty-card:hover .difficulty-card-glow {
    opacity: 1;
  }

  .difficulty-card:hover {
    transform: translateY(-8px);
    box-shadow: 0 16px 48px rgba(0, 0, 0, 0.25);
  }

  .difficulty-card.easy:hover   { border-color: rgba(52, 211, 153, 0.5); }
  .difficulty-card.medium:hover { border-color: rgba(251, 191, 36, 0.5); }
  .difficulty-card.hard:hover   { border-color: rgba(249, 115, 22, 0.5); }
  .difficulty-card.expert:hover { border-color: rgba(248, 113, 113, 0.5); }

  .difficulty-icon {
    font-size: 2.6rem;
    line-height: 1;
    filter: drop-shadow(0 4px 8px rgba(0, 0, 0, 0.2));
    transition: transform 0.35s cubic-bezier(0.34, 1.56, 0.64, 1);
    position: relative;
    z-index: 1;
  }

  .difficulty-card:hover .difficulty-icon {
    transform: scale(1.2) translateY(-3px);
  }

  .difficulty-title {
    font-size: 1.15rem;
    font-weight: 700;
    margin: 0;
    color: var(--text-primary);
    position: relative;
    z-index: 1;
  }

  .difficulty-desc {
    font-size: 0.78rem;
    color: var(--text-muted);
    margin: 0;
    position: relative;
    z-index: 1;
  }

  .difficulty-bar {
    width: 60%;
    height: 4px;
    border-radius: 2px;
    background: var(--border);
    margin-top: 4px;
    overflow: hidden;
    position: relative;
    z-index: 1;
  }

  .difficulty-bar-fill {
    display: block;
    height: 100%;
    border-radius: 2px;
    transition: width 0.3s ease;
  }

  .difficulty-bar-fill.easy   { width: 25%; background: linear-gradient(90deg, #34d399, #6ee7b7); }
  .difficulty-bar-fill.medium { width: 50%; background: linear-gradient(90deg, #fbbf24, #f59e0b); }
  .difficulty-bar-fill.hard   { width: 75%; background: linear-gradient(90deg, #f97316, #ea580c); }
  .difficulty-bar-fill.expert { width: 100%; background: linear-gradient(90deg, #f87171, #ef4444); }

  @media (max-width: 600px) {
    .difficulty-cards {
      grid-template-columns: repeat(2, 1fr);
    }
  }

  /* ── Game Container ── */
  .game-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 20px;
    padding: 28px 20px;
    min-height: 100svh;
    box-sizing: border-box;
    position: relative;
    z-index: 1;
    overflow: hidden;
  }

  .game-container.game-over-active {
    overflow: hidden;
  }

  .game-header {
    text-align: center;
  }

  .play-area {
    display: flex;
    align-items: center;
    gap: 28px;
  }

  /* ── Sidebar ── */
  .sidebar {
    display: flex;
    flex-direction: column;
    gap: 14px;
    width: 200px;
    flex-shrink: 0;
  }

  .scoreboard {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .player {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 14px 16px;
    border-radius: 14px;
    background: var(--surface);
    border: 1.5px solid var(--border);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
  }

  .player.active {
    background: var(--surface-hover);
    border-color: rgba(110, 231, 183, 0.4);
    box-shadow: 0 0 24px rgba(110, 231, 183, 0.12);
  }

  .player-avatar {
    width: 38px;
    height: 38px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 1.1rem;
    font-weight: 800;
    color: #fff;
    flex-shrink: 0;
    transition: all 0.3s ease;
  }

  .player-avatar.p1 {
    background: linear-gradient(135deg, #34d399, #059669);
  }

  .player-avatar.p2 {
    background: linear-gradient(135deg, #60a5fa, #2563eb);
  }

  .player.active .player-avatar {
    transform: scale(1.1);
    box-shadow: 0 0 12px rgba(255, 255, 255, 0.2);
  }

  .player-info {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .player-label {
    font-size: 0.78rem;
    color: var(--text-muted);
    letter-spacing: 0.5px;
  }

  .player-score {
    font-size: 1.6rem;
    font-weight: 800;
    color: var(--text-primary);
    line-height: 1;
  }

  /* ── Stats ── */
  .stats {
    display: flex;
    gap: 10px;
  }

  .stat {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 12px 8px;
    border-radius: 12px;
    background: var(--surface-2);
    border: 1px solid var(--border-soft);
  }

  .stat-value {
    font-size: 1.3rem;
    font-weight: 700;
    color: var(--text-primary);
    line-height: 1;
  }

  .stat-label {
    font-size: 0.72rem;
    color: var(--text-dim);
    margin-top: 4px;
    letter-spacing: 0.5px;
    text-transform: uppercase;
  }

  /* ── Status ── */
  .status-box {
    padding: 14px 16px;
    border-radius: 12px;
    background: var(--surface-2);
    border: 1px solid var(--border-soft);
    text-align: center;
    min-height: 50px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .turn-text {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 0.9rem;
    color: var(--text-secondary);
  }

  .turn-dot {
    width: 10px;
    height: 10px;
    border-radius: 50%;
    animation: pulse 1.5s ease-in-out infinite;
  }

  .turn-dot.p1 {
    background: #34d399;
    box-shadow: 0 0 8px rgba(52, 211, 153, 0.5);
  }

  .turn-dot.p2 {
    background: #60a5fa;
    box-shadow: 0 0 8px rgba(96, 165, 250, 0.5);
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.5; transform: scale(0.8); }
  }

  .game-over {
    font-size: 1.05rem;
    font-weight: 700;
    background: linear-gradient(135deg, #fbbf24, #f59e0b);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  /* ── Buttons ── */
  .restart-btn {
    padding: 14px 24px;
    font-size: 0.95rem;
    font-weight: 700;
    color: #0f1019;
    background: linear-gradient(135deg, #6ee7b7, #34d399);
    border: none;
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .restart-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(110, 231, 183, 0.35);
  }

  .restart-btn:active {
    transform: translateY(0);
  }

  .menu-btn {
    padding: 12px 24px;
    font-size: 0.88rem;
    font-weight: 600;
    color: var(--text-secondary);
    background: var(--surface);
    border: 1.5px solid var(--border);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .menu-btn:hover {
    background: var(--surface-hover);
    border-color: var(--border-hover);
  }

  /* ── Card Grid ── */
  .grid {
    display: grid;
    --card-size: 82px;
    --gap: 10px;
    grid-template-columns: repeat(var(--cols, 6), var(--card-size));
    gap: var(--gap);
  }

  .grid.easy   { --cols: 4; }
  .grid.medium { --cols: 6; }
  .grid.hard   { --cols: 6; --card-size: 68px; }
  .grid.expert { --cols: 8; --card-size: 58px; }

  .card {
    width: var(--card-size);
    height: var(--card-size);
    cursor: pointer;
    border: none;
    border-radius: 14px;
    padding: 0;
    background: transparent;
    perspective: 800px;
  }

  .card:disabled {
    cursor: default;
  }

  .card-inner {
    position: relative;
    width: 100%;
    height: 100%;
    transform-style: preserve-3d;
    transition: transform 0.5s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .card.flipped .card-inner,
  .card.matched .card-inner {
    transform: rotateY(180deg);
  }

  .card-back,
  .card-front {
    position: absolute;
    inset: 0;
    border-radius: 14px;
    display: flex;
    align-items: center;
    justify-content: center;
    backface-visibility: hidden;
    -webkit-backface-visibility: hidden;
  }

  .card-back {
    background: var(--card-back-bg);
    border: 1.5px solid var(--card-back-border);
    transition: all 0.2s ease;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  .card-back-pattern {
    width: calc(var(--card-size) * 0.34);
    height: calc(var(--card-size) * 0.34);
    border-radius: 8px;
    border: 2px solid var(--pattern-stroke);
    background:
      linear-gradient(45deg, transparent 46%, var(--pattern-green) 47%, var(--pattern-green) 53%, transparent 54%),
      linear-gradient(-45deg, transparent 46%, var(--pattern-blue) 47%, var(--pattern-blue) 53%, transparent 54%);
    transition: all 0.2s ease;
  }

  .card-back-overlay {
    position: absolute;
    inset: 0;
    border-radius: 14px;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s ease;
    overflow: hidden;
  }

  /* ── Card Style: Dots ── */
  .app[data-card-style='dots'] .card-back-pattern { display: none; }
  .app[data-card-style='dots'] .card-back-overlay {
    opacity: 1;
    background-image: radial-gradient(circle, var(--pattern-green) 25%, transparent 26%);
    background-size: calc(var(--card-size) * 0.24) calc(var(--card-size) * 0.24);
    background-position: center;
  }

  /* ── Card Style: Waves ── */
  .app[data-card-style='waves'] .card-back-pattern { display: none; }
  .app[data-card-style='waves'] .card-back-overlay {
    opacity: 1;
    background-image:
      repeating-linear-gradient(
        0deg,
        transparent 0,
        transparent calc(var(--card-size) * 0.13),
        var(--pattern-green) calc(var(--card-size) * 0.13),
        var(--pattern-green) calc(var(--card-size) * 0.15),
        transparent calc(var(--card-size) * 0.15),
        transparent calc(var(--card-size) * 0.26)
      );
  }

  /* ── Card Style: Stripes ── */
  .app[data-card-style='stripes'] .card-back-pattern { display: none; }
  .app[data-card-style='stripes'] .card-back-overlay {
    opacity: 1;
    background-image: repeating-linear-gradient(
      45deg,
      transparent 0,
      transparent 5px,
      var(--pattern-green) 5px,
      var(--pattern-green) 7px,
      transparent 7px,
      transparent 13px,
      var(--pattern-blue) 13px,
      var(--pattern-blue) 15px
    );
  }

  .card:not(:disabled):hover .card-back {
    border-color: var(--card-hover-border);
    box-shadow: 0 6px 20px var(--card-glow);
    transform: translateY(-2px);
  }

  .card:not(:disabled):hover .card-back-pattern {
    border-color: rgba(110, 231, 183, 0.5);
    transform: scale(1.12);
  }

  .card:not(:disabled):hover .card-back-overlay {
    filter: brightness(1.3);
  }

  .card-front {
    background: var(--card-front-bg);
    border: 1.5px solid var(--card-front-border);
    transform: rotateY(180deg);
  }

  .emoji {
    font-size: calc(var(--card-size) * 0.52);
    line-height: 1;
    filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.2));
  }

  .card.matched .card-front {
    border-color: rgba(110, 231, 183, 0.6);
    box-shadow: 0 0 20px rgba(110, 231, 183, 0.25);
    animation: matchPop 0.4s ease;
  }

  @keyframes matchPop {
    0% { transform: rotateY(180deg) scale(1); }
    50% { transform: rotateY(180deg) scale(1.12); }
    100% { transform: rotateY(180deg) scale(1); }
  }

  .card.matched {
    opacity: 0;
    transition: opacity 0.6s ease 0.4s, transform 0.6s ease 0.4s;
    transform: scale(0.85);
    pointer-events: none;
  }

  /* ── Confetti ── */
  .confetti-layer {
    position: fixed;
    inset: 0;
    pointer-events: none;
    z-index: 100;
    overflow: hidden;
  }

  .confetti {
    position: absolute;
    top: -40px;
    width: var(--size);
    height: var(--size);
    animation: confettiFall linear forwards;
  }

  .confetti-emoji {
    font-size: calc(var(--size) * 1.6);
    line-height: 1;
    display: block;
    animation: confettiSpin linear forwards;
    animation-duration: inherit;
  }

  .confetti-shape {
    display: block;
    width: var(--size);
    height: var(--size);
    background: var(--color);
    border-radius: 3px;
    animation: confettiSpin linear forwards;
    animation-duration: inherit;
  }

  @keyframes confettiFall {
    0% {
      transform: translateY(0) rotate(var(--rotate));
      opacity: 1;
    }
    90% {
      opacity: 1;
    }
    100% {
      transform: translateY(110vh) rotate(var(--rotate));
      opacity: 0;
    }
  }

  @keyframes confettiSpin {
    0% { transform: rotate(0deg) scale(1); }
    50% { transform: rotate(360deg) scale(1.1); }
    100% { transform: rotate(720deg) scale(0.9); }
  }

  /* ── Winner Overlay ── */
  .winner-overlay {
    position: fixed;
    inset: 0;
    background: var(--overlay-bg);
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 200;
    cursor: pointer;
    animation: fadeIn 0.4s ease;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  .winner-card {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    padding: 44px 52px;
    border-radius: 24px;
    background: var(--winner-card-bg);
    border: 1.5px solid var(--winner-border);
    box-shadow: var(--winner-shadow);
    animation: winnerPop 0.5s cubic-bezier(0.34, 1.56, 0.64, 1);
  }

  @keyframes winnerPop {
    0% { transform: scale(0.5) translateY(20px); opacity: 0; }
    100% { transform: scale(1) translateY(0); opacity: 1; }
  }

  .winner-trophy {
    font-size: 4rem;
    line-height: 1;
    animation: trophyBounce 1.5s ease-in-out infinite;
    filter: drop-shadow(0 4px 12px rgba(251, 191, 36, 0.4));
  }

  @keyframes trophyBounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-10px); }
  }

  .winner-title {
    font-size: 1.85rem;
    font-weight: 800;
    margin: 8px 0 0;
    color: var(--text-primary);
    letter-spacing: -0.5px;
  }

  .winner-subtitle {
    font-size: 1rem;
    color: var(--text-muted);
    margin: 0;
  }

  .winner-scores {
    display: flex;
    align-items: center;
    gap: 24px;
    margin: 20px 0 8px;
  }

  .winner-score-item {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 4px;
  }

  .winner-score-num {
    font-size: 2rem;
    font-weight: 800;
    line-height: 1;
  }

  .winner-score-item:first-child .winner-score-num {
    color: #34d399;
  }

  .winner-score-item:last-child .winner-score-num {
    color: #60a5fa;
  }

  .winner-score-label {
    font-size: 0.75rem;
    color: var(--text-dim);
    letter-spacing: 0.5px;
    text-transform: uppercase;
  }

  .winner-score-divider {
    width: 1px;
    height: 36px;
    background: var(--divider);
  }

  .winner-moves {
    font-size: 0.85rem;
    color: var(--text-dim);
    margin: 0 0 16px;
  }

  .winner-buttons {
    display: flex;
    flex-direction: column;
    gap: 10px;
    align-items: stretch;
  }

  .winner-restart {
    padding: 14px 32px;
    font-size: 1rem;
    font-weight: 700;
    color: #0f1019;
    background: linear-gradient(135deg, #6ee7b7, #34d399);
    border: none;
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .winner-restart:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(110, 231, 183, 0.35);
  }

  .winner-menu {
    padding: 10px 32px;
    font-size: 0.88rem;
    font-weight: 600;
    color: var(--text-secondary);
    background: var(--surface);
    border: 1.5px solid var(--border);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .winner-menu:hover {
    background: var(--surface-hover);
    border-color: var(--border-hover);
  }

  /* ── Responsive ── */
  @media (max-width: 760px) {
    .play-area {
      flex-direction: column-reverse;
      align-items: center;
      gap: 16px;
    }

    .sidebar {
      width: 100%;
      max-width: 380px;
      flex-direction: row;
      flex-wrap: wrap;
      align-items: center;
      justify-content: center;
    }

    .scoreboard {
      flex-direction: row;
      flex: 1 1 100%;
    }

    .player {
      flex: 1;
    }

    .stats {
      flex: 1 1 100%;
    }

    .status-box {
      flex: 1 1 100%;
    }

    .restart-btn {
      flex: 1 1 45%;
    }

    .menu-btn {
      flex: 1 1 45%;
    }

    .grid {
      --card-size: 52px;
      --gap: 7px;
    }

    .grid.hard { --card-size: 46px; }
    .grid.expert { --card-size: 40px; }

    .title {
      font-size: 1.8rem;
    }

    .menu-header .title {
      font-size: 2.4rem;
    }

    .menu-card {
      width: 100%;
      max-width: 300px;
    }

    .bg-orb {
      display: none;
    }
  }

  @media (max-width: 380px) {
    .grid {
      --card-size: 40px;
      --gap: 5px;
    }

    .grid.hard { --card-size: 36px; }
    .grid.expert { --card-size: 32px; }
  }
</style>
