### 2.3 数据读取与预处理

有了规范的数据文件和完善的开发环境，我们终于可以开始写代码了。这一节的核心目标只有一个：将CSV文件中的数据完整、准确地读入Python环境，使其成为TA-Lib可以消费的高质量时间序列。这个看似简单的过程，实则包含了大量的细节——从文件路径的处理到数据类型的校验，从日期索引的设置到数据质量的验证。跳过这些步骤或许能让代码在短期内"跑通"，但在处理真实数据时，数据格式的微小偏差就可能导致计算结果的巨大差异，而这种差异往往难以察觉。

#### 使用Pandas读取CSV文件

Pandas是Python数据分析领域的基石库，它提供的`read_csv`函数是我们与CSV文件打交道的主要接口。这个函数的设计极为灵活，几乎可以应对任何格式的CSV文件。但在实际使用中，很多人只用了它不到一半的参数——而恰恰是那些被忽略的参数，往往决定了数据是否能够被正确解析。

下面是一个基础的数据读取代码，它包含了大多数场景下所需的全部关键参数：

```python
import pandas as pd
import numpy as np
from pathlib import Path

def load_csv_data(filepath):
    """
    Load OHLCV data from CSV file with proper parsing.
    Returns a DataFrame with datetime index and correct dtypes.
    """
    # Use pathlib for cross-platform path handling
    data_path = Path(filepath)
    if not data_path.exists():
        raise FileNotFoundError(f"Data file not found: {filepath}")
    
    # Read CSV with explicit parameter settings
    df = pd.read_csv(
        data_path,
        parse_dates=['trade_date'],          # Parse date column as datetime
        encoding='utf-8',                    # Explicit encoding to avoid issues
        na_values=['', 'NA', 'null', 'NULL'], # Recognize common null representations
        thousands=None,                      # No thousands separators expected
        decimal='.'                          # Decimal point character
    )
    
    # Sort by date ascending - this is critical for time series analysis
    df = df.sort_values('trade_date').reset_index(drop=True)
    
    # Set date as the DataFrame index
    df.set_index('trade_date', inplace=True)
    
    return df

# Usage
df = load_csv_data('data/AAPL.csv')
print(f"Loaded {len(df)} records from {df.index[0]} to {df.index[-1]}")
```

这里有几个参数值得展开说明。`parse_dates=['trade_date']`告诉Pandas将指定的列解析为日期时间类型，而不是当作普通的字符串。这是一个关键步骤，因为日期作为索引时必须是时间类型，否则后续的切片、重采样等操作都无法正常进行。

`na_values`参数定义了一组在CSV文件中应被识别为空值的字符串。不同的数据源使用不同的方式表示缺失值——有的用空字符串，有的用`NA`，有的用`null`。通过这个参数，我们可以将所有这些变体统一转换为Pandas内部的`NaN`，便于后续统一处理。

`thousands`和`decimal`参数在处理不同地区格式的数据时尤为重要。例如，欧洲数据常用逗号作为小数点（如"123,45"），而美国数据常用句点作为小数点，同时用逗号作为千位分隔符（如"1,234.56"）。正确设置这两个参数可以避免将"1,234.56"误解为"1.23456"这种灾难性的解析错误。

#### 数据类型转换与强制校验

Pandas的`read_csv`虽然会尽力推断每列的数据类型，但在面对复杂或杂乱的数据时，它的推断并不总是可靠的。比如，当一列数据绝大多数是数字，但中间混入了一个非数字的字符串时，Pandas可能会将整列推断为`object`类型，导致后续的所有数值计算都无法进行。

因此，在读取数据之后，一个稳妥的做法是显式地进行数据类型转换，同时对可能存在的异常值进行检测和处理：

```python
def validate_and_convert_dtypes(df):
    """
    Ensure all price and volume columns have correct numeric types.
    Coerce invalid values to NaN for further handling.
    """
    # Define the expected columns
    price_cols = ['open', 'high', 'low', 'close']
    volume_cols = ['volume']
    
    # Convert price columns to float, coercing errors to NaN
    for col in price_cols:
        df[col] = pd.to_numeric(df[col], errors='coerce')
    
    # Convert volume column to float (allows decimals, some sources use floats)
    df['volume'] = pd.to_numeric(df['volume'], errors='coerce')
    
    # Check for any columns that couldn't be converted
    for col in price_cols + volume_cols:
        null_count = df[col].isna().sum()
        if null_count > 0:
            print(f"Warning: {null_count} invalid values found in column '{col}'")
    
    return df
```

`pd.to_numeric`是Pandas提供的一个非常实用的转换函数。它的`errors='coerce'`参数会将所有无法转换为数字的值强制设为`NaN`。这比默认行为（抛出异常）更加健壮——它允许我们记录问题并继续处理，而不是让整个程序崩溃。

