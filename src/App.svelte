<script lang="ts">
  type Card = {
    id: number;
    name: string;
    flipped: boolean;
    matched: boolean;
  };

  type GameMode = 'menu' | 'single' | 'multi';

  const baseEmojis = [
    '🍎', '🍌', '🍕', '🚀', '🐱', '🎸', '⚽', '📚',
    '🌈', '🐙', '🔥', '🎯', '💎', '🦋', '🍀', '🌊',
    '⚡', '🍩', '👑', '🎲', '🌸', '🧩', '🎈', '🍍',
  ];

  function createCards(): Card[] {
    const duplicated = [...baseEmojis, ...baseEmojis];
    return duplicated
      .map((name, index) => ({ id: index, name, flipped: false, matched: false }))
      .sort(() => Math.random() - 0.5);
  }

  let gameMode = $state<GameMode>('menu');
  let cards = $state<Card[]>(createCards());
  let selectedCards = $state<Card[]>([]);
  let isChecking = $state(false);

  let currentPlayer = $state(1);
  let scores = $state<Record<number, number>>({ 1: 0, 2: 0 });
  let gameOver = $state(false);
  let winner = $state<string>('');
  let moves = $state(0);

  // Computer memory: map of card id -> emoji name for cards it has seen
  let aiMemory = $state<Map<number, string>>(new Map());

  let player1Label = $derived(gameMode === 'single' ? 'Du' : 'Spieler 1');
  let player2Label = $derived(gameMode === 'single' ? 'Computer' : 'Spieler 2');

  function startGame(mode: GameMode) {
    gameMode = mode;
    restartGame();
  }

  function flipCard(card: Card) {
    if (isChecking || card.flipped || card.matched) return;

    card.flipped = true;
    selectedCards = [...selectedCards, card];

    // Computer learns every card that gets flipped
    if (gameMode === 'single') {
      aiMemory.set(card.id, card.name);
      aiMemory = new Map(aiMemory);
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

    if (knownPair) {
      flipCard(knownPair[0]);
      setTimeout(() => flipCard(knownPair![1]), 450);
      return;
    }

    // No known pair — flip an unknown card first to learn it
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
    const secondCard = knownMatch ?? remaining[Math.floor(Math.random() * remaining.length)];

    setTimeout(() => flipCard(secondCard), 450);
  }

  function restartGame() {
    cards = createCards();
    selectedCards = [];
    currentPlayer = 1;
    scores = { 1: 0, 2: 0 };
    isChecking = false;
    gameOver = false;
    winner = '';
    moves = 0;
    aiMemory = new Map();
  }

  function backToMenu() {
    gameMode = 'menu';
    restartGame();
  }

  let matchedCount = $derived(cards.filter((c) => c.matched).length);
  let totalPairs = baseEmojis.length;
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
    } else {
      confettiPieces = [];
    }
  });
</script>

