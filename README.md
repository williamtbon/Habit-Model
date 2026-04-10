# Habit-Model
import React, { useCallback, useEffect, useMemo, useRef, useState } from "react";
import {
  BookOpen, Calculator, Code2, Dumbbell, NotebookPen, Trophy,
  Wine, Cigarette, RotateCcw, Flame, Target, TrendingUp, CheckCheck,
} from "lucide-react";

const DAYS_SHORT = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
const DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];

const HABITS = [
  { key: "reducedSmoking", label: "Reduced Smoking", target: "≤ 1 / day",   icon: Cigarette,  color: "#f97316", category: "discipline" },
  { key: "noDrinking",     label: "No Drinking",     target: "0 drinks",    icon: Wine,        color: "#a855f7", category: "discipline" },
  { key: "exercise",       label: "Exercise",        target: "≥ 30 min",    icon: Dumbbell,    color: "#22c55e", category: "health"     },
  { key: "journaling",     label: "Journaling",      target: "Entry done",  icon: NotebookPen, color: "#f59e0b", category: "health"     },
  { key: "code",           label: "Coding",          target: "Session done",icon: Code2,       color: "#3b82f6", category: "learning"   },
  { key: "math",           label: "Math",            target: "Session done",icon: Calculator,  color: "#ec4899", category: "learning"   },
  { key: "reading",        label: "Reading",         target: "≥ 20 pages",  icon: BookOpen,    color: "#14b8a6", category: "learning"   },
  { key: "studying",       label: "Studying",        target: "Session done",icon: Trophy,      color: "#eab308", category: "learning"   },
];

const STORAGE_KEY = "work-habit-schedule-v5";
const HISTORY_KEY = "work-habit-history-v1";
const AI_KEY_STORAGE = "work-habit-ai-key";
const SOLID_DAY_THRESHOLD = 6;

const FEEDBACK_MSGS = [
  "🔥 Keep the streak alive!",
  "💪 That's what discipline looks like!",
  "⚡ You're building momentum!",
  "🎯 One step closer to your goal!",
  "✨ Consistency is your superpower!",
  "🚀 Making it happen!",
  "💡 Another win for the books!",
  "🏆 Champions show up every day!",
  "🌊 In the flow state now!",
  "🎖️ Elite habits, elite results!",
];

function buildInitialState() {
  return DAYS.reduce((acc, day) => {
    acc[day] = HABITS.reduce((h, habit) => { h[habit.key] = false; return h; }, {});
    acc[day].notes = "";
    return acc;
  }, {});
}

/* ─── AGI helpers ─── */
function exportHabitContext(schedule, weekStats) {
  return {
    week: schedule,
    summary: weekStats,
    habits: HABITS.map(({ key, label, category, target }) => ({ key, label, category, target })),
  };
}

