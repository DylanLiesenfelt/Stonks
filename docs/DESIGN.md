# Design Document

---

**Author:** Dylan Liesenfelt

**E-mail:** `djliesenfelt@outlook.com`

**Creation Date:** September 15, 2026

**Last Updated:** September 19, 2026

**Version:** v1.0

---

## Outline

Stonks is a portfolio project of mine that has seen different implementations and itterations by me over the past two. As have grown as a software engineer and my knowledege and skills have evolved, different itterations of Stonks have existed. This will be the final and definitive itteration of Stonks to show a professional level piece of open-source/free-ware stock market data platform. Its main purpose is to exist on my portfolio site and help me test and research things related to finanical markets.

---

## System Overview

This project follows a layered architecture. This is a backend only piece of software. But a frontend exists for it at my personal website `www.bubbanaut.net`.

The design is meant to be modular and easily maintainable, so it follows OOP, and SOLID principles moderately (not hyper strict).
The layered appraoch is straight forward; The Gateway is used to process and authenticate requests, that deligates down to each module.

I was tempted to have each module as its own micro-service (and a previous itteration did do this), however this goes back to over-engineering for the sake over-engineering. the scale of the modules is small, not massive systems of their own, so making microservices out of them is; 1. silly and unnecessary, 2. inefficent due to hops from http requests instead of intra program calls.

## System Architecture

---

### Request/Response Handling

```mermaid

graph LR
User((User))

subgraph API["Gateway"]
Request[Request] --> Validation[Validation] -->|request valid| ReqHandling[Request Handling]
Logging
Request -->|log request| Logging
Response
end

Features
Third[Third Party Data]
DB[Internal Data]
Errors[Error Handling]

User --> Request
ReqHandling --> Features
Features <-->|get data| Third
Features <-->|get data| DB
Features -->|throws| Errors
Errors -->|log errors| Logging
Validation -->|invalid request| Errors
Errors -->|error response| Response
ReqHandling -->|returns| Response
Response --> User
```

### Features

---

#### Market Data

`Market Data` is the core component, our hub of data. Almost all othe modules use it in some way or another. By default the main data provider is massive.com, but can be expanded to any other market data provider as long as the contracts are matched.

##### Design

