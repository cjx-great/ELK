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
接下来会调用`def find_recent_pending_alerts(self, time_limit)`根据该字段查找最近未处理的alerts(在[Metadata Index](https://elastalert.readthedocs.io/en/latest/elastalert_status.html#elastalert)中查找):  

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
> > 1.匹配alert与rule（`if rule['name'] == rule_name`）,匹配成功则进行告警，不成功则保留至下一次。  
> > > 1.1 根据rule中的配置连接ES（`self.current_es = elasticsearch_client(rule)`）。  
> > > 1.2 确定当前alert是需要发送但是未发送的（`if ts_now() > ts_to_dt(alert_time)`）。  
> > > 1.3 查询所有该聚集下的告警`def get_aggregated_matches(self, _id)`。  
> > 
> > 2.告警。  
> > 3.从ES中删除该alert。

