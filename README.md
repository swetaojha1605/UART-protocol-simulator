# UART-protocol-simulator
import { useState, useRef, useEffect, useCallback } from "react";

const TX_STATES = ["IDLE", "TX_START", "TX_DATA", "TX_STOP"];
const RX_STATES = ["IDLE", "RX_START", "RX_DATA", "RX_STOP"];

function FsmPill({ id, label, active }) {
  return (
    <div style={{
      padding: "5px 12px", borderRadius: 20, fontSize: 12, fontWeight: 500,
      border: `0.5px solid ${active ? "#185FA5" : "rgba(0,0,0,0.12)"}`,
      color: active ? "#E6F1FB" : "#5f5e5a",
      background: active ? "#185FA5" : "#f5f4f0",
      transition: "all 0.2s",
    }}>{label}</div>
  );
}

function FrameSlot({ label, val, isCurrent }) {
  let bg = "#f5f4f0", color = "#888780", border = "rgba(0,0,0,0.12)";
  if (val !== null) {
    if (label === "S")  { bg = "#FAECE7"; color = "#712B13"; border = "#D85A30"; }
    else if (label === "P" && val === 1) { bg = "#E1F5EE"; color = "#085041"; border = "#1D9E75"; }
    else if (label === "P" && val === 0) { bg = "#FCEBEB"; color = "#A32D2D"; border = "#E24B4A"; }
    else if (val === 1) { bg = "#EAF3DE"; color = "#27500A"; border = "#639922"; }
    else                { bg = "#FAEEDA"; color = "#633806"; border = "#EF9F27"; }
  }
  return (
    <div style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 3 }}>
      <div style={{
        width: 36, height: 36, borderRadius: 6, display: "flex",
        alignItems: "center", justifyContent: "center",
        fontSize: 13, fontWeight: 500, background: bg, color, border: `0.5px solid ${border}`,
        outline: isCurrent ? "2.5px solid #185FA5" : "none",
        outlineOffset: 2, transition: "all 0.2s",
      }}>
        {val !== null ? val : ""}
      </div>
      <div style={{ fontSize: 10, color: "#888780" }}>{label}</div>
    </div>
  );
}

function Waveform({ history }) {
  const canvasRef = useRef(null);
  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext("2d");
    const W = canvas.width, H = canvas.height;
    const isDark = matchMedia("(prefers-color-scheme:dark)").matches;
    ctx.fillStyle = isDark ? "#1e1e1c" : "#f5f4f0";
    ctx.fillRect(0, 0, W, H);
    const pts = history;
    if (pts.length < 2) return;
    const pad = 24, top = 14, bot = H - 14;
    const bitW = Math.min(48, (W - pad * 2) / Math.max(pts.length, 10));
    const colors = ["#888780", "#D85A30",
      "#378ADD","#378ADD","#378ADD","#378ADD","#378ADD","#378ADD","#378ADD","#378ADD",
      "#1D9E75"];
    ctx.strokeStyle = isDark ? "rgba(255,255,255,0.06)" : "rgba(0,0,0,0.06)";
    ctx.lineWidth = 0.5;
    for (let i = 0; i < pts.length; i++) {
      const x = pad + i * bitW;
      ctx.beginPath(); ctx.moveTo(x, top - 3); ctx.lineTo(x, bot + 3); ctx.stroke();
    }
    ctx.beginPath(); ctx.moveTo(pad, top); ctx.lineTo(pad + pts.length * bitW, top); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(pad, bot); ctx.lineTo(pad + pts.length * bitW, bot); ctx.stroke();
    ctx.fillStyle = isDark ? "#888780" : "#73726c";
    ctx.font = "10px monospace";
    ctx.fillText("1", 6, top + 4);
    ctx.fillText("0", 6, bot + 4);
    for (let i = 0; i < pts.length - 1; i++) {
      const x0 = pad + i * bitW, x1 = pad + (i + 1) * bitW;
      const y0 = pts[i] === 1 ? top : bot, y1 = pts[i + 1] === 1 ? top : bot;
      ctx.strokeStyle = colors[Math.min(i, colors.length - 1)];
      ctx.lineWidth = 2; ctx.lineJoin = "round";
      ctx.beginPath(); ctx.moveTo(x0, y0);
      if (y1 !== y0) ctx.lineTo(x1, y0);
      ctx.lineTo(x1, y1); ctx.stroke();
      ctx.fillStyle = ctx.strokeStyle;
      ctx.font = "10px monospace"; ctx.textAlign = "center";
      ctx.fillText(pts[i], x0 + bitW / 2, (top + bot) / 2 + 4);
    }
    const last = pts.length - 1;
    ctx.fillStyle = colors[Math.min(last, colors.length - 1)];
    ctx.font = "10px monospace"; ctx.textAlign = "center";
    ctx.fillText(pts[last], pad + last * bitW + bitW / 2, (top + bot) / 2 + 4);
  }, [history]);
  return (
    <canvas ref={canvasRef} width={640} height={90}
      style={{ width: "100%", height: 90, display: "block", borderRadius: 6, background: "#f5f4f0" }} />
  );
}

