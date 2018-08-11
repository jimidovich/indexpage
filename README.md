
# 基于即期收益率曲线的债券定价

[TOC]

## 1. 以固定利率债为例，简单说明使用Quantlib进行即期利率期限结构的构建和对债券的定价。

利率曲线以及利率期限结构的含义：

> In finance, the **yield curve**(利率曲线) is a curve showing several yields or interest rates across different contract lengths (2 month, 2 year, 20 year, etc. ...) for a similar debt contract. The curve shows the relation between the (level of the) interest rate (or cost of borrowing) and the time to maturity, known as the "term", of the debt for a given borrower in a given currency. For example, the U.S. dollar interest rates paid on U.S. Treasury securities for various maturities are closely watched by many traders, and are commonly plotted on a graph such as the one on the right which is informally called "the yield curve". More formal mathematical descriptions of this relation are often called the **term structure of interest rates**(利率的期限结构).

即期收益率，又称**Spot Rate**或**Zero Rate**，因为该利率是特定期限的零息债（即单个现金流）的贴现利率。所有期限的Zero Rate形成的期限结构即为一条即期利率曲线Zero Yield Cuve(简称Zero Curve)。

市场不同类型和信用等级的债券，在进行未来现金流折现时使用的利率水平是不同的，需要参考不同的Zero Curve。注意现实中这样的一条的曲线不是针对某只具体的债券，而是很多同类债券共同形成的，所以如果使用这条曲线对每只单个债券定价，并不能使得计算价格完全等于该债券的市场价。Zero Curve的拟合是另外一个要处理的问题。

Example: 某日的国开债ZeroCurve

![yc](yield_curve_guokai_180809.png)

以下演示根据ZeroCurve对测试债券计算理论价。

假设已有国开债利率曲线的几个标准期限的ZeroRate的数据点(仅用到3年期限做测试)：

|期限|即期收益率%|
|--|--|
|Spot|1.88|
|1M|1.88|
|6M|2.47|
|1Y|2.7874|
|2Y|3.3283|
|3Y|3.5334|

首先生成测试债券，以一个固息国开债为例：

```java
    System.loadLibrary("QuantLib");

    // 日历类参数：China.Market已经自带IB（银行间市场）和SSE（上交所）两个日历，具体生产使用的时候我们
    // 需要用setHolidays方法把日历的节假日设置好
    Calendar calendar = new China(China.Market.IB);
    // 全局估值日
    Date evaluationDate = new Date(9, Month.August, 2018);
    Settings.instance().setEvaluationDate(evaluationDate);
    // 结算日距离估值日时间
    int settlementDays = 0;

    // 测试债券
    String bondCode = "140203.IB";
    String bondName = "14国开03";
    Date issueDate = new Date(14, Month.January, 2017);
    Date maturityDate = new Date(14, Month.January, 2021);
    Period paymentFreq = new Period(Frequency.Annual);
    DayCounter accrualDayCounter = new ActualActual(ActualActual.Convention.Bond);
    BusinessDayConvention paymentConvention = BusinessDayConvention.Following;
    double redemption = 100.0;
    double faceValue = 100.0;
    // 例子使用的这种方法设置coupon利率
    DoubleVector couponVector = new DoubleVector();
    couponVector.add(0.0579);

    // 构造方法之一，先生产schedule，然后可以生产债券
    Schedule sch = new Schedule(issueDate, maturityDate, paymentFreq, calendar,
            BusinessDayConvention.Unadjusted,
            BusinessDayConvention.Unadjusted,
            DateGeneration.Rule.Backward,
            false);
    FixedRateBond bond = new FixedRateBond(settlementDays, faceValue, sch, couponVector,
            accrualDayCounter, paymentConvention, redemption, issueDate);
```

然后构建ZeroCurve，对债券做Pricing。ZeroCurve中的非样本点利率，采用线性插值方法。

