import React, { useState, useEffect, useMemo, useRef, useCallback } from "react";

/* ============================================================
   AWALÉ — moteur de jeu (fonctions pures, testées)
   Plateau : 12 entiers. Cases 0–5 = Humain, 6–11 = IA.
   ============================================================ */

const sumRange = (b, s, e) => { let t = 0; for (let i = s; i <= e; i++) t += b[i]; return t; };
const sideOf = (pit) => (pit <= 5 ? "human" : "ai");
const oppRange = (p) => (p === "human" ? [6, 11] : [0, 5]);
const ownRange = (p) => (p === "human" ? [0, 5] : [6, 11]);

function applyCapture(b, lastIdx, player) {
  const [os, oe] = oppRange(player);
  const list = []; let captured = 0; let c = lastIdx;
  while (c >= os && c <= oe && (b[c] === 2 || b[c] === 3)) { list.push(c); captured += b[c]; c--; }
  if (captured > 0 && captured === sumRange(b, os, oe)) return { captured: 0, list: [] };
  for (const p of list) b[p] = 0;
  return { captured, list };
}
function simulateMove(board, pit) {
  const b = board.slice();
  let seeds = b[pit];
  if (seeds === 0) return null;
  const player = sideOf(pit);
  b[pit] = 0;
  let idx = pit;
  while (seeds > 0) { idx = (idx + 1) % 12; if (idx === pit) continue; b[idx]++; seeds--; }
  const { captured, list } = applyCapture(b, idx, player);
  return { board: b, captured, capturedPits: list, lastIndex: idx };
}
function getValidMoves(board, player) {
  const [s, e] = ownRange(player);
  const [os, oe] = oppRange(player);
  const cands = [];
  for (let i = s; i <= e; i++) if (board[i] > 0) cands.push(i);
  if (sumRange(board, os, oe) === 0) {
    const feeding = cands.filter((i) => sumRange(simulateMove(board, i).board, os, oe) > 0);
    return feeding.length ? feeding : [];
  }
  return cands;
}
function isMoveValid(board, player, pit) { return getValidMoves(board, player).includes(pit); }
function playMove(state, pit) {
  const player = state.turn;
  const sim = simulateMove(state.board, pit);
  return {
    board: sim.board,
    humanScore: state.humanScore + (player === "human" ? sim.captured : 0),
    aiScore: state.aiScore + (player === "ai" ? sim.captured : 0),
    turn: player === "human" ? "ai" : "human",
  };
}
function checkGameOver(state) {
  const { board, humanScore, aiScore, turn } = state;
  const reach = humanScore >= 25 || aiScore >= 25;
  const blocked = getValidMoves(board, turn).length === 0;
  if (reach || blocked) {
    const hs = humanScore + sumRange(board, 0, 5);
    const as = aiScore + sumRange(board, 6, 11);
    return { over: true, finalHuman: hs, finalAi: as, winner: hs > as ? "human" : as > hs ? "ai" : "draw", reason: reach ? "25" : "blocked" };
  }
  return { over: false, finalHuman: humanScore, finalAi: aiScore };
}
// IA niveau « Facile » : capture immédiate maximale, puis graines conservées, puis aléatoire.
function aiChooseMove(board) {
  const moves = getValidMoves(board, "ai");
  if (!moves.length) return null;
  const res = moves.map((m) => { const s = simulateMove(board, m); return { m, cap: s.captured, own: sumRange(s.board, 6, 11) }; });
  const maxCap = Math.max(...res.map((r) => r.cap));
  let f = res.filter((r) => r.cap === maxCap);
  const maxOwn = Math.max(...f.map((r) => r.own));
  f = f.filter((r) => r.own === maxOwn);
  return f[Math.floor(Math.random() * f.length)].m;
}
// IA « Moyen / Difficile » : minimax + alpha-bêta + approfondissement itératif borné en temps.
function evaluate(s) { return (s.aiScore - s.humanScore) * 14 + (sumRange(s.board, 6, 11) - sumRange(s.board, 0, 5)); }
const TIMEOUT = {};
function minimax(state, depth, alpha, beta, deadline) {
  if (Date.now() > deadline) throw TIMEOUT;
  const over = checkGameOver(state);
  if (over.over) return (over.finalAi - over.finalHuman) * 1000;
  if (depth <= 0) return evaluate(state);
  const moves = getValidMoves(state.board, state.turn);
  if (state.turn === "ai") {
    let best = -Infinity;
    for (const m of moves) { const v = minimax(playMove(state, m), depth - 1, alpha, beta, deadline); if (v > best) best = v; if (best > alpha) alpha = best; if (alpha >= beta) break; }
    return best;
  } else {
    let best = Infinity;
    for (const m of moves) { const v = minimax(playMove(state, m), depth - 1, alpha, beta, deadline); if (v < best) best = v; if (best < beta) beta = best; if (beta <= alpha) break; }
    return best;
  }
}
function rootBest(state, depth, deadline) {
  const moves = getValidMoves(state.board, "ai");
  let bestScore = -Infinity; const scored = [];
  for (const m of moves) { const v = minimax(playMove(state, m), depth - 1, -Infinity, Infinity, deadline); scored.push({ m, v }); if (v > bestScore) bestScore = v; }
  const top = scored.filter((s) => s.v === bestScore);
  return top[Math.floor(Math.random() * top.length)].m;
}
function aiSearchMove(state, maxDepth, budgetMs) {
  const deadline = Date.now() + budgetMs;
  let chosen = getValidMoves(state.board, "ai")[0];
  for (let d = 2; d <= maxDepth; d++) { try { chosen = rootBest(state, d, deadline); } catch (e) { if (e !== TIMEOUT) throw e; break; } }
  return chosen;
}
function sowPath(board, pit) {
  let seeds = board[pit]; let idx = pit; const path = [];
  while (seeds > 0) { idx = (idx + 1) % 12; if (idx === pit) continue; path.push(idx); seeds--; }
  return path;
}

