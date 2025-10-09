## Transactions on the Ethereum Network
Using the Etherscan API, we analyzed 25 million transactions on the Ethereum network. The transactions were obtained by traversing the Ethereum multigraph, starting from the Binance-associated address 0x4976A4A02f38326660D17bf34b431dC6e2eb2327. Addresses correspond to the vertices of this multigraph, while transactions correspond to the edges. The address mentioned above was selected because it is involved in a large number of transactions. During the multigraph traversal, a queue of the most frequently encountered addresses was maintained and continuously updated. At each step, traversal proceeded to the most popular address that had not yet been visited. This approach helps parse the largest possible number of transactions at each iteration. For every visited address, no more than 10_000 most recent transactions (in chronological order) were retrieved, and only those matching the following criteria were selected:
```Rust
 contractAddress == ""
 && isError == "0"
 && from != "GENESIS"
```
That is, only transactions that (1) are not related to a smart contract, (2) were successful, and (3) are not Genesis transactions were included. Below is an example of such transaction:
```JSON
{
"blockHash":"0x0b2e24b614a9035e3ad38e62af2a453083f13150dba44be5b19856825c711ce0",
"blockNumber":"19934203",
"from":"0x9a078509314a42af6e6e175d5e3def800af6697e",
"to":"0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
"gas":"90000",
"gasPrice":"13952663038",
"gasUsed":"40360",
"hash":"0x4691cb7350cafa17bcfd61a9d090ccb1262b9ad1f48b4015765ddc7d4f95c898",
"value":"0",
"nonce":"7",
"transactionIndex":"80",
"timeStamp":"1716486887",
"isError":"0",
"txreceipt_status":"1",
"input":"0xa9059cbb000000000000000000000000f89d7b9c864f589bbf53a82105107622b35eaa4000000000000000000000000000000000000000000000000000000000181571de",
"contractAddress":"",
"cumulativeGasUsed":"6485071",
"functionName":"transfer(address _to, uint256 _value)",
"methodId":"0xa9059cbb"
}
```
To calculate the monetary value transferred by each transaction, we multiplied the number in transaction’s value field by 1E-18 (the conversion factor from wei to ETH) and by the USD price of ETH at the time of the transaction. ETH-USD price data were obtained using the following SQL query to the Dune database:
```SQL
SELECT
    to_unixtime(DATE_TRUNC('hour', "minute")) AS "unix_epoch_at_the_start_of_averaging_period",
    AVG("price") AS "average_price_in_usd",
    "symbol"
FROM prices.usd
WHERE "symbol" = 'ETH' AND "contract_address" IS NULL
GROUP BY
    DATE_TRUNC('hour', "minute"),
    "symbol"
ORDER BY
    "unix_epoch_at_the_start_of_averaging_period" DESC
LIMIT 1000;
```
This SQL query retrieves the hourly average ETH price in USD. Averaging makes it possible to obtain ETH prices for the entire observation history. The decline in the purchasing power of the U.S. dollar over time was not accounted for.

The correctness of the statistical analysis was verified by constructing and manually calculating a small model multigraph of transactions. The parameters computed manually matched those obtained automatically for the model multigraph.

In the plot below, the horizontal axis represents the base-10 logarithm of transaction value in USD, while the vertical axis represents the number of transactions within each bin. For example, a value of log10(value in USD) ≈ 6.0 corresponds to transactions worth approximately 1E6 = 1_000_000 USD, and the graph shows that there are about 100_000 such transactions. Each plot contains 200 bins. The bin widths are variable — the higher the transaction value, the wider the bin. In other words, both the horizontal axis and bin widths are logarithmic. The transactions were not filtered by time, thus the graphs represent aggregate statistics across the entire Ethereum blockchain history (but no more than 10_000 most recent transactions from each address). Distribution of ETH transactions on the Ethereum network by value in USD:

![ETH-on-Ethereum-statistics.png](ETH-on-Ethereum-statistics.png)

