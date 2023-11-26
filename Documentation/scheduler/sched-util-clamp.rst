.. SPDX-License-Identifier: GPL-2.0

====================
Utilization Clamping
====================

1. Introduction
===============

Utilization clamping, also known as util clamp or uclamp, is a scheduler
feature that allows user space to help in managing the performance requirement
of tasks. It was introduced in v5.3 release. The CGroup support was merged in
v5.4.

Uclamp is a hinting mechanism that allows the scheduler to understand the
performance requirements and restrictions of the tasks, thus it helps the
scheduler to make a better decision. And when schedutil cpufreq governor is
used, util clamp will influence the CPU frequency selection as well.

Since the scheduler and schedutil are both driven by PELT (util_avg) signals,
util clamp acts on that to achieve its goal by clamping the signal to a certain
point; hence the name. That is, by clamping utilization we are making the
system run at a certain performance point.

The right way to view util clamp is as a mechanism to make request or hint on
performance constraints. It consists of two tunables:

        * UCLAMP_MIN, which sets the lower bound.
        * UCLAMP_MAX, which sets the upper bound.

These two bounds will ensure a task will operate within this performance range
of the system. UCLAMP_MIN implies boosting a task, while UCLAMP_MAX implies
capping a task.

One can tell the system (scheduler) that some tasks require a minimum
performance point to operate at to deliver the desired user experience. Or one
can tell the system that some tasks should be restricted from consuming too
much resources and should not go above a specific performance point. Viewing
the uclamp values as performance points rather than utilization is a better
abstraction from user space point of view.

As an example, a game can use util clamp to form a feedback loop with its
perceived Frames Per Second (FPS). It can dynamically increase the minimum
performance point required by its display pipeline to ensure no frame is
dropped. It can also dynamically 'prime' up these tasks if it knows in the
coming few hundred milliseconds a computationally intensive scene is about to
happen.

On mobile hardware where the capability of the devices varies a lot, this
dynamic feedback loop offers a great flexibility to ensure best user experience
given the capabilities of any system.

Of course a static configuration is possible too. The exact usage will depend
on the system, application and the desired outcome.

Another example is in Android where tasks are classified as background,
foreground, top-app, etc. Util clamp can be used to constrain how much
resources background tasks are consuming by capping the performance point they
can run at. This constraint helps reserve resources for important tasks, like
the ones belonging to the currently active app (top-app group). Beside this
helps in limiting how much power they consume. This can be more obvious in
heterogeneous systems (e.g. Arm big.LITTLE); the constraint will help bias the
background tasks to stay on the little cores which will ensure that:

        1. The big cores are free to run top-app tasks immediately. top-app
           tasks are the tasks the user is currently interacting with, hence
           the most important tasks in the system.
        2. They don't run on a power hungry core and drain battery even if they
           are CPU intensive tasks.

.. note::
  **little cores**:
    CPUs with capacity < 1024

  **big cores**:
    CPUs with capacity = 1024

By making these uclamp performance requests, or rather hints, user space can
ensure system resources are used optimally to deliver the best possible user
experience.

Another use case is to help with **overcoming the ramp up latency inherit in
how scheduler utilization signal is calculated**.

On the other hand, a busy task for instance that requires to run at maximum
performance point will suffer a delay of ~200ms (PELT HALFIFE = 32ms) for the
scheduler to realize that. This is known to affect workloads like gaming on
mobile devices where frames will drop due to slow response time to select the
higher frequency required for the tasks to finish their work in time. Setting
UCLAMP_MIN=1024 will ensure such tasks will always see the highest performance
level when they start running.

The overall visible effect goes beyond better perceived user
experience/performance and stretches to help achieve a better overall
performance/watt if used effectively.

User space can form a feedback loop with the thermal subsystem too to ensure
the device doesn't heat up to the point where it will throttle.

Both SCHED_NORMAL/OTHER and SCHED_FIFO/RR honour uclamp requests/hints.

In the SCHED_FIFO/RR case, uclamp gives the option to run RT tasks at any
performance point rather than being tied to MAX frequency all the time. Which
can be useful on general purpose systems that run on battery powered devices.

