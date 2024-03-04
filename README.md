# The problem

Back in Sep 2023, after a dist-upgrade on ubuntu 22.04, nouveau started to spam my journal with an excessive number of `fifo: CACHE_ERROR` entries.

Here is one documented session that saw 168128 spam entries in 192 minutes (that's about 876 entries per minute):
```
$ sudo journalctl -b | grep 'nouveau.*fifo' | head -n1
Sep 03 12:29:02 *** kernel: nouveau 0000:00:05.0: fifo: CACHE_ERROR - ch 0 [kworker/0:0[7]] subc 2 mthd 012c data 00000000
$ 
$ sudo journalctl -b | grep 'nouveau.*fifo' | tail -n1
Sep 03 15:40:56 *** kernel: nouveau 0000:00:05.0: fifo: CACHE_ERROR - ch 0 [kworker/0:0[7]] subc 2 mthd 0130 data 00000000
$ 
$ sudo journalctl -b | grep 'nouveau.*fifo' | wc -l
168128
```

As the result, the journal became almost entirely spam and could not be viewed without `grep -v`, not to mention the annoyance of constant IO and disk noise.

Unfortunately, it seemed that the only relevant and officially supported solution for filtering the journal before saving is [PR #24058](https://github.com/systemd/systemd/pull/24058) which added `LogIncludeRegex=` and `LogExcludeRegex=` for per unit filtering, and which only came after a [lengthy discussion spanning more than five years](https://github.com/systemd/systemd/issues/6432).

This solution yet to be available on my ubuntu wasn't even going to solve the problem I was dealing with: spam coming from the kernel.

I did not want to turn off `/dev/kmsg` ingestion entirely, so I had to come up with my own solution.

# The solution

*jksf* starts early in the boot process (with the help of `jksf.service`), reads from `/dev/kmsg`, runs the filters defined in `/etc/jksf-rules.conf`, and prints out the result which then gets recorded by `journald`.

To use `jksf`:
- set `ReadKMsg=no` in `/etc/systemd/journald.conf` to prevent `journald` from ingesting `/dev/kmsg`
- place `jksf` under `/usr/local/sbin/`
- place `jksf.service` under `/etc/systemd/system/`
- write filtering rules (see below) and save them in `/etc/jksf-rules.conf`
- reboot to see `jksf` do its work

`/dev/kmsg` entries that pass through the filter should now appear in the journal as logged by "jksf", and entries filtered out are dumped into `/dev/shm/jksf-dump`. Once you are comfortable that nothing valuable is lost, you can remove the `-d /dev/shm/jksf-dump` option from the `ExecStart=/usr/local/sbin/jksf` line in `jksf.service` so that no spam gets saved even temporarily.

If you need to adjust the filtering rules, simply edit `/etc/jksf-rules.conf` and restart `jksf.service`.

# Iterative Inclusion Exclusion (IIE) Filter

IIE is a natural extension of the algorithm in [PR #24058](https://github.com/systemd/systemd/pull/24058) Since it may actually be a novel idea, some explanation is due.

There are two types of IIE rules: inclusion and exclusion.

Each rule starts with either `i` for an inclusion rule or `e` for an exclusion rule, followed by at least one space, followed by a python regex enclosed between two slashes.

As an example, my `/etc/jksf-rules.conf` for dealing with the nouveau spam problem contains just this line:
```
e /nouveau.*CACHE_ERROR/ 
```

The IIE algorithm evaluates a list of rules as follows:
- Before applying rule-0, the initial state is set to the opposite of the rule type, ie., "exclude" for an inclusion rule, and "include" otherwise;
- If before applying rule-n the state is the opposite of the rule type, it's negated iff the rule matches;
- Otherwise, rule-n is simply ignored; 
- After all the rules have been considered, the final state determines whether the item is included or excluded.






