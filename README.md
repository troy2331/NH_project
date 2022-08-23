from pykrx import stock
import pandas as pd
import mplfinance as mpf
import numpy as np
import plotly.graph_objects as go
import plotly.subplots as ms
import plotly.express as px

# 승률 출력 

data = pd.read_csv("하락_전체_올인.csv")
data
len_a = (len(data[data['종목명'] == data['종목명'][5]]))
print(len_a)

win = 0
lose = 0
state = 0
b = 0
s = 0

b_c = 0
m_c = 0
r_c = 0
c_c = 0
s_c = 0

for i in range(len(data)):
    if i%len_a == 0:
        state = 0
    if state==0:
        if data['BOL_trade'][i] == 1:
            b = data['Close'][i]
            state = 1
    if state== 1:
        if data['BOL_trade'][i] == 2:
            s = data['Close'][i]  
            state = 0
            b_c += 1 
            if b < s:
                win +=1
            else:
                lose+=1
            # print(b,s)
            b=0
            s=0
        elif data['BOL_trade'][i] == 1:
            # b = data['Close'][i]
            # lose += 1
            state = 1
print(win/(win+lose))

win = 0
lose = 0
state = 0
b = 0
s = 0

for i in range(len(data)):
    if i%len_a == 0:
        state = 0
    if state==0:
        if data['MACD_trade'][i] == 1:
            b = data['Close'][i]
            state = 1
    if state== 1:
        if data['MACD_trade'][i] == 2:
            s = data['Close'][i]  
            state = 0
            m_c += 1 
            if b < s:
                win +=1
            else:
                lose+=1
            b=0
            s=0
        elif data['MACD_trade'][i] == 1:
            # b = data['Close'][i]
            # lose += 1
            state = 1
print(win/(win+lose))

win = 0
lose = 0
state = 0
b = 0
s = 0

for i in range(len(data)):
    if i%len_a == 0:
        state = 0
    if state==0:
        if data['RSI_trade'][i] == 1:
            b = data['Close'][i]
            state = 1
    if state== 1:
        if data['RSI_trade'][i] == 2:
            s = data['Close'][i]  
            state = 0
            r_c += 1 
            if b < s:
                win +=1
            else:
                lose+=1
            b=0
            s=0
        elif data['RSI_trade'][i] == 1:
            # b = data['Close'][i]
            # lose += 1
            state = 1
print(win/(win+lose))

win = 0
lose = 0
state = 0
b = 0
s = 0

for i in range(len(data)):
    if i%len_a == 0:
        state = 0
    if state==0:
        if data['CCI_trade'][i] == 1:
            b = data['Close'][i]
            state = 1
    if state== 1:
        if data['CCI_trade'][i] == 2:
            s = data['Close'][i]  
            state = 0
            c_c += 1 
            if b < s:
                win +=1
            else:
                lose+=1
            b=0
            s=0
        elif data['CCI_trade'][i] == 1:
            # b = data['Close'][i]
            # lose += 1
            state = 1
print(win/(win+lose))



win = 0
lose = 0
state = 0
b = 0
s = 0

for i in range(len(data)):
    if i%len_a == 0:
        state = 0
        
    if state==0:
        if data['STO_trade'][i] == 1:
            b = data['Close'][i]
            state = 1
    if state== 1:
        if data['STO_trade'][i] == 2:
            s = data['Close'][i]  
            state = 0
            s_c += 1 
            if b < s:
                win +=1
            else:
                lose+=1
            b=0
            s=0
        elif data['STO_trade'][i] == 1:
            # b = data['Close'][i]
            # lose += 1
            state = 1
print(win/(win+lose))
# 56.9% 40.4% 57.8% 44.5% 34.0%

print(b_c,m_c,r_c,c_c,s_c)
data

print(len(data[data['BOL_trade']  == 1]), len(data[data['BOL_trade']  == 2]), "BOL",1/((len(data[data['BOL_trade']  == 1])+len(data[data['BOL_trade']  == 2]))/len(data)))


print(len(data[data['MACD_trade']  == 1]), len(data[data['MACD_trade']  == 2]),"MACD",1/((len(data[data['MACD_trade']  == 1])+len(data[data['MACD_trade']  == 2]))/len(data)))