```mermaid
---
config:
  layout: elk
  elk:
    nodePlacementStrategy: LINEAR_SEGMENTS
---
classDiagram
direction RL
namespace Presentation_Layer {
class MarketDataRouter {
        <<Router>>
+get_tickers(tickers: list)
+get_quotes(tickers: list)
+get_historical_prices(tickers: list, dates: list)
+get_bars(ticker: str, multiplier: int, timeframe: str, start: int, end: int)
+get_news(tickers: list)
+get_moving_average(ticker: str, method: str, multiplier: int, timeframe: str, start: int, end: int)
+get_atr(ticker: str, multiplier: int, timeframe: str, start: int, end: int)
}

class TickersResponse {
        <<Schema>>
        +results : dict[Tickers]
}

class QuotesResponse {
        <<Schema>>
        +results : dict[Quotes]
}

class PriceHistoryResponse {
        <<Schema>>
        +ticker : str
        +price : float
        +ts : int
}

class BarsResponse {
        <<Schema>>
        +results : dict[Bars]
}

class NewsResponse {
        <<Schema>>
        +results : dict[News]
}

class IndicatorResponse {
        <<Schema>>
        +results : dict[IndicatorPoint]
}
}

namespace Application_Layer {
    class MarketDataService {
        <<Facade>>
        -ticker_providers : list
        -quote_providers : list
        -historical_providers : list
        -news_providers : list
        -indicator_registry : dict[str, Indicator]
        +get_tickers()
        +get_quotes()
        +get_historical_prices()
        +get_bars()
        +get_news()
        +get_indicator(name: str, ticker: str, params: dict)
    }
}

namespace Domain_Layer {
class Quote {
        price : float
        ts : int
}

class Ticker {
        ticker : str
        name : str
        city : str
        state : str
        market_cap : float
        logo : str
}

class IndicatorPoint {
        value : float
        ts : int
}

class Bar {
        open : float
        high : float
        low : float
        close : float
        volume : float
        ts : int
}

class News {
        ticker : str
        title : str
        summary : str
        link : str
        img : str
}

class Indicator {
        <<Strategy Port>>
+calculate(bars: list) list~IndicatorPoint~
}

class MovingAverage {
        <<Strategy>>
        -method : str
        -multiplier : int
+calculate(bars: list) list~IndicatorPoint~
}

class ATR {
        <<Strategy>>
        -multiplier : int
+calculate(bars: list) list~IndicatorPoint~
}
}

namespace Infrastructure_Layer {
class TickerProvider {
        <<Port>>
+get_tickers()
}

class QuoteProvider {
        <<Port>>
+get_quotes()
}

class HistoricalProvider {
        <<Port>>
+get_historical_prices()
+get_bars()
}

class NewsProvider {
        <<Port>>
+get_news()
}

class MassiveClient {
        <<Adapter>>
        -API_KEY : str
+get_tickers()
+get_quotes()
+get_historical_prices()
+get_bars()
}

class YahooFinanceClient {
        <<Adapter>>
        -API_KEY : str
+get_news()
}
}

MarketDataRouter --> MarketDataService : calls

MarketDataRouter --> TickersResponse : returns
MarketDataRouter --> QuotesResponse : returns
MarketDataRouter --> PriceHistoryResponse : returns
MarketDataRouter --> BarsResponse : returns
MarketDataRouter --> NewsResponse : returns
MarketDataRouter --> IndicatorResponse : returns

MarketDataService --> TickerProvider : depends on
MarketDataService --> QuoteProvider : depends on
MarketDataService --> HistoricalProvider : depends on
MarketDataService --> NewsProvider : depends on
MarketDataService --> Indicator : registry of

MassiveClient ..|> TickerProvider : implements
MassiveClient ..|> QuoteProvider : implements
MassiveClient ..|> HistoricalProvider : implements
YahooFinanceClient ..|> NewsProvider : implements

MovingAverage ..|> Indicator : implements
ATR ..|> Indicator : implements

Indicator --> Bar : consumes
Indicator --> IndicatorPoint : produces

Quote --> QuotesResponse : used by
Ticker --> TickersResponse : used by
Bar --> BarsResponse : used by
IndicatorPoint --> IndicatorResponse : used by

TickerProvider --> Ticker : gives data
QuoteProvider --> Quote : gives data
HistoricalProvider --> Bar : gives data
NewsProvider --> News : gives data
```

##### Endpoints

| Method | Path | Router call | Params | Returns |
|---|---|---|---|---|
| GET | `/v1/market-data/tickers` | `get_tickers()` | query: `tickers` (repeatable) | `TickersResponse` |
| GET | `/v1/market-data/quotes` | `get_quotes()` | query: `tickers` (repeatable) | `QuotesResponse` |
| GET | `/v1/market-data/historical-prices` | `get_historical_prices()` | query: `tickers` (repeatable), `dates` (repeatable) | `PriceHistoryResponse` |
| GET | `/v1/market-data/bars` | `get_bars()` | query: `ticker`, `multiplier`, `timeframe`, `start`, `end` | `BarsResponse` |
| GET | `/v1/market-data/news` | `get_news()` | query: `tickers` (repeatable) | `NewsResponse` |
| GET | `/v1/market-data/indicators/moving-average` | `get_moving_average()` | query: `ticker`, `method`, `multiplier`, `timeframe`, `start`, `end` | `IndicatorResponse` |
| GET | `/v1/market-data/indicators/atr` | `get_atr()` | query: `ticker`, `multiplier`, `timeframe`, `start`, `end` | `IndicatorResponse` |


#### Indexes

The Indexes in module provides the means to make, measure, and monitor user created stock indexes.

##### Design

---