/* ============================================================
   SON (Web Audio, synthétisé — aucun fichier externe)
   ============================================================ */
let actx = null;
function audio() {
  if (typeof window === "undefined") return null;
  if (!actx) { const AC = window.AudioContext || window.webkitAudioContext; if (AC) actx = new AC(); }
  if (actx && actx.state === "suspended") actx.resume();
  return actx;
}
function tone(freq, dur, type, gain, when = 0) {
  const ctx = audio(); if (!ctx) return;
  const o = ctx.createOscillator(), g = ctx.createGain();
  o.type = type; o.frequency.value = freq;
  o.connect(g); g.connect(ctx.destination);
  const t = ctx.currentTime + when;
  g.gain.setValueAtTime(0.0001, t);
  g.gain.exponentialRampToValueAtTime(gain, t + 0.008);
  g.gain.exponentialRampToValueAtTime(0.0001, t + dur);
  o.start(t); o.stop(t + dur + 0.02);
}
const sfx = {
  drop: (i) => tone(196 + (i % 8) * 16, 0.09, "triangle", 0.05),
  capture: () => { tone(540, 0.12, "sine", 0.06); tone(820, 0.16, "sine", 0.05, 0.07); },
  win: () => [523, 659, 784, 1047].forEach((f, i) => tone(f, 0.22, "sine", 0.06, i * 0.1)),
  lose: () => [392, 330, 262].forEach((f, i) => tone(f, 0.28, "sine", 0.06, i * 0.12)),
  draw: () => { tone(440, 0.3, "sine", 0.05); tone(440, 0.3, "sine", 0.05, 0.12); },
};
const buzz = (p) => { try { navigator.vibrate && navigator.vibrate(p); } catch (e) {} };

/* ============================================================
   UI
   ============================================================ */
const TOP_ROW = [11, 10, 9, 8, 7, 6];
const BOTTOM_ROW = [0, 1, 2, 3, 4, 5];
const LAYOUTS = {
  1: [[.5, .5]], 2: [[.38, .5], [.62, .5]], 3: [[.5, .35], [.36, .63], [.64, .63]],
  4: [[.36, .36], [.64, .36], [.36, .64], [.64, .64]],
  5: [[.5, .3], [.32, .5], [.68, .5], [.4, .72], [.6, .72]],
  6: [[.36, .32], [.64, .32], [.28, .56], [.72, .56], [.42, .76], [.58, .76]],
};
const SCATTER = [[.5, .5], [.33, .34], [.67, .33], [.31, .67], [.69, .66], [.5, .25], [.5, .75], [.24, .5], [.76, .5], [.4, .42], [.6, .42], [.4, .6], [.6, .6]];
const seedPositions = (n) => (n <= 6 ? LAYOUTS[n] || [] : SCATTER.slice(0, Math.min(n, SCATTER.length)));
const SEED_TONES = [
  "radial-gradient(circle at 34% 27%, #fdf6e3, #d9c294 58%, #a98a55)",
  "radial-gradient(circle at 34% 27%, #f7ecd2, #cdb583 58%, #997b4b)",
  "radial-gradient(circle at 34% 27%, #fff4de, #e0caa0 58%, #b39668)",
  "radial-gradient(circle at 34% 27%, #f3e6c9, #c8ad78 58%, #927442)",
];
const C = {
  human: "#e8a33d", humanGlow: "rgba(232,163,61,.55)",
  ai: "#56b9ad", aiGlow: "rgba(86,185,173,.5)",
  text: "#f3e9d6", muted: "#a98f70",
  serif: "'Fraunces', Georgia, 'Times New Roman', serif",
  sans: "'Inter', system-ui, -apple-system, sans-serif",
};
const DIFFS = [
  { id: "easy", label: "Facile" },
  { id: "medium", label: "Moyen" },
  { id: "hard", label: "Difficile" },
];