Note that by design RT tasks don't have per-task PELT signal and must always
run at a constant frequency to combat undeterministic DVFS rampup delays.

Note that using schedutil always implies a single delay to modify the frequency
when an RT task wakes up. This cost is unchanged by using uclamp. Uclamp only
helps picking what frequency to request instead of schedutil always requesting
MAX for all RT tasks.

See :ref:`section 3.4 <uclamp-default-values>` for default values and
:ref:`3.4.1 <sched-util-clamp-min-rt-default>` on how to change RT tasks
default value.

2. Design
=========

Util clamp is a property of every task in the system. It sets the boundaries of
its utilization signal; acting as a bias mechanism that influences certain
decisions within the scheduler.

The actual utilization signal of a task is never clamped in reality. If you
inspect PELT signals at any point of time you should continue to see them as
they are intact. Clamping happens only when needed, e.g: when a task wakes up
and the scheduler needs to select a suitable CPU for it to run on.

Since the goal of util clamp is to allow requesting a minimum and maximum
performance point for a task to run on, it must be able to influence the
frequency selection as well as task placement to be most effective.

For task placement case, only Energy Aware and Capacity Aware Scheduling
(EAS/CAS) make use of uclamp for now, which implies that it is applied on
heterogeneous systems only. When a task wakes up, the scheduler will look at
the effective uclamp values of the woken task to check if it will fit the
capacity of the CPU. Favouring the most energy efficient CPU that fits the
performant request hints. Enabling it to favour a bigger CPU if uclamp_min is
high even if the utilization of the task is low. Or enable it to run on
a smaller CPU if UCLAMP_MAX is low, even if the utilization of the task is
high.

Similarly in schedutil, when it needs to make a frequency update it will look
at the effective uclamp values of the current running task on the rq and select
the appropriate frequency that will satisfy the performance request hints of
the task, taking into account the current utilization of the rq.

Other paths like setting overutilization state (which effectively disables EAS)
make use of uclamp as well. Such cases are considered necessary housekeeping to
allow the 2 main use cases above and will not be covered in detail here as they
could change with implementation details.

2.1. Frequency hints are applied at context switch
--------------------------------------------------

At context switch, we tell schedutil of the new uclamp values (or min/max perf
requirments) of the newly RUNNING task. It is up to the governor to try its
best to honour these requests.

For uclamp to be effective, it is desired to have a hardware with reasonably
fast DVFS hardware (rate_limit_us is short).

It is believed that most modern hardware (including lower rend ones) has
rate_limit_us <= 2ms which should be good enough. Having 500us or below would
be ideal so the hardware can reasonably catch up with each task's performance
constraints.

Schedutil might ignore the task's performance request if its historical runtime
has been shorter than the rate_limit_us.

See :ref:`Section 5.2 <schedutil-response-time-issues>` for more details on
schedutil related limitations.

2.2. Hierarchical aggregation
-----------------------------

As stated earlier, util clamp is a property of every task in the system. But
the actual applied (effective) value can be influenced by more than just the
request made by the task or another actor on its behalf (middleware library).

The effective util clamp value of any task is restricted as follows:

  1. By the uclamp settings defined by the cgroup CPU controller it is attached
     to, if any.
  2. The restricted value in (1) is then further restricted by the system wide
     uclamp settings.

:ref:`Section 3 <uclamp-interfaces>` discusses the interfaces and will expand
further on that.

For now suffice to say that if a task makes a request, its actual effective
value will have to adhere to some restrictions imposed by cgroup and system
wide settings.

The system will still accept the request even if effectively will be beyond the
constraints, but as soon as the task moves to a different cgroup or a sysadmin
modifies the system settings, the request will be satisfied only if it is
within new constraints.

In other words, this aggregation will not cause an error when a task changes
its uclamp values, but rather the system may not be able to satisfy requests
based on those factors.

2.3. Range
----------

Uclamp performance request has the range of 0 to 1024 inclusive.

For cgroup interface percentage is used (that is 0 to 100 inclusive).
Just like other cgroup interfaces, you can use 'max' instead of 100.

2.4. Older design
-----------------

