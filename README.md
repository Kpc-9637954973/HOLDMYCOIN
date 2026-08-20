import React, { useState, useEffect, useMemo } from "react";
import {
  LayoutDashboard, ArrowLeftRight, Eye, PieChart as PieChartIcon, PiggyBank,
  Plus, Trash2, TrendingUp, TrendingDown, Wallet, ArrowUpRight, ArrowDownRight,
  X, Coins, Target, Pencil,
} from "lucide-react";
import {
  ResponsiveContainer, PieChart, Pie, Cell, Tooltip, BarChart, Bar, XAxis,
  YAxis, CartesianGrid, Legend,
} from "recharts";

/* ---------------------------------------------------------------------- */
/*  Storage helpers — this is the "backend": persistent key/value store   */
/* ---------------------------------------------------------------------- */
async function loadKey(key, fallback) {
  try {
    if (window.storage?.get) {
      const res = await window.storage.get(key, false);
      return res && res.value ? JSON.parse(res.value) : fallback;
    }
    const raw = window.localStorage.getItem(key);
    return raw ? JSON.parse(raw) : fallback;
  } catch (e) {
    console.warn("Storage read failed for", key, e);
    return fallback;
  }
}
async function saveKey(key, value) {
  try {
    const serialized = JSON.stringify(value);
    if (window.storage?.set) {
      await window.storage.set(key, serialized, false);
    } else {
      window.localStorage.setItem(key, serialized);
    }
  } catch (e) {
    console.error("Storage error on", key, e);
  }
}
const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
const inr = (n) =>
  "₹" + Number(n || 0).toLocaleString("en-IN", { maximumFractionDigits: 0 });
const inrDec = (n) =>
  "₹" + Number(n || 0).toLocaleString("en-IN", { minimumFractionDigits: 2, maximumFractionDigits: 2 });

const CATEGORIES = ["Salary", "Freelance", "Rent", "Groceries", "Transport", "Utilities", "Dining", "Entertainment", "Health", "Shopping", "Other"];
const PIE_COLORS = ["#4F46E5", "#6366F1", "#818CF8", "#A5B4FC", "#38BDF8", "#94A3B8", "#F59E0B"];

const SEED_TXNS = [
  { id: uid(), type: "income", category: "Salary", amount: 85000, note: "Monthly salary", date: "2026-08-01" },
  { id: uid(), type: "expense", category: "Rent", amount: 22000, note: "August rent", date: "2026-08-02" },
  { id: uid(), type: "expense", category: "Groceries", amount: 6400, note: "BigBasket order", date: "2026-08-05" },
  { id: uid(), type: "expense", category: "Transport", amount: 2100, note: "Ola rides", date: "2026-08-07" },
  { id: uid(), type: "expense", category: "Utilities", amount: 3200, note: "Electricity + wifi", date: "2026-08-09" },
  { id: uid(), type: "freelance" === "x" ? "income" : "income", category: "Freelance", amount: 15000, note: "Design project", date: "2026-08-11" },
  { id: uid(), type: "expense", category: "Dining", amount: 2850, note: "Weekend dinner", date: "2026-08-13" },
  { id: uid(), type: "expense", category: "Shopping", amount: 4700, note: "Flipkart order", date: "2026-08-15" },
  { id: uid(), type: "expense", category: "Entertainment", amount: 999, note: "Streaming subs", date: "2026-08-16" },
  { id: uid(), type: "income", category: "Salary", amount: 84000, note: "Monthly salary", date: "2026-07-01" },
  { id: uid(), type: "expense", category: "Rent", amount: 22000, note: "July rent", date: "2026-07-02" },
  { id: uid(), type: "expense", category: "Groceries", amount: 5900, note: "Groceries", date: "2026-07-06" },
  { id: uid(), type: "expense", category: "Health", amount: 1800, note: "Pharmacy", date: "2026-07-10" },
  { id: uid(), type: "income", category: "Salary", amount: 84000, note: "Monthly salary", date: "2026-06-01" },
  { id: uid(), type: "expense", category: "Rent", amount: 22000, note: "June rent", date: "2026-06-02" },
  { id: uid(), type: "expense", category: "Transport", amount: 1900, note: "Fuel + cabs", date: "2026-06-08" },
];
const SEED_WATCHLIST = [
  { id: uid(), symbol: "TCS", name: "Tata Consultancy Services", price: 4128.5, target: 4400 },
  { id: uid(), symbol: "HDFCBANK", name: "HDFC Bank", price: 1687.2, target: 1800 },
  { id: uid(), symbol: "INFY", name: "Infosys", price: 1912.4, target: 2000 },
];
const SEED_PORTFOLIO = [
  { id: uid(), symbol: "NIFTYBEES", name: "Nifty 50 ETF", qty: 120, avgPrice: 198.4, currentPrice: 231.7 },
  { id: uid(), symbol: "GOLDBEES", name: "Gold ETF", qty: 60, avgPrice: 52.1, currentPrice: 61.8 },
  { id: uid(), symbol: "RELIANCE", name: "Reliance Industries", qty: 15, avgPrice: 2410, currentPrice: 2588 },
  { id: uid(), symbol: "PPF", name: "Public Provident Fund", qty: 1, avgPrice: 180000, currentPrice: 194500 },
];
const SEED_GOALS = [
  { id: uid(), name: "Emergency fund", target: 300000, current: 184000, deadline: "2026-12-31" },
  { id: uid(), name: "Goa trip", target: 60000, current: 21500, deadline: "2026-11-15" },
  { id: uid(), name: "New laptop", target: 120000, current: 92000, deadline: "2026-10-01" },
];

