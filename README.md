# app.py - SWING TRADING PRO ANALYZER
# Ottimizzato per Streamlit Cloud e mobile iOS/Android

import streamlit as st
import pandas as pd
import numpy as np
import yfinance as yf
from datetime import datetime, timedelta
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import json
import base64
import hashlib
from io import BytesIO
import requests
from PIL import Image
import pytz

# Configurazione pagina
st.set_page_config(
    page_title="Swing Trading Pro",
    page_icon="📈",
    layout="wide",
    initial_sidebar_state="expanded"
)

# CSS Custom avanzato
st.markdown("""
<style>
    @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
    
    * { font-family: 'Inter', sans-serif; }
    
    .main {
        background: linear-gradient(135deg, #0f0c29 0%, #302b63 50%, #24243e 100%);
        color: #ffffff;
    }
    
    .stApp {
        background: linear-gradient(135deg, #0f0c29 0%, #302b63 50%, #24243e 100%);
    }
    
    .metric-card {
        background: rgba(255,255,255,0.05);
        border-radius: 16px;
        padding: 20px;
        border: 1px solid rgba(255,255,255,0.1);
        backdrop-filter: blur(10px);
    }
    
    .signal-buy {
        background: linear-gradient(135deg, #00b894 0%, #00cec9 100%);
        color: white;
        padding: 15px;
        border-radius: 12px;
        text-align: center;
        font-weight: 700;
        font-size: 18px;
        margin: 10px 0;
    }
    
    .signal-neutral {
        background: linear-gradient(135deg, #fdcb6e 0%, #e17055 100%);
        color: white;
        padding: 15px;
        border-radius: 12px;
        text-align: center;
        font-weight: 700;
        font-size: 18px;
        margin: 10px 0;
    }
    
    .signal-avoid {
        background: linear-gradient(135deg, #d63031 0%, #e84393 100%);
        color: white;
        padding: 15px;
        border-radius: 12px;
        text-align: center;
        font-weight: 700;
        font-size: 18px;
        margin: 10px 0;
    }
    
    .info-box {
        background: rgba(116, 185, 255, 0.1);
        border-left: 4px solid #74b9ff;
        padding: 15px;
        border-radius: 8px;
        margin: 10px 0;
    }
    
    .warning-box {
        background: rgba(253, 203, 110, 0.1);
        border-left: 4px solid #fdcb6e;
        padding: 15px;
        border-radius: 8px;
        margin: 10px 0;
    }
    
    .danger-box {
        background: rgba(214, 48, 49, 0.1);
        border-left: 4px solid #d63031;
        padding: 15px;
        border-radius: 8px;
        margin: 10px 0;
    }
    
    .stButton>button {
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        color: white;
        border: none;
        border-radius: 12px;
        padding: 15px 30px;
        font-weight: 600;
        transition: all 0.3s;
    }
    
    .stButton>button:hover {
        transform: translateY(-2px);
        box-shadow: 0 10px 20px rgba(102, 126, 234, 0.3);
    }
    
    .trade-log {
        background: rgba(0,0,0,0.2);
        border-radius: 12px;
        padding: 15px;
        margin: 10px 0;
    }
    
    div[data-testid="stMetricValue"] {
        font-size: 28px;
        font-weight: 700;
        color: #74b9ff;
    }
    
    .stTabs [data-baseweb="tab-list"] {
        gap: 8px;
    }
    
    .stTabs [data-baseweb="tab"] {
        background: rgba(255,255,255,0.05);
        border-radius: 8px 8px 0 0;
        padding: 12px 24px;
        color: #a0a0a0;
    }
    
    .stTabs [aria-selected="true"] {
        background: rgba(116, 185, 255, 0.2) !important;
        color: #74b9ff !important;
    }
    
    div.stSpinner > div {
        border-top-color: #74b9ff;
    }
    
    /* Mobile optimization */
    @media (max-width: 768px) {
        .main .block-container {
            padding: 1rem;
        }
        h1 { font-size: 24px !important; }
        .metric-card { padding: 12px; }
    }
</style>
""", unsafe_allow_html=True)