Older design was behaving differently due what was called max-aggregation rule
which was adding high complexity and had some limitations. Please consult the
docs corresponding to your kernel version as part of this doc might reflect how
it works on your kernel.

.. _uclamp-interfaces:

3. Interfaces
=============

3.1. Per task interface
-----------------------

sched_setattr() syscall was extended to accept two new fields:

* sched_util_min: requests the minimum performance point the system should run
  at when this task is running. Or lower performance bound.
* sched_util_max: requests the maximum performance point the system should run
  at when this task is running. Or upper performance bound.

For example, the following scenario have 40% to 80% utilization constraints:

::

        attr->sched_util_min = 40% * 1024;
        attr->sched_util_max = 80% * 1024;

When task @p is running, **the scheduler should try its best to ensure it
starts at 40% performance level**. If the task runs for a long enough time so
that its actual utilization goes above 80%, the utilization, or performance
level, will be capped.

The special value -1 is used to reset the uclamp settings to the system
default.

Note that resetting the uclamp value to system default using -1 is not the same
as manually setting uclamp value to system default. This distinction is
important because as we shall see in system interfaces, the default value for
RT could be changed. SCHED_NORMAL/OTHER might gain similar knobs too in the
future.

3.2. cgroup interface
---------------------

There are two uclamp related values in the CPU cgroup controller:

* cpu.uclamp.min
* cpu.uclamp.max

When a task is attached to a CPU controller, its uclamp values will be impacted
as follows:

* cpu.uclamp.min is a protection as described in :ref:`section 3-3 of cgroup
  v2 documentation <cgroupv2-protections-distributor>`.

  If a task uclamp_min value is lower than cpu.uclamp.min, then the task will
  inherit the cgroup cpu.uclamp.min value.

  In a cgroup hierarchy, effective cpu.uclamp.min is the max of (child,
  parent).

* cpu.uclamp.max is a limit as described in :ref:`section 3-2 of cgroup v2
  documentation <cgroupv2-limits-distributor>`.

  If a task uclamp_max value is higher than cpu.uclamp.max, then the task will
  inherit the cgroup cpu.uclamp.max value.

  In a cgroup hierarchy, effective cpu.uclamp.max is the min of (child,
  parent).

For example, given following parameters:

::

        p0->uclamp[UCLAMP_MIN] = // system default;
        p0->uclamp[UCLAMP_MAX] = // system default;

        p1->uclamp[UCLAMP_MIN] = 40% * 1024;
        p1->uclamp[UCLAMP_MAX] = 50% * 1024;

        cgroup0->cpu.uclamp.min = 20% * 1024;
        cgroup0->cpu.uclamp.max = 60% * 1024;

        cgroup1->cpu.uclamp.min = 60% * 1024;
        cgroup1->cpu.uclamp.max = 100% * 1024;

when p0 and p1 are attached to cgroup0, the values become:

::

        p0->uclamp[UCLAMP_MIN] = cgroup0->cpu.uclamp.min = 20% * 1024;
        p0->uclamp[UCLAMP_MAX] = cgroup0->cpu.uclamp.max = 60% * 1024;

        p1->uclamp[UCLAMP_MIN] = 40% * 1024; // intact
        p1->uclamp[UCLAMP_MAX] = 50% * 1024; // intact

when p0 and p1 are attached to cgroup1, these instead become:

::

        p0->uclamp[UCLAMP_MIN] = cgroup1->cpu.uclamp.min = 60% * 1024;
        p0->uclamp[UCLAMP_MAX] = cgroup1->cpu.uclamp.max = 100% * 1024;

        p1->uclamp[UCLAMP_MIN] = cgroup1->cpu.uclamp.min = 60% * 1024;
        p1->uclamp[UCLAMP_MAX] = 50% * 1024; // intact

Note that cgroup interfaces allows cpu.uclamp.max value to be lower than
cpu.uclamp.min. Other interfaces don't allow that.

3.3. System interface
---------------------

3.3.1 sched_util_clamp_min
--------------------------