/* ---------------------------------------------------------------------- */
/*  Small shared UI bits                                                  */
/* ---------------------------------------------------------------------- */
function Card({ children, className = "" }) {
  return (
    <div className={`bg-white border border-slate-200 rounded-2xl ${className}`}>
      {children}
    </div>
  );
}
function StatCard({ icon, label, value, delta, deltaPositive, tint }) {
  return (
    <Card className="p-5">
      <div className="flex items-start justify-between">
        <div
          className="w-10 h-10 rounded-xl flex items-center justify-center"
          style={{ background: tint }}
        >
          {icon}
        </div>
        {delta !== undefined && (
          <span
            className={`text-xs font-semibold flex items-center gap-1 ${
              deltaPositive ? "text-emerald-600" : "text-rose-600"
            }`}
          >
            {deltaPositive ? <ArrowUpRight className="w-3.5 h-3.5" /> : <ArrowDownRight className="w-3.5 h-3.5" />}
            {delta}
          </span>
        )}
      </div>
      <div className="text-xs font-medium text-slate-500 mt-4">{label}</div>
      <div
        className="text-2xl font-semibold text-slate-900 mt-1"
        style={{ fontFamily: "'Space Grotesk', sans-serif" }}
      >
        {value}
      </div>
    </Card>
  );
}
function Modal({ title, onClose, children }) {
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/40 backdrop-blur-sm">
      <div className="bg-white rounded-2xl border border-slate-200 w-full max-w-md shadow-2xl">
        <div className="flex items-center justify-between px-5 py-4 border-b border-slate-100">
          <h3 className="font-semibold text-slate-900" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
            {title}
          </h3>
          <button onClick={onClose} className="w-8 h-8 rounded-lg hover:bg-slate-100 flex items-center justify-center">
            <X className="w-4 h-4 text-slate-500" />
          </button>
        </div>
        <div className="p-5">{children}</div>
      </div>
    </div>
  );
}
function Field({ label, children }) {
  return (
    <label className="block mb-3">
      <span className="block text-xs font-medium text-slate-500 mb-1.5">{label}</span>
      {children}
    </label>
  );
}
const inputCls =
  "w-full rounded-lg border border-slate-200 px-3 py-2 text-sm text-slate-900 focus:outline-none focus:ring-2 focus:ring-indigo-500/40 focus:border-indigo-500";

