### 3.3 图表保存与导出

一张精心绘制的图表，往往凝聚了大量的数据整理工作和分析思考。如果图表只能停留在屏幕上，而无法被保存、分享或嵌入到报告中，它的价值就会大打折扣。在实际工作中，我们通常需要将图表以多种形式导出——高清的PNG图片用于PPT演示，矢量格式的PDF用于学术论文，可交互的HTML用于网页展示。Matplotlib和mplfinance都提供了完整的图表导出机制，这一节我们将详细讲解如何灵活地保存和导出图表，以及如何在不同场景中选择最合适的格式。

#### 基本的图表保存方法

Matplotlib的`savefig`方法是图表保存的基础。它几乎可以在所有常见的图形格式中自由切换，且对输出质量有着精细的控制：

```python
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np

# Prepare some sample data
dates = pd.date_range('2024-01-01', periods=100, freq='D')
prices = 100 + np.cumsum(np.random.randn(100) * 2)

fig, ax = plt.subplots(figsize=(12, 6))
ax.plot(dates, prices, linewidth=1.5, color='blue')
ax.set_title('Sample Price Chart', fontsize=14)
ax.set_xlabel('Date')
ax.set_ylabel('Price ($)')
ax.grid(True, alpha=0.3)

# Basic save - PNG format with default settings
plt.savefig('chart_basic.png')
```

在上面的例子中，`plt.savefig('chart_basic.png')`将当前Figure保存为一个PNG文件。但默认设置往往不够理想——图片分辨率低、边缘留白过多、背景色可能不符合预期。因此，在实际使用中我们通常需要指定更多的参数。

#### 保存参数详解

`savefig`函数提供了十多个参数，其中最常用的几个值得深入了解：

```python
fig, ax = plt.subplots(figsize=(12, 6))
ax.plot(dates, prices, linewidth=1.5, color='blue')
ax.set_title('Sample Price Chart')
ax.grid(True, alpha=0.3)

plt.savefig(
    'chart_optimized.png',
    dpi=300,                    # Resolution in dots per inch
    bbox_inches='tight',        # Crop whitespace tightly
    facecolor='white',          # Background color
    edgecolor='none',           # Edge color of the figure
    transparent=False,          # Transparency for background
    pad_inches=0.1,             # Padding around the figure
    format='png'                # Explicit file format
)
```

**dpi（分辨率）** 是最常调整的参数。`dpi=300`是打印品质的标准，这意味着保存的图片每英寸包含300个像素点。而屏幕显示通常只需要`dpi=100`或`dpi=150`。较高的dpi会显著增大文件体积，因此需要根据实际使用场景权衡取舍。

| 使用场景              | 推荐dpi | 说明             |
| --------------------- | ------- | ---------------- |
| 屏幕展示（PPT、网页） | 100-150 | 文件小，加载快   |
| 打印文档              | 200-300 | 清晰度足够       |
| 高质量印刷            | 300-600 | 需要大幅面输出时 |

**bbox_inches** 控制图表的裁剪范围。默认值为`None`，会保留Figure内部所有的空白区域。`bbox_inches='tight'`则自动检测图表的实际内容区域，裁剪掉多余的空白，让图表尽可能紧凑。这在插入到报告或PPT中时尤为有用。

**facecolor** 设置图表的背景色。在默认情况下，Matplotlib图表的背景是透明的（`transparent=False`时实际上呈现白色）。如果需要在深色背景的文档中插入图表，可以将背景色设置为透明（`transparent=True`），或者使用`facecolor`来明确指定背景色。当`transparent=True`时，`facecolor`的设置会被覆盖，生成完全透明的背景。

#### 不同格式的特点与选择

Matplotlib支持多种图像格式，每种格式都有其独特的优势和适用场景：

```python
# PNG - Raster format, best for photos and complex images
plt.savefig('chart.png', dpi=150, bbox_inches='tight')

# PDF - Vector format, scalable without quality loss, ideal for publications
plt.savefig('chart.pdf', bbox_inches='tight')

# SVG - Vector format, editable, ideal for web use
plt.savefig('chart.svg', bbox_inches='tight')

# EPS - Vector format, compatibility with old publishing systems
plt.savefig('chart.eps', bbox_inches='tight')

# JPEG - Raster format, small file size, but loses quality (lossy)
plt.savefig('chart.jpg', dpi=150, bbox_inches='tight', quality=95)
```

