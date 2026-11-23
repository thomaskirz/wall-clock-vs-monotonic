For my master's thesis, I worked with the Neo4j graph database. While conducting experiments, I noticed that Neo4j returned a negative query execution time: -40.55 s. It turns out that the database uses *wall clock time* to measure runtime, and my system's wall clock was unstable and had been adjusted during the measurement.

A better approach would be to use a monotonic clock. Monotonic clocks only ever move forward and are used to measure durations (e.g. how long it takes to execute a query), while their absolute values are not meaningful.
Wall clocks, on the other hand, tell the time. They can be corrected via NTP when they drift, or adjusted manually, so they may jump backwards, which makes them unreliable for measuring durations.
As a rule of thumb: use wall clocks to know *when* something happened, and monotonic clocks to measure *how long* something took.

In my case, the clock instability seemed to be related to the Windows Subsystem for Linux VM I used for my experiments. I saw an execution time of 40.95 s for one query, followed by -40.55 s for the next. The two values sum to 0.40 s, which is around what I expected the two queries to take. From this, I concluded a clock jump of ~41 s during the first query and a jump back during the next one, which is roughly consistent with the system logs.
The following diagram shows how the two jumps produced these execution times:

Importantly, while it's obviouse that the negative value is wrong, the positive one is arguably more dangerous: it looks plausible, but it is just as affected by the clock instability, so every measurement from that period might be skewed.

I traced this to Neo4j's [`SystemNanoClock`](https://github.com/neo4j/neo4j/blob/2026.10/community/common/src/main/java/org/neo4j/time/SystemNanoClock.java): its `millis()` method, which Neo4j uses to time queries, returns `System.currentTimeMillis()`, the wall clock, even though the same class offers the monotonic `System.nanoTime()`. I'm planning to propose a fix upstream.