在实际生产环境中，你可能会发现某些价格列中包含了百分比符号（如"2.5%"）或货币符号（如"$152.30"）。这些符号在`pd.to_numeric`中会导致转换失败。如果遇到这种情况，有两种处理思路：一是在读取CSV之前对原始文件进行清洗，移除这些符号；二是在转换前使用正则表达式提取数字部分：

```python
# Extract numeric values from strings with currency symbols
df['close'] = df['close'].astype(str).str.replace(r'[$,%]', '', regex=True)
df['close'] = pd.to_numeric(df['close'], errors='coerce')
```

#### 日期索引的设置与时间序列对齐

将日期设置为DataFrame的索引，不仅仅是为了方便查看数据。在时间序列分析中，一个正确设置的日期索引是实现数据对齐、重采样、滚动计算等操作的基础。

设置日期索引时，需要注意一个问题：**索引必须严格递增**。TA-Lib的底层实现假设输入数据是按照时间从旧到新排列的，如果顺序是乱的，计算结果将是错误的。因此，在设置索引之前，我们总是先执行排序：

```python
# Ensure the index is sorted chronologically
df = df.sort_index()

# Verify monotonicity
if not df.index.is_monotonic_increasing:
    raise ValueError("Date index is not monotonically increasing")

# Check for duplicate indices
if df.index.duplicated().any():
    print(f"Warning: Found {df.index.duplicated().sum()} duplicate dates")
    # Keep the last occurrence for each date, or use your own logic
    df = df[~df.index.duplicated(keep='last')]
```

`is_monotonic_increasing`是Pandas索引对象的一个属性，它可以快速判断索引是否严格递增。如果发现索引不单调，往往意味着数据排序出了问题，或者是CSV文件中混入了异常日期的记录。

关于时区（timezone）的问题也值得一提。如果你的数据涉及多个交易所（比如同时处理美股和港股），每个市场的交易时间不同，时区信息变得重要。在读取日期时，可以通过`utc=True`参数将日期统一转换为UTC时间，或者通过`tz_localize`和`tz_convert`方法进行时区转换。对于本书中的大多数示例，我们假设数据已经是当地时区的时间，并且所有数据来自同一市场，因此时区问题暂时不在我们的考虑范围内。

#### 数据完整性验证

在数据开始计算之前，进行一轮完整性验证是明智的做法。这些验证能够帮助我们在早期发现数据问题，避免"垃圾进、垃圾出"（garbage in, garbage out）的陷阱。

TA-Lib的大多数指标函数不会主动检查OHLC数据之间的逻辑一致性——如果你传入的数据中"最高价"低于"最低价"，函数并不会报错，而是会基于错误的数据计算出毫无意义的结果。因此，在数据进入计算流程之前，我们需要自己做好这道防线：

```python
def validate_ohlc_integrity(df):
    """
    Perform logical integrity checks on OHLC data.
    Raises AssertionError if any check fails.
    """
    # Fundamental OHLC constraints
    assert (df['high'] >= df['low']).all(), "High price must be >= Low price for all rows"
    assert (df['high'] >= df['open']).all(), "High price must be >= Open price for all rows"
    assert (df['high'] >= df['close']).all(), "High price must be >= Close price for all rows"
    assert (df['low'] <= df['open']).all(), "Low price must be <= Open price for all rows"
    assert (df['low'] <= df['close']).all(), "Low price must be <= Close price for all rows"
    
    # Check for non-negative prices
    price_cols = ['open', 'high', 'low', 'close']
    for col in price_cols:
        assert (df[col] >= 0).all(), f"Price column '{col}' contains negative values"
    
    # Check for non-negative volume
    assert (df['volume'] >= 0).all(), "Volume column contains negative values"
    
    # Check for excessive outliers (optional: 10x median)
    for col in price_cols:
        median = df[col].median()
        max_allowed = median * 10
        if (df[col] > max_allowed).any():
            print(f"Warning: Column '{col}' has values exceeding 10x median")
    
    print("All integrity checks passed.")
    return True
```

在实际使用中，你可能还需要根据具体的数据源和市场特征添加额外的校验规则。例如，某些市场的价格有最小变动单位（tick size），如果价格数据中出现非tick的数值，可能意味着数据精度存在问题。

#### 空值处理的策略选择

数据中的空值（NaN）是时间序列分析中无法回避的问题。TA-Lib本身的许多函数对输入中的NaN有特定的处理方式——有些函数会传播NaN（即如果输入包含NaN，输出也是NaN），有些则会跳过NaN进行计算。为了确保结果的可靠性，我们通常会在计算之前主动处理空值。

处理空值的策略取决于数据缺失的性质。主要有以下三种选择：