System wide limit of allowed UCLAMP_MIN range. By default it is set to 1024,
which means that permitted effective UCLAMP_MIN range for tasks is [0:1024].
By changing it to 512 for example the range reduces to [0:512]. This is useful
to restrict how much boosting tasks are allowed to acquire.

Requests from tasks to go above this knob value will still succeed, but
they won't be satisfied until it is more than p->uclamp[UCLAMP_MIN].

The value must be smaller than or equal to sched_util_clamp_max.

3.3.2 sched_util_clamp_max
--------------------------

System wide limit of allowed UCLAMP_MAX range. By default it is set to 1024,
which means that permitted effective UCLAMP_MAX range for tasks is [0:1024].

By changing it to 512 for example the effective allowed range reduces to
[0:512]. This means is that no task can run above 512, which implies that all
rqs are restricted too. IOW, the whole system is capped to half its performance
capacity.

This is useful to restrict the overall maximum performance point of the system.
For example, it can be handy to limit performance when running low on battery
or when the system wants to limit access to more energy hungry performance
levels when it's in idle state or screen is off.

Requests from tasks to go above this knob value will still succeed, but they
won't be satisfied until it is more than p->uclamp[UCLAMP_MAX].

The value must be greater than or equal to sched_util_clamp_min.

.. _uclamp-default-values:

3.4. Default values
-------------------

By default all SCHED_NORMAL/SCHED_OTHER tasks are initialized to:

::

        p_fair->uclamp[UCLAMP_MIN] = 0
        p_fair->uclamp[UCLAMP_MAX] = 1024

That is, by default they're boosted to run at the maximum performance point of
changed at boot or runtime. No argument was made yet as to why we should
provide this, but can be added in the future.

For SCHED_FIFO/SCHED_RR tasks:

::

        p_rt->uclamp[UCLAMP_MIN] = 1024
        p_rt->uclamp[UCLAMP_MAX] = 1024

That is by default they're boosted to run at the maximum performance point of
the system which retains the historical behavior of the RT tasks.

RT tasks default uclamp_min value can be modified at boot or runtime via
sysctl. See below section.

.. _sched-util-clamp-min-rt-default:

3.4.1 sched_util_clamp_min_rt_default
-------------------------------------

Running RT tasks at maximum performance point is expensive on battery powered
devices and not necessary. To allow system developer to offer good performance
guarantees for these tasks without pushing it all the way to maximum
performance point, this sysctl knob allows tuning the best boost value to
address the system requirement without burning power running at maximum
performance point all the time.

Application developer are encouraged to use the per task util clamp interface
to ensure they are performance and power aware. Ideally this knob should be set
to 0 by system designers and leave the task of managing performance
requirements to the apps.

4. How to use util clamp
========================

Util clamp promotes the concept of user space assisted power and performance
management. At the scheduler level there is no info required to make the best
decision. However, with util clamp user space can hint to the scheduler to make
better decision about task placement and frequency selection.

Best results are achieved by not making any assumptions about the system the
application is running on and to use it in conjunction with a feedback loop to
dynamically monitor and adjust. Ultimately this will allow for a better user
experience at a better perf/watt.

For some systems and use cases, static setup will help to achieve good results.
Portability will be a problem in this case. How much work one can do at 100,
200 or 1024 is different for each system. Unless there's a specific target
system, static setup should be avoided.

There are enough possibilities to create a whole framework based on util clamp
or self contained app that makes use of it directly.

4.1. Boost important and DVFS-latency-sensitive tasks
-----------------------------------------------------

A GUI task might not be busy to warrant driving the frequency high when it
wakes up. However, it requires to finish its work within a specific time window
to deliver the desired user experience. The right frequency it requires at
wakeup will be system dependent. On some underpowered systems it will be high,
on other overpowered ones it will be low or 0.

This task can increase its UCLAMP_MIN value every time it misses the deadline
to ensure on next wake up it runs at a higher performance point. It should try
to approach the lowest UCLAMP_MIN value that allows to meet its deadline on any
particular system to achieve the best possible perf/watt for that system.

On heterogeneous systems, it might be important for this task to run on
a faster CPU.

**Generally it is advised to perceive the input as performance level or point
which will imply both task placement and frequency selection**.

