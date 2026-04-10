# Habit-Model
import React, { useCallback, useEffect, useMemo, useRef, useState } from "react";
import {
  BookOpen, Calculator, Code2, Dumbbell, NotebookPen, Trophy,
  Wine, Cigarette, RotateCcw, Flame, Target, CheckCircle2, Zap, Star,
} from "lucide-react";

const DAYS_SHORT = ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"];
const DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];

const HABITS = [
  { key: "reducedSmoking", label: "Smoking",  target: "≤ 1",      icon: Cigarette,  color: "#f97316" },
  { key: "noDrinking",     label: "Sober",    target: "0 drinks", icon: Wine,        color: "#a855f7" },
  { key: "exercise",       label: "Exercise", target: "≥ 30 min", icon: Dumbbell,    color: "#22c55e" },
  { key: "journaling",     label: "Journal",  target: "Done",     icon: NotebookPen, color: "#f59e0b" },
  { key: "code",           label: "Code",     target: "Done",     icon: Code2,       color: "#3b82f6" },
  { key: "math",           label: "Math",     target: "Done",     icon: Calculator,  color: "#ec4899" },
  { key: "reading",        label: "Read",     target: "Done",     icon: BookOpen,    color: "#14b8a6" },
  { key: "studying",       label: "Study",    target: "Done",     icon: Trophy,      color: "#eab308" },
];

const STORAGE_KEY = "work-habit-schedule-v4";
const SOLID_DAY_THRESHOLD = 6;

const FEEDBACK = [
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
    acc[day].focusBlock = "";
    return acc;
  }, {});
}

/* ─── Toast notifications ─── */
function Toast({ toasts }) {
  return (
    <div className="pointer-events-none fixed bottom-6 right-6 z-50 flex flex-col-reverse gap-2">
      {toasts.map((t) => (
        <div
          key={t.id}
          className="rounded-2xl border border-white/15 px-4 py-3 text-sm font-semibold text-white shadow-2xl"
          style={{
            background: "linear-gradient(135deg, #0d2980, #1e3a8a)",
            animation: "toastIn 0.3s ease-out forwards",
          }}
        >
          {t.message}
        </div>
      ))}
    </div>
  );
}

/* ─── SVG Ring progress indicator ─── */
function Ring({ pct, size = 56, stroke = 5, color = "#3b82f6", children }) {
  const r = (size - stroke * 2) / 2;
  const circ = 2 * Math.PI * r;
  const dash = (Math.min(100, Math.max(0, pct)) / 100) * circ;
  const ringColor = pct === 100 ? "#22c55e" : color;
  return (
    <div className="relative grid place-items-center flex-shrink-0" style={{ width: size, height: size }}>
      <svg width={size} height={size} className="-rotate-90" style={{ position: "absolute" }}>
        <circle cx={size / 2} cy={size / 2} r={r} strokeWidth={stroke} stroke="rgba(255,255,255,0.07)" fill="none" />
        <circle
          cx={size / 2} cy={size / 2} r={r}
          strokeWidth={stroke} stroke={ringColor} fill="none"
          strokeDasharray={`${dash} ${circ}`}
          strokeLinecap="round"
          style={{ transition: "stroke-dasharray 0.6s ease-out, stroke 0.4s" }}
        />
      </svg>
      <div className="relative z-10 text-center">{children}</div>
    </div>
  );
}

/* ─── CSS bar chart for daily completion ─── */
function DayBarChart({ dayStats }) {
  const maxDone = Math.max(...dayStats.map((d) => d.done), 1);
  return (
    <div className="flex h-32 items-end gap-2">
      {dayStats.map((item, i) => {
        const isPerfect = item.done === HABITS.length;
        const isGood = item.done >= SOLID_DAY_THRESHOLD;
        const barColor = isPerfect ? "#22c55e" : isGood ? "#3b82f6" : item.done > 0 ? "#6366f1" : "rgba(255,255,255,0.06)";
        const heightPct = item.done === 0 ? 4 : Math.round((item.done / maxDone) * 100);
        return (
          <div key={item.day} className="group flex flex-1 flex-col items-center gap-1">
            <div
              className="text-[10px] font-semibold opacity-0 transition-opacity group-hover:opacity-100"
              style={{ color: barColor }}
            >
              {item.done}/{HABITS.length}
            </div>
            <div className="relative flex w-full flex-1 items-end">
              <div
                className="w-full rounded-t-md transition-all duration-700 ease-out"
                style={{ height: `${heightPct}%`, background: barColor, minHeight: 4,
                  boxShadow: item.done > 0 ? `0 0 8px ${barColor}60` : "none" }}
              />
            </div>
            <div className="text-[11px] font-semibold text-blue-300/50">{DAYS_SHORT[i]}</div>
          </div>
        );
      })}
    </div>
  );
}