print(len(data[data['RSI_trade']  == 1]), len(data[data['RSI_trade']  == 2]),"RSI",1/((len(data[data['RSI_trade']  == 1])+len(data[data['RSI_trade']  == 2]))/len(data)))


print(len(data[data['CCI_trade']  == 1]), len(data[data['CCI_trade']  == 2]),"CCI",1/((len(data[data['CCI_trade']  == 1])+len(data[data['CCI_trade']  == 2]))/len(data)))


print(len(data[data['STO_trade']  == 1]), len(data[data['STO_trade']  == 2]),"STO",1/((len(data[data['STO_trade']  == 1])+len(data[data['STO_trade']  == 2]))/len(data)))

data = pd.read_csv("상승_전체_올인.csv")

a = [0]*(len(data[data['종목명'] == data['종목명'][0]])-1)
b = [1]
c = (a+b)*int(len(data)/(len(data[data['종목명'] == data['종목명'][0]])))

data['out_%'] = c
data_out = data[data['out_%'] == 1]

#data_out = data_out.reset_index(drop=False, inplace = False)

q3 = data_out.quantile(0.75)
q1 = data_out.quantile(0.25)

iqr = q3 - q1

def is_outlier(df):
    indicator = "BOL_%"
    score = df[indicator]
    if score > q3[indicator] +1.5*iqr[indicator] or score < q1[indicator] - 1.5*iqr[indicator]:
        return True
    else:
        return False


data_out['BOL_outlier'] = data_out.apply(is_outlier, axis=1)
data_out_BOL = data_out.loc[data_out['BOL_outlier'] == False]

def is_outlier(df):
    indicator = "MACD_%"
    score = df[indicator]
    if score > q3[indicator] +1.5*iqr[indicator] or score < q1[indicator] - 1.5*iqr[indicator]:
        return True
    else:
        return False


data_out['MACD_outlier'] = data_out.apply(is_outlier, axis=1)
data_out_MACD = data_out.loc[data_out['MACD_outlier'] == False]

def is_outlier(df):
    indicator = "RSI_%"
    score = df[indicator]
    if score > q3[indicator] +1.5*iqr[indicator] or score < q1[indicator] - 1.5*iqr[indicator]:
        return True
    else:
        return False


data_out['RSI_outlier'] = data_out.apply(is_outlier, axis=1)
data_out_RSI = data_out.loc[data_out['RSI_outlier'] == False]

def is_outlier(df):
    indicator = "CCI_%"
    score = df[indicator]
    if score > q3[indicator] +1.5*iqr[indicator] or score < q1[indicator] - 1.5*iqr[indicator]:
        return True
    else:
        return False


data_out['CCI_outlier'] = data_out.apply(is_outlier, axis=1)
data_out_CCI = data_out.loc[data_out['CCI_outlier'] == False]

def is_outlier(df):
    indicator = "STO_%"
    score = df[indicator]
    if score > q3[indicator] +1.5*iqr[indicator] or score < q1[indicator] - 1.5*iqr[indicator]:
        return True
    else:
        return False


data_out['STO_outlier'] = data_out.apply(is_outlier, axis=1)
data_out_STO = data_out.loc[data_out['STO_outlier'] == False]

import numpy as np
import matplotlib.pyplot as plt
plt.figure(figsize = (5,5))
boxplot = data_out_BOL.boxplot(column = ['BOL_%'])
plt.yticks(np.arange(-10, 30, step=3))
plt.ylim([-40, 30])
plt.show
print(data_out_BOL['BOL_%'].mean())

import numpy as np
import matplotlib.pyplot as plt
plt.figure(figsize = (5,5))
boxplot = data_out_MACD.boxplot(column = ['MACD_%'])
plt.yticks(np.arange(-10, 30, step=3))
plt.ylim([-40, 30])
plt.show
print(data_out_MACD['MACD_%'].mean())

import numpy as np
import matplotlib.pyplot as plt
plt.figure(figsize = (5,5))
boxplot = data_out_RSI.boxplot(column = ['RSI_%'])
plt.yticks(np.arange(-10, 30, step=3))
plt.ylim([-40, 30])
plt.show
print(data_out_RSI['RSI_%'].mean())
