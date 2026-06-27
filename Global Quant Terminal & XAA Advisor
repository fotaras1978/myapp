from __future__ import annotations
import streamlit as st
import math
import time
from datetime import datetime
from typing import Any

try:
    import numpy as np
    import pandas as pd
    import yfinance as yf
except ImportError:
    st.error("Λείπουν βιβλιοθήκες. Εγκατάστησε τες τρέχοντας: pip install pandas numpy yfinance streamlit")
    raise SystemExit(1)

# --- ΡΥΘΜΙΣΕΙΣ ΣΕΛΙΔΑΣ STREAMLIT ---
st.set_page_config(page_title="Global Quant Terminal", layout="wide", page_icon="🏛️")

# --- ΚΕΝΤΡΙΚΟΣ ΤΙΤΛΟΣ ΚΑΙ ΕΝΤΟΝΗ ΠΡΟΕΙΔΟΠΟΙΗΣΗ ΚΙΝΔΥΝΟΥ ---
st.title("🏛️ Global Quant Terminal & XAA Advisor")

# Η ΕΝΤΟΝΗ ΠΡΟΕΙΔΟΠΟΙΗΣΗ ΠΟΥ ΖΗΤΗΣΕΣ ΣΤΗΝ ΚΟΡΥΦΗ ΤΟΥ ΠΡΟΓΡΑΜΜΑΤΟΣ
st.error("⚠️ **ΣΗΜΑΝΤΙΚΗ ΠΡΟΕΙΔΟΠΟΙΗΣΗ: Η ΑΓΟΡΑ ΕΠΕΝΔΥΤΙΚΩΝ ΠΡΟΪΟΝΤΩΝ ΕΝΕΧΕΙ ΣΟΒΑΡΟΥΣ ΚΙΝΔΥΝΟΥΣ ΑΠΩΛΕΙΑΣ ΚΕΦΑΛΑΙΟΥ. ΟΙ ΑΝΑΛΥΣΕΙΣ ΤΟΥ ΣΥΣΤΗΜΑΤΟΣ ΕΙΝΑΙ ΚΑΘΑΡΑ ΠΟΣΟΤΙΚΕΣ / ΕΚΠΑΙΔΕΥΤΙΚΕΣ ΚΑΙ ΔΕΝ ΑΠΟΤΕΛΟΥΝ ΣΥΜΒΟΥΛΗ Ή ΠΡΟΤΡΟΠΗ ΓΙΑ ΕΠΕΝΔΥΣΕΙΣ.**")

st.markdown("*Επαγγελματικό πολυπαραγοντικό σύστημα αυτόματης αξιολόγησης διεθνών και ελληνικών τίτλων.*")

# --- ΠΑΓΚΟΣΜΙΟ ΥΠΟΣΥΜΠΑΝ ΜΕΤΟΧΩΝ (85+ ΤΙΤΛΟΙ) ---
GLOBAL_UNIVERSE = [
    "AAPL","MSFT","NVDA","AMZN","GOOGL","META","TSLA","AVGO","AMD","ASML","TSM",
    "ARM","AMAT","LRCX","KLAC","QCOM","ORCL","CRM","NOW","ADBE","INTU","PANW",
    "CRWD","SNOW","NET","DDOG","PLTR","SHOP","UBER","BKNG","MELI","SPOT","NFLX",
    "COST","WMT","LLY","NVO","VRTX","REGN","ISRG","TMO","DHR","UNH","MA","V",
    "JPM","MS","GS","BRK-B","CAT","GE","ETN","DE","URI","LIN",
    "SAP.DE","SIE.DE","RMS.PA","MC.PA","OR.PA","AIR.PA","SU.PA","SAN.PA",
    "NOVO-B.CO","ASML.AS","ADYEN.AS","NESN.SW","ROG.SW","AZN.L","REL.L",
    "7203.T","6758.T","6861.T","8035.T","6098.T","9983.T","6954.T","6501.T",
    "005930.KS","000660.KS","0700.HK","9988.HK","RELIANCE.NS","TCS.NS","INFY.NS"
]

GREEK_UNIVERSE = [
    "ETE.AT", "ALPHA.AT", "EUROB.AT", "PIR.AT", "OPAP.AT", "OTE.AT",
    "PPC.AT", "MYTIL.AT", "BELA.AT", "MOH.AT", "ELPE.AT", "TITC.AT",
    "GEKTERNA.AT", "LAMDA.AT", "AEGN.AT", "CENER.AT", "VIO.AT", "SAR.AT"
]

