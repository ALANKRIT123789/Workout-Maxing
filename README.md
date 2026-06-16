# Workout-Maxing
import { useState, useEffect, useRef } from "react";

// ─── CONSTANTS ───────────────────────────────────────────────────────────────
const DAILY_QUEST = [
  { id: 1, name: "Push-ups", target: 100, unit: "reps", xp: 300, icon: "💪" },
  { id: 2, name: "Sit-ups", target: 100, unit: "reps", xp: 300, icon: "🔥" },
  { id: 3, name: "Squats", target: 100, unit: "reps", xp: 300, icon: "⚡" },
  { id: 4, name: "Running", target: 10, unit: "km", xp: 500, icon: "🏃" },
];

const RANKS = [
  { rank: "E", minXP: 0, color: "#888", glow: "#888" },
  { rank: "D", minXP: 1000, color: "#4ade80", glow: "#4ade80" },
  { rank: "C", minXP: 3000, color: "#60a5fa", glow: "#60a5fa" },
  { rank: "B", minXP: 7000, color: "#a78bfa", glow: "#a78bfa" },
  { rank: "A", minXP: 15000, color: "#f59e0b", glow: "#f59e0b" },
  { rank: "S", minXP: 30000, color: "#8b5cf6", glow: "#8b5cf6" },
  { rank: "SS", minXP: 60000, color: "#ec4899", glow: "#ec4899" },
];

const WARNING_MESSAGES = [
  { title: "⚠ SYSTEM WARNING ⚠", body: "You have failed to complete the daily quest. The penalty dungeon awaits.", sub: "Failure is not an option, Hunter." },
  { title: "☠ CRITICAL ALERT ☠", body: "Penalty Quest Activated. Complete your training or face the consequences.", sub: "The System does not forgive weakness." },
  { title: "🚨 PENALTY NOTICE 🚨", body: "An unfinished hunter is a dead hunter. Resume your training NOW.", sub: "Only the strong survive." },
];

function getRank(xp) {
  let current = RANKS[0];
  for (const r of RANKS) {
    if (xp >= r.minXP) current = r;
  }
  return current;
}

function getNextRank(xp) {
  for (let i = 0; i < RANKS.length; i++) {
    if (xp < RANKS[i].minXP) return RANKS[i];
  }
  return null;
}

// ─── PARTICLE BACKGROUND ─────────────────────────────────────────────────────
function Particles() {
  const canvasRef = useRef(null);
  useEffect(() => {
    const canvas = canvasRef.current;
    const ctx = canvas.getContext("2d");
    let animId;
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight;

    const particles = Array.from({ length: 60 }, () => ({
      x: Math.random() * canvas.width,
      y: Math.random() * canvas.height,
      r: Math.random() * 1.5 + 0.3,
      dx: (Math.random() - 0.5) * 0.3,
      dy: -Math.random() * 0.4 - 0.1,
      alpha: Math.random() * 0.6 + 0.1,
    }));

    function draw() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      for (const p of particles) {
        ctx.beginPath();
        ctx.arc(p.x, p.y, p.r, 0, Math.PI * 2);
        ctx.fillStyle = `rgba(139, 92, 246, ${p.alpha})`;
        ctx.fill();
        p.x += p.dx;
        p.y += p.dy;
        if (p.y < -5) { p.y = canvas.height + 5; p.x = Math.random() * canvas.width; }
        if (p.x < -5) p.x = canvas.width + 5;
        if (p.x > canvas.width + 5) p.x = -5;
      }
      animId = requestAnimationFrame(draw);
    }
    draw();
    const onResize = () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; };
    window.addEventListener("resize", onResize);
    return () => { cancelAnimationFrame(animId); window.removeEventListener("resize", onResize); };
  }, []);
  return <canvas ref={canvasRef} style={{ position: "fixed", inset: 0, zIndex: 0, pointerEvents: "none" }} />;
}