/* ---------------------------------------------------------------------- */
/*  Main App                                                              */
/* ---------------------------------------------------------------------- */
export default function App() {
  const [tab, setTab] = useState("dashboard");
  const [loading, setLoading] = useState(true);
  const [txns, setTxns] = useState([]);
  const [watchlist, setWatchlist] = useState([]);
  const [portfolio, setPortfolio] = useState([]);
  const [goals, setGoals] = useState([]);
  const [txnFilter, setTxnFilter] = useState("all");
  const [showTxnForm, setShowTxnForm] = useState(false);
  const [showWatchForm, setShowWatchForm] = useState(false);
  const [showHoldingForm, setShowHoldingForm] = useState(false);
  const [showGoalForm, setShowGoalForm] = useState(false);
  const [contribGoal, setContribGoal] = useState(null);
  const [toast, setToast] = useState(null);

  useEffect(() => {
    (async () => {
      const [t, w, p, g] = await Promise.all([
        loadKey("hmc:transactions", null),
        loadKey("hmc:watchlist", null),
        loadKey("hmc:portfolio", null),
        loadKey("hmc:goals", null),
      ]);
      const finalT = t ?? SEED_TXNS;
      const finalW = w ?? SEED_WATCHLIST;
      const finalP = p ?? SEED_PORTFOLIO;
      const finalG = g ?? SEED_GOALS;
      setTxns(finalT);
      setWatchlist(finalW);
      setPortfolio(finalP);
      setGoals(finalG);
      if (t === null) saveKey("hmc:transactions", finalT);
      if (w === null) saveKey("hmc:watchlist", finalW);
      if (p === null) saveKey("hmc:portfolio", finalP);
      if (g === null) saveKey("hmc:goals", finalG);
      setLoading(false);
    })();
  }, []);

  function flash(msg) {
    setToast(msg);
    setTimeout(() => setToast(null), 2200);
  }

  /* ---- Transactions ---- */
  function addTransaction(txn) {
    const next = [{ ...txn, id: uid() }, ...txns];
    setTxns(next);
    saveKey("hmc:transactions", next);
    setShowTxnForm(false);
    flash("Transaction added");
  }
  function deleteTransaction(id) {
    const next = txns.filter((t) => t.id !== id);
    setTxns(next);
    saveKey("hmc:transactions", next);
  }

  /* ---- Watchlist ---- */
  function addWatchItem(item) {
    const next = [{ ...item, id: uid() }, ...watchlist];
    setWatchlist(next);
    saveKey("hmc:watchlist", next);
    setShowWatchForm(false);
    flash("Added to watchlist");
  }
  function removeWatchItem(id) {
    const next = watchlist.filter((w) => w.id !== id);
    setWatchlist(next);
    saveKey("hmc:watchlist", next);
  }

  /* ---- Portfolio ---- */
  function addHolding(h) {
    const next = [{ ...h, id: uid() }, ...portfolio];
    setPortfolio(next);
    saveKey("hmc:portfolio", next);
    setShowHoldingForm(false);
    flash("Holding added");
  }
  function removeHolding(id) {
    const next = portfolio.filter((h) => h.id !== id);
    setPortfolio(next);
    saveKey("hmc:portfolio", next);
  }

  /* ---- Goals ---- */
  function addGoal(g) {
    const next = [{ ...g, id: uid(), current: Number(g.current) || 0 }, ...goals];
    setGoals(next);
    saveKey("hmc:goals", next);
    setShowGoalForm(false);
    flash("Goal created");
  }
  function removeGoal(id) {
    const next = goals.filter((g) => g.id !== id);
    setGoals(next);
    saveKey("hmc:goals", next);
  }
  function contribute(id, amount) {
    const next = goals.map((g) => (g.id === id ? { ...g, current: Math.max(0, Number(g.current) + Number(amount)) } : g));
    setGoals(next);
    saveKey("hmc:goals", next);
    setContribGoal(null);
    flash("Goal updated");
  }

  /* ---- Derived data ---- */
  const totalIncome = useMemo(() => txns.filter((t) => t.type === "income").reduce((s, t) => s + Number(t.amount), 0), [txns]);
  const totalExpense = useMemo(() => txns.filter((t) => t.type === "expense").reduce((s, t) => s + Number(t.amount), 0), [txns]);
  const balance = totalIncome - totalExpense;
  const savingsRate = totalIncome > 0 ? Math.round(((totalIncome - totalExpense) / totalIncome) * 100) : 0;

  const spendByCategory = useMemo(() => {
    const map = {};
    txns.filter((t) => t.type === "expense").forEach((t) => {
      map[t.category] = (map[t.category] || 0) + Number(t.amount);
    });
    return Object.entries(map).map(([name, value]) => ({ name, value })).sort((a, b) => b.value - a.value);
  }, [txns]);

  const monthlyTrend = useMemo(() => {
    const map = {};
    txns.forEach((t) => {
      const key = t.date ? t.date.slice(0, 7) : "unknown";
      if (!map[key]) map[key] = { month: key, income: 0, expense: 0 };
      if (t.type === "income") map[key].income += Number(t.amount);
      else map[key].expense += Number(t.amount);
    });
    return Object.values(map)
      .sort((a, b) => a.month.localeCompare(b.month))
      .slice(-6)
      .map((r) => ({
        ...r,
        label: new Date(r.month + "-01").toLocaleDateString("en-IN", { month: "short" }),
      }));
  }, [txns]);

  const portfolioValue = useMemo(() => portfolio.reduce((s, h) => s + h.qty * h.currentPrice, 0), [portfolio]);
  const portfolioCost = useMemo(() => portfolio.reduce((s, h) => s + h.qty * h.avgPrice, 0), [portfolio]);
  const portfolioGain = portfolioValue - portfolioCost;
  const portfolioGainPct = portfolioCost > 0 ? (portfolioGain / portfolioCost) * 100 : 0;

  const filteredTxns = txns.filter((t) => txnFilter === "all" || t.type === txnFilter);

  const navItems = [
    { id: "dashboard", label: "Dashboard", icon: LayoutDashboard },
    { id: "transactions", label: "Transactions", icon: ArrowLeftRight },
    { id: "watchlist", label: "Watchlist", icon: Eye },
    { id: "portfolio", label: "Portfolio", icon: PieChartIcon },
    { id: "savings", label: "Savings", icon: PiggyBank },
  ];
  const pageTitles = {
    dashboard: "Dashboard",
    transactions: "Transactions",
    watchlist: "Watchlist",
    portfolio: "Portfolio",
    savings: "Savings goals",
  };

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-slate-50">
        <div className="flex flex-col items-center gap-3">
          <div className="w-8 h-8 rounded-full border-2 border-indigo-200 border-t-indigo-600 animate-spin" />
          <span className="text-sm text-slate-500">Loading your dashboard…</span>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 flex" style={{ fontFamily: "'Inter', sans-serif" }}>
      <style>{`@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600;700&family=IBM+Plex+Mono:wght@500;600&display=swap');`}</style>

      {/* Sidebar */}
      <aside className="w-20 md:w-64 shrink-0 bg-indigo-950 flex flex-col py-6 sticky top-0 h-screen">
        <div className="flex items-center gap-2.5 px-4 md:px-6 mb-8">
          <div className="w-9 h-9 rounded-xl bg-gradient-to-br from-amber-300 to-amber-500 flex items-center justify-center shrink-0">
            <Coins className="w-4.5 h-4.5 text-indigo-950" />
          </div>
          <span
            className="hidden md:block text-white font-semibold text-[15px]"
            style={{ fontFamily: "'Space Grotesk', sans-serif" }}
          >
            HoldMyCoin
          </span>
        </div>

        <nav className="flex-1 flex flex-col gap-1 px-3">
          {navItems.map((item) => {
            const Icon = item.icon;
            const active = tab === item.id;
            return (
              <button
                key={item.id}
                onClick={() => setTab(item.id)}
                className={`flex items-center gap-3 px-3 py-2.5 rounded-xl text-sm font-medium transition-colors justify-center md:justify-start ${
                  active ? "bg-indigo-600 text-white" : "text-indigo-200/70 hover:bg-white/5 hover:text-white"
                }`}
              >
                <Icon className="w-[18px] h-[18px] shrink-0" />
                <span className="hidden md:block">{item.label}</span>
              </button>
            );
          })}
        </nav>

        <div className="px-3 mt-4">
          <div className="hidden md:flex items-center gap-3 px-3 py-3 rounded-xl bg-white/5">
            <div className="w-9 h-9 rounded-lg bg-gradient-to-br from-indigo-400 to-indigo-600 flex items-center justify-center text-white text-xs font-semibold shrink-0">
              RA
            </div>
            <div className="min-w-0">
              <div className="text-white text-sm font-medium truncate">Riya Anand</div>
              <div className="text-indigo-300 text-xs truncate">Personal account</div>
            </div>
          </div>
        </div>
      </aside>

      {/* Main content */}
      <main className="flex-1 min-w-0">
        <header className="sticky top-0 z-10 bg-slate-50/90 backdrop-blur border-b border-slate-200 px-6 md:px-10 py-5 flex items-center justify-between">
          <div>
            <h1
              className="text-xl font-semibold text-slate-900"
              style={{ fontFamily: "'Space Grotesk', sans-serif" }}
            >
              {pageTitles[tab]}
            </h1>
            <p className="text-xs text-slate-500 mt-0.5">
              {new Date().toLocaleDateString("en-IN", { weekday: "long", day: "numeric", month: "long", year: "numeric" })}
            </p>
          </div>
          <div className="flex items-center gap-2 bg-white border border-slate-200 rounded-full px-4 py-2">
            <Wallet className="w-4 h-4 text-indigo-600" />
            <span className="text-sm font-semibold text-slate-900" style={{ fontFamily: "'IBM Plex Mono', monospace" }}>
              {inr(balance)}
            </span>
          </div>
        </header>

        <div className="px-6 md:px-10 py-7">
          {tab === "dashboard" && (
            <DashboardView
              totalIncome={totalIncome}
              totalExpense={totalExpense}
              balance={balance}
              savingsRate={savingsRate}
              spendByCategory={spendByCategory}
              monthlyTrend={monthlyTrend}
              txns={txns}
              portfolioValue={portfolioValue}
              portfolioGainPct={portfolioGainPct}
              onSeeAllTxns={() => setTab("transactions")}
            />
          )}

          {tab === "transactions" && (
            <TransactionsView
              txns={filteredTxns}
              filter={txnFilter}
              setFilter={setTxnFilter}
              onAdd={() => setShowTxnForm(true)}
              onDelete={deleteTransaction}
            />
          )}

          {tab === "watchlist" && (
            <WatchlistView watchlist={watchlist} onAdd={() => setShowWatchForm(true)} onRemove={removeWatchItem} />
          )}

          {tab === "portfolio" && (
            <PortfolioView
              portfolio={portfolio}
              value={portfolioValue}
              cost={portfolioCost}
              gain={portfolioGain}
              gainPct={portfolioGainPct}
              onAdd={() => setShowHoldingForm(true)}
              onRemove={removeHolding}
            />
          )}

          {tab === "savings" && (
            <SavingsView
              goals={goals}
              onAdd={() => setShowGoalForm(true)}
              onRemove={removeGoal}
              onContribute={(g) => setContribGoal(g)}
            />
          )}
        </div>
      </main>

      {/* Modals */}
      {showTxnForm && (
        <TxnFormModal onClose={() => setShowTxnForm(false)} onSubmit={addTransaction} />
      )}
      {showWatchForm && (
        <WatchFormModal onClose={() => setShowWatchForm(false)} onSubmit={addWatchItem} />
      )}
      {showHoldingForm && (
        <HoldingFormModal onClose={() => setShowHoldingForm(false)} onSubmit={addHolding} />
      )}
      {showGoalForm && (
        <GoalFormModal onClose={() => setShowGoalForm(false)} onSubmit={addGoal} />
      )}
      {contribGoal && (
        <ContributeModal goal={contribGoal} onClose={() => setContribGoal(null)} onSubmit={contribute} />
      )}

      {/* Toast */}
      {toast && (
        <div className="fixed bottom-6 left-1/2 -translate-x-1/2 bg-slate-900 text-white text-sm font-medium px-4 py-2.5 rounded-full shadow-xl z-50">
          {toast}
        </div>
      )}
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Dashboard                                                             */
/* ---------------------------------------------------------------------- */
function DashboardView({ totalIncome, totalExpense, balance, savingsRate, spendByCategory, monthlyTrend, txns, portfolioValue, portfolioGainPct, onSeeAllTxns }) {
  return (
    <div className="space-y-7">
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <StatCard
          icon={<Wallet className="w-5 h-5 text-indigo-600" />}
          tint="#EEF2FF"
          label="Total balance"
          value={inr(balance)}
        />
        <StatCard
          icon={<TrendingUp className="w-5 h-5 text-emerald-600" />}
          tint="#ECFDF5"
          label="Total income"
          value={inr(totalIncome)}
        />
        <StatCard
          icon={<TrendingDown className="w-5 h-5 text-rose-600" />}
          tint="#FEF2F2"
          label="Total expenses"
          value={inr(totalExpense)}
        />
        <StatCard
          icon={<PiggyBank className="w-5 h-5 text-amber-600" />}
          tint="#FFFBEB"
          label="Savings rate"
          value={`${savingsRate}%`}
        />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-5 gap-5">
        <Card className="p-6 lg:col-span-2">
          <h3 className="font-semibold text-slate-900 mb-4" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
            Spending by category
          </h3>
          {spendByCategory.length === 0 ? (
            <p className="text-sm text-slate-400 py-10 text-center">No expenses logged yet</p>
          ) : (
            <>
              <div className="h-52">
                <ResponsiveContainer width="100%" height="100%">
                  <PieChart>
                    <Pie data={spendByCategory} dataKey="value" nameKey="name" innerRadius={55} outerRadius={80} paddingAngle={2}>
                      {spendByCategory.map((_, i) => (
                        <Cell key={i} fill={PIE_COLORS[i % PIE_COLORS.length]} />
                      ))}
                    </Pie>
                    <Tooltip formatter={(v) => inr(v)} />
                  </PieChart>
                </ResponsiveContainer>
              </div>
              <div className="grid grid-cols-2 gap-x-4 gap-y-2 mt-3">
                {spendByCategory.slice(0, 6).map((c, i) => (
                  <div key={c.name} className="flex items-center gap-2 text-xs">
                    <span className="w-2 h-2 rounded-full shrink-0" style={{ background: PIE_COLORS[i % PIE_COLORS.length] }} />
                    <span className="text-slate-600 truncate">{c.name}</span>
                  </div>
                ))}
              </div>
            </>
          )}
        </Card>

        <Card className="p-6 lg:col-span-3">
          <h3 className="font-semibold text-slate-900 mb-4" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
            Income vs expenses
          </h3>
          <div className="h-64">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={monthlyTrend}>
                <CartesianGrid vertical={false} stroke="#F1F5F9" />
                <XAxis dataKey="label" tick={{ fontSize: 12, fill: "#64748B" }} axisLine={false} tickLine={false} />
                <YAxis tick={{ fontSize: 11, fill: "#94A3B8" }} axisLine={false} tickLine={false} tickFormatter={(v) => `${Math.round(v / 1000)}k`} />
                <Tooltip formatter={(v) => inr(v)} cursor={{ fill: "#F8FAFC" }} />
                <Legend wrapperStyle={{ fontSize: 12 }} />
                <Bar dataKey="income" name="Income" fill="#4F46E5" radius={[6, 6, 0, 0]} />
                <Bar dataKey="expense" name="Expense" fill="#CBD5E1" radius={[6, 6, 0, 0]} />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </Card>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-5 gap-5">
        <Card className="p-6 lg:col-span-3">
          <div className="flex items-center justify-between mb-4">
            <h3 className="font-semibold text-slate-900" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
              Recent transactions
            </h3>
            <button onClick={onSeeAllTxns} className="text-xs font-semibold text-indigo-600 hover:text-indigo-700">
              View all
            </button>
          </div>
          <div className="divide-y divide-slate-100">
            {txns.slice(0, 5).map((t) => (
              <div key={t.id} className="flex items-center justify-between py-2.5">
                <div className="min-w-0">
                  <div className="text-sm font-medium text-slate-900 truncate">{t.note || t.category}</div>
                  <div className="text-xs text-slate-400">{t.category} · {t.date}</div>
                </div>
                <span
                  className={`text-sm font-semibold shrink-0 ml-3 ${t.type === "income" ? "text-emerald-600" : "text-rose-600"}`}
                  style={{ fontFamily: "'IBM Plex Mono', monospace" }}
                >
                  {t.type === "income" ? "+" : "−"} {inr(t.amount)}
                </span>
              </div>
            ))}
            {txns.length === 0 && <p className="text-sm text-slate-400 py-8 text-center">No transactions yet</p>}
          </div>
        </Card>

        <Card className="p-6 lg:col-span-2">
          <h3 className="font-semibold text-slate-900 mb-1" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
            Portfolio snapshot
          </h3>
          <p className="text-xs text-slate-400 mb-4">Across all holdings</p>
          <div className="text-3xl font-semibold text-slate-900" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
            {inr(portfolioValue)}
          </div>
          <div className={`text-sm font-medium mt-1 flex items-center gap-1 ${portfolioGainPct >= 0 ? "text-emerald-600" : "text-rose-600"}`}>
            {portfolioGainPct >= 0 ? <ArrowUpRight className="w-4 h-4" /> : <ArrowDownRight className="w-4 h-4" />}
            {Math.abs(portfolioGainPct).toFixed(1)}% overall
          </div>
        </Card>
      </div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Transactions                                                          */
/* ---------------------------------------------------------------------- */
function TransactionsView({ txns, filter, setFilter, onAdd, onDelete }) {
  return (
    <div>
      <div className="flex items-center justify-between mb-5 gap-3 flex-wrap">
        <div className="flex gap-2">
          {["all", "income", "expense"].map((f) => (
            <button
              key={f}
              onClick={() => setFilter(f)}
              className={`px-3.5 py-1.5 rounded-full text-xs font-semibold capitalize border ${
                filter === f ? "bg-indigo-600 text-white border-indigo-600" : "bg-white text-slate-600 border-slate-200 hover:border-slate-300"
              }`}
            >
              {f}
            </button>
          ))}
        </div>
        <button
          onClick={onAdd}
          className="flex items-center gap-1.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold px-4 py-2 rounded-xl"
        >
          <Plus className="w-4 h-4" /> Add transaction
        </button>
      </div>

      <Card className="overflow-hidden">
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b border-slate-100 text-left text-xs text-slate-400">
              <th className="font-medium px-5 py-3">Date</th>
              <th className="font-medium px-5 py-3">Category</th>
              <th className="font-medium px-5 py-3">Note</th>
              <th className="font-medium px-5 py-3 text-right">Amount</th>
              <th className="font-medium px-5 py-3 w-10"></th>
            </tr>
          </thead>
          <tbody className="divide-y divide-slate-50">
            {txns.map((t) => (
              <tr key={t.id} className="hover:bg-slate-50/60">
                <td className="px-5 py-3 text-slate-500">{t.date}</td>
                <td className="px-5 py-3">
                  <span className="inline-flex items-center gap-1.5 text-slate-700 font-medium">
                    <span className={`w-1.5 h-1.5 rounded-full ${t.type === "income" ? "bg-emerald-500" : "bg-indigo-400"}`} />
                    {t.category}
                  </span>
                </td>
                <td className="px-5 py-3 text-slate-500">{t.note}</td>
                <td
                  className={`px-5 py-3 text-right font-semibold ${t.type === "income" ? "text-emerald-600" : "text-rose-600"}`}
                  style={{ fontFamily: "'IBM Plex Mono', monospace" }}
                >
                  {t.type === "income" ? "+" : "−"} {inr(t.amount)}
                </td>
                <td className="px-5 py-3 text-right">
                  <button onClick={() => onDelete(t.id)} className="text-slate-300 hover:text-rose-500">
                    <Trash2 className="w-4 h-4" />
                  </button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
        {txns.length === 0 && <p className="text-sm text-slate-400 py-10 text-center">No transactions to show</p>}
      </Card>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Watchlist                                                             */
/* ---------------------------------------------------------------------- */
function WatchlistView({ watchlist, onAdd, onRemove }) {
  return (
    <div>
      <div className="flex justify-end mb-5">
        <button
          onClick={onAdd}
          className="flex items-center gap-1.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold px-4 py-2 rounded-xl"
        >
          <Plus className="w-4 h-4" /> Add to watchlist
        </button>
      </div>
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
        {watchlist.map((w) => {
          const dist = w.target ? ((w.target - w.price) / w.price) * 100 : null;
          return (
            <Card key={w.id} className="p-5 relative group">
              <button
                onClick={() => onRemove(w.id)}
                className="absolute top-4 right-4 text-slate-300 hover:text-rose-500 opacity-0 group-hover:opacity-100 transition-opacity"
              >
                <Trash2 className="w-4 h-4" />
              </button>
              <div className="flex items-center gap-3 mb-4">
                <div className="w-10 h-10 rounded-xl bg-indigo-50 flex items-center justify-center text-indigo-600 text-xs font-bold">
                  {w.symbol.slice(0, 3)}
                </div>
                <div className="min-w-0">
                  <div className="text-sm font-semibold text-slate-900 truncate">{w.symbol}</div>
                  <div className="text-xs text-slate-400 truncate">{w.name}</div>
                </div>
              </div>
              <div className="text-xl font-semibold text-slate-900" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
                {inrDec(w.price)}
              </div>
              {w.target ? (
                <div className="text-xs text-slate-400 mt-1.5">
                  Target {inr(w.target)} ·{" "}
                  <span className={dist >= 0 ? "text-emerald-600 font-medium" : "text-rose-600 font-medium"}>
                    {dist >= 0 ? "+" : ""}
                    {dist.toFixed(1)}% to go
                  </span>
                </div>
              ) : (
                <div className="text-xs text-slate-300 mt-1.5">No target set</div>
              )}
            </Card>
          );
        })}
        {watchlist.length === 0 && <p className="text-sm text-slate-400 py-10 text-center col-span-full">Your watchlist is empty</p>}
      </div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Portfolio                                                             */
/* ---------------------------------------------------------------------- */
function PortfolioView({ portfolio, value, cost, gain, gainPct, onAdd, onRemove }) {
  return (
    <div>
      <div className="grid grid-cols-1 sm:grid-cols-3 gap-4 mb-6">
        <StatCard icon={<Wallet className="w-5 h-5 text-indigo-600" />} tint="#EEF2FF" label="Current value" value={inr(value)} />
        <StatCard icon={<Coins className="w-5 h-5 text-slate-600" />} tint="#F1F5F9" label="Total invested" value={inr(cost)} />
        <StatCard
          icon={gain >= 0 ? <TrendingUp className="w-5 h-5 text-emerald-600" /> : <TrendingDown className="w-5 h-5 text-rose-600" />}
          tint={gain >= 0 ? "#ECFDF5" : "#FEF2F2"}
          label="Overall gain / loss"
          value={`${gain >= 0 ? "+" : ""}${inr(gain)}`}
          delta={`${gainPct.toFixed(1)}%`}
          deltaPositive={gain >= 0}
        />
      </div>

      <div className="flex justify-end mb-5">
        <button
          onClick={onAdd}
          className="flex items-center gap-1.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold px-4 py-2 rounded-xl"
        >
          <Plus className="w-4 h-4" /> Add holding
        </button>
      </div>

      <Card className="overflow-hidden">
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b border-slate-100 text-left text-xs text-slate-400">
              <th className="font-medium px-5 py-3">Asset</th>
              <th className="font-medium px-5 py-3 text-right">Qty</th>
              <th className="font-medium px-5 py-3 text-right">Avg cost</th>
              <th className="font-medium px-5 py-3 text-right">LTP</th>
              <th className="font-medium px-5 py-3 text-right">Value</th>
              <th className="font-medium px-5 py-3 text-right">Gain</th>
              <th className="font-medium px-5 py-3 w-10"></th>
            </tr>
          </thead>
          <tbody className="divide-y divide-slate-50">
            {portfolio.map((h) => {
              const hVal = h.qty * h.currentPrice;
              const hCost = h.qty * h.avgPrice;
              const hGainPct = hCost > 0 ? ((hVal - hCost) / hCost) * 100 : 0;
              return (
                <tr key={h.id} className="hover:bg-slate-50/60">
                  <td className="px-5 py-3">
                    <div className="font-semibold text-slate-900">{h.symbol}</div>
                    <div className="text-xs text-slate-400">{h.name}</div>
                  </td>
                  <td className="px-5 py-3 text-right text-slate-600">{h.qty}</td>
                  <td className="px-5 py-3 text-right text-slate-600" style={{ fontFamily: "'IBM Plex Mono', monospace" }}>{inrDec(h.avgPrice)}</td>
                  <td className="px-5 py-3 text-right text-slate-600" style={{ fontFamily: "'IBM Plex Mono', monospace" }}>{inrDec(h.currentPrice)}</td>
                  <td className="px-5 py-3 text-right font-semibold text-slate-900" style={{ fontFamily: "'IBM Plex Mono', monospace" }}>{inr(hVal)}</td>
                  <td className={`px-5 py-3 text-right font-semibold ${hGainPct >= 0 ? "text-emerald-600" : "text-rose-600"}`}>
                    {hGainPct >= 0 ? "+" : ""}{hGainPct.toFixed(1)}%
                  </td>
                  <td className="px-5 py-3 text-right">
                    <button onClick={() => onRemove(h.id)} className="text-slate-300 hover:text-rose-500">
                      <Trash2 className="w-4 h-4" />
                    </button>
                  </td>
                </tr>
              );
            })}
          </tbody>
        </table>
        {portfolio.length === 0 && <p className="text-sm text-slate-400 py-10 text-center">No holdings yet</p>}
      </Card>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Savings                                                               */
/* ---------------------------------------------------------------------- */
function SavingsView({ goals, onAdd, onRemove, onContribute }) {
  return (
    <div>
      <div className="flex justify-end mb-5">
        <button
          onClick={onAdd}
          className="flex items-center gap-1.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold px-4 py-2 rounded-xl"
        >
          <Plus className="w-4 h-4" /> New goal
        </button>
      </div>
      <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
        {goals.map((g) => {
          const pct = Math.min(100, Math.round((g.current / g.target) * 100));
          return (
            <Card key={g.id} className="p-5">
              <div className="flex items-start justify-between mb-3">
                <div className="flex items-center gap-3">
                  <div className="w-10 h-10 rounded-xl bg-amber-50 flex items-center justify-center">
                    <Target className="w-4.5 h-4.5 text-amber-600" />
                  </div>
                  <div>
                    <div className="text-sm font-semibold text-slate-900">{g.name}</div>
                    <div className="text-xs text-slate-400">By {g.deadline}</div>
                  </div>
                </div>
                <button onClick={() => onRemove(g.id)} className="text-slate-300 hover:text-rose-500">
                  <Trash2 className="w-4 h-4" />
                </button>
              </div>
              <div className="flex items-baseline justify-between mb-2">
                <span className="text-lg font-semibold text-slate-900" style={{ fontFamily: "'Space Grotesk', sans-serif" }}>
                  {inr(g.current)}
                </span>
                <span className="text-xs text-slate-400">of {inr(g.target)}</span>
              </div>
              <div className="h-2 rounded-full bg-slate-100 overflow-hidden mb-3">
                <div className="h-full bg-indigo-600 rounded-full transition-all" style={{ width: `${pct}%` }} />
              </div>
              <div className="flex items-center justify-between">
                <span className="text-xs font-semibold text-indigo-600">{pct}% funded</span>
                <button onClick={() => onContribute(g)} className="text-xs font-semibold text-slate-600 hover:text-indigo-600 flex items-center gap-1">
                  <Pencil className="w-3 h-3" /> Add funds
                </button>
              </div>
            </Card>
          );
        })}
        {goals.length === 0 && <p className="text-sm text-slate-400 py-10 text-center col-span-full">No savings goals yet</p>}
      </div>
    </div>
  );
}

/* ---------------------------------------------------------------------- */
/*  Forms (modals)                                                        */
/* ---------------------------------------------------------------------- */
function TxnFormModal({ onClose, onSubmit }) {
  const [type, setType] = useState("expense");
  const [category, setCategory] = useState(CATEGORIES[0]);
  const [amount, setAmount] = useState("");
  const [note, setNote] = useState("");
  const [date, setDate] = useState(new Date().toISOString().slice(0, 10));

  function submit(e) {
    e.preventDefault();
    if (!amount || Number(amount) <= 0) return;
    onSubmit({ type, category, amount: Number(amount), note, date });
  }

  return (
    <Modal title="Add transaction" onClose={onClose}>
      <form onSubmit={submit}>
        <div className="flex gap-2 mb-4">
          {["expense", "income"].map((t) => (
            <button
              type="button"
              key={t}
              onClick={() => setType(t)}
              className={`flex-1 py-2 rounded-lg text-sm font-semibold capitalize border ${
                type === t ? "bg-indigo-600 text-white border-indigo-600" : "bg-white text-slate-500 border-slate-200"
              }`}
            >
              {t}
            </button>
          ))}
        </div>
        <Field label="Amount (₹)">
          <input required type="number" min="1" className={inputCls} value={amount} onChange={(e) => setAmount(e.target.value)} placeholder="0" />
        </Field>
        <Field label="Category">
          <select className={inputCls} value={category} onChange={(e) => setCategory(e.target.value)}>
            {CATEGORIES.map((c) => (
              <option key={c}>{c}</option>
            ))}
          </select>
        </Field>
        <Field label="Note">
          <input className={inputCls} value={note} onChange={(e) => setNote(e.target.value)} placeholder="Optional note" />
        </Field>
        <Field label="Date">
          <input required type="date" className={inputCls} value={date} onChange={(e) => setDate(e.target.value)} />
        </Field>
        <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2.5 rounded-lg mt-2">
          Save transaction
        </button>
      </form>
    </Modal>
  );
}

function WatchFormModal({ onClose, onSubmit }) {
  const [symbol, setSymbol] = useState("");
  const [name, setName] = useState("");
  const [price, setPrice] = useState("");
  const [target, setTarget] = useState("");

  function submit(e) {
    e.preventDefault();
    if (!symbol || !price) return;
    onSubmit({ symbol: symbol.toUpperCase(), name, price: Number(price), target: target ? Number(target) : null });
  }

  return (
    <Modal title="Add to watchlist" onClose={onClose}>
      <form onSubmit={submit}>
        <Field label="Symbol">
          <input required className={inputCls} value={symbol} onChange={(e) => setSymbol(e.target.value)} placeholder="e.g. TCS" />
        </Field>
        <Field label="Company name">
          <input className={inputCls} value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. Tata Consultancy Services" />
        </Field>
        <Field label="Current price (₹)">
          <input required type="number" step="0.05" className={inputCls} value={price} onChange={(e) => setPrice(e.target.value)} />
        </Field>
        <Field label="Target price (₹, optional)">
          <input type="number" step="0.05" className={inputCls} value={target} onChange={(e) => setTarget(e.target.value)} />
        </Field>
        <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2.5 rounded-lg mt-2">
          Add to watchlist
        </button>
      </form>
    </Modal>
  );
}

function HoldingFormModal({ onClose, onSubmit }) {
  const [symbol, setSymbol] = useState("");
  const [name, setName] = useState("");
  const [qty, setQty] = useState("");
  const [avgPrice, setAvgPrice] = useState("");
  const [currentPrice, setCurrentPrice] = useState("");

  function submit(e) {
    e.preventDefault();
    if (!symbol || !qty || !avgPrice || !currentPrice) return;
    onSubmit({ symbol: symbol.toUpperCase(), name, qty: Number(qty), avgPrice: Number(avgPrice), currentPrice: Number(currentPrice) });
  }

  return (
    <Modal title="Add holding" onClose={onClose}>
      <form onSubmit={submit}>
        <Field label="Symbol">
          <input required className={inputCls} value={symbol} onChange={(e) => setSymbol(e.target.value)} placeholder="e.g. RELIANCE" />
        </Field>
        <Field label="Name">
          <input className={inputCls} value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. Reliance Industries" />
        </Field>
        <Field label="Quantity">
          <input required type="number" step="0.001" className={inputCls} value={qty} onChange={(e) => setQty(e.target.value)} />
        </Field>
        <Field label="Average buy price (₹)">
          <input required type="number" step="0.05" className={inputCls} value={avgPrice} onChange={(e) => setAvgPrice(e.target.value)} />
        </Field>
        <Field label="Current price (₹)">
          <input required type="number" step="0.05" className={inputCls} value={currentPrice} onChange={(e) => setCurrentPrice(e.target.value)} />
        </Field>
        <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2.5 rounded-lg mt-2">
          Add holding
        </button>
      </form>
    </Modal>
  );
}

function GoalFormModal({ onClose, onSubmit }) {
  const [name, setName] = useState("");
  const [target, setTarget] = useState("");
  const [current, setCurrent] = useState("");
  const [deadline, setDeadline] = useState("");

  function submit(e) {
    e.preventDefault();
    if (!name || !target) return;
    onSubmit({ name, target: Number(target), current: Number(current) || 0, deadline });
  }

  return (
    <Modal title="New savings goal" onClose={onClose}>
      <form onSubmit={submit}>
        <Field label="Goal name">
          <input required className={inputCls} value={name} onChange={(e) => setName(e.target.value)} placeholder="e.g. Emergency fund" />
        </Field>
        <Field label="Target amount (₹)">
          <input required type="number" className={inputCls} value={target} onChange={(e) => setTarget(e.target.value)} />
        </Field>
        <Field label="Already saved (₹)">
          <input type="number" className={inputCls} value={current} onChange={(e) => setCurrent(e.target.value)} placeholder="0" />
        </Field>
        <Field label="Target date">
          <input type="date" className={inputCls} value={deadline} onChange={(e) => setDeadline(e.target.value)} />
        </Field>
        <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2.5 rounded-lg mt-2">
          Create goal
        </button>
      </form>
    </Modal>
  );
}

function ContributeModal({ goal, onClose, onSubmit }) {
  const [amount, setAmount] = useState("");
  function submit(e) {
    e.preventDefault();
    if (!amount) return;
    onSubmit(goal.id, Number(amount));
  }
  return (
    <Modal title={`Add funds — ${goal.name}`} onClose={onClose}>
      <form onSubmit={submit}>
        <Field label="Amount to add (₹)">
          <input required type="number" autoFocus className={inputCls} value={amount} onChange={(e) => setAmount(e.target.value)} placeholder="0" />
        </Field>
        <button type="submit" className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-semibold py-2.5 rounded-lg mt-2">
          Add funds
        </button>
      </form>
    </Modal>
  );
}
