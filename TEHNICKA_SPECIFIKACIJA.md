# Tehnička specifikacija CryptoBot aplikacije

## Arhitektura aplikacije

Predložena je slojevita (layered) i modularna arhitektura sa jasnim razdvajanjem domena, integracija i korisničkog interfejsa.

### Predlog namespace strukture

- `Core`
  - `Core.Models` — domen modeli (`MarketTicker`, `WalletBalance`, `TradeOrder`, `TradeExecution`).
  - `Core.DTOs` — ulazni/izlazni DTO modeli za servise.
  - `Core.Enums` — statusi i tipovi (`OrderSide`, `OrderType`, `Timeframe`).
  - `Core.Validation` — zajednička pravila validacije.
- `Services`
  - `Services.Interfaces` — ugovori (`IMarketService`, `IWalletService`, `ITradeService`).
  - `Services.Bybit` — implementacije servisa koje koriste Bybit API (market, wallet i trade podaci se preuzimaju isključivo sa Bybit API-ja).
  - `Services.Mapping` — mapiranje API modela u interne DTO modele.
- `Infrastructure`
  - `Infrastructure.Http` — HTTP klijent, retry politike, autentikacija.
  - `Infrastructure.Csv` — CSV čitanje/upis, validacija i deduplikacija.
  - `Infrastructure.Logging` — strukturirani logovi i audit zapisi.
  - `Infrastructure.Config` — učitavanje konfiguracije (`appsettings`, env varijable).
- `UI`
  - `UI.Forms` — forma/aplikativni ekran za ručni rad i monitoring.
  - `UI.ViewModels` — priprema podataka za prikaz.
- `Application`
  - `Application.UseCases` — orkestracija poslovnih tokova (npr. “Otvori poziciju”).
  - `Application.Scheduling` — periodično osvežavanje tržišnih i wallet podataka.

## Servisi i ključne metode

DTO modeli su dati kao predlog ugovora između slojeva.

### MarketService

1. `Task<GetTickerResponseDto> GetTickerAsync(GetTickerRequestDto request)`
   - Ulaz: `GetTickerRequestDto { string Symbol }`
   - Izlaz: `GetTickerResponseDto { string Symbol, decimal LastPrice, decimal Bid, decimal Ask, DateTime TimestampUtc }`
2. `Task<GetOrderBookResponseDto> GetOrderBookAsync(GetOrderBookRequestDto request)`
   - Ulaz: `GetOrderBookRequestDto { string Symbol, int Depth }`
   - Izlaz: `GetOrderBookResponseDto { string Symbol, List<OrderBookLevelDto> Bids, List<OrderBookLevelDto> Asks, DateTime TimestampUtc }`
3. `Task<GetKlinesResponseDto> GetKlinesAsync(GetKlinesRequestDto request)`
   - Ulaz: `GetKlinesRequestDto { string Symbol, string Interval, DateTime FromUtc, DateTime ToUtc, int Limit }`
   - Izlaz: `GetKlinesResponseDto { string Symbol, string Interval, List<KlineDto> Candles }`
4. `Task<GetInstrumentsResponseDto> GetInstrumentsAsync(GetInstrumentsRequestDto request)`
   - Ulaz: `GetInstrumentsRequestDto { string Category }`
   - Izlaz: `GetInstrumentsResponseDto { string Category, List<InstrumentDto> Instruments }`

### WalletService

1. `Task<GetBalancesResponseDto> GetBalancesAsync(GetBalancesRequestDto request)`
   - Ulaz: `GetBalancesRequestDto { string AccountType }`
   - Izlaz: `GetBalancesResponseDto { string AccountType, List<AssetBalanceDto> Assets, DateTime TimestampUtc }`
2. `Task<GetAccountSummaryResponseDto> GetAccountSummaryAsync(GetAccountSummaryRequestDto request)`
   - Ulaz: `GetAccountSummaryRequestDto { string AccountType, string BaseCurrency }`
   - Izlaz: `GetAccountSummaryResponseDto { decimal Equity, decimal WalletBalance, decimal AvailableBalance, decimal TotalValueUsd, decimal TotalValueEur, DateTime TimestampUtc }`
