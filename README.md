import { useState, useEffect, useRef } from "react";

// ─── Walrus Testnet Config ───────────────────────────────────────────────────
const WALRUS_PUBLISHER = "https://publisher.walrus-testnet.walrus.space";
const WALRUS_AGGREGATOR = "https://aggregator.walrus-testnet.walrus.space";
const STORAGE_KEY = "wc2026_grudge_agent";

// ─── World Cup 2026 Match Data ───────────────────────────────────────────────
const GROUPS = {
  A: ["Qatar", "Ecuador", "Senegal", "Netherlands"],
  B: ["England", "Iran", "USA", "Wales"],
  C: ["Argentina", "Saudi Arabia", "Mexico", "Poland"],
  D: ["France", "Australia", "Denmark", "Tunisia"],
  E: ["Spain", "Costa Rica", "Germany", "Japan"],
  F: ["Belgium", "Canada", "Morocco", "Croatia"],
  G: ["Brazil", "Serbia", "Switzerland", "Cameroon"],
  H: ["Portugal", "Ghana", "Uruguay", "South Korea"],
};

const FLAGS = {
  Qatar: "🇶🇦", Ecuador: "🇪🇨", Senegal: "🇸🇳", Netherlands: "🇳🇱",
  England: "🏴󠁧󠁢󠁥󠁮󠁧󠁿", Iran: "🇮🇷", USA: "🇺🇸", Wales: "🏴󠁧󠁢󠁷󠁬󠁳󠁿",
  Argentina: "🇦🇷", "Saudi Arabia": "🇸🇦", Mexico: "🇲🇽", Poland: "🇵🇱",
  France: "🇫🇷", Australia: "🇦🇺", Denmark: "🇩🇰", Tunisia: "🇹🇳",
  Spain: "🇪🇸", "Costa Rica": "🇨🇷", Germany: "🇩🇪", Japan: "🇯🇵",
  Belgium: "🇧🇪", Canada: "🇨🇦", Morocco: "🇲🇦", Croatia: "🇭🇷",
  Brazil: "🇧🇷", Serbia: "🇷🇸", Switzerland: "🇨🇭", Cameroon: "🇨🇲",
  Portugal: "🇵🇹", Ghana: "🇬🇭", Uruguay: "🇺🇾", "South Korea": "🇰🇷",
};

const ALL_TEAMS = Object.values(GROUPS).flat();

// ─── Walrus Storage ───────────────────────────────────────────────────────────
async function saveToWalrus(data) {
  try {
    const blob = new Blob([JSON.stringify(data)], { type: "application/json" });
    const res = await fetch(`${WALRUS_PUBLISHER}/v1/blobs?epochs=5`, {
      method: "PUT",
      body: blob,
    });
    if (!res.ok) throw new Error(`Walrus PUT failed: ${res.status}`);
    const json = await res.json();
    const blobId =
      json?.newlyCreated?.blobObject?.blobId ||
      json?.alreadyCertified?.blobId ||
      null;
    return { success: true, blobId };
  } catch (e) {
    return { success: false, error: e.message };
  }
}

async function loadFromWalrus(blobId) {
  try {
    const res = await fetch(`${WALRUS_AGGREGATOR}/v1/blobs/${blobId}`);
    if (!res.ok) throw new Error(`Walrus GET failed: ${res.status}`);
    const data = await res.json();
    return { success: true, data };
  } catch (e) {
    return { success: false, error: e.message };
  }
}

// ─── Claude Roaster API ───────────────────────────────────────────────────────
async function getRoast(predictions, newPrediction, grudges) {
  const grudgeList = grudges.length
    ? grudges.map((g) => `- ${g}`).join("\n")
    : "None yet.";

  const predList = predictions.length
    ? predictions
        .map(
          (p) =>
            `${p.match}: Predicted ${p.predicted} → Actual ${p.actual || "TBD"} (${p.correct === true ? "✅ Correct" : p.correct === false ? "❌ WRONG" : "⏳ Pending"})`
        )
        .join("\n")
    : "No previous predictions.";

  const prompt = `You are GRUDGE — a savage, unforgiving World Cup prediction judge AI. You remember every bad call, every overconfident take, every embarrassing miss. You hold grudges and bring them up constantly.

CURRENT GRUDGES YOU HOLD:
${grudgeList}

PREDICTION HISTORY:
${predList}

NEW PREDICTION JUST MADE:
Match: ${newPrediction.match}
They predicted: ${newPrediction.predicted}

Respond with a short, biting roast (2-4 sentences max). Reference past failures if they exist. If this is their first prediction, set the tone — warn them you're watching. Be brutal but funny. Use football/soccer references. End with what new grudge you're filing if this prediction turns out wrong. Format:
ROAST: [your roast here]
NEW_GRUDGE: [one-line grudge to remember if wrong, or "Watching closely" if first]`;

  try {
    const res = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 300,
        messages: [{ role: "user", content: prompt }],
      }),
    });
    const data = await res.json();
    const text = data.content?.[0]?.text || "";
    const roastMatch = text.match(/ROAST:\s*([\s\S]*?)(?=NEW_GRUDGE:|$)/);
    const grudgeMatch = text.match(/NEW_GRUDGE:\s*(.*)/);
    return {
      roast: roastMatch?.[1]?.trim() || text,
      newGrudge: grudgeMatch?.[1]?.trim() || null,
    };
  } catch (e) {
    return {
      roast: "The servers are down, but I still remember your bad takes.",
      newGrudge: null,
    };
  }
}

