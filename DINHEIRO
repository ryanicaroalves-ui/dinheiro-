import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';
import {
  Wallet, TrendingUp, TrendingDown, ArrowLeftRight, CreditCard,
  PiggyBank, Plus, X, AlertTriangle, Settings as SettingsIcon, Trash2,
  Landmark, ReceiptText, Check
} from 'lucide-react';

const STORAGE_KEY = 'controle-financeiro-v1';

const DEFAULT_DATA = {
  transactions: [],
  cards: [],
  settings: { payday: 5, minReserve: 0, monthlyInvestmentGoal: 0 },
};

const EXPENSE_CATEGORIES = [
  'Moradia', 'Alimentação', 'Transporte', 'Lazer', 'Saúde', 'Educação',
  'Assinaturas', 'Compras', 'Contas', 'Impostos', 'Cartão', 'Outros',
];
const INCOME_CATEGORIES = ['Salário', 'Outras entradas'];

const TYPE_META = {
  entrada: { label: 'Entrada', group: 'in' },
  gasto: { label: 'Gasto', group: 'out' },
  transferencia: { label: 'Transferência interna', group: 'neutral' },
  aporte: { label: 'Aporte em investimento', group: 'invest' },
  resgate: { label: 'Resgate de investimento', group: 'neutral' },
  rendimento: { label: 'Rendimento', group: 'in' },
  dividendo: { label: 'Dividendo', group: 'in' },
};