3. `Task<GetTransactionHistoryResponseDto> GetTransactionHistoryAsync(GetTransactionHistoryRequestDto request)`
   - Ulaz: `GetTransactionHistoryRequestDto { DateTime FromUtc, DateTime ToUtc, string? Currency, int Limit }`
   - Izlaz: `GetTransactionHistoryResponseDto { List<WalletTransactionDto> Items, string? NextCursor }`
4. `Task<TransferFundsResponseDto> TransferFundsAsync(TransferFundsRequestDto request)`
   - Ulaz: `TransferFundsRequestDto { string FromAccountType, string ToAccountType, string Currency, decimal Amount }`
   - Izlaz: `TransferFundsResponseDto { string TransferId, string Status, DateTime TimestampUtc }`
5. `Task<GetFxConversionResponseDto> GetFxConversionAsync(GetFxConversionRequestDto request)`
   - Ulaz: `GetFxConversionRequestDto { decimal Amount, string Asset, string[] QuoteCurrencies /* USD, EUR */ }`
   - Izlaz: `GetFxConversionResponseDto { decimal AmountInUsd, decimal AmountInEur, DateTime RateTimestampUtc, string Source }`

### TradeService

1. `Task<PlaceOrderResponseDto> PlaceOrderAsync(PlaceOrderRequestDto request)`
   - Ulaz: `PlaceOrderRequestDto { string Symbol, string Side, string OrderType, decimal Quantity, decimal? Price, string TimeInForce, string? ClientOrderId }`
   - Izlaz: `PlaceOrderResponseDto { string OrderId, string Symbol, string Status, DateTime CreatedAtUtc }`
2. `Task<CancelOrderResponseDto> CancelOrderAsync(CancelOrderRequestDto request)`
   - Ulaz: `CancelOrderRequestDto { string Symbol, string OrderId }`
   - Izlaz: `CancelOrderResponseDto { string OrderId, string Status, DateTime UpdatedAtUtc }`
3. `Task<GetOpenOrdersResponseDto> GetOpenOrdersAsync(GetOpenOrdersRequestDto request)`
   - Ulaz: `GetOpenOrdersRequestDto { string Symbol, int Limit, string? Cursor }`
   - Izlaz: `GetOpenOrdersResponseDto { List<OrderDto> Orders, string? NextCursor }`
4. `Task<GetTradeHistoryResponseDto> GetTradeHistoryAsync(GetTradeHistoryRequestDto request)`
   - Ulaz: `GetTradeHistoryRequestDto { string Symbol, DateTime FromUtc, DateTime ToUtc, int Limit }`
   - Izlaz: `GetTradeHistoryResponseDto { List<TradeFillDto> Fills, string? NextCursor }`
5. `Task<AmendOrderResponseDto> AmendOrderAsync(AmendOrderRequestDto request)`
   - Ulaz: `AmendOrderRequestDto { string Symbol, string OrderId, decimal? NewPrice, decimal? NewQuantity }`
   - Izlaz: `AmendOrderResponseDto { string OrderId, string Status, DateTime UpdatedAtUtc }`


Napomena o izvoru podataka: svi API pozivi za market, wallet i trade servise idu prema Bybit API v5 endpoint-ima preko `Services.Bybit` i `Infrastructure.Http` sloja.

## CSV specifikacija

### Opšta pravila CSV formata

- Separator kolona: **`,` (comma)**.
- Separator redova: `\n` (LF).
- Enkoding: `UTF-8`.
- Decimalni separator: **tačka (`.`)**.
- Format datuma/vremena: **ISO 8601 u UTC**, npr. `2026-04-28T10:15:30Z`.
- Prvi red je zaglavlje sa nazivima kolona.
- String vrednosti se ne citiraju osim kada sadrže separator ili navodnike (standard CSV escaping sa `"`).

### CSV: Market snapshot (`market_snapshot.csv`)