function Card({ children, faded }) {
  return (
    <div style={{
      background: "#ffffff", border: "0.5px solid rgba(0,0,0,0.12)",
      borderRadius: 12, padding: "14px 16px", marginBottom: 10,
      opacity: faded ? 0.4 : 1, pointerEvents: faded ? "none" : "auto",
      transition: "opacity 0.2s",
    }}>{children}</div>
  );
}

function SectionLabel({ children }) {
  return (
    <div style={{ fontSize: 11, fontWeight: 500, color: "#5f5e5a", letterSpacing: "0.08em", textTransform: "uppercase", marginBottom: 10 }}>
      {children}
    </div>
  );
}

function Btn({ onClick, children, variant = "default", disabled }) {
  const styles = {
    default: { background: "#f5f4f0", color: "#1a1a18", border: "0.5px solid rgba(0,0,0,0.22)" },
    primary: { background: "#185FA5", color: "#E6F1FB", border: "0.5px solid #185FA5" },
    danger:  { background: "#FCEBEB", color: "#A32D2D", border: "0.5px solid #E24B4A" },
    bit1:    { background: "#EAF3DE", color: "#27500A", border: "0.5px solid #639922" },
    bit0:    { background: "#FAEEDA", color: "#633806", border: "0.5px solid #EF9F27" },
  };
  const s = styles[variant] || styles.default;
  return (
    <button onClick={onClick} disabled={disabled} style={{
      padding: "8px 16px", borderRadius: 8, fontSize: 13,
      fontFamily: "monospace", cursor: disabled ? "not-allowed" : "pointer",
      opacity: disabled ? 0.4 : 1, transition: "all 0.1s", ...s,
    }}>{children}</button>
  );
}