function uid() { return Math.random().toString(36).slice(2, 10) + Date.now().toString(36); }
function todayStr() { return new Date().toISOString().slice(0, 10); }
function monthKeyOf(dateStr) { return dateStr.slice(0, 7); }
function addMonths(mKey, n) {
  const [y, m] = mKey.split('-').map(Number);
  const d = new Date(y, m - 1 + n, 1);
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}`;
}
function formatBRL(v) {
  const n = Number(v) || 0;
  return n.toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
}
function formatDateBR(dateStr) {
  if (!dateStr) return '';
  const [, m, d] = dateStr.split('-');
  return `${d}/${m}`;
}
function monthLabel(mKey) {
  const [y, m] = mKey.split('-').map(Number);
  const d = new Date(y, m - 1, 1);
  const s = d.toLocaleDateString('pt-BR', { month: 'long', year: 'numeric' });
  return s.charAt(0).toUpperCase() + s.slice(1);
}

function computeAccountBalances(transactions) {
  const balances = {};
  const sorted = [...transactions].sort((a, b) => a.date.localeCompare(b.date));
  for (const t of sorted) {
    if (t.type === 'entrada' || t.type === 'rendimento' || t.type === 'dividendo') {
      balances[t.account] = (balances[t.account] || 0) + t.amount;
    } else if (t.type === 'gasto') {
      balances[t.account] = (balances[t.account] || 0) - t.amount;
    } else {
      balances[t.from] = (balances[t.from] || 0) - t.amount;
      balances[t.to] = (balances[t.to] || 0) + t.amount;
    }
  }
  return balances;
}

function computeNetWorthHistory(transactions) {
  const sorted = [...transactions].sort((a, b) => a.date.localeCompare(b.date));
  const balances = {};
  const byDate = new Map();
  for (const t of sorted) {
    if (t.type === 'entrada' || t.type === 'rendimento' || t.type === 'dividendo') {
      balances[t.account] = (balances[t.account] || 0) + t.amount;
    } else if (t.type === 'gasto') {
      balances[t.account] = (balances[t.account] || 0) - t.amount;
    } else {
      balances[t.from] = (balances[t.from] || 0) - t.amount;
      balances[t.to] = (balances[t.to] || 0) + t.amount;
    }
    const total = Object.values(balances).reduce((s, v) => s + v, 0);
    byDate.set(t.date, total);
  }
  return Array.from(byDate.entries()).map(([date, value]) => ({ date, value }));
}

function cardStatus(card, currentMonth) {
  let utilizado = 0, faturaAtual = 0, proximaFatura = 0;
  for (const p of card.purchases) {
    const parcela = p.totalAmount / p.installments;
    if (p.paidCount < p.installments) {
      utilizado += p.totalAmount - parcela * p.paidCount;
      const dueMonth = addMonths(monthKeyOf(p.date), p.paidCount);
      if (dueMonth === currentMonth) faturaAtual += parcela;
      if (dueMonth === addMonths(currentMonth, 1)) proximaFatura += parcela;
    }
  }
  return { utilizado, faturaAtual, proximaFatura, disponivel: card.limit - utilizado };
}

const STYLE = `
@import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,450;9..144,560;9..144,650&family=IBM+Plex+Mono:wght@400;500;600&family=Inter:wght@400;500;600&display=swap');
.cf-root{
  --paper:#EEF1E9; --panel:#E6EADF; --ink:#20291F; --ink-soft:#5B6A5A;
  --line:#C7CFBE; --income:#2F6B4F; --expense:#8C3B33; --transfer:#3A5273;
  --invest:#A9791F; --accent:#A9791F;
  font-family:'Inter',sans-serif; background:var(--paper); color:var(--ink);
  min-height:100vh;
}
.cf-serif{ font-family:'Fraunces',serif; }
.cf-mono{ font-family:'IBM Plex Mono',monospace; font-variant-numeric:tabular-nums; }
.cf-hline{ border-color:var(--line); }
.cf-tab{ border-bottom:2px solid transparent; color:var(--ink-soft); }
.cf-tab-active{ border-bottom:2px solid var(--ink); color:var(--ink); }
.cf-row:hover{ background:var(--panel); }
.cf-input{
  background:var(--paper); border:1px solid var(--line); color:var(--ink);
  font-family:'Inter',sans-serif;
}
.cf-input:focus{ outline:2px solid var(--ink); outline-offset:-1px; }
.cf-btn-dark{ background:var(--ink); color:var(--paper); }
.cf-btn-dark:hover{ opacity:0.85; }
.cf-btn-ghost{ border:1px solid var(--line); color:var(--ink); background:transparent; }
.cf-btn-ghost:hover{ background:var(--panel); }
.cf-scroll::-webkit-scrollbar{ width:6px; height:6px; }
.cf-scroll::-webkit-scrollbar-thumb{ background:var(--line); }
`;

function Amount({ value, kind, size = 'base' }) {
  const colorVar = kind === 'in' ? 'var(--income)' : kind === 'out' ? 'var(--expense)' : kind === 'invest' ? 'var(--invest)' : 'var(--ink)';
  const sizeClass = size === 'lg' ? 'text-2xl' : size === 'xl' ? 'text-3xl' : 'text-sm';
  const prefix = kind === 'in' ? '+ ' : kind === 'out' ? '- ' : '';
  return (
    <span className={`cf-mono ${sizeClass}`} style={{ color: colorVar }}>
      {prefix}{formatBRL(Math.abs(value))}
    </span>
  );
}

function SectionTitle({ children, action }) {
  return (
    <div className="flex items-center justify-between mb-4">
      <h2 className="cf-serif text-xl" style={{ color: 'var(--ink)' }}>{children}</h2>
      {action}
    </div>
  );
}

function Modal({ title, onClose, children, wide }) {
  return (
    <div className="fixed inset-0 z-50 flex items-start sm:items-center justify-center p-3 sm:p-6 overflow-y-auto"
      style={{ background: 'rgba(32,41,31,0.45)' }}>
      <div className={`cf-root w-full ${wide ? 'max-w-2xl' : 'max-w-md'} my-6`}
        style={{ background: 'var(--paper)', border: '1px solid var(--line)' }}>
        <div className="flex items-center justify-between px-5 py-4 border-b cf-hline">
          <h3 className="cf-serif text-lg">{title}</h3>
          <button onClick={onClose} className="p-1 cf-btn-ghost"><X size={18} /></button>
        </div>
        <div className="p-5">{children}</div>
      </div>
    </div>
  );
}

function Field({ label, children }) {
  return (
    <label className="block mb-3">
      <span className="block text-xs mb-1" style={{ color: 'var(--ink-soft)' }}>{label}</span>
      {children}
    </label>
  );
}

function TransactionForm({ accounts, onSave, onClose, initial }) {
  const [type, setType] = useState(initial?.type || 'gasto');
  const [amount, setAmount] = useState(initial?.amount || '');
  const [date, setDate] = useState(initial?.date || todayStr());
  const [category, setCategory] = useState(initial?.category || '');
  const [description, setDescription] = useState(initial?.description || '');
  const [account, setAccount] = useState(initial?.account || accounts[0] || 'Conta Corrente');
  const [from, setFrom] = useState(initial?.from || 'Conta Corrente');
  const [to, setTo] = useState(initial?.to || (type === 'aporte' ? 'Investimentos' : accounts[1] || 'Investimentos'));

  const isSimple = ['entrada', 'gasto', 'rendimento', 'dividendo'].includes(type);
  const isDouble = ['transferencia', 'aporte', 'resgate'].includes(type);
  const categories = type === 'gasto' ? EXPENSE_CATEGORIES : type === 'entrada' ? INCOME_CATEGORIES : [];

  function handleSubmit(e) {
    e.preventDefault();
    const amt = parseFloat(String(amount).replace(',', '.'));
    if (!amt || amt <= 0 || !date) return;
    const base = { id: initial?.id || uid(), type, amount: amt, date, description };
    if (isSimple) {
      onSave({ ...base, category: category || (type === 'gasto' ? 'Outros' : 'Outras entradas'), account });
    } else {
      onSave({ ...base, from, to });
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <Field label="Tipo">
        <select className="cf-input w-full px-3 py-2 text-sm" value={type} onChange={e => setType(e.target.value)}>
          {Object.entries(TYPE_META).map(([k, v]) => <option key={k} value={k}>{v.label}</option>)}
        </select>
      </Field>
      <div className="grid grid-cols-2 gap-3">
        <Field label="Valor (R$)">
          <input className="cf-input w-full px-3 py-2 text-sm cf-mono" inputMode="decimal" placeholder="0,00"
            value={amount} onChange={e => setAmount(e.target.value)} />
        </Field>
        <Field label="Data">
          <input type="date" className="cf-input w-full px-3 py-2 text-sm" value={date} onChange={e => setDate(e.target.value)} />
        </Field>
      </div>
      {isSimple && (
        <>
          <Field label="Conta">
            <input list="cf-accounts" className="cf-input w-full px-3 py-2 text-sm" value={account} onChange={e => setAccount(e.target.value)} />
          </Field>
          {(type === 'gasto' || type === 'entrada') && (
            <Field label="Categoria">
              <input list="cf-categories" className="cf-input w-full px-3 py-2 text-sm" value={category} onChange={e => setCategory(e.target.value)} placeholder="Escolha ou digite uma categoria" />
              <datalist id="cf-categories">{categories.map(c => <option key={c} value={c} />)}</datalist>
            </Field>
          )}
        </>
      )}
      {isDouble && (
        <div className="grid grid-cols-2 gap-3">
          <Field label="Conta de origem">
            <input list="cf-accounts" className="cf-input w-full px-3 py-2 text-sm" value={from} onChange={e => setFrom(e.target.value)} />
          </Field>
          <Field label="Conta de destino">
            <input list="cf-accounts" className="cf-input w-full px-3 py-2 text-sm" value={to} onChange={e => setTo(e.target.value)} />
          </Field>
        </div>
      )}
      <datalist id="cf-accounts">{accounts.map(a => <option key={a} value={a} />)}</datalist>
      <Field label="Descrição (opcional)">
        <input className="cf-input w-full px-3 py-2 text-sm" value={description} onChange={e => setDescription(e.target.value)} placeholder="Ex.: Pix para João" />
      </Field>
      <div className="flex justify-end gap-2 mt-4">
        <button type="button" onClick={onClose} className="px-4 py-2 text-sm cf-btn-ghost">Cancelar</button>
        <button type="submit" className="px-4 py-2 text-sm cf-btn-dark">Salvar movimentação</button>
      </div>
    </form>
  );
}

function CardForm({ onSave, onClose }) {
  const [name, setName] = useState('');
  const [limit, setLimit] = useState('');
  function handleSubmit(e) {
    e.preventDefault();
    const lim = parseFloat(String(limit).replace(',', '.'));
    if (!name || !lim) return;
    onSave({ id: uid(), name, limit: lim, purchases: [] });
  }
  return (
    <form onSubmit={handleSubmit}>
      <Field label="Nome do cartão"><input className="cf-input w-full px-3 py-2 text-sm" value={name} onChange={e => setName(e.target.value)} placeholder="Ex.: Cartão XP" /></Field>
      <Field label="Limite total (R$)"><input className="cf-input w-full px-3 py-2 text-sm cf-mono" inputMode="decimal" value={limit} onChange={e => setLimit(e.target.value)} placeholder="0,00" /></Field>
      <div className="flex justify-end gap-2 mt-4">
        <button type="button" onClick={onClose} className="px-4 py-2 text-sm cf-btn-ghost">Cancelar</button>
        <button type="submit" className="px-4 py-2 text-sm cf-btn-dark">Adicionar cartão</button>
      </div>
    </form>
  );
}

function PurchaseForm({ onSave, onClose }) {
  const [description, setDescription] = useState('');
  const [totalAmount, setTotalAmount] = useState('');
  const [installments, setInstallments] = useState(1);
  const [date, setDate] = useState(todayStr());
  const [category, setCategory] = useState('Compras');
  function handleSubmit(e) {
    e.preventDefault();
    const amt = parseFloat(String(totalAmount).replace(',', '.'));
    if (!amt || !description) return;
    onSave({ id: uid(), description, totalAmount: amt, installments: Number(installments) || 1, date, category, paidCount: 0 });
  }
  return (
    <form onSubmit={handleSubmit}>
      <Field label="Descrição"><input className="cf-input w-full px-3 py-2 text-sm" value={description} onChange={e => setDescription(e.target.value)} placeholder="Ex.: Tênis novo" /></Field>
      <div className="grid grid-cols-2 gap-3">
        <Field label="Valor total (R$)"><input className="cf-input w-full px-3 py-2 text-sm cf-mono" inputMode="decimal" value={totalAmount} onChange={e => setTotalAmount(e.target.value)} placeholder="0,00" /></Field>
        <Field label="Parcelas"><input type="number" min="1" className="cf-input w-full px-3 py-2 text-sm cf-mono" value={installments} onChange={e => setInstallments(e.target.value)} /></Field>
      </div>
      <div className="grid grid-cols-2 gap-3">
        <Field label="Data da compra"><input type="date" className="cf-input w-full px-3 py-2 text-sm" value={date} onChange={e => setDate(e.target.value)} /></Field>
        <Field label="Categoria">
          <input list="cf-categories-2" className="cf-input w-full px-3 py-2 text-sm" value={category} onChange={e => setCategory(e.target.value)} />
          <datalist id="cf-categories-2">{EXPENSE_CATEGORIES.map(c => <option key={c} value={c} />)}</datalist>
        </Field>
      </div>
      <div className="flex justify-end gap-2 mt-4">
        <button type="button" onClick={onClose} className="px-4 py-2 text-sm cf-btn-ghost">Cancelar</button>
        <button type="submit" className="px-4 py-2 text-sm cf-btn-dark">Registrar compra</button>
      </div>
    </form>
  );
}

function SettingsForm({ settings, onSave, onClose }) {
  const [payday, setPayday] = useState(settings.payday);
  const [minReserve, setMinReserve] = useState(settings.minReserve);
  const [monthlyInvestmentGoal, setMonthlyInvestmentGoal] = useState(settings.monthlyInvestmentGoal);
  function handleSubmit(e) {
    e.preventDefault();
    onSave({
      payday: Number(payday) || 5,
      minReserve: parseFloat(String(minReserve).replace(',', '.')) || 0,
      monthlyInvestmentGoal: parseFloat(String(monthlyInvestmentGoal).replace(',', '.')) || 0,
    });
  }
  return (
    <form onSubmit={handleSubmit}>
      <Field label="Dia do mês em que o salário cai"><input type="number" min="1" max="28" className="cf-input w-full px-3 py-2 text-sm cf-mono" value={payday} onChange={e => setPayday(e.target.value)} /></Field>
      <Field label="Reserva mínima que não deve ser gasta (R$)"><input className="cf-input w-full px-3 py-2 text-sm cf-mono" value={minReserve} onChange={e => setMinReserve(e.target.value)} /></Field>
      <Field label="Meta de investimento mensal (R$)"><input className="cf-input w-full px-3 py-2 text-sm cf-mono" value={monthlyInvestmentGoal} onChange={e => setMonthlyInvestmentGoal(e.target.value)} /></Field>
      <div className="flex justify-end gap-2 mt-4">
        <button type="button" onClick={onClose} className="px-4 py-2 text-sm cf-btn-ghost">Cancelar</button>
        <button type="submit" className="px-4 py-2 text-sm cf-btn-dark">Salvar</button>
      </div>
    </form>
  );
}

export default function ControleFinanceiro() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [saveError, setSaveError] = useState(false);
  const [tab, setTab] = useState('dashboard');
  const [modal, setModal] = useState(null);
  const [month, setMonth] = useState(monthKeyOf(todayStr()));

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get(STORAGE_KEY, false);
        setData(res ? JSON.parse(res.value) : DEFAULT_DATA);
      } catch (e) {
        setData(DEFAULT_DATA);
      } finally {
        setLoading(false);
      }
    })();
  }, []);

  const persist = useCallback(async (next) => {
    setData(next);
    try {
      const res = await window.storage.set(STORAGE_KEY, JSON.stringify(next), false);
      if (!res) setSaveError(true); else setSaveError(false);
    } catch (e) {
      setSaveError(true);
    }
  }, []);

  const addTransaction = (t) => { persist({ ...data, transactions: [...data.transactions, t] }); setModal(null); };
  const deleteTransaction = (id) => { persist({ ...data, transactions: data.transactions.filter(t => t.id !== id) }); };
  const addCard = (c) => { persist({ ...data, cards: [...data.cards, c] }); setModal(null); };
  const deleteCard = (id) => { persist({ ...data, cards: data.cards.filter(c => c.id !== id) }); };
  const addPurchase = (cardId, purchase) => {
    persist({ ...data, cards: data.cards.map(c => c.id === cardId ? { ...c, purchases: [...c.purchases, purchase] } : c) });
    setModal(null);
  };
  const saveSettings = (s) => { persist({ ...data, settings: s }); setModal(null); };

  const payInvoice = (cardId, targetMonth, fromAccount) => {
    const card = data.cards.find(c => c.id === cardId);
    if (!card) return;
    let total = 0;
    const purchases = card.purchases.map(p => {
      if (p.paidCount >= p.installments) return p;
      const dueMonth = addMonths(monthKeyOf(p.date), p.paidCount);
      if (dueMonth === targetMonth) {
        total += p.totalAmount / p.installments;
        return { ...p, paidCount: p.paidCount + 1 };
      }
      return p;
    });
    if (total <= 0) return;
    const newTransaction = {
      id: uid(), type: 'gasto', amount: total, date: todayStr(),
      category: 'Cartão', description: `Fatura ${card.name} (${monthLabel(targetMonth)})`, account: fromAccount,
    };
    persist({
      ...data,
      transactions: [...data.transactions, newTransaction],
      cards: data.cards.map(c => c.id === cardId ? { ...c, purchases } : c),
    });
  };

  const accounts = useMemo(() => {
    if (!data) return ['Conta Corrente', 'Investimentos'];
    const set = new Set(['Conta Corrente', 'Investimentos']);
    data.transactions.forEach(t => {
      if (t.account) set.add(t.account);
      if (t.from) set.add(t.from);
      if (t.to) set.add(t.to);
    });
    return Array.from(set);
  }, [data]);

  const balances = useMemo(() => data ? computeAccountBalances(data.transactions) : {}, [data]);
  const netWorthHistory = useMemo(() => data ? computeNetWorthHistory(data.transactions) : [], [data]);
  const currentMonth = monthKeyOf(todayStr());

  const totalCardDebt = useMemo(() => {
    if (!data) return 0;
    return data.cards.reduce((s, c) => s + cardStatus(c, currentMonth).utilizado, 0);
  }, [data, currentMonth]);

  const totalCardInvoiceUnpaid = useMemo(() => {
    if (!data) return 0;
    return data.cards.reduce((s, c) => s + cardStatus(c, currentMonth).faturaAtual, 0);
  }, [data, currentMonth]);

  const liquidBalance = useMemo(() => {
    return Object.entries(balances).filter(([k]) => k !== 'Investimentos').reduce((s, [, v]) => s + v, 0);
  }, [balances]);

  const investBalance = balances['Investimentos'] || 0;
  const netWorth = liquidBalance + investBalance - totalCardDebt;

  const monthTransactions = useMemo(() => {
    if (!data) return [];
    return data.transactions.filter(t => monthKeyOf(t.date) === month);
  }, [data, month]);

  const monthStats = useMemo(() => {
    const s = { entradas: 0, gastos: 0, investido: 0, rendimentos: 0, dividendos: 0, transferido: 0, byCategory: {} };
    for (const t of monthTransactions) {
      if (t.type === 'entrada') s.entradas += t.amount;
      if (t.type === 'gasto') { s.gastos += t.amount; s.byCategory[t.category] = (s.byCategory[t.category] || 0) + t.amount; }
      if (t.type === 'aporte') s.investido += t.amount;
      if (t.type === 'rendimento') s.rendimentos += t.amount;
      if (t.type === 'dividendo') s.dividendos += t.amount;
      if (t.type === 'transferencia') s.transferido += t.amount;
    }
    return s;
  }, [monthTransactions]);

  const investedThisMonth = useMemo(() => {
    return data ? data.transactions.filter(t => t.type === 'aporte' && monthKeyOf(t.date) === currentMonth).reduce((s, t) => s + t.amount, 0) : 0;
  }, [data, currentMonth]);

  const quantoPossoGastar = useMemo(() => {
    if (!data) return 0;
    const remainingGoal = Math.max(0, (data.settings.monthlyInvestmentGoal || 0) - investedThisMonth);
    return liquidBalance - totalCardInvoiceUnpaid - (data.settings.minReserve || 0) - remainingGoal;
  }, [data, liquidBalance, totalCardInvoiceUnpaid, investedThisMonth]);

  const daysUntilPayday = useMemo(() => {
    if (!data) return 1;
    const now = new Date();
    const payday = data.settings.payday || 5;
    let next = new Date(now.getFullYear(), now.getMonth(), payday);
    if (next <= now) next = new Date(now.getFullYear(), now.getMonth() + 1, payday);
    return Math.max(1, Math.ceil((next - now) / (1000 * 60 * 60 * 24)));
  }, [data]);

  const dailyBudget = quantoPossoGastar > 0 ? quantoPossoGastar / daysUntilPayday : 0;

  const alerts = useMemo(() => {
    if (!data) return [];
    const list = [];
    const today = todayStr();
    const todayTx = data.transactions.filter(t => t.date === today);
    const salaryToday = todayTx.find(t => t.type === 'entrada' && t.category === 'Salário');
    if (salaryToday) list.push({ text: `Seu salário caiu hoje: ${formatBRL(salaryToday.amount)}.`, tone: 'in' });
    const spentToday = todayTx.filter(t => t.type === 'gasto').reduce((s, t) => s + t.amount, 0);
    if (spentToday > 0) list.push({ text: `Você já gastou ${formatBRL(spentToday)} hoje.`, tone: 'out' });
    const dividendToday = todayTx.filter(t => t.type === 'dividendo').reduce((s, t) => s + t.amount, 0);
    if (dividendToday > 0) list.push({ text: `Você recebeu ${formatBRL(dividendToday)} de dividendos hoje.`, tone: 'in' });
    data.cards.forEach(c => {
      const st = cardStatus(c, currentMonth);
      if (st.faturaAtual > 0) {
        const payday = data.settings.payday || 5;
        const daysToDue = payday - new Date().getDate();
        if (daysToDue >= 0 && daysToDue <= 5) {
          list.push({ text: `A fatura do ${c.name} de ${formatBRL(st.faturaAtual)} vence em breve.`, tone: 'out' });
        }
      }
    });
    if (netWorthHistory.length >= 2) {
      const delta = netWorthHistory[netWorthHistory.length - 1].value - netWorthHistory[netWorthHistory.length - 2].value;
      if (Math.abs(delta) > 0.01) {
        list.push({ text: `Seu patrimônio ${delta >= 0 ? 'aumentou' : 'diminuiu'} ${formatBRL(Math.abs(delta))} desde a última movimentação.`, tone: delta >= 0 ? 'in' : 'out' });
      }
    }
    if (quantoPossoGastar > 0) list.push({ text: `Você pode gastar cerca de ${formatBRL(dailyBudget)} por dia até o próximo salário sem comprometer suas metas.`, tone: 'neutral' });
    return list;
  }, [data, currentMonth, netWorthHistory, quantoPossoGastar, dailyBudget]);

  if (loading || !data) {
    return (
      <div className="cf-root flex items-center justify-center p-10">
        <style>{STYLE}</style>
        <p className="cf-serif" style={{ color: 'var(--ink-soft)' }}>Carregando seu controle financeiro…</p>
      </div>
    );
  }

  const tabs = [
    { id: 'dashboard', label: 'Painel' },
    { id: 'transacoes', label: 'Transações' },
    { id: 'cartoes', label: 'Cartões' },
    { id: 'patrimonio', label: 'Patrimônio' },
    { id: 'resumo', label: 'Resumo do mês' },
  ];

  return (
    <div className="cf-root">
      <style>{STYLE}</style>
      <div className="max-w-5xl mx-auto px-4 sm:px-6 py-6">
        <div className="flex items-start justify-between mb-6">
          <div>
            <h1 className="cf-serif text-3xl" style={{ color: 'var(--ink)' }}>Controle Financeiro</h1>
            <p className="text-sm mt-1" style={{ color: 'var(--ink-soft)' }}>
              {new Date().toLocaleDateString('pt-BR', { weekday: 'long', day: 'numeric', month: 'long' })}
            </p>
          </div>
          <div className="flex gap-2">
            <button onClick={() => setModal({ kind: 'settings' })} className="p-2 cf-btn-ghost" title="Configurações">
              <SettingsIcon size={18} />
            </button>
            <button onClick={() => setModal({ kind: 'transaction' })} className="flex items-center gap-1 px-3 py-2 text-sm cf-btn-dark">
              <Plus size={16} /> Nova movimentação
            </button>
          </div>
        </div>

        {saveError && (
          <div className="mb-4 px-3 py-2 text-sm border" style={{ borderColor: 'var(--expense)', color: 'var(--expense)' }}>
            Não consegui salvar sua última alteração. Tente novamente em instantes.
          </div>
        )}

        <div className="flex gap-5 border-b cf-hline mb-6 overflow-x-auto cf-scroll">
          {tabs.map(t => (
            <button key={t.id} onClick={() => setTab(t.id)}
              className={`pb-3 text-sm whitespace-nowrap cf-tab ${tab === t.id ? 'cf-tab-active' : ''}`}>
              {t.label}
            </button>
          ))}
        </div>

        {tab === 'dashboard' && (
          <Dashboard
            liquidBalance={liquidBalance} investBalance={investBalance} netWorth={netWorth}
            totalCardDebt={totalCardDebt} quantoPossoGastar={quantoPossoGastar} dailyBudget={dailyBudget}
            monthStats={monthStats} alerts={alerts} netWorthHistory={netWorthHistory}
          />
        )}
        {tab === 'transacoes' && (
          <Transacoes transactions={data.transactions} onDelete={deleteTransaction} onAdd={() => setModal({ kind: 'transaction' })} />
        )}
        {tab === 'cartoes' && (
          <Cartoes
            cards={data.cards} currentMonth={currentMonth} accounts={accounts}
            onAddCard={() => setModal({ kind: 'card' })}
            onAddPurchase={(cardId) => setModal({ kind: 'purchase', cardId })}
            onDeleteCard={deleteCard}
            onPayInvoice={payInvoice}
          />
        )}
        {tab === 'patrimonio' && (
          <Patrimonio netWorthHistory={netWorthHistory} liquidBalance={liquidBalance} investBalance={investBalance} totalCardDebt={totalCardDebt} netWorth={netWorth} />
        )}
        {tab === 'resumo' && (
          <Resumo month={month} setMonth={setMonth} monthStats={monthStats} />
        )}
      </div>

      {modal?.kind === 'transaction' && (
        <Modal title="Nova movimentação" onClose={() => setModal(null)}>
          <TransactionForm accounts={accounts} onSave={addTransaction} onClose={() => setModal(null)} />
        </Modal>
      )}
      {modal?.kind === 'card' && (
        <Modal title="Novo cartão" onClose={() => setModal(null)}>
          <CardForm onSave={addCard} onClose={() => setModal(null)} />
        </Modal>
      )}
      {modal?.kind === 'purchase' && (
        <Modal title="Registrar compra no cartão" onClose={() => setModal(null)}>
          <PurchaseForm onSave={(p) => addPurchase(modal.cardId, p)} onClose={() => setModal(null)} />
        </Modal>
      )}
      {modal?.kind === 'settings' && (
        <Modal title="Configurações" onClose={() => setModal(null)}>
          <SettingsForm settings={data.settings} onSave={saveSettings} onClose={() => setModal(null)} />
        </Modal>
      )}
    </div>
  );
}

function StatRow({ icon: Icon, label, value, kind, big }) {
  return (
    <div className="flex items-center justify-between py-3 border-b cf-hline">
      <div className="flex items-center gap-2">
        <Icon size={16} style={{ color: 'var(--ink-soft)' }} />
        <span className={big ? 'cf-serif text-base' : 'text-sm'} style={{ color: 'var(--ink-soft)' }}>{label}</span>
      </div>
      <Amount value={value} kind={kind} size={big ? 'lg' : 'base'} />
    </div>
  );
}

function Dashboard({ liquidBalance, investBalance, netWorth, totalCardDebt, quantoPossoGastar, dailyBudget, monthStats, alerts, netWorthHistory }) {
  const chartData = netWorthHistory.slice(-30).map(p => ({ ...p, label: formatDateBR(p.date) }));
  return (
    <div className="grid sm:grid-cols-3 gap-6">
      <div className="sm:col-span-2">
        <SectionTitle>Quanto você pode gastar</SectionTitle>
        <div className="p-5 mb-6" style={{ background: 'var(--panel)', border: '1px solid var(--line)' }}>
          <p className="text-xs mb-1" style={{ color: 'var(--ink-soft)' }}>Dinheiro realmente disponível</p>
          <p className="cf-mono text-3xl mb-2" style={{ color: quantoPossoGastar >= 0 ? 'var(--ink)' : 'var(--expense)' }}>
            {formatBRL(quantoPossoGastar)}
          </p>
          <p className="text-sm" style={{ color: 'var(--ink-soft)' }}>
            Você pode gastar aproximadamente <span className="cf-mono">{formatBRL(dailyBudget)}</span> por dia até o próximo salário, sem comprometer suas metas.
          </p>
        </div>

        <SectionTitle>Visão geral</SectionTitle>
        <div className="mb-6">
          <StatRow icon={Wallet} label="Saldo em contas" value={liquidBalance} kind="neutral" big />
          <StatRow icon={PiggyBank} label="Investido" value={investBalance} kind="invest" />
          <StatRow icon={CreditCard} label="Dívida em cartões" value={totalCardDebt} kind="out" />
          <StatRow icon={TrendingUp} label="Patrimônio líquido" value={netWorth} kind={netWorth >= 0 ? 'in' : 'out'} big />
        </div>

        <SectionTitle>Este mês</SectionTitle>
        <div className="grid grid-cols-2 gap-4">
          <StatRow icon={TrendingUp} label="Entradas" value={monthStats.entradas} kind="in" />
          <StatRow icon={TrendingDown} label="Gastos" value={monthStats.gastos} kind="out" />
          <StatRow icon={PiggyBank} label="Investido" value={monthStats.investido} kind="invest" />
          <StatRow icon={Landmark} label="Rendimentos + dividendos" value={monthStats.rendimentos + monthStats.dividendos} kind="in" />
        </div>

        {chartData.length >= 2 && (
          <div className="mt-6">
            <SectionTitle>Evolução recente</SectionTitle>
            <div style={{ height: 180 }}>
              <ResponsiveContainer width="100%" height="100%">
                <LineChart data={chartData}>
                  <CartesianGrid stroke="var(--line)" vertical={false} />
                  <XAxis dataKey="label" tick={{ fontSize: 11, fill: '#5B6A5A' }} axisLine={{ stroke: '#C7CFBE' }} tickLine={false} />
                  <YAxis hide domain={['auto', 'auto']} />
                  <Tooltip formatter={(v) => formatBRL(v)} contentStyle={{ fontFamily: 'IBM Plex Mono', fontSize: 12, border: '1px solid #C7CFBE' }} />
                  <Line type="monotone" dataKey="value" stroke="#2F6B4F" strokeWidth={2} dot={false} />
                </LineChart>
              </ResponsiveContainer>
            </div>
          </div>
        )}
      </div>

      <div>
        <SectionTitle>Alertas</SectionTitle>
        {alerts.length === 0 && <p className="text-sm" style={{ color: 'var(--ink-soft)' }}>Nenhum alerta por enquanto.</p>}
        <div className="space-y-3">
          {alerts.map((a, i) => (
            <div key={i} className="flex items-start gap-2 p-3 text-sm" style={{ background: 'var(--panel)', border: '1px solid var(--line)' }}>
              <AlertTriangle size={14} className="mt-0.5 flex-shrink-0" style={{ color: a.tone === 'out' ? 'var(--expense)' : a.tone === 'in' ? 'var(--income)' : 'var(--ink-soft)' }} />
              <span>{a.text}</span>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

function Transacoes({ transactions, onDelete, onAdd }) {
  const sorted = [...transactions].sort((a, b) => b.date.localeCompare(a.date));
  return (
    <div>
      <SectionTitle action={
        <button onClick={onAdd} className="flex items-center gap-1 px-3 py-1.5 text-sm cf-btn-ghost"><Plus size={14} /> Adicionar</button>
      }>Todas as movimentações</SectionTitle>
      {sorted.length === 0 && <p className="text-sm" style={{ color: 'var(--ink-soft)' }}>Nenhuma movimentação registrada ainda.</p>}
      <div>
        {sorted.map(t => {
          const meta = TYPE_META[t.type];
          const label = t.type === 'transferencia' || t.type === 'aporte' || t.type === 'resgate'
            ? `${meta.label}: ${t.from} → ${t.to}`
            : `${t.category}${t.account ? ' · ' + t.account : ''}`;
          return (
            <div key={t.id} className="cf-row flex items-center justify-between py-3 border-b cf-hline group">
              <div className="flex items-center gap-3">
                <span className="cf-mono text-xs" style={{ color: 'var(--ink-soft)' }}>{formatDateBR(t.date)}</span>
                <div>
                  <p className="text-sm">{t.description || meta.label}</p>
                  <p className="text-xs" style={{ color: 'var(--ink-soft)' }}>{label}</p>
                </div>
              </div>
              <div className="flex items-center gap-3">
                <Amount value={t.amount} kind={meta.group} />
                <button onClick={() => onDelete(t.id)} className="opacity-0 group-hover:opacity-100 p-1" style={{ color: 'var(--ink-soft)' }}>
                  <Trash2 size={14} />
                </button>
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

function Cartoes({ cards, currentMonth, accounts, onAddCard, onAddPurchase, onDeleteCard, onPayInvoice }) {
  const [payFrom, setPayFrom] = useState({});
  return (
    <div>
      <SectionTitle action={
        <button onClick={onAddCard} className="flex items-center gap-1 px-3 py-1.5 text-sm cf-btn-ghost"><Plus size={14} /> Novo cartão</button>
      }>Cartões de crédito</SectionTitle>
      {cards.length === 0 && <p className="text-sm" style={{ color: 'var(--ink-soft)' }}>Nenhum cartão cadastrado ainda.</p>}
      <div className="space-y-6">
        {cards.map(card => {
          const st = cardStatus(card, currentMonth);
          const from = payFrom[card.id] || accounts[0] || 'Conta Corrente';
          return (
            <div key={card.id} className="p-5" style={{ background: 'var(--panel)', border: '1px solid var(--line)' }}>
              <div className="flex items-center justify-between mb-3">
                <h3 className="cf-serif text-lg">{card.name}</h3>
                <button onClick={() => onDeleteCard(card.id)} style={{ color: 'var(--ink-soft)' }}><Trash2 size={14} /></button>
              </div>
              <div className="grid grid-cols-2 sm:grid-cols-4 gap-4 mb-4">
                <div><p className="text-xs" style={{ color: 'var(--ink-soft)' }}>Limite</p><p className="cf-mono">{formatBRL(card.limit)}</p></div>
                <div><p className="text-xs" style={{ color: 'var(--ink-soft)' }}>Utilizado</p><p className="cf-mono" style={{ color: 'var(--expense)' }}>{formatBRL(st.utilizado)}</p></div>
                <div><p className="text-xs" style={{ color: 'var(--ink-soft)' }}>Disponível</p><p className="cf-mono">{formatBRL(st.disponivel)}</p></div>
                <div><p className="text-xs" style={{ color: 'var(--ink-soft)' }}>Fatura atual</p><p className="cf-mono">{formatBRL(st.faturaAtual)}</p></div>
              </div>
              <p className="text-xs mb-3" style={{ color: 'var(--ink-soft)' }}>Próxima fatura: <span className="cf-mono">{formatBRL(st.proximaFatura)}</span></p>

              {card.purchases.length > 0 && (
                <div className="mb-4">
                  {card.purchases.map(p => (
                    <div key={p.id} className="flex items-center justify-between py-2 border-b cf-hline text-sm">
                      <span>{p.description} <span style={{ color: 'var(--ink-soft)' }}>({p.paidCount}/{p.installments} parcelas)</span></span>
                      <span className="cf-mono">{formatBRL(p.totalAmount)}</span>
                    </div>
                  ))}
                </div>
              )}

              <div className="flex flex-wrap items-center gap-2">
                <button onClick={() => onAddPurchase(card.id)} className="flex items-center gap-1 px-3 py-1.5 text-sm cf-btn-ghost"><Plus size={14} /> Compra</button>
                {st.faturaAtual > 0 && (
                  <>
                    <select value={from} onChange={e => setPayFrom({ ...payFrom, [card.id]: e.target.value })} className="cf-input px-2 py-1.5 text-sm">
                      {accounts.filter(a => a !== 'Investimentos').map(a => <option key={a} value={a}>{a}</option>)}
                    </select>
                    <button onClick={() => onPayInvoice(card.id, currentMonth, from)} className="flex items-center gap-1 px-3 py-1.5 text-sm cf-btn-dark">
                      <Check size={14} /> Pagar fatura atual
                    </button>
                  </>
                )}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

function Patrimonio({ netWorthHistory, liquidBalance, investBalance, totalCardDebt, netWorth }) {
  const chartData = netWorthHistory.map(p => ({ ...p, label: formatDateBR(p.date) }));
  const parts = [
    { label: 'Contas', value: liquidBalance, color: 'var(--ink)' },
    { label: 'Investimentos', value: investBalance, color: 'var(--invest)' },
    { label: 'Dívida em cartões', value: -totalCardDebt, color: 'var(--expense)' },
  ];
  const maxAbs = Math.max(1, ...parts.map(p => Math.abs(p.value)));
  return (
    <div>
      <SectionTitle>Patrimônio líquido</SectionTitle>
      <p className="cf-mono text-3xl mb-6">{formatBRL(netWorth)}</p>

      {chartData.length >= 2 ? (
        <div style={{ height: 240 }} className="mb-8">
          <ResponsiveContainer width="100%" height="100%">
            <LineChart data={chartData}>
              <CartesianGrid stroke="var(--line)" vertical={false} />
              <XAxis dataKey="label" tick={{ fontSize: 11, fill: '#5B6A5A' }} axisLine={{ stroke: '#C7CFBE' }} tickLine={false} />
              <YAxis tick={{ fontSize: 11, fill: '#5B6A5A' }} axisLine={false} tickLine={false} width={70} tickFormatter={(v) => formatBRL(v)} />
              <Tooltip formatter={(v) => formatBRL(v)} contentStyle={{ fontFamily: 'IBM Plex Mono', fontSize: 12, border: '1px solid #C7CFBE' }} />
              <Line type="monotone" dataKey="value" stroke="#2F6B4F" strokeWidth={2} dot={false} />
            </LineChart>
          </ResponsiveContainer>
        </div>
      ) : (
        <p className="text-sm mb-8" style={{ color: 'var(--ink-soft)' }}>Adicione mais movimentações para ver a evolução ao longo do tempo.</p>
      )}
      <p className="text-xs mb-4" style={{ color: 'var(--ink-soft)' }}>O gráfico soma o saldo de contas e investimentos ao longo do tempo (não inclui a dívida do cartão histórica, apenas a atual).</p>

      <SectionTitle>Composição atual</SectionTitle>
      <div className="space-y-3">
        {parts.map(p => (
          <div key={p.label}>
            <div className="flex justify-between text-sm mb-1">
              <span style={{ color: 'var(--ink-soft)' }}>{p.label}</span>
              <span className="cf-mono">{formatBRL(p.value)}</span>
            </div>
            <div style={{ height: 6, background: 'var(--panel)' }}>
              <div style={{ height: 6, width: `${(Math.abs(p.value) / maxAbs) * 100}%`, background: p.color }} />
            </div>
          </div>
        ))}
      </div>
    </div>
  );
}

function Resumo({ month, setMonth, monthStats }) {
  const totalRecebido = monthStats.entradas + monthStats.rendimentos + monthStats.dividendos;
  const saveRate = totalRecebido > 0 ? ((totalRecebido - monthStats.gastos) / totalRecebido) * 100 : 0;
  const catEntries = Object.entries(monthStats.byCategory).sort((a, b) => b[1] - a[1]);
  const explainedTotal = monthStats.gastos + monthStats.investido + monthStats.transferido;
  const saldoRestante = monthStats.entradas - explainedTotal;

  function shiftMonth(delta) {
    setMonth(addMonths(month, delta));
  }

  return (
    <div>
      <div className="flex items-center gap-3 mb-6">
        <button onClick={() => shiftMonth(-1)} className="px-3 py-1.5 text-sm cf-btn-ghost">← Anterior</button>
        <h2 className="cf-serif text-xl">{monthLabel(month)}</h2>
        <button onClick={() => shiftMonth(1)} className="px-3 py-1.5 text-sm cf-btn-ghost">Próximo →</button>
      </div>

      <div className="grid sm:grid-cols-2 gap-8">
        <div>
          <SectionTitle>Resumo</SectionTitle>
          <StatRow icon={TrendingUp} label="Total recebido" value={totalRecebido} kind="in" />
          <StatRow icon={TrendingDown} label="Total gasto" value={monthStats.gastos} kind="out" />
          <StatRow icon={PiggyBank} label="Total investido" value={monthStats.investido} kind="invest" />
          <StatRow icon={Landmark} label="Rendimentos" value={monthStats.rendimentos} kind="in" />
          <StatRow icon={Landmark} label="Dividendos" value={monthStats.dividendos} kind="in" />
          <div className="mt-4 p-4" style={{ background: 'var(--panel)', border: '1px solid var(--line)' }}>
            <p className="text-xs mb-1" style={{ color: 'var(--ink-soft)' }}>Taxa de poupança</p>
            <p className="cf-mono text-2xl">{saveRate.toFixed(0)}%</p>
            {totalRecebido > 0 && (
              <p className="text-xs mt-2" style={{ color: 'var(--ink-soft)' }}>
                De cada R$ 100 recebidos, {formatBRL(monthStats.gastos / totalRecebido * 100).replace('R$', '')} foram gastos e {formatBRL(monthStats.investido / totalRecebido * 100).replace('R$', '')} foram investidos.
              </p>
            )}
          </div>
        </div>

        <div>
          <SectionTitle>Para onde foi o dinheiro</SectionTitle>
          {catEntries.length === 0 && <p className="text-sm" style={{ color: 'var(--ink-soft)' }}>Sem gastos categorizados neste mês.</p>}
          {catEntries.map(([cat, val]) => (
            <div key={cat} className="flex justify-between text-sm py-2 border-b cf-hline">
              <span>{cat}</span>
              <Amount value={val} kind="out" />
            </div>
          ))}
          {monthStats.investido > 0 && (
            <div className="flex justify-between text-sm py-2 border-b cf-hline">
              <span>Investido</span>
              <Amount value={monthStats.investido} kind="invest" />
            </div>
          )}
          {monthStats.transferido > 0 && (
            <div className="flex justify-between text-sm py-2 border-b cf-hline">
              <span>Transferido para outras contas</span>
              <span className="cf-mono text-sm">{formatBRL(monthStats.transferido)}</span>
            </div>
          )}
          <div className="flex justify-between text-sm py-2 mt-1">
            <span style={{ color: 'var(--ink-soft)' }}>Ainda disponível do que entrou</span>
            <span className="cf-mono">{formatBRL(saldoRestante)}</span>
          </div>
        </div>
      </div>
    </div>
  );
}
