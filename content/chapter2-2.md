### 2.2 CSV数据文件规范

开发环境配置妥当之后，接下来我们需要解决一个更根本的问题：数据从哪里来，以及数据应该是什么样子。

在后续的章节中，我们将使用CSV文件作为数据的存储介质。CSV（Comma-Separated Values，逗号分隔值）是一种极为简单且通用的数据格式。它不需要任何数据库引擎的支持，也不需要网络连接，只要有文本编辑器就能查看和编辑。这种简洁性使CSV成为数据交换领域的"通用语言"——几乎所有的数据分析工具和编程语言都支持CSV的读写。

但"简单"并不等于"随意"。为了让TA-Lib能够正确处理数据，我们的CSV文件需要遵循一套明确的规范。这一节将详细定义这些规范，并提供一个完整的示例数据文件。

#### 标准OHLCV数据结构

技术分析所需要的数据，本质上是一组时间序列。每一行代表一个交易日（或任意时间周期）的市场行情快照，每一列代表一个特定的数据字段。TA-Lib的大多数函数都接受以下五个核心字段作为输入：

| 字段名       | 含义     | 数据类型           | 说明                             |
| ------------ | -------- | ------------------ | -------------------------------- |
| `trade_date` | 交易日期 | 日期（YYYY-MM-DD） | 时间序列的索引，按升序排列       |
| `open`       | 开盘价   | 浮点数             | 交易时段开始时的第一笔成交价格   |
| `high`       | 最高价   | 浮点数             | 交易时段内达到的最高成交价格     |
| `low`        | 最低价   | 浮点数             | 交易时段内达到的最低成交价格     |
| `close`      | 收盘价   | 浮点数             | 交易时段结束时的最后一笔成交价格 |
| `volume`     | 成交量   | 整数或浮点数       | 交易时段内的总成交数量或金额     |

这六个字段统称为OHLCV数据（Open-High-Low-Close-Volume）。它们是绝大多数技术分析工作的原材料，也是TA-Lib函数签名中反复出现的参数。

需要注意的是，某些数据源可能使用不同的字段命名——例如用`date`代替`trade_date`，用`vol`代替`volume`。这种命名差异本身并不是问题，因为在读取数据时，我们可以通过Pandas的参数映射来统一字段名称。关键是数据本身的内容要符合规范。

#### 数据类型的严格要求

CSV文件本身是纯文本，所有的数据都以字符串形式存储。但当我们用Pandas读取这些数据时，必须将它们转换为正确的数据类型，否则TA-Lib的计算将无法进行甚至会产生错误结果。

**日期字段**必须能够被Pandas解析为`datetime64`类型。这是最基本的要求，因为时间序列分析的核心就在于数据的顺序和时间间隔。如果日期格式不规范（例如使用了`2023/01/15`或`15-Jan-2023`这样的格式），Pandas在解析时可能会出错或产生不可预期的结果。统一使用`YYYY-MM-DD`格式是最安全的选择。

在CSV文件头部加上`#`作为注释行是允许的，Pandas的`read_csv`函数提供了`comment`参数来忽略注释行。但需要注意，注释行必须符合格式规范，否则可能导致解析中断。

**价格字段**（open、high、low、close）必须能够被解析为浮点数。在实际市场中，价格可能是整数（如某些指数产品）或小数（如外汇）。无论哪种情况，数值的精度都需要保留。一个常见的错误是，在读取数据时Pandas将价格列识别为字符串或对象类型，这通常是因为文件中包含了非数字字符（如逗号分隔的千位符`1,234.56`或货币符号`$123.45`）。此时需要在读取时指定`thousands=','`或预先清洗数据。

**成交量字段**通常是整数（表示成交股数或手数），但在某些数据源中也可能以浮点数表示（如成交金额）。TA-Lib并不严格要求成交量为整数类型，但在计算量能指标时，成交量的相对大小比绝对数值更重要，因此保持其原始精度即可。

##### 一个规范的CSV文件示例

下面展示了一个完全符合我们规范的CSV文件内容。这是某只股票在2024年1月的前几个交易日的数据：

```csv
trade_date,open,high,low,close,volume
2024-01-02,150.20,152.80,149.50,152.30,2845600
2024-01-03,152.00,153.40,151.20,152.80,3123400
2024-01-04,153.00,155.60,152.50,155.20,3567800
2024-01-05,155.00,156.80,154.30,155.80,2981200
2024-01-08,156.00,158.20,155.00,157.60,4235600
```

请注意以下几点：

- 列名全部使用小写字母，无空格，使用下划线分隔单词。
- 日期格式为`YYYY-MM-DD`，这是国际标准ISO 8601格式，Pandas可以直接解析。
- 所有价格保留两位小数。
- 每行数据按日期升序排列——这非常重要，因为在后续使用Pandas时我们将设置日期为索引，升序的时间索引是TA-Lib计算正确性的基础。

在实际项目中，你可能也会遇到日期作为行索引而不是独立列的情况。两种格式Pandas都可以处理，但如果数据保存为行索引格式，在读取时需要注意设置`index_col=0`参数。

#### 数据文件命名规范

当你的分析工作从单只股票扩展到多只股票时，数据文件的命名方式就会成为一个重要的工程问题。一套清晰、一致的命名规范能让批量处理变得更加高效。