**向前填充（Forward Fill）** 是时间序列中最常用的方法。它的逻辑是：用缺失值之前最近的一个有效值来填补当前的空缺。这种方法适用于价格数据中个别交易日缺失的情况，因为价格在短期内通常具有连续性——没有交易的日子，价格可以视为与上一个交易日相同（严格来说，这并不精确，但作为一种近似处理，在大多数分析场景中是可以接受的）。

```python
# Forward fill: use the last valid value to fill missing ones
df.ffill(inplace=True)
```

**向后填充（Backward Fill）** 则是相反的方向——用缺失值之后的有效值来填充。这种方法在数据分析中用得相对较少，但在某些特殊场景中（比如数据采集存在延迟时）可能会有用。

```python
# Backward fill: use the next valid value to fill missing ones
df.bfill(inplace=True)
```

**插值（Interpolation）** 提供了更精细的控制。Pandas的`interpolate`方法支持多种插值算法，包括线性插值、多项式插值、样条插值等。对于价格数据，线性插值通常已经足够：

```python
# Linear interpolation for missing values
df['close'] = df['close'].interpolate(method='linear')
df[['open', 'high', 'low']] = df[['open', 'high', 'low']].interpolate(method='linear')
```

插值的优势在于它利用了缺失值两侧的信息来估计其合理位置，比简单的前向填充更加平滑。但它也有一个隐含假设：价格在缺失期间是线性变化的——这个假设在较长的缺失期（比如长达数周的停牌）中可能并不成立。

无论选择哪种方法，我们都需要在填充后进行最后的检查，确保所有空值已被消除：

```python
# Final null check after all imputation strategies
if df[['open', 'high', 'low', 'close', 'volume']].isnull().any().any():
    # If any nulls remain, drop those rows
    print("Warning: Some null values remain after filling. Dropping affected rows.")
    df.dropna(subset=['open', 'high', 'low', 'close', 'volume'], inplace=True)
```

#### 完整的读取与预处理流程

将上述所有步骤整合在一起，我们得到了一个完整的、可投入生产使用的数据加载函数。这个函数包含了从文件读取到数据验证的全链路处理：

```python
def prepare_stock_data(filepath, fill_method='ffill'):
    """
    Complete data preparation pipeline.
    
    Parameters:
    -----------
    filepath : str or Path
        Path to the CSV file
    fill_method : str
        Method for handling missing values: 'ffill', 'bfill', 'interpolate', or 'drop'
    
    Returns:
    --------
    pd.DataFrame
        Cleaned and validated OHLCV data with datetime index
    """
    # Step 1: Read CSV
    df = pd.read_csv(
        filepath,
        parse_dates=['trade_date'],
        encoding='utf-8',
        na_values=['', 'NA', 'null', 'NULL']
    )
    
    # Step 2: Sort and set index
    df = df.sort_values('trade_date').reset_index(drop=True)
    df.set_index('trade_date', inplace=True)
    
    # Step 3: Convert data types
    price_cols = ['open', 'high', 'low', 'close']
    for col in price_cols:
        df[col] = pd.to_numeric(df[col], errors='coerce')
    df['volume'] = pd.to_numeric(df['volume'], errors='coerce')
    
    # Step 4: Handle missing values based on specified method
    if fill_method == 'ffill':
        df[price_cols + ['volume']] = df[price_cols + ['volume']].ffill()
    elif fill_method == 'bfill':
        df[price_cols + ['volume']] = df[price_cols + ['volume']].bfill()
    elif fill_method == 'interpolate':
        df[price_cols + ['volume']] = df[price_cols + ['volume']].interpolate(
            method='linear', limit_area='inside'
        )
    elif fill_method == 'drop':
        df = df.dropna(subset=price_cols + ['volume'])
    else:
        raise ValueError(f"Unknown fill_method: {fill_method}")
    
    # Step 5: Final cleanup - drop any remaining rows with nulls
    df = df.dropna(subset=price_cols + ['volume'])
    
    # Step 6: Validate OHLC integrity
    validate_ohlc_integrity(df)
    
    # Step 7: Ensure numeric columns are float32 to save memory (optional)
    for col in price_cols:
        df[col] = df[col].astype(np.float32)
    df['volume'] = df['volume'].astype(np.float32)
    
    return df

# Usage example
df = prepare_stock_data('data/AAPL.csv', fill_method='ffill')
print(df.tail())
```

在调用这个函数时，`fill_method`参数的选择需要根据具体情况来判断。如果数据缺失率较低（例如少于1%），前向填充通常是最合适的。如果数据缺失率较高，可能需要考虑使用插值或直接删除相应行。对于书中的示例代码，我们默认使用`fill_method='ffill'`来保持一致性。

至此，数据准备工作已经全部完成。我们的DataFrame中包含了干净的、经过验证的OHLCV数据，日期索引正确设置，每一列都是正确的数值类型。在接下来的章节中，我们将把这份准备好的数据传递给TA-Lib的各项指标函数，开始真正的技术指标计算之旅。