```mermaid
---
config:
  layout: elk
  elk:
    nodePlacementStrategy: LINEAR_SEGMENTS
---
classDiagram
direction LR
namespace Presentation_Layer {
    class IndexesRouter {
        <<Router>>
        +get_index(ticker: str)
        +get_performance(ticker: str)
        +get_quotes(tickers: list[str])
        +get_price_history(ticker: str, start: int, end: int, multiplier: int, timeframe: str)
        +list_indexes()
        +create_index(CreateIndex)
        +update_constituents(ticker: str, add: str, remove: str, rebalance: bool)
        +delete_index(ticker: str)
    }

    class CreateIndex {
        <<Schema>>
        ticker: str
        name: str
        description: str
        conception_date: int
        constituents: list[str]
        weighting_method: str
        rebalance_period: str
        base_value: float
    }
    class IndexResponse {
        <<Schema>>
        ticker: str
        name: str
        description: str
        conception_date: int
        constituents: list
        weighting_method: str
        rebalance_period: str
        base_value: float
        weightings: dict
        divisor: float
        last: float
        ts: int
        last_updated: int
    }
    class QuotesResponse {
        <<Schema>>
        +results: dict[Quote]
    }
    class PriceHistoryResponse {
        <<Schema>>
        +results: dict[Quote]
    }
    class PerformanceResponse {
        <<Schema>>
        +ticker: str
        +all_time: float
        +1y: float
        +6m: float
        +3m: float
        +1m: float
        +1w: float
        +1d: float
        +ts: int
    }
}

IndexesRouter --> IndexResponse : returns
IndexesRouter --> QuotesResponse : returns
IndexesRouter --> PerformanceResponse : returns
IndexesRouter --> PriceHistoryResponse : returns
IndexesRouter --> CreateIndex : receives

namespace Application_Layer {
    class IndexesService {
        <<Facade>>
        -indexes_repository: IndexesRepository
        +get_index(ticker: str)
        +get_performance(ticker: str)
        +get_quotes(tickers: list)
        +get_price_history(ticker, start: int, end: int, multiplier: int, timeframe: str)
        +list_indexes()
    }
    class IndexesMaintenance {
        <<Facade>>
        -indexes_repository: IndexesRepository
        -market_data: MarketDataClient
        -index_calculator: IndexCalculator
        -weighting_registry: dict[str, WeightingMethod]
        -indexes_register: list
        -constituents_register: list
        +create_index(request: CreateIndex)
        +update_constituents(ticker: str, constituents: list)
        +delete_index(ticker: str)
        +check_constituent_health()
        +repair_constituent(ticker: str)
        +check_index_health(ticker: str)
        +repair_index(ticker: str)
        +update_index_values()
        +rebalance_index_weights(ticker: str)
        +update_index_divisor(ticker: str)
        -load_registers()
    }
    class UserInputScreening {
        -blacklist
        +screen(Index) 
    }
}

IndexMaintenance --> UserInputScreening : calls
IndexesRouter --> IndexesService : calls
IndexesRouter --> IndexesMaintenance : calls

namespace Domain_Layer {
    class Index {
        +ticker: str
        +name: str
        +description: str
        +conception_date: int
        +constituents: list
        +weighting_method: str
        +rebalance_period: str
        +base_value: float
        +weightings: dict
        +divisor: float
        +last: float
        +ts: int
        +last_updated: int
    }
    class Quote {
        +ticker: str
        +price: float
        +quote_ts: int
    }
    class ConstituentsResponse {
        <<DTO>>
        +results: list[Quote]
    }
    class IndexCalculator {
        <<Domain Service>>
        +compute_value(index: Index, quotes: dict) float
        +compute_divisor(index: Index) float
        +apply_weighting(index: Index, method: WeightingMethod) dict
    }
    class WeightingMethod {
        <<Strategy Port>>
        +calculate(index: Index) dict
    }
    class EqualWeight {
        <<Strategy>>
        +calculate(index: Index) dict
    }
    class FixedWeight {
        <<Strategy>>
        +calculate(index: Index) dict
    }
    class MarketCapWeight {
        <<Strategy>>
        +calculate(index: Index) dict
    }
}

EqualWeight ..|> WeightingMethod : implements
FixedWeight ..|> WeightingMethod : implements
MarketCapWeight ..|> WeightingMethod : implements
IndexCalculator --> WeightingMethod : uses
IndexCalculator <--> Index : reads/updates

ConstituentsResponse --> Quote : contains
IndexesMaintenance --> ConstituentsResponse : uses
IndexesMaintenance --> IndexCalculator : uses
MarketDataClient --> ConstituentsResponse : returns

namespace Infrastructure_Layer {
    class IndexesRepository {
        <<Repository>>
        +make_index_table()
        +make_constituents_table()
        +create_index(index: Index)
        +update_index(index: Index)
        +delete_index(ticker: str)
        +create_constituent(ticker: str)
        +get_index_latest(ticker: str)
        +get_constituent_latest(ticker)
        +list_indexes()
        +list_constituents()
        +get_index_history(ticker: str, start: int, end: int, multiplier: int, timeframe: str)
        +get_constituent_history(ticker: str, start: int, end: int, multiplier: int, timeframe: str)
    }
    class MarketDataClient {
        <<Client>>
        +get_quotes(tickers: list)
    }
    
    class Scheduler {
        <<Driver>>
        -maintenance: IndexesMaintenance
        -rebalance_calendar: dict
        +run()
        -premarket_routine()
        -open_routine()
        -after_hours_routine()
        -closed_routine()
    }
    class DB {
    }
}
Scheduler --> IndexesMaintenance : drives
IndexesMaintenance --> IndexesRepository : uses
IndexesRepository --> DB : manages
```