本书推荐使用**股票代码**作为文件名，例如`AAPL.csv`、`MSFT.csv`、`000001.csv`。这种命名方式的优势在于：代码本身是唯一的，且与大多数数据源的标识符一致。如果你同时处理多个市场的股票，可以在文件名中加入市场前缀以作区分，例如`US_AAPL.csv`、`CN_000001.csv`。

建议将所有的数据文件集中存放在一个目录中，例如项目根目录下的`data/`文件夹。这样，在批量处理时只需要遍历这个目录即可。

#### 数据预处理：缺失值与异常值的处理

现实世界中的数据从来不会完美无缺。你可能会遇到某一天的数据完全缺失（停牌、节假日休市），也可能遇到某个价格字段为空值或异常值。在将这些数据交给TA-Lib之前，我们需要进行适当的预处理。

**日期连续性检查**是首先要做的。金融市场的交易日并非连续——周末、法定节假日都会造成数据中断。TA-Lib并不要求数据在日历上完全连续，但它要求时间序列按升序排列且中间不能有空行。也就是说，如果某一天没有交易数据，这一行就不应该出现在CSV文件中。在Pandas中，我们可以在读取数据后检查日期的最大间隔：

```python
# Check for date gaps in the data
date_diffs = df['trade_date'].diff().dropna()
max_gap = date_diffs.max()
# If max_gap is more than a few days, it may indicate missing data
```

如果发现意外的断点，需要决定是填充还是跳过。对于技术指标计算来说，通常情况下跳过缺失的数据点不会造成问题，因为大多数TA-Lib函数会自动处理输入数据的边界条件。

**空值检查**。在CSV文件中，空值可能以空字符串或`NA`、`NaN`、`null`等形式出现。Pandas在读取时会将这些值识别为`NaN`。我们需要在计算前检查并处理这些空值：

```python
# Check for null values in the data
null_counts = df[['open', 'high', 'low', 'close', 'volume']].isnull().sum()
print(null_counts)
```

处理空值的常见方式有两种：向前填充（用前一个有效值填充）或删除包含空值的行。对于时间序列数据，向前填充通常是更合理的选择，尤其是在价格数据中：

```python
# Forward-fill missing values
df[['open', 'high', 'low', 'close', 'volume']] = df[['open', 'high', 'low', 'close', 'volume']].fillna(method='ffill')
```

**异常值检测**。有时候，数据中可能会包含明显不合理的数值——比如某一天的最高价低于最低价，或者价格出现了一个数量级的跳变。这些异常值通常源于数据录入错误或数据源本身的错误。对于这类情况，简单的过滤可能不够，需要结合具体的数据源和市场情况进行人工判断或采用统计方法（如基于标准差）进行检测和标注。

#### 数据读取的完整流程

将上述所有要点综合起来，下面是一个完整的、符合生产环境标准的数据读取与预处理流程：

```python
import pandas as pd
import numpy as np

def load_stock_data(filepath):
    """
    Load stock data from CSV file with proper data type conversion.
    This function handles date parsing, column mapping, and basic cleaning.
    """
    # Read CSV file, parse dates, and set index
    df = pd.read_csv(
        filepath,
        parse_dates=['trade_date'],
        encoding='utf-8'
    )
    
    # Sort by date ascending (TA-Lib requires this)
    df = df.sort_values('trade_date').reset_index(drop=True)
    
    # Set date as the DataFrame index
    df.set_index('trade_date', inplace=True)
    
    # Ensure all price columns are float type
    price_cols = ['open', 'high', 'low', 'close']
    for col in price_cols:
        df[col] = pd.to_numeric(df[col], errors='coerce')
    
    # Ensure volume is numeric
    df['volume'] = pd.to_numeric(df['volume'], errors='coerce')
    
    # Forward-fill any missing values (if the gap is small)
    df[price_cols + ['volume']] = df[price_cols + ['volume']].fillna(method='ffill')
    
    # Drop any remaining rows with null values
    df = df.dropna(subset=price_cols + ['volume'])
    
    # Verify data integrity
    assert (df['high'] >= df['low']).all(), "High must be >= Low in all rows"
    assert (df['high'] >= df['close']).all(), "High must be >= Close in all rows"
    assert (df['low'] <= df['close']).all(), "Low must be <= Close in all rows"
    
    return df

# Usage example
df = load_stock_data('data/AAPL.csv')
print(f"Loaded {len(df)} trading days of data")
print(df.head())
print("\nData info:")
print(df.info())
```

在这段代码中，有几个关键点需要留意。`errors='coerce'`参数的作用是，当某个字段无法被转换为数字时，将其设为`NaN`而不是抛出异常——这在处理脏数据时更为稳健。`assert`语句提供了一种轻量级的数据验证手段，用于确保OHLC数据在逻辑上是自洽的。如果数据违反了这些基本约束（例如最高价低于最低价），程序会立即报错并停止，这样可以防止错误的计算结果在后续的分析中传播。

至此，我们已经明确了CSV数据文件的规范、阅读方式以及预处理流程。在下一节中，我们将把这份规范落实到实践中，使用Pandas从CSV文件中读取数据，为后续的技术指标计算做好最后的准备。