# Inizializzazione session state
def init_session():
    defaults = {
        'portfolio': [],
        'trade_history': [],
        'watchlist': ['AAPL', 'MSFT', 'TSLA', 'NVDA', 'META'],
        'settings': {'risk_per_trade': 1.0, 'max_positions': 5, 'account_size': 50000},
        'last_analysis': None
    }
    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value

init_session()

# Utility functions
def get_stock_data(ticker, period="6mo"):
    """Scarica dati da Yahoo Finance con gestione errori"""
    try:
        stock = yf.Ticker(ticker)
        df = stock.history(period=period)
        if df.empty:
            return None, "Dati non disponibili"
        info = stock.info
        return df, info
    except Exception as e:
        return None, str(e)

def calculate_indicators(df):
    """Calcola indicatori tecnici avanzati"""
    # Medie mobili
    df['EMA20'] = df['Close'].ewm(span=20, adjust=False).mean()
    df['EMA50'] = df['Close'].ewm(span=50, adjust=False).mean()
    df['SMA200'] = df['Close'].rolling(window=200).mean()
    
    # RSI
    delta = df['Close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=14).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean()
    rs = gain / loss
    df['RSI'] = 100 - (100 / (1 + rs))
    
    # MACD
    exp1 = df['Close'].ewm(span=12, adjust=False).mean()
    exp2 = df['Close'].ewm(span=26, adjust=False).mean()
    df['MACD'] = exp1 - exp2
    df['Signal'] = df['MACD'].ewm(span=9, adjust=False).mean()
    df['Histogram'] = df['MACD'] - df['Signal']
    
    # ATR (Average True Range)
    high_low = df['High'] - df['Low']
    high_close = np.abs(df['High'] - df['Close'].shift())
    low_close = np.abs(df['Low'] - df['Close'].shift())
    ranges = pd.concat([high_low, high_close, low_close], axis=1)
    true_range = np.max(ranges, axis=1)
    df['ATR'] = true_range.rolling(14).mean()
    
    # Bande di Bollinger
    df['BB_Middle'] = df['Close'].rolling(window=20).mean()
    bb_std = df['Close'].rolling(window=20).std()
    df['BB_Upper'] = df['BB_Middle'] + (bb_std * 2)
    df['BB_Lower'] = df['BB_Middle'] - (bb_std * 2)
    
    # Volume media
    df['Vol_MA20'] = df['Volume'].rolling(window=20).mean()
    
    return df

def analyze_setup(df, info, entry_price=None, manual_stop=None):
    """Analisi completa del setup di swing trading"""
    if df is None or len(df) < 50:
        return None
    
    current = df.iloc[-1]
    prev = df.iloc[-2]
    
    analysis = {
        'trend_daily': 'Bullish' if current['Close'] > current['EMA50'] else 'Bearish',
        'trend_weekly': 'Bullish' if current['Close'] > current['SMA200'] else 'Bearish',
        'rsi': current['RSI'],
        'rsi_condition': 'Oversold' if current['RSI'] < 30 else 'Overbought' if current['RSI'] > 70 else 'Neutral',
        'near_ema20': abs(current['Close'] - current['EMA20']) / current['Close'] < 0.02,
        'macd_bullish': current['MACD'] > current['Signal'],
        'volume_spike': current['Volume'] > current['Vol_MA20'] * 1.5,
        'bb_position': 'Lower' if current['Close'] < current['BB_Lower'] else 'Upper' if current['Close'] > current['BB_Upper'] else 'Middle',
        'atr': current['ATR'],
        'current_price': current['Close'],
        'recommendation': 'NEUTRAL'
    }
    
    # Logica strategia Pullback su Trend
    score = 0
    checks = []
    
    # 1. Trend settimanale rialzista (25 punti)
    if analysis['trend_weekly'] == 'Bullish':
        score += 25
        checks.append("✅ Trend settimanale rialzista")
    else:
        checks.append("❌ Trend settimanale ribassista - EVITARE")
        analysis['recommendation'] = 'AVOID'
        analysis['score'] = score
        analysis['checks'] = checks
        return analysis
    
    # 2. Prezzo vicino a EMA20 (25 punti)
    if analysis['near_ema20']:
        score += 25
        checks.append("✅ Prezzo su EMA20 (zona valore)")
    else:
        distance = abs(current['Close'] - current['EMA20']) / current['EMA20'] * 100
        checks.append(f"⚠️ Distanza da EMA20: {distance:.1f}%")
    
    # 3. RSI in zona ottimale (20 punti)
    if 30 <= analysis['rsi'] <= 50:
        score += 20
        checks.append(f"✅ RSI ottimale ({analysis['rsi']:.1f})")
    elif analysis['rsi'] < 30:
        score += 15
        checks.append(f"⚠️ RSI ipervenduto ({analysis['rsi']:.1f}) - possibile rimbalzo")
    else:
        checks.append(f"❌ RSI troppo alto ({analysis['rsi']:.1f}) - attesa")
    
    # 4. MACD bullish (15 punti)
    if analysis['macd_bullish']:
        score += 15
        checks.append("✅ MACD bullish")
    else:
        checks.append("⚠️ MACD non ancora bullish")
    
    # 5. Volume (15 punti)
    if not analysis['volume_spike']:
        score += 15
        checks.append("✅ Volume normale (no spike sospetto)")
    else:
        checks.append("⚠️ Volume anomalo - verificare notizie")
    
    # Calcolo entry/stop ottimale se non forniti
    if entry_price is None:
        analysis['suggested_entry'] = current['Close']
        analysis['suggested_stop'] = current['EMA20'] - (current['ATR'] * 0.5)
    else:
        analysis['suggested_entry'] = entry_price
        if manual_stop:
            analysis['suggested_stop'] = manual_stop
        else:
            analysis['suggested_stop'] = entry_price - (current['ATR'] * 1.5)
    
    # Calcolo target
    risk = analysis['suggested_entry'] - analysis['suggested_stop']
    analysis['target_2r'] = analysis['suggested_entry'] + (risk * 2)
    analysis['target_3r'] = analysis['suggested_entry'] + (risk * 3)
    analysis['risk_reward'] = 2.0 if risk > 0 else 0
    
    # Determinazione finale
    if score >= 75:
        analysis['recommendation'] = 'STRONG_BUY'
    elif score >= 60:
        analysis['recommendation'] = 'BUY'
    elif score >= 40:
        analysis['recommendation'] = 'WATCH'
    else:
        analysis['recommendation'] = 'AVOID'
    
    analysis['score'] = score
    analysis['checks'] = checks
    
    return analysis

def position_size_calc(account, risk_pct, entry, stop):
    """Calcolo preciso position sizing"""
    if entry <= stop or risk_pct <= 0:
        return 0, 0, 0
    
    risk_amount = account * (risk_pct / 100)
    risk_per_share = entry - stop
    shares = int(risk_amount / risk_per_share)
    
    total_exposure = shares * entry
    position_pct = (total_exposure / account) * 100 if account > 0 else 0
    
    return shares, total_exposure, position_pct

def create_chart(df, ticker, analysis):
    """Crea grafico professionale con Plotly"""
    fig = make_subplots(
        rows=3, cols=1,
        shared_xaxes=True,
        vertical_spacing=0.05,
        row_heights=[0.6, 0.2, 0.2],
        subplot_titles=(f'{ticker} - Price Action', 'MACD', 'RSI')
    )
    
    # Candlestick
    fig.add_trace(go.Candlestick(
        x=df.index,
        open=df['Open'],
        high=df['High'],
        low=df['Low'],
        close=df['Close'],
        name='OHLC',
        increasing_line_color='#00b894',
        decreasing_line_color='#d63031'
    ), row=1, col=1)
    
    # Medie mobili
    fig.add_trace(go.Scatter(
        x=df.index, y=df['EMA20'],
        line=dict(color='#74b9ff', width=2),
        name='EMA20'
    ), row=1, col=1)
    
    fig.add_trace(go.Scatter(
        x=df.index, y=df['EMA50'],
        line=dict(color='#fdcb6e', width=2),
        name='EMA50'
    ), row=1, col=1)
    
    fig.add_trace(go.Scatter(
        x=df.index, y=df['SMA200'],
        line=dict(color='#e84393', width=2, dash='dash'),
        name='SMA200'
    ), row=1, col=1)
    
    # Bande Bollinger
    fig.add_trace(go.Scatter(
        x=df.index, y=df['BB_Upper'],
        line=dict(color='rgba(255,255,255,0.2)', width=1),
        name='BB Upper',
        showlegend=False
    ), row=1, col=1)
    
    fig.add_trace(go.Scatter(
        x=df.index, y=df['BB_Lower'],
        line=dict(color='rgba(255,255,255,0.2)', width=1),
        fill='tonexty',
        fillcolor='rgba(255,255,255,0.05)',
        name='BB Lower',
        showlegend=False
    ), row=1, col=1)
    
    # Entry/Stop/Target se analisi presente
    if analysis and 'suggested_entry' in analysis:
        fig.add_hline(y=analysis['suggested_entry'], line_dash="dash", 
                     line_color="#00b894", annotation_text="Entry", row=1, col=1)
        fig.add_hline(y=analysis['suggested_stop'], line_dash="dash", 
                     line_color="#d63031", annotation_text="Stop", row=1, col=1)
        fig.add_hline(y=analysis['target_2r'], line_dash="dot", 
                     line_color="#fdcb6e", annotation_text="Target 2R", row=1, col=1)
    
    # MACD
    colors_macd = ['#00b894' if val >= 0 else '#d63031' for val in df['Histogram']]
    fig.add_trace(go.Bar(
        x=df.index, y=df['Histogram'],
        marker_color=colors_macd,
        name='MACD Hist'
    ), row=2, col=1)
    
    fig.add_trace(go.Scatter(
        x=df.index, y=df['MACD'],
        line=dict(color='#74b9ff', width=2),
        name='MACD'
    ), row=2, col=1)
    
    fig.add_trace(go.Scatter(
        x=df.index, y=df['Signal'],
        line=dict(color='#fdcb6e', width=2),
        name='Signal'
    ), row=2, col=1)
    
    # RSI
    fig.add_trace(go.Scatter(
        x=df.index, y=df['RSI'],
        line=dict(color='#a29bfe', width=2),
        name='RSI'
    ), row=3, col=1)
    
    fig.add_hline(y=70, line_dash="dash", line_color="#d63031", row=3, col=1)
    fig.add_hline(y=30, line_dash="dash", line_color="#00b894", row=3, col=1)
    
    # Layout
    fig.update_layout(
        template='plotly_dark',
        paper_bgcolor='rgba(0,0,0,0)',
        plot_bgcolor='rgba(0,0,0,0.2)',
        height=800,
        showlegend=True,
        xaxis_rangeslider_visible=False
    )
    
    fig.update_yaxes(gridcolor='rgba(255,255,255,0.1)', zerolinecolor='rgba(255,255,255,0.2)')
    
    return fig

def save_trade_to_history(trade_data):
    """Salva trade nella history di sessione"""
    trade_data['timestamp'] = datetime.now().isoformat()
    trade_data['id'] = hashlib.md5(trade_data['timestamp'].encode()).hexdigest()[:8]
    st.session_state.trade_history.append(trade_data)
    return trade_data['id']

def export_trades():
    """Esporta trades come CSV"""
    if not st.session_state.trade_history:
        return None
    
    df = pd.DataFrame(st.session_state.trade_history)
    csv = df.to_csv(index=False)
    b64 = base64.b64encode(csv.encode()).decode()
    return f"data:file/csv;base64,{b64}"

# UI Principale
def main():
    # Header
    col1, col2 = st.columns([3, 1])
    with col1:
        st.title("📈 Swing Trading Pro Analyzer")
        st.caption("Analisi tecnica avanzata | Gestione rischio | Tracking portfolio")
    with col2:
        st.metric("Account", f"€{st.session_state.settings['account_size']:,.0f}", 
                 f"{st.session_state.settings['risk_per_trade']}% risk/trade")
    
    # Tabs principali
    tab1, tab2, tab3, tab4 = st.tabs([
        "🔍 Analisi Setup", 
        "📊 Screener Mercato", 
        "💼 Portfolio Tracking", 
        "⚙️ Impostazioni"
    ])
    
    # TAB 1: Analisi Setup
    with tab1:
        st.markdown("### Strategia: Pullback su Trend Rialzista")
        
        col_input1, col_input2 = st.columns([1, 1])
        
        with col_input1:
            ticker = st.text_input("Ticker Simbolo", "AAPL", 
                                  help="Inserisci il simbolo Yahoo Finance (es. AAPL, MSFT, TSLA)").upper().strip()
            
            col_manual = st.columns(2)
            with col_manual[0]:
                use_manual = st.checkbox("Input manuale", False, 
                                        help="Disabilita download dati per analisi offline")
            with col_manual[1]:
                if use_manual:
                    manual_price = st.number_input("Prezzo Corrente", min_value=0.01, value=150.0, step=0.01)
            
            if not use_manual:
                # Scarica dati automatici
                with st.spinner(f"Caricamento dati {ticker}..."):
                    df, info = get_stock_data(ticker)
                    if df is None:
                        st.error(f"Impossibile caricare dati per {ticker}")
                        st.stop()
                    df = calculate_indicators(df)
                    current_price = df['Close'].iloc[-1]
            else:
                current_price = manual_price
                df = None
                info = None
            
            # Input operativi
            entry_col, stop_col = st.columns(2)
            with entry_col:
                entry_price = st.number_input("Prezzo Entry (€)", 
                                            min_value=0.01, 
                                            value=round(current_price, 2),
                                            step=0.01,
                                            help="Prezzo dove intendi entrare")
            with stop_col:
                if df is not None:
                    suggested_stop = df['EMA20'].iloc[-1] - (df['ATR'].iloc[-1] * 0.5)
                else:
                    suggested_stop = entry_price * 0.97
                
                stop_loss = st.number_input("Stop Loss (€)", 
                                          min_value=0.01, 
                                          value=round(suggested_stop, 2),
                                          step=0.01,
                                          help="Stop tecnico sotto EMA20 o supporto")
            
            # Calcolo position size
            account_size = st.session_state.settings['account_size']
            risk_pct = st.session_state.settings['risk_per_trade']
            
            shares, exposure, pos_pct = position_size_calc(account_size, risk_pct, entry_price, stop_loss)
            
            st.markdown("---")
            st.markdown("#### 📊 Position Sizing")
            
            metrics = st.columns(3)
            metrics[0].metric("Azioni", f"{shares:,}")
            metrics[1].metric("Esposizione", f"€{exposure:,.0f}")
            metrics[2].metric("% Portfoglio", f"{pos_pct:.1f}%")
            
            # Risk analysis
            risk_amount = account_size * (risk_pct / 100)
            actual_risk = shares * (entry_price - stop_loss)
            
            if actual_risk > risk_amount * 1.1:
                st.warning(f"⚠️ Risk effettivo (€{actual_risk:.0f}) supera il limite impostato!")
        
        with col_input2:
            if df is not None:
                # Analisi automatica
                analysis = analyze_setup(df, info, entry_price, stop_loss)
                
                # Signal box
                if analysis['recommendation'] == 'STRONG_BUY':
                    st.markdown('<div class="signal-buy">🚀 STRONG BUY - Setup Ottimale</div>', 
                               unsafe_allow_html=True)
                elif analysis['recommendation'] == 'BUY':
                    st.markdown('<div class="signal-buy">✅ BUY - Setup Valido</div>', 
                               unsafe_allow_html=True)
                elif analysis['recommendation'] == 'WATCH':
                    st.markdown('<div class="signal-neutral">👁️ WATCH - Attendere conferme</div>', 
                               unsafe_allow_html=True)
                else:
                    st.markdown('<div class="signal-avoid">❌ AVOID - Setup non valido</div>', 
                               unsafe_allow_html=True)
                
                # Score
                st.progress(analysis['score'] / 100, text=f"Score Strategia: {analysis['score']}/100")
                
                # Metriche tecniche
                st.markdown("#### 📈 Indicatori Tecnici")
                tech_cols = st.columns(2)
                tech_cols[0].metric("RSI", f"{analysis['rsi']:.1f}", 
                                   delta="Oversold" if analysis['rsi'] < 30 else "Overbought" if analysis['rsi'] > 70 else "OK")
                tech_cols[1].metric("ATR", f"€{analysis['atr']:.2f}", "Volatilità 14gg")
                
                tech_cols[0].metric("Trend Weekly", analysis['trend_weekly'])
                tech_cols[1].metric("Near EMA20", "Sì" if analysis['near_ema20'] else "No")
                
                # Checklist
                st.markdown("#### ✅ Checklist Verifica")
                for check in analysis['checks']:
                    st.write(check)
                
                # Target
                st.markdown("#### 🎯 Target Operativi")
                target_cols = st.columns(2)
                target_cols[0].metric("Target 2R", f"€{analysis['target_2r']:.2f}", 
                                   f"+{((analysis['target_2r']/entry_price-1)*100):.1f}%")
                target_cols[1].metric("Target 3R", f"€{analysis['target_3r']:.2f}",
                                   f"+{((analysis['target_3r']/entry_price-1)*100):.1f}%")
                
                # Salva analisi in sessione
                st.session_state.last_analysis = {
                    'ticker': ticker,
                    'entry': entry_price,
                    'stop': stop_loss,
                    'shares': shares,
                    'analysis': analysis,
                    'timestamp': datetime.now()
                }
        
        # Grafico
        if df is not None:
            st.markdown("---")
            fig = create_chart(df, ticker, analysis if 'analysis' in locals() else None)
            st.plotly_chart(fig, use_container_width=True)
            
            # Pulsanti azione
            st.markdown("---")
            col_btn1, col_btn2, col_btn3 = st.columns(3)
            
            with col_btn1:
                if st.button("💾 Salva in Watchlist", use_container_width=True):
                    if ticker not in st.session_state.watchlist:
                        st.session_state.watchlist.append(ticker)
                    st.success(f"{ticker} aggiunto a watchlist!")
            
            with col_btn2:
                if st.button("📝 Registra Trade", use_container_width=True):
                    if 'last_analysis' in st.session_state and st.session_state.last_analysis:
                        trade = st.session_state.last_analysis
                        trade_id = save_trade_to_history({
                            'ticker': trade['ticker'],
                            'direction': 'LONG',
                            'entry': trade['entry'],
                            'stop': trade['stop'],
                            'shares': trade['shares'],
                            'risk': st.session_state.settings['risk_per_trade'],
                            'status': 'OPEN',
                            'score': trade['analysis']['score']
                        })
                        st.success(f"Trade registrato! ID: {trade_id}")
            
            with col_btn3:
                # Export report
                if st.session_state.last_analysis:
                    report_text = f"""
SWING TRADING REPORT
Ticker: {st.session_state.last_analysis['ticker']}
Entry: {st.session_state.last_analysis['entry']}
Stop: {st.session_state.last_analysis['stop']}
Shares: {st.session_state.last_analysis['shares']}
Score: {st.session_state.last_analysis['analysis']['score']}/100
Date: {datetime.now().strftime('%Y-%m-%d %H:%M')}
                    """
                    b64 = base64.b64encode(report_text.encode()).decode()
                    href = f'<a href="data:file/txt;base64,{b64}" download="report_{ticker}_{datetime.now().strftime("%Y%m%d")}.txt">📥 Download Report</a>'
                    st.markdown(href, unsafe_allow_html=True)
    
    # TAB 2: Screener
    with tab2:
        st.markdown("### 🔍 Screener Swing Trading")
        st.info("Analisi automatica della watchlist per setup validi")
        
        if st.button("🔄 Avvia Scansione Watchlist", use_container_width=True):
            progress_bar = st.progress(0)
            results = []
            
            for idx, symbol in enumerate(st.session_state.watchlist):
                progress_bar.progress((idx + 1) / len(st.session_state.watchlist))
                
                df_sym, info_sym = get_stock_data(symbol, period="3mo")
                if df_sym is not None:
                    df_sym = calculate_indicators(df_sym)
                    analysis_sym = analyze_setup(df_sym, info_sym)
                    
                    if analysis_sym and analysis_sym['score'] >= 60:
                        results.append({
                            'Ticker': symbol,
                            'Price': f"€{analysis_sym['current_price']:.2f}",
                            'Score': analysis_sym['score'],
                            'RSI': f"{analysis_sym['rsi']:.1f}",
                            'Signal': analysis_sym['recommendation'],
                            'Near EMA20': 'Yes' if analysis_sym['near_ema20'] else 'No'
                        })
            
            if results:
                df_results = pd.DataFrame(results)
                st.dataframe(df_results, use_container_width=True, hide_index=True)
                
                # Evidenzia migliori
                best = max(results, key=lambda x: x['Score'])
                st.success(f"🏆 Miglior setup: {best['Ticker']} con score {best['Score']}/100")
            else:
                st.warning("Nessun setup valido trovato nella watchlist. Attendere pullback migliori.")
    
    # TAB 3: Portfolio
    with tab3:
        st.markdown("### 💼 Gestione Portfolio")
        
        col_port1, col_port2 = st.columns([2, 1])
        
        with col_port1:
            if st.session_state.trade_history:
                df_trades = pd.DataFrame(st.session_state.trade_history)
                
                # Filtro status
                status_filter = st.selectbox("Filtra", ["Tutti", "Aperti", "Chiusi"], index=0)
                
                if status_filter == "Aperti":
                    df_display = df_trades[df_trades['status'] == 'OPEN']
                elif status_filter == "Chiusi":
                    df_display = df_trades[df_trades['status'] == 'CLOSED']
                else:
                    df_display = df_trades
                
                st.dataframe(df_display, use_container_width=True, hide_index=True)
                
                # Statistiche
                if len(df_trades) > 0:
                    st.markdown("#### 📊 Statistiche Trading")
                    stat_cols = st.columns(4)
                    stat_cols[0].metric("Totali Trade", len(df_trades))
                    stat_cols[1].metric("Aperti", len(df_trades[df_trades['status'] == 'OPEN']))
                    stat_cols[2].metric("Avg Score", f"{df_trades['score'].mean():.1f}")
                    stat_cols[3].metric("Risk Medio", f"{df_trades['risk'].mean():.2f}%")
            else:
                st.info("Nessun trade registrato. Usa il Tab Analisi per registrare operazioni.")
        
        with col_port2:
            st.markdown("#### ⚡ Azioni Rapide")
            
            # Chiudi trade
            if st.session_state.trade_history:
                open_trades = [t for t in st.session_state.trade_history if t['status'] == 'OPEN']
                if open_trades:
                    trade_to_close = st.selectbox(
                        "Seleziona trade da chiudere",
                        options=[f"{t['id']} - {t['ticker']}" for t in open_trades]
                    )
                    exit_price = st.number_input("Prezzo Uscita", min_value=0.01, step=0.01)
                    
                    if st.button("🔒 Chiudi Trade", use_container_width=True):
                        trade_id = trade_to_close.split(' - ')[0]
                        for t in st.session_state.trade_history:
                            if t['id'] == trade_id:
                                t['status'] = 'CLOSED'
                                t['exit_price'] = exit_price
                                t['exit_date'] = datetime.now().isoformat()
                                pnl = (exit_price - t['entry']) * t['shares']
                                t['pnl'] = pnl
                                st.rerun()
            
            # Export
            st.markdown("---")
            if st.session_state.trade_history:
                csv_link = export_trades()
                if csv_link:
                    st.markdown(f'<a href="{csv_link}" download="trading_history.csv">📥 Export CSV</a>', 
                               unsafe_allow_html=True)
            
            # Reset
            if st.button("🗑️ Cancella Tutti i Dati", use_container_width=True):
                st.session_state.trade_history = []
                st.session_state.portfolio = []
                st.rerun()
    
    # TAB 4: Impostazioni
    with tab4:
        st.markdown("### ⚙️ Configurazione Account")
        
        with st.form("settings_form"):
            new_capital = st.number_input(
                "Capitale Account (€)", 
                min_value=1000.0, 
                value=float(st.session_state.settings['account_size']),
                step=1000.0
            )
            
            new_risk = st.slider(
                "Risk per Trade (%)", 
                min_value=0.1, 
                max_value=5.0, 
                value=float(st.session_state.settings['risk_per_trade']),
                step=0.1,
                help="Percentuale di capitale rischiata per singola operazione"
            )
            
            new_max_pos = st.number_input(
                "Max Posizioni Aperte", 
                min_value=1, 
                max_value=20,
                value=int(st.session_state.settings['max_positions']),
                step=1
            )
            
            # Watchlist editor
            st.markdown("#### 📝 Watchlist")
            watchlist_text = st.text_area(
                "Simboli (uno per riga)",
                value="\n".join(st.session_state.watchlist),
                height=150
            )
            
            submitted = st.form_submit_button("💾 Salva Impostazioni", use_container_width=True)
            
            if submitted:
                st.session_state.settings['account_size'] = new_capital
                st.session_state.settings['risk_per_trade'] = new_risk
                st.session_state.settings['max_positions'] = new_max_pos
                st.session_state.watchlist = [s.strip().upper() for s in watchlist_text.split('\n') if s.strip()]
                st.success("Impostazioni salvate!")
                st.rerun()
        
        # Info
        st.markdown("---")
        st.markdown("""
        <div class="info-box">
        <b>ℹ️ Informazioni Strategia</b><br>
        Questa app implementa la strategia <b>Pullback al Valore nel Trend</b>:<br>
        1. Trend settimanale rialzista (SMA200)<br>
        2. Pullback su EMA20 giornaliera<br>
        3. RSI tra 30-50 (non ipercomprato)<br>
        4. Entry su pattern candlestick di riferimento<br>
        5. Stop sotto EMA20 o minimo pullback<br>
        6. Target 2:1 e 3:1 Risk/Reward
        </div>
        """, unsafe_allow_html=True)
        
        st.markdown("""
        <div class="danger-box">
        <b>⚠️ Disclaimer Legale</b><br>
        Questo software è fornito ESCLUSIVAMENTE a scopo educativo e di analisi. 
        Non costituisce consulenza finanziaria. Il trading comporta rischi sostanziali 
        di perdita. L'autore non è responsabile di decisioni di investimento prese 
        utilizzando questo tool. Utilizzare sempre conto demo prima di operare 
        con capitale reale.
        </div>
        """, unsafe_allow_html=True)

if __name__ == "__main__":
    main()