Obavezna polja:
- `timestamp_utc` (`string`, ISO 8601)
- `symbol` (`string`, npr. `BTCUSDT`)
- `last_price` (`decimal`)
- `bid` (`decimal`)
- `ask` (`decimal`)

Opciona polja:
- `volume_24h` (`decimal?`)
- `source` (`string?`, npr. `bybit-v5`)

Validacija pre upisa:
- `timestamp_utc` mora biti validan ISO 8601 UTC format.
- `symbol` nije prazan i prolazi regex `^[A-Z0-9]{3,20}$`.
- `last_price`, `bid`, `ask` > 0.
- `bid <= ask`.
- Ako postoji `volume_24h`, mora biti `>= 0`.

### CSV: Wallet stanje (`wallet_balance.csv`)

Obavezna polja:
- `timestamp_utc` (`string`, ISO 8601)
- `account_type` (`string`)
- `asset` (`string`, npr. `USDT`)
- `wallet_balance` (`decimal`)
- `available_balance` (`decimal`)

Opciona polja:
- `equity` (`decimal?`)
- `usd_value` (`decimal?`)
- `eur_value` (`decimal?`)

Validacija pre upisa:
- `wallet_balance >= 0`, `available_balance >= 0`.
- `available_balance <= wallet_balance` (osim ako model berze eksplicitno dozvoljava drugačije; tada log warning).
- `asset` prolazi regex `^[A-Z0-9]{2,15}$`.
- Ako postoje `usd_value`/`eur_value`, obe vrednosti moraju biti `>= 0` i izračunate iz istog timestamp-a kursa.

### CSV: Trade log (`trade_log.csv`)

Obavezna polja:
- `timestamp_utc` (`string`, ISO 8601)
- `order_id` (`string`)
- `symbol` (`string`)
- `side` (`string`, `Buy|Sell`)
- `order_type` (`string`, `Market|Limit`)
- `qty` (`decimal`)
- `status` (`string`)

Opciona polja:
- `price` (`decimal?`, obavezno za `Limit`, opcionalno za `Market`)
- `fee` (`decimal?`)
- `fee_asset` (`string?`)
- `client_order_id` (`string?`)

Validacija pre upisa:
- `qty > 0`.
- Ako je `order_type=Limit`, `price` mora postojati i biti `> 0`.
- `status` mora biti iz dozvoljenog skupa (`New`, `PartiallyFilled`, `Filled`, `Cancelled`, `Rejected`).
- Kombinacija (`order_id`, `timestamp_utc`, `status`) ne sme biti duplikat.

## Obrada grešaka

### Tipični scenariji i preporučeni odgovor sistema

1. **API timeout**
   - Retry sa exponential backoff (npr. 3 pokušaja: 500ms, 1s, 2s).
   - Ako i dalje ne uspe, vrati kontrolisanu grešku (`ExternalServiceTimeout`) i upiši incident u log.

2. **Rate limit (HTTP 429 / API code)**
   - Pročitati `Retry-After` ako postoji.
   - Pauzirati izvršavanje za traženi interval i ponoviti poziv.
   - Uvesti lokalni throttling po endpoint-u da se limit ne probija kontinuirano.

3. **Nevalidan JSON odgovor**
   - Uhvatiti deserialization grešku i sačuvati sirovi payload (skraćen/hashovan radi bezbednosti).
   - Označiti odgovor kao `InvalidExternalPayload` i ne upisivati nekonzistentne podatke u CSV.

4. **Prazan CSV**
   - Ako fajl ne postoji: kreirati fajl i zaglavlje.
   - Ako postoji ali je prazan: reinicijalizovati zaglavlje pre prvog upisa.
   - Ako nedostaje zaglavlje: tretirati kao format grešku i napraviti backup pre popravke.

5. **Duplikati zapisa**
   - Pre upisa proveriti jedinstveni ključ zapisa (npr. `order_id + timestamp_utc + status`).
   - Kod duplikata: preskočiti upis (idempotentno ponašanje) i evidentirati događaj kao `DuplicateIgnored`.