BENCHMARKS = {
    "^GSPC": "S&P 500",
    "^IXIC": "Nasdaq",
    "URTH": "MSCI World ETF",
    "^N225": "Nikkei 225",
    "^STOXX50E": "Euro Stoxx 50",
}

# --- ΒΟΗΘΗΤΙΚΟΙ ΜΑΘΗΜΑΤΙΚΟΙ ΜΕΤΑΣΧΗΜΑΤΙΣΜΟΙ ---
def n(x: Any, default=float("nan")) -> float:
    try:
        x = float(x)
        return x if not math.isnan(x) and not math.isinf(x) else default
    except Exception:
        return default

def clamp(x, lo=0, hi=100):
    return max(lo, min(hi, x)) if not math.isnan(x) else 0

def pct(x):
    return "n/a" if math.isnan(x) else f"{x*100:+.1f}%".replace(".", ",")

# ΑΝΑΒΑΘΜΙΣΜΕΝΗ ΣΥΝΑΡΤΗΣΗ ΝΟΜΙΣΜΑΤΩΝ ΓΙΑ ΔΙΕΘΝΗ ΧΡΗΜΑΤΙΣΤΗΡΙΑ
def money(x, cur=""):
    if math.isnan(x): return "n/a"
    # Έξυπνη διάγνωση τοπικού νομίσματος βάσει της κατάληξης του Ticker
    if ".AT" in cur or ".DE" in cur or ".PA" in cur or ".AS" in cur:
        symbol_cur = "€"
    elif ".L" in cur:
        symbol_cur = "£"
    elif ".T" in cur:
        symbol_cur = "¥"
    elif "USD" in cur or cur == "":
        symbol_cur = "$"
    else:
        symbol_cur = cur
    return f"{x:,.2f} {symbol_cur}".replace(",", "X").replace(".", ",").replace("X", ".")

def compact(x):
    if math.isnan(x): return "n/a"
    if abs(x) >= 1e12: return f"{x/1e12:.2f}T"
    if abs(x) >= 1e9: return f"{x/1e9:.2f}B"
    if abs(x) >= 1e6: return f"{x/1e6:.2f}M"
    return f"{x:.0f}"

# --- ΛΗΨΗ ΔΕΔΟΜΕΝΩΝ ΚΑΙ ΥΠΟΛΟΓΙΣΜΟΣ ΔΕΙΚΤΩΝ ---
@st.cache_data(ttl=600)
def get_history(symbol, period="5y"):
    try:
        df = yf.download(symbol, period=period, interval="1d", auto_adjust=True, progress=False)
    except Exception:
        return pd.DataFrame()
    if df.empty: return df
    if isinstance(df.columns, pd.MultiIndex):
        df.columns = df.columns.get_level_values(0)
    df.columns = [str(c).title() for c in df.columns]
    cols = ["Open","High","Low","Close","Volume"]
    if any(c not in df.columns for c in cols): return pd.DataFrame()
    return df[cols].dropna()

def add_indicators(df):
    d = df.copy()
    c, h, l, v = d["Close"], d["High"], d["Low"], d["Volume"].fillna(0)
    d["ret"] = c.pct_change()
    for w in (20, 50, 100, 200):
        d[f"sma{w}"] = c.rolling(w).mean()

    ema12 = c.ewm(span=12, adjust=False).mean()
    ema26 = c.ewm(span=26, adjust=False).mean()
    d["macd"] = ema12 - ema26
    d["macd_signal"] = d["macd"].ewm(span=9, adjust=False).mean()
    d["macd_hist"] = d["macd"] - d["macd_signal"]

    delta = c.diff()
    gain = delta.clip(lower=0).ewm(alpha=1/14, min_periods=14, adjust=False).mean()
    loss = (-delta.clip(upper=0)).ewm(alpha=1/14, min_periods=14, adjust=False).mean()
    d["rsi"] = 100 - 100 / (1 + gain / loss.replace(0, np.nan))

    prev = c.shift(1)
    tr = pd.concat([(h-l), (h-prev).abs(), (l-prev).abs()], axis=1).max(axis=1)
    d["atr"] = tr.rolling(14).mean()
    d["atr_pct"] = d["atr"] / c
    d["vol20"] = v.rolling(20).mean()
    d["high20"], d["low20"] = h.rolling(20).max(), l.rolling(20).min()
    d["high50"], d["low50"] = h.rolling(50).max(), l.rolling(50).min()
    d["high252"], d["low252"] = h.rolling(252, min_periods=60).max(), l.rolling(252, min_periods=60).min()
    d["drawdown"] = c / d["high252"] - 1
    return d.dropna(subset=["sma50","rsi","atr"])