function Seed({ x, y, d, i }) {
  const sz = Math.max(5, d * 0.17), rot = (i * 47) % 360;
  return <span style={{
    position: "absolute", left: `${x * 100}%`, top: `${y * 100}%`, width: sz, height: sz * 0.92,
    marginLeft: -sz / 2, marginTop: -sz * 0.46, borderRadius: "50%", transform: `rotate(${rot}deg)`,
    background: SEED_TONES[i % SEED_TONES.length],
    boxShadow: "0 1px 2px rgba(0,0,0,.55), inset 0 -1px 1px rgba(120,90,40,.4), inset 0 1px 1px rgba(255,255,255,.45)",
  }} />;
}

function Pit({ index, count, diameter, side, isPlayable, isLast, isCapturing, isSource, isDrop, pitRef, onClick }) {
  const accent = side === "human" ? C.human : C.ai;
  const ring = isPlayable
    ? `, 0 0 0 2.5px ${accent}, 0 0 20px ${side === "human" ? C.humanGlow : C.aiGlow}`
    : isLast ? `, 0 0 0 2px ${accent}88` : "";
  return (
    <button ref={pitRef} onClick={onClick} disabled={!isPlayable} aria-label={`Case ${index}, ${count} graines`}
      className={"pit" + (isPlayable ? " playable" : "")}
      style={{
        position: "relative", width: diameter, height: diameter, borderRadius: "50%",
        border: "1px solid rgba(20,10,2,.55)", padding: 0,
        background: isCapturing ? "radial-gradient(circle at 50% 38%, #7a5320, #2a1a0e 82%)" : "radial-gradient(circle at 50% 36%, #2c1c0f, #150c06 84%)",
        boxShadow: `inset 0 6px 11px rgba(0,0,0,.7), inset 0 -2px 5px rgba(255,205,130,.08), inset 0 0 0 1px rgba(0,0,0,.25)${ring}`,
        cursor: isPlayable ? "pointer" : "default",
        transition: "box-shadow .22s ease, transform .15s cubic-bezier(.34,1.56,.64,1)",
        transform: isDrop ? "scale(1.06)" : isSource ? "scale(.97)" : "scale(1)",
        WebkitTapHighlightColor: "transparent",
      }}>
      {seedPositions(count).map(([x, y], i) => <Seed key={i} x={x} y={y} d={diameter} i={index * 7 + i} />)}
      <span style={{
        position: "absolute", bottom: -4, right: -4, minWidth: 25, height: 25, padding: "0 6px",
        display: "flex", alignItems: "center", justifyContent: "center",
        fontFamily: C.serif, fontSize: 16, fontWeight: 700,
        color: count === 0 ? C.muted : "#1a120a", background: count === 0 ? "rgba(0,0,0,.5)" : accent,
        borderRadius: 13, boxShadow: "0 2px 5px rgba(0,0,0,.6)", lineHeight: 1, transition: "background .2s ease, color .2s ease",
      }}>{count}</span>
    </button>
  );
}

function ScoreCard({ label, score, color, active, pop }) {
  return (
    <div style={{
      position: "relative", flex: 1, padding: "11px 15px", borderRadius: 16,
      background: active ? "rgba(255,255,255,.065)" : "rgba(255,255,255,.025)",
      border: `1.5px solid ${active ? color : "rgba(255,255,255,.08)"}`,
      boxShadow: active ? `0 0 22px ${color}30` : "none", transition: "all .3s ease",
    }}>
      <div style={{ fontFamily: C.sans, fontSize: 13, letterSpacing: ".13em", textTransform: "uppercase", color: C.muted }}>{label}</div>
      <span key={score} style={{ fontFamily: C.serif, fontSize: 42, fontWeight: 600, color, lineHeight: 1.1, display: "inline-block", animation: "scorepop .45s cubic-bezier(.34,1.56,.64,1)" }}>{score}</span>
      {pop ? <span key={pop.id} style={{ position: "absolute", right: 14, top: 6, fontFamily: C.serif, fontSize: 22, fontWeight: 600, color, animation: "floatUp 1s ease forwards", pointerEvents: "none" }}>+{pop.amount}</span> : null}
    </div>
  );
}

