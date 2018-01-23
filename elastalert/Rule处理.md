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
>> 1. 先处理上次未发送的alerts，即`send_pending_alerts()`。
>> 2. 循环处理所有的rule，即`def run_rule(self, rule, endtime, starttime=None)`。
>> 3. 移除过时的事件，即`def remove_old_events(self, rule)`。
>> 4. 启动时如果没有设置 [--pin_rules](https://elastalert.readthedocs.io/en/latest/elastalert.html#running-elastalert)（`action='store_true'`），则重新加载rules。

<br>
####1. 处理上次未发送的alerts  

```   
    def send_pending_alerts(self):
        pending_alerts = self.find_recent_pending_alerts(self.alert_time_limit)
        # 省略
        ...
```
首先看一下[`alert_time_limit`](http://elastalert.readthedocs.io/en/latest/running_elastalert.html#downloading-and-configuring)配置字段的意思，官方解释如下:  
![](/assets/屏幕快照 2018-01-23 下午5.53.08.png)   
接下来会调用`def find_recent_pending_alerts(self, time_limit)`根据该字段查找alerts:  

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
