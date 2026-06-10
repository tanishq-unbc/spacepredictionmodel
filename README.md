import { useState, useMemo, useRef, useEffect } from "react";
import {
  AreaChart, Area, BarChart, Bar,
  XAxis, YAxis, CartesianGrid, Tooltip, Legend,
  ResponsiveContainer,
} from "recharts";
import { LineChart, Line } from "recharts";

const YEARS = [2025,2030,2035,2040,2045,2050,2055,2060,2065,2070,2075];

const RAW = {
  cost: {
    moon:  { opt:[45,36,26,18,12,8,6,4.5,3.5,2.8,2.2],  real:[45,50,44,38,32,27,23,19,16,14,12],    pess:[45,62,60,58,57,55,53,51,50,48,47] },
    mars:  { opt:[0,185,145,105,75,54,40,30,23,18,14],   real:[0,0,260,205,165,132,108,88,72,60,50],  pess:[0,0,0,360,318,284,258,238,220,205,192] },
    outer: { opt:[0,0,85,125,105,86,70,57,46,38,31],     real:[0,0,0,210,188,164,144,126,110,97,86],  pess:[0,0,0,0,435,405,375,348,324,303,284] },
  },
  supply: {
    moon:  { opt:[2,9,22,48,96,168,272,420,610,840,1120], real:[2,5,13,26,44,72,108,152,204,264,334], pess:[2,3,7,12,18,26,36,48,62,78,96] },
    mars:  { opt:[0,1,6,18,44,96,184,326,534,820,1220],  real:[0,0,2,9,24,55,110,194,308,452,624],   pess:[0,0,0,2,8,22,44,78,122,176,244] },
    outer: { opt:[0,0,1,4,12,28,60,118,212,358,586],     real:[0,0,0,1,4,13,30,62,112,182,272],      pess:[0,0,0,0,1,4,11,24,43,68,102] },
  },
  population: {
    moon:  { opt:[0,4,28,115,460,2000,8200,26000,72000,185000,440000], real:[0,0,9,40,155,580,2000,6200,16500,40000,90000], pess:[0,0,0,6,24,82,230,540,1180,2400,4600] },
    mars:  { opt:[0,0,5,28,135,740,3800,17500,68000,228000,700000],   real:[0,0,0,5,26,155,790,3700,14200,46000,140000],  pess:[0,0,0,0,3,18,70,220,620,1650,3900] },
    outer: { opt:[0,0,0,3,12,50,220,880,3100,9400,24000],             real:[0,0,0,0,3,16,66,240,720,2100,5600],           pess:[0,0,0,0,0,2,11,35,96,240,570] },
  },
  launches: {
    moon:  { opt:[2,6,14,28,56,96,156,236,338,460,608], real:[2,4,8,15,26,41,60,85,114,148,188], pess:[2,2,4,7,10,14,19,25,32,40,50] },
    mars:  { opt:[0,1,3,7,16,30,56,96,158,244,360],    real:[0,0,1,3,7,14,24,38,58,84,116],     pess:[0,0,0,1,2,4,7,11,16,22,29] },
    outer: { opt:[0,0,1,3,6,13,22,39,63,97,146],       real:[0,0,0,1,2,6,11,20,30,44,61],       pess:[0,0,0,0,1,2,4,6,9,13,18] },
  },
};

const DEST_CONFIG = {
  moon:  { label:"Moon",        color:"#93C5FD", icon:"🌕" },
  mars:  { label:"Mars",        color:"#F97316", icon:"🔴" },
  outer: { label:"Outer Solar", color:"#A78BFA", icon:"🪐" },
};

const SCENARIO = {
  opt:  { label:"Optimistic",  color:"#4ADE80" },
  real: { label:"Realistic",   color:"#FBBF24" },
  pess: { label:"Pessimistic", color:"#F87171" },
};