async function getResultRoast(prediction, grudges) {
  const grudgeList = grudges.map((g) => `- ${g}`).join("\n") || "None yet.";
  const outcome = prediction.correct ? "CORRECT" : "WRONG";

  const prompt = `You are GRUDGE — a savage World Cup prediction judge AI with a long memory.

CURRENT GRUDGE FILE:
${grudgeList}

RESULT JUST IN:
Match: ${prediction.match}
They predicted: ${prediction.predicted}
Actual result: ${prediction.actual}
Verdict: ${outcome}

${outcome === "CORRECT" ? "They got it right. Give a backhanded compliment — act suspicious, suggest it was luck, question if they truly deserve credit. Be grudgingly impressed but don't let them off easy." : "They got it WRONG. Absolutely destroy them. Reference the grudge file if relevant. Be merciless but funny. This is what you've been waiting for."}

2-4 sentences. Pure football banter energy.`;

  try {
    const res = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 250,
        messages: [{ role: "user", content: prompt }],
      }),
    });
    const data = await res.json();
    return data.content?.[0]?.text?.trim() || "...I'm speechless. Truly.";
  } catch {
    return outcome === "CORRECT"
      ? "Fine. You got lucky. Don't push it."
      : "Called it. Adding this to the file.";
  }
}

// ─── Grudge Meter Component ───────────────────────────────────────────────────
function GrudgeMeter({ predictions }) {
  const total = predictions.filter((p) => p.correct !== undefined).length;
  const wrong = predictions.filter((p) => p.correct === false).length;
  const pct = total === 0 ? 0 : Math.round((wrong / total) * 100);

  const color =
    pct < 30 ? "#00c853" : pct < 60 ? "#ff6b35" : "#ff1744";
  const label =
    pct === 0
      ? "CLEAN SLATE"
      : pct < 30
      ? "MILDLY SUSPICIOUS"
      : pct < 60
      ? "GRUDGE BUILDING"
      : pct < 80
      ? "FULL GRUDGE MODE"
      : "ETERNAL ENEMY";

  return (
    <div style={styles.grudgeMeter}>
      <div style={styles.grudgeHeader}>
        <span style={styles.grudgeLabel}>GRUDGE METER</span>
        <span style={{ ...styles.grudgeStatus, color }}>{label}</span>
      </div>
      <div style={styles.grudgeTrack}>
        <div
          style={{
            ...styles.grudgeFill,
            width: `${pct}%`,
            background: `linear-gradient(90deg, #ff6b35, ${color})`,
            boxShadow: pct > 0 ? `0 0 12px ${color}88` : "none",
          }}
        />
      </div>
      <div style={styles.grudgeStats}>
        <span style={{ color: "#888" }}>
          {wrong} wrong / {total} settled
        </span>
        <span style={{ color, fontWeight: 700 }}>{pct}% miss rate</span>
      </div>
    </div>
  );
}