```java
    // Building Zero Curve
    DateVector zcDates = new DateVector();
    zcDates.add(evaluationDate);
    zcDates.add(evaluationDate.add(new Period(1, TimeUnit.Months)));
    zcDates.add(evaluationDate.add(new Period(6, TimeUnit.Months)));
    zcDates.add(evaluationDate.add(new Period(12, TimeUnit.Months)));
    zcDates.add(evaluationDate.add(new Period(24, TimeUnit.Months)));
    zcDates.add(evaluationDate.add(new Period(36, TimeUnit.Months)));

    DoubleVector zcRates = new DoubleVector();
    for (double r : new double[]{1.88, 1.88, 2.47, 2.7874, 3.3283, 3.5334}) {
        zcRates.add(r/100);
    }
    // Linear作为插值方法参数传入，可替换为其他
    ZeroCurve spotCurve = new ZeroCurve(zcDates, zcRates, accrualDayCounter, calendar,
        new Linear(), Compounding.Compounded, Frequency.Annual);
    spotCurve.enableExtrapolation();

    System.out.println("Zero Curve:");
    for (int i=0; i<30; i++){
        System.out.println(rounding.getValue(0.1*i) + "\t" 
            + spotCurve.zeroRate(0.1*i, Compounding.Compounded));
    }

    // 依次生成Curve -> Curve Handle -> Pricing Engine
    RelinkableYieldTermStructureHandle curveHandle = new RelinkableYieldTermStructureHandle(spotCurve);
    DiscountingBondEngine bondEngine = new DiscountingBondEngine(curveHandle);
    bond.setPricingEngine(bondEngine);
    // 直接获得NPV(=债券全价)
    System.out.println(bond.NPV());
```

这里的curve生成的bondEngine，可以作为系统内所有国开债依赖的**全局计算引擎**，来直接计算理论值。相应的，国债有对应的国债即期收益率曲线，AAA级信用债有AAA ZeroCurve，等等。

## 2. KRD(Key Rate Duration)

接下来可以顺便带出一类风险指标的计算：KRD（关键利率久期）。

> **Key rate duration** measures the duration of a security or portfolio at a specific maturity point along the entirety of the yield curve. When keeping other maturities constant, the key rate duration can be used to measure the sensitivity in a bond's price to a 1% change in yield for a specific maturity.

简单讲就是设定一列关键期限（3个月，1年，2年...），考察利率曲线上某个关键期限的利率发生变化(一般计算100bp或1bp)而造成债券理论价的变化。一般常用的是11个关键时间点：3m, 1y, 2y, 3y, 5y, 7y, 10y, 15y, 20y, 25y, and 30y。

**计算公式：**

> $KRD=\frac{P_- - P_+}{2}\cdot\frac{1}{1\% \cdot P_0}$

$where:$

$P_0=$ 原利率曲线下的债券价格

$P_-=$ 关键利率下降100bp后债券理论价

$P_+=$ 关键利率上升100bp后债券理论价

以计算2年位置的KRD为例，Java中函数写法：
```java
    double[] keyRateYears = new double[]{0.25, 1, 2, 3, 5, 7, 10};
    DateVector keyRateDates = new DateVector();
    keyRateDates.add(evaluationDate.add(new Period(3, TimeUnit.Months)));
    SimpleQuote[] bumps = new SimpleQuote[keyRateYears.length];
    QuoteHandleVector bumpsHandle = new QuoteHandleVector();
    for (int i = 0; i < keyRateYears.length; i++) {
        if (i > 0)
            keyRateDates.add(evaluationDate.add(new Period((int) keyRateYears[i], TimeUnit.Years)));
        bumps[i] = new SimpleQuote(0.0);
        bumpsHandle.add(new QuoteHandle(bumps[i]));
    }
    SpreadedLinearZeroInterpolatedTermStructure spreadedCurve = new SpreadedLinearZeroInterpolatedTermStructure(
            new YieldTermStructureHandle(spotCurve),
            bumpsHandle,
            keyRateDates
    );
    spreadedCurve.enableExtrapolation();

    curveHandle.linkTo(spreadedCurve);
    double p_origin = bond.NPV();
    bumps[2].setValue(-0.01);
    double p_minus = bond.NPV();
    bumps[2].setValue(0.01);
    double p_plus = bond.NPV();
    double KRD2y = (p_minus - p_plus) / (0.02 * p_origin);
    System.out.println(p_origin);
    System.out.println(p_minus);
    System.out.println(p_plus);
    System.out.println("KRD of 2Y point: " + KRD2y);
```

```
Output:
108.83285089063948
110.25059876802445
107.43468245821413
KRD of 2Y point: 1.2936885723226568
```

另一个指标容易计算：

> **Key Rate Duration 01s**: Change to market value ($) due to a 1 basis point decrease in the key rate.

> $KRD 01 = (KRD * Market Value) / 10000$


(to be continued)

## 3. 利率冲击

## 4. 利率曲线拟合