def ret(close, days):
    if len(close.dropna()) <= days: return float("nan")
    old, new = n(close.iloc[-days-1]), n(close.iloc[-1])
    return new / old - 1 if old > 0 else float("nan")

@st.cache_data(ttl=3600)
def get_info(symbol):
    try:
        data = yf.Ticker(symbol).info
        return data if isinstance(data, dict) else {}
    except Exception:
        return {}

@st.cache_data(ttl=600)
def global_regime():
    scores, details = [], []
    for sym, name in BENCHMARKS.items():
        df = add_indicators(get_history(sym, "1y"))
        if df.empty or len(df) < 80: continue
        last = df.iloc[-1]
        price, sma50, sma200 = n(last["Close"]), n(last["sma50"]), n(last["sma200"])
        r3 = ret(df["Close"], 63)
        score = 50 + (18 if price > sma200 > 0 else -15) + (10 if sma50 > sma200 > 0 else -8) + (8 if r3 > 0 else -6)
        score = clamp(score)
        scores.append(score)
        details.append(f"**{name}**: Σκόρ {score:.1f}/100, 3μηνο {pct(r3)}, ΚΜΟ200: {'✅ Πάνω' if price > sma200 else '❌ Κάτω'}")
    avg = sum(scores)/len(scores) if scores else 50
    label = "🟢 Risk-On (Ανοδικό κλίμα παγκοσμίως)" if avg >= 68 else "🟡 Ουδέτερο κλίμα" if avg >= 48 else "🔴 Risk-Off (Αμυντικό κλίμα)"
    return avg, label, details

# --- ΠΟΣΟΤΙΚΗ ΜΟΝΤΕΛΟΠΟΙΗΣΗ (QUANT SCORING) ---
def fundamental_scores(info_dict):
    good, risks = [], []
    rg, eg = n(info_dict.get("revenueGrowth")), n(info_dict.get("earningsGrowth"))
    gm, om, pm = n(info_dict.get("grossMargins")), n(info_dict.get("operatingMargins")), n(info_dict.get("profitMargins"))
    roe, debt = n(info_dict.get("returnOnEquity")), n(info_dict.get("debtToEquity"))
    pe, fpe, ps, peg = n(info_dict.get("trailingPE")), n(info_dict.get("forwardPE")), n(info_dict.get("priceToSalesTrailing12Months")), n(info_dict.get("pegRatio"))

    growth = 50
    if rg > .20: growth += 18; good.append("Πολύ ισχυρή αύξηση εσόδων")
    elif rg > .10: growth += 12; good.append("Καλή αύξηση εσόδων")
    elif rg < -.03: growth -= 10; risks.append("Αρνητική τάση εσόδων")
    if eg > .25: growth += 18; good.append("Πολύ ισχυρή αύξηση κερδών")
    elif eg > .10: growth += 12; good.append("Καλή αύξηση κερδών")
    elif eg < -.05: growth -= 12; risks.append("Αρνητική τάση κερδών")

    quality = 50
    if gm > .55: quality += 8; good.append("Υψηλό μικτό περιθώριο (Gross Margin)")
    if om > .22: quality += 10; good.append("Υψηλό λειτουργικό περιθώριο (Operating Margin)")
    elif om < .04: quality -= 8; risks.append("Χαμηλό λειτουργικό περιθώριο")
    if pm > .15: quality += 10; good.append("Υγιές καθαρό περιθώριο (Profit Margin)")
    elif pm < .03: quality -= 8; risks.append("Χαμηλό καθαρό περιθώριο")
    if roe > .18: quality += 12; good.append("Υψηλή απόδοση ιδίων κεφαλαίων (ROE)")
    elif roe < .05: quality -= 7; risks.append("Χαμηλό ROE")
    if debt <= 80: quality += 5; good.append("Ελεγχόμενο εταιρικό χρέος")
    elif debt > 180: quality -= 10; risks.append("Υψηλό εταιρικό χρέος")

    valuation = 50
    if fpe > 0:
        if fpe <= 18: valuation += 14; good.append("Λογικό Forward P/E")
        elif fpe <= 32: valuation += 4
        elif fpe > 55: valuation -= 12; risks.append("Πολύ απαιτητικό/ακριβό Forward P/E")
    elif pe > 0:
        if pe <= 20: valuation += 10
        elif pe > 60: valuation -= 12; risks.append("Πολύ υψηλό τρέχον P/E")
    if ps > 0:
        if ps <= 5: valuation += 6
        elif ps > 18: valuation -= 10; risks.append("Υψηλός δείκτης Price-to-Sales")
    if peg > 0:
        if peg <= 1.6: valuation += 8; good.append("Λογικό PEG για αναπτυσσόμενη μετοχή")
        elif peg > 3: valuation -= 8; risks.append("Υψηλός δείκτης PEG")

    return clamp(growth), clamp(quality), clamp(valuation), good, risks

