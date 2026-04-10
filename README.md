# Habit-Model
import React, { useEffect, useMemo, useState } from "react";
import { BookOpen, Calculator, Code2, Dumbbell, NotebookPen, Trophy, Wine, Cigarette, RotateCcw } from "lucide-react";

const DAYS = [
  "Monday",
  "Tuesday",
  "Wednesday",
  "Thursday",
  "Friday",
  "Saturday",
  "Sunday",
];

const HABITS = [
  { key: "reducedSmoking", label: "Reduced Smoking", target: "≤ 1", icon: Cigarette },
  { key: "noDrinking", label: "No Drinking", target: "0 drinks", icon: Wine },
  { key: "exercise", label: "Exercise", target: ">= 30 min", icon: Dumbbell },
  { key: "journaling", label: "Journaling", target: "Done", icon: NotebookPen },
  { key: "code", label: "Code", target: "Done", icon: Code2 },
  { key: "math", label: "Math", target: "Done", icon: Calculator },
  { key: "reading", label: "Reading", target: "Done", icon: BookOpen },
  { key: "studying", label: "Studying", target: "Done", icon: Trophy },
];

const STORAGE_KEY = "work-habit-schedule-inspired-v2";

function buildInitialState() {
  return DAYS.reduce((acc, day) => {
    acc[day] = HABITS.reduce((habitAcc, habit) => {
      habitAcc[habit.key] = false;
      return habitAcc;
    }, {});
    acc[day].notes = "";
    acc[day].focusBlock = "";
    return acc;
  }, {});
}

function Donut({ value, size = 84, stroke = 10, label = "" }) {
  const angle = Math.max(0, Math.min(100, value)) * 3.6;
  return (
    <div
      className="relative grid place-items-center rounded-full"
      style={{
        width: size,
        height: size,
        background: `conic-gradient(#ffffff ${angle}deg, rgba(255,255,255,0.18) ${angle}deg 360deg)`,
      }}
    >
      <div
        className="grid place-items-center rounded-full bg-[#0a2366] text-white"
        style={{ width: size - stroke * 2, height: size - stroke * 2 }}
      >
        <div className="text-center leading-tight">
          <div className="text-lg font-bold">{value}%</div>
          {label ? <div className="text-[10px] uppercase tracking-[0.16em] text-blue-200">{label}</div> : null}
        </div>
      </div>
    </div>
  );
}

function MetricCard({ title, value, sub }) {
  return (
    <div className="rounded-[24px] border border-white/15 bg-white/10 p-4 text-white backdrop-blur-sm">
      <div className="text-[11px] uppercase tracking-[0.18em] text-blue-200">{title}</div>
      <div className="mt-2 text-2xl font-bold">{value}</div>
      <div className="mt-1 text-sm text-blue-100/80">{sub}</div>
    </div>
  );
}

