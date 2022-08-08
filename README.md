# NH_project

# 데이터 만들기
from pykrx import stock
import pandas as pd
import mplfinance as mpf
import numpy as np
import plotly.graph_objects as go
import plotly.subplots as ms
import plotly.express as px

# 전체 종목코드와 종목명 가져오기
stock_list = pd.DataFrame({'종목코드':stock.get_market_ticker_list(market="ALL")})
stock_list['종목명'] = stock_list['종목코드'].map(lambda x: stock.get_market_ticker_name(x))

name_list = []
for _ in range(len(stock_list)):
    name_list.append(stock_list['종목명'][_])
    
dff = pd.DataFrame()

for name in name_list:
    stock_name = name
    stock_from = "20190102"
    stock_to = "20220803"
    
    ticker = stock_list.loc[stock_list['종목명']== stock_name, '종목코드']
    df = stock.get_market_ohlcv_by_date(fromdate=stock_from, todate=stock_to, ticker=ticker)
# 칼럼명을 영문명으로 변경
    df['name'] = name
    df = df.rename(columns={'시가':'Open', '고가':'High', '저가':'Low', '종가':'Close', '거래량':'Volume'})
    df["Close"]=df["Close"].apply(pd.to_numeric,errors="coerce")
    dff = pd.concat([dff,df])
    
df = dff
df

dname = df['2019-01-02']['name']
dname = list(dname.values)

dff = pd.DataFrame()
i=0
for name in dname:
    
    stock_name = name
    stock_from = "20190102"
    stock_to = "20220803"
    
    ticker = stock_list.loc[stock_list['종목명']== stock_name, '종목코드']
    df = stock.get_market_ohlcv_by_date(fromdate=stock_from, todate=stock_to, ticker=ticker)
    df = df.assign(종목명=stock_name)
# 칼럼명을 영문명으로 변경
    df = df.rename(columns={'시가':'Open', '고가':'High', '저가':'Low', '종가':'Close', '거래량':'Volume'})
    df["Close"]=df["Close"].apply(pd.to_numeric,errors="coerce")
    dff = pd.concat([dff,df])
    i += 1
    
df = dff

df['ma20'] = df['Close'].rolling(window=20).mean() # 20일 이동평균
df['stddev'] = df['Close'].rolling(window=20).std() # 20일 이동표준편차
df['upper'] = df['ma20'] + 2*df['stddev'] # 상단밴드
df['lower'] = df['ma20'] - 2*df['stddev'] # 하단밴드

df['ma12'] = df['Close'].rolling(window=12).mean() # 12일 이동평균
df['ma26'] = df['Close'].rolling(window=26).mean() # 26일 이동평균
df['MACD'] = df['ma12'] - df['ma26']    # MACD
df['MACD_Signal'] = df['MACD'].rolling(window=9).mean() # MACD Signal(MACD 9일 이동평균)
df['MACD_Oscil'] = df['MACD'] - df['MACD_Signal']   #MACD 오실레이터

df['ndays_high'] = df['High'].rolling(window=14, min_periods=1).max()    # 14일 중 최고가
df['ndays_low'] = df['Low'].rolling(window=14, min_periods=1).min()      # 14일 중 최저가
df['fast_k'] = (df['Close'] - df['ndays_low']) / (df['ndays_high'] - df['ndays_low']) * 100  # Fast %K 구하기
df['slow_d'] = df['fast_k'].rolling(window=3).mean()    # Slow %D 구하기

#RSI 구하기
U = np.where(df['Close'].diff(1) > 0, df['Close'].diff(1), 0)
D = np.where(df['Close'].diff(1) < 0, df['Close'].diff(1) *(-1), 0)
AU = pd.DataFrame(U, index=df.index).rolling(window=14).mean()
AD = pd.DataFrame(D, index=df.index).rolling(window=14).mean()
RSI = AU / (AD+AU) *100
df['RSI'] = RSI