/* ─── 7-cell heat row sparkline per habit ─── */
function HeatRow({ weekData, habitKey, color }) {
  return (
    <div className="flex gap-1">
      {DAYS.map((day, i) => {
        const active = weekData[day]?.[habitKey];
        return (
          <div
            key={i}
            className="h-2 flex-1 rounded-sm transition-all duration-300"
            style={{ background: active ? color : "rgba(255,255,255,0.07)",
              boxShadow: active ? `0 0 4px ${color}70` : "none" }}
            title={`${DAYS_SHORT[i]}: ${active ? "✓" : "—"}`}
          />
        );
      })}
    </div>
  );
}

/* ─── Main component ─── */
export default function HabitDashboard() {
  const [weekData, setWeekData] = useState(buildInitialState);
  const [activeDay, setActiveDay] = useState(null);
  const [justToggled, setJustToggled] = useState(null);
  const [toasts, setToasts] = useState([]);
  const toastCounter = useRef(0);

  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) setWeekData(JSON.parse(saved));
    } catch (err) { console.error(err); }
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(weekData));
    } catch (err) { console.error(err); }
  }, [weekData]);

  const addToast = useCallback((message) => {
    const id = ++toastCounter.current;
    setToasts((t) => [...t.slice(-3), { id, message }]);
    setTimeout(() => setToasts((t) => t.filter((x) => x.id !== id)), 3200);
  }, []);

  const toggleHabit = (day, key) => {
    setWeekData((prev) => {
      const next = { ...prev, [day]: { ...prev[day], [key]: !prev[day][key] } };
      if (!prev[day][key]) {
        const dayDone = HABITS.filter((h) => next[day][h.key]).length;
        if (dayDone === HABITS.length) {
          setTimeout(() => addToast("🌟 PERFECT DAY — every habit nailed!"), 80);
        } else if (dayDone === SOLID_DAY_THRESHOLD) {
          setTimeout(() => addToast("🔒 Locked in! 6 habits done today."), 80);
        } else {
          const msg = FEEDBACK[Math.floor(Math.random() * FEEDBACK.length)];
          setTimeout(() => addToast(msg), 80);
        }
      }
      return next;
    });
    setJustToggled(`${day}-${key}`);
    setTimeout(() => setJustToggled(null), 400);
  };

  const setField = (day, field, value) =>
    setWeekData((prev) => ({ ...prev, [day]: { ...prev[day], [field]: value } }));

  const resetAll = () => setWeekData(buildInitialState());

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

  const bestDay = [...dayStats].sort((a, b) => b.done - a.done)[0];
  const solidDays = dayStats.filter((d) => d.done >= SOLID_DAY_THRESHOLD).length;
  const perfectDays = dayStats.filter((d) => d.done === HABITS.length).length;

  const currentStreak = useMemo(() => {
    let streak = 0;
    for (const { done } of dayStats) {
      if (done > 0) streak++;
      else break;
    }
    return streak;
  }, [dayStats]);

  const habitBreakdown = useMemo(
    () => HABITS.map((habit) => {
      const count = DAYS.reduce((s, d) => s + (weekData[d]?.[habit.key] ? 1 : 0), 0);
      return { ...habit, count, pct: Math.round((count / DAYS.length) * 100) };
    }),
    [weekData],
  );

  const weekScore = overallPct >= 90 ? "🌟 Exceptional week!"
    : overallPct >= 70 ? "💪 Strong progress!"
    : overallPct >= 40 ? "⚡ Keep pushing!"
    : "🎯 Build the habit!";

  return (
    <div
      className="min-h-screen bg-[#06091f] p-4 text-white md:p-5"
      style={{ fontFamily: "'Inter', system-ui, sans-serif" }}
    >
      {/* Ambient glows */}
      <div className="pointer-events-none fixed inset-0 overflow-hidden">
        <div className="absolute -top-32 left-1/3 h-[500px] w-[500px] rounded-full bg-blue-700/8 blur-[160px]" />
        <div className="absolute top-1/2 right-0 h-[400px] w-[400px] rounded-full bg-violet-600/8 blur-[130px]" />
        <div className="absolute bottom-0 left-0 h-[300px] w-[300px] rounded-full bg-indigo-500/8 blur-[110px]" />
      </div>

      <Toast toasts={toasts} />

      <div className="relative mx-auto max-w-7xl space-y-3">

        {/* ── TOP BAR: title + stat chips + reset ── */}
        <div className="flex flex-wrap items-center justify-between gap-3 rounded-2xl border border-white/8 bg-gradient-to-r from-[#0c1d5c]/80 to-[#080f30]/80 px-5 py-3 backdrop-blur-sm">
          <div className="flex items-center gap-3">
            <div className="rounded-xl bg-blue-500/15 p-2">
              <Flame className="h-5 w-5 text-blue-400" />
            </div>
            <div>
              <h1 className="text-base font-bold tracking-tight leading-none">Work Habits</h1>
              <p className="mt-0.5 text-[10px] text-blue-300/40 uppercase tracking-widest">Weekly Tracker</p>
            </div>
          </div>

          <div className="flex flex-wrap items-center gap-2">
            {[
              { label: "Overall", value: `${overallPct}%`, color: "#3b82f6", icon: Target },
              { label: "Streak",  value: `${currentStreak}d`, color: "#f97316", icon: Flame },
              { label: "Perfect", value: perfectDays,     color: "#22c55e", icon: Star },
              { label: "Solid",   value: solidDays,       color: "#a855f7", icon: Zap },
            ].map(({ label, value, color, icon: Icon }) => (
              <div key={label} className="flex items-center gap-1.5 rounded-xl border border-white/8 bg-white/4 px-3 py-1.5">
                <Icon className="h-3 w-3" style={{ color }} />
                <span className="text-xs font-bold" style={{ color }}>{value}</span>
                <span className="text-[9px] uppercase tracking-widest text-blue-300/35">{label}</span>
              </div>
            ))}
            <button
              onClick={resetAll}
              className="flex items-center gap-1.5 rounded-xl border border-white/8 bg-white/4 px-3 py-1.5 text-[11px] font-medium text-blue-300/50 transition hover:border-red-400/30 hover:bg-red-500/8 hover:text-red-300"
            >
              <RotateCcw className="h-3 w-3" />
              Reset
            </button>
          </div>
        </div>

        {/* ── HABIT GRID TABLE ── */}
        <div className="overflow-hidden rounded-2xl border border-white/8 bg-[#080d28]">
          <div className="flex items-center justify-between border-b border-white/5 px-5 py-3">
            <div>
              <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Weekly Schedule</div>
              <h2 className="text-sm font-bold leading-tight">Habit Grid</h2>
            </div>
            <div className="flex items-center gap-1.5 rounded-xl bg-white/4 px-3 py-1.5 text-[10px] font-medium text-blue-300/40">
              <CheckCircle2 className="h-3 w-3 text-green-500/60" />
              Tap to check off
            </div>
          </div>

          <div className="overflow-x-auto">
            <table className="min-w-full text-sm">
              <thead>
                <tr className="border-b border-white/5">
                  <th className="px-4 py-2.5 text-left text-[10px] font-medium uppercase tracking-widest text-blue-300/35 w-36">
                    Habit
                  </th>
                  {DAYS.map((day, i) => {
                    const stat = dayStats[i];
                    const isActive = activeDay === day;
                    const isPerfect = stat.done === HABITS.length;
                    const isGood = stat.done >= SOLID_DAY_THRESHOLD;
                    return (
                      <th
                        key={day}
                        onClick={() => setActiveDay(isActive ? null : day)}
                        className={`cursor-pointer px-2 py-2.5 text-center transition-colors select-none ${isActive ? "bg-blue-500/10" : "hover:bg-white/3"}`}
                      >
                        <div className="text-xs font-bold text-blue-200/60">{DAYS_SHORT[i]}</div>
                        <div
                          className="mt-0.5 text-[10px] font-semibold"
                          style={{
                            color: isPerfect ? "#4ade80" : isGood ? "#fbbf24" : stat.done > 0 ? "#818cf8" : "rgba(147,197,253,0.25)",
                          }}
                        >
                          {stat.done}/{HABITS.length}
                        </div>
                      </th>
                    );
                  })}
                </tr>
              </thead>
              <tbody>
                {HABITS.map((habit, hIdx) => {
                  const Icon = habit.icon;
                  return (
                    <tr
                      key={habit.key}
                      className={`border-b border-white/[0.04] last:border-0 transition-colors ${hIdx % 2 === 0 ? "bg-white/[0.015]" : ""}`}
                    >
                      <td className="px-4 py-2.5">
                        <div className="flex items-center gap-2">
                          <div className="flex-shrink-0 rounded-md p-1.5" style={{ background: `${habit.color}18`, color: habit.color }}>
                            <Icon className="h-3.5 w-3.5" />
                          </div>
                          <div>
                            <div className="text-xs font-semibold text-white/85">{habit.label}</div>
                            <div className="text-[9px] text-blue-300/35">{habit.target}</div>
                          </div>
                        </div>
                      </td>
                      {DAYS.map((day) => {
                        const active = weekData[day]?.[habit.key];
                        const cellKey = `${day}-${habit.key}`;
                        const popping = justToggled === cellKey;
                        const highlighted = activeDay === day;
                        return (
                          <td
                            key={cellKey}
                            className={`px-2 py-2.5 text-center transition-colors ${highlighted ? "bg-blue-500/8" : ""}`}
                          >
                            <button
                              onClick={() => toggleHabit(day, habit.key)}
                              aria-label={`${habit.label} on ${day}: ${active ? "completed" : "not completed"}`}
                              aria-pressed={active}
                              className={`mx-auto grid h-8 w-8 place-items-center rounded-full transition-all duration-200 ${popping ? "scale-125" : "scale-100"} ${
                                active
                                  ? ""
                                  : "border border-white/10 bg-white/3 hover:border-white/20 hover:bg-white/6"
                              }`}
                              style={active ? {
                                background: habit.color,
                                boxShadow: `0 0 14px ${habit.color}55`,
                              } : {}}
                            >
                              {active
                                ? <CheckCircle2 className="h-4 w-4 text-white" />
                                : <span className="text-[10px] text-white/15">○</span>
                              }
                            </button>
                          </td>
                        );
                      })}
                    </tr>
                  );
                })}
              </tbody>
            </table>
          </div>
        </div>

        {/* ── BOTTOM ROW: charts left + breakdown right ── */}
        <div className="grid gap-3 lg:grid-cols-[1fr_300px]">

          {/* Left column: bar chart + horizontal day notes */}
          <div className="space-y-3">

            {/* Bar chart card */}
            <div className="rounded-2xl border border-white/8 bg-[#080d28] p-5">
              <div className="mb-4 flex items-start justify-between">
                <div>
                  <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Analytics</div>
                  <h3 className="text-sm font-bold leading-tight">Daily Completion</h3>
                </div>
                <div className="flex items-center gap-3 text-[10px] font-medium text-blue-300/35">
                  <span className="flex items-center gap-1">
                    <span className="inline-block h-2 w-2 rounded-sm bg-green-500" />Perfect
                  </span>
                  <span className="flex items-center gap-1">
                    <span className="inline-block h-2 w-2 rounded-sm bg-blue-500" />Solid
                  </span>
                  <span className="flex items-center gap-1">
                    <span className="inline-block h-2 w-2 rounded-sm bg-indigo-400" />Progress
                  </span>
                </div>
              </div>
              <DayBarChart dayStats={dayStats} />
            </div>

            {/* Horizontal scrolling day cards */}
            <div className="rounded-2xl border border-white/8 bg-[#080d28] p-4">
              <div className="mb-3">
                <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Daily Notes</div>
                <h3 className="text-sm font-bold leading-tight">Focus &amp; Notes</h3>
              </div>
              <div className="flex gap-2.5 overflow-x-auto pb-1">
                {dayStats.map((item, i) => {
                  const isPerfect = item.done === HABITS.length;
                  const isGood = item.done >= SOLID_DAY_THRESHOLD;
                  return (
                    <div
                      key={item.day}
                      className="w-36 flex-shrink-0 space-y-2 rounded-xl border p-3"
                      style={{
                        borderColor: isPerfect ? "rgba(34,197,94,0.25)"
                          : isGood ? "rgba(59,130,246,0.2)"
                          : "rgba(255,255,255,0.06)",
                        background: isPerfect ? "rgba(34,197,94,0.05)"
                          : isGood ? "rgba(59,130,246,0.05)"
                          : "rgba(255,255,255,0.02)",
                      }}
                    >
                      <div className="flex items-center justify-between">
                        <div>
                          <div className={`text-xs font-bold ${isPerfect ? "text-green-400" : "text-white/75"}`}>
                            {DAYS_SHORT[i]}
                          </div>
                          {isPerfect && (
                            <div className="text-[8px] font-bold uppercase tracking-widest text-green-400/80">✦ Perfect</div>
                          )}
                        </div>
                        <Ring
                          pct={item.pct}
                          size={30}
                          stroke={3}
                          color={isPerfect ? "#22c55e" : isGood ? "#3b82f6" : "#6366f1"}
                        >
                          <span className="text-[7px] font-bold text-white/60">{item.pct}%</span>
                        </Ring>
                      </div>
                      <input
                        value={weekData[item.day]?.focusBlock || ""}
                        onChange={(e) => setField(item.day, "focusBlock", e.target.value)}
                        placeholder="Focus…"
                        className="w-full rounded-lg border border-white/8 bg-white/4 px-2 py-1 text-[10px] text-white/70 outline-none placeholder:text-white/18 focus:border-blue-500/40 focus:bg-white/6"
                      />
                      <textarea
                        value={weekData[item.day]?.notes || ""}
                        onChange={(e) => setField(item.day, "notes", e.target.value)}
                        placeholder="Notes…"
                        rows={3}
                        className="w-full resize-none rounded-lg border border-white/8 bg-white/4 px-2 py-1 text-[10px] text-white/70 outline-none placeholder:text-white/18 focus:border-blue-500/40 focus:bg-white/6"
                      />
                    </div>
                  );
                })}
              </div>
            </div>
          </div>

          {/* Right column: habit breakdown with sparklines */}
          <div className="rounded-2xl border border-white/8 bg-[#080d28] p-5">
            <div className="mb-4">
              <div className="text-[9px] uppercase tracking-widest text-blue-400/50">Analysis</div>
              <h3 className="text-sm font-bold leading-tight">Habit Breakdown</h3>
            </div>

            {/* Weekly score ring */}
            <div className="mb-4 flex items-center gap-3 rounded-xl border border-white/6 bg-white/3 p-3">
              <Ring pct={overallPct} size={60} stroke={6} color="#3b82f6">
                <span className="text-sm font-bold text-white">{overallPct}%</span>
              </Ring>
              <div className="min-w-0">
                <div className="text-xs font-bold text-white/85">Weekly Score</div>
                <div className="text-[10px] text-blue-300/45">{totalDone}/{totalPossible} habits</div>
                <div className="mt-1 text-[10px] font-semibold text-blue-300">{weekScore}</div>
                <div className="mt-0.5 text-[9px] text-blue-300/35">
                  Best: {bestDay?.day?.slice(0, 3) || "—"} ({bestDay?.done || 0}/{HABITS.length})
                </div>
              </div>
            </div>

            {/* Per-habit sparklines */}
            <div className="space-y-3">
              {habitBreakdown.map((habit) => {
                const Icon = habit.icon;
                return (
                  <div key={habit.key}>
                    <div className="mb-1 flex items-center gap-2">
                      <div className="rounded-md p-1 flex-shrink-0" style={{ background: `${habit.color}18`, color: habit.color }}>
                        <Icon className="h-3 w-3" />
                      </div>
                      <span className="flex-1 text-xs font-medium text-white/75 truncate">{habit.label}</span>
                      <span className="text-[11px] font-bold flex-shrink-0" style={{ color: habit.color }}>{habit.count}/7</span>
                    </div>
                    <HeatRow weekData={weekData} habitKey={habit.key} color={habit.color} />
                  </div>
                );
              })}
            </div>

            <div className="mt-4 rounded-xl border border-white/6 bg-white/3 p-3">
              <div className="flex items-center gap-2 text-[11px] font-semibold text-white/60">
                <Flame className="h-3.5 w-3.5 text-orange-400" />
                Weekly Rule
              </div>
              <p className="mt-1.5 text-[10px] leading-4 text-blue-200/35">
                Stack clean days. Miss one — keep moving. Miss the whole day — restart tomorrow.
              </p>
            </div>
          </div>
        </div>
      </div>

      <style>{`
        @keyframes toastIn {
          from { opacity: 0; transform: translateX(24px) scale(0.96); }
          to   { opacity: 1; transform: translateX(0) scale(1); }
        }
      `}</style>
    </div>
  );
}