**Statistics for ETH Transactions on the Ethereum Network**<br/>
Value transferred by transactions: 394_471_990_922 USD<br/>
Average transaction value: 15_778 USD<br/>
Number of transactions: 25_000_706<br/>

It is interesting that only 28.6% of transactions have a value greater than or equal to 1 cent USD. The remaining 71.4% of transactions have a value below 1 cent, often zero (see note ‡). Transactions with zero and near-zero value correspond to the bin at the far left of the plot, most of which is not visible at the chosen scale. The distribution clearly shows a mode of transactions around 100_000 USD, while transactions with a value above 10_000_000 USD are very rare. If we keep only transactions with a value of at least 1 cent, we obtain the following statistics:

Value transferred by transactions (>= 1 cent USD): 394_471_990_889 USD<br/>
Average transaction value (>= 1 cent USD): 55_135 USD<br/>
Number of transactions (>=1 cent USD): 7_154_708<br/>

‡ Note: Transactions with zero and near-zero values were not distinguished. Based on the statistics above, we can only state that transactions with zero value are ≤ 71.4% of all transactions. The SQL query below to the Dune database provides an estimate of 51.5% for the share of zero-value transactions under similar filtering for the year 2024.
```SQL
WITH t AS (
    SELECT value
    FROM ethereum.transactions
    WHERE 
        "success" = true AND
        EXTRACT(year FROM "block_time") = 2024
)
SELECT 
    ( CAST(COUNT(*) FILTER (WHERE "value" = 0) AS DOUBLE) / 
    CAST(COUNT(*) AS DOUBLE) )
    AS "zero_value_transaction_ratio"
FROM t;
```

### Two-Way ETH Transactions on the Ethereum Network
We define two-way transactions as all transactions between two addresses A and B, if there is at least one transaction A → B and at least one transaction B → A. The time between these transactions is not considered. Example of such a set of transactions:

```
Two-way transactions for addresses 
"0xa7efae728d2936e78bda97dc267687568dd593f3" <-> "0x55160fa2eb5eff4456d25482b9afb03c868e9350"
(3 + 2):
      |-> hash: 0x5293b603162cfff14b41d902c82c7249349103b4946448289e033b0591a05887 at 1716965987 unix epoch, value: 375 USD
      |-> hash: 0xa3c9df8204a0fc98be85e23c4405c9fc68240cff20c310600a0a46fe9e3345fe at 1717056155 unix epoch, value: 38 USD
      |-> hash: 0x76842c9c9753c8a976ef31f51260829f1d33033f768d3586e3ffc55ea8b7cccf at 1717061507 unix epoch, value: 745 USD
      <-| hash: 0xf035bdbcdf8c3ef1dccdd9a081a28ca7683d9737689179923b29157f5a3e98cb at 1716962819 unix epoch, value: 187 USD
      <-| hash: 0x2997b83b390ad8914052eb9cd0767c0b42d84899835c0fd807e42826ca014aba at 1717058807 unix epoch, value: 444 USD
Value for this set: 1789 USD, Flow for this set: 528 USD
```
It is notable that transactions filtered in this way show a different statistical distribution. Such transactions account for only 1.8% of the total number, but they contain 26.2% of all transferred monetary value.

![ETH-two-way-on-Ethereum-statistics.png](ETH-two-way-on-Ethereum-statistics.png)

**Statistics for Two-Way Transactions**<br/>
Value transferred by transactions (two-way): 103_188_315_036 USD<br/>
Average transaction value (two-way): 232_218 USD<br/>
Number of transactions (two-way): 444_360<br/>

### Transactions with USDT and USDC on the Ethereum Network
The data above show statistics only for ETH transfers within the Ethereum network. To obtain similar statistics for USDT and USDC movements, we conducted two additional experiments, each analyzing 10 million transactions.

