# 100 Days Of Code - Log

### Day 1: June 28, 2024 (Friday)

**Hour 1 Progress**: Browsed the osquery issues list. Started with macOS. Wish I could work on [8220](https://github.com/osquery/osquery/issues/8220) but this is a TCC rabbit hole issue, which means testing entitlements and re-signing the executables, which I can't do. Location Services is apparently now the only way a macOS application can request SSID information. I get it -- that's a proxy for your location, and ought to be private. But how would the osquery daemon even prompt the user? It would have to start pre-authorized with the entitlement. https://developer.apple.com/forums/thread/739712#768907022 Ok so is osqueryd a launch "agent" or a "daemon"? I left a comment in the issue.

**Hour 2 Progress**: All right how do I build osquery again? [Here](https://osquery.readthedocs.io/en/latest/development/building/#macos). The first build took >2 hours and >100% of my laptop battery. LOL when this 1-hour-a-day coding challenge was envisioned they were not even thinking about compiled languages. Oh hey, issue [7750](https://github.com/osquery/osquery/issues/7750) looks easy, just remove an old table column alias. But wait, this column is part of the schema for every OS not just macOS. "total_size" is intended to be "Total virtual memory size" of a process. Is osquery/specs/processes.table accurate when it says implementation("system/processes@genProcesses")? Because it's clearly in three places: osquery/tables/system/windows/processes.cpp and osquery/tables/system/linux/processes.cpp and osquery/tables/system/darwin/processes.cpp. The relevant field seems to be generated by [genProcResourceUsage()](https://github.com/osquery/osquery/blob/ca14540bd01ce792b1fc43add8502059ac71835c/osquery/tables/system/darwin/processes.cpp#L450). It originates from the ri_phys_footprint field of a rusage_info_v2 struct returned by the proc_pid_rusage() API of [libproc](https://opensource.apple.com/source/xnu/xnu-2422.1.72/libsyscall/wrappers/libproc/libproc.h.auto.html). "This header file contains private interfaces to obtain process information." lmao osquery is not supposed to do this. Is it a supported API? People seem to keep using it https://stackoverflow.com/questions/66328149/the-cpu-time-obtained-by-proc-pid-rusage-does-not-meet-expectations-on-the-macos From investigating, it looks like everyone uses it and that there are no public interface alternatives. Even the Apple representative acknowledges the problems with this: https://developer.apple.com/forums/thread/46963

Whereas, on Linux, it is the `total_size` field of a SimpleProcStat struct, which is something filled with its constructor, it's defined in osquery. Output from string parsing /proc/<pid>/status. That's it. In procfs it's `VmSize`.

**Hour 3 Progress**: I wonder if the osquery processes table is actually measuring physical RAM usage or virtual memory size? Activity Monitor should be a cross-check. Also `ps -e -o pid,rss,vsz` or `ps -xm -o %mem,rss,comm -p $(pgrep Notes)` and `vm_stat` commands. Also `vmmap --summary "$pid"` But there is now Physical Memory, Wired Memory, and Compressed Memory??

    UGH, well one thing I just learned about memory is that Safari will lose the text I'm drafting in a long GitHub issue comment!

**Thoughts:** Directions state to use Twitter and subscribe on Substack, I won't be doing that. There's a Slack, I'll check that out...oh no the invite link is dead and they expect me to use Twitter to ask for a new one. Guess I'll do this on my own. I'm also making it my own, because as it was originally envisioned this whole thing has the smell of Learn to Code Bootcamp JavaScript in 28 Days How to Land a Career in Tech self-help.

    One of the fundamental problems with osquery is having to maintain something with so much code that is so close to the operating system that OS updates are constantly creating breaking changes. And there are multiple operating systems it supports.

**Link to work:** I will be working here on  [osquery](https://github.com/michael-myers/osquery)

### Day 2: June 29, 2024 (Saturday)

**Hour 1 Progress**: Since updating my fork of osquery, my GitHub Actions have been failing and flooding my email. So I went to go turn off GitHub Actions in my account, but I couldn't find a toggle, so I deleted .github/workflows/ from my fork of the repo.

    All right, let's determine what osquery calls "total virtual memory" versus the other tools.
    Activity Monitor: Notes.app: Memory 377.2MB, Real Mem 219.0MB, Private Mem 161.0MB, Shared Mem 180.1MB, Purgeable Mem 19.2MB, VM Compressed 208.2MB. The compressed amount seems to mean the savings, not the resulting occupied memory size. In the lower panel, "Wired Memory" is that used by macOS itself (cannot be compressed) and "App Memory" is everything used by applications, which can be. "Real Memory" is also known as resident_size in task_info()"? and although it includes memory shared with other processes, I guess it is a rough guide? The Private Mem is the part of Real Mem that is _not_ shared. Purgeable Mem can be tossed if memory is tight, it is never written to swap.
    ps: Notes.app: %MEM 1.3, RSS 224296, VSZ 35440844. RSS is the "real memory (resident set) size of the process, in 1024-byte units" -- so 219.0MB, which matches up with Activity Monitor. VSZ or VSIZE is the virtual size in Kbytes. What? that seems too big? %MEM is mostly not what I want, it is the percentage of memory usage.
    vm_stat: this can't be run on a single process, so is not of use here.
    vmmap --summary $pid: ah, now we get "Physical footprint" is 377.3MB, so this is what Activity Monitor calls "Memory". It also shows Physical Footprint (peak) is 639.6M. In the table it prints, VOLATILE SIZE total matches up with Activity Monitor's "Purgeable Mem." Its man page notes that "The size of the virtual memory region represents the virtual memory pages reserved, but not necessarily allocated.  For example, using the vm_allocate Mach system call reserves the pages, but physical memory won't be allocated for the page until the memory is actually touched.  A memory-mapped file may have a virtual memory page reserved, but the pages are not instantiated until a read or write happens.  Thus, this size may not correctly describe the application's true memory usage.""
    Ok, so do we care about "Memory" (Physical Footprint) or "Real Memory"?? "Real Mem" is supposedly the RSS, which is the amount of physical memory that a process is actively using.
    top -o mem: we have MEM, which is apparently "Memory" and is described as "Physical memory footprint of the process", "PURG" which is apparently "Purgeable Mem", "CMPRS" which is VM Compressed. If it shows VSIZE, that is "Total memory size" but I only see N/A in that column.
    Mac OSX Developer Tool, "Big Top": ??
    Does anyone actually see yellow or red memory pressure in Activity Monitor? With what? I am green with 16GB.
    osquery returns these numbers:
    osquery> select name,phys_footprint,total_size,wired_size,resident_size from processes where name is "Notes";
+-------+----------------+------------+------------+---------------+
| name  | phys_footprint | total_size | wired_size | resident_size |
+-------+----------------+------------+------------+---------------+
| Notes |                | 395317248  | 4096       | 226889728     |
+-------+----------------+------------+------------+---------------+
  Which, is in bytes, so the `total_size` is once again 377MB.

**Hour 2 Progress**: I'm thinking of making a venn diagram of this macOS memory types nonsense. "It is not possible to add up shared and private memory to arrive at the physical memory used by a process." What? Indeed, the "Real Private" and "Real Shared" columns add up to more than the "Real Memory" column in Activity Monitor.
    Whoa, there's another CLI command: footprint.
    footprint -p Notes: yea it is 377MB, the same as "Memory" in Activity Monitor.
    Look at this gold nugget from the man page of footprint:
    "Many operating systems have historically exposed memory metrics such as Virtual Size (VSIZE) and Resident Size (RSIZE/RPRVT/RSS/etc.). Metrics such as these, which are
     useful in their own respect, are not great indicators of the amount of physical memory required by a process to run (and therefore the memory pressure that a process
     applies to the system). For instance, Virtual Size includes allocations that may not be backed by physical memory, and Resident Size includes clean and volatile
     purgeable memory that can be reclaimed by the kernel (as described earlier).
     On the other hand, analyzing the dirty memory reported by the underlying VM objects mapped into a process (the approach taken by --vmObjectDirty), while more accurate,
     is expensive and cannot be done in real-time for systems that need to frequently know the memory footprint of a process.

     Apple platforms instead keep track of the 'physical footprint' by using a per-process ledger in the kernel that is kept up-to-date by the pmap and other subsystems.
     This ledger is cheap to query, suitably accurate, and provides additional features such as tracking peak memory and the ability to charge one process for memory that is
     no longer mapped into it or that may have been allocated by another process. In cases where footprint is unable to analyze a portion of 'physical footprint' that is not
     mapped into a process, this memory will be listed as 'Owned physical footprint (unmapped)'. If this memory is mapped into another userspace process then the --unmapped
     argument can be used to search all processes for a mapping of the same VM object, which if found will provide a better description and what process(s) have mapped the
     memory. This also happens by default when targeting all processes via --all.  Any memory still listed as "(unmapped)" after using --unmapped is likely not mapped into
     any userspace process and instead only referenced by the kernel or drivers.
     The exact definition of this 'physical footprint' ledger is complicated and subject to change, but suffice it to say that the default mode of footprint aims to present
     an accurate memory breakdown that matches the value reported by the ledger. Most other diagnostic tools, such as the 'MEM' column in top(1), the 'Memory' column in
     Activity Monitor.app, and the Memory Debug Gauge in Xcode.app, query this ledger to populate their metrics.

     Physical footprint can be potentially be split into multiple subcategories, such as network related memory, graphics, etc. When a memory allocation (either directly
     mapped into a process, or owned but unmapped) has such a classification, footprint will append it to the category name such as 'IOKit (graphics)' or 'Owned physical
     footprint (unmapped) (media)'."
     
     Ended this hour with a comment on how we can resolve issue 7750 on osquery.

**Hour 3 Progress**: Just reading a bunch about macOS memory metrics.

**Thoughts:** So, Activity Monitor reports "physical footprint" which is not a measure of any real number of bytes of occpuied physical RAM, but calls it "Memory". Then it reports "Real Memory" which ostensibly is the opposite of "Virtual Memory" which it does not report at all, and is not even a column. Then it reports two kinds of real memory, shared and private, which do not add up to equal the Real Memory column. And it reports "Compressed" which is not the size after compression, but rather the delta or savings of the compression???? You have to be fucking kidding me. But today has been a bit of a lesson on memory metrics I guess.

    Normally I trust the macOS technical blog eclecticlight.co, but I don't think he really understands virtual memory or memory management as well as he thinks he does. For a couple of years he's been accusing Finder of having a memory leak, when all it might be doing is not freeing allocations when it doesn't have to, when there's so much RAM that holding onto them might increase responsiveness compared to having to flush and free caches just to make a number go down in Activity Monitor. You want RAM to be used for caching disk! Did he even look at whether the allocations were "purgeable memory"? He admits "I have absolutely no idea what Apple means by Real Memory" [1](https://eclecticlight.co/2022/01/19/memory-lane-grokking-memory-problems-in-activity-monitor/) [2](https://eclecticlight.co/2022/08/10/getting-more-from-activity-monitor-memory/) [3 here he fails to correctly explain the memory reporting columns, in the comments](https://eclecticlight.co/2022/08/13/activity-monitor-meanings-and-misleadings/) [4 omg he is still talking about this Finder results-caching behavior](https://eclecticlight.co/2023/06/06/what-to-do-when-an-app-uses-too-much-memory/)

     People do like to complain about a single Safari tab showing 300MB or 1GB of memory, and that's justifiable to complain about, but also how much of it basically disappears due to virtual memory compression when you're not using that tab? How much of it is Shared Memory because it's using a bunch of libraries that all of the other tabs are also using?
     
     It would really be nice if there was a Memory resource viewer that sorted processes hierarchically but also totaled up the subprocesses of a process. As-is, you can't tell what the sum total of, e.g., your browser is on memory usage.

**Resources:**
 
 * https://ladedu.com/how-to-interpret-memory-in-mac-activity-monitor/
 * `man leaks` which explains how to search a process's memory for unreferenced malloc buffers, before assuming that all memory usage increase is a leak, like Howard does.
 * Small web blogs are always the best for facts you can't find anywhere else! https://www.mikeash.com/pyblog/friday-qa-2009-06-19-mac-os-x-process-memory-statistics.html

**Link to work:** [osquery issue 7750](https://github.com/osquery/osquery/issues/7750)

### Day 3: June 30, 2024 (Sunday)

What if I just moved 'status' to the Linux-only extended schema, then?
https://github.com/osquery/osquery/issues/8073
It's so simple, but nobody has replied. So, what's the deal?

Is this another TCC problem that requires shelling out from an extension?
`osquery> select * from device_partitions where device is "/dev/disk0";` returns nothing. But so does "/dev/disk1" which is "synthesized" and "/dev/disk2" which is "external" ... so is it not just 'internal' disks but all? Or am I supplying the wrong constraint? No it appears it just doesn't work.
https://github.com/osquery/osquery/issues/7777
Also, this is one of three tables (along with device_table, and device_hash) that uses The Sleuth Kit (TSK) dependency to get results. Is this dependency worth maintaining? These don't seem like results you'd need a dependency to retrieve?

What if the schema comments were more descriptive? Like for processes, where "total_size" is a differently derived value on macOS versus Linux.

What if the schema didn't have missing comments for columns? Like in device_partitions.
https://osquery.io/schema/5.12.1/#device_partitions

This seems like it could be answered and easily closed?
https://github.com/osquery/osquery/issues/7167

This seems like it should just be closed as fixed-as-it-will-get?
https://github.com/osquery/osquery/issues/7220

This should be closed with a schema comment that this is only a stat for ethernet interfaces that are connected:
https://github.com/osquery/osquery/issues/6595

"Good first issue" lol it has been open since 2019. Just need to convert some numbers better, I bet we can do that. "we are missing the flt type" says Stefano, and the fix should be in smc_keys which should just produce the string representation that power_sensors expects. https://github.com/osquery/osquery/blob/4f78848794cd2df1532b9c8d7e3fe4b17dde3e2a/osquery/tables/system/darwin/smc_keys.cpp#L381-L404
https://github.com/osquery/osquery/issues/5896

**Thoughts:** a macOS daemon requires so many entitlements now that it is becoming untenable to use the APIs. More and more things require shelling out to a CLI command from an Apple executable that actually has the entitlement.

### Day 4: July 1, 2024 (Monday)

**Hour 1 Progress**:
Update the table spec for processes.
Did the alias point to something in the extended schema that was removed? Or why does the alias not work??
Also update docs/wiki/introduction/sql.md
Also update specs/posix/docker_container_processes.table
A rebuild, with only these minor changes, took less time than the initial build: 6m45s.
Make a git branch with just these changes, push it to my fork, open a pull request to close issue 7750

**Hour 2 Progress**:
All right if I am going to add table spec column comments for device_paritions, I need to read osquery/tables/sleuthkit/sleuthkit.cpp.
"Label" is the partition description, pretty straightforward.
"Flags" is interesting since it's exposing an internal enum of The Sleuthkit, but okay.
Etc.

Oh this would be easy to fix, I'll go ahead and do that (documentation)
https://github.com/osquery/osquery/issues/7392

**Thoughts:** I see there's a lot of resentment that https://github.com/osquery/osquery/issues/7627 the temperature sensors table still doesn't work on Apple Silicon macs (which is basically all of them now). I did some of that initial investigation in 2022, but now I don't have Apple Silicon to work on.

    I see that I could squash some compiler warnings for security: "warning: 'sprintf' is deprecated: This function is provided for compatibility reasons only.  Due to security concerns inherent in the design of sprintf(3), it is highly recommended that you use snprintf(3) instead"
    
    Fixing more of the schema on the website should be easy: https://github.com/osquery/osquery/issues/7198

    Is this the spirit of 100 Days of Code? Analyzing issues and updating metadata like docs and schemas? 

**Links to work:** 
* https://github.com/osquery/osquery/pull/8363
* https://github.com/osquery/osquery/pull/8364
* https://github.com/osquery/osquery/pull/8365