// ─── Main App ─────────────────────────────────────────────────────────────────
export default function App() {
  const [predictions, setPredictions] = useState([]);
  const [grudges, setGrudges] = useState([]);
  const [blobId, setBlobId] = useState(null);
  const [restoreBlobId, setRestoreBlobId] = useState("");
  const [roastMessage, setRoastMessage] = useState(null);
  const [isRoasting, setIsRoasting] = useState(false);
  const [isSaving, setIsSaving] = useState(false);
  const [saveStatus, setSaveStatus] = useState(null);
  const [activeTab, setActiveTab] = useState("predict");
  const [showGrudgeFile, setShowGrudgeFile] = useState(false);

  // Match builder
  const [team1, setTeam1] = useState("Brazil");
  const [team2, setTeam2] = useState("Argentina");
  const [predicted, setPredicted] = useState("Brazil");

  // Result setter
  const [selectedPredId, setSelectedPredId] = useState(null);
  const [actualResult, setActualResult] = useState("");

  const roastRef = useRef(null);

  // Load from localStorage on mount
  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) {
        const { predictions: p, grudges: g, blobId: b } = JSON.parse(saved);
        if (p) setPredictions(p);
        if (g) setGrudges(g);
        if (b) setBlobId(b);
      }
    } catch {}
  }, []);

  // Persist to localStorage on change
  useEffect(() => {
    localStorage.setItem(
      STORAGE_KEY,
      JSON.stringify({ predictions, grudges, blobId })
    );
  }, [predictions, grudges, blobId]);

  const handlePredict = async () => {
    if (team1 === team2) return;
    setIsRoasting(true);
    setRoastMessage(null);

    const match = `${team1} vs ${team2}`;
    const newPred = {
      id: Date.now(),
      match,
      team1,
      team2,
      predicted,
      actual: null,
      correct: undefined,
      timestamp: new Date().toISOString(),
    };

    const { roast, newGrudge } = await getRoast(predictions, newPred, grudges);

    const updatedPredictions = [newPred, ...predictions];
    const updatedGrudges = newGrudge
      ? [`${match} — ${newGrudge}`, ...grudges]
      : grudges;

    setPredictions(updatedPredictions);
    setGrudges(updatedGrudges);
    setRoastMessage({ text: roast, type: "predict" });
    setIsRoasting(false);

    setTimeout(() => roastRef.current?.scrollIntoView({ behavior: "smooth" }), 100);
  };

  const handleSetResult = async () => {
    if (!selectedPredId || !actualResult) return;
    setIsRoasting(true);

    const pred = predictions.find((p) => p.id === selectedPredId);
    if (!pred) return;

    const correct = pred.predicted === actualResult;
    const updated = predictions.map((p) =>
      p.id === selectedPredId ? { ...p, actual: actualResult, correct } : p
    );

    // Get roast for result
    const roast = await getResultRoast({ ...pred, actual: actualResult, correct }, grudges);

    // Add to grudges if wrong
    let updatedGrudges = grudges;
    if (!correct) {
      updatedGrudges = [
        `${pred.match}: Predicted ${pred.predicted}, ${actualResult} won instead. Pathetic.`,
        ...grudges,
      ];
      setGrudges(updatedGrudges);
    }

    setPredictions(updated);
    setRoastMessage({ text: roast, type: correct ? "correct" : "wrong" });
    setSelectedPredId(null);
    setActualResult("");
    setIsRoasting(false);
    setTimeout(() => roastRef.current?.scrollIntoView({ behavior: "smooth" }), 100);
  };

  const handleSaveToWalrus = async () => {
    setIsSaving(true);
    setSaveStatus(null);
    const payload = {
      version: "1.0",
      app: "WC2026 Grudge Agent",
      savedAt: new Date().toISOString(),
      predictions,
      grudges,
    };
    const result = await saveToWalrus(payload);
    if (result.success && result.blobId) {
      setBlobId(result.blobId);
      setSaveStatus({ ok: true, msg: `Saved! Blob ID: ${result.blobId}` });
    } else {
      setSaveStatus({ ok: false, msg: `Failed: ${result.error}` });
    }
    setIsSaving(false);
  };

  const handleRestoreFromWalrus = async () => {
    if (!restoreBlobId.trim()) return;
    setIsSaving(true);
    setSaveStatus(null);
    const result = await loadFromWalrus(restoreBlobId.trim());
    if (result.success) {
      setPredictions(result.data.predictions || []);
      setGrudges(result.data.grudges || []);
      setBlobId(restoreBlobId.trim());
      setSaveStatus({ ok: true, msg: "Memory restored from Walrus!" });
    } else {
      setSaveStatus({ ok: false, msg: `Restore failed: ${result.error}` });
    }
    setIsSaving(false);
  };

  const stats = {
    total: predictions.length,
    correct: predictions.filter((p) => p.correct === true).length,
    wrong: predictions.filter((p) => p.correct === false).length,
    pending: predictions.filter((p) => p.correct === undefined).length,
  };

  return (
    <div style={styles.root}>
      {/* Background pitch lines */}
      <div style={styles.pitchBg} />

      {/* Header */}
      <header style={styles.header}>
        <div style={styles.headerInner}>
          <div style={styles.logo}>
            <span style={styles.logoIcon}>⚽</span>
            <div>
              <div style={styles.logoTitle}>GRUDGE AGENT</div>
              <div style={styles.logoSub}>FIFA World Cup 2026 · Prediction Tracker</div>
            </div>
          </div>
          <div style={styles.headerRight}>
            <div style={styles.statPill}>
              <span style={{ color: "#00c853" }}>{stats.correct}W</span>
              <span style={{ color: "#666" }}>/</span>
              <span style={{ color: "#ff1744" }}>{stats.wrong}L</span>
              <span style={{ color: "#666" }}>/</span>
              <span style={{ color: "#888" }}>{stats.pending}P</span>
            </div>
            <div
              style={{
                ...styles.walrusBadge,
                background: blobId ? "#00c85320" : "#ffffff10",
                borderColor: blobId ? "#00c853" : "#333",
              }}
            >
              <span style={{ fontSize: 14 }}>🦭</span>
              <span style={{ color: blobId ? "#00c853" : "#666", fontSize: 11 }}>
                {blobId ? "SYNCED" : "LOCAL"}
              </span>
            </div>
          </div>
        </div>
        <GrudgeMeter predictions={predictions} />
      </header>

      {/* Roast Toast */}
      {roastMessage && (
        <div
          ref={roastRef}
          style={{
            ...styles.roastToast,
            borderColor:
              roastMessage.type === "correct"
                ? "#00c853"
                : roastMessage.type === "wrong"
                ? "#ff1744"
                : "#ff6b35",
            background:
              roastMessage.type === "correct"
                ? "#00c85310"
                : roastMessage.type === "wrong"
                ? "#ff174410"
                : "#ff6b3510",
          }}
        >
          <div style={styles.roastIcon}>
            {roastMessage.type === "correct" ? "😤" : roastMessage.type === "wrong" ? "💀" : "🔥"}
          </div>
          <div style={styles.roastContent}>
            <div style={styles.roastLabel}>GRUDGE SPEAKS</div>
            <div style={styles.roastText}>{roastMessage.text}</div>
          </div>
          <button style={styles.roastClose} onClick={() => setRoastMessage(null)}>
            ✕
          </button>
        </div>
      )}

      {/* Main Layout */}
      <div style={styles.main}>
        {/* Left Panel */}
        <div style={styles.leftPanel}>
          {/* Tabs */}
          <div style={styles.tabs}>
            {["predict", "results", "memory"].map((tab) => (
              <button
                key={tab}
                style={{
                  ...styles.tab,
                  ...(activeTab === tab ? styles.tabActive : {}),
                }}
                onClick={() => setActiveTab(tab)}
              >
                {tab === "predict" ? "⚽ Predict" : tab === "results" ? "📋 Results" : "🦭 Memory"}
              </button>
            ))}
          </div>

          {/* Predict Tab */}
          {activeTab === "predict" && (
            <div style={styles.card}>
              <div style={styles.cardTitle}>Make a Prediction</div>
              <div style={styles.matchBuilder}>
                <div style={styles.teamSelect}>
                  <label style={styles.selectLabel}>HOME</label>
                  <select
                    style={styles.select}
                    value={team1}
                    onChange={(e) => {
                      setTeam1(e.target.value);
                      if (predicted === team1) setPredicted(e.target.value);
                    }}
                  >
                    {ALL_TEAMS.map((t) => (
                      <option key={t} value={t}>
                        {FLAGS[t]} {t}
                      </option>
                    ))}
                  </select>
                </div>
                <div style={styles.vsLabel}>VS</div>
                <div style={styles.teamSelect}>
                  <label style={styles.selectLabel}>AWAY</label>
                  <select
                    style={styles.select}
                    value={team2}
                    onChange={(e) => {
                      setTeam2(e.target.value);
                      if (predicted === team2) setPredicted(e.target.value);
                    }}
                  >
                    {ALL_TEAMS.map((t) => (
                      <option key={t} value={t}>
                        {FLAGS[t]} {t}
                      </option>
                    ))}
                  </select>
                </div>
              </div>

              <div style={styles.pickerSection}>
                <div style={styles.selectLabel}>YOUR PICK</div>
                <div style={styles.pickButtons}>
                  {[team1, "Draw", team2].map((option) => (
                    <button
                      key={option}
                      style={{
                        ...styles.pickBtn,
                        ...(predicted === option ? styles.pickBtnActive : {}),
                      }}
                      onClick={() => setPredicted(option)}
                    >
                      {option !== "Draw" && FLAGS[option]
                        ? `${FLAGS[option]} `
                        : ""}
                      {option}
                    </button>
                  ))}
                </div>
              </div>

              <button
                style={{
                  ...styles.submitBtn,
                  opacity: isRoasting ? 0.6 : 1,
                }}
                onClick={handlePredict}
                disabled={isRoasting || team1 === team2}
              >
                {isRoasting ? (
                  <span style={styles.loadingDots}>GRUDGE IS JUDGING</span>
                ) : (
                  "SUBMIT PREDICTION 🔥"
                )}
              </button>
              {team1 === team2 && (
                <div style={{ color: "#ff6b35", fontSize: 12, textAlign: "center", marginTop: 8 }}>
                  Pick two different teams
                </div>
              )}
            </div>
          )}

          {/* Results Tab */}
          {activeTab === "results" && (
            <div style={styles.card}>
              <div style={styles.cardTitle}>Enter Match Result</div>
              {predictions.filter((p) => p.correct === undefined).length === 0 ? (
                <div style={styles.emptyState}>No pending predictions. Make some calls first.</div>
              ) : (
                <>
                  <div style={styles.selectLabel}>SELECT PREDICTION</div>
                  <select
                    style={{ ...styles.select, width: "100%", marginBottom: 16 }}
                    value={selectedPredId || ""}
                    onChange={(e) => setSelectedPredId(Number(e.target.value))}
                  >
                    <option value="">— Pick a pending match —</option>
                    {predictions
                      .filter((p) => p.correct === undefined)
                      .map((p) => (
                        <option key={p.id} value={p.id}>
                          {p.match} → You said: {p.predicted}
                        </option>
                      ))}
                  </select>

                  {selectedPredId && (
                    <>
                      <div style={styles.selectLabel}>ACTUAL WINNER</div>
                      <div style={styles.pickButtons}>
                        {(() => {
                          const pred = predictions.find((p) => p.id === selectedPredId);
                          return [pred?.team1, "Draw", pred?.team2].filter(Boolean).map(
                            (option) => (
                              <button
                                key={option}
                                style={{
                                  ...styles.pickBtn,
                                  ...(actualResult === option ? styles.pickBtnActive : {}),
                                }}
                                onClick={() => setActualResult(option)}
                              >
                                {option !== "Draw" && FLAGS[option]
                                  ? `${FLAGS[option]} `
                                  : ""}
                                {option}
                              </button>
                            )
                          );
                        })()}
                      </div>
                      <button
                        style={{
                          ...styles.submitBtn,
                          background: "#1a2740",
                          borderColor: "#c9a227",
                          color: "#c9a227",
                          marginTop: 16,
                          opacity: isRoasting ? 0.6 : 1,
                        }}
                        onClick={handleSetResult}
                        disabled={!actualResult || isRoasting}
                      >
                        {isRoasting ? "GRUDGE PROCESSING..." : "SUBMIT RESULT ⚡"}
                      </button>
                    </>
                  )}
                </>
              )}
            </div>
          )}

          {/* Memory / Walrus Tab */}
          {activeTab === "memory" && (
            <div style={styles.card}>
              <div style={styles.cardTitle}>Walrus Memory</div>

              <div style={styles.walrusInfo}>
                <div style={styles.infoRow}>
                  <span style={styles.infoLabel}>NETWORK</span>
                  <span style={styles.infoValue}>Walrus Testnet</span>
                </div>
                <div style={styles.infoRow}>
                  <span style={styles.infoLabel}>PREDICTIONS</span>
                  <span style={styles.infoValue}>{predictions.length} stored</span>
                </div>
                <div style={styles.infoRow}>
                  <span style={styles.infoLabel}>GRUDGES FILED</span>
                  <span style={{ ...styles.infoValue, color: "#ff6b35" }}>
                    {grudges.length} active
                  </span>
                </div>
                {blobId && (
                  <div style={styles.blobIdBox}>
                    <div style={styles.infoLabel}>CURRENT BLOB ID</div>
                    <div style={styles.blobId}>{blobId}</div>
                    <a
                      href={`${WALRUS_AGGREGATOR}/v1/blobs/${blobId}`}
                      target="_blank"
                      rel="noreferrer"
                      style={styles.blobLink}
                    >
                      View on Walrus ↗
                    </a>
                  </div>
                )}
              </div>

              <button
                style={{
                  ...styles.submitBtn,
                  opacity: isSaving ? 0.6 : 1,
                  marginBottom: 12,
                }}
                onClick={handleSaveToWalrus}
                disabled={isSaving}
              >
                {isSaving ? "SAVING TO WALRUS..." : "💾 SAVE TO WALRUS TESTNET"}
              </button>

              <div style={styles.restoreSection}>
                <div style={styles.selectLabel}>RESTORE FROM BLOB ID</div>
                <input
                  style={styles.input}
                  placeholder="Paste a Blob ID to restore memory..."
                  value={restoreBlobId}
                  onChange={(e) => setRestoreBlobId(e.target.value)}
                />
                <button
                  style={{
                    ...styles.submitBtn,
                    background: "#0f1e0f",
                    borderColor: "#00c853",
                    color: "#00c853",
                    marginTop: 8,
                    opacity: isSaving ? 0.6 : 1,
                  }}
                  onClick={handleRestoreFromWalrus}
                  disabled={isSaving || !restoreBlobId.trim()}
                >
                  🔄 RESTORE MEMORY
                </button>
              </div>

              {saveStatus && (
                <div
                  style={{
                    ...styles.saveStatus,
                    color: saveStatus.ok ? "#00c853" : "#ff1744",
                    borderColor: saveStatus.ok ? "#00c85330" : "#ff174430",
                  }}
                >
                  {saveStatus.ok ? "✓" : "✗"} {saveStatus.msg}
                </div>
              )}

              <button
                style={{ ...styles.dangerBtn, marginTop: 16 }}
                onClick={() => {
                  if (confirm("Reset all predictions and grudges? The grudge agent will forget everything.")) {
                    setPredictions([]);
                    setGrudges([]);
                    setBlobId(null);
                    setSaveStatus(null);
                  }
                }}
              >
                🗑 CLEAR ALL MEMORY
              </button>
            </div>
          )}
        </div>

        {/* Right Panel — Predictions Feed */}
        <div style={styles.rightPanel}>
          {/* Grudge File Toggle */}
          <div style={styles.card}>
            <button
              style={styles.grudgeFileBtn}
              onClick={() => setShowGrudgeFile(!showGrudgeFile)}
            >
              <span>📁 GRUDGE FILE</span>
              <span style={{ color: "#ff6b35" }}>
                {grudges.length} entries {showGrudgeFile ? "▲" : "▼"}
              </span>
            </button>
            {showGrudgeFile && (
              <div style={styles.grudgeList}>
                {grudges.length === 0 ? (
                  <div style={styles.emptyState}>No grudges yet. Keep predicting.</div>
                ) : (
                  grudges.map((g, i) => (
                    <div key={i} style={styles.grudgeEntry}>
                      <span style={{ color: "#ff6b35", marginRight: 8 }}>⚠</span>
                      {g}
                    </div>
                  ))
                )}
              </div>
            )}
          </div>

          {/* Predictions Feed */}
          <div style={styles.feedTitle}>PREDICTION HISTORY</div>
          {predictions.length === 0 ? (
            <div style={{ ...styles.card, ...styles.emptyState }}>
              No predictions yet. GRUDGE is waiting. Don't disappoint.
            </div>
          ) : (
            predictions.map((p) => (
              <div
                key={p.id}
                style={{
                  ...styles.predCard,
                  borderColor:
                    p.correct === true
                      ? "#00c85340"
                      : p.correct === false
                      ? "#ff174440"
                      : "#ffffff10",
                }}
              >
                <div style={styles.predTop}>
                  <span style={styles.predMatch}>{p.match}</span>
                  <span
                    style={{
                      ...styles.predBadge,
                      background:
                        p.correct === true
                          ? "#00c85320"
                          : p.correct === false
                          ? "#ff174420"
                          : "#ffffff10",
                      color:
                        p.correct === true
                          ? "#00c853"
                          : p.correct === false
                          ? "#ff1744"
                          : "#666",
                    }}
                  >
                    {p.correct === true ? "✅ CORRECT" : p.correct === false ? "❌ WRONG" : "⏳ PENDING"}
                  </span>
                </div>
                <div style={styles.predDetails}>
                  <span>
                    Predicted:{" "}
                    <strong style={{ color: "#fff" }}>
                      {FLAGS[p.predicted] || ""} {p.predicted}
                    </strong>
                  </span>
                  {p.actual && (
                    <span>
                      {" "}· Actual:{" "}
                      <strong style={{ color: p.correct ? "#00c853" : "#ff1744" }}>
                        {FLAGS[p.actual] || ""} {p.actual}
                      </strong>
                    </span>
                  )}
                </div>
                <div style={styles.predTime}>
                  {new Date(p.timestamp).toLocaleString()}
                </div>
              </div>
            ))
          )}
        </div>
      </div>

      {/* Footer */}
      <footer style={styles.footer}>
        <span>GRUDGE AGENT · FIFA World Cup 2026 · Powered by Claude + Walrus Testnet</span>
        {blobId && (
          <span style={{ color: "#00c853", fontFamily: "monospace", fontSize: 11 }}>
            🦭 {blobId.slice(0, 20)}...
          </span>
        )}
      </footer>
    </div>
  );
}