const MODELS = [
  { key:"cost",       label:"Mission Cost",      unit:"$B / mission",   icon:"💰", desc:"Per-mission program cost in constant 2025 USD." },
  { key:"supply",     label:"Supply Chain",      unit:"tons / year",    icon:"📦", desc:"Net delivered cargo tonnage per year to destination." },
  { key:"population", label:"Colony Population", unit:"residents",      icon:"👥", desc:"Permanent or long-duration residents at destination." },
  { key:"launches",   label:"Launch Frequency",  unit:"missions / year",icon:"🚀", desc:"Total crewed + cargo missions dispatched annually." },
];

const SUGGESTED = [
  "What does the realistic Mars colony population look like by 2060?",
  "How do mission costs compare across Moon, Mars, and Outer Solar in 2050?",
  "Which scenario shows the biggest divergence in launch frequency by 2075?",
  "What are the main supply chain bottlenecks for Mars colonization?",
  "How does the pessimistic scenario affect outer solar system expansion?",
  "What assumptions drive the optimistic Moon cost reduction?",
];

const fmt = (v) => {
  if (v == null || v === 0) return "0";
  if (v >= 1e6) return (v/1e6).toFixed(2)+"M";
  if (v >= 1e3) return (v/1e3).toFixed(1)+"K";
  return typeof v === "number" ? v.toFixed(1) : v;
};
const fmtShort = (v) => {
  if (!v) return "0";
  if (v >= 1e6) return (v/1e6).toFixed(1)+"M";
  if (v >= 1e3) return (v/1e3).toFixed(0)+"K";
  return Math.round(v);
};

const CustomTooltip = ({ active, payload, label, unit }) => {
  if (!active || !payload?.length) return null;
  return (
    <div style={{ background:"#0D1A2E", border:"1px solid #1E3A5F", borderRadius:8, padding:"10px 14px", fontSize:12 }}>
      <div style={{ color:"#64748B", marginBottom:6, fontWeight:600 }}>{label}</div>
      {payload.map(p => (
        <div key={p.dataKey} style={{ display:"flex", justifyContent:"space-between", gap:20, color:SCENARIO[p.dataKey]?.color || p.color, marginBottom:2 }}>
          <span>{SCENARIO[p.dataKey]?.label || p.name}</span>
          <span style={{ fontFamily:"monospace", fontWeight:700 }}>{fmt(p.value)} <span style={{ opacity:0.6 }}>{unit}</span></span>
        </div>
      ))}
    </div>
  );
};

// Build a data summary string to inject into the AI system prompt
const buildDataContext = () => {
  let ctx = "You are an expert analyst for a space exploration logistics model covering the years 2025–2075. ";
  ctx += "The model tracks four dimensions across three destinations (Moon, Mars, Outer Solar System) under three scenarios (Optimistic, Realistic, Pessimistic).\n\n";
  ctx += "KEY DATA SNAPSHOTS (selected years):\n";
  const snapYears = [2030, 2040, 2050, 2060, 2075];
  for (const dim of ["cost","supply","population","launches"]) {
    const units = { cost:"$B/mission", supply:"tons/yr", population:"residents", launches:"missions/yr" };
    ctx += `\n${dim.toUpperCase()} (${units[dim]}):\n`;
    for (const dest of ["moon","mars","outer"]) {
      ctx += `  ${dest}: `;
      snapYears.forEach(y => {
        const i = YEARS.indexOf(y);
        const d = RAW[dim][dest];
        ctx += `${y}→[opt:${d.opt[i]}, real:${d.real[i]}, pess:${d.pess[i]}] `;
      });
      ctx += "\n";
    }
  }
  ctx += "\nSCENARIO ASSUMPTIONS:\n";
  ctx += "- Optimistic: rapid private-sector cost reduction, sustained government funding, no major failures.\n";
  ctx += "- Realistic: follows historical NASA/ESA/SpaceX cost curves with moderate tech advancement.\n";
  ctx += "- Pessimistic: geopolitical friction, budget constraints, technical setbacks.\n";
  ctx += "- Outer Solar System includes Europa/Titan research bases, asteroid belt mining stations, deep-space relays.\n";
  ctx += "- Population = permanent or long-duration residents only (transient crew excluded).\n";
  ctx += "- All costs in constant 2025 USD. Supply tonnage = net delivered payload.\n\n";
  ctx += "Answer questions about this model analytically, referencing specific numbers when relevant. Be concise but insightful. Format your response in plain text, avoiding markdown headers or bullet symbols — use plain sentences and paragraphs. Keep answers to 3–5 sentences unless the question genuinely requires more depth.";
  return ctx;
};

