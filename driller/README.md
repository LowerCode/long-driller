## Driller

Driller is an implementation of the [driller paper](https://www.internetsociety.org/sites/default/files/blogs-media/driller-augmenting-fuzzing-through-selective-symbolic-execution.pdf). This implementation was built on top of AFL with angr being used as a symbolic tracer. Driller selectively traces inputs generated by AFL when AFL stops reporting any paths as 'favorites'. Driller will take all untraced paths which exist in AFL's queue and look for basic block transitions AFL failed to find satisfying inputs for. Driller will then use angr to synthesize inputs for these basic block transitions and present it to AFL for syncing. From here, AFL can determine if any paths generated by Driller are interesting, it will then go ahead and mutate these as normal in an attempt to find more paths.

### The "Stuck" heuristic

Driller's symbolic execution component is invoked when AFL is 'stuck'. 
In this implementation, AFL's progress is determined by its 'pending\_favs' attribute which can found in the fuzzer\_stats file. 
When this attribute reaches 0, Driller is invoked. Other heuristics could also be used, and it's infact likely that better heuristics exist.

### Use in the Cyber Grand Challenge

This same implementation of Driller was used team Shellphish in DARPA's Cyber Grand Challenge (CGC) to aid in the discovery of exploitable bugs.
To see how Driller's invokation was scheduled for the CGC you can look at the Mechanical Phish's scheduler component 'meister'.

### Current State and Caveats

The code currently supports three modes of operation:

+ A script that facilitates AFL and driller on one machine (over many cores if needed): https://github.com/shellphish/fuzzer/blob/master/shellphuzz
+ A monitor process watches over the fuzzer\_stats file to determine when Driller should be invoked. When Driller looks like it could be useful, the monitor process schedules 'jobs' to work over all the inputs AFL has discovered / deemed interesting.
+ Celery tasks are assigned over a fleet of machines, some number of these tasks are assigned to fuzzing, some are assigned to drilling. Fuzzer tasks monitors the stats file, and invokes driller tasks when Driller looks like it could be useful. Redis is used to sync testcases to the filesystem of the fuzzer.

Driller was built and developed for DECREE binaries.
While some support for other formats should work out-of-the-box, expect `TracerMisfollowError`s to occur when unsupported or incorrectly implemented simprocedures are hit.

### Example

Here is an example of using driller to find new testcases based off the trace of a single testcase.

```python
import driller

d = driller.Driller("./CADET_00001",  # path to the target binary
                    "racecar", # initial testcase
                    "\xff" * 65535, # AFL bitmap with no discovered transitions
                   )

new_inputs = d.drill()
```

#### Dependencies

+ Mechaphish Fuzzer component
+ Mechaphish Tracer component