// ─── Styles ───────────────────────────────────────────────────────────────────
const styles = {
  root: {
    minHeight: "100vh",
    background: "#07090f",
    color: "#e8eaed",
    fontFamily: "'Inter', system-ui, -apple-system, sans-serif",
    position: "relative",
    overflowX: "hidden",
  },
  pitchBg: {
    position: "fixed",
    inset: 0,
    backgroundImage: `
      linear-gradient(rgba(0,200,83,0.03) 1px, transparent 1px),
      linear-gradient(90deg, rgba(0,200,83,0.03) 1px, transparent 1px)
    `,
    backgroundSize: "60px 60px",
    pointerEvents: "none",
    zIndex: 0,
  },
  header: {
    position: "sticky",
    top: 0,
    background: "rgba(7,9,15,0.95)",
    backdropFilter: "blur(12px)",
    borderBottom: "1px solid #1a1f2e",
    padding: "16px 24px 12px",
    zIndex: 100,
  },
  headerInner: {
    display: "flex",
    alignItems: "center",
    justifyContent: "space-between",
    marginBottom: 12,
  },
  logo: { display: "flex", alignItems: "center", gap: 12 },
  logoIcon: { fontSize: 32 },
  logoTitle: {
    fontSize: 22,
    fontWeight: 900,
    letterSpacing: "0.12em",
    color: "#fff",
    lineHeight: 1,
  },
  logoSub: { fontSize: 11, color: "#666", letterSpacing: "0.06em", marginTop: 2 },
  headerRight: { display: "flex", alignItems: "center", gap: 12 },
  statPill: {
    background: "#0f1520",
    border: "1px solid #1e2535",
    borderRadius: 20,
    padding: "6px 14px",
    fontSize: 13,
    fontWeight: 700,
    display: "flex",
    gap: 8,
    alignItems: "center",
  },
  walrusBadge: {
    display: "flex",
    alignItems: "center",
    gap: 6,
    padding: "5px 10px",
    borderRadius: 6,
    border: "1px solid",
    fontSize: 11,
    fontWeight: 700,
    letterSpacing: "0.08em",
  },
  grudgeMeter: {
    background: "#0d1120",
    borderRadius: 8,
    padding: "10px 14px",
    border: "1px solid #1a1f2e",
  },
  grudgeHeader: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 6,
  },
  grudgeLabel: {
    fontSize: 10,
    fontWeight: 800,
    letterSpacing: "0.15em",
    color: "#555",
  },
  grudgeStatus: {
    fontSize: 11,
    fontWeight: 800,
    letterSpacing: "0.1em",
  },
  grudgeTrack: {
    height: 6,
    background: "#1a1f2e",
    borderRadius: 3,
    overflow: "hidden",
    marginBottom: 6,
  },
  grudgeFill: {
    height: "100%",
    borderRadius: 3,
    transition: "width 0.6s cubic-bezier(0.34, 1.56, 0.64, 1)",
  },
  grudgeStats: {
    display: "flex",
    justifyContent: "space-between",
    fontSize: 10,
    fontWeight: 600,
  },
  roastToast: {
    position: "relative",
    zIndex: 50,
    margin: "16px 24px",
    padding: "16px 20px",
    borderRadius: 10,
    border: "1px solid",
    display: "flex",
    gap: 16,
    alignItems: "flex-start",
    animation: "slideIn 0.3s ease",
  },
  roastIcon: { fontSize: 28, flexShrink: 0 },
  roastContent: { flex: 1 },
  roastLabel: {
    fontSize: 10,
    fontWeight: 800,
    letterSpacing: "0.15em",
    color: "#888",
    marginBottom: 6,
  },
  roastText: {
    fontSize: 15,
    lineHeight: 1.5,
    color: "#e8eaed",
    fontStyle: "italic",
  },
  roastClose: {
    background: "none",
    border: "none",
    color: "#555",
    cursor: "pointer",
    fontSize: 16,
    padding: 0,
    flexShrink: 0,
  },
  main: {
    display: "grid",
    gridTemplateColumns: "380px 1fr",
    gap: 20,
    padding: "20px 24px",
    position: "relative",
    zIndex: 1,
    maxWidth: 1200,
    margin: "0 auto",
  },
  leftPanel: { display: "flex", flexDirection: "column", gap: 0 },
  rightPanel: { display: "flex", flexDirection: "column", gap: 12 },
  tabs: {
    display: "grid",
    gridTemplateColumns: "1fr 1fr 1fr",
    gap: 2,
    marginBottom: 12,
  },
  tab: {
    background: "#0d1120",
    border: "1px solid #1a1f2e",
    color: "#666",
    padding: "10px 6px",
    fontSize: 12,
    fontWeight: 700,
    letterSpacing: "0.04em",
    cursor: "pointer",
    borderRadius: 6,
    transition: "all 0.15s",
  },
  tabActive: {
    background: "#ff6b3515",
    borderColor: "#ff6b35",
    color: "#ff6b35",
  },
  card: {
    background: "#0d1120",
    border: "1px solid #1a1f2e",
    borderRadius: 10,
    padding: "20px",
    marginBottom: 12,
  },
  cardTitle: {
    fontSize: 12,
    fontWeight: 800,
    letterSpacing: "0.15em",
    color: "#555",
    marginBottom: 16,
  },
  matchBuilder: {
    display: "grid",
    gridTemplateColumns: "1fr auto 1fr",
    gap: 12,
    alignItems: "end",
    marginBottom: 16,
  },
  teamSelect: { display: "flex", flexDirection: "column", gap: 6 },
  selectLabel: {
    fontSize: 10,
    fontWeight: 800,
    letterSpacing: "0.15em",
    color: "#555",
    marginBottom: 4,
  },
  select: {
    background: "#060a14",
    border: "1px solid #1e2535",
    color: "#e8eaed",
    padding: "10px 12px",
    borderRadius: 6,
    fontSize: 13,
    cursor: "pointer",
    outline: "none",
  },
  vsLabel: {
    fontSize: 12,
    fontWeight: 900,
    color: "#ff6b35",
    letterSpacing: "0.1em",
    paddingBottom: 10,
    textAlign: "center",
  },
  pickerSection: { marginBottom: 20 },
  pickButtons: { display: "flex", gap: 8, marginTop: 8 },
  pickBtn: {
    flex: 1,
    background: "#060a14",
    border: "1px solid #1e2535",
    color: "#888",
    padding: "10px 8px",
    borderRadius: 6,
    fontSize: 12,
    fontWeight: 700,
    cursor: "pointer",
    transition: "all 0.15s",
    textAlign: "center",
  },
  pickBtnActive: {
    background: "#ff6b3520",
    borderColor: "#ff6b35",
    color: "#ff6b35",
  },
  submitBtn: {
    width: "100%",
    background: "#ff6b3515",
    border: "1px solid #ff6b35",
    color: "#ff6b35",
    padding: "14px",
    borderRadius: 8,
    fontSize: 13,
    fontWeight: 900,
    letterSpacing: "0.08em",
    cursor: "pointer",
    transition: "all 0.2s",
  },
  loadingDots: { animation: "pulse 1.2s ease infinite" },
  dangerBtn: {
    width: "100%",
    background: "transparent",
    border: "1px solid #ff174430",
    color: "#ff174480",
    padding: "10px",
    borderRadius: 6,
    fontSize: 11,
    fontWeight: 700,
    cursor: "pointer",
    letterSpacing: "0.08em",
  },
  walrusInfo: {
    background: "#060a14",
    borderRadius: 8,
    padding: "14px",
    marginBottom: 16,
    border: "1px solid #1a1f2e",
  },
  infoRow: {
    display: "flex",
    justifyContent: "space-between",
    marginBottom: 8,
    fontSize: 12,
  },
  infoLabel: {
    fontSize: 10,
    fontWeight: 800,
    letterSpacing: "0.12em",
    color: "#555",
  },
  infoValue: { fontSize: 12, color: "#aaa", fontWeight: 600 },
  blobIdBox: {
    marginTop: 12,
    paddingTop: 12,
    borderTop: "1px solid #1a1f2e",
  },
  blobId: {
    fontFamily: "monospace",
    fontSize: 11,
    color: "#00c853",
    wordBreak: "break-all",
    marginTop: 4,
    marginBottom: 6,
    background: "#00c85308",
    padding: "6px 8px",
    borderRadius: 4,
    border: "1px solid #00c85320",
  },
  blobLink: {
    fontSize: 11,
    color: "#00c85380",
    textDecoration: "none",
  },
  restoreSection: { marginBottom: 4 },
  input: {
    width: "100%",
    background: "#060a14",
    border: "1px solid #1e2535",
    color: "#e8eaed",
    padding: "10px 12px",
    borderRadius: 6,
    fontSize: 12,
    outline: "none",
    boxSizing: "border-box",
    fontFamily: "monospace",
  },
  saveStatus: {
    padding: "10px 14px",
    borderRadius: 6,
    border: "1px solid",
    fontSize: 12,
    marginTop: 8,
    fontFamily: "monospace",
    wordBreak: "break-all",
  },
  feedTitle: {
    fontSize: 10,
    fontWeight: 800,
    letterSpacing: "0.15em",
    color: "#444",
    marginBottom: 8,
    paddingLeft: 4,
  },
  predCard: {
    background: "#0d1120",
    border: "1px solid",
    borderRadius: 8,
    padding: "14px 16px",
    transition: "border-color 0.2s",
  },
  predTop: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    marginBottom: 6,
  },
  predMatch: { fontSize: 14, fontWeight: 700, color: "#fff" },
  predBadge: {
    fontSize: 10,
    fontWeight: 800,
    padding: "3px 8px",
    borderRadius: 4,
    letterSpacing: "0.06em",
  },
  predDetails: { fontSize: 12, color: "#666", marginBottom: 4 },
  predTime: { fontSize: 10, color: "#333", fontFamily: "monospace" },
  grudgeFileBtn: {
    width: "100%",
    background: "none",
    border: "none",
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    cursor: "pointer",
    padding: 0,
    color: "#888",
    fontSize: 12,
    fontWeight: 800,
    letterSpacing: "0.1em",
  },
  grudgeList: { marginTop: 12 },
  grudgeEntry: {
    fontSize: 12,
    color: "#888",
    padding: "8px 0",
    borderBottom: "1px solid #1a1f2e",
    lineHeight: 1.4,
  },
  emptyState: {
    color: "#444",
    fontSize: 13,
    textAlign: "center",
    padding: "20px 0",
    fontStyle: "italic",
  },
  footer: {
    borderTop: "1px solid #1a1f2e",
    padding: "14px 24px",
    display: "flex",
    justifyContent: "space-between",
    fontSize: 11,
    color: "#333",
    letterSpacing: "0.06em",
    position: "relative",
    zIndex: 1,
  },
};
