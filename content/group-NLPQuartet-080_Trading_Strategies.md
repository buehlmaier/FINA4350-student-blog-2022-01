---
Title: Developing Trading Strategies (Group NLPQuartet) 
Date: 2022-05-02 16:21
Category: Progress Report
---

By Group "NLPQuartet", written by Kwok Tsun Hei and Tse Chun Hei


After obtaining the sentiment score from four different sentiment analysis models, we started thinking about what we could do with the correlation between sentiment score and the stock price. We finally decided to build a trading strategy to somehow implement what we have and to test the robustness of our findings.

## The 0.2 buy-sell strategy 

We came up with this strategy quite intuitively as it directly tells us if people view the stock as bullish or bearish and we hypothesized that a 0.2 change in sentiment score will represent that change of view. We started with no clue of whether to set the threshold at 0.2 or 0.3 or 0.4. Then we attempted all of them and concluded that 0.2 is the most reasonable one as the scale of the sentiment score is from -1 to 1, and it is not too often that the sentiment score changes would get too volatile; Meanwhile a 0.1 threshold will trigger some unreasonable trade that is too sensitive and may even be executed only due to errors. Thus, 0.2 seems to be a reasonable middle ground.
```python
def change_buy_sell():
    #set ac_balance (Account Balance) as $1M
    ac_balance = 1000000
    #set amount as number of stocks purchased
    amount=0
    #Set buy price
    buyprice = 0
    #Set sell price 
    sellprice = 0
    #Set Ireturn(investment return) as price difference of each trade 
    Ireturn = 0
    i=0
    securitiesMV = 0 
    #Intiate a list of return 
    Return = []
    for i in range(len(sscore)-1):
            #buy_action
            #if sentiment score increases by 0.2 over a day
             if(sscore[i+1]-sscore[i])>=0.2:
            #set the buy price of this trade 
                 buyprice = stock_price[i+1]
            #calculte the amount of stock and Market Value of securities bought
            #The two lines below are the same but makes this code more readable
                 amount = ac_balance/buyprice
                 securitiesMV = amount * buyprice

            #sell_action 
            #if sentiment score decreases by 0.2 over a day
             elif(sscore[i+1]-sscore[i]) <=-0.2: 
                 #set the sell price of this trade 
                 sellprice = stock_price[i+1]
                 #calculte the Market Value of securities sold
                 sellingMV = (amount*sellprice)
                 #Avoid calculating the price difference when there is no trade 
                 if buyprice != 0:
                     #price difference 
                     Ireturn = sellingMV- securitiesMV  
                     #Add to list
                     Return.append(Ireturn)
                     #Set to 0 and be ready to next trade
                     buyprice = sellprice = amount = 0
                     #renewing Account Balance
                     ac_balance = Ireturn + ac_balance
    #Calculate the ROI（％）
    return  (sum(Return)/1000000)*100
```

## Cross strategy 

The 0.2 buy-sell strategy looks good but it is a rather rudimentary trading principle and a limited validation of our results. Therefore , we developed the Cross method which is similar to the Moving Average Convergence Divergence (MACD). The Cross strategy buys when the 2MA crosses above the 4MA(moving average) and sells when the opposite occurs. This strategy, unlike the 0.2 strategy , also considers the trends of emotions in news and twitter. 

Also, for the scale, we were trying to micmic what people usually use the MACD, like looking at 20MA and 50MA etc, which means some 10s days. However, we found it is not reasonable for tweets and news as they may change rapidly due to some special announcements or firm’s activity. Therefore, we tried some combinations with 3 to 10 days MA, and found that 2 and 4 MA are the most profitable ones.

```python
def cross_trade():
    #set amount of stock as 0 
    amount=0
    #set account balance as 1000000
    Ac_balance=1000000
    #set buy sell and return as 0
    buyprice = 0
    sellprice = 0
    Ireturn = 0
    i=0
    securitiesMV = 0 
    #create a list for each return 
    Return = []
    for i in range(21,len(sscore)-1):
        #calculate 2day and 4day moving average 
        #fl = fast line      sl = slow line 
        fl=sum(sscore[i-2:i])/2
        sl=sum(sscore[i-4:i])/4
        #calculate 2day and 4dat moving average on a day before 
        #in order to spot if there is a breakthough (cross)
        #cfl = fast line for comparsion      sl = slow line for comparsion 
        cfl=sum(sscore[i-3:i-1])/2
        csl=sum(sscore[i-5:i-1])/4
        
        
        #trigger buy action if fast line is above slow line and 
        #if there is a twit
        #twit = (slow line - buy price) changing from negative to positive 
        if sl-fl <=0 and (cfl-csl)*(fl-sl)<=0: 
            #buy action
            #set buy price 
            buyprice = stock_price[i+1]
            #calculate amount of stock and securities market value 
            amount = Ac_balance/buyprice
            securitiesMV = amount * buyprice
       #sell_action 
       #trigger sell action if fast line is below slow line and 
       #if there is a twit
       #twit = (slow line - buy price) changing from positive to negative 
        elif sl-fl >=0 and (cfl-csl)*(fl-sl)<=0:   
           #set sellprice and selling market value 
           sellprice = stock_price[i+1]
           sellingMV = (amount*sellprice)
           #avoid selling before buying 
           if buyprice != 0:
               #calculate return (price difference)
               Ireturn = sellingMV-securitiesMV
               #add return to the list 
               Return.append(Ireturn)
               #set variable as 0 to be ready for next trade
               buyprice = sellprice = amount = 0
               #renew account balance 
               Ac_balance = Ireturn + Ac_balance
    #return ROI (%)           
    return  ((sum(Return))/1000000)*100
```