// ─── MAIN APP ─────────────────────────────────────────────────────────────────
export default function App() {
  const [xp, setXp] = useState(0);
  const [progress, setProgress] = useState({ 1: 0, 2: 0, 3: 0, 4: 0 });
  const [completed, setCompleted] = useState({});
  const [warning, setWarning] = useState(null);
  const [showComplete, setShowComplete] = useState(false);
  const [activeTab, setActiveTab] = useState("quest");
  const [inputValues, setInputValues] = useState({ 1: "", 2: "", 3: "", 4: "" });
  const [log, setLog] = useState([]);
  const [streakDays, setStreakDays] = useState(3);
  const [pulseRank, setPulseRank] = useState(false);
  const [questStarted, setQuestStarted] = useState(false);
  const [warningIdx, setWarningIdx] = useState(0);
  const warningTimer = useRef(null);

  const rank = getRank(xp);
  const nextRank = getNextRank(xp);
  const totalXP = nextRank ? nextRank.minXP : RANKS[RANKS.length - 1].minXP;
  const currentXP = nextRank ? xp - getRank(xp).minXP : xp;
  const xpForNext = nextRank ? nextRank.minXP - getRank(xp).minXP : 1;
  const xpPct = Math.min((currentXP / xpForNext) * 100, 100);

  const allDone = DAILY_QUEST.every((q) => completed[q.id]);

  // Trigger warning if quest started but not all done after 30s of inactivity
  useEffect(() => {
    if (questStarted && !allDone) {
      warningTimer.current = setTimeout(() => {
        setWarningIdx(Math.floor(Math.random() * WARNING_MESSAGES.length));
        setWarning(true);
      }, 30000);
    }
    return () => clearTimeout(warningTimer.current);
  }, [questStarted, allDone, progress]);

  function handleLog(id, value) {
    const q = DAILY_QUEST.find((x) => x.id === id);
    const num = parseFloat(value);
    if (isNaN(num) || num <= 0) return;
    const newProg = Math.min((progress[id] || 0) + num, q.target);
    const newProgress = { ...progress, [id]: newProg };
    setProgress(newProgress);
    setInputValues((v) => ({ ...v, [id]: "" }));
    setQuestStarted(true);
    clearTimeout(warningTimer.current);

    const entry = `[${new Date().toLocaleTimeString()}] ${q.icon} ${q.name}: +${num} ${q.unit} (${newProg}/${q.target})`;
    setLog((l) => [entry, ...l].slice(0, 20));

    if (newProg >= q.target && !completed[id]) {
      const newCompleted = { ...completed, [id]: true };
      setCompleted(newCompleted);
      const gained = q.xp;
      setXp((x) => x + gained);
      setPulseRank(true);
      setTimeout(() => setPulseRank(false), 1500);
      if (DAILY_QUEST.every((dq) => newCompleted[dq.id])) {
        setTimeout(() => setShowComplete(true), 400);
      }
    }
  }

  function handleAbandon() {
    setWarningIdx(Math.floor(Math.random() * WARNING_MESSAGES.length));
    setWarning(true);
  }

  function dismissWarning() { setWarning(false); }

  function resetQuest() {
    setProgress({ 1: 0, 2: 0, 3: 0, 4: 0 });
    setCompleted({});
    setShowComplete(false);
    setWarning(false);
    setQuestStarted(false);
    setLog([]);
    setInputValues({ 1: "", 2: "", 3: "", 4: "" });
  }

  const wMsg = WARNING_MESSAGES[warningIdx];
  const totalDayXP = DAILY_QUEST.reduce((s, q) => s + q.xp, 0);
  const earnedToday = DAILY_QUEST.filter((q) => completed[q.id]).reduce((s, q) => s + q.xp, 0);

  return (
    <div style={styles.root}>
      <style>{CSS}</style>
      <Particles />

      {/* ── WARNING MODAL ── */}
      {warning && (
        <div style={styles.modalOverlay} onClick={dismissWarning}>
          <div style={styles.warningModal} onClick={(e) => e.stopPropagation()} className="warnPulse">
            <div style={styles.warningSkull}>☠</div>
            <div style={styles.warningTitle}>{wMsg.title}</div>
            <div style={styles.warningSep} />
            <p style={styles.warningBody}>{wMsg.body}</p>
            <p style={styles.warningSub}>{wMsg.sub}</p>
            <div style={styles.warningButtons}>
              <button style={styles.warnAccept} onClick={dismissWarning} className="glowBtn">
                RESUME TRAINING
              </button>
              <button style={styles.warnIgnore} onClick={dismissWarning}>
                ACKNOWLEDGE
              </button>
            </div>
          </div>
        </div>
      )}

      {/* ── QUEST COMPLETE MODAL ── */}
      {showComplete && (
        <div style={styles.modalOverlay}>
          <div style={styles.completeModal} className="completeAnim">
            <div style={styles.completeGlow} />
            <div style={styles.completeTitle}>QUEST COMPLETE</div>
            <div style={styles.completeStar}>✦</div>
            <p style={styles.completeXP}>+{totalDayXP} XP EARNED</p>
            <p style={styles.completeRank}>Current Rank: <span style={{ color: rank.color }}>{rank.rank}</span></p>
            <p style={styles.completeStreak}>🔥 {streakDays} Day Streak Maintained</p>
            <button style={styles.completeBtn} onClick={resetQuest} className="glowBtn">
              NEW DAY — START AGAIN
            </button>
          </div>
        </div>
      )}

      {/* ── HEADER ── */}
      <header style={styles.header}>
        <div style={styles.headerLeft}>
          <div style={styles.logo}>SYSTEM</div>
          <div style={styles.logoSub}>HUNTER WORKOUT</div>
        </div>
        <div style={styles.headerRight}>
          <div style={styles.streakBadge}>🔥 {streakDays}d</div>
          <div
            style={{ ...styles.rankBadge, color: rank.color, boxShadow: `0 0 12px ${rank.glow}40, 0 0 24px ${rank.glow}20` }}
            className={pulseRank ? "rankPulse" : ""}
          >
            {rank.rank}
          </div>
        </div>
      </header>

      {/* ── XP BAR ── */}
      <div style={styles.xpSection}>
        <div style={styles.xpTop}>
          <span style={styles.xpLabel}>HUNTER XP</span>
          <span style={styles.xpValue}>{xp.toLocaleString()} XP</span>
        </div>
        <div style={styles.xpTrack}>
          <div style={{ ...styles.xpFill, width: `${xpPct}%`, background: `linear-gradient(90deg, ${rank.color}80, ${rank.color})` }} />
        </div>
        <div style={styles.xpBottom}>
          <span style={{ color: "#555" }}>Rank {rank.rank}</span>
          {nextRank && <span style={{ color: "#555" }}>{(nextRank.minXP - xp).toLocaleString()} XP to Rank {nextRank.rank}</span>}
          {!nextRank && <span style={{ color: rank.color }}>MAX RANK</span>}
        </div>
      </div>

      {/* ── TABS ── */}
      <div style={styles.tabs}>
        {["quest", "stats", "log"].map((t) => (
          <button
            key={t}
            style={{ ...styles.tab, ...(activeTab === t ? styles.tabActive : {}) }}
            onClick={() => setActiveTab(t)}
          >
            {t === "quest" ? "⚔ DAILY QUEST" : t === "stats" ? "📊 STATS" : "📜 LOG"}
          </button>
        ))}
      </div>

      {/* ── QUEST TAB ── */}
      {activeTab === "quest" && (
        <div style={styles.content}>
          <div style={styles.questHeader}>
            <div style={styles.questTitle}>☰ DAILY TRAINING QUEST</div>
            <div style={styles.questXP}>{earnedToday}/{totalDayXP} XP</div>
          </div>
          <div style={styles.questDivider} />

          {DAILY_QUEST.map((q) => {
            const prog = progress[q.id] || 0;
            const pct = Math.min((prog / q.target) * 100, 100);
            const done = !!completed[q.id];
            return (
              <div key={q.id} style={{ ...styles.questCard, ...(done ? styles.questCardDone : {}) }} className="questCard">
                <div style={styles.questCardTop}>
                  <div style={styles.questIcon}>{q.icon}</div>
                  <div style={styles.questInfo}>
                    <div style={styles.questName}>{q.name}</div>
                    <div style={styles.questTarget}>{prog} / {q.target} {q.unit}</div>
                  </div>
                  <div style={styles.questXPBadge}>+{q.xp} XP</div>
                  {done && <div style={styles.checkMark}>✔</div>}
                </div>
                <div style={styles.progressTrack}>
                  <div
                    style={{
                      ...styles.progressFill,
                      width: `${pct}%`,
                      background: done
                        ? "linear-gradient(90deg, #4ade8060, #4ade80)"
                        : "linear-gradient(90deg, #6d28d9, #8b5cf6)",
                    }}
                  />
                </div>
                {!done && (
                  <div style={styles.inputRow}>
                    <input
                      type="number"
                      min="0"
                      placeholder={`Add ${q.unit}…`}
                      value={inputValues[q.id]}
                      onChange={(e) => setInputValues((v) => ({ ...v, [q.id]: e.target.value }))}
                      onKeyDown={(e) => e.key === "Enter" && handleLog(q.id, inputValues[q.id])}
                      style={styles.input}
                      className="sysInput"
                    />
                    <button
                      style={styles.logBtn}
                      onClick={() => handleLog(q.id, inputValues[q.id])}
                      className="glowBtn"
                    >
                      LOG
                    </button>
                  </div>
                )}
              </div>
            );
          })}

          <div style={styles.abandonSection}>
            <button style={styles.abandonBtn} onClick={handleAbandon}>
              ☠ ABANDON QUEST
            </button>
          </div>
        </div>
      )}

      {/* ── STATS TAB ── */}
      {activeTab === "stats" && (
        <div style={styles.content}>
          <div style={styles.statsGrid}>
            <StatCard label="TOTAL XP" value={xp.toLocaleString()} sub="Experience Points" color="#8b5cf6" />
            <StatCard label="RANK" value={rank.rank} sub="Current Class" color={rank.color} />
            <StatCard label="STREAK" value={`${streakDays}d`} sub="Days Active" color="#f59e0b" />
            <StatCard label="TODAY" value={`${earnedToday}`} sub="XP Earned" color="#4ade80" />
          </div>

          <div style={styles.rankTable}>
            <div style={styles.rankTableTitle}>RANK PROGRESSION</div>
            {RANKS.map((r) => (
              <div key={r.rank} style={{ ...styles.rankRow, ...(rank.rank === r.rank ? styles.rankRowActive : {}) }}>
                <div style={{ ...styles.rankLetter, color: r.color, textShadow: `0 0 8px ${r.glow}` }}>{r.rank}</div>
                <div style={styles.rankXP}>{r.minXP.toLocaleString()} XP</div>
                {xp >= r.minXP && <div style={{ color: "#4ade80", fontSize: 12 }}>UNLOCKED ✔</div>}
                {xp < r.minXP && <div style={{ color: "#444", fontSize: 12 }}>{(r.minXP - xp).toLocaleString()} XP away</div>}
              </div>
            ))}
          </div>
        </div>
      )}

      {/* ── LOG TAB ── */}
      {activeTab === "log" && (
        <div style={styles.content}>
          <div style={styles.logTitle}>SYSTEM ACTIVITY LOG</div>
          {log.length === 0 && (
            <div style={styles.emptyLog}>No activity yet. Begin your training, Hunter.</div>
          )}
          {log.map((entry, i) => (
            <div key={i} style={{ ...styles.logEntry, opacity: 1 - i * 0.04 }}>
              <span style={styles.logBullet}>▶</span> {entry}
            </div>
          ))}
        </div>
      )}

      {/* ── BOTTOM NAV HINT ── */}
      <div style={styles.footer}>
        THE SYSTEM WATCHES. NEVER STOP TRAINING.
      </div>
    </div>
  );
}

