import streamlit as st
import yfinance as yf
import pandas as pd
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# 1. Page Setup
st.set_page_config(layout="wide", page_title="VCP Live Charts")
st.title("🇮🇳 Indian Stocks: VCP Tightness & Momentum")
st.write("Automatically showing charts for stocks meeting your momentum and VCP criteria.")

# 2. Define the Stocks (Add your preferred NSE symbols here)
symbols = [
    "TRENT.NS", "HAL.NS", "ZOMATO.NS", "BEL.NS", "MAZDOCK.NS", 
    "RVNL.NS", "DIXON.NS", "BSE.NS", "MCX.NS", "ADANIPORTS.NS",
    "HUDCO.NS", "IRFC.NS", "JWL.NS", "TATAELXSI.NS"
]

def check_vcp_and_momentum(df):
    """Checks the user's specific technical criteria."""
    curr_price = df['Close'].iloc[-1]
    high_52w = df['High'].max()
    
    # Logic 1: Within 25% of 52-week high
    near_high = curr_price >= (high_52w * 0.75)
    
    # Logic 2: 30% move in 3 months & 20% move in 1 month
    ret_3m = (curr_price / df['Close'].iloc[-63]) - 1 if len(df) > 63 else 0
    ret_1m = (curr_price / df['Close'].iloc[-21]) - 1 if len(df) > 21 else 0
    
    # Logic 3: VCP Tightness (Volatility contraction proxy)
    # Checks if the standard deviation of the last 5 days is less than the last 20 days
    is_tight = df['Close'].tail(5).std() < df['Close'].tail(20).std()
    
    return near_high and ret_3m >= 0.30 and ret_1m >= 0.20 and is_tight, ret_1m, ret_3m

# 3. Main Loop: Fetch and Display
for ticker in symbols:
    try:
        data = yf.Ticker(ticker)
        df = data.history(period="1y")
        
        if len(df) < 100: continue
        
        is_match, r1, r3 = check_vcp_and_momentum(df)
        
        if is_match:
            # Create a header for the stock
            st.subheader(f"✅ {ticker} | 1-Month: {r1:.1%} | 3-Month: {r3:.1%}")
            
            # Create Subplots: Top for Price, Bottom for Volume
            fig = make_subplots(rows=2, cols=1, shared_xaxes=True, 
                               vertical_spacing=0.05, row_width=[0.3, 0.7])

            # Candlestick Trace
            fig.add_trace(go.Candlestick(
                x=df.index[-100:], open=df['Open'][-100:], 
                high=df['High'][-100:], low=df['Low'][-100:], 
                close=df['Close'][-100:], name="Price"
            ), row=1, col=1)

            # Add Moving Averages
            df['MA50'] = df['Close'].rolling(50).mean()
            fig.add_trace(go.Scatter(x=df.index[-100:], y=df['MA50'][-100:], 
                                     line=dict(color='orange', width=2), name="50 DMA"), row=1, col=1)

            # Volume Bars
            colors = ['green' if df['Close'].iloc[i] >= df['Open'].iloc[i] else 'red' for i in range(len(df)-100, len(df))]
            fig.add_trace(go.Bar(x=df.index[-100:], y=df['Volume'][-100:], 
                                 marker_color=colors, name="Volume"), row=2, col=1)

            # Layout Styling
            fig.update_layout(height=600, xaxis_rangeslider_visible=False, template="plotly_dark")
            st.plotly_chart(fig, use_container_width=True)
            st.divider() # Visual separation between stocks
            
    except Exception as e:
        continue