const SYSTEM_PROMPT = buildDataContext();

export default function SpaceModel() {
  const [activeModel, setActiveModel] = useState("cost");
  const [activeDest,  setActiveDest]  = useState("moon");
  const [activeTab,   setActiveTab]   = useState("model"); // "model" | "ask"

  // Chat state
  const [messages,  setMessages]  = useState([]);
  const [input,     setInput]     = useState("");
  const [loading,   setLoading]   = useState(false);
  const chatEndRef = useRef(null);
  const inputRef   = useRef(null);

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior:"smooth" });
  }, [messages, loading]);

  const modelCfg = MODELS.find(m => m.key === activeModel);
  const destCfg  = DEST_CONFIG[activeDest];

  const chartData = useMemo(() => {
    const src = RAW[activeModel][activeDest];
    return YEARS.map((y,i) => ({ year:y, opt:src.opt[i], real:src.real[i], pess:src.pess[i] }));
  }, [activeModel, activeDest]);

  const compareIdx  = YEARS.indexOf(2050);
  const compareData = useMemo(() =>
    ["opt","real","pess"].map(s => ({
      scenario: SCENARIO[s].label,
      Moon:  RAW[activeModel].moon[s][compareIdx]  || 0,
      Mars:  RAW[activeModel].mars[s][compareIdx]  || 0,
      Outer: RAW[activeModel].outer[s][compareIdx] || 0,
    })), [activeModel]);

  const milestoneYears = [2030,2035,2040,2050,2060,2075];
  const milestones = milestoneYears.map(y => {
    const i = YEARS.indexOf(y);
    const s = RAW[activeModel][activeDest];
    return { year:y, opt:s.opt[i], real:s.real[i], pess:s.pess[i] };
  });

  const kpiIdx = YEARS.length - 1;
  const kpiSrc = RAW[activeModel][activeDest];
  const kpis = [
    { label:"Optimistic 2075",  val: kpiSrc.opt[kpiIdx],  color: SCENARIO.opt.color },
    { label:"Realistic 2075",   val: kpiSrc.real[kpiIdx], color: SCENARIO.real.color },
    { label:"Pessimistic 2075", val: kpiSrc.pess[kpiIdx], color: SCENARIO.pess.color },
    { label:"Scenario Spread",  val: kpiSrc.opt[kpiIdx] - kpiSrc.pess[kpiIdx], color:"#94A3B8" },
  ];

  const sendMessage = async (text) => {
    const q = (text || input).trim();
    if (!q || loading) return;
    setInput("");
    const newMessages = [...messages, { role:"user", content: q }];
    setMessages(newMessages);
    setLoading(true);

    try {
      const response = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: SYSTEM_PROMPT,
          messages: newMessages.map(m => ({ role: m.role, content: m.content })),
        }),
      });
      const data = await response.json();
      const reply = data.content?.map(b => b.text || "").join("") || "No response received.";
      setMessages(prev => [...prev, { role:"assistant", content: reply }]);
    } catch (err) {
      setMessages(prev => [...prev, { role:"assistant", content: "Sorry, there was an error connecting to the AI. Please try again." }]);
    } finally {
      setLoading(false);
    }
  };

  const handleKey = (e) => {
    if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); sendMessage(); }
  };

  // ── STYLES ──
  const S = {
    card: { background:"#0D1526", border:"1px solid #1E3A5F", borderRadius:12, padding:"18px 20px" },
    tabBtn: (active) => ({
      padding:"9px 20px", borderRadius:"8px 8px 0 0", cursor:"pointer", fontSize:13, fontWeight: active ? 600 : 400,
      border: "1px solid #1E3A5F", borderBottom: active ? "1px solid #0D1526" : "1px solid #1E3A5F",
      background: active ? "#0D1526" : "#060B18",
      color: active ? "#93C5FD" : "#4A5568",
      marginBottom: active ? -1 : 0,
      transition:"all 0.15s",
      position:"relative", zIndex: active ? 2 : 1,
    }),
  };

  return (
    <div style={{ background:"#060B18", minHeight:"100vh", color:"#CBD5E1", fontFamily:"'Segoe UI', system-ui, sans-serif" }}>

      {/* HEADER */}
      <div style={{ background:"linear-gradient(135deg,#0A1428 0%,#0D1E3D 60%,#0A1428 100%)", borderBottom:"1px solid #1E3A5F", padding:"22px 28px 18px", position:"relative", overflow:"hidden" }}>
        {[...Array(28)].map((_,i) => (
          <div key={i} style={{ position:"absolute", borderRadius:"50%",
            width: i%4===0?2:1, height: i%4===0?2:1, background:"#fff",
            opacity: 0.1+(i%5)*0.07, top:`${(i*37)%100}%`, left:`${(i*53)%100}%` }}/>
        ))}
        <div style={{ position:"relative", zIndex:1 }}>
          <div style={{ fontSize:10, letterSpacing:4, color:"#3B82F6", textTransform:"uppercase", marginBottom:5 }}>
            Research Tool · Space Expansion Prediction Model
          </div>
          <h1 style={{ margin:0, fontSize:22, fontWeight:700, color:"#E2E8F0", letterSpacing:"-0.3px" }}>
            Interplanetary Logistics Forecaster
          </h1>
          <p style={{ margin:"5px 0 0", fontSize:12, color:"#475569" }}>
            Scenario-based projections 2025–2075 · Cost · Supply Chain · Colony Growth · Fleet Expansion
          </p>
        </div>
      </div>

      <div style={{ padding:"22px 28px 0", maxWidth:1100 }}>

        {/* TOP-LEVEL TABS */}
        <div style={{ display:"flex", gap:4, marginBottom:0, borderBottom:"1px solid #1E3A5F" }}>
          <button style={S.tabBtn(activeTab==="model")} onClick={() => setActiveTab("model")}>
            📊 Model & Projections
          </button>
          <button style={S.tabBtn(activeTab==="ask")} onClick={() => setActiveTab("ask")}>
            🤖 Ask the Model
          </button>
        </div>
      </div>

      {/* ═══════════════════════════════════════════════════════ */}
      {/* TAB: MODEL                                             */}
      {/* ═══════════════════════════════════════════════════════ */}
      {activeTab === "model" && (
        <div style={{ padding:"22px 28px", maxWidth:1100 }}>

          {/* MODEL TABS */}
          <div style={{ display:"flex", gap:8, marginBottom:18, flexWrap:"wrap" }}>
            {MODELS.map(m => {
              const active = activeModel === m.key;
              return (
                <button key={m.key} onClick={() => setActiveModel(m.key)} style={{
                  padding:"8px 16px", borderRadius:8, cursor:"pointer", fontSize:13, fontWeight: active?600:400,
                  border: active?"1px solid #3B82F6":"1px solid #1E3A5F",
                  background: active?"#0D2448":"transparent",
                  color: active?"#93C5FD":"#4A5568", transition:"all 0.15s",
                }}>
                  {m.icon} {m.label}
                </button>
              );
            })}
          </div>

          {/* DESTINATION PILLS */}
          <div style={{ display:"flex", gap:8, marginBottom:22, flexWrap:"wrap" }}>
            {Object.entries(DEST_CONFIG).map(([key, cfg]) => {
              const active = activeDest === key;
              return (
                <button key={key} onClick={() => setActiveDest(key)} style={{
                  padding:"5px 14px", borderRadius:20, cursor:"pointer", fontSize:12,
                  fontWeight: active?600:400,
                  border: active?`1px solid ${cfg.color}`:"1px solid #1E3A5F",
                  background: active?`${cfg.color}20`:"transparent",
                  color: active?cfg.color:"#4A5568",
                }}>
                  {cfg.icon} {cfg.label}
                </button>
              );
            })}
          </div>

          {/* KPI STRIP */}
          <div style={{ display:"grid", gridTemplateColumns:"repeat(4,1fr)", gap:12, marginBottom:22 }}>
            {kpis.map(k => (
              <div key={k.label} style={{ ...S.card, padding:"14px 16px" }}>
                <div style={{ fontSize:11, color:"#475569", marginBottom:4 }}>{k.label}</div>
                <div style={{ fontSize:22, fontWeight:700, fontFamily:"monospace", color:k.color }}>{fmt(k.val)}</div>
                <div style={{ fontSize:10, color:"#334155", marginTop:3 }}>{modelCfg.unit}</div>
              </div>
            ))}
          </div>

          {/* MAIN CHART */}
          <div style={{ ...S.card, padding:"20px 16px 14px", marginBottom:20 }}>
            <div style={{ display:"flex", justifyContent:"space-between", alignItems:"flex-start", marginBottom:14, flexWrap:"wrap", gap:8 }}>
              <div>
                <span style={{ fontSize:15, fontWeight:600, color:"#E2E8F0" }}>
                  {destCfg.icon} {destCfg.label} · {modelCfg.label}
                </span>
                <span style={{ fontSize:11, color:"#3B82F6", marginLeft:10 }}>{modelCfg.unit}</span>
                <div style={{ fontSize:11, color:"#334155", marginTop:3 }}>{modelCfg.desc}</div>
              </div>
              <div style={{ display:"flex", gap:16 }}>
                {Object.entries(SCENARIO).map(([k,s]) => (
                  <div key={k} style={{ display:"flex", alignItems:"center", gap:6 }}>
                    <div style={{ width:20, height:3, borderRadius:2, background:s.color }}/>
                    <span style={{ fontSize:11, color:"#64748B" }}>{s.label}</span>
                  </div>
                ))}
              </div>
            </div>
            <ResponsiveContainer width="100%" height={300}>
              <AreaChart data={chartData} margin={{ top:6, right:16, left:8, bottom:4 }}>
                <defs>
                  <linearGradient id="gOpt" x1="0" y1="0" x2="0" y2="1">
                    <stop offset="0%" stopColor="#4ADE80" stopOpacity={0.18}/>
                    <stop offset="100%" stopColor="#4ADE80" stopOpacity={0.02}/>
                  </linearGradient>
                  <linearGradient id="gPess" x1="0" y1="0" x2="0" y2="1">
                    <stop offset="0%" stopColor="#F87171" stopOpacity={0.08}/>
                    <stop offset="100%" stopColor="#F87171" stopOpacity={0.01}/>
                  </linearGradient>
                </defs>
                <CartesianGrid strokeDasharray="3 3" stroke="#1A2E4A" />
                <XAxis dataKey="year" stroke="#1E3A5F" tick={{ fill:"#4A5568", fontSize:11 }} />
                <YAxis stroke="#1E3A5F" tick={{ fill:"#4A5568", fontSize:11 }} tickFormatter={fmtShort} width={52} />
                <Tooltip content={<CustomTooltip unit={modelCfg.unit} />} />
                <Area type="monotone" dataKey="opt"  stroke="none" fill="url(#gOpt)"  legendType="none" />
                <Area type="monotone" dataKey="pess" stroke="none" fill="url(#gPess)" legendType="none" />
                <Line type="monotone" dataKey="pess" stroke={SCENARIO.pess.color} strokeWidth={1.8} strokeDasharray="5 4" dot={false} />
                <Line type="monotone" dataKey="real" stroke={SCENARIO.real.color} strokeWidth={2.5} dot={{ r:3, fill:SCENARIO.real.color }} />
                <Line type="monotone" dataKey="opt"  stroke={SCENARIO.opt.color}  strokeWidth={1.8} strokeDasharray="5 4" dot={false} />
              </AreaChart>
            </ResponsiveContainer>
          </div>

          {/* BOTTOM ROW */}
          <div style={{ display:"grid", gridTemplateColumns:"1.1fr 0.9fr", gap:20, marginBottom:20 }}>
            <div style={S.card}>
              <div style={{ fontSize:13, fontWeight:600, color:"#93C5FD", marginBottom:14 }}>
                Projection Milestones — {destCfg.label}
              </div>
              <table style={{ width:"100%", borderCollapse:"collapse", fontSize:12 }}>
                <thead>
                  <tr>
                    {["Year","Optimistic","Realistic","Pessimistic"].map((h,i) => (
                      <th key={h} style={{ textAlign:i===0?"left":"right", color:["#475569","#4ADE80","#FBBF24","#F87171"][i], paddingBottom:8, fontWeight:500, borderBottom:"1px solid #1E3A5F" }}>{h}</th>
                    ))}
                  </tr>
                </thead>
                <tbody>
                  {milestones.map(row => (
                    <tr key={row.year}>
                      <td style={{ padding:"7px 0", color:"#94A3B8", fontFamily:"monospace", borderBottom:"1px solid #0F1E34" }}>{row.year}</td>
                      <td style={{ textAlign:"right", color:SCENARIO.opt.color,  fontFamily:"monospace", borderBottom:"1px solid #0F1E34" }}>{fmt(row.opt)}</td>
                      <td style={{ textAlign:"right", color:SCENARIO.real.color, fontFamily:"monospace", borderBottom:"1px solid #0F1E34" }}>{fmt(row.real)}</td>
                      <td style={{ textAlign:"right", color:SCENARIO.pess.color, fontFamily:"monospace", borderBottom:"1px solid #0F1E34" }}>{fmt(row.pess)}</td>
                    </tr>
                  ))}
                </tbody>
              </table>
              <div style={{ fontSize:10, color:"#2D4A6A", marginTop:10 }}>Unit: {modelCfg.unit}</div>
            </div>

            <div style={S.card}>
              <div style={{ fontSize:13, fontWeight:600, color:"#93C5FD", marginBottom:4 }}>Destination Comparison · 2050</div>
              <div style={{ fontSize:11, color:"#334155", marginBottom:14 }}>{modelCfg.label} by destination across all scenarios</div>
              <ResponsiveContainer width="100%" height={198}>
                <BarChart data={compareData} margin={{ top:4, right:8, left:0, bottom:4 }}>
                  <CartesianGrid strokeDasharray="3 3" stroke="#1A2E4A" />
                  <XAxis dataKey="scenario" tick={{ fill:"#4A5568", fontSize:10 }} stroke="#1E3A5F" />
                  <YAxis tick={{ fill:"#4A5568", fontSize:10 }} stroke="#1E3A5F" tickFormatter={fmtShort} width={44} />
                  <Tooltip contentStyle={{ background:"#0D1A2E", border:"1px solid #1E3A5F", borderRadius:8, fontSize:11 }}
                    formatter={(v) => [fmt(v)+" "+modelCfg.unit]} />
                  <Bar dataKey="Moon"  fill={DEST_CONFIG.moon.color}  radius={[3,3,0,0]} />
                  <Bar dataKey="Mars"  fill={DEST_CONFIG.mars.color}  radius={[3,3,0,0]} />
                  <Bar dataKey="Outer" fill={DEST_CONFIG.outer.color} radius={[3,3,0,0]} />
                  <Legend wrapperStyle={{ fontSize:11, color:"#94A3B8" }} />
                </BarChart>
              </ResponsiveContainer>
            </div>
          </div>

          <div style={{ background:"#090F1C", border:"1px solid #152135", borderRadius:8, padding:"14px 20px", fontSize:11, color:"#334155", lineHeight:1.7 }}>
            <span style={{ color:"#4A5568", fontWeight:600 }}>Model assumptions · </span>
            <strong style={{ color:"#4ADE80", fontWeight:500 }}>Optimistic</strong>: Rapid private-sector learning curves, sustained government funding, and no major systemic failures.&nbsp;
            <strong style={{ color:"#FBBF24", fontWeight:500 }}>Realistic</strong>: Follows historical NASA/ESA/SpaceX cost curves with moderate but uneven technological progress.&nbsp;
            <strong style={{ color:"#F87171", fontWeight:500 }}>Pessimistic</strong>: Geopolitical friction, budget constraints, and significant technical setbacks.&nbsp;
            Population = permanent residents only. All costs in constant 2025 USD.
          </div>
        </div>
      )}

      {/* ═══════════════════════════════════════════════════════ */}
      {/* TAB: ASK THE MODEL                                     */}
      {/* ═══════════════════════════════════════════════════════ */}
      {activeTab === "ask" && (
        <div style={{ padding:"22px 28px", maxWidth:1100 }}>
          <div style={{ display:"grid", gridTemplateColumns:"1fr 300px", gap:20, alignItems:"start" }}>

            {/* CHAT PANEL */}
            <div style={{ ...S.card, display:"flex", flexDirection:"column", height:580 }}>
              <div style={{ display:"flex", alignItems:"center", gap:10, marginBottom:16, paddingBottom:14, borderBottom:"1px solid #1E3A5F" }}>
                <div style={{ width:34, height:34, borderRadius:"50%", background:"linear-gradient(135deg,#1E40AF,#7C3AED)", display:"flex", alignItems:"center", justifyContent:"center", fontSize:16 }}>🛰️</div>
                <div>
                  <div style={{ fontSize:14, fontWeight:600, color:"#E2E8F0" }}>Mission Analyst</div>
                  <div style={{ fontSize:11, color:"#4A9EF5" }}>Powered by Claude · Full model data loaded</div>
                </div>
              </div>

              {/* Messages */}
              <div style={{ flex:1, overflowY:"auto", display:"flex", flexDirection:"column", gap:14, paddingRight:4, marginBottom:14 }}>
                {messages.length === 0 && (
                  <div style={{ margin:"auto", textAlign:"center", color:"#2D4A6A", padding:"30px 20px" }}>
                    <div style={{ fontSize:36, marginBottom:12 }}>🛸</div>
                    <div style={{ fontSize:14, color:"#3B5A8A", marginBottom:6 }}>Ask anything about the model</div>
                    <div style={{ fontSize:12, color:"#2D4A6A" }}>Costs, populations, logistics, scenario differences — the full dataset is loaded.</div>
                  </div>
                )}
                {messages.map((m, i) => (
                  <div key={i} style={{ display:"flex", gap:10, flexDirection: m.role==="user"?"row-reverse":"row" }}>
                    <div style={{
                      width:28, height:28, borderRadius:"50%", flexShrink:0,
                      background: m.role==="user" ? "linear-gradient(135deg,#1D4ED8,#2563EB)" : "linear-gradient(135deg,#1E40AF,#7C3AED)",
                      display:"flex", alignItems:"center", justifyContent:"center", fontSize:13,
                    }}>
                      {m.role==="user" ? "👤" : "🛰️"}
                    </div>
                    <div style={{
                      maxWidth:"80%", padding:"10px 14px", borderRadius: m.role==="user"?"12px 4px 12px 12px":"4px 12px 12px 12px",
                      background: m.role==="user" ? "#0D2448" : "#0A1A2E",
                      border: `1px solid ${m.role==="user"?"#1E3A5F":"#152240"}`,
                      fontSize:13, lineHeight:1.65, color: m.role==="user"?"#93C5FD":"#CBD5E1",
                    }}>
                      {m.content}
                    </div>
                  </div>
                ))}
                {loading && (
                  <div style={{ display:"flex", gap:10 }}>
                    <div style={{ width:28, height:28, borderRadius:"50%", background:"linear-gradient(135deg,#1E40AF,#7C3AED)", display:"flex", alignItems:"center", justifyContent:"center", fontSize:13 }}>🛰️</div>
                    <div style={{ padding:"10px 16px", borderRadius:"4px 12px 12px 12px", background:"#0A1A2E", border:"1px solid #152240" }}>
                      <div style={{ display:"flex", gap:5, alignItems:"center" }}>
                        {[0,1,2].map(d => (
                          <div key={d} style={{
                            width:6, height:6, borderRadius:"50%", background:"#3B82F6",
                            animation:"pulse 1.2s infinite", animationDelay:`${d*0.2}s`,
                          }}/>
                        ))}
                      </div>
                    </div>
                  </div>
                )}
                <div ref={chatEndRef} />
              </div>

              {/* Input */}
              <div style={{ display:"flex", gap:8, paddingTop:12, borderTop:"1px solid #1E3A5F" }}>
                <textarea
                  ref={inputRef}
                  value={input}
                  onChange={e => setInput(e.target.value)}
                  onKeyDown={handleKey}
                  placeholder="Ask about any projection, scenario, or metric..."
                  rows={2}
                  style={{
                    flex:1, background:"#060B18", border:"1px solid #1E3A5F", borderRadius:8,
                    color:"#CBD5E1", fontSize:13, padding:"10px 12px", resize:"none", outline:"none",
                    fontFamily:"inherit", lineHeight:1.5,
                  }}
                />
                <button
                  onClick={() => sendMessage()}
                  disabled={loading || !input.trim()}
                  style={{
                    padding:"0 18px", borderRadius:8, cursor: loading||!input.trim()?"not-allowed":"pointer",
                    background: loading||!input.trim() ? "#0D1526" : "linear-gradient(135deg,#1D4ED8,#2563EB)",
                    border:"1px solid #1E3A5F", color: loading||!input.trim() ? "#334155" : "#fff",
                    fontSize:18, transition:"all 0.15s",
                  }}>
                  ↑
                </button>
              </div>
              <div style={{ fontSize:10, color:"#1E3A5F", marginTop:6, textAlign:"center" }}>
                Press Enter to send · Shift+Enter for new line
              </div>
            </div>

            {/* SUGGESTED QUESTIONS */}
            <div>
              <div style={{ ...S.card, marginBottom:14 }}>
                <div style={{ fontSize:12, fontWeight:600, color:"#64748B", marginBottom:12, letterSpacing:1, textTransform:"uppercase" }}>
                  Suggested Questions
                </div>
                <div style={{ display:"flex", flexDirection:"column", gap:8 }}>
                  {SUGGESTED.map((q,i) => (
                    <button key={i} onClick={() => { setInput(q); inputRef.current?.focus(); }} style={{
                      textAlign:"left", padding:"10px 12px", borderRadius:8, cursor:"pointer",
                      background:"#060B18", border:"1px solid #1E3A5F", color:"#64748B",
                      fontSize:12, lineHeight:1.5, transition:"all 0.15s",
                    }}
                    onMouseEnter={e => { e.currentTarget.style.borderColor="#3B82F6"; e.currentTarget.style.color="#93C5FD"; }}
                    onMouseLeave={e => { e.currentTarget.style.borderColor="#1E3A5F"; e.currentTarget.style.color="#64748B"; }}>
                      {q}
                    </button>
                  ))}
                </div>
              </div>

              <div style={{ ...S.card, background:"#090F1C" }}>
                <div style={{ fontSize:12, fontWeight:600, color:"#64748B", marginBottom:10, letterSpacing:1, textTransform:"uppercase" }}>
                  What it knows
                </div>
                {[
                  ["💰","Cost data","2025–2075, all 3 destinations"],
                  ["📦","Supply chain","Cargo tonnage projections"],
                  ["👥","Population","Colony growth trajectories"],
                  ["🚀","Launch freq.","Mission cadence forecasts"],
                  ["🎯","3 scenarios","Opt / Realistic / Pessimistic"],
                ].map(([icon, label, sub]) => (
                  <div key={label} style={{ display:"flex", gap:8, alignItems:"center", marginBottom:8 }}>
                    <span style={{ fontSize:14 }}>{icon}</span>
                    <div>
                      <div style={{ fontSize:11, color:"#475569", fontWeight:500 }}>{label}</div>
                      <div style={{ fontSize:10, color:"#334155" }}>{sub}</div>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </div>

          <style>{`
            @keyframes pulse {
              0%, 80%, 100% { opacity: 0.2; transform: scale(0.8); }
              40% { opacity: 1; transform: scale(1); }
            }
          `}</style>
        </div>
      )}
    </div>
  );
}