{#if gameMode === 'menu'}
  <div class="menu-container">
    <header class="menu-header">
      <h1 class="title">Memory</h1>
      <p class="subtitle">Finde alle Paare</p>
    </header>

    <div class="menu-cards">
      <button class="menu-card single" onclick={() => startGame('single')}>
        <div class="menu-icon">🤖</div>
        <h2 class="menu-title">Einzelspieler</h2>
        <p class="menu-desc">Spiele gegen den Computer</p>
      </button>

      <button class="menu-card multi" onclick={() => startGame('multi')}>
        <div class="menu-icon">👥</div>
        <h2 class="menu-title">Mehrspieler</h2>
        <p class="menu-desc">Spieler 1 gegen Spieler 2</p>
      </button>
    </div>
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
      <p class="subtitle">{gameMode === 'single' ? 'Einzelspieler' : 'Mehrspieler'}</p>
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

      <div class="grid">
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

<style>
  /* ── Shared ── */
  .title {
    font-size: 2.2rem;
    font-weight: 800;
    margin: 0;
    color: #f0f1f8;
    letter-spacing: -1px;
    text-transform: uppercase;
    background: linear-gradient(135deg, #6ee7b7 0%, #38bdf8 50%, #818cf8 100%);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
  }

  .subtitle {
    font-size: 0.85rem;
    color: #6b7090;
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
    background: radial-gradient(ellipse at top, #1a1c2e 0%, #0f1019 100%);
    color: #e2e4f0;
  }

  .menu-header {
    text-align: center;
  }

  .menu-header .title {
    font-size: 3rem;
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
    gap: 8px;
    width: 240px;
    padding: 36px 28px;
    border-radius: 20px;
    background: linear-gradient(145deg, #1e2138, #161828);
    border: 1.5px solid rgba(255, 255, 255, 0.06);
    cursor: pointer;
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    color: #e2e4f0;
  }

  .menu-card:hover {
    transform: translateY(-6px);
    border-color: rgba(110, 231, 183, 0.3);
    box-shadow: 0 12px 40px rgba(110, 231, 183, 0.1);
  }

  .menu-card.single:hover {
    border-color: rgba(96, 165, 250, 0.35);
    box-shadow: 0 12px 40px rgba(96, 165, 250, 0.12);
  }

  .menu-card.multi:hover {
    border-color: rgba(110, 231, 183, 0.35);
    box-shadow: 0 12px 40px rgba(110, 231, 183, 0.12);
  }

  .menu-icon {
    font-size: 3rem;
    line-height: 1;
    margin-bottom: 4px;
    filter: drop-shadow(0 4px 8px rgba(0, 0, 0, 0.3));
  }

  .menu-title {
    font-size: 1.3rem;
    font-weight: 700;
    margin: 0;
    color: #f0f1f8;
  }

  .menu-desc {
    font-size: 0.88rem;
    color: #8b90b0;
    margin: 0;
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
    background: radial-gradient(ellipse at top, #1a1c2e 0%, #0f1019 100%);
    color: #e2e4f0;
    position: relative;
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
    background: rgba(255, 255, 255, 0.04);
    border: 1.5px solid rgba(255, 255, 255, 0.06);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }

  .player.active {
    background: rgba(255, 255, 255, 0.07);
    border-color: rgba(110, 231, 183, 0.4);
    box-shadow: 0 0 20px rgba(110, 231, 183, 0.12);
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
    color: #8b90b0;
    letter-spacing: 0.5px;
  }

  .player-score {
    font-size: 1.6rem;
    font-weight: 800;
    color: #f0f1f8;
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
    background: rgba(255, 255, 255, 0.03);
    border: 1px solid rgba(255, 255, 255, 0.05);
  }

  .stat-value {
    font-size: 1.3rem;
    font-weight: 700;
    color: #f0f1f8;
    line-height: 1;
  }

  .stat-label {
    font-size: 0.72rem;
    color: #6b7090;
    margin-top: 4px;
    letter-spacing: 0.5px;
    text-transform: uppercase;
  }

  /* ── Status ── */
  .status-box {
    padding: 14px 16px;
    border-radius: 12px;
    background: rgba(255, 255, 255, 0.03);
    border: 1px solid rgba(255, 255, 255, 0.05);
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
    color: #c2c5e0;
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
    box-shadow: 0 6px 20px rgba(110, 231, 183, 0.3);
  }

  .restart-btn:active {
    transform: translateY(0);
  }

  .menu-btn {
    padding: 12px 24px;
    font-size: 0.88rem;
    font-weight: 600;
    color: #c2c5e0;
    background: rgba(255, 255, 255, 0.05);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .menu-btn:hover {
    background: rgba(255, 255, 255, 0.08);
    border-color: rgba(255, 255, 255, 0.12);
  }

  /* ── Card Grid ── */
  .grid {
    display: grid;
    grid-template-columns: repeat(6, 82px);
    gap: 10px;
  }

  .card {
    width: 82px;
    height: 82px;
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
    background: linear-gradient(145deg, #252840, #1a1c2e);
    border: 1.5px solid rgba(255, 255, 255, 0.08);
    transition: all 0.2s ease;
  }

  .card-back-pattern {
    width: 28px;
    height: 28px;
    border-radius: 8px;
    border: 2px solid rgba(110, 231, 183, 0.25);
    background:
      linear-gradient(45deg, transparent 46%, rgba(110, 231, 183, 0.12) 47%, rgba(110, 231, 183, 0.12) 53%, transparent 54%),
      linear-gradient(-45deg, transparent 46%, rgba(96, 165, 250, 0.12) 47%, rgba(96, 165, 250, 0.12) 53%, transparent 54%);
  }

  .card:not(:disabled):hover .card-back {
    border-color: rgba(110, 231, 183, 0.35);
    box-shadow: 0 4px 16px rgba(110, 231, 183, 0.12);
  }

  .card:not(:disabled):hover .card-back-pattern {
    border-color: rgba(110, 231, 183, 0.45);
    transform: scale(1.1);
    transition: transform 0.2s ease;
  }

  .card-front {
    background: linear-gradient(145deg, #2a2d48, #1e2138);
    border: 1.5px solid rgba(110, 231, 183, 0.3);
    transform: rotateY(180deg);
  }

  .emoji {
    font-size: 2.2rem;
    line-height: 1;
    filter: drop-shadow(0 2px 4px rgba(0, 0, 0, 0.3));
  }

  .card.matched .card-front {
    border-color: rgba(110, 231, 183, 0.6);
    box-shadow: 0 0 16px rgba(110, 231, 183, 0.2);
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
    background: rgba(15, 16, 25, 0.8);
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
    padding: 40px 48px;
    border-radius: 24px;
    background: linear-gradient(145deg, #1e2138, #161828);
    border: 1.5px solid rgba(110, 231, 183, 0.25);
    box-shadow: 0 0 60px rgba(110, 231, 183, 0.15), 0 20px 60px rgba(0, 0, 0, 0.5);
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
    font-size: 1.8rem;
    font-weight: 800;
    margin: 8px 0 0;
    color: #f0f1f8;
    letter-spacing: -0.5px;
  }

  .winner-subtitle {
    font-size: 1rem;
    color: #8b90b0;
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
    color: #6b7090;
    letter-spacing: 0.5px;
    text-transform: uppercase;
  }

  .winner-score-divider {
    width: 1px;
    height: 36px;
    background: rgba(255, 255, 255, 0.1);
  }

  .winner-moves {
    font-size: 0.85rem;
    color: #6b7090;
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
    box-shadow: 0 6px 20px rgba(110, 231, 183, 0.3);
  }

  .winner-menu {
    padding: 10px 32px;
    font-size: 0.88rem;
    font-weight: 600;
    color: #c2c5e0;
    background: rgba(255, 255, 255, 0.05);
    border: 1px solid rgba(255, 255, 255, 0.08);
    border-radius: 12px;
    cursor: pointer;
    transition: all 0.2s ease;
    letter-spacing: 0.5px;
  }

  .winner-menu:hover {
    background: rgba(255, 255, 255, 0.08);
    border-color: rgba(255, 255, 255, 0.12);
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
      grid-template-columns: repeat(6, 52px);
      gap: 7px;
    }

    .card {
      width: 52px;
      height: 52px;
    }

    .emoji {
      font-size: 1.4rem;
    }

    .card-back-pattern {
      width: 18px;
      height: 18px;
    }

    .title {
      font-size: 1.8rem;
    }

    .menu-header .title {
      font-size: 2.2rem;
    }

    .menu-card {
      width: 100%;
      max-width: 300px;
    }
  }

  @media (max-width: 380px) {
    .grid {
      grid-template-columns: repeat(6, 44px);
      gap: 5px;
    }

    .card {
      width: 44px;
      height: 44px;
    }

    .emoji {
      font-size: 1.2rem;
    }
  }
</style>