export default function UARTManualSim() {
  const [phase, setPhase] = useState("idle"); // idle | data | stop | done
  const [dataBits, setDataBits] = useState([]);
  const [waveHistory, setWaveHistory] = useState([1, 1, 1]);
  const [txState, setTxState] = useState("IDLE");
  const [rxState, setRxState] = useState("IDLE");
  const [rxOutput, setRxOutput] = useState(null);
  const [logs, setLogs] = useState([{ msg: "Manual UART simulator ready. Start by sending a START bit.", cls: "info" }]);
  const [charIn, setCharIn] = useState("A");
  const [autoFilling, setAutoFilling] = useState(false);
  const logRef = useRef(null);

  useEffect(() => { if (logRef.current) logRef.current.scrollTop = logRef.current.scrollHeight; }, [logs]);

  const addLog = useCallback((msg, cls = "info") => {
    setLogs(prev => [...prev, { msg, cls }]);
  }, []);

  const slotLabels = ["S", "b0", "b1", "b2", "b3", "b4", "b5", "b6", "b7", "P"];
  const currentIdx = phase === "idle" ? -1 : phase === "start_pending" ? 0
    : phase === "data" ? dataBits.length + 1
    : phase === "stop" ? 9 : -1;

  const slotValues = slotLabels.map((lbl, i) => {
    if (phase === "idle") return null;
    if (i === 0) return 0;
    if (i >= 1 && i <= 8) return dataBits[i - 1] !== undefined ? dataBits[i - 1] : null;
    return null;
  });

  function sendStart() {
    setPhase("data");
    setDataBits([]);
    setWaveHistory([1, 1, 0]);
    setTxState("TX_START");
    setRxState("RX_START");
    setRxOutput(null);
    addLog("→ TX: IDLE → TX_START | Line = LOW (START bit)", "tx");
    addLog("→ RX: Falling edge detected → RX_START", "rx");
    setTimeout(() => {
      setTxState("TX_DATA");
      setRxState("RX_DATA");
      addLog("→ RX: Mid-bit sample OK → RX_DATA (sampling incoming bits)", "rx");
    }, 500);
  }

  function pushBit(b) {
    setDataBits(prev => {
      const next = [...prev, b];
      setWaveHistory(h => [...h, b]);
      addLog(`→ TX: DATA bit[${prev.length}] = ${b}  |  line = ${b ? "HIGH" : "LOW"}`, "tx");
      addLog(`→ RX: Sampled b${prev.length} = ${b}`, "rx");
      if (next.length === 8) {
        setTxState("TX_STOP");
        setRxState("RX_STOP");
        setPhase("stop");
        addLog("→ TX: All 8 bits sent → TX_STOP", "tx");
        addLog("→ RX: 8 bits received → RX_STOP (checking STOP bit...)", "rx");
      }
      return next;
    });
  }

  function autoFill() {
    if (phase !== "data" || autoFilling) return;
    const code = (charIn || "A").charCodeAt(0);
    setDataBits([]);
    setWaveHistory(h => h.slice(0, 3));
    addLog(`Auto-filling '${charIn || "A"}' (0x${code.toString(16).toUpperCase()}) = ${code.toString(2).padStart(8, "0")}b`, "info");
    setAutoFilling(true);
    let i = 0;
    function next() {
      if (i >= 8) { setAutoFilling(false); return; }
      const bit = (code >> i) & 1;
      pushBit(bit);
      i++;
      if (i < 8) setTimeout(next, 200);
      else setTimeout(() => setAutoFilling(false), 200);
    }
    setTimeout(next, 100);
  }

  function sendStop(b) {
    setWaveHistory(h => [...h, b, 1, 1]);
    setPhase("done");
    setTxState("IDLE");
    setRxState("IDLE");
    const val = dataBits.reduce((acc, bit, i) => acc | (bit << i), 0);
    const hex = "0x" + val.toString(16).toUpperCase().padStart(2, "0");
    const ch = String.fromCharCode(val);
    if (b === 1) {
      addLog(`→ TX: STOP bit = 1 → TX_IDLE`, "tx");
      addLog(`→ RX: STOP valid → rx_done HIGH | rx_data = ${hex} = '${ch}'`, "rx");
      setRxOutput({ ok: true, ch, hex, dec: val, bin: val.toString(2).padStart(8, "0") });
    } else {
      addLog("→ TX: STOP bit = 0 (framing error injected)", "err");
      addLog("→ RX: STOP bit ≠ 1 → FRAMING ERROR! frame_err = HIGH", "err");
      setRxOutput({ ok: false });
    }
    addLog("→ TX → IDLE | RX → IDLE | Frame complete", "info");
  }

  function resetAll() {
    setPhase("idle");
    setDataBits([]);
    setWaveHistory([1, 1, 1]);
    setTxState("IDLE");
    setRxState("IDLE");
    setRxOutput(null);
    setAutoFilling(false);
    setLogs([{ msg: "System reset. Line idle HIGH.", cls: "info" }]);
  }

  const logColors = { tx: "#185FA5", rx: "#0F6E56", err: "#A32D2D", info: "#888780" };

  return (
    <div style={{ fontFamily: "monospace", padding: "14px 0", maxWidth: 700 }}>

      <Card faded={phase !== "idle"}>
        <SectionLabel>Step 1 — Send START bit</SectionLabel>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <Btn variant="primary" onClick={sendStart} disabled={phase !== "idle"}>Send START (0)</Btn>
          <span style={{ fontSize: 12, color: "#888780" }}>
            {phase === "idle" ? "Press to begin a UART frame" : "START bit sent — line goes LOW"}
          </span>
        </div>
      </Card>

      <Card faded={phase !== "data"}>
        <SectionLabel>Step 2 — Enter 8 data bits (LSB first)</SectionLabel>
        <div style={{ display: "flex", gap: 8, alignItems: "center", marginBottom: 10, flexWrap: "wrap" }}>
          <button onClick={() => pushBit(1)} disabled={phase !== "data" || autoFilling} style={{
            width: 48, height: 48, borderRadius: 8, border: "0.5px solid #639922",
            background: "#EAF3DE", color: "#27500A", fontSize: 20, fontWeight: 500,
            cursor: phase !== "data" ? "not-allowed" : "pointer", fontFamily: "monospace",
          }}>1</button>
          <button onClick={() => pushBit(0)} disabled={phase !== "data" || autoFilling} style={{
            width: 48, height: 48, borderRadius: 8, border: "0.5px solid #EF9F27",
            background: "#FAEEDA", color: "#633806", fontSize: 20, fontWeight: 500,
            cursor: phase !== "data" ? "not-allowed" : "pointer", fontFamily: "monospace",
          }}>0</button>
          <span style={{ fontSize: 12, color: "#5f5e5a" }}>Bit {dataBits.length}/8</span>
        </div>
        <div style={{ display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" }}>
          <span style={{ fontSize: 12, color: "#888780" }}>Or auto-fill from char:</span>
          <input value={charIn} maxLength={1} onChange={e => setCharIn(e.target.value)}
            style={{ width: 48, padding: "6px 8px", borderRadius: 8, border: "0.5px solid rgba(0,0,0,0.22)", fontFamily: "monospace", fontSize: 14, textAlign: "center" }} />
          <Btn onClick={autoFill} disabled={phase !== "data" || autoFilling}>Auto-fill</Btn>
        </div>
      </Card>

      <Card faded={phase !== "stop"}>
        <SectionLabel>Step 3 — Send STOP bit</SectionLabel>
        <div style={{ display: "flex", gap: 8, flexWrap: "wrap" }}>
          <Btn variant="primary" onClick={() => sendStop(1)} disabled={phase !== "stop"}>Send valid STOP (1)</Btn>
          <Btn variant="danger" onClick={() => sendStop(0)} disabled={phase !== "stop"}>Inject framing error (0)</Btn>
        </div>
      </Card>

      <Card>
        <SectionLabel>UART frame</SectionLabel>
        <div style={{ display: "flex", gap: 3, flexWrap: "wrap" }}>
          {slotLabels.map((lbl, i) => (
            <FrameSlot key={i} label={lbl} val={slotValues[i]}
              isCurrent={i === currentIdx} />
          ))}
        </div>
      </Card>

      <Card>
        <SectionLabel>FSM states</SectionLabel>
        <div style={{ display: "flex", gap: 6, alignItems: "center", marginBottom: 8, flexWrap: "wrap" }}>
          <span style={{ fontSize: 11, color: "#888780", minWidth: 24 }}>TX</span>
          {TX_STATES.map((s, i) => (
            <div key={s} style={{ display: "flex", gap: 6, alignItems: "center" }}>
              <FsmPill label={s} active={txState === s} />
              {i < TX_STATES.length - 1 && <span style={{ color: "#888780", fontSize: 11 }}>→</span>}
            </div>
          ))}
        </div>
        <div style={{ display: "flex", gap: 6, alignItems: "center", flexWrap: "wrap" }}>
          <span style={{ fontSize: 11, color: "#888780", minWidth: 24 }}>RX</span>
          {RX_STATES.map((s, i) => (
            <div key={s} style={{ display: "flex", gap: 6, alignItems: "center" }}>
              <FsmPill label={s} active={rxState === s} />
              {i < RX_STATES.length - 1 && <span style={{ color: "#888780", fontSize: 11 }}>→</span>}
            </div>
          ))}
        </div>
      </Card>

      <Card>
        <SectionLabel>Waveform — tx_serial</SectionLabel>
        <Waveform history={waveHistory} />
      </Card>

      <Card>
        <SectionLabel>RX decoded output</SectionLabel>
        {!rxOutput && <span style={{ fontSize: 13, color: "#888780" }}>Waiting for complete frame...</span>}
        {rxOutput?.ok === false && (
          <span style={{ color: "#A32D2D", fontWeight: 500, fontSize: 14 }}>
            Framing error — frame_err = 1 &nbsp;
            <span style={{ color: "#888780", fontWeight: 400, fontSize: 12 }}>(Expected STOP=1, got 0)</span>
          </span>
        )}
        {rxOutput?.ok === true && (
          <span style={{ fontSize: 14 }}>
            <span style={{ color: "#0F6E56", fontWeight: 500 }}>'{rxOutput.ch}'</span>
            &nbsp;&nbsp;
            <span style={{ color: "#5f5e5a" }}>{rxOutput.hex}</span>
            &nbsp;&nbsp;
            <span style={{ color: "#888780" }}>({rxOutput.dec} dec, {rxOutput.bin}b)</span>
          </span>
        )}
      </Card>

      <Card>
        <SectionLabel>Log</SectionLabel>
        <div ref={logRef} style={{ maxHeight: 110, overflowY: "auto", fontSize: 12, lineHeight: "1.8" }}>
          {logs.map((l, i) => (
            <div key={i} style={{ color: logColors[l.cls] || "#888780" }}>{l.msg}</div>
          ))}
        </div>
      </Card>

      <div style={{ marginTop: 8 }}>
        <Btn onClick={resetAll}>↺ Reset everything</Btn>
      </div>
    </div>
  );
}