4.2. Cap background tasks
-------------------------

Like explained for Android case in the introduction. Any app can lower
UCLAMP_MAX for some background tasks that don't care about performance but
could end up being busy and consume unnecessary system resources on the system.

4.3. Powersave mode
-------------------

sched_util_clamp_max system wide interface can be used to limit all tasks from
operating at the higher performance points which are usually energy
inefficient.

This is not unique to uclamp as one can achieve the same by reducing max
frequency of the cpufreq governor. It can be considered a more convenient
alternative interface.

4.4. Per-app performance restriction
------------------------------------

Middleware/Utility can provide the user an option to set UCLAMP_MIN/MAX for an
app every time it is executed to guarantee a minimum performance point and/or
limit it from draining system power at the cost of reduced performance for
these apps.

If you want to prevent your laptop from heating up while on the go from
compiling the kernel and happy to sacrifice performance to save power, but
still would like to keep your browser performance intact, uclamp makes it
possible.

5. Limitations
==============

5.1. UCLAMP_MAX can break PELT (util_avg) signal
------------------------------------------------

PELT assumes that frequency will always increase as the signals grow to ensure
there's always some idle time on the CPU. But with UCLAMP_MAX, this frequency
increase will be prevented which can lead to no idle time in some
circumstances. When there's no idle time, a task will stuck in a busy loop,
which would result in util_avg being 1024.

Combing with issue described below, this can lead to unwanted frequency spikes
when severely capped tasks share the rq with a small non capped task.

As an example if task p, which have:

::

        p0->util_avg = 300
        p0->uclamp[UCLAMP_MAX] = 0

wakes up on an idle CPU, then it will run at min frequency (Fmin) this
CPU is capable of. The max CPU frequency (Fmax) matters here as well,
since it designates the shortest computational time to finish the task's
work on this CPU.

If the ratio of Fmax/Fmin is 3, then maximum value will be:

::

        300 * (Fmax/Fmin) = 900

which indicates the CPU will still see idle time since 900 is < 1024. The
_actual_ util_avg will not be 900 though, but somewhere between 300 and 900. As
long as there's idle time, p->util_avg updates will be off by a some margin,
but not proportional to Fmax/Fmin.

::

        p0->util_avg = 300 + small_error

Now if the ratio of Fmax/Fmin is 4, the maximum value becomes:

::

        300 * (Fmax/Fmin) = 1200

which is higher than 1024 and indicates that the CPU has no idle time. When
this happens, then the _actual_ util_avg will become:

::

        p0->util_avg = 1024

If task p1 wakes up on this CPU, which have:

::

        p1->util_avg = 200
        p1->uclamp[UCLAMP_MAX] = 1024

Since the capped p0 task was running and throttled severely, then the
rq->util_avg will be:

::

        p0->util_avg = 1024
        p1->util_avg = 200

        rq->util_avg = 1024

Hence lead to a frequency spike when p1 is running. If p0 wasn't throttled we
should get:

::

        p0->util_avg = 300
        p1->util_avg = 200

        rq->util_avg = 500

and run somewhere near mid performance point of that CPU, not the Fmax p1 gets.

.. _schedutil_response_time_issues:

5.2. Schedutil response time issues
-----------------------------------

schedutil has three limitations:

        1. Hardware takes non-zero time to respond to any frequency change
           request. On some platforms can be in the order of few ms.
        2. Non fast-switch systems require a worker deadline thread to wake up
           and perform the frequency change, which adds measurable overhead.
        3. schedutil rate_limit_us drops any requests during this rate_limit_us
           window.

If a relatively small task is doing critical job and requires a certain
performance point when it wakes up and starts running, then all these
limitations will prevent it from getting what it wants in the time scale it
expects.

This limitation is not only impactful when using uclamp, but will be more
prevalent as we no longer gradually ramp up or down. We could easily be
jumping between frequencies depending on the order tasks wake up, and their
respective uclamp values.

We regard that as a limitation of the capabilities of the underlying system
itself.

There is room to improve the behavior of schedutil rate_limit_us, but not much
to be done for 1 or 2. They are considered hard limitations of the system.
