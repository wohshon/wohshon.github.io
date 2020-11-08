---
layout: post
title:  Pulling Historical Intraday Stock Info from Yahoo Finance
date:   2020-11-08
categories: tech
tags: python rapidapi
#image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
#image2: /assets/article_images/2014-11-30-mediator_features/night-track-mobile.JPG
---

Helped a friend with a weekend project recently, attempted a rather amateurish python program to pull historical stock data from [Yahoo Finance](https://finance.yahoo.com/), using [rapidapi's](https://rapidapi.com/marketplace) API platform. 

Do not expect any ground breaking contents as I am relatively new to Python as compared to Java and Node Js, with only a couple of Phython and Data Analytics online learning courses that I can boast of.

The choice of Python is of course driven by the usecase. The data are needed for analytics purposes and likely going to have some basic cleaning and data manipulation before it reach its final form. 

The overall flow looks like this, no rocket science:

<pre>

+-----------------+
|  CSV file with   | 
|   6000 stock    | <-- 1. get symbols                        
|   symbols       |          ▲                  
+-----------------+          |                                                       
                         [Program] ---- 2. get historical data based on symbols --▶ [Yahoo Finance]
                             |
+-----------------+          ▼ 
|  Final CSV file  | <-- 3. save all the historical data in csv file 
|                 |  
|                 |
+-----------------+                                                                 

</pre>

I don't have any requirements to package this into a deployable form in the near future, the deliverable is a csv file, which the friend is going to pump into analytical tool (likely to be Tableau); so I decided to use jupyter notebook. There is the tool I am most familiar with in the python world.


Here is a overview of what I have done:

#### Environment

I spin up a new CentOS vm, installed Conda, and setup an environment with the usual libraries like Pandas, Numpy and of course Jupyter notebook.

#### Reading from the CSV file

The original file lives in google drive, and I could have extract from the googlesheet directly, you can reference this [page](https://developers.google.com/sheets/api/quickstart/python) for instructions, it works but lots of hassle to grant authorization and it is my main google account, so I decide to just work on a offline copy of csv file.

There are about 6000 over records in the source csv file (let's call the `tickers.csv`) that looks like that
<pre>

Market,Symbol,Name,Sector,Industry,Summary Quote
NYSE,CMS,CMS Energy Corporation,Public Utilities,Power Generation,https://old.nasdaq.com/symbol/cms
NYSE,AX,"Axos Financial, Inc.",Finance,Savings Institutions,https://old.nasdaq.com/symbol/ax
...

</pre>

Of the 6000 over recrods, there were many dirty records with `n/a` data in the Sector or Industry fields, this subsequently caused problem when I tried to pull data from Yahoo Finance.

I tried to locate and filter off the records using panda's loc function but this amateur programmer couldn't get it working with the forward slash despite googling. In the bid to meet the deadline (I have a full time job), I decided to do a quick hack to replace the csv `n/a` values , removing the forward slashes.

And i saved the cleaned ticker symbol into an array for futher processing 

My python snipperts are as follows: 

```
# replace all the n/a with n_a before calling this
# :%s/n\/a/n_a/g
# Dunno why the pandas drop function cannot detect forward slash
# Load data main
import pandas as pd

# load ticker file
raw = pd.read_csv("tickers.csv")
# print(data.Symbol)
na_data = raw.loc[raw['Sector'] == "n_a"]
print("# of invalid data: " + str(na_data["Sector"].size))
clean_data = raw.drop(raw[raw['Sector'] == "n_a"].index)
arr = clean_data["Symbol"].to_numpy()
print(arr.size)
```
#### Pulling historical data from Yahoo Finance

The steps to pull data of the remainding 5000 ish records is straightforwad. I had a loop to read through the array of tickers and call the remote API one by one. As my error handling is not sophiscated, I break up the retrival into managable sizes by allowing a start and end row. 

The API actually returns the results in Json format, I originally returned a massive array of arrays, with each ticker symbol pointing to an array of their historical data. But my friend (or the tool) couldn't handle the json format. So we switched over to csv format, with a flat layout, repeating the ticker value every single row.

e.g. output

```
symbol,date,open,high,low,close,volume,adjclose,amount,type,data,numerator,denominator,splitRatio
ESLT,10-29-2020,115.12999725341797,116.91999816894531,112.83000183105469,113.25,124354.0,113.25,,,,,,
ESLT,10-27-2020,114.62000274658203,114.62000274658203,111.58999633789062,112.0999984741211,65800.0,112.0999984741211,,,,,,

```

I then save the records by their row counts. (Ran a script to merged in into a big file in the end) 

The code snipperts looks like this.

```
start_row = 5000
end_row = 6000
# arr is loaded from previous cell
for symbol in arr[start_row:end_row]:
    if not symbol:
        print("no more symbols to pull....")
        break    
    cnt=cnt+1
    print(str(cnt)+'. ' + symbol)

    url = "https://apidojo-yahoo-finance-v1.p.rapidapi.com/stock/v3/get-historical-data"

    querystring = {"region":"US","symbol":symbol}

    headers = {
        'x-rapidapi-host': "apidojo-yahoo-finance-v1.p.rapidapi.com",
        'x-rapidapi-key': "<hidden>"
    }
    response = requests.request("GET", url, headers=headers, params=querystring)
    try:
        j = json.loads(response.text)
        # print(j)

        if cnt > 1 :
            df1 = pd.DataFrame.from_dict(j["prices"])
            df1.insert (0, "symbol", symbol)
            df1['date'] = df1['date'].apply(lambda y: datetime.datetime.fromtimestamp(y).strftime("%m-%d-%Y"))
            df = pd.concat([df,df1], ignore_index=True)
        else:
            df = pd.DataFrame.from_dict(j["prices"])
            df.insert (0, "symbol", symbol)
            df['date'] = df['date'].apply(lambda y: datetime.datetime.fromtimestamp(y).strftime("%m-%d-%Y"))

        # print(df.to_csv(index=False))
        # display(df)
        print("*****done pulling record*****************")
    except:
        print("ERROR processing, skipping "+symbol)  
        errorSymbol.append(symbol)
        
df.to_csv('historical_records/records-'+str(start_row+1)+'_'+str(cnt+start_row-1)+'.csv',index=False)    
print('*****done saving file *****************records-'+str(start_row+1)+'_'+str(cnt)+'.csv') 
print("Error encountered:")
print(errorSymbol)
```

And .... That's all for now!