The first filtering stage was the same as in the ETH experiment, but with an additional condition transaction.value == "0" (a zero in the value field). It should be noted that a zero in the value field does not mean that the amount of USDT or USDC transferred is zero — it only means that the transaction transfers zero ETH; the USDT or USDC amount is encoded in the input field. Thus, transactions with zero in the value field can still transfer USDT or USDC.

```Rust
 contractAddress == ""
 && isError == "0"
 && from != "GENESIS" 
 && transaction.value == "0"
```
Transactions with a zero value field make up 51.5% of all transactions for 2024. These can be divided into three categories: transactions transferring USDT, transactions transferring USDC, and other transactions (many of which transfer other tokens, not considered in this experiment). Below are the data for USDT and USDC.

![USDT-on-Ethereum-statistics.png](USDT-on-Ethereum-statistics.png)
![USDC-on-Ethereum-statistics.png](USDC-on-Ethereum-statistics.png)

**Statistics for USDT Transactions**<br/>
Number of transactions analyzed: 10_003_726<br/>
Number of transactions found: 235_265 (2.35%)<br/>
Average nonzero transaction value: 375_130 USD<br/>
Average transaction value: 8_822 USD<br/>
Value transferred by transactions: 88_255_036_629 USD<br/>

**Statistics for USDC Transactions**<br/>
Number of transactions analyzed: 10_004_501<br/>
Number of transactions found: 98_506 (0.98%)<br/>
Average nonzero transaction value: 709_041 USD<br/>
Average transaction value: 6_981 USD<br/>
Value transferred by transactions: 69_844_827_233 USD<br/>

Among transactions with a zero value field, only 2.35% involve USDT transfers (51.5% * 2.35% = 1.2% of all transactions) and only 0.98% involve USDC transfers (51.5% * 0.98% = 0.5% of all transactions). There are fewer USDC transactions, but on average they carry higher value. As a result, the total value transferred by USDT and USDC transactions is approximately the same.

## Transactions with USDT and USDC on the Polygon Network
The methodology for analyzing transactions with USDT and USDC tokens on the Polygon network was similar to the one previously used for the Ethereum network. In this experiment, we performed the following steps:

1. We obtained data on the Polygon network throughput for 2024 [from this source](https://polygonscan.com/chart/tx). It fluctuated significantly from November to December 2023 but remained almost constant throughout 2024 at around 4_000_000 transactions per day.
2. Using an SQL query to the Dune database (similar to the one shown earlier), we calculated the share of transactions with a zero value field in the Polygon network for 2024 — 85.7%.
3. We examined 10_002_728 transactions with a zero value field and filtered out transactions involving USDC and USDT that used the on-chain functions Transfer and TransferFrom. The results are as follows:

![USDT-and-USDC-on-Polygon-statistics.png](USDT-and-USDC-on-Polygon-statistics.png)

**Statistics for USDC Transactions**<br/>
Number of transactions analyzed: 10_002_728<br/>
Number of transactions found: 49_188 (0.49%)<br/>
Average transaction value: 2_391 USD<br/>
Value transferred by transactions: 117_584_394 USD<br/>

**Statistics for USDT Transactions**<br/>
Number of transactions analyzed: 10_002_728<br/>
Number of transactions found: 139_715 (1.40%)<br/>
Average transaction value: 2_495 USD<br/>
Value transferred by transactions: 348_611_442 USD<br/>

## Transactions with BSCUSD and USDC on the BSC (BNB Smart Chain) Network
The methodology for analyzing transactions with BSCUSD and USDC tokens on the BSC network was identical to the one used for the Polygon network. In this experiment, we performed the following steps:

1. We obtained data on the BSC network throughput for 2024 [from this source](https://bscscan.com/chart/tx). It fluctuated significantly from November to December 2023 but remained almost constant throughout 2024 at around 4_000_000 transactions per day (which is, interestingly, almost identical to Polygon’s throughput).
2. Using an SQL query to the Dune database (similar to the one shown earlier), we calculated the share of transactions with a zero value field in the BSC network for 2024 — 77.49%.
3. We examined 10_005_859 transactions with a zero value field and filtered out transactions involving BSCUSD and USDC that used the on-chain functions Transfer and TransferFrom. The results are as follows:

![USDC-and-BSCUSD-on-BSC-statistics.png](USDC-and-BSCUSD-on-BSC-statistics.png)

**Statistics for USDC Transactions**<br/>
Number of transactions analyzed: 10_005_859<br/>
Number of transactions found: 15_431 (0.15%)<br/>
Average transaction value: 319_986 USD<br/>
Value transferred by transactions: 4_937_706_788 USD<br/>

**Statistics for BSCUSD Transactions**<br/>
Number of transactions analyzed: 10_005_859<br/>
Number of transactions found: 367_433 (3.67%)<br/>
Average transaction value: 30_990 USD<br/>
Value transferred by transactions: 11_386_906_760 USD<br/>

The high average value of USDC transactions is explained by the fact that the dataset includes seventeen transactions each exceeding 1E8 USD in value — for example, see this transaction:<br/>
https://bscscan.com/tx/0x495974a6473cf0990d008547799e71cf269f0eb48a0dbddba5559b9d3e5fb92a 


# Russian translation

## Транзакции в сети Ethereum
С помощью Etherscan API мы проанализировали 25 миллионов транзакций в сети Ethereum. Транзакции были получены обходом мультиграфа Ethereum, начиная со связанного с Binance адреса 0x4976A4A02f38326660D17bf34b431dC6e2eb2327. Адреса соответствуют вершинам этого мультиграфа, а транзакции — ребрами. Указанный выше адрес был выбран потому, что он участвует в большом количестве транзакций. Во время обхода мультиграфа хранилась и обновлялась очередь из наиболее часто встречающихся в транзакциях адресов. Каждый новый шаг совершался к самому популярному из ещё не посещённых адресов. Такой подход позволяет на каждом шаге парсить наибольшее количество транзакций. С каждого адреса при обходе просматривалось не более 10_000 хронологически последних транзакций; из них выбирались только те транзакции, которые соответствуют следующим критериям:
```Rust
 contractAddress == ""
 && isError == "0"
 && from != "GENESIS"
```
То есть те транзакции, которые не связаны с контрактом, были успешными и не являются Genesis транзакциями. Ниже приведён пример выбранной транзакции:
```JSON
{
"blockHash":"0x0b2e24b614a9035e3ad38e62af2a453083f13150dba44be5b19856825c711ce0",
"blockNumber":"19934203",
"from":"0x9a078509314a42af6e6e175d5e3def800af6697e",
"to":"0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
"gas":"90000",
"gasPrice":"13952663038",
"gasUsed":"40360",
"hash":"0x4691cb7350cafa17bcfd61a9d090ccb1262b9ad1f48b4015765ddc7d4f95c898",
"value":"0",
"nonce":"7",
"transactionIndex":"80",
"timeStamp":"1716486887",
"isError":"0",
"txreceipt_status":"1",
"input":"0xa9059cbb000000000000000000000000f89d7b9c864f589bbf53a82105107622b35eaa4000000000000000000000000000000000000000000000000000000000181571de",
"contractAddress":"",
"cumulativeGasUsed":"6485071",
"functionName":"transfer(address _to, uint256 _value)",
"methodId":"0xa9059cbb"
}
```
Для расчета передаваемой транзакциями ценности мы умножали значение поля value транзакции на 1E-18 (коэффициент для пересчета из wei в ETH) и цену ETH в долларах США на момент транзакции. Значения цен ETH в USD были получены с помощью следующего SQL запроса к БД Dune:
```SQL
SELECT
    to_unixtime(DATE_TRUNC('hour', "minute")) AS "unix_epoch_at_the_start_of_averaging_period",
    AVG("price") AS "average_price_in_usd",
    "symbol"
FROM prices.usd
WHERE "symbol" = 'ETH' AND "contract_address" IS NULL
GROUP BY
    DATE_TRUNC('hour', "minute"),
    "symbol"
ORDER BY
    "unix_epoch_at_the_start_of_averaging_period" DESC
LIMIT 1000;
```
Этот SQL запрос позволяет получить среднюю почасовую стоимость ETH. Использование усреднения дает возможность получить цены на ETH за всю историю наблюдений. Снижение покупательной способности доллара не учитывалось.

Корректность статистического анализа проверяли с помощью создания и ручного обсчета маленького модельного мультиграфа транзакций. Рассчитанные вручную параметры совпали с рассчитанными автоматически для этого модельного мультиграфа.

Тут и далее на графиках по горизонтальной оси отложен десятичный логарифм ценности в долларах США, а по вертикальной оси отложено количество транзакций в данной корзине. Например log10 (value in USD) примерно 6.0 означает, что ценность каждой такой транзакции примерно равна 1E6 = 1_000_000 USD и на графике видно, таких транзакций около 100_000. Всего на графиках по 200 корзин. Ширина корзин различна: чем выше ценность, тем шире корзины. Иными словами, и горизонтальная ось, и ширины корзин логарифмические. Приведенные транзакции не фильтровались по времени; они описывают статистику за всю историю блокчейна Ethereum (но не более 10_000 последних транзакций с каждого адреса). Распределение транзакций ETH в сети Ethereum по ценности в долларах:

![ETH-on-Ethereum-statistics.png](ETH-on-Ethereum-statistics.png)

**Статистика для транзакций с ETH в сети Ethereum**<br/>
Отправленная транзакциям ценность: 394_471_990_922 USD<br/>
Средняя ценность транзакции: 15_778 USD<br/>
Количество транзакций: 25_000_706<br/>

Интересно, что только 28.6% транзакций имеет ценность выше или равную 1 центу USD. Остальные 71.4% транзакций имеют ценностью < 1 цента, часто нулевую ценность (см. примечание ‡). Транзакциям с нулевой и близкой к нулевой ценностью соответствует пик в крайней левой части графика, бОльшая часть которого не видна при выбранном масштабе. На распределении хорошо заметна компонента транзакций с ценностью около 100_000 USD, а вот транзакций с ценностью выше 10_000_000 USD очень мало. Если оставить только транзакции с ценностью не меньше цента, то получаем следующую статистику:

Отправленная транзакциям (>= 1 cent USD) ценность: 394_471_990_889 USD<br/>
Средняя ценность транзакции (>= 1 cent USD): 55_135 USD<br/>
Количество транзакций (>= 1 cent USD): 7_154_708<br/>

‡ Примечание: Транзакции с нулевой ценностью и транзакции с почти нулевой ценностью мы не разделяли. Из приведенной статистики можно утверждать только то, что транзакций с нулевой ценностью <= 71.4% от всех транзакциий. Приведенный ниже SQL запрос к БД Dune позволяет получить оценку в 51.5% для доли нулевых транзакций при аналогичной фильтрации за 2024 года.
```SQL
WITH t AS (
    SELECT value
    FROM ethereum.transactions
    WHERE 
        "success" = true AND
        EXTRACT(year FROM "block_time") = 2024
)
SELECT 
    ( CAST(COUNT(*) FILTER (WHERE "value" = 0) AS DOUBLE) / 
    CAST(COUNT(*) AS DOUBLE) )
    AS "zero_value_transaction_ratio"
FROM t;
```
### Взаимные транзакции ETH в сети Ethereum
Взаимными транзакциями будем называть все транзакции между двумя адресами A и B, если между этими адресами есть как минимум одна транзакция A → B и как минимум одна транзакция B → A. Время между этими транзакциями мы не учитываем. Пример набора таких транзакций:

```
Two-way transactions for addresses 
"0xa7efae728d2936e78bda97dc267687568dd593f3" <-> "0x55160fa2eb5eff4456d25482b9afb03c868e9350"
(3 + 2):
      |-> hash: 0x5293b603162cfff14b41d902c82c7249349103b4946448289e033b0591a05887 at 1716965987 unix epoch, value: 375 USD
      |-> hash: 0xa3c9df8204a0fc98be85e23c4405c9fc68240cff20c310600a0a46fe9e3345fe at 1717056155 unix epoch, value: 38 USD
      |-> hash: 0x76842c9c9753c8a976ef31f51260829f1d33033f768d3586e3ffc55ea8b7cccf at 1717061507 unix epoch, value: 745 USD
      <-| hash: 0xf035bdbcdf8c3ef1dccdd9a081a28ca7683d9737689179923b29157f5a3e98cb at 1716962819 unix epoch, value: 187 USD
      <-| hash: 0x2997b83b390ad8914052eb9cd0767c0b42d84899835c0fd807e42826ca014aba at 1717058807 unix epoch, value: 444 USD
Value for this set: 1789 USD, Flow for this set: 528 USD
```
Интересно, что у отфильтрованных таким образом транзакций другая статистика. Всего таких транзакций 1.8%, но в них содержится 26.2% всей пересылаемой ценности.

![ETH-two-way-on-Ethereum-statistics.png](ETH-two-way-on-Ethereum-statistics.png)

**Статистика для взаимных транзакций**<br/>
Отправленная транзакциям ценность (two-way): 103_188_315_036 USD<br/>
Средняя ценность транзакции (two-way): 232_218 USD<br/>
Количество транзакций (two-way): 444_360<br/>
### Транзакции с USDT и USDC в сети Ethereum
Данные выше показывают статистику только по перемещению ETH в сети Ethereum. С целью получить статистику по перемещению USDT и USDC мы провели два эксперимента, просмотрев по 10 миллионов транзакций в каждом.

Первый этап фильтрования был таким же, как в эксперименте с ETH, но с дополнительным условием  transaction.value == "0"  (с нулевым значением поля value). Стоит обратить внимание, что нулевое значение поля value не означает нулевое количество USDT или USDC в транзакции — нулевое значение поля value говорит только о том, что у транзакции нулевое перемещаемое количество ETH; количество USDT или USDC же кодируется в поле input. То есть транзакции с нулевым значением поля value могут нести USDT или USDC. 
```Rust
 contractAddress == ""
 && isError == "0"
 && from != "GENESIS" 
 && transaction.value == "0"
```
Транзакции с нулевым значением поля value составляют 51.5% от всех транзакций за 2024 год. Их можно разделить на три категории: транзакции с перемещением USDT, транзакции с перемещением USDC и иные транзакции (многие из которых перемещают другие токены, но в данном эксперименте мы их не рассматривали). Ниже приведены данные по USDT и USDC.

![USDT-on-Ethereum-statistics.png](USDT-on-Ethereum-statistics.png)
![USDC-on-Ethereum-statistics.png](USDC-on-Ethereum-statistics.png)

**Статистика для транзакций с USDT**<br/>
Количество просмотренных транзакций: 10_003_726<br/>
Количество найденных транзакций: 235_265 (2.35%)<br/>
Средняя ценность ненулевой транзакции: 375_130 USD<br/>
Средняя ценность транзакции: 8_822 USD<br/>
Отправленная транзакциям ценность: 88_255_036_629 USD<br/>

**Статистика для транзакций с USDC**<br/>
Количество просмотренных транзакций: 10_004_501<br/>
Количество найденных транзакций: 98_506 (0.98%)<br/>
Средняя ценность ненулевой транзакции: 709_041 USD<br/>
Средняя ценность транзакции: 6_981 USD<br/>
Отправленная транзакциям ценность: 69_844_827_233 USD<br/>

Среди транзакций с нулевым значением поля value только в 2.35% происходит перемещение USDT (51.5% * 2.35% = 1.2% от всех транзакций) и только в 0.98% происходит перемещение USDC (51.5% * 0.98% = 0.5% от всех транзакций). Транзакций с USDC меньше, но они в среднем несут большую ценность. В итоге транзакции с USDT и транзакции с USDC создают похожие объемы перемещения ценности. 

![Ethereum-circle-diagram.png](Ethereum-circle-diagram.png)

## Транзакции с USDT и USDC в сети Polygon 
Методология анализа транзакций с токенами USDT и USDC в сети Polygon была похожа на использованную ранее методологию для сети Ethereum. В данном эксперименте мы проделали следующие шаги:

1. Мы взяли данные о мощности сети Polygon за 2024 год [отсюда](https://polygonscan.com/chart/tx). Она сильно менялась с ноября по декабрь 2023 года, но за 2024 год была почти постоянна и составляла около 4_000_000 транзакций в день.
2. С помощью SQL запроса к БД Dune (аналогичного приведённому выше запросу) мы получили долю транзакций с нулевым значением поля value для сети Polygon за 2024 год — 85.7%.
3. Мы просмотрели 10_002_728 транзакций с нулевым значением поля value, отфильтровали транзакции с USDC и USDT, в которых использовались on-chain функции Transfer и TransferFrom. Результаты:

![USDT-and-USDC-on-Polygon-statistics.png](USDT-and-USDC-on-Polygon-statistics.png)

**Статистика для транзакций с USDC**<br/>
Количество просмотренных транзакций: 10_002_728<br/>
Количество найденных транзакций: 49_188 (0.49 %)<br/>
Средняя ценность транзакции: 2391 USD<br/>
Отправленная транзакциям ценность: 117_584_394 USD<br/>

**Статистика для транзакций с USDT**<br/>
Количество просмотренных транзакций: 10_002_728<br/>
Количество найденных транзакций: 139_715 (1.40 %)<br/>
Средняя ценность транзакции: 2495 USD<br/>
Отправленная транзакциям ценность: 348_611_442 USD<br/>

## Транзакции с BSCUSD и USDC в сети BSC (BNB Smart Chain)
Методология для анализа транзакций с токенами BSCUSD и USDC в сети BSC полностью аналогична использованной ранее методологии для сети Polygon. В данном эксперименте мы проделали следующие шаги:

1. Мы взяли данные о мощности сети BSC за 2024 год [отсюда](https://bscscan.com/chart/tx). Она сильно менялась с ноября по декабрь 2023 года, но за 2024 год была почти постоянна и составляла около 4_000_000 транзакций в день (да, это почти численное совпадение с мощностью Polygon).
2. С помощью SQL запроса к БД Dune (аналогичного приведённому выше запросу) мы получили долю транзакций с нулевым значением поля value для сети BSC за 2024 год — 77.49%.
3. Мы просмотрели 10_005_859 транзакций с нулевым значением поля value, отфильтровали транзакции с BSCUSD и USDC, в которых использовались on-chain функции Transfer и TransferFrom. Результаты:

![USDC-and-BSCUSD-on-BSC-statistics.png](USDC-and-BSCUSD-on-BSC-statistics.png)

**Статистика для транзакций с USDC**<br/>
Количество просмотренных транзакций: 10_005_859<br/>
Количество найденных транзакций:  15_431 (0.15%)<br/>
Средняя ценность транзакции: 319_986 USD<br/>
Отправленная транзакциям ценность: 4_937_706_788 USD<br/>

**Статистика для транзакций с BSCUSD**<br/>
Количество просмотренных транзакций: 10_005_859<br/>
Количество найденных транзакций: 367_433 (3.67%)<br/>
Средняя ценность транзакции: 30_990 USD<br/>
Отправленная транзакциям ценность: 11_386_906_760 USD<br/>

Высокая средняя стоимость транзакций с USDC объясняется тем, что мы просмотрели в том числе семнадцать транзакций с ценностью > 1E8 USD  каждая, например см. эту транзакцию:<br/>
https://bscscan.com/tx/0x495974a6473cf0990d008547799e71cf269f0eb48a0dbddba5559b9d3e5fb92a 