function SpeakerIcon({ on }) {
  return (
    <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke={on ? C.human : C.muted} strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
      <path d="M4 9v6h4l5 4V5L8 9H4z" fill={on ? C.human : "none"} fillOpacity={on ? 0.25 : 0} />
      {on ? <><path d="M16.5 8.5a5 5 0 0 1 0 7" /><path d="M19 6a8 8 0 0 1 0 12" /></> : <path d="M17 9l5 5M22 9l-5 5" />}
    </svg>
  );
}

export default function Awale() {
  const makeStart = () => ({ board: Array(12).fill(4), humanScore: 0, aiScore: 0, turn: "human" });
  const [state, setState] = useState(makeStart);
  const [displayBoard, setDisplayBoard] = useState(() => Array(12).fill(4));
  const [busy, setBusy] = useState(false);
  const [lastMove, setLastMove] = useState({ pit: -1, captured: [] });
  const [source, setSource] = useState(-1);
  const [dropPit, setDropPit] = useState(-1);
  const [captureFlash, setCaptureFlash] = useState([]);
  const [flying, setFlying] = useState({ x: 0, y: 0, visible: false, t: 0 });
  const [pop, setPop] = useState({ human: null, ai: null });
  const [difficulty, setDifficulty] = useState("medium");
  const [soundOn, setSoundOn] = useState(true);
  const [vw, setVw] = useState(typeof window !== "undefined" ? Math.min(window.innerWidth, 480) : 400);
  const [mounted, setMounted] = useState(false);

  const boardRef = useRef(null);
  const pitRefs = useRef([]);
  const runningRef = useRef(false);
  const cancelled = useRef(false);
  const diffRef = useRef("medium");
  const soundRef = useRef(true);

  useEffect(() => {
    const l = document.createElement("link");
    l.rel = "stylesheet";
    l.href = "https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,500;9..144,600&family=Inter:wght@400;500;600&display=swap";
    document.head.appendChild(l);
    const onR = () => setVw(Math.min(window.innerWidth, 480));
    window.addEventListener("resize", onR);
    requestAnimationFrame(() => setMounted(true));
    return () => window.removeEventListener("resize", onR);
  }, []);

  const result = useMemo(() => checkGameOver(state), [state]);
  const validHuman = useMemo(
    () => (state.turn === "human" && !result.over && !busy ? getValidMoves(state.board, "human") : []),
    [state, result.over, busy]
  );

  // Effets de fin de partie (son + vibration), une seule fois.
  const endFired = useRef(false);
  useEffect(() => {
    if (result.over && !busy && !endFired.current) {
      endFired.current = true;
      if (soundRef.current) (result.winner === "human" ? sfx.win : result.winner === "ai" ? sfx.lose : sfx.draw)();
      buzz(result.winner === "human" ? [40, 50, 90] : 60);
    }
    if (!result.over) endFired.current = false;
  }, [result.over, busy, result.winner]);

  const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
  const reduced = typeof window !== "undefined" && window.matchMedia && window.matchMedia("(prefers-reduced-motion: reduce)").matches;

  const pitCenters = useCallback(() => {
    const cont = boardRef.current; const res = {};
    if (!cont) return res;
    const cr = cont.getBoundingClientRect();
    for (let i = 0; i < 12; i++) { const el = pitRefs.current[i]; if (el) { const r = el.getBoundingClientRect(); res[i] = { x: r.left - cr.left + r.width / 2, y: r.top - cr.top + r.height / 2 }; } }
    return res;
  }, []);

  const animateMove = useCallback(async (curState, pit) => {
    if (runningRef.current || pit == null) return;
    runningRef.current = true; cancelled.current = false; setBusy(true);
    const player = curState.turn;
    const sim = simulateMove(curState.board, pit);
    const path = sowPath(curState.board, pit);
    const n = path.length;

    if (reduced) {
      setDisplayBoard(sim.board.slice());
      setLastMove({ pit, captured: sim.capturedPits });
      if (sim.capturedPits.length) { setCaptureFlash(sim.capturedPits); if (soundRef.current) sfx.capture(); buzz(25); setPop((p) => ({ ...p, [player]: { amount: sim.captured, id: Date.now() } })); await sleep(450); setCaptureFlash([]); }
      setState(playMove(curState, pit)); runningRef.current = false; setBusy(false); return;
    }

    const perHop = Math.max(52, Math.min(132, 900 / n));
    const disp = curState.board.slice(); disp[pit] = 0;
    setSource(pit); setDisplayBoard(disp.slice());
    const centers = pitCenters();
    setFlying({ ...(centers[pit] || { x: 0, y: 0 }), visible: true, t: 0 });
    await sleep(110);
    if (cancelled.current) { runningRef.current = false; return; }

    for (let i = 0; i < n; i++) {
      const tgt = path[i];
      setFlying({ ...(centers[tgt] || { x: 0, y: 0 }), visible: true, t: perHop });
      await sleep(perHop);
      if (cancelled.current) { runningRef.current = false; return; }
      disp[tgt] += 1; setDisplayBoard(disp.slice()); setDropPit(tgt);
      if (soundRef.current) sfx.drop(i);
      await sleep(perHop * 0.4); setDropPit(-1);
    }
    setFlying((f) => ({ ...f, visible: false })); setSource(-1);
    await sleep(170);
    if (cancelled.current) { runningRef.current = false; return; }

    if (sim.capturedPits.length) {
      setCaptureFlash(sim.capturedPits);
      if (soundRef.current) sfx.capture(); buzz(28);
      setPop((p) => ({ ...p, [player]: { amount: sim.captured, id: Date.now() } }));
      await sleep(150); setDisplayBoard(sim.board.slice());
      await sleep(540); setCaptureFlash([]);
    }
    if (cancelled.current) { runningRef.current = false; return; }
    setLastMove({ pit, captured: sim.capturedPits });
    setDisplayBoard(sim.board.slice());
    setState(playMove(curState, pit));
    runningRef.current = false; setBusy(false);
  }, [pitCenters, reduced]);

  // Tour de l'IA, selon la difficulté.
  useEffect(() => {
    if (state.turn === "ai" && !result.over && !runningRef.current) {
      const t = setTimeout(() => {
        const d = diffRef.current;
        const mv = d === "easy" ? aiChooseMove(state.board)
          : d === "medium" ? aiSearchMove(state, 5, 250)
            : aiSearchMove(state, 11, 650);
        animateMove(state, mv);
      }, 460);
      return () => clearTimeout(t);
    }
  }, [state, result.over, animateMove]);

  function handleHumanClick(pit) {
    audio();
    if (state.turn !== "human" || result.over || busy) return;
    if (!isMoveValid(state.board, "human", pit)) return;
    animateMove(state, pit);
  }
  function reset() {
    cancelled.current = true; runningRef.current = false;
    setBusy(false); setSource(-1); setDropPit(-1); setCaptureFlash([]);
    setFlying({ x: 0, y: 0, visible: false, t: 0 });
    setPop({ human: null, ai: null }); setLastMove({ pit: -1, captured: [] });
    setDisplayBoard(Array(12).fill(4)); setState(makeStart());
  }
  function pickDifficulty(id) { if (busy || state.turn === "ai") return; setDifficulty(id); diffRef.current = id; }
  function toggleSound() { audio(); const v = !soundOn; setSoundOn(v); soundRef.current = v; if (v) sfx.drop(0); }

  const pad = 17, gap = 8;
  const boardW = vw - 46;
  const diameter = Math.floor((boardW - pad * 2 - gap * 5) / 6);
  const flySize = Math.max(9, diameter * 0.26);
  const controlsLocked = busy || state.turn === "ai";

  const statusText = result.over ? "Partie terminée" : state.turn === "human" ? (busy ? "Semis en cours" : "À vous de jouer") : "L'IA réfléchit";

  // Particules de fin de partie.
  const confetti = useMemo(() => {
    if (!result.over) return [];
    const cols = result.winner === "human" ? [C.human, "#f3e9d6", "#f4c66b"] : result.winner === "ai" ? [C.ai, "#f3e9d6", "#8fd6cd"] : [C.muted, "#f3e9d6"];
    return Array.from({ length: 22 }).map((_, i) => ({ id: i, left: Math.random() * 100, delay: Math.random() * 0.5, dur: 1.4 + Math.random() * 1.1, size: 7 + Math.random() * 7, col: cols[i % cols.length], rot: Math.random() * 360 }));
  }, [result.over, result.winner]);

  const renderRow = (row, side) => (
    <div style={{ display: "flex", gap, justifyContent: "center" }}>
      {row.map((idx) => (
        <Pit key={idx} index={idx} count={displayBoard[idx]} diameter={diameter} side={side}
          isPlayable={side === "human" && validHuman.includes(idx)} isLast={lastMove.pit === idx}
          isCapturing={captureFlash.includes(idx)} isSource={source === idx} isDrop={dropPit === idx}
          pitRef={(el) => (pitRefs.current[idx] = el)} onClick={() => handleHumanClick(idx)} />
      ))}
    </div>
  );

  return (
    <div style={{ minHeight: "100vh", width: "100%", position: "relative", display: "flex", justifyContent: "center", fontFamily: C.sans, overflowX: "hidden", background: "#0c0805" }}>
      {/* Paysage évoquant le lac de Buyo : lumière dorée de fin d'après-midi sur l'eau, rives forestières */}
      <div aria-hidden style={{ position: "fixed", inset: 0, zIndex: 0, overflow: "hidden" }}>
        {/* Ciel */}
        <div style={{ position: "absolute", inset: 0, background: "linear-gradient(180deg, #234e57 0%, #356a64 24%, #7e7a52 46%, #c98a4a 58%, #f0b257 66%)" }} />
        {/* Soleil bas sur l'horizon */}
        <div style={{ position: "absolute", left: "50%", top: "47%", width: "min(70vw, 360px)", height: "min(70vw, 360px)", transform: "translate(-50%,-50%)", borderRadius: "50%", background: "radial-gradient(circle, rgba(255,239,188,.95), rgba(255,201,112,.55) 34%, rgba(255,180,90,0) 70%)", filter: "blur(2px)" }} />
        {/* Collines forestières lointaines (voilées par la brume) */}
        <div style={{ position: "absolute", left: "-6%", right: "-6%", top: "55%", height: "13%", filter: "blur(2px)", opacity: 0.85, background: "radial-gradient(58% 150% at 22% 100%, #2a5040 0 60%, transparent 72%), radial-gradient(64% 150% at 56% 100%, #234a3c 0 60%, transparent 74%), radial-gradient(60% 150% at 86% 100%, #2c5343 0 60%, transparent 72%)" }} />
        {/* Forêt de la rive (plus sombre, plus nette) */}
        <div style={{ position: "absolute", left: "-6%", right: "-6%", top: "61%", height: "9%", background: "radial-gradient(50% 150% at 14% 100%, #163320 0 62%, transparent 74%), radial-gradient(55% 150% at 44% 100%, #102a1a 0 62%, transparent 75%), radial-gradient(52% 150% at 74% 100%, #14311e 0 62%, transparent 74%), radial-gradient(50% 150% at 96% 100%, #0f2818 0 62%, transparent 74%)" }} />
        {/* Lac */}
        <div style={{ position: "absolute", left: 0, right: 0, top: "66%", bottom: 0, background: "linear-gradient(180deg, #e0a258 0%, #9a8c55 12%, #5c8069 30%, #2f6059 58%, #173f3c 100%)" }} />
        {/* Reflet du soleil sur l'eau */}
        <div style={{ position: "absolute", left: "50%", transform: "translateX(-50%)", top: "66%", bottom: 0, width: "min(40vw, 150px)", background: "linear-gradient(180deg, rgba(255,228,156,.65), rgba(255,210,120,0) 65%)", filter: "blur(3px)" }} />
        {/* Ondulations */}
        <div style={{ position: "absolute", left: 0, right: 0, top: "66%", bottom: 0, opacity: 0.5, background: "repeating-linear-gradient(180deg, rgba(255,255,255,.06) 0 1px, transparent 1px 8px)" }} />
        {/* Voile pour la lisibilité du texte */}
        <div style={{ position: "absolute", inset: 0, background: "linear-gradient(180deg, rgba(8,6,3,.5) 0%, rgba(8,6,3,.1) 20%, rgba(8,6,3,.08) 56%, rgba(8,6,3,.42) 100%)" }} />
      </div>
      <style>{`
        @keyframes blink { 0%,100%{opacity:.3} 50%{opacity:1} }
        @keyframes scorepop { 0%{transform:scale(.6);opacity:.4} 60%{transform:scale(1.18)} 100%{transform:scale(1)} }
        @keyframes floatUp { 0%{transform:translateY(4px);opacity:0} 25%{opacity:1} 100%{transform:translateY(-26px);opacity:0} }
        @keyframes cardin { 0%{transform:scale(.85);opacity:0} 60%{transform:scale(1.04)} 100%{transform:scale(1);opacity:1} }
        @keyframes fall { 0%{transform:translateY(-20px) rotate(0);opacity:0} 12%{opacity:1} 100%{transform:translateY(70vh) rotate(540deg);opacity:0} }
        .pit:active.playable{ transform: scale(.95) !important; }
        @media (prefers-reduced-motion: reduce){ *{animation:none !important} .pit{transition:none !important} }
      `}</style>

      <div style={{ position: "relative", zIndex: 1, width: "100%", maxWidth: 480, padding: "20px 12px 30px", boxSizing: "border-box" }}>
        <div style={{ borderRadius: 26, padding: "16px 11px 18px", background: "rgba(16,11,6,.5)", border: "1px solid rgba(255,225,180,.13)", boxShadow: "0 20px 52px rgba(0,0,0,.45)", backdropFilter: "blur(8px)", WebkitBackdropFilter: "blur(8px)" }}>
        <div style={{ textAlign: "center", marginBottom: 15 }}>
          <h1 style={{ margin: 0, fontFamily: C.serif, fontWeight: 600, fontSize: 31, color: C.text, letterSpacing: ".01em" }}>Awalé</h1>
          <p style={{ margin: "3px 0 0", fontSize: 12.5, color: C.muted, letterSpacing: ".14em", textTransform: "uppercase" }}>Vous contre l'IA · 25 graines pour gagner</p>
        </div>

        <div style={{ display: "flex", gap: 10, marginBottom: 12 }}>
          <ScoreCard label="Vous" score={state.humanScore} color={C.human} active={state.turn === "human" && !result.over} pop={pop.human} />
          <ScoreCard label="IA" score={state.aiScore} color={C.ai} active={state.turn === "ai" && !result.over} pop={pop.ai} />
        </div>

        {/* Réglages : difficulté + son */}
        <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 12 }}>
          <div style={{ display: "flex", flex: 1, padding: 3, gap: 3, borderRadius: 12, background: "rgba(255,255,255,.04)", border: "1px solid rgba(255,255,255,.08)", opacity: controlsLocked ? 0.5 : 1 }}>
            {DIFFS.map((d) => {
              const on = difficulty === d.id;
              return (
                <button key={d.id} onClick={() => pickDifficulty(d.id)} disabled={controlsLocked}
                  style={{ flex: 1, padding: "7px 0", borderRadius: 9, border: "none", cursor: controlsLocked ? "default" : "pointer",
                    fontFamily: C.sans, fontSize: 14, fontWeight: on ? 700 : 500,
                    color: on ? "#1a120a" : C.muted, background: on ? C.human : "transparent",
                    boxShadow: on ? "0 2px 8px rgba(232,163,61,.35)" : "none", transition: "all .2s ease", WebkitTapHighlightColor: "transparent" }}>
                  {d.label}
                </button>
              );
            })}
          </div>
          <button onClick={toggleSound} aria-label={soundOn ? "Couper le son" : "Activer le son"}
            style={{ width: 42, height: 42, flexShrink: 0, display: "flex", alignItems: "center", justifyContent: "center",
              borderRadius: 12, cursor: "pointer", background: "rgba(255,255,255,.04)", border: `1px solid ${soundOn ? C.human + "66" : "rgba(255,255,255,.08)"}`, WebkitTapHighlightColor: "transparent" }}>
            <SpeakerIcon on={soundOn} />
          </button>
        </div>

        <div style={{ display: "flex", justifyContent: "center", marginBottom: 13 }}>
          <div style={{ display: "inline-flex", alignItems: "center", gap: 8, padding: "7px 17px", borderRadius: 999, background: "rgba(255,255,255,.05)", border: `1px solid ${result.over ? "rgba(255,255,255,.15)" : (state.turn === "human" ? C.human : C.ai) + "66"}`, color: C.text, fontSize: 15, fontWeight: 500 }}>
            <span style={{ width: 8, height: 8, borderRadius: "50%", background: result.over ? C.muted : state.turn === "human" ? C.human : C.ai, animation: result.over ? "none" : "blink 1.2s infinite" }} />
            {statusText}{!result.over && state.turn === "ai" && <span style={{ animation: "blink 1s infinite" }}>…</span>}
          </div>
        </div>

        <div style={{ position: "relative", transition: "opacity .5s ease, transform .5s ease", opacity: mounted ? 1 : 0, transform: mounted ? "translateY(0)" : "translateY(10px)" }}>
          <div ref={boardRef} style={{
            position: "relative", width: boardW, margin: "0 auto", padding: pad, boxSizing: "border-box", borderRadius: 28,
            background: `repeating-linear-gradient(96deg, rgba(255,225,180,.05) 0 2px, rgba(0,0,0,0) 2px 9px), repeating-linear-gradient(91deg, rgba(40,18,4,.18) 0 1px, rgba(0,0,0,0) 1px 14px), linear-gradient(158deg, #8a572c, #5e3818 50%, #3a2010)`,
            boxShadow: "0 22px 48px rgba(0,0,0,.55), 0 2px 0 rgba(255,210,150,.12), inset 0 2px 0 rgba(255,220,170,.22), inset 0 -14px 30px rgba(0,0,0,.45), inset 0 0 0 1px rgba(20,10,2,.5)",
          }}>
            {renderRow(TOP_ROW, "ai")}
            <div style={{ height: 2, margin: `${gap + 3}px ${diameter / 2}px`, borderRadius: 2, background: "linear-gradient(90deg, rgba(0,0,0,.05), rgba(0,0,0,.35), rgba(0,0,0,.05))", boxShadow: "0 1px 0 rgba(255,210,150,.14)" }} />
            {renderRow(BOTTOM_ROW, "human")}
            <div style={{ display: "flex", justifyContent: "space-between", marginTop: 9, padding: "0 5px", fontSize: 12, fontWeight: 600, letterSpacing: ".16em", textTransform: "uppercase" }}>
              <span style={{ color: C.ai }}>Camp IA</span><span style={{ color: C.human }}>Votre camp</span>
            </div>
            {flying.visible && (
              <div style={{ position: "absolute", left: flying.x, top: flying.y, width: flySize, height: flySize * 0.92, marginLeft: -flySize / 2, marginTop: -flySize * 0.46, borderRadius: "50%", zIndex: 5, pointerEvents: "none", background: SEED_TONES[0], boxShadow: "0 2px 6px rgba(0,0,0,.6), 0 0 12px rgba(255,210,140,.55), inset 0 1px 1px rgba(255,255,255,.5)", transition: flying.t ? `left ${flying.t}ms cubic-bezier(.4,.05,.5,1), top ${flying.t}ms cubic-bezier(.45,-.25,.55,1.25)` : "none" }} />
            )}
          </div>
          {(busy || (state.turn === "ai" && !result.over)) && <div style={{ position: "absolute", inset: 0, borderRadius: 28, background: "rgba(12,8,5,.14)", pointerEvents: "none" }} />}
        </div>

        <div style={{ textAlign: "center", marginTop: 18 }}>
          <button onClick={reset} style={{ fontFamily: C.sans, fontSize: 15.5, fontWeight: 600, color: C.text, padding: "11px 26px", borderRadius: 12, cursor: "pointer", background: "rgba(255,255,255,.05)", border: "1px solid rgba(255,255,255,.14)", WebkitTapHighlightColor: "transparent" }}>Nouvelle partie</button>
        </div>
        </div>

        {result.over && (
          <div style={{ position: "fixed", inset: 0, display: "flex", alignItems: "center", justifyContent: "center", background: "rgba(8,5,3,.74)", padding: 24, zIndex: 50, overflow: "hidden" }}>
            {confetti.map((p) => (
              <span key={p.id} style={{ position: "absolute", top: -20, left: `${p.left}%`, width: p.size, height: p.size * 0.92, borderRadius: "50%", background: p.col, transform: `rotate(${p.rot}deg)`, animation: `fall ${p.dur}s ease-in ${p.delay}s forwards`, pointerEvents: "none" }} />
            ))}
            <div style={{ position: "relative", maxWidth: 340, width: "100%", textAlign: "center", padding: "32px 26px", borderRadius: 24, background: "linear-gradient(160deg, #8a572c, #3a2010)", border: "1px solid rgba(255,210,150,.22)", boxShadow: "0 26px 64px rgba(0,0,0,.62), inset 0 2px 0 rgba(255,220,170,.2)", animation: "cardin .45s cubic-bezier(.34,1.56,.64,1)" }}>
              <div style={{ fontSize: 11.5, letterSpacing: ".2em", textTransform: "uppercase", color: "#e9d4ab" }}>{result.reason === "25" ? "Seuil de 25 atteint" : "Plus de coup possible"}</div>
              <h2 style={{ margin: "9px 0 4px", fontFamily: C.serif, fontWeight: 600, fontSize: 29, color: C.text }}>{result.winner === "human" ? "Vous gagnez" : result.winner === "ai" ? "L'IA gagne" : "Égalité"}</h2>
              <div style={{ fontFamily: C.serif, fontSize: 23, margin: "10px 0 24px" }}>
                <span style={{ color: C.human }}>{result.finalHuman}</span><span style={{ color: "#e9d4ab", margin: "0 11px" }}>–</span><span style={{ color: C.ai }}>{result.finalAi}</span>
              </div>
              <button onClick={reset} style={{ fontFamily: C.sans, fontSize: 15, fontWeight: 600, color: "#1a120a", padding: "12px 30px", borderRadius: 12, cursor: "pointer", border: "none", background: C.human, boxShadow: `0 6px 20px ${C.humanGlow}`, WebkitTapHighlightColor: "transparent" }}>Rejouer</button>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
