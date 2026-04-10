# Habit-Model
import React, { useEffect, useMemo, useState } from "react";
import {
  BookOpen, Calculator, Code2, Dumbbell, NotebookPen, Trophy,
  Wine, Cigarette, RotateCcw, Flame, Target, CheckCircle2,
} from "lucide-react";

const DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"];

const HABITS = [
  { key: "reducedSmoking", label: "Reduced Smoking", target: "≤ 1",      icon: Cigarette,  color: "#f97316" },
  { key: "noDrinking",     label: "No Drinking",     target: "0 drinks", icon: Wine,        color: "#a855f7" },
  { key: "exercise",       label: "Exercise",        target: "≥ 30 min", icon: Dumbbell,    color: "#22c55e" },
  { key: "journaling",     label: "Journaling",      target: "Done",     icon: NotebookPen, color: "#f59e0b" },
  { key: "code",           label: "Code",            target: "Done",     icon: Code2,       color: "#3b82f6" },
  { key: "math",           label: "Math",            target: "Done",     icon: Calculator,  color: "#ec4899" },
  { key: "reading",        label: "Reading",         target: "Done",     icon: BookOpen,    color: "#14b8a6" },
  { key: "studying",       label: "Studying",        target: "Done",     icon: Trophy,      color: "#eab308" },
];

const STORAGE_KEY = "work-habit-schedule-v3";
const SOLID_DAY_THRESHOLD = 6;

function buildInitialState() {
  return DAYS.reduce((acc, day) => {
    acc[day] = HABITS.reduce((h, habit) => { h[habit.key] = false; return h; }, {});
    acc[day].notes = "";
    acc[day].focusBlock = "";
    return acc;
  }, {});
}

/* ─── Donut chart ─── */
function Donut({ value, size = 84, stroke = 10, label = "", color = "#ffffff" }) {
  const angle = Math.max(0, Math.min(100, value)) * 3.6;
  const isComplete = value === 100;
  return (
    <div
      className="relative grid place-items-center rounded-full transition-all duration-500"
      style={{
        width: size,
        height: size,
        background: `conic-gradient(${isComplete ? "#22c55e" : color} ${angle}deg, rgba(255,255,255,0.10) ${angle}deg 360deg)`,
      }}
    >
      <div
        className="grid place-items-center rounded-full bg-[#060e2b] text-white"
        style={{ width: size - stroke * 2, height: size - stroke * 2 }}
      >
        <div className="text-center leading-tight">
          <div
            className="font-bold"
            style={{
              fontSize: size > 90 ? "1.1rem" : "0.75rem",
              color: isComplete ? "#4ade80" : "white",
            }}
          >
            {value}%
          </div>
          {label && (
            <div className="uppercase tracking-wider text-blue-300/70" style={{ fontSize: "0.55rem" }}>
              {label}
            </div>
          )}
        </div>
      </div>
    </div>
  );
}

/* ─── Thin animated progress bar ─── */
function ProgressBar({ pct, color = "#3b82f6" }) {
  return (
    <div className="mt-1 h-1.5 overflow-hidden rounded-full bg-white/10">
      <div
        className="h-full rounded-full transition-all duration-700 ease-out"
        style={{ width: `${pct}%`, background: `linear-gradient(90deg, ${color}99, ${color})` }}
      />
    </div>
  );
}

/* ─── Metric card ─── */
function MetricCard({ title, value, sub, icon: Icon, accent = "#3b82f6" }) {
  return (
    <div className="group relative overflow-hidden rounded-2xl border border-white/10 bg-white/5 p-5 text-white backdrop-blur-sm transition-all hover:border-white/20 hover:bg-white/10">
      <div
        className="pointer-events-none absolute inset-0 opacity-0 transition-opacity group-hover:opacity-100"
        style={{ background: `radial-gradient(circle at 50% 0%, ${accent}20 0%, transparent 70%)` }}
      />
      <div className="relative">
        {Icon && (
          <div className="mb-3 inline-flex rounded-xl p-2" style={{ background: `${accent}22`, color: accent }}>
            <Icon className="h-4 w-4" />
          </div>
        )}
        <div className="text-[10px] uppercase tracking-[0.22em] text-blue-300/70">{title}</div>
        <div className="mt-1 text-3xl font-bold tracking-tight">{value}</div>
        <div className="mt-1 text-xs text-blue-100/50">{sub}</div>
      </div>
    </div>
  );
}

