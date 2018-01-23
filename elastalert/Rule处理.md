```
    def run_all_rules(self):
        """ Run each rule one time """
        self.send_pending_alerts()

        next_run = datetime.datetime.utcnow() + self.run_every

        for rule in self.rules:
            rule['writeback_es'] =  self.writeback_es
            rule['alerts_send_index'] =  self.alerts_send_index
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
            if self.conf['rules_in_es']:
                self.load_rule_changes_from_es()
            else:
                self.load_rule_changes()
```

大体流程：  
>> 1. 先处理上次未发送的alerts，即`send_pending_alerts()`。
>> 2. 循环处理所有的rule，即`def run_rule(self, rule, endtime, starttime=None)`。
>> 3. 移除过时的事件，即`def remove_old_events(self, rule)`。
>> 4. 启动时如果没有设置 [--pin_rules](https://elastalert.readthedocs.io/en/latest/elastalert.html#running-elastalert)（`action='store_true'`），则从ES或者本地重新加载rules。

<br>
####1. 处理上次未发送的alerts  

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
                    self.alert([match_body], rule, alert_time=alert_time)

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
                except:  # TODO: Give this a more relevant exception, try:except: is evil.
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
首先看一下[`alert_time_limit`](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#downloading-and-configuring)配置字段的意思，官方解释如下:  
![](/assets/屏幕快照 2018-01-23 下午5.53.08.png)   
接下来将会根据该字段查找alerts。