ori_data = df

# cci 구하기
ori_data['M'] = 0.0
ori_data['m'] = 0.0
ori_data['D'] = 0.0
ori_data['CCI'] = 0.0

ori_data['M'] = (ori_data['High']+ori_data['Low']+ori_data['Close'])/3

ori_data['m'] = ori_data['M'].rolling(window=20).mean()
ori_data['D'] = (ori_data['M'] - ori_data['m']).rolling(window=20).mean()
ori_data['CCI'] = (ori_data['M']-ori_data['m'])/(0.015*ori_data['D'])

ori_data = ori_data.reset_index(drop=False, inplace = False)

a = [0]*38
b = [1]*849
c = (a+b)*2233
c = pd.DataFrame(c)

ori_data['bool'] = c
ori_data = ori_data[ori_data['bool'] == 1]

# 2019 ~ 2022 데이터 2233개 추출 
ori_data.to_csv("Data_2233.csv")

trade_data = pd.read_csv("Data_2233.csv")

# name_list는 volume = 0 인적 잇는것들 
name_list = list(trade_data[trade_data['Volume'] == 0]['종목명'].unique())

for n in name_list:
    trade_data = trade_data[trade_data['종목명'] != n ]
trade_data

# 데이터 1697개 최종 저장 
trade_data = trade_data.reset_index(drop=False, inplace = False)
trade_data.to_csv("Data_1697")

# 지표만들기 시작
Base_data = trade_data
Base_data['BOL_trade'] = 0 # 밴드의 상한 및 하한선 벗어날 때
Base_data['MACD_trade'] = 0 # 음->양 매수 양->음 매도 
Base_data['RSI_trade'] = 0 # 70이상 매도 , 30이하 매수 
Base_data['CCI_trade'] = 0 # -100 ->상향 매수, 100 -> 하향 매도 
Base_data['STO_trade'] = 0 # k가 d를 상향 돌파, 매수 / k가 d를 하향 돌파 매도 

# 볼린저밴드 upper, lower 
# macd = MACD_Oscil
# sto = fast_k, slow_d
# RSI
# CCI

