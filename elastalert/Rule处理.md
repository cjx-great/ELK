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
                self.handle_error("Error running rule %s: %s" % (rule['name'], e), {'rule': rule['name']})
            except Exception as e:
                self.handle_uncaught_exception(e, rule)
            else:
                old_starttime = pretty_ts(rule.get('original_starttime'), rule.get('use_local_time'))
                elastalert_logger.info("Ran %s from %s to %s: %s query hits (%s already seen), %s matches,"
                                       " %s alerts sent" % (
                                           rule['name'], old_starttime, pretty_ts(endtime, rule.get('use_local_time')),
                                           self.num_hits, self.num_dupes, num_matches, self.alerts_sent))
                self.alerts_sent = 0

                if next_run < datetime.datetime.utcnow():
                    # We were processing for longer than our refresh interval
                    # This can happen if --start was specified with a large time period
                    # or if we are running too slow to process events in real time.
                    logging.warning(
                        "Querying from %s to %s took longer than %s!" % (
                            old_starttime,
                            pretty_ts(endtime, rule.get('use_local_time')),
                            self.run_every
                        )
                    )

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
> 1. 先处理上次未完成的rule，即`send_pending_alerts()`。
> 2. 循环处理所有的rule，即`def run_rule(self, rule, endtime, starttime=None)`。
> 3. 移除过时的事件，即`def remove_old_events(self, rule)`。
> 4. 启动时如果没有设置 [--pin_rules](https://elastalert.readthedocs.io/en/latest/elastalert.html#running-elastalert)（`action='store_true'`），则从ES或者本地重新加载rules。