async function fetchAIMessage(prompt, apiKey, maxTokens = 400) {
  const res = await fetch("https://api.openai.com/v1/chat/completions", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey}` },
    body: JSON.stringify({
      model: "gpt-4o-mini",
      messages: [{ role: "user", content: prompt }],
      max_tokens: maxTokens,
      temperature: 0.8,
    }),
  });
  if (!res.ok) {
    const body = await res.text().catch(() => "");
    throw new Error(`OpenAI ${res.status} ${res.statusText}${body ? `: ${body}` : ""}`);
  }
  const data = await res.json();
  return data.choices[0].message.content.trim();
}

/* ─── Toast ─── */
function Toast({ toasts }) {
  return (
    <div className="pointer-events-none fixed bottom-6 right-6 z-50 flex flex-col-reverse gap-2">
      {toasts.map((t) => (
        <div
          key={t.id}
          className="rounded-2xl border border-white/20 px-4 py-3 text-sm font-semibold text-white shadow-2xl"
          style={{
            background: "linear-gradient(135deg, #0f2f96, #1e3a8a)",
            animation: "toastIn 0.3s ease-out forwards",
          }}
        >
          {t.message}
        </div>
      ))}
    </div>
  );
}

/* ─── SVG Ring ─── */
function Ring({ pct, size = 52, stroke = 5, color = "#3b82f6", children }) {
  const r = (size - stroke * 2) / 2;
  const circ = 2 * Math.PI * r;
  const dash = (Math.min(100, Math.max(0, pct)) / 100) * circ;
  const ringColor = pct === 100 ? "#22c55e" : color;
  return (
    <div className="relative grid flex-shrink-0 place-items-center" style={{ width: size, height: size }}>
      <svg width={size} height={size} className="-rotate-90" style={{ position: "absolute" }}>
        <circle cx={size / 2} cy={size / 2} r={r} strokeWidth={stroke} stroke="rgba(255,255,255,0.08)" fill="none" />
        <circle
          cx={size / 2} cy={size / 2} r={r}
          strokeWidth={stroke} stroke={ringColor} fill="none"
          strokeDasharray={`${dash} ${circ}`} strokeLinecap="round"
          style={{ transition: "stroke-dasharray 0.6s ease-out, stroke 0.4s" }}
        />
      </svg>
      <div className="relative z-10 text-center">{children}</div>
    </div>
  );
}

/* ─── Horizontal analysis progress bar ─── */
function AnalysisBar({ pct, color }) {
  return (
    <div className="h-2.5 w-full overflow-hidden rounded-full bg-white/8">
      <div
        className="h-full rounded-full transition-all duration-700 ease-out"
        style={{
          width: `${pct}%`,
          background: pct === 100
            ? "linear-gradient(90deg, #16a34a, #22c55e)"
            : `linear-gradient(90deg, ${color}99, ${color})`,
          boxShadow: pct > 0 ? `0 0 6px ${color}60` : "none",
        }}
      />
    </div>
  );
}

/* ─── CSS bar chart ─── */
function DayBarChart({ dayStats }) {
  const maxDone = Math.max(...dayStats.map((d) => d.done), 1);
  return (
    <div className="flex h-28 items-end gap-1.5">
      {dayStats.map((item, i) => {
        const isPerfect = item.done === HABITS.length;
        const isGood    = item.done >= SOLID_DAY_THRESHOLD;
        const barColor  = isPerfect ? "#22c55e" : isGood ? "#3b82f6" : item.done > 0 ? "#6366f1" : "rgba(255,255,255,0.05)";
        const heightPct = item.done === 0 ? 3 : Math.round((item.done / maxDone) * 100);
        return (
          <div key={item.day} className="group flex flex-1 flex-col items-center gap-1">
            <div className="text-[9px] font-bold opacity-0 transition-opacity group-hover:opacity-100" style={{ color: barColor }}>
              {item.done}/{HABITS.length}
            </div>
            <div className="relative flex w-full flex-1 items-end">
              <div
                className="w-full rounded-t transition-all duration-700 ease-out"
                style={{
                  height: `${heightPct}%`, background: barColor, minHeight: 3,
                  boxShadow: item.done > 0 ? `0 0 8px ${barColor}50` : "none",
                }}
              />
            </div>
            <div className="text-[10px] font-semibold text-slate-400">{DAYS_SHORT[i]}</div>
          </div>
        );
      })}
    </div>
  );
}

/* ─── Main component ─── */
export default function HabitDashboard() {
  const [weekData, setWeekData]         = useState(buildInitialState);
  const [activeDay, setActiveDay]       = useState(null);
  const [justToggled, setJustToggled]   = useState(null);
  const [toasts, setToasts]             = useState([]);
  const toastCounter = useRef(0);

  const [aiKey, setAiKey]               = useState(() => {
    try { return localStorage.getItem(AI_KEY_STORAGE) || ""; } catch (e) { console.error(e); return ""; }
  });
  const [showKeyInput, setShowKeyInput] = useState(false);
  const [history, setHistory]           = useState([]);
  const [aiInsight, setAiInsight]       = useState("");
  const [aiInsightLoading, setAiInsightLoading] = useState(false);
  const [chatMessages, setChatMessages] = useState([]);
  const [chatInput, setChatInput]       = useState("");
  const [chatLoading, setChatLoading]   = useState(false);

  useEffect(() => {
    try { const s = localStorage.getItem(STORAGE_KEY); if (s) setWeekData(JSON.parse(s)); }
    catch (e) { console.error(e); }
  }, []);

  useEffect(() => {
    try { localStorage.setItem(STORAGE_KEY, JSON.stringify(weekData)); }
    catch (e) { console.error(e); }
  }, [weekData]);

  useEffect(() => {
    try { const h = localStorage.getItem(HISTORY_KEY); if (h) setHistory(JSON.parse(h)); }
    catch (e) { console.error(e); }
  }, []);

  useEffect(() => {
    try { localStorage.setItem(AI_KEY_STORAGE, aiKey); }
    catch (e) { console.error(e); }
  }, [aiKey]);

  const addToast = useCallback((msg) => {
    const id = ++toastCounter.current;
    setToasts((t) => [...t.slice(-3), { id, message: msg }]);
    setTimeout(() => setToasts((t) => t.filter((x) => x.id !== id)), 3400);
  }, []);

  const toggleHabit = (day, key) => {
    // Compute derived values from the current snapshot — avoids side effects inside the updater
    const wasOff     = !weekData[day]?.[key];
    const nextDay    = { ...weekData[day], [key]: !weekData[day]?.[key] };
    const newDayDone = HABITS.filter((h) => nextDay[h.key]).length;

    setWeekData((prev) => ({ ...prev, [day]: { ...prev[day], [key]: !prev[day][key] } }));
    setJustToggled(`${day}-${key}`);
    setTimeout(() => setJustToggled(null), 380);

    // Side effects run outside the updater so React Strict Mode double-invoke doesn't double-fire
    setTimeout(() => {
      if (!wasOff) return; // only show toast when checking, not unchecking
      if (newDayDone === HABITS.length) {
        addToast("🌟 PERFECT DAY — every single habit nailed!");
      } else if (newDayDone === SOLID_DAY_THRESHOLD) {
        addToast("🔒 Locked in! 6 habits crushed today.");
      } else if (aiKey.trim()) {
        const habitLabel = HABITS.find((h) => h.key === key)?.label || key;
        const ctx = exportHabitContext(
          { [day]: nextDay },
          { dayDone: newDayDone, day },
        );
        const prompt = `You are a habit coach. In ≤15 words, give one short personalized encouraging message for completing "${habitLabel}" today. Week data: ${JSON.stringify(ctx.summary)}. Reply with just the message, no quotes.`;
        fetchAIMessage(prompt, aiKey.trim(), 60)
          .then((msg) => addToast(msg))
          .catch((err) => {
            if (err.message.includes("401") || err.message.toLowerCase().includes("auth")) {
              addToast("⚠️ OpenAI key rejected. Check your key in ⚙️ AI Key.");
            } else {
              addToast(FEEDBACK_MSGS[Math.floor(Math.random() * FEEDBACK_MSGS.length)]);
            }
          });
      } else {
        addToast(FEEDBACK_MSGS[Math.floor(Math.random() * FEEDBACK_MSGS.length)]);
      }
    }, 80);
  };

  const setNotes = (day, val) =>
    setWeekData((prev) => ({ ...prev, [day]: { ...prev[day], notes: val } }));

  const resetAll = () => {
    if (window.confirm("Reset the entire week?")) {
      setHistory((prev) => {
        const updated = [{ savedAt: new Date().toISOString(), data: weekData }, ...prev].slice(0, 4);
        try { localStorage.setItem(HISTORY_KEY, JSON.stringify(updated)); } catch (e) { console.error(e); }
        return updated;
      });
      setWeekData(buildInitialState());
      setAiInsight("");
      setChatMessages([]);
    }
  };

  /* ── computed ── */
  const totalPossible = DAYS.length * HABITS.length;
  const totalDone = useMemo(
    () => DAYS.reduce((s, d) => s + HABITS.reduce((i, h) => i + (weekData[d]?.[h.key] ? 1 : 0), 0), 0),
    [weekData],
  );
  const overallPct = Math.round((totalDone / totalPossible) * 100);

  const dayStats = useMemo(
    () => DAYS.map((day) => {
      const done = HABITS.filter((h) => weekData[day]?.[h.key]).length;
      return { day, done, pct: Math.round((done / HABITS.length) * 100) };
    }),
    [weekData],
  );

  const habitBreakdown = useMemo(
    () => HABITS.map((habit) => {
      const actual = DAYS.reduce((s, d) => s + (weekData[d]?.[habit.key] ? 1 : 0), 0);
      return { ...habit, actual, pct: Math.round((actual / DAYS.length) * 100) };
    }),
    [weekData],
  );

  const solidDays   = dayStats.filter((d) => d.done >= SOLID_DAY_THRESHOLD).length;
  const perfectDays = dayStats.filter((d) => d.done === HABITS.length).length;
  const bestDay     = [...dayStats].sort((a, b) => b.done - a.done)[0];

  const currentStreak = useMemo(() => {
    let s = 0;
    for (const { done } of dayStats) { if (done > 0) s++; else break; }
    return s;
  }, [dayStats]);

  const weekMsg = overallPct >= 90 ? "🌟 Exceptional week!"
    : overallPct >= 70 ? "💪 Strong progress!"
    : overallPct >= 40 ? "⚡ Keep pushing!"
    : "🎯 Build the habit!";

  const weekStats = { overallPct, totalDone, totalPossible, solidDays, perfectDays, currentStreak, bestDay: bestDay?.day };

  const getWeeklyInsight = async () => {
    if (!aiKey.trim()) { addToast("⚙️ Add your OpenAI key first."); return; }
    setAiInsightLoading(true);
    setAiInsight("");
    try {
      const ctx = exportHabitContext(weekData, weekStats);
      const prompt = `You are a habit coach. Analyze this week's habit data and write a 3-4 sentence coaching summary. Mention specific habits by name, call out both wins and gaps, and give one actionable suggestion for next week. Data: ${JSON.stringify(ctx)}`;
      const msg = await fetchAIMessage(prompt, aiKey.trim());
      setAiInsight(msg);
    } catch (e) {
      const isAuth = e.message.includes("401") || e.message.toLowerCase().includes("auth");
      setAiInsight(isAuth
        ? "⚠️ API key rejected (401). Click ⚙️ AI Key and verify your key is correct."
        : "⚠️ Could not fetch insight. Check your API key and try again.");
    } finally {
      setAiInsightLoading(false);
    }
  };

  const sendChatMessage = async () => {
    const raw = chatInput.trim();
    if (!raw) return;
    if (!aiKey.trim()) { addToast("⚙️ Add your OpenAI key first."); return; }
    // Sanitize: strip control characters and cap at 500 chars to prevent prompt injection / oversized requests
    const q = raw.replace(/[\x00-\x1F\x7F]/g, " ").slice(0, 500);
    setChatMessages((prev) => [...prev, { role: "user", content: q }]);
    setChatInput("");
    setChatLoading(true);
    try {
      const ctx = exportHabitContext(weekData, weekStats);
      const prompt = `You are a habit coach AI. Answer this question using only the week's habit data provided. Be concise (≤3 sentences). Data: ${JSON.stringify(ctx)}\n\nQuestion: ${q}`;
      const reply = await fetchAIMessage(prompt, aiKey.trim());
      setChatMessages((prev) => [...prev, { role: "assistant", content: reply }]);
    } catch (e) {
      const isAuth = e.message.includes("401") || e.message.toLowerCase().includes("auth");
      setChatMessages((prev) => [...prev, {
        role: "assistant",
        content: isAuth
          ? "⚠️ API key rejected (401). Click ⚙️ AI Key and verify your key is correct."
          : "⚠️ Error reaching AI. Check your API key.",
      }]);
    } finally {
      setChatLoading(false);
    }
  };

  /* ── render ── */
  return (
    <div
      className="min-h-screen bg-[#050c22] text-white"
      style={{ fontFamily: "'Inter', system-ui, sans-serif" }}
    >
      {/* Ambient glows */}
      <div className="pointer-events-none fixed inset-0 overflow-hidden">
        <div className="absolute -top-40 left-1/4 h-[600px] w-[600px] rounded-full bg-blue-700/6 blur-[180px]" />
        <div className="absolute bottom-0 right-0 h-[500px] w-[500px] rounded-full bg-violet-700/6 blur-[160px]" />
      </div>

      <Toast toasts={toasts} />

      <div className="relative mx-auto max-w-[1400px] px-4 py-5 md:px-6">

        {/* ══════════════════════════════════════
            HEADER BANNER  (replicates the reference spreadsheet header)
        ══════════════════════════════════════ */}
        <div
          className="mb-4 overflow-hidden rounded-2xl shadow-2xl"
          style={{ background: "linear-gradient(135deg, #0b1f6b 0%, #091645 60%, #050c22 100%)" }}
        >
          {/* Top stripe */}
          <div className="flex flex-wrap items-center justify-between gap-4 border-b border-white/8 px-6 py-4">
            <div className="flex items-center gap-3">
              <div className="rounded-xl bg-blue-500/15 p-2.5">
                <Flame className="h-5 w-5 text-blue-400" />
              </div>
              <div>
                <h1 className="text-lg font-extrabold tracking-tight leading-none">Work Habits Tracker</h1>
                <p className="mt-0.5 text-[10px] uppercase tracking-[0.25em] text-blue-300/40">Weekly Discipline System</p>
              </div>
            </div>
            <div className="flex flex-wrap items-center gap-2">
              <button
                onClick={() => setShowKeyInput((v) => !v)}
                className="flex items-center gap-1.5 rounded-xl border border-white/8 bg-white/4 px-3 py-2 text-[11px] font-medium text-slate-400 transition hover:border-blue-400/30 hover:bg-blue-500/8 hover:text-blue-300"
              >
                ⚙️ AI Key
              </button>
              <button
                onClick={resetAll}
                className="flex items-center gap-1.5 rounded-xl border border-white/8 bg-white/4 px-3 py-2 text-[11px] font-medium text-slate-400 transition hover:border-red-400/30 hover:bg-red-500/8 hover:text-red-300"
              >
                <RotateCcw className="h-3 w-3" />
                Reset Week
              </button>
            </div>
          </div>

          {/* AI key input (collapsible) */}
          {showKeyInput && (
            <div className="flex items-center gap-3 border-b border-white/6 px-6 py-3">
              <span className="flex-shrink-0 text-[11px] text-slate-400">OpenAI API Key</span>
              <input
                type="password"
                value={aiKey}
                onChange={(e) => setAiKey(e.target.value.trim())}
                placeholder="sk-..."
                className="flex-1 rounded-lg border border-white/10 bg-white/4 px-3 py-1.5 text-xs text-slate-200 outline-none placeholder:text-slate-600 focus:border-blue-500/40"
              />
              <button
                onClick={() => setShowKeyInput(false)}
                className="text-[11px] text-slate-500 hover:text-slate-300"
              >
                Done
              </button>
            </div>
          )}

          {/* Stats row — mirrors "Completed habits | Progress | Progress in %" */}
          <div className="grid grid-cols-2 gap-px bg-white/5 md:grid-cols-4">
            {[
              { label: "Completed Habits", value: totalDone,           sub: `out of ${totalPossible}`,     color: "#3b82f6", icon: CheckCheck },
              { label: "Progress",         value: `${overallPct}%`,    sub: weekMsg,                       color: overallPct >= 70 ? "#22c55e" : "#f59e0b", icon: TrendingUp },
              { label: "Current Streak",   value: `${currentStreak}d`, sub: "consecutive days active",     color: "#f97316", icon: Flame },
              { label: "Best Day",         value: bestDay?.day?.slice(0,3) || "—", sub: `${bestDay?.done || 0}/${HABITS.length} habits`, color: "#a855f7", icon: Target },
            ].map(({ label, value, sub, color, icon: Icon }) => (
              <div key={label} className="flex items-center gap-3 bg-[#060e30] px-5 py-4">
                <div className="flex-shrink-0 rounded-lg p-2" style={{ background: `${color}18`, color }}>
                  <Icon className="h-4 w-4" />
                </div>
                <div className="min-w-0">
                  <div className="text-[9px] uppercase tracking-[0.22em] text-slate-500">{label}</div>
                  <div className="mt-0.5 text-xl font-black tracking-tight leading-none" style={{ color }}>{value}</div>
                  <div className="mt-0.5 text-[10px] text-slate-500 truncate">{sub}</div>
                </div>
              </div>
            ))}
          </div>

          {/* Overall progress bar */}
          <div className="px-6 py-3">
            <div className="flex items-center gap-3">
              <div className="flex-1 h-2 overflow-hidden rounded-full bg-white/8">
                <div
                  className="h-full rounded-full transition-all duration-700 ease-out"
                  style={{
                    width: `${overallPct}%`,
                    background: overallPct === 100 ? "linear-gradient(90deg, #16a34a, #22c55e)"
                              : overallPct >= 70    ? "linear-gradient(90deg, #1d4ed8, #3b82f6)"
                              :                      "linear-gradient(90deg, #7c3aed, #a855f7)",
                    boxShadow: overallPct > 0 ? "0 0 12px rgba(59,130,246,0.5)" : "none",
                  }}
                />
              </div>
              <span className="flex-shrink-0 text-xs font-bold text-slate-300">{overallPct}%</span>
            </div>
          </div>
        </div>

        {/* ══════════════════════════════════════
            HABIT GRID TABLE  (spreadsheet style)
        ══════════════════════════════════════ */}
        <div className="mb-4 overflow-hidden rounded-2xl shadow-2xl" style={{ background: "#060e30" }}>
          <div className="overflow-x-auto">
            <table className="min-w-full border-separate border-spacing-0 text-sm">

              {/* ─── Column headers ─── */}
              <thead>
                <tr>
                  {/* Habit name column */}
                  <th
                    className="sticky left-0 z-20 min-w-[180px] border-b border-r border-white/8 px-4 py-3 text-left text-[10px] font-semibold uppercase tracking-widest text-slate-500"
                    style={{ background: "#040b25" }}
                  >
                    Habit / Target
                  </th>

                  {/* Day columns */}
                  {DAYS.map((day, i) => {
                    const stat     = dayStats[i];
                    const isActive = activeDay === day;
                    const isPerfect = stat.done === HABITS.length;
                    const isGood    = stat.done >= SOLID_DAY_THRESHOLD;
                    return (
                      <th
                        key={day}
                        onClick={() => setActiveDay(isActive ? null : day)}
                        className="cursor-pointer select-none border-b border-r border-white/6 px-2 py-3 text-center transition-colors"
                        style={{
                          background: isActive ? "rgba(59,130,246,0.12)" : "#040b25",
                          minWidth: 52,
                        }}
                      >
                        <div className="text-[11px] font-bold text-slate-300">{DAYS_SHORT[i]}</div>
                        <div
                          className="mt-0.5 text-[10px] font-semibold"
                          style={{
                            color: isPerfect ? "#4ade80"
                                 : isGood    ? "#fbbf24"
                                 : stat.done > 0 ? "#818cf8"
                                 : "rgba(148,163,184,0.3)",
                          }}
                        >
                          {stat.done}/{HABITS.length}
                        </div>
                      </th>
                    );
                  })}

                  {/* Analysis columns header */}
                  <th
                    colSpan={3}
                    className="border-b border-l border-white/8 px-4 py-3 text-center text-[10px] font-semibold uppercase tracking-widest text-blue-400"
                    style={{ background: "#091645", minWidth: 200 }}
                  >
                    Analysis
                  </th>
                </tr>

                {/* Sub-header for analysis */}
                <tr>
                  <th className="sticky left-0 z-20 border-b border-r border-white/6 bg-[#040b25]" />
                  {DAYS.map((d) => (
                    <th key={d} className="border-b border-r border-white/4 bg-[#040b25]" />
                  ))}
                  {[
                    { label: "Goal", w: 44 },
                    { label: "Actual", w: 52 },
                    { label: "Progress", w: 110 },
                  ].map(({ label, w }) => (
                    <th
                      key={label}
                      className="border-b border-l border-white/6 px-2 py-1.5 text-[9px] font-semibold uppercase tracking-widest text-slate-500"
                      style={{ background: "#060f36", minWidth: w }}
                    >
                      {label}
                    </th>
                  ))}
                </tr>
              </thead>

              {/* ─── Habit rows ─── */}
              <tbody>
                {HABITS.map((habit, hIdx) => {
                  const { actual, pct } = habitBreakdown[hIdx];
                  const Icon = habit.icon;
                  const rowBg = hIdx % 2 === 0 ? "rgba(255,255,255,0.012)" : "transparent";
                  return (
                    <tr
                      key={habit.key}
                      className="group border-b border-white/[0.04] last:border-0 transition-colors hover:bg-white/[0.025]"
                    >
                      {/* Habit label */}
                      <td
                        className="sticky left-0 z-10 border-r border-white/6 px-4 py-3"
                        style={{ background: `color-mix(in srgb, #040b25 ${hIdx % 2 === 0 ? "97%" : "100%"}, ${habit.color} 0%)` }}
                      >
                        <div className="flex items-center gap-2.5">
                          <div className="flex-shrink-0 rounded-md p-1.5" style={{ background: `${habit.color}18`, color: habit.color }}>
                            <Icon className="h-3.5 w-3.5" />
                          </div>
                          <div>
                            <div className="whitespace-nowrap text-xs font-semibold text-slate-200">{habit.label}</div>
                            <div className="text-[9px] text-slate-600">{habit.target}</div>
                          </div>
                        </div>
                      </td>

                      {/* Day checkbox cells */}
                      {DAYS.map((day) => {
                        const active    = weekData[day]?.[habit.key];
                        const cellKey   = `${day}-${habit.key}`;
                        const popping   = justToggled === cellKey;
                        const highlight = activeDay === day;
                        return (
                          <td
                            key={cellKey}
                            className="border-r border-white/[0.04] px-1.5 py-2.5 text-center transition-colors"
                            style={{
                              background: highlight
                                ? "rgba(59,130,246,0.08)"
                                : active
                                ? `${habit.color}10`
                                : rowBg,
                            }}
                          >
                            <button
                              onClick={() => toggleHabit(day, habit.key)}
                              aria-label={`${habit.label} on ${day}`}
                              aria-pressed={active}
                              className={`mx-auto grid h-7 w-7 place-items-center rounded transition-all duration-200 ${popping ? "scale-125" : "scale-100"}`}
                              style={
                                active
                                  ? {
                                      background: habit.color,
                                      boxShadow: `0 0 10px ${habit.color}55`,
                                    }
                                  : {
                                      background: "rgba(255,255,255,0.04)",
                                      border: "1.5px solid rgba(255,255,255,0.1)",
                                    }
                              }
                            >
                              {active
                                ? <svg viewBox="0 0 12 10" className="h-3 w-3 text-white fill-current"><path d="M1 5l3.5 3.5L11 1" stroke="white" strokeWidth="1.8" fill="none" strokeLinecap="round" strokeLinejoin="round"/></svg>
                                : null
                              }
                            </button>
                          </td>
                        );
                      })}

                      {/* Analysis: Goal */}
                      <td className="border-l border-white/6 px-2 py-2.5 text-center text-xs font-bold text-slate-400" style={{ background: "#040e2a" }}>
                        {DAYS.length}
                      </td>
                      {/* Analysis: Actual */}
                      <td
                        className="px-2 py-2.5 text-center text-xs font-bold"
                        style={{ background: "#040e2a", color: actual === DAYS.length ? "#4ade80" : habit.color }}
                      >
                        {actual}
                      </td>
                      {/* Analysis: Progress bar */}
                      <td className="px-3 py-2.5" style={{ background: "#040e2a" }}>
                        <div className="flex items-center gap-2">
                          <AnalysisBar pct={pct} color={habit.color} />
                          <span className="w-7 flex-shrink-0 text-right text-[10px] font-bold text-slate-400">{pct}%</span>
                        </div>
                      </td>
                    </tr>
                  );
                })}
              </tbody>

              {/* ─── Footer row: per-day completion % ─── */}
              <tfoot>
                <tr>
                  <td
                    className="sticky left-0 z-10 border-r border-t border-white/8 px-4 py-2.5 text-[10px] font-semibold uppercase tracking-widest text-slate-500"
                    style={{ background: "#040b25" }}
                  >
                    Day %
                  </td>
                  {dayStats.map((stat, i) => {
                    const isPerfect = stat.done === HABITS.length;
                    const isGood    = stat.done >= SOLID_DAY_THRESHOLD;
                    const col       = isPerfect ? "#4ade80" : isGood ? "#fbbf24" : stat.done > 0 ? "#818cf8" : "rgba(148,163,184,0.25)";
                    return (
                      <td
                        key={stat.day}
                        className="border-r border-t border-white/6 px-1.5 py-2.5 text-center"
                        style={{ background: "#040b25" }}
                      >
                        <div className="text-[10px] font-bold" style={{ color: col }}>{stat.pct}%</div>
                        <div className="mt-0.5 text-[8px] text-slate-600">{stat.done}/{HABITS.length}</div>
                      </td>
                    );
                  })}
                  <td colSpan={3} className="border-l border-t border-white/6 px-4 py-2.5 text-[10px] text-slate-600" style={{ background: "#040b25" }}>
                    {totalDone} / {totalPossible} total
                  </td>
                </tr>
              </tfoot>
            </table>
          </div>
        </div>

        {/* ══════════════════════════════════════
            BOTTOM SECTION
        ══════════════════════════════════════ */}
        <div className="grid gap-4 lg:grid-cols-[1fr_260px]">

          {/* Left: bar chart */}
          <div className="rounded-2xl border border-white/6 p-5" style={{ background: "#060e30" }}>
            <div className="mb-4 flex items-start justify-between">
              <div>
                <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Analytics</div>
                <h3 className="text-sm font-bold leading-tight text-slate-200">Daily Completion Chart</h3>
              </div>
              <div className="flex gap-3 text-[9px] font-medium text-slate-600">
                <span className="flex items-center gap-1"><span className="inline-block h-2 w-2 rounded-sm bg-green-500" />Perfect</span>
                <span className="flex items-center gap-1"><span className="inline-block h-2 w-2 rounded-sm bg-blue-500" />Solid</span>
                <span className="flex items-center gap-1"><span className="inline-block h-2 w-2 rounded-sm bg-indigo-400" />Progress</span>
              </div>
            </div>
            <DayBarChart dayStats={dayStats} />

            {/* Day notes strip */}
            <div className="mt-4 border-t border-white/5 pt-4">
              <div className="text-[9px] mb-2 uppercase tracking-widest text-slate-600">Daily Notes</div>
              <div className="flex gap-2 overflow-x-auto pb-1">
                {dayStats.map((item, i) => (
                  <div key={item.day} className="w-28 flex-shrink-0">
                    <div className="mb-1 flex items-center justify-between">
                      <span className="text-[10px] font-bold text-slate-400">{DAYS_SHORT[i]}</span>
                      <span className="text-[9px]" style={{ color: item.pct === 100 ? "#4ade80" : item.pct >= 75 ? "#fbbf24" : "#6366f1" }}>{item.pct}%</span>
                    </div>
                    <textarea
                      value={weekData[item.day]?.notes || ""}
                      onChange={(e) => setNotes(item.day, e.target.value)}
                      placeholder="Notes…"
                      rows={2}
                      className="w-full resize-none rounded-lg border border-white/6 bg-white/3 px-2 py-1 text-[9px] text-slate-400 outline-none placeholder:text-slate-700 focus:border-blue-500/30 focus:bg-white/5 transition"
                    />
                  </div>
                ))}
              </div>
            </div>
          </div>

          {/* Right: habit completion rings + weekly score */}
          <div className="rounded-2xl border border-white/6 p-5" style={{ background: "#060e30" }}>
            <div className="mb-4">
              <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Summary</div>
              <h3 className="text-sm font-bold leading-tight text-slate-200">Weekly Score</h3>
            </div>

            {/* Big ring */}
            <div className="mb-4 flex items-center gap-4 rounded-xl border border-white/6 bg-white/3 p-3">
              <Ring pct={overallPct} size={68} stroke={7} color="#3b82f6">
                <span className="text-base font-black text-white">{overallPct}%</span>
              </Ring>
              <div>
                <div className="text-xs font-bold text-slate-200">Overall Progress</div>
                <div className="text-[10px] text-slate-500">{totalDone} / {totalPossible} habits</div>
                <div className="mt-1 text-[10px] font-semibold text-blue-300">{weekMsg}</div>
                <div className="mt-0.5 text-[9px] text-slate-600">{solidDays} solid · {perfectDays} perfect</div>
              </div>
            </div>

            {/* Per-habit mini bars */}
            <div className="space-y-2.5">
              {habitBreakdown.map((h) => {
                const Icon = h.icon;
                return (
                  <div key={h.key}>
                    <div className="mb-1 flex items-center gap-1.5">
                      <div className="rounded p-1 flex-shrink-0" style={{ background: `${h.color}18`, color: h.color }}>
                        <Icon className="h-2.5 w-2.5" />
                      </div>
                      <span className="flex-1 truncate text-[10px] font-medium text-slate-400">{h.label}</span>
                      <span className="text-[10px] font-bold flex-shrink-0" style={{ color: h.color }}>{h.actual}/7</span>
                    </div>
                    <AnalysisBar pct={h.pct} color={h.color} />
                  </div>
                );
              })}
            </div>

            <div className="mt-4 rounded-xl border border-white/5 bg-white/[0.025] p-3">
              <div className="flex items-center gap-1.5 text-[10px] font-semibold text-slate-400">
                <Flame className="h-3 w-3 text-orange-400" /> Weekly Rule
              </div>
              <p className="mt-1 text-[9px] leading-3.5 text-slate-600">
                Stack clean days. Miss one — keep moving. Miss the whole day — restart tomorrow.
              </p>
            </div>
          </div>
        </div>

        {/* ══════════════════════════════════════
            AI INSIGHT PANEL
        ══════════════════════════════════════ */}
        <div className="mt-4 rounded-2xl border border-white/6 p-5" style={{ background: "#060e30" }}>
          <div className="mb-3 flex items-center justify-between">
            <div>
              <div className="text-[9px] uppercase tracking-widest text-blue-400/50">AI Coach</div>
              <h3 className="text-sm font-bold leading-tight text-slate-200">Weekly Insight</h3>
            </div>
            <button
              onClick={getWeeklyInsight}
              disabled={aiInsightLoading}
              className="flex items-center gap-1.5 rounded-xl border border-blue-500/30 bg-blue-500/10 px-3 py-2 text-[11px] font-semibold text-blue-300 transition hover:bg-blue-500/20 disabled:opacity-50 disabled:cursor-not-allowed"
            >
              {aiInsightLoading ? "Thinking…" : "✨ Get AI Insight"}
            </button>
          </div>
          {aiInsight ? (
            <p className="text-xs leading-relaxed text-slate-300">{aiInsight}</p>
          ) : (
            <p className="text-[11px] text-slate-600">Click "Get AI Insight" to receive a personalized weekly coaching summary powered by GPT-4o mini.</p>
          )}
        </div>

        {/* ══════════════════════════════════════
            CHAT / QUERY PANEL
        ══════════════════════════════════════ */}
        <div className="mt-4 rounded-2xl border border-white/6 p-5" style={{ background: "#060e30" }}>
          <div className="mb-3">
            <div className="text-[9px] uppercase tracking-widest text-blue-400/50">AI Coach</div>
            <h3 className="text-sm font-bold leading-tight text-slate-200">Ask About Your Habits</h3>
          </div>

          {/* Message history */}
          <div className="mb-3 max-h-56 space-y-2 overflow-y-auto">
            {chatMessages.length === 0 && (
              <p className="text-[11px] text-slate-600">Try asking: "What's my weakest habit this week?" or "How can I improve my streak?"</p>
            )}
            {chatMessages.map((m, i) => (
              <div
                key={i}
                className={`rounded-xl px-3 py-2 text-xs leading-relaxed ${
                  m.role === "user"
                    ? "ml-8 bg-blue-500/15 text-blue-100"
                    : "mr-8 bg-white/5 text-slate-300"
                }`}
              >
                {m.content}
              </div>
            ))}
            {chatLoading && (
              <div className="mr-8 rounded-xl bg-white/5 px-3 py-2 text-xs text-slate-500">Thinking…</div>
            )}
          </div>

          {/* Input row */}
          <div className="flex gap-2">
            <input
              type="text"
              value={chatInput}
              onChange={(e) => setChatInput(e.target.value)}
              onKeyDown={(e) => { if (e.key === "Enter" && !chatLoading) sendChatMessage(); }}
              placeholder="Ask about your habits…"
              className="flex-1 rounded-xl border border-white/10 bg-white/4 px-3 py-2 text-xs text-slate-200 outline-none placeholder:text-slate-600 focus:border-blue-500/40"
            />
            <button
              onClick={sendChatMessage}
              disabled={chatLoading || !chatInput.trim()}
              className="rounded-xl border border-blue-500/30 bg-blue-500/10 px-4 py-2 text-xs font-semibold text-blue-300 transition hover:bg-blue-500/20 disabled:opacity-40 disabled:cursor-not-allowed"
            >
              Send
            </button>
          </div>
        </div>
      </div>

      <style>{`
        @keyframes toastIn {
          from { opacity: 0; transform: translateX(24px) scale(0.95); }
          to   { opacity: 1; transform: translateX(0) scale(1); }
        }
      `}</style>
    </div>
  );
}