Alpha = 0 
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['CCI'][i] < -1:
            Alpha = -1 
    if Alpha == -1:
        if Base_data['CCI'][i] > -1:
            Base_data['CCI_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:        
        if Base_data['CCI'][i] > 1:
            Alpha = 1
    if Alpha == 1:
        if Base_data['CCI'][i] < 1:
            Base_data['CCI_trade'][i] = 2
            Alpha = 0
 
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['Close'][i] < Base_data['lower'][i]: # 종가가 하한선보다 작다면 
            Alpha = -1 
    if Base_data['Close'][i] > Base_data['lower'][i]:
        if Alpha == -1:
            Base_data['BOL_trade'][i] = 1
            Alpha = 0 
            
    if Alpha == 0:
        if Base_data['Close'][i] > Base_data['upper'][i]: # 종가가 하한선보다 크면
            Alpha = 1
    if Base_data['Close'][i] < Base_data['upper'][i]:
        if Alpha == 1:
            Base_data['BOL_trade'][i] = 2
            Alpha = 0
            
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['MACD_Oscil'][i] < 0:
            Alpha = -1
    if Base_data['MACD_Oscil'][i] > 0:
        if Alpha == -1:
            Base_data['MACD_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:           
        if Base_data['MACD_Oscil'][i] > 0:
            Alpha = 1
    if Base_data['MACD_Oscil'][i] < 0:
        if Alpha == 1:
            Base_data['MACD_trade'][i] = 2
            Alpha = 0
            
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['RSI'][i] < 30:
            Alpha = -1
    if Base_data['RSI'][i] > 30:
        if Alpha == -1:
            Base_data['RSI_trade'][i] = 1
            Alpha = 0
    if Alpha == 0:           
        if Base_data['RSI'][i] > 70:
            Alpha = 1 
    if Base_data['RSI'][i] < 70:
        if Alpha == 1:
            Base_data['RSI_trade'][i] = 2
            Alpha = 0
    
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['fast_k'][i] < Base_data['slow_d'][i]:
            Alpha = -1
    if Alpha == -1:
        if Base_data['fast_k'][i] > Base_data['slow_d'][i]:
            Base_data['STO_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:
        if Base_data['fast_k'][i] > Base_data['slow_d'][i]:
            Alpha = 1
    if Alpha == 1:
        if Base_data['fast_k'][i] < Base_data['slow_d'][i]:
            Base_data['STO_trade'][i] = 2
            Alpha = 0


Base_data = trade_data
Base_data['BOL_trade'] = 0 # 밴드의 상한 및 하한선 벗어날 때
Base_data['MACD_trade'] = 0 # 음->양 매수 양->음 매도 
Base_data['RSI_trade'] = 0 # 70이상 매도 , 30이하 매수 
Base_data['CCI_trade'] = 0 # -100 ->상향 매수, 100 -> 하향 매도 
Base_data['STO_trade'] = 0 # k가 d를 상향 돌파, 매수 / k가 d를 하향 돌파 매도 

# 볼린저밴드 upper, lower 
# macd = MACD_Oscil
# sto = fast_k, slow_d
# RSI
# CCI

Alpha = 0 
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['CCI'][i] < -1:
            Alpha = -1 
    if Alpha == -1:
        if Base_data['CCI'][i] > -1:
            Base_data['CCI_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:        
        if Base_data['CCI'][i] > 1:
            Alpha = 1
    if Alpha == 1:
        if Base_data['CCI'][i] < 1:
            Base_data['CCI_trade'][i] = 2
            Alpha = 0
 
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['Close'][i] < Base_data['lower'][i]: # 종가가 하한선보다 작다면 
            Alpha = -1 
    if Base_data['Close'][i] > Base_data['lower'][i]:
        if Alpha == -1:
            Base_data['BOL_trade'][i] = 1
            Alpha = 0 
            
    if Alpha == 0:
        if Base_data['Close'][i] > Base_data['upper'][i]: # 종가가 하한선보다 크면
            Alpha = 1
    if Base_data['Close'][i] < Base_data['upper'][i]:
        if Alpha == 1:
            Base_data['BOL_trade'][i] = 2
            Alpha = 0
            
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['MACD_Oscil'][i] < 0:
            Alpha = -1
    if Base_data['MACD_Oscil'][i] > 0:
        if Alpha == -1:
            Base_data['MACD_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:           
        if Base_data['MACD_Oscil'][i] > 0:
            Alpha = 1
    if Base_data['MACD_Oscil'][i] < 0:
        if Alpha == 1:
            Base_data['MACD_trade'][i] = 2
            Alpha = 0
            
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['RSI'][i] < 30:
            Alpha = -1
    if Base_data['RSI'][i] > 30:
        if Alpha == -1:
            Base_data['RSI_trade'][i] = 1
            Alpha = 0
    if Alpha == 0:           
        if Base_data['RSI'][i] > 70:
            Alpha = 1 
    if Base_data['RSI'][i] < 70:
        if Alpha == 1:
            Base_data['RSI_trade'][i] = 2
            Alpha = 0
    
Alpha = 0
for i in range(len(Base_data)):
    if Alpha == 0:
        if Base_data['fast_k'][i] < Base_data['slow_d'][i]:
            Alpha = -1
    if Alpha == -1:
        if Base_data['fast_k'][i] > Base_data['slow_d'][i]:
            Base_data['STO_trade'][i] = 1
            Alpha = 0
            
    if Alpha == 0:
        if Base_data['fast_k'][i] > Base_data['slow_d'][i]:
            Alpha = 1
    if Alpha == 1:
        if Base_data['fast_k'][i] < Base_data['slow_d'][i]:
            Base_data['STO_trade'][i] = 2
            Alpha = 0

trade_data = Base_data

trade_data['BOL_profit']= 0
trade_data['MACD_profit']= 0
trade_data['RSI_profit']= 0
trade_data['CCI_profit']= 0
trade_data['STO_profit']= 0
trade_data['Ori']= 0

# 0-대기 1-매수 2-매도
def trade(data, asset, indicator):
    day = 0
    ori = 0
    count = 0 
    can_b = 0
    can_s = 0
    for _ in range(len(data)-1):
        if _%849 == 0:
            day = 0
            ori = 0
            count = 0 
            can_b = 0
            can_s = 0
        if _ != 0:
            if day%30 == 0:
                ori += 1
                day = 0 
                asset += 3000000
        if data[indicator + '_trade'][_] == 0:
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
        elif data[indicator + '_trade'][_] == 1:
            can_b = int(asset / data['Open'][_+1])
            asset -= can_b*data['Open'][_+1]
            count += can_b 
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_b = 0
        elif data[indicator + '_trade'][_] == 2:
            can_s = int(count/2)
            asset += can_s*data['Open'][_+1]
            count -= can_s
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_s = 0
        data['Ori'][_] = 30000000 + (ori)*3000000
        day += 1
    # print("원금 : ", 30000000 + (ori)*3000000)
    # print(ori)
    return data

trade_data = trade(trade_data, 30000000, "BOL")
trade_data = trade(trade_data, 30000000, "MACD")
trade_data = trade(trade_data, 30000000, "RSI")
trade_data = trade(trade_data, 30000000, "CCI")
trade_data = trade(trade_data, 30000000, "STO")

trade_data['BOL_profit']= 0
trade_data['MACD_profit']= 0
trade_data['RSI_profit']= 0
trade_data['CCI_profit']= 0
trade_data['STO_profit']= 0
trade_data['Ori']= 0

# 0-대기 1-매수 2-매도

def trade(data, asset, indicator):
    day = 0
    ori = 0
    count = 0 
    can_b = 0
    can_s = 0
    
    for _ in range(len(data)-1):
        
        if _ != 0:
            if day%30 == 0:
                ori += 1
                day = 0 
                asset += 3000000
            
        if data[indicator + '_trade'][_] == 0:
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])

        elif data[indicator + '_trade'][_] == 1:
            can_b = int(asset / data['Open'][_+1])
            asset -= can_b*data['Open'][_+1]
            count += can_b 
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_b = 0

        elif data[indicator + '_trade'][_] == 2:
            can_s = int(count/2)
            asset += can_s*data['Open'][_+1]
            count -= can_s
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_s = 0

        
        data['Ori'][_] = 30000000 + (ori)*3000000
        day += 1
        
        
    print("원금 : ", 30000000 + (ori)*3000000)
    print(ori)
    return data

# 0-대기 1-매수 2-매도

def trade(data, asset, indicator):
    day = 0
    ori = 0
    count = 0 
    can_b = 0
    can_s = 0
    
    for _ in range(len(data)-1):
        
        if _ != 0:
            if day%30 == 0:
                ori += 1
                day = 0 
                asset += 3000000
            
        if data[indicator + '_trade'][_] == 0:
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])

        elif data[indicator + '_trade'][_] == 1:
            can_b = int(asset / data['Open'][_+1])
            asset -= can_b*data['Open'][_+1]
            count += can_b 
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_b = 0

        elif data[indicator + '_trade'][_] == 2:
            can_s = int(count/2)
            asset += can_s*data['Open'][_+1]
            count -= can_s
            data[indicator + '_profit'][_] = asset + (count*data['Close'][_])
            can_s = 0

        
        data['Ori'][_] = 30000000 + (ori)*3000000
        day += 1
        
        
    print("원금 : ", 30000000 + (ori)*3000000)
    print(ori)
    return data
