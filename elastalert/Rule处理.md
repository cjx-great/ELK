```
    def run_all_rules(self):
        """ Run each rule one time """
        self.send_pending_alerts()

        next_run = datetime.datetime.utcnow() + self.run_every

        for rule in self.rules:
            # Set endtime based on the rule's delay
            delay = rule.get('query_delay')
            if hasattr(self.args, 'end') and self.args.end:
                endtime = ts_to_dt(self.args.end)
            elif delay:
                endtime = ts_now() - delay
            else:
                endtime = ts_now()

            try:
                num_matches = self.run_rule(rule, endtime, self.starttime)
            except EAException as e:
                ...
                # 省略部分代码
                ...

            self.remove_old_events(rule)

        # Only force starttime once
        self.starttime = None

        if not self.args.pin_rules:
            self.load_rule_changes()
```

大体流程：  
>> 1. 先处理尚未发送的alerts，即`self.send_pending_alerts()`。
>> 2. 循环处理所有的rule，即`self.run_rule(rule, endtime, self.starttime)`。
>> 3. 移除过时的事件，即`self.remove_old_events(rule)`。
>> 4. 启动时如果没有设置 [--pin_rules](https://elastalert.readthedocs.io/en/latest/elastalert.html#running-elastalert)（`action='store_true'`），则`self.load_rule_changes()`。

<br>
####1. 处理尚未发送的alerts  

```   
    def send_pending_alerts(self):
        pending_alerts = self.find_recent_pending_alerts(self.alert_time_limit)
        # 省略
        ...
```
首先看一下[`alert_time_limit`](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#downloading-and-configuring)配置字段的意思，官方解释如下:  
![](/assets/屏幕快照 2018-01-23 下午5.53.08.png)   
接下来会调用`def find_recent_pending_alerts(self, time_limit)`根据该字段查找最近未处理的alerts(在[Metadata Index](https://elastalert.readthedocs.io/en/latest/elastalert_status.html#elastalert)中查找)，其中包括上次未发送的alert和最新的alert，时间以`time_limit `为基准:  

```
    def find_recent_pending_alerts(self, time_limit):
        """ Queries writeback_es to find alerts that did not send
        and are newer than time_limit """

        # XXX only fetches 1000 results. If limit is reached, next loop will catch them
        # unless there is constantly more than 1000 alerts to send.

        # Fetch recent, unsent alerts that aren't part of an aggregate, earlier alerts first.
        inner_query = {'query_string': {'query': '!_exists_:aggregate_id AND alert_sent:false'}}
        time_filter = {'range': {'alert_time': {'from': dt_to_ts(ts_now() - time_limit),
                                                'to': dt_to_ts(ts_now())}}}
        sort = {'sort': {'alert_time': {'order': 'asc'}}}
        if self.is_five():
            query = {'query': {'bool': {'must': inner_query, 'filter': time_filter}}}
        else:
            query = {'query': inner_query, 'filter': time_filter}
        query.update(sort)
        try:
            res = self.writeback_es.search(index=self.writeback_index,
                                           doc_type='elastalert',
                                           body=query,
                                           size=1000)
            if res['hits']['hits']:
                return res['hits']['hits']
        except ElasticsearchException as e:
            logging.exception("Error finding recent pending alerts: %s %s" % (e, query))
        return []
```
重点在于`inner_query`和`time_filter`变量的计算，其中`doc_type='elastalert'`的官方解释如下：  
![](/assets/屏幕快照 2018-01-24 上午10.46.20.png)
<br>
回到`def send_pending_alerts(self)`方法：

```
    def send_pending_alerts(self):
        pending_alerts = self.find_recent_pending_alerts(self.alert_time_limit)
        for alert in pending_alerts:
            _id = alert['_id']
            alert = alert['_source']
            try:
                rule_name = alert.pop('rule_name')
                alert_time = alert.pop('alert_time')
                match_body = alert.pop('match_body')
            except KeyError:
                # Malformed alert, drop it
                continue

            # Find original rule
            for rule in self.rules:
                if rule['name'] == rule_name:
                    break
            else:
                # Original rule is missing, keep alert for later if rule reappears
                continue

            # Set current_es for top_count_keys query
            self.current_es = elasticsearch_client(rule)
            self.current_es_addr = (rule['es_host'], rule['es_port'])

            # Send the alert unless it's a future alert
            if ts_now() > ts_to_dt(alert_time):
                aggregated_matches = self.get_aggregated_matches(_id)
                if aggregated_matches:
                    matches = [match_body] + [agg_match['match_body'] for agg_match in aggregated_matches]
                    self.alert(matches, rule, alert_time=alert_time)
                else:
                    # If this rule isn't using aggregation, this must be a retry of a failed alert
                    retried = False
                    if not rule.get('aggregation'):
                        retried = True
                    self.alert([match_body], rule, alert_time=alert_time, retried=retried)

                if rule['current_aggregate_id']:
                    for qk, agg_id in rule['current_aggregate_id'].iteritems():
                        if agg_id == _id:
                            rule['current_aggregate_id'].pop(qk)
                            break

                # Delete it from the index
                try:
                    self.writeback_es.delete(index=self.writeback_index,
                                             doc_type='elastalert',
                                             id=_id)
                except ElasticsearchException:  # TODO: Give this a more relevant exception, try:except: is evil.
                    self.handle_error("Failed to delete alert %s at %s" % (_id, alert_time))

        # Send in memory aggregated alerts
        for rule in self.rules:
            if rule['agg_matches']:
                for aggregation_key_value, aggregate_alert_time in rule['aggregate_alert_time'].iteritems():
                    if ts_now() > aggregate_alert_time:
                        alertable_matches = [
                            agg_match
                            for agg_match
                            in rule['agg_matches']
                            if self.get_aggregation_key_value(rule, agg_match) == aggregation_key_value
                        ]
                        self.alert(alertable_matches, rule)
                        rule['agg_matches'] = [
                            agg_match
                            for agg_match
                            in rule['agg_matches']
                            if self.get_aggregation_key_value(rule, agg_match) != aggregation_key_value
                        ]
```
大体流程：
> > 1.&nbsp;匹配alert与rule（`if rule['name'] == rule_name`）,匹配成功则进行告警，不成功则保留至下一次。  
> > > 1.1 &nbsp;根据rule中的配置连接ES（`self.current_es = elasticsearch_client(rule)`）。  
> > > 1.2 &nbsp;确定当前alert是需要发送但是未发送的（`if ts_now() > ts_to_dt(alert_time)`）。  
> > > 1.3 &nbsp;查询所有该聚集下的告警`def get_aggregated_matches(self, _id)`。  
> > 
> > 2.&nbsp;告警。  
> > 3.&nbsp;从ES中删除该alert。  
> > 4.&nbsp;发送“聚合”告警。  

<br>
####2. 循环处理所有的rule  
```
def run_rule(self, rule, endtime, starttime=None):
        """ Run a rule for a given time period, including querying and alerting on results.

        :param rule: The rule configuration.
        :param starttime: The earliest timestamp to query.
        :param endtime: The latest timestamp to query.
        :return: The number of matches that the rule produced.
        """
        run_start = time.time()

        self.current_es = elasticsearch_client(rule)
        self.current_es_addr = (rule['es_host'], rule['es_port'])

        # If there are pending aggregate matches, try processing them
        for x in range(len(rule['agg_matches'])):
            match = rule['agg_matches'].pop()
            self.add_aggregated_alert(match, rule)

        # Start from provided time if it's given
        if starttime:
            rule['starttime'] = starttime
        else:
            self.set_starttime(rule, endtime)

        rule['original_starttime'] = rule['starttime']

        # Don't run if starttime was set to the future
        if ts_now() <= rule['starttime']:
            logging.warning("Attempted to use query start time in the future (%s), sleeping instead" % (starttime))
            return 0

        # Run the rule. If querying over a large time period, split it up into segments
        self.num_hits = 0
        self.num_dupes = 0
        segment_size = self.get_segment_size(rule)

        tmp_endtime = rule['starttime']

        while endtime - rule['starttime'] > segment_size:
            tmp_endtime = tmp_endtime + segment_size
            if not self.run_query(rule, rule['starttime'], tmp_endtime):
                return 0
            rule['starttime'] = tmp_endtime
            rule['type'].garbage_collect(tmp_endtime)

        if rule.get('aggregation_query_element'):
            if endtime - tmp_endtime == segment_size:
                self.run_query(rule, tmp_endtime, endtime)
            elif total_seconds(rule['original_starttime'] - tmp_endtime) == 0:
                rule['starttime'] = rule['original_starttime']
                return 0
            else:
                endtime = tmp_endtime
        else:
            if not self.run_query(rule, rule['starttime'], endtime):
                return 0
            rule['type'].garbage_collect(endtime)

        # Process any new matches
        num_matches = len(rule['type'].matches)
        while rule['type'].matches:
            match = rule['type'].matches.pop(0)
            match['num_hits'] = self.num_hits
            match['num_matches'] = num_matches

            # If realert is set, silence the rule for that duration
            # Silence is cached by query_key, if it exists
            # Default realert time is 0 seconds
            silence_cache_key = rule['name']
            query_key_value = self.get_query_key_value(rule, match)
            if query_key_value is not None:
                silence_cache_key += '.' + query_key_value

            if self.is_silenced(rule['name'] + "._silence") or self.is_silenced(silence_cache_key):
                elastalert_logger.info('Ignoring match for silenced rule %s' % (silence_cache_key,))
                continue

            if rule['realert']:
                next_alert, exponent = self.next_alert_time(rule, silence_cache_key, ts_now())
                self.set_realert(silence_cache_key, next_alert, exponent)

            if rule.get('run_enhancements_first'):
                try:
                    for enhancement in rule['match_enhancements']:
                        try:
                            enhancement.process(match)
                        except EAException as e:
                            self.handle_error("Error running match enhancement: %s" % (e), {'rule': rule['name']})
                except DropMatchException:
                    continue

            # If no aggregation, alert immediately
            if not rule['aggregation']:
                self.alert([match], rule)
                continue

            # Add it as an aggregated match
            self.add_aggregated_alert(match, rule)

        # Mark this endtime for next run's start
        rule['previous_endtime'] = endtime

        time_taken = time.time() - run_start
        # Write to ES that we've run this rule against this time period
        body = {'rule_name': rule['name'],
                'endtime': endtime,
                'starttime': rule['original_starttime'],
                'matches': num_matches,
                'hits': self.num_hits,
                '@timestamp': ts_now(),
                'time_taken': time_taken}
        self.writeback('elastalert_status', body)

        return num_matches
```