**PNG**是最常用的栅格图像格式。它使用无损压缩，支持透明背景，适合绝大多数日常使用场景——PPT演示、网页嵌入、邮件附件等。对于包含大量文字和线条的金融图表，PNG是可靠的选择。它的缺点是文件体积相对较大。

**PDF**是矢量格式中的主力。矢量格式的特点是图像由数学公式描述而非像素点阵，因此无论放大多少倍，边缘始终清晰锐利。PDF格式在学术论文、技术报告、书籍出版等场景中被广泛采用。它还可以在Adobe Acrobat等软件中进一步编辑和标注。

**SVG**（可缩放矢量图形）是专为Web设计的矢量格式。它的文件体积通常比PDF更小，且可以被浏览器直接渲染。如果你需要将图表嵌入到网页中，或者希望图表在网页缩放时保持清晰，SVG是最佳选择。

**EPS**（Encapsulated PostScript）是一种较古老的矢量格式，主要用于兼容LaTeX等旧式排版系统。在最新的出版流程中，PDF已经取代了EPS的主流地位。除非你工作在与老式系统对接的环境中，否则EPS并不是首要选择。

**JPEG**使用有损压缩，文件体积小，但不适合包含清晰文字和线条的图表——JPEG的压缩算法会在锐利边缘周围产生"振铃效应"（ringing artifacts），导致文字变得模糊。金融图表包含大量的文字标签和网格线，使用JPEG会严重损害可读性。因此，除非你对文件体积有极端的限制，否则建议避免在图表中使用JPEG格式。

#### 使用mplfinance保存图表

对于使用mplfinance绘制的K线图，保存方式与Matplotlib基本一致，只是`savefig`参数被整合到了`mpf.plot()`函数中：

```python
import mplfinance as mpf

# Load prepared OHLCV data (assuming mpl_df is ready)
# For demonstration, we create a minimal DataFrame
# In practice, you would load your actual data

mpf.plot(
    mpl_df,
    type='candle',
    style='charles',
    title='AAPL Daily Chart',
    volume=True,
    savefig='charts/aapl_daily.png',      # Direct file path
    figsize=(14, 8)
)

# More control with a dictionary
mpf.plot(
    mpl_df,
    type='candle',
    style='charles',
    title='AAPL Daily Chart',
    volume=True,
    savefig={
        'fname': 'charts/aapl_daily_hq.png',
        'dpi': 300,
        'facecolor': 'white',
        'bbox_inches': 'tight'
    },
    figsize=(14, 8)
)

# Save as PDF for publication
mpf.plot(
    mpl_df,
    type='candle',
    style='charles',
    title='AAPL Daily Chart',
    volume=True,
    savefig='charts/aapl_daily.pdf',
    figsize=(14, 8)
)
```

#### 批量生成图表

在量化分析中，我们经常需要为多只股票生成相同的图表。这种情况下，手动逐一保存显然不可行。使用循环遍历所有股票，批量生成并保存图表，是常见的工程实践：

```python
import os
from pathlib import Path

def generate_charts_for_stocks(data_dir, chart_dir):
    """
    Batch generate candlestick charts for all stocks in data_dir.
    """
    # Create output directory if it doesn't exist
    Path(chart_dir).mkdir(parents=True, exist_ok=True)
    
    # Iterate through all CSV files in the data directory
    for file in Path(data_dir).glob('*.csv'):
        stock_code = file.stem  # Filename without extension
        
        # Load and prepare data (using functions from previous chapters)
        # df = prepare_stock_data(file)
        # mpl_df = prepare_data_for_mpl(df)
        
        # Generate and save the chart
        mpf.plot(
            mpl_df,
            type='candle',
            style='charles',
            title=f'{stock_code} Daily Chart',
            volume=True,
            savefig=os.path.join(chart_dir, f'{stock_code}.png'),
            figsize=(12, 8)
        )
        
        print(f"Chart saved for {stock_code}")

# Example usage
# generate_charts_for_stocks('data', 'charts')
```

这段代码的核心是使用`Path.glob('*.csv')`遍历目录中的所有CSV文件，对每个文件读取数据、生成图表并保存，并以股票代码作为输出文件名。在实际使用中，你可能还需要在图表中叠加均线或自定义指标，这些都可以根据需求在循环内部灵活添加。

通过这一节的学习，我们已经全面掌握了图表保存与导出的方法——包括不同格式的选择、分辨率的控制、mplfinance的保存方法以及批量生成的工程实践。至此，本书第一部分已经圆满结束。在第二部分中，我们将真正进入技术指标的世界，从最简单的移动平均线开始，逐一深入探索TA-Lib覆盖的各类指标。