def megatrend(info_dict):
    text = f"{info_dict.get('sector','')} {info_dict.get('industry','')}".lower()
    score, reasons = 50, []
    keys = {
        "semiconductor": "Ημιαγωγοί / AI Infrastructure",
        "software": "Software με Scalable Μοντέλο",
        "cyber": "Cybersecurity",
        "biotechnology": "Βιοτεχνολογία",
        "drug": "Φαρμακευτική Καινοτομία",
        "medical": "Ιατρική Τεχνολογία",
        "aerospace": "Αεροδιαστημική / Άμυνα",
        "internet": "Πλατφόρμες Διαδικτύου",
        "payments": "Παγκόσμια Ψηφιακά Δίκτυα Πληρωμών",
    }
    for k, label in keys.items():
        if k in text:
            score += 12
            reasons.append(f"Έκθεση σε διεθνές Megatrend: {label}")
            break
    if "technology" in text and len(reasons) == 0:
        score += 7
        reasons.append("Ισχυρή έκθεση στον ευρύτερο τομέα της τεχνολογίας")
    return clamp(score), reasons

def analyze_stock(symbol, period, gscore, glabel):
    raw = get_history(symbol, period)
    if raw.empty or len(raw) < 180: return None
    df = add_indicators(raw)
    if df.empty or len(df) < 120: return None

    inf = get_info(symbol)
    last, prev = df.iloc[-1], df.iloc[-2]
    price = n(last["Close"])
    cur = symbol if "AT" in symbol else (inf.get("currency") or "")
    good, wait, risks = [], [], []

    tech = 45
    if price > last["sma200"]: tech += 14; good.append("Τιμή πάνω από τον ΚΜΟ 200 ημερών (Ανοδική Μακροπρόθεσμη Τάση)")
    else: tech -= 12; risks.append("Τιμή κάτω από τον ΚΜΟ 200 ημερών (Πτωτική Μακροπρόθεσμη Τάση)")
    if last["sma50"] > last["sma200"]: tech += 12; good.append("Golden Cross: ΚΜΟ 50 πάνω από ΚΜΟ 200")
    if price > last["sma20"] and price > last["sma50"]: tech += 9; good.append("Θετική βραχυπρόθεσμη ορμή (Τιμή > ΚΜΟ20 & ΚΜΟ50)")
    elif price < last["sma20"] and price < last["sma50"]: tech -= 8; risks.append("Τιμή κάτω από κοντινούς μέσους όρους")
    if last["macd"] > last["macd_signal"] and last["macd_hist"] > prev["macd_hist"]: tech += 8; good.append("Βελτίωση και αγοραστικό σήμα MACD")
    elif last["macd"] < last["macd_signal"]: tech -= 7; risks.append("Αδύναμο/Πτωτικό Momentum MACD")
    rsi = n(last["rsi"])
    if 45 <= rsi <= 68: tech += 7; good.append(f"Υγιές και ισορροπημένο RSI ({rsi:.1f})")
    elif rsi > 76: tech -= 8; risks.append(f"Υπεραγορασμένη ζώνη κινδύνου RSI ({rsi:.1f})")
    elif rsi < 35: tech -= 7; risks.append(f"Αδύναμο/Υπερπουλημένο RSI ({rsi:.1f})")

    r3, r6, r12 = ret(df["Close"], 63), ret(df["Close"], 126), ret(df["Close"], 252)
    if r3 > 0 and r6 > 0 and r12 > 0: tech += 10; good.append("Συγχρονισμένες θετικές αποδόσεις 3, 6 και 12 μηνών")
    elif r3 < 0 and r6 < 0: tech -= 8; risks.append("Αρνητική μεσοπρόθεσμη απόδοση")
    tech = clamp(tech)

    growth, quality, valuation, fgood, frisks = fundamental_scores(inf)
    mega, mgood = megatrend(inf)
    good += mgood + fgood
    risks += frisks

    volatility = n(df["ret"].dropna().tail(252).std() * math.sqrt(252))
    atr_pct = n(last["atr_pct"])
    high1, low1 = n(df["High"].tail(252).max()), n(df["Low"].tail(252).min())
    drawdown = price / high1 - 1 if high1 > 0 else float("nan")
    rebound = price / low1 - 1 if low1 > 0 else float("nan")
    liquidity = n((df.tail(20)["Close"] * df.tail(20)["Volume"]).mean())

    risk = 28
    if volatility > .45: risk += 18; risks.append("Υψηλή μεταβλητότητα διακύμανσης")
    if atr_pct > .04: risk += 10
    if drawdown < -.30: risk += 10; risks.append("Σημαντική υποχώρηση από τα υψηλά 12μήνου")
    if liquidity < 1_000_000: risk += 10
    risk = clamp(risk)

    atr = n(last["atr"], price * .03)
    stop = max(0, price - 2.5 * atr)
    base_target = price + 2.8 * atr
    
    # Η ΓΡΑΜΜΗ ΠΟΥ ΔΙΟΡΘΩΘΗΚΕ ΚΑΙ ΟΡΙΖΕΙ ΤΟ RESISTANCE
    resistance = min([x for x in [n(last["high20"]), n(last["high50"]), high1] if x > price], default=float("nan"))
    
    if not math.isnan(resistance):
        base_target = min(max(base_target, price + 1.5 * atr), resistance)
    bull_target = price + 5.0 * atr
    upside = base_target / price - 1
    bull_upside = bull_target / price - 1
    downside = stop / price - 1

    total = clamp(.24*growth + .22*quality + .20*tech + .14*valuation + .08*mega + .12*(100-risk) + (gscore-50)*.06)

    if total >= 78 and risk <= 52 and upside >= 0.07: rec = "🔥 ΔΥΝΑΤΗ ΑΓΟΡΑ (Top Quant Pick)"
    elif total >= 68 and risk <= 65: rec = "🟢 ΑΓΟΡΑ (Υψηλή Πιστότητα)"
    elif total >= 56: rec = "🛒 ΕΠΙΛΕΚΤΙΚΗ ΑΓΟΡΑ / WATCHLIST"
    elif risk >= 72 or total < 45: rec = "⛔ ΑΠΟΦΥΓΗ (Υψηλό Ρίσκο/Αδυναμία)"
    else: rec = "⏳ ΑΝΑΜΟΝΗ (Ουδέτερη Εικόνα)"

    name = inf.get("shortName") or inf.get("longName") or symbol
    
    return {
        "Μετοχή": name, "Σύμβολο": symbol, "Τιμή": money(price, cur), "Πρόταση": rec,
        "Total Score": total, "Tech Score": tech, "Growth Score": growth, "Quality Score": quality, "Valuation Score": valuation,
        "Upside %": f"{upside * 100:.1f}%", "Βασικός Στόχος": money(base_target, cur),
        "Αισιόδοξος Στόχος": money(bull_target, cur), "Stop Loss": money(stop, cur),
        "Good": good, "Wait": wait, "Risks": risks,
        "1M Return": pct(ret(df["Close"], 21)), "3M Return": pct(r3), "6M Return": pct(r6), "1Y Return": pct(r12),
        "Χώρα": inf.get("country","n/a"), "Κλάδος": inf.get("sector","n/a"), "Market Cap": compact(n(inf.get("marketCap")))
    }