export default function HabitDashboardInspired() {
  const [weekData, setWeekData] = useState(buildInitialState);

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
    setWeekData((prev) => ({
      ...prev,
      [day]: {
        ...prev[day],
        [key]: !prev[day][key],
      },
    }));
  };

  const setField = (day, field, value) => {
    setWeekData((prev) => ({
      ...prev,
      [day]: {
        ...prev[day],
        [field]: value,
      },
    }));
  };

  const resetAll = () => setWeekData(buildInitialState());

  const totalPossible = DAYS.length * HABITS.length;
  const totalDone = useMemo(() => {
    return DAYS.reduce(
      (sum, day) =>
        sum + HABITS.reduce((inner, habit) => inner + (weekData[day]?.[habit.key] ? 1 : 0), 0),
      0
    );
  }, [weekData]);

  const overallPct = Math.round((totalDone / totalPossible) * 100);

  const dayStats = useMemo(() => {
    return DAYS.map((day) => {
      const done = HABITS.filter((habit) => weekData[day]?.[habit.key]).length;
      return {
        day,
        done,
        pct: Math.round((done / HABITS.length) * 100),
      };
    });
  }, [weekData]);

  const bestDay = [...dayStats].sort((a, b) => b.done - a.done)[0];
  const solidDays = dayStats.filter((d) => d.done >= 6).length;

  const habitBreakdown = useMemo(() => {
    return HABITS.map((habit) => {
      const count = DAYS.reduce((sum, day) => sum + (weekData[day]?.[habit.key] ? 1 : 0), 0);
      return {
        ...habit,
        count,
        pct: Math.round((count / DAYS.length) * 100),
      };
    });
  }, [weekData]);

  return (
    <div className="min-h-screen bg-[#07133d] p-4 text-white md:p-6">
      <div className="mx-auto max-w-7xl space-y-5">
        <div className="overflow-hidden rounded-[32px] border border-white/10 bg-gradient-to-br from-[#0b2673] via-[#0a2366] to-[#06143f] shadow-2xl">
          <div className="grid gap-5 p-5 lg:grid-cols-[300px_1fr] lg:p-6">
            <div className="rounded-[28px] border border-white/10 bg-white/10 p-5 backdrop-blur-sm">
              <div className="text-xs uppercase tracking-[0.24em] text-blue-200">Weekly System</div>
              <h1 className="mt-2 text-3xl font-bold tracking-tight">Work Habits</h1>
              <p className="mt-2 text-sm leading-6 text-blue-100/85">
                Inspired by the video style. Built like a clean discipline dashboard so you can track the week fast.
              </p>

              <div className="mt-6 flex justify-center">
                <Donut value={overallPct} label="overall" size={126} stroke={14} />
              </div>

              <div className="mt-6 rounded-[24px] bg-[#091d59] p-4 ring-1 ring-white/10">
                <div className="text-sm font-semibold">My Habits</div>
                <div className="mt-3 space-y-3">
                  {HABITS.map((habit) => {
                    const Icon = habit.icon;
                    return (
                      <div key={habit.key} className="flex items-center justify-between rounded-2xl bg-white/5 px-3 py-2">
                        <div className="flex items-center gap-3">
                          <div className="rounded-xl bg-white/10 p-2">
                            <Icon className="h-4 w-4" />
                          </div>
                          <div>
                            <div className="text-sm font-medium">{habit.label}</div>
                            <div className="text-xs text-blue-200/80">{habit.target}</div>
                          </div>
                        </div>
                        <div className="text-xs font-semibold text-blue-100">{habitBreakdown.find((h) => h.key === habit.key)?.count}/7</div>
                      </div>
                    );
                  })}
                </div>
              </div>

              <button
                onClick={resetAll}
                className="mt-5 flex w-full items-center justify-center gap-2 rounded-[20px] border border-white/15 bg-white/10 px-4 py-3 text-sm font-semibold transition hover:bg-white/15"
              >
                <RotateCcw className="h-4 w-4" />
                Reset Week
              </button>
            </div>

            <div className="space-y-5">
              <div className="grid gap-4 md:grid-cols-3">
                <MetricCard title="Progress" value={`${overallPct}%`} sub={`${totalDone} of ${totalPossible} boxes checked`} />
                <MetricCard title="Best Day" value={bestDay?.day || "—"} sub={`${bestDay?.done || 0} of ${HABITS.length} completed`} />
                <MetricCard title="Locked In Days" value={solidDays} sub={`Days with 6+ habits done`} />
              </div>

              <div className="rounded-[28px] border border-white/10 bg-white p-4 text-slate-900 shadow-xl md:p-5">
                <div className="flex flex-col gap-3 md:flex-row md:items-end md:justify-between">
                  <div>
                    <div className="text-xs uppercase tracking-[0.22em] text-slate-500">Weekly View</div>
                    <h2 className="mt-1 text-2xl font-bold text-[#0a2366]">7 Day Habit Schedule</h2>
                  </div>
                  <div className="rounded-2xl bg-slate-100 px-4 py-2 text-sm font-medium text-slate-600">Click each circle when complete</div>
                </div>

                <div className="mt-5 overflow-x-auto">
                  <table className="min-w-full border-separate border-spacing-y-2 text-sm">
                    <thead>
                      <tr>
                        <th className="rounded-l-2xl bg-[#0a2366] px-4 py-3 text-left font-semibold text-white">Habit</th>
                        {DAYS.map((day) => (
                          <th key={day} className="bg-[#0a2366] px-3 py-3 text-center font-semibold text-white last:rounded-r-2xl">
                            <div className="text-xs uppercase tracking-[0.12em] text-blue-200">{day.slice(0, 3)}</div>
                          </th>
                        ))}
                      </tr>
                    </thead>
                    <tbody>
                      {HABITS.map((habit) => {
                        const Icon = habit.icon;
                        return (
                          <tr key={habit.key}>
                            <td className="rounded-l-2xl border border-slate-200 bg-slate-50 px-4 py-3">
                              <div className="flex items-center gap-3">
                                <div className="rounded-xl bg-[#0a2366] p-2 text-white">
                                  <Icon className="h-4 w-4" />
                                </div>
                                <div>
                                  <div className="font-semibold text-slate-900">{habit.label}</div>
                                  <div className="text-xs text-slate-500">{habit.target}</div>
                                </div>
                              </div>
                            </td>
                            {DAYS.map((day, idx) => {
                              const active = weekData[day]?.[habit.key];
                              return (
                                <td
                                  key={`${day}-${habit.key}`}
                                  className={`border border-slate-200 bg-white px-3 py-3 text-center ${idx === DAYS.length - 1 ? "rounded-r-2xl" : ""}`}
                                >
                                  <button
                                    onClick={() => toggleHabit(day, habit.key)}
                                    className={`mx-auto grid h-10 w-10 place-items-center rounded-full border-2 transition ${
                                      active
                                        ? "border-[#0a2366] bg-[#0a2366] text-white shadow-md"
                                        : "border-slate-300 bg-white text-slate-300 hover:border-[#0a2366] hover:text-[#0a2366]"
                                    }`}
                                  >
                                    {active ? "✓" : ""}
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

        <div className="grid gap-5 xl:grid-cols-[1.2fr_0.8fr]">
          <div className="rounded-[28px] border border-white/10 bg-white p-5 text-slate-900 shadow-xl">
            <div className="flex items-end justify-between gap-4">
              <div>
                <div className="text-xs uppercase tracking-[0.2em] text-slate-500">Daily Analytics</div>
                <h3 className="mt-1 text-2xl font-bold text-[#0a2366]">Progress by Day</h3>
              </div>
              <div className="text-sm text-slate-500">Spreadsheet-style view</div>
            </div>

            <div className="mt-5 grid gap-4 md:grid-cols-2 xl:grid-cols-4">
              {dayStats.map((item) => (
                <div key={item.day} className="rounded-[24px] border border-slate-200 bg-slate-50 p-4">
                  <div className="flex items-start justify-between gap-3">
                    <div>
                      <div className="text-sm font-semibold text-slate-800">{item.day}</div>
                      <div className="text-xs text-slate-500">{item.done} of {HABITS.length}</div>
                    </div>
                    <Donut value={item.pct} label="done" size={76} stroke={9} />
                  </div>

                  <div className="mt-4 space-y-3">
                    <input
                      value={weekData[item.day]?.focusBlock || ""}
                      onChange={(e) => setField(item.day, "focusBlock", e.target.value)}
                      placeholder="Main focus block"
                      className="w-full rounded-2xl border border-slate-200 px-3 py-2 text-sm outline-none focus:border-[#0a2366]"
                    />
                    <textarea
                      value={weekData[item.day]?.notes || ""}
                      onChange={(e) => setField(item.day, "notes", e.target.value)}
                      placeholder="Quick notes"
                      className="min-h-[92px] w-full rounded-2xl border border-slate-200 px-3 py-2 text-sm outline-none focus:border-[#0a2366]"
                    />
                  </div>
                </div>
              ))}
            </div>
          </div>

          <div className="rounded-[28px] border border-white/10 bg-gradient-to-br from-[#0b2673] via-[#0a2366] to-[#06143f] p-5 text-white shadow-2xl">
            <div>
              <div className="text-xs uppercase tracking-[0.2em] text-blue-200">Analysis</div>
              <h3 className="mt-1 text-2xl font-bold">Habit Completion</h3>
            </div>

            <div className="mt-5 space-y-4">
              {habitBreakdown.map((habit) => (
                <div key={habit.key}>
                  <div className="mb-2 flex items-center justify-between text-sm">
                    <span className="font-medium">{habit.label}</span>
                    <span className="text-blue-100">{habit.count}/7</span>
                  </div>
                  <div className="h-3 overflow-hidden rounded-full bg-white/15">
                    <div
                      className="h-full rounded-full bg-white transition-all"
                      style={{ width: `${habit.pct}%` }}
                    />
                  </div>
                </div>
              ))}
            </div>

            <div className="mt-6 rounded-[24px] border border-white/10 bg-white/10 p-4">
              <div className="text-sm font-semibold">Weekly Rule</div>
              <p className="mt-2 text-sm leading-6 text-blue-100/90">
                Treat this like a scoreboard. Stack clean days. Miss one box and keep moving. Miss the whole day and restart the next morning.
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
