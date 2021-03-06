# WiFi分析
## 1问题
### 背景
通过WiFi万能钥匙连接了两个免费wifi，然而网络时常不稳定，希望通过数据对时间段进行分析，实现“错峰上网”

### 提出
1哪些时间段延迟低？

2哪些时间段容易断网？

## 2数据来源
这里使用的是一款PingInfoView的软件，能自动进行ping操作并日志记录，默认格式为csv
记录时间范围为2019年9月12日-2019年11月18日，间隔为1s,文件总大小为164MB,共计1056926条数据(本人收集)
![PingInfoView](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/81179.jpg?raw=true)

## 3数据处理
![原始数据概览](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/12873.jpg?raw=true)

去除无关列

       tb = tb.get[:,['time','delay']]

初步观察数据后发现，delay列中存在数据缺失，不难发现这是丢包情况产生的
为了保持数据类型一致性，便于后续分析，将丢包情况下的延迟设为1000ms

        tb = tb.fill(1000)

        tb.info()

        RangeIndex: 1056927 entries, 0 to 1056926
        Data columns (total 2 columns):
        time    1056927 non-null object
        delay       1056927 non-null object

总共1056926条数据，数据类型为object
将time列数据类型转为datetime，新建hour列，代表每个时间段（间隔1小时）的起始时间
        
    tb.transform('datetime',
        lambda dt_str:t.parse_time(dt_str,format='%Y/%m/%d %H:%M:%S'))
        .transfer('hour','datetime',lambda dt:dt.hour)

![](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/28649.jpg?raw=true)

## 4数据分析
### 问题1
针对问题1，计算每个时间段的平均延迟，并进行排序

    tb.dtype('int',['delay']).group('hour').delay.mean().line()

![](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/38192.jpg?raw=true)

可以看到，延迟的谷点在上午9点左右，延迟较低的时间段主要集中在白天，符合一般规律
延迟的峰点在凌晨2点左右，0点-4点延迟都处于高点

另外，6点、11点、17点这些饭点较网络空闲时段延迟明显偏高(侧面反映附近人群的饭点主要集中在:6点到7点，11点到12点，17点到18点)

总体趋势是，0点以后延迟有较大增长，在2点左右延迟达到高峰，随后迅速下降
6点-9点延迟持续下降

为方便进一步分析，计算延迟变化率

    Table('gx2.pkl').dtype('int',['delay']).
        group('hour').delay.mean().pct_change().line()

![网络行为变化率](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/74496.jpg?raw=true)

可以看到，变化率谷点在凌晨4点，结合常识和统计一般性推断，附近部分人群有在3点-4点入睡的习惯

5点有一个小高峰，结合常识和统计一般性推断，附近部分人群有在5点-6点起床的习惯

变化率峰点在上午10点左右，结合常识和统计一般性推断，附近部分人群有在10点居家上网的习惯

对变化率取绝对值，这个指标在某种程度上反应用户的行为波动情况

        tb['delay'] = abs(a['delay'])
        tb.plot().show()

![网络行为波动率](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/69508.jpg?raw=true)

可以看到，最高峰值为凌晨4点和上午10点，在下午17点和晚上7点-9点有2个峰值，在凌晨2点有1个较明显的峰值，谷点在21点-22点左右

另外，在12点和17点这两个饭点，波动率都有所提升

或许是受上班影响，6点-9点波动率持续下降

可以推断，附近部分人群在凌晨4点和上午10点最为活跃，所有人群在21点-22点左右最为稳定


### 问题2
针对问题2，对每个时间段的丢包次数进行统计，并计算丢包百分比

        tb.groupby(by='hour').apply(lambda x:x['delay'].value_counts(normalize=True)[1000])
        .sort_values(ascending=True).bar()

![断线率](https://github.com/heinz-lxy/data-analysis/blob/master/1.WiFi%E5%88%86%E6%9E%90/images/83710.jpg?raw=true)

可以观察到，丢包和延迟率的时间分布基本相同
0点以后丢包率明显提升

## 5总结
1应该尽量选择在白天上网。其中，低延迟的最佳时间段是9点-10点，此时适合浏览网页、查询资料

2强烈不建议开夜车

3尽可能避开饭点：6点到7点，11点到12点，17点到18点