# --- UI ΣΥΝΙΣΤΩΣΑ ΑΝΑΦΟΡΑΣ ---
def render_quant_report(r):
    st.markdown(f"### 📑 Ποσοτική Έκθεση Βάθους: **{r['Μετοχή']}** ({r['Σύμβολο']})")
    
    col_a, col_b, col_c = st.columns(3)
    col_a.metric("Σύσταση Μοντέλου", r['Πρόταση'])
    col_b.metric("Τρέχουσα Τιμή", r['Τιμή'])
    col_c.metric("Περιθώριο Κέρδους (Upside)", r['Upside %'])
    
    st.write("**🧮 Κατανομή AI Scoring (0-100):**")
    st.progress(int(r['Total Score']), text=f"Συνολικό Quant Score: {r['Total Score']:.1f}/100")
    st.progress(int(r['Tech Score']), text=f"Τεχνική Εικόνα (Tech): {r['Tech Score']:.1f}/100")
    st.progress(int(r['Growth Score']), text=f"Ρυθμός Ανάπτυξης (Growth): {r['Growth Score']:.1f}/100")
    st.progress(int(r['Quality Score']), text=f"Ποιότητα & Ισολογισμός (Quality): {r['Quality Score']:.1f}/100")
    st.progress(int(r['Valuation Score']), text=f"Αποτίμηση / Πολλαπλασιαστές (Valuation): {r['Valuation Score']:.1f}/100")
    
    st.write(f"🎯 **Επενδυτική Στρατηγική:** Βασικός Στόχος: **{r['Βασικός Στόχος']}** | Αισιόδοξος Στόχος (Bull): **{r['Αισιόδοξος Στόχος']}** | Όριο Ακύρωσης (Stop Loss): **{r['Stop Loss']}**")
    st.write(f"📊 **Ιστορικές Αποδόσεις:** 1 Μήνας: {r['1M Return']} | 3 Μήνες: {r['3M Return']} | 6 Μήνες: {r['6M Return']} | 1 Έτος: {r['1Y Return']}")
    st.write(f"🏢 **Προφίλ:** Χώρα: {r['Χώρα']} | Κλάδος: {r['Κλάδος']} | Κεφαλαιοποίηση: {r['Market Cap']}")
    
    c1, c2 = st.columns(2)
    with c1:
        st.success("🟢 **Θετικοί Καταλύτες / Δυνάμεις:**\n" + ("\n".join([f"- {x}" for x in r['Good']]) if r['Good'] else "- Κανένα έντονο θετικό στοιχείο."))
    with c2:
        st.warning("🟡 **Σημεία Προσοχής / Αναμονής:**\n" + ("\n".join([f"- {x}" for x in r['Wait']]) if r['Wait'] else "- Δεν εντοπίστηκαν τεχνικά εμπόδια."))
    if r['Risks']:
        st.error("🔴 **Κίνδυνοι & Απειλές (Risks):**\n" + "\n".join([f"- {x}" for x in r['Risks']]))