/* ─── Main component ─── */
export default function HabitDashboard() {
  const [weekData, setWeekData] = useState(buildInitialState);
  const [activeDay, setActiveDay] = useState(null);
  const [justToggled, setJustToggled] = useState(null);

  useEffect(() => {
    try {
      const saved = localStorage.getItem(STORAGE_KEY);
      if (saved) setWeekData(JSON.parse(saved));
    } catch (err) {
      console.error(err);
    }
  }, []);

  useEffect(() => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(weekData));
    } catch (err) {
      console.error(err);
    }
  }, [weekData]);

  const toggleHabit = (day, key) => {
    setWeekData((prev) => ({ ...prev, [day]: { ...prev[day], [key]: !prev[day][key] } }));
    const id = `${day}-${key}`;
    setJustToggled(id);
    setTimeout(() => setJustToggled(null), 350);
  };

  const setField = (day, field, value) => {
    setWeekData((prev) => ({ ...prev, [day]: { ...prev[day], [field]: value } }));
  };

  const resetAll = () => setWeekData(buildInitialState());

  const totalPossible = DAYS.length * HABITS.length;

  const totalDone = useMemo(
    () => DAYS.reduce((s, d) => s + HABITS.reduce((i, h) => i + (weekData[d]?.[h.key] ? 1 : 0), 0), 0),
    [weekData],
  );

  const overallPct = Math.round((totalDone / totalPossible) * 100);

  const dayStats = useMemo(
    () =>
      DAYS.map((day) => {
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
    () =>
      HABITS.map((habit) => {
        const count = DAYS.reduce((s, d) => s + (weekData[d]?.[habit.key] ? 1 : 0), 0);
        return { ...habit, count, pct: Math.round((count / DAYS.length) * 100) };
      }),
    [weekData],
  );

  return (
    <div className="min-h-screen bg-[#060e2b] p-4 text-white md:p-6" style={{ fontFamily: "'Inter', system-ui, sans-serif" }}>
      {/* Ambient glow blobs */}
      <div className="pointer-events-none fixed inset-0 overflow-hidden">
        <div className="absolute -top-40 left-1/4 h-96 w-96 rounded-full bg-blue-600/10 blur-[120px]" />
        <div className="absolute bottom-0 right-1/4 h-80 w-80 rounded-full bg-indigo-500/10 blur-[100px]" />
      </div>

      <div className="relative mx-auto max-w-7xl space-y-5">
        {/* ── Header card ── */}
        <div className="overflow-hidden rounded-3xl border border-white/10 bg-gradient-to-br from-[#0d2980] via-[#0a2060] to-[#060e2b] shadow-2xl">
          <div className="grid gap-5 p-5 lg:grid-cols-[300px_1fr] lg:p-7">

            {/* Sidebar */}
            <div className="flex flex-col gap-4 rounded-2xl border border-white/10 bg-white/5 p-5 backdrop-blur-md">
              <div>
                <div className="text-[10px] uppercase tracking-[0.3em] text-blue-400">Weekly System</div>
                <h1 className="mt-1 text-3xl font-bold tracking-tight">Work Habits</h1>
                <p className="mt-2 text-sm leading-6 text-blue-200/60">
                  Your personal discipline dashboard. Track every habit, every day.
                </p>
              </div>

              <div className="flex justify-center py-1">
                <Donut value={overallPct} label="overall" size={128} stroke={14} />
              </div>

              {/* Mini stat row */}
              <div className="grid grid-cols-3 gap-2 text-center">
                {[
                  { label: "Streak", value: currentStreak, suffix: "d", color: "#f97316" },
                  { label: "Perfect", value: perfectDays, suffix: "",  color: "#22c55e" },
                  { label: "Solid",   value: solidDays,   suffix: "",  color: "#3b82f6" },
                ].map(({ label, value, suffix, color }) => (
                  <div key={label} className="rounded-xl border border-white/10 bg-white/5 py-2">
                    <div className="text-xl font-bold" style={{ color }}>{value}{suffix}</div>
                    <div className="text-[9px] uppercase tracking-wider text-blue-300/60">{label}</div>
                  </div>
                ))}
              </div>

              {/* Habit list with per-habit progress bars */}
              <div className="rounded-2xl border border-white/8 bg-[#07103a] p-3">
                <div className="mb-2 text-[10px] font-semibold uppercase tracking-widest text-blue-400">Habits</div>
                <div className="space-y-1.5">
                  {habitBreakdown.map((habit) => {
                    const Icon = habit.icon;
                    return (
                      <div key={habit.key} className="flex items-center gap-2.5 rounded-xl bg-white/4 px-3 py-2 transition hover:bg-white/8">
                        <div className="flex-shrink-0 rounded-lg p-1.5" style={{ background: `${habit.color}22`, color: habit.color }}>
                          <Icon className="h-3.5 w-3.5" />
                        </div>
                        <div className="min-w-0 flex-1">
                          <div className="flex items-center justify-between">
                            <span className="truncate text-xs font-medium">{habit.label}</span>
                            <span className="ml-2 flex-shrink-0 text-[11px] font-bold" style={{ color: habit.color }}>{habit.count}/7</span>
                          </div>
                          <ProgressBar pct={habit.pct} color={habit.color} />
                        </div>
                      </div>
                    );
                  })}
                </div>
              </div>

              <button
                onClick={resetAll}
                className="flex items-center justify-center gap-2 rounded-2xl border border-white/10 bg-white/5 px-4 py-2.5 text-sm font-medium text-blue-200 transition hover:border-red-400/40 hover:bg-red-500/10 hover:text-red-300"
              >
                <RotateCcw className="h-3.5 w-3.5" />
                Reset Week
              </button>
            </div>

            {/* Main area */}
            <div className="space-y-5">
              {/* Metric cards */}
              <div className="grid gap-3 sm:grid-cols-3">
                <MetricCard title="Progress"   value={`${overallPct}%`}          sub={`${totalDone} / ${totalPossible} checked`}      icon={Target}   accent="#3b82f6" />
                <MetricCard title="Best Day"   value={bestDay?.day?.slice(0, 3) || "—"} sub={`${bestDay?.done || 0} of ${HABITS.length} habits`} icon={Trophy}   accent="#eab308" />
                <MetricCard title="Locked In"  value={solidDays}                  sub="Days with 6+ habits done"                        icon={Flame}    accent="#f97316" />
              </div>

              {/* Habit grid */}
              <div className="overflow-hidden rounded-2xl border border-slate-200/80 bg-white shadow-xl">
                <div className="flex flex-col gap-3 border-b border-slate-100 px-5 py-4 sm:flex-row sm:items-end sm:justify-between">
                  <div>
                    <div className="text-[10px] uppercase tracking-[0.25em] text-slate-400">Weekly View</div>
                    <h2 className="mt-0.5 text-xl font-bold text-[#0a1f5e]">7-Day Habit Schedule</h2>
                  </div>
                  <div className="inline-flex items-center gap-1.5 rounded-xl bg-slate-100 px-3 py-1.5 text-xs font-medium text-slate-500">
                    <CheckCircle2 className="h-3.5 w-3.5 text-green-500" />
                    Click circles to mark complete
                  </div>
                </div>

                <div className="overflow-x-auto">
                  <table className="min-w-full border-separate border-spacing-0 text-sm">
                    <thead>
                      <tr>
                        <th className="sticky left-0 z-10 bg-[#0a1f5e] px-5 py-3 text-left text-xs font-semibold uppercase tracking-wider text-blue-200">
                          Habit
                        </th>
                        {DAYS.map((day) => {
                          const stat = dayStats.find((d) => d.day === day);
                          const isActive = activeDay === day;
                          return (
                            <th
                              key={day}
                              onClick={() => setActiveDay(isActive ? null : day)}
                              className={`cursor-pointer bg-[#0a1f5e] px-3 py-3 text-center transition hover:bg-[#0d2980] ${isActive ? "bg-[#1a3d9e]" : ""}`}
                            >
                              <div className="text-xs font-semibold uppercase tracking-wider text-blue-200">{day.slice(0, 3)}</div>
                              {stat && (
                                <div
                                  className="mt-0.5 text-[10px] font-semibold"
                                  style={{
                                    color: stat.done === HABITS.length ? "#4ade80"
                                         : stat.done >= SOLID_DAY_THRESHOLD ? "#fbbf24"
                                         : "rgba(147,197,253,0.5)",
                                  }}
                                >
                                  {stat.done}/{HABITS.length}
                                </div>
                              )}
                            </th>
                          );
                        })}
                      </tr>
                    </thead>
                    <tbody>
                      {HABITS.map((habit, hIdx) => {
                        const Icon = habit.icon;
                        return (
                          <tr key={habit.key} className={hIdx % 2 === 0 ? "bg-slate-50/60" : "bg-white"}>
                            <td className="sticky left-0 z-10 border-b border-slate-100 bg-inherit px-5 py-3">
                              <div className="flex items-center gap-3">
                                <div className="flex-shrink-0 rounded-lg p-2" style={{ background: `${habit.color}18`, color: habit.color }}>
                                  <Icon className="h-4 w-4" />
                                </div>
                                <div>
                                  <div className="text-sm font-semibold text-slate-800">{habit.label}</div>
                                  <div className="text-[11px] text-slate-400">{habit.target}</div>
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
                                  className={`border-b border-slate-100 px-3 py-3 text-center transition ${highlighted ? "bg-blue-50" : ""}`}
                                >
                                  <button
                                    onClick={() => toggleHabit(day, habit.key)}
                                    aria-label={`${habit.label} on ${day}: ${active ? "completed" : "not completed"}`}
                                    aria-pressed={active}
                                    className={`mx-auto grid h-9 w-9 place-items-center rounded-full border-2 transition-all duration-200 ${popping ? "scale-125" : "scale-100"} ${
                                      active
                                        ? "border-transparent shadow-lg"
                                        : "border-slate-200 bg-white text-slate-300 hover:border-slate-400"
                                    }`}
                                    style={active ? { background: habit.color, boxShadow: `0 2px 14px ${habit.color}55` } : {}}
                                  >
                                    {active
                                      ? <CheckCircle2 className="h-4 w-4 text-white" />
                                      : <span className="text-xs text-slate-300">○</span>
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
            </div>
          </div>
        </div>

        {/* ── Bottom section ── */}
        <div className="grid gap-5 xl:grid-cols-[1.3fr_0.7fr]">

          {/* Daily analytics */}
          <div className="rounded-3xl border border-white/10 bg-white p-5 text-slate-900 shadow-xl">
            <div className="flex flex-col gap-3 sm:flex-row sm:items-end sm:justify-between">
              <div>
                <div className="text-[10px] uppercase tracking-[0.25em] text-slate-400">Daily Analytics</div>
                <h3 className="mt-0.5 text-xl font-bold text-[#0a1f5e]">Progress by Day</h3>
              </div>
              <div className="text-xs text-slate-400">Add notes &amp; focus blocks per day</div>
            </div>

            <div className="mt-4 grid gap-3 sm:grid-cols-2 xl:grid-cols-4">
              {dayStats.map((item) => {
                const isPerfect = item.done === HABITS.length;
                const isGood    = item.done >= SOLID_DAY_THRESHOLD;
                return (
                  <div
                    key={item.day}
                    className={`rounded-2xl border p-4 transition-all ${
                      isPerfect ? "border-green-200 bg-green-50"
                      : isGood  ? "border-blue-100 bg-blue-50/40"
                      :           "border-slate-200 bg-slate-50"
                    }`}
                  >
                    <div className="flex items-start justify-between gap-2">
                      <div>
                        <div className={`text-sm font-bold ${isPerfect ? "text-green-700" : "text-slate-700"}`}>
                          {item.day.slice(0, 3)}
                        </div>
                        <div className={`text-xs ${isPerfect ? "text-green-600" : "text-slate-400"}`}>
                          {item.done}/{HABITS.length}
                        </div>
                        {isPerfect && (
                          <div className="mt-0.5 text-[10px] font-bold uppercase tracking-wider text-green-500">Perfect ✦</div>
                        )}
                      </div>
                      <Donut
                        value={item.pct}
                        size={54}
                        stroke={7}
                        color={isPerfect ? "#22c55e" : isGood ? "#3b82f6" : "#94a3b8"}
                      />
                    </div>
                    <div className="mt-3 space-y-2">
                      <input
                        value={weekData[item.day]?.focusBlock || ""}
                        onChange={(e) => setField(item.day, "focusBlock", e.target.value)}
                        placeholder="Main focus block"
                        className="w-full rounded-xl border border-slate-200 bg-white px-3 py-1.5 text-xs outline-none transition focus:border-blue-400 focus:ring-2 focus:ring-blue-100"
                      />
                      <textarea
                        value={weekData[item.day]?.notes || ""}
                        onChange={(e) => setField(item.day, "notes", e.target.value)}
                        placeholder="Quick notes..."
                        className="min-h-[68px] w-full resize-none rounded-xl border border-slate-200 bg-white px-3 py-1.5 text-xs outline-none transition focus:border-blue-400 focus:ring-2 focus:ring-blue-100"
                      />
                    </div>
                  </div>
                );
              })}
            </div>
          </div>

          {/* Habit breakdown */}
          <div className="flex flex-col gap-4 rounded-3xl border border-white/10 bg-gradient-to-br from-[#0d2980] via-[#0a2060] to-[#060e2b] p-5 text-white shadow-2xl">
            <div>
              <div className="text-[10px] uppercase tracking-[0.25em] text-blue-400">Analysis</div>
              <h3 className="mt-0.5 text-xl font-bold">Habit Completion</h3>
            </div>

            <div className="flex-1 space-y-2">
              {habitBreakdown.map((habit) => {
                const Icon = habit.icon;
                return (
                  <div key={habit.key} className="rounded-xl bg-white/4 px-3 py-2.5 transition hover:bg-white/7">
                    <div className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <div className="rounded-lg p-1.5" style={{ background: `${habit.color}25`, color: habit.color }}>
                          <Icon className="h-3.5 w-3.5" />
                        </div>
                        <span className="text-sm font-medium">{habit.label}</span>
                      </div>
                      <span className="text-xs font-bold" style={{ color: habit.color }}>{habit.count}/7</span>
                    </div>
                    <div className="mt-2 h-1.5 overflow-hidden rounded-full bg-white/10">
                      <div
                        className="h-full rounded-full transition-all duration-700 ease-out"
                        style={{ width: `${habit.pct}%`, background: `linear-gradient(90deg, ${habit.color}99, ${habit.color})` }}
                      />
                    </div>
                  </div>
                );
              })}
            </div>

            <div className="rounded-2xl border border-white/8 bg-white/5 p-4">
              <div className="flex items-center gap-2 text-sm font-semibold">
                <Flame className="h-4 w-4 text-orange-400" />
                Weekly Rule
              </div>
              <p className="mt-2 text-xs leading-5 text-blue-200/65">
                Treat this like a scoreboard. Stack clean days. Miss one box and keep moving.
                Miss the whole day — restart the next morning.
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
