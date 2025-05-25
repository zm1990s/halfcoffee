---
layout: post
comments: true
title:  "Freqtrade 量化交易"
date:   2025-02-05 11:11:11
categories: 杂文
tags: 杂文
typora-root-url: ../../halfcoffee
---



## 前言

大概 7 月份的时候，一个朋友推荐了两个量化交易的软件，一个是开源的 Freqtrade，另一个是商业的 gunbot。本文记录下开源这个的使用。

## 安装

项目地址：[https://github.com/freqtrade/freqtrade](https://github.com/freqtrade/freqtrade)

本文使用 Docker 版本安装，非常简单：https://www.freqtrade.io/en/stable/docker_quickstart/[](https://www.freqtrade.io/en/stable/docker_quickstart/)



初始化步骤：

```shell
mkdir ft_userdata
cd ft_userdata/
# Download the docker-compose file from the repository
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# Pull the freqtrade image
docker compose pull

# Create user directory structure
docker compose run --rm freqtrade create-userdir --userdir user_data

# Create configuration - Requires answering interactive questions
docker compose run --rm freqtrade new-config --config user_data/config.json
```

运行最后一条命令时，会以向导的形式让用户填写配置，根据实际情况配置即可：

```
[root@ ft_userdata]# docker compose run --rm freqtrade new-config --config user_data/config.json
2025-02-05 13:40:53,682 - freqtrade - INFO - freqtrade 2025.1
2025-02-05 13:40:54,057 - numexpr.utils - INFO - NumExpr defaulting to 6 threads.
? Do you want to enable Dry-run (simulated trades)? Yes
? Please insert your stake currency: USDT
? Please insert your stake amount (Number or 'unlimited'): unlimited
? Please insert max_open_trades (Integer or -1 for unlimited open trades): 3
? Time Have the strategy define timeframe.
? Please insert your display Currency for reporting (leave empty to disable FIAT conversion): USD
? Select exchange binance
? Do you want to trade Perpetual Swaps (perpetual futures)? No
? Do you want to enable Telegram? No
? Do you want to enable the Rest API (includes FreqUI)? No
2025-02-05 13:41:37,028 - freqtrade.configuration.deploy_config - INFO - Writing config to `user_data/config.json`.
2025-02-05 13:41:37,028 - freqtrade.configuration.deploy_config - INFO - Please make sure to check the configuration contents and adjust settings to your needs.
```

之后 `ft_userdata` 目录下会有一系列的配置文件，我们后续只需要关心 `docker-compose.yml`,`config.json`以及 `strategies/xx.py`

```shell
[root@ft_userdata]# tree
.
├── docker-compose.yml
└── user_data
    ├── backtest_results
    ├── config.json
    ├── data
    │   └── binance
    ├── freqaimodels
    ├── hyperopt_results
    ├── hyperopts
    │   └── sample_hyperopt_loss.py
    ├── logs
    │   └── freqtrade.log
    ├── notebooks
    │   └── strategy_analysis_example.ipynb
    ├── plot
    └── strategies
        └── sample_strategy.py

11 directories, 6 files
```



修改 `config.json`，需要关注的参数如下：

- dry_run：是否使用测试模式，不进行真实的交易
- trading_mode：默认是 spot，可以改为 futures（合约）
- exchange 下的 key 以及 secret，具体获取方式参考交易所文档
- pair_whitelist：仅对列表中的币进行交易，此选项需要搭配 pairlists>method 为 StaticPairList 使用（具体配置看后面补充内容）
- pair_blacklist：不对列表中的币进行交易
- ccxt_config：为到交易所的连接配置代理（防止访问被夹）
- pairlists：使用静态的虚拟币列表还是基于交易量来获得列表
- telegram：telegram 对接相关配置，个人用下来还是挺好用的，会有交易消息推送，也可以远程查询获利情况
- listen_ip_address：建议改为 0.0.0.0，方便外部访问

```shell

{
    "max_open_trades": 5,
    "stake_currency": "USDT",
    "stake_amount": "unlimited",
    "tradable_balance_ratio": 0.99,
    "fiat_display_currency": "USD",
    "dry_run": true,
    "dry_run_wallet": 100,
    "cancel_open_orders_on_exit": false,
    "trading_mode": "spot",
    "margin_mode": "isolated",
    "unfilledtimeout": {
        "entry": 10,
        "exit": 10,
        "exit_timeout_count": 0,
        "unit": "minutes"
    },
    "entry_pricing": {
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "exit_pricing":{
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1
    },
    "exchange": {
        "name": "binance",
        "key": "XXXX",
        "secret": "XXXX",
        "pair_whitelist": [
            "DOGE/USDT",
            "BTC/USDT",
            "SOL/USDT",
            "ETH/USDT",
            "TRUMP/USDT"
        ],
        "pair_blacklist": [
            "BNB/.*"
        ],
        "ccxt_config": {
            "httpsProxy": "https://user:PASSWD@PROXYSERVER",
            "wsProxy": "https://user:PASSWD@PROXYSERVER"
          }
    },
    "pairlists": [
        {
      "method": "VolumePairList",
	    "number_assets": 20,
	    "sort_key": "quoteVolume",
	    "min_value": 0,
	    "refresh_period": 30
        }
    ],
    "telegram": {
        "enabled": true,
        "token": "1234567:xxxxxxxxxxxxxxxxxxxxxx",
        "chat_id": "34564666"
    },
    "api_server": {
        "enabled": true,
        "listen_ip_address": "0.0.0.0",
        "listen_port": 8080,
        "verbosity": "error",
        "enable_openapi": false,
        "jwt_secret_key": "xxxx",
        "ws_token": "xxxx",
        "CORS_origins": [],
        "username": "freqtrader",
        "password": "freqtrader"
    },
    "bot_name": "freqbot",
    "initial_state": "running",
    "force_entry_enable": false,
    "internals": {
        "process_throttle_secs": 5
    }
}
```

修改 `docker-compose.yml`，需要关注的参数如下：

- environment 中的 proxy，防止访问被夹
- strategy：指定要使用哪个交易策略（策略 python 文件需要保存在 `ft_userdata/user_data/strategies` 下）

```shell
cat docker-compose.yml
---
version: '3'
services:
  freqtrade:
    image: freqtradeorg/freqtrade:stable
    restart: unless-stopped
    container_name: freqtrade
    volumes:
      - "./user_data:/freqtrade/user_data"
    ports:
      - "8080:8080"
    environment:
      - HTTP_PROXY=https://user:PASSWD@PROXYSERVER
      - HTTPS_PROXY=https://user:PASSWD@PROXYSERVER
    command: >
      trade
      --logfile /freqtrade/user_data/logs/freqtrade.log
      --db-url sqlite:////freqtrade/user_data/tradesv3.sqlite
      --config /freqtrade/user_data/config.json
      --strategy KamaFama_2
```

最终启动：

```shell
docker compose up -d
```

访问服务器的 8080 端口，可以看到下列 Web 页面，可以直观地去查看交易情况：

<img src="/pics/WX20250205-221145@2x.png"  style="zoom:50%;" />



## 高级配置细项

### 交易策略

官方有很多示例策略：[https://github.com/freqtrade/freqtrade-strategies/tree/main](https://github.com/freqtrade/freqtrade-strategies/tree/main)

除此之外，看到这个网站也有策略以及策略排行：[https://strat.ninja/ranking.php](https://strat.ninja/ranking.php)

下载后将其保存在  `ft_userdata/user_data/strategies` 下，然后修改 `docker-compose.yml` 即可。

*注意：部分策略需要安装额外的 Python 依赖，具体需要看策略的备注。*

在写这篇文章的时候，我用的是 [KamaFama 策略](https://github.com/Mastaaa1987/freqtrade-strategie/tree/main/KamaFama)，同时在对 [Diamond 策略](https://github.com/freqtrade/freqtrade-strategies/blob/main/user_data/strategies/Diamond.py)进行测试。

### 基于静态列表进行交易

修改 config.json 中的下列配置：

```json
"pair_whitelist":["EOS/USDT", "ETH/USDT", "BTC/USDT", "SOL/USDT", "TRUMP/USDT", "PROM/USDT", "HIVE/USDT",  "ZRX/USDT"],

中间部分略

"pairlists": [
        {
            "method": "StaticPairList"
        }
  ],
```



### stoploss 调整

跑了两个月，突然发现策略的 stoploss 都没有生效（关闭的），于是参照官方文档调整了：

```shell
    # Optimal stoploss designed for the strategy
    # This attribute will be overridden if the config file contains "stoploss"
    # 修改默认止损从 -0.1 到 -0.08（-10%到-8%）
    stoploss = -0.08

    # Optimal timeframe for the strategy
    timeframe = '5m'

    # trailing stoploss
    # 开启了 trailing_stop
    trailing_stop = True
    trailing_stop_positive = 0.01
    trailing_stop_positive_offset = 0.02
    
    # Optional order type mapping
    # 开启了stoploss_on_exchange，在下单后就设置止损策略
    order_types = {
        'entry': 'limit',
        'exit': 'limit',
        'stoploss': 'market',
        'stoploss_on_exchange': True
    }
```

trailing_stop 主要用于在盈利时调整止损比例，比如收益到 2%时，只要下跌 1% 就止损。这样可以尽可能锁住盈利。但缺点可能是交易更频繁。

### stoploss 调整后续

自从调整 trailing_stop 后，全部 Loss...应该是设置的点有点低，基本每天稳定亏钱，先取消再看看吧...只留止损策略。



### 20250512

有一个设置了 `trailing_stop = True` 竟然没事，一个月赢率比较高，贴下策略：

```json
# cat ../ft_userdata4/user_data/config.json

{
    "max_open_trades": 3,
    "stake_currency": "USDT",
    "stake_amount": "unlimited",
    "tradable_balance_ratio": 0.99,
    "fiat_display_currency": "USD",
    "dry_run": false,
    "dry_run_wallet": 100,
    "cancel_open_orders_on_exit": false,
    "trading_mode": "spot",
    "margin_mode": "isolated",
    "unfilledtimeout": {
        "entry": 10,
        "exit": 10,
        "exit_timeout_count": 0,
        "unit": "minutes"
    },
    "entry_pricing": {
        "price_side": "same",
        "use_order_book": true,
        "order_book_top": 1,
        "price_last_balance": 0.0,
        "check_depth_of_market": {
            "enabled": false,
            "bids_to_ask_delta": 1
        }
    },
    "exit_pricing":{
        "price_side": "other",
        "use_order_book": true,
        "order_book_top": 1
    },
    "exchange": {
        "name": "binance",
        "pair_whitelist": [
            "BTC/USDT", "ETH/USDT", "XRP/USDT", "SOL/USDT", "DOGE/USDT", "ADA/USDT", "TRX/USDT", "AVAX/USDT", "SUI/USDT", "LTC/USDT", "TON/USDT", "HBAR/USDT", "OM/USDT", "BCH/USDT", "NEAR/USDT", "APT/USDT", "TAO/USDT", "ICP/USDT", "ETC/USDT", "S/USDT", "VET/USDT", "ALGO/USDT"
        ],
        "pair_blacklist": [
            "BNB/.*"
        ],
    },
    "pairlists": [
        {
            "method": "StaticPairList"
        }
    ],

    },
    "bot_name": "xx003-marketcap",
    "initial_state": "running",
    "force_entry_enable": false,
    "internals": {
        "process_throttle_secs": 5
    }
}
```



```shell
cat ../ft_userdata4/user_data/strategies/Strategy003.py

# --- Do not remove these libs ---
from freqtrade.strategy import IStrategy
from typing import Dict, List
from functools import reduce
from pandas import DataFrame
# --------------------------------

import talib.abstract as ta
import freqtrade.vendor.qtpylib.indicators as qtpylib
import numpy # noqa


class Strategy003(IStrategy):
    """
    Strategy 003
    author@: Gerald Lonlas
    github@: https://github.com/freqtrade/freqtrade-strategies

    How to use it?
    > python3 ./freqtrade/main.py -s Strategy003
    """

    INTERFACE_VERSION: int = 3
    # Minimal ROI designed for the strategy.
    # This attribute will be overridden if the config file contains "minimal_roi"
    minimal_roi = {
        "60":  0.01,
        "30":  0.03,
        "20":  0.04,
        "0":  0.05
    }

    # Optimal stoploss designed for the strategy
    # This attribute will be overridden if the config file contains "stoploss"
    stoploss = -0.4

    # Optimal timeframe for the strategy
    timeframe = '5m'

    # trailing stoploss
    trailing_stop = True
    trailing_stop_positive = 0.01
    trailing_stop_positive_offset = 0.02

    # run "populate_indicators" only for new candle
    process_only_new_candles = True

    # Experimental settings (configuration will overide these if set)
    use_exit_signal = True
    exit_profit_only = True
    ignore_roi_if_entry_signal = False

    # Optional order type mapping
    order_types = {
        'entry': 'limit',
        'exit': 'limit',
        'stoploss': 'market',
        'stoploss_on_exchange': True
    }

    def informative_pairs(self):
        """
        Define additional, informative pair/interval combinations to be cached from the exchange.
        These pair/interval combinations are non-tradeable, unless they are part
        of the whitelist as well.
        For more information, please consult the documentation
        :return: List of tuples in the format (pair, interval)
            Sample: return [("ETH/USDT", "5m"),
                            ("BTC/USDT", "15m"),
                            ]
        """
        return []

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Adds several different TA indicators to the given DataFrame

        Performance Note: For the best performance be frugal on the number of indicators
        you are using. Let uncomment only the indicator you are using in your strategies
        or your hyperopt configuration, otherwise you will waste your memory and CPU usage.
        """

        # MFI
        dataframe['mfi'] = ta.MFI(dataframe)

        # Stoch fast
        stoch_fast = ta.STOCHF(dataframe)
        dataframe['fastd'] = stoch_fast['fastd']
        dataframe['fastk'] = stoch_fast['fastk']

        # RSI
        dataframe['rsi'] = ta.RSI(dataframe)

        # Inverse Fisher transform on RSI, values [-1.0, 1.0] (https://goo.gl/2JGGoy)
        rsi = 0.1 * (dataframe['rsi'] - 50)
        dataframe['fisher_rsi'] = (numpy.exp(2 * rsi) - 1) / (numpy.exp(2 * rsi) + 1)

        # Bollinger bands
        bollinger = qtpylib.bollinger_bands(qtpylib.typical_price(dataframe), window=20, stds=2)
        dataframe['bb_lowerband'] = bollinger['lower']

        # EMA - Exponential Moving Average
        dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
        dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
        dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
        dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)

        # SAR Parabol
        dataframe['sar'] = ta.SAR(dataframe)

        # SMA - Simple Moving Average
        dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Based on TA indicators, populates the buy signal for the given dataframe
        :param dataframe: DataFrame
        :return: DataFrame with buy column
        """
        dataframe.loc[
            (
                (dataframe['rsi'] < 28) &
                (dataframe['rsi'] > 0) &
                (dataframe['close'] < dataframe['sma']) &
                (dataframe['fisher_rsi'] < -0.94) &
                (dataframe['mfi'] < 16.0) &
                (
                    (dataframe['ema50'] > dataframe['ema100']) |
                    (qtpylib.crossed_above(dataframe['ema5'], dataframe['ema10']))
                ) &
                (dataframe['fastd'] > dataframe['fastk']) &
                (dataframe['fastd'] > 0)
            ),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Based on TA indicators, populates the sell signal for the given dataframe
        :param dataframe: DataFrame
        :return: DataFrame with buy column
        """
        dataframe.loc[
            (
                (dataframe['sar'] > dataframe['close']) &
                (dataframe['fisher_rsi'] > 0.3)
            ),
            'exit_long'] = 1
        return dataframe
```



## 策略记录

### kamafama_2

运行约 18 天，一开始用的作者使用的币清单，亏损严重，于是换成了 volume 前 25，效果也一般，有些 Loss 损失很大，直接把之前的收益清零。**目前已放弃该策略。**

```shell
 Running Freqtrade 2025.1

Running with 4xunlimited USDT on binance in spot markets, with Strategy KamaFama_2.

Stoploss on exchange is disabled.

Currently running, force entry: false

Dry-Run

Avg Profit -0.277% (∑ -23.025%) in 83 Trades, with an average duration of 8:02:42. Best pair: BNX/USDT.

Bot start date: 2025-02-04 09:03:05 (UTC) First trade opened: 2025-02-04 21:11:17 (UTC) Last trade opened: 2025-02-22 02:40:09 (UTC)

Profit factor: 0.42 Trading volume: 4431.301 USDT
Metric	Value
ROI closed trades	-33.189 USDT (0.12%)
ROI all trades	-38.739 USDT (-0.28%)
Total Trade count	83
Bot started	2025-02-04 09:03:05
First Trade opened	2025-02-04 21:11:17
Latest Trade opened	2025-02-22 02:40:09
Win / Loss	73 / 6
Winrate	92.405%
Expectancy (ratio)	-0.42 (-0.04)
Avg. Duration	8:02:42
Best performing	BNX/USDT: 3.16%
Trading volume	4431.301 USDT
Profit factor	0.42
Max Drawdown	42.37% (44.657 USDT) from 2025-02-06 05:39:56 to 2025-02-08 14:05:09
```

![image-20250222115356150](/pics/image-20250222115356150.png)

![image-20250222115404757](/pics/image-20250222115404757.png)

### Diamond

跑了两天全是亏损的，放弃

### NFI5MOHO

跑了一周左右亏损，放弃