def execute_screening(universe_list, is_global, unique_key):
    results = []
    progress_bar = st.progress(0, text="Έναρξη ποσοτικής σάρωσης...")
    
    total = len(universe_list)
    for idx, sym in enumerate(universe_list):
        res = analyze_stock(sym, "5y", gscore, glabel)
        if res: results.append(res)
        progress_bar.progress((idx + 1) / total, text=f"Αναλύεται: {sym} ({idx+1}/{total})")
        time.sleep(0.02) # Επιταχύνθηκε ο χρόνος σάρωσης με ασφάλεια
    progress_bar.empty()

    if results:
        df_all = pd.DataFrame(results).sort_values(by=["Total Score"], ascending=False)
        cols_display = ["Μετοχή", "Σύμβολο", "Τιμή", "Πρόταση", "Total Score", "Βασικός Στόχος", "Upside %"]
        
        df_show = df_all[cols_display].copy()
        df_show["Total Score"] = df_show["Total Score"].apply(lambda x: round(x, 1))
        st.dataframe(df_show, use_container_width=True, hide_index=True)
        
        st.write("---")
        selected = st.selectbox("Επίλεξε τίτλο για έκδοση αναλυτικής έκθεσης:", df_all['Μετοχή'].tolist(), key=f"select_{unique_key}")
        if st.button("📊 Παραγωγή Έκθεσης", key=f"btn_{unique_key}"):
            render_quant_report(df_all[df_all['Μετοχή'] == selected].iloc[0])
    else:
        st.warning("Αδυναμία λήψης δεδομένων.")

# --- ΕΚΤΕΛΕΣΗ ΚΥΡΙΟΥ ΠΡΟΓΡΑΜΜΑΤΟΣ ---
if st.button("🔄 Ανανέωση Live Δεδομένων (Full Sync)", use_container_width=True, key="main_sync_btn"):
    st.cache_data.clear()
    st.rerun()

with st.spinner("Υπολογισμός παγκόσμιου ρίσκου..."):
    gscore, glabel, gdetails = global_regime()