##### Endpoints

| Method | Path | Router call | Body / Params | Returns |
|---|---|---|---|---|
| GET | `/v1/indexes/` | `list_indexes()` | none | list of index tickers |
| POST | `/v1/indexes/` | `create_index()` | body: `CreateIndex` | `IndexResponse` |
| GET | `/v1/indexes/quotes` | `get_quotes()` | query: `tickers` (repeatable) | `QuotesResponse` |
| GET | `/v1/indexes/{ticker}` | `get_index()` | path: `ticker` | `IndexResponse` |
| DELETE | `/v1/indexes/{ticker}` | `delete_index()` | path: `ticker` | 204 No Content |
| PATCH | `/v1/indexes/{ticker}/constituents` | `update_constituents()` | query/body: `add`, `remove`, `rebalance` | `IndexResponse` |
| GET | `/v1/indexes/{ticker}/performance` | `get_performance()` | path: `ticker` | `PerformanceResponse` |
| GET | `/v1/indexes/{ticker}/history` | `get_price_history()` | query: `start`, `end`, `multiplier`, `timeframe` | `PriceHistoryResponse` |

#### Trading Algorithims

Trading Algos return "signals"; BUY, HOLD, SELL. Based on the conditions set in the algo by its maker, the signal is returned on demand, not periodically its up the user to determine when to call for the signal (`Paper Trading`).

##### Design 

```mermaid
---
config:
  layout: elk
  elk:
    nodePlacementStrategy: LINEAR_SEGMENTS
---
classDiagram 
direction
namespace Presentation_Layer {
    class AlgorithmRouter {
        +get_algo(id: uuid)
        +get_algo_signal(id: uuid, data: dict)
        +create_algo(CreateAlgo)
        +update_algo(UpdateAlgo)
        +delete_algo()
    }

    class CreateAlgo {
    }

    class UpdateAlgo {
    }
}
namespace Application_Layer {
    class AlgorithimService {
        +get_algo(id: uuid)
        +get_algo_signal(id: uuid, data: dict)

    }

    class AlgorithmMaintenance {
        +create_algo(CreateAlgo)
        +update_algo(UpdateAlgo)
        +delete_algo()
    }

}
namespace Domain_Layer {
    class Algo {
        +id: uuid
        +name: str
        +description: str
        +created: int
        +conditions: dict[Condition]
        +screener: Screener
        +last_updated: dict[]
    }

    class Condition {

    }
}
namespace Infrastructure_Layer {
    class AlgoRepository {
    }
}
```

##### Endpoints

#### Paper Trading

##### Design

```mermaid
---
config:
  layout: elk
  elk:
    nodePlacementStrategy: LINEAR_SEGMENTS
---
classDiagram 
direction
namespace Presentation_Layer {

}
namespace Application_Layer {
    
}
namespace Domain_Layer {

}
namespace Infrastructure_Layer {
    
}
```


##### Endpoints