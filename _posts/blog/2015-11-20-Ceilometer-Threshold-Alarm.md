---
layout: post
title: Ceilometer中边界触发型告警器的状态更新和动作触发源码分析
description: OpenStack, Ceilometer, Threshold Alarm, evaluate, _statistics, _sufficient, _transition
category: blog
---

##文档

* [Ceilometer的alarm模块代码分析](http://www.aboutyun.com/thread-10273-1-1.html)
* [Ceilometer Distributed Alarm](http://blog.csdn.net/hackerain/article/details/38172941)
* [Ceilometer报警器状态评估之联合报警器状态评估的具体实现](http://blog.csdn.net/gaoxingnengjisuan/article/details/41631623)
* Ceilometer源码Liberty版本

##告警器(Alarm)

类Alarm的定义位于../ceilometer/api/controllers/v2/alarms.py:Alarm，Alarm的参数如下所列：

* __alarm_id__: The UUID of the alarm
* __name__: The name for the alarm
* __description__: The description of the alarm
* __enabled__: This alarm is enabled?
* __ok_actions__: The actions to do when alarm state change to ok
* __alarm_actions__: The actions to do when alarm state change to alarm
* __insufficient_data_actions__: The actions to do when alarm state change to insufficient data
* __repeat_actions__: The actions should be re-triggered on each evaluation cycle
* __type__: Explicit type specifier to select which rule to follow below,include AlarmThresholdRule,AlarmCombinationRule,etc.
* __time_constraints__: Describle time constraints for the alarm
* __project_id__: The ID of the project or tenant that owns the alarm
* __user_id__: The ID if the user who created the alarm
* __timestamp__: The date of the last alarm definition update
* __state__: The state of the alarm,include ok,insufficient data,alarm
* __state_timestamp__: The date of the last alarm state changed
* __severity__: The severity of the alarm

类AlarmThresholdRule的定义位于../ceilometer/api/controllers/v2/alarm_rules/threshold.py:AlarmThresholdRule,
AlarmThresholdRule的参数如下所列：

* __meter_name__: The name of the meter
* __query__: THe query to find the data for computing statistics. Owenership settings are automatically included based on the Alarm owner.
* __period__: The time range in seconds over which query
* __comparison_operator__: The comparison against the alarm threshold,include lt,le,eq,ne,ge,gt
* __threshold__: The threshold of the alarm
* __statistic__: The statistic to compare to the threshold,include max,min,avg,sum,count
* __evaluation_periods__: The number of historical periods to evaluate the threshold
* __exclude_outliers__: Whether datapoints with anomalously low sample counts are exculded

##原理框图
![Ceilometer Alarm](/images/2015-11-20-Ceilometer-Threshold-Alarm/2015-11-20-Ceilometer-Alarm.png)

首先，通过horizon提供的界面设置告警器规则，它会通过ceilometerclient调用相应的API把告警器规则写入数据库。Alarm Evaluator会周期性的检查用户所创建的告警器的状态，当告警器的状态发生变化时，Alarm Evaluator会先通过ceilometerclient调用相应的API更新告警器的规则，然后再通知Alarm Notifier执行该告警器定义的报警动作。Alarm Evaluator包括Alarm Threshold Evaluator、Alarm Combination Evaluator等，其中Alarm Threshold Evaluator的功能主要在../ceilometer/alarm/evaluator/threshold.py的ThresholdEvaluator.evaluate函数中实现，下面开始分析此函数。

##ThresholdEvaluator.evaluate函数

ThresholdEvaluator.evaluate函数位于../ceilometer/alarm/evaluator/threshold.py文件中，它首先会判断现在的时间是否在alarm.time_constraints规定的时间段内，然后再利用alarm中的参数构造好形式如：```ceilometer statistics -m "meter_name" -q "timestamp<=now;timestamp>=start" -p "period"```的查询语句，再调用_sufficient函数判断数据是否足够判断告警器的状态，如果数据不足以判断告警器状态，则调用_refresh函数来判断是否需要更新数据库以及通知notifier去调用相关动作；如果数据足以判断告警器状态，则会调用_transition函数，而_transition函数也会调用_refresh函数做相应处理。

* ThresholdEvaluator.evaluate函数的源码如下所示：
    
    ```
    def evaluate(self, alarm):
        if not self.within_time_constraint(alarm):
            LOG.debug('Attempted to evaluate alarm %s, but it is not within its time constraint.', alarm.alarm_id)
            return
        query = self._bound_duration(alarm, alarm.rule['query'])
        
        statistics = self._sanitize(alarm, self._statistics(alarm, query))
        
        if self._sufficient(alarm, statistics):
            def _compare(stat):
                op = COMPARATORS[alarm.rule['comparison_operator']]
                value = getattr(stat, alarm.rule['statistic'])
                limit = alarm.rule['threshold']
                LOG.debug('comparing value %(value)s against threshold %(limit)s',{'value':value, 'limit':limit})
                return op(value, limit)

            self._transition(alarm, statistics, [_compare(statistic) for statistic in statistics])
    ```

### Evaluator.within_time_constraint函数

函数self.within_time_constraint的具体实现位于../ceilometer/alarm/evaluator/```__init__.py```:Evaluator.within_time_constraint,函数within_time_constraint的作用是检查当前时间是否在alarm.time_constraints规定的时间段内,如果当前时间不在alarm.time_constraints所规定的时间段内，就返回False，否则返回True。


* Evaluator.within_time_constraint函数的源码如下所示：

    ```
    @classmethod
    def within_time_constraint(cls, alarm):
        if not alarm.time_constraints:
            return True

        now_utc = timeutils.utcnow().replace(tzinfo=pytz.utc)
        for tc in alarm.time_constraints:
            tz = pytz.timezone(tc['timezone']) if tc['timezone'] else None
            now_tz = now_utc.astimezone(tz_ if tz else now_utc
            start_cron = croniter.croniter(tc['start'], now_tz)
            if cls._is_exact_match(start_cron, now_tz):
                return True
            start_cron = croniter.croniter(tc['start'],,now_tz)
            latest_start = start_cron.get_prev(datetime.datetime)
            duration = datetime.timedelta(seconds=tc['duration'])
            if latest_start <= now_tz <= latest_start + duration:
                return True
        return False
    ```

### ThresholdEvaluator._bound_duration函数

ThresholdEvaluator._bound_duration函数位于../ceilometer/alarm/evaluator/threshold.py文件中，它的主要作用是构造后面形如：```ceilometer statistics -m "alarm.meter_name" -q "timestamp>=start;timestamp<=now" -p "AlarmThresholdRule.period"```命令中-q部分的查询条件；now就是现在的时间，start则为now-(AlarmThresholdRule.period*AlarmThresholdRule.evaluation_periods+look_back)。

* ThresholdEvaluator._boudn_duration函数的源码如下所示：

    ```
    @classmethod
    def _bound_duration(cls, alarm, constraints):
        now = timeutils.utcnow()
        look_back = (cls.look_back if not alarm.rule.get('exclude_outliers') else alarm.rule['evaluation_periods'])
        window = (alarm.rule['period'] * (alarm.rule['evaluation_periods'] + look_back))
        start = now - datetime.timedelta(seconds=window)
        LOG.debug(_('query stats from %(start)s to %(now)s) % {'start': start, 'now': now})
        after = dict(field='timestamp', op='ge', value=start.isoformat())
        before = dict(field='timestamp', op='le', value=now.isoformat())
        constraints.extend([before, after])
        return constraints
    ```

### ThresholdEvaluator._statistics函数

ThresholdEvaluator._statistics函数位于../ceilometer/alarm/evaluator/threshold.py文件中，它的主要作用就是调用Evaluator._client.statistics.list函数，并传入三个参数：AlarmThresholdRule.meter_name, 函数ThresholdEvaluator._bound_duration的返回值，AlarmThresholdRule.period。

* ThresholdEvaluator._statistics函数的源码如下所示：

    ```
    def _statistics(self, alarm, query):
        LOG.debug(_('stats query %s') % query)
        try:
            return self._client.statistics.list(
                meter_name=alarm.rule['meter_name'],q=query,
                period=alarm.rule['period'])
        except Exception:
            LOG.exception(_('alarm stats retrieval failed'))
            return []
    ```

Evaluator._client.statistics.list函数位于../ceilometerclient/v2/statistics.py:StatisticsManager.list,跟着代码一步一步找就可以找到这；从中我们可以看到这个函数在构造url：('/v2/meters/' + meter_name + '/statistics',q,p),这相当于我们通过python-ceilometerclient执行命令：ceilometer statistics -m "meter_name" -q "" -p "",而-m、-q、-p的参数都传了进来分别为：AlarmThresholdRule.meter_name, 函数ThresholdEvaluator._bound_duration的返回值,
AlarmThresholdRule.period。

* Evaluator._client.statistics.list函数的源码如下所示：

    ```
    def list(self, meter_name, q=None, period=None, groupby=None, aggregates=None):
        groupby = groupby or []
        aggregates = aggregates of []
        p = ['period=%s' % period] if period else []
        if isinstance(groupby, six.string_tryps):
            groupby = [groupby]
        p.extend(['groupby=%s' % g for g in groupby] if groupby else [])
        p.extend(self._build_aggregates(aggregates))
        return self._list(options.build_url('/v2/meters/' + meter_name + '/statistics', q, p))
    ```

### ThresholdEvaluator._sanitize函数

ThresholdEvaluator._sanitize函数位于：../ceilometer/alarm/evaluator/threshold.py文件中，它主要的作用是根据参数AlarmThresholdRule.exclude_outliers和AlarmThresholdRule.evaluation_periods来精简函数ThresholdEvaluator._statistics的返回值。

* ThresholdEvaluator._sanitize函数的源码如下所示：

    ```
    @staticmethod
    def _sanitize(alarm, statistics):
        LOG.debug(_('sanitize stats %s') % statistics)
        if alarm.rule.get('exclude_outliers'):
            key = operator.attrgetter('count')
            mean = utils.mean(statistics, key)
            stddev = utils.stddev(statistics, key, mean)
            lower = mean - 2*stddev
            upper = mean + 2*stddev
            inliers outliers = utils.anomalies(statistics, key, lower, upper)
            if outlier:
                LOG.debug(_('excluded weak datapoints with sample count %s'), [s.count for s in outliers])
                statistics = inliers
            else:
                LOG.debug('no excluded weak datapoints')

        statistics = statistics[-alarm.rule['evaluation_periods']:]
        LOG.debug(_('pruned statistics to %d') % len(statistics))
        return statistics
    ```

###  ThresholdEvaluator._sufficient函数

Threshold._sufficient函数位于../ceilometer/alarm/evaluator/threshold.py文件中，函数Threshold._statistics的返回值经过函数Threshold._sanitize的精简后得到了值statistics，而函数Threshold._sufficient的作用就是检查值statistics的长度是否大于Threshold.quorum(默认为1)，由此来判断alarm的state是否为not sufficient(UNKNOWN在前面被定义为了"not sufficient")。

* Thresholdevaluator._sufficient函数的源码如下所示：
    
    ```
    _sufficient(self, alarm, statistics):
        sufficient = len(statistics) >= self.quorum
        if not sufficient and alarm.state != evaluator.UNKNOWN:
            reason = _('%d datapoints are unknown') % alarm.rule['evaluation_periods']
            reason_data = self._reason_data('unkonwn', alarm.rule['evaluation_periods'],None)
            self._refresh(alarm, evaluator.UNKOWN, reason, reason_data, statistics)
        return sufficient
    ```

### ThresholdEvaluator._transition函数

ThresholdEvaluator._transition函数位于../ceilometer/alarm/evaluator/threshold.py文件中，函数Threshold._statistics的返回值经过函数Threshold._sanitize的精简后得到了值statistics，而函数ThresholdEvaluator._transition会将statistics与AlarmThresholdRule.threshold按AlarmThresholdRule中规定的方式做比较，根据比较结果设定alarm的状态，再根据当前状态、先前状态以及AlarmThresholdRule.repeat_actions来判断如何调用ThresholdEvaluator._refresh函数。ThresholdEvaluator._transition函数相当绕，还得仔细看下源代码。

* ThresholdEvaluator._transition函数的源码如下所示：

    ```
    def _transition(self, alarm, statistics, compared):
        distilled = all(compared)
        unequivocal = distilled or not any(compared)
        unknown = alarm.state == evaluator.UNKOWN
        continuous = alarm.repeat_actions

        if unequivocal:
            state = evaluator.ALARM if distilled else evaluator.OK
            reason, reason_date = self._reason(alarm, statistics, distilled, state)
            if alarm.state != state or continuous:
                self._refresh(alarm, state, reason, reason_date, statistics)
        elif unknown or continuous:
            trending_state = evaluator.ALARM if comapred[-1] else evaluator.OK
            state = trending_state if unknown else alarm.state
            reason, reason_data = self._reason(alarm, statistics, distilled, state)
            self._refresh(alarm, state, reason, reason_data, statistics)
    ```

### Evaluator._refresh函数

Evaluator._refresh函数位于../ceilometer/alarm/evaluator/```__init__.py```文件中，Evaluator._refresh函数有两个作用，一个是判断alarm的状态是否发生变化，当发生变化时就更改数据库中相应alarm的状态；另一个作用时通知notifier执行相关动作。

* Evaluator._refresh函数的源码如下所示:

    ```
    def _refresh(self, alarm, state, reason, reason_data):
        try:
            previous = alarm.state
            if previous != state:
                LOG.info(_('alarm %(id)s transitioning to %(state)s because %(reason)s') % {'id': alarm.alarm_id,'state':state,'reason':reason})
                self._client.alarms.set_state(alarm.alarm_id, state=state)
            alarm.state = state
            if self.notifier:
                self.notifier.notify(alarm, previous, reason, reason_data)
        except Exception:
            LOG.exception(_('alarm state update failed'))
    ```

## 最后
还有几个问题：

* notifier是如何去执行相关动作的
* alarm_history数据库是如何更新的，当一条alarm发生时，产生的alarm事如何写到alarm_history中的
* alarm的创建过程，这其中涉及到很多参数的设置，如alarm.type(AlarmThresholdRule,AlarmCombinationRule等), AlarmThreshodRule.period等，如何设置这些参数
* setup.cfg/ceilometer.alarm.rule, setup.cfg/ceilometer.alarm.evaluator, setup.cfg/ceilometer.alarm.notifier是如何加载的