st.subheader("🌍 Παγκόσμιο Οικονομικό Κλίμα")
st.info(f"**Κατάσταση Αγορών:** {glabel} (Global Score: {gscore:.1f}/100)")
with st.expander("Δες την κατάσταση των παγκόσμιων δεικτών", expanded=True):
    for det in gdetails: st.write(det)

st.write("---")

# ΔΗΜΙΟΥΡΓΙΑ ΚΑΡΤΕΛΩΝ (TABS)
tab1, tab2, tab3, tab4 = st.tabs(["🌎 Global Top 85+", "🇬🇷 Ελληνικές Μετοχές", "🔍 Custom Αναζήτηση", "📖 Λεξικό Όρων"])

with tab1:
    st.subheader("🌎 Παγκόσμιο Quant Universe (Top 85+ Stocks)")
    execute_screening(GLOBAL_UNIVERSE, is_global=True, unique_key="global")

with tab2:
    st.subheader("🇬🇷 Ελληνικό Χρηματιστήριο (ΧΑ)")
    execute_screening(GREEK_UNIVERSE, is_global=False, unique_key="greek")

with tab3:
    st.subheader("🔍 Custom Αναζήτηση & Ανάλυση Εκτός Λίστας")
    ticker_input = st.text_input("Πληκτρολόγησε το σύμβολο (π.χ. TSLA, MSTR, BTC-USD):", key="custom_search_input").upper().strip()
    if st.button("🚀 Ανάλυση Τώρα", key="custom_search_btn"):
        if ticker_input:
            with st.spinner(f"Λήψη δεδομένων για {ticker_input}..."):
                res = analyze_stock(ticker_input, "5y", gscore, glabel)
                if res:
                    st.success("Η ανάλυση ολοκληρώθηκε!")
                    render_quant_report(res)
                else:
                    st.error("Δεν βρέθηκαν επαρκή δεδομένα. Ελέγξτε το σύμβολο.")

with tab4:
    st.subheader("📖 Επεξηγηματικό Λεξικό Χρηματοοικονομικών Όρων")
    st.write("Κάνε κλικ σε κάθε κατηγορία για να δεις την απλή επεξήγηση χωρίς δύσκολους όρους.")
    
    # ΔΙΟΡΘΩΘΗΚΑΝ ΟΡΙΣΤΙΚΑ ΤΑ st.expander ΜΕ ΜΙΚΡΟ s
    with st.expander("📈 Όροι Τεχνικής Ανάλυσης (Διαγράμματα & Τάσεις)"):
        st.markdown("""
        * **ΚΜΟ 200 (Κινητός Μέσος Όρος 200 ημερών):** Η μέση τιμή της μετοχής τις τελευταίες 200 ημέρες. Δείχνει τη μακροπρόθεσμη κατεύθυνση.
        * **Golden Cross (Χρυσός Σταυρός):** Συμβαίνει όταν ο γρήγορος ΚΜΟ 50 ημερών διασταυρώνεται και περνάει πάνω από τον αργό ΚΜΟ 200 ημερών. Σήμα έναρξης μεγάλης ανόδου.
        * **RSI (Relative Strength Index):** Δείκτης ορμής που μετράει αν μια μετοχή έχει αγοραστεί υπερβολικά (>70) ή έχει πουληθεί υπερβολικά (<35).
        * **MACD:** Δείκτης που μετράει τη δύναμη και τη «φόρα» της τάσης.
        """)
        
    with st.expander("💰 Όροι Θεμελιώδους Ανάλυσης (Λογιστικά & Ισολογισμοί)"):
        st.markdown("""
        * **P/E Ratio (Price-to-Earnings):** Δείχνει πόσες φορές πληρώνει ο επενδυτής τα ετήσια κέρδη της εταιρείας. Χαμηλό σημαίνει φθηνή μετοχή.
        * **Forward P/E:** Βασίζεται στα κέρδη που αναμένεται να βγάλει η εταιρεία τον επόμενο χρόνο.
        * **ROE (Return on Equity):** Δείχνει πόσο αποτελεσματικά παράγει κέρδη η εταιρεία με τα λεφτά των μετόχων.
        * **PEG Ratio:** Δείχνει αν το P/E μιας μετοχής δικαιολογείται από τον ρυθμό ανάπτυξής της. Κοντά στο 1 είναι ιδανικό.
        """)