function StatCard({ label, value, sub, color }) {
  return (
    <div style={{ ...styles.statCard, borderColor: `${color}30` }}>
      <div style={{ ...styles.statValue, color }}>{value}</div>
      <div style={styles.statLabel}>{label}</div>
      <div style={styles.statSub}>{sub}</div>
    </div>
  );
}

// ─── STYLES ──────────────────────────────────────────────────────────────────
const styles = {
  root: {
    minHeight: "100vh",
    background: "#050509",
    color: "#e2e8f0",
    fontFamily: "'Share Tech Mono', 'Courier New', monospace",
    position: "relative",
    overflowX: "hidden",
    paddingBottom: 60,
  },
  header: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    padding: "16px 20px 12px",
    borderBottom: "1px solid #1a1a2e",
    position: "relative",
    zIndex: 10,
    background: "rgba(5,5,9,0.9)",
  },
  headerLeft: { display: "flex", flexDirection: "column" },
  logo: {
    fontSize: 22,
    fontWeight: 900,
    letterSpacing: 6,
    color: "#8b5cf6",
    textShadow: "0 0 20px #8b5cf6, 0 0 40px #6d28d940",
    lineHeight: 1,
  },
  logoSub: { fontSize: 9, letterSpacing: 4, color: "#6d28d9", marginTop: 2 },
  headerRight: { display: "flex", alignItems: "center", gap: 10 },
  streakBadge: {
    background: "#1a0f00",
    border: "1px solid #92400e",
    color: "#f59e0b",
    borderRadius: 6,
    padding: "4px 10px",
    fontSize: 12,
    letterSpacing: 1,
  },
  rankBadge: {
    width: 44,
    height: 44,
    borderRadius: "50%",
    border: "2px solid currentColor",
    display: "flex",
    alignItems: "center",
    justifyContent: "center",
    fontSize: 16,
    fontWeight: 900,
    letterSpacing: 1,
    background: "#0d0d1a",
    transition: "box-shadow 0.3s",
  },
  xpSection: {
    padding: "14px 20px",
    position: "relative",
    zIndex: 10,
    background: "rgba(5,5,9,0.7)",
    borderBottom: "1px solid #1a1a2e",
  },
  xpTop: { display: "flex", justifyContent: "space-between", marginBottom: 6 },
  xpLabel: { fontSize: 10, letterSpacing: 3, color: "#6d28d9" },
  xpValue: { fontSize: 11, color: "#a78bfa", letterSpacing: 1 },
  xpTrack: {
    height: 6,
    background: "#1a1a2e",
    borderRadius: 3,
    overflow: "hidden",
    border: "1px solid #2d2d4e",
  },
  xpFill: {
    height: "100%",
    borderRadius: 3,
    transition: "width 0.6s cubic-bezier(0.4,0,0.2,1)",
    boxShadow: "0 0 8px #8b5cf6",
  },
  xpBottom: {
    display: "flex",
    justifyContent: "space-between",
    marginTop: 5,
    fontSize: 9,
    letterSpacing: 1,
  },
  tabs: {
    display: "flex",
    gap: 0,
    borderBottom: "1px solid #1a1a2e",
    position: "relative",
    zIndex: 10,
    background: "rgba(5,5,9,0.85)",
  },
  tab: {
    flex: 1,
    padding: "11px 4px",
    background: "transparent",
    border: "none",
    color: "#4a4a6a",
    fontSize: 10,
    letterSpacing: 2,
    cursor: "pointer",
    transition: "all 0.2s",
    borderBottom: "2px solid transparent",
    fontFamily: "inherit",
  },
  tabActive: {
    color: "#a78bfa",
    borderBottom: "2px solid #8b5cf6",
    background: "#0d0d1a",
  },
  content: {
    padding: "16px 16px 24px",
    position: "relative",
    zIndex: 10,
    maxWidth: 600,
    margin: "0 auto",
  },
  questHeader: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 4,
  },
  questTitle: { fontSize: 11, letterSpacing: 3, color: "#8b5cf6" },
  questXP: { fontSize: 11, color: "#6d28d9", letterSpacing: 1 },
  questDivider: {
    height: 1,
    background: "linear-gradient(90deg, #8b5cf6, transparent)",
    marginBottom: 14,
  },
  questCard: {
    background: "rgba(13,13,26,0.9)",
    border: "1px solid #1e1e3a",
    borderRadius: 8,
    padding: "14px 14px 10px",
    marginBottom: 12,
    transition: "border-color 0.3s, box-shadow 0.3s",
    backdropFilter: "blur(4px)",
  },
  questCardDone: {
    borderColor: "#4ade8030",
    boxShadow: "0 0 16px #4ade8010",
    opacity: 0.85,
  },
  questCardTop: {
    display: "flex",
    alignItems: "center",
    gap: 12,
    marginBottom: 10,
  },
  questIcon: { fontSize: 22, minWidth: 28 },
  questInfo: { flex: 1 },
  questName: { fontSize: 13, fontWeight: 700, letterSpacing: 2, color: "#c4b5fd", marginBottom: 2 },
  questTarget: { fontSize: 10, color: "#6d28d9", letterSpacing: 1 },
  questXPBadge: {
    fontSize: 10,
    color: "#a78bfa",
    background: "#1a1a2e",
    border: "1px solid #6d28d9",
    borderRadius: 4,
    padding: "2px 7px",
    letterSpacing: 1,
    whiteSpace: "nowrap",
  },
  checkMark: { color: "#4ade80", fontSize: 18, marginLeft: 4 },
  progressTrack: {
    height: 4,
    background: "#1a1a2e",
    borderRadius: 2,
    overflow: "hidden",
    marginBottom: 10,
  },
  progressFill: {
    height: "100%",
    borderRadius: 2,
    transition: "width 0.5s cubic-bezier(0.4,0,0.2,1)",
    boxShadow: "0 0 6px #8b5cf6",
  },
  inputRow: { display: "flex", gap: 8 },
  input: {
    flex: 1,
    background: "#0a0a16",
    border: "1px solid #2d2d4e",
    borderRadius: 6,
    color: "#e2e8f0",
    fontFamily: "inherit",
    fontSize: 13,
    pa
