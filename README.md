# :rowboat: Raft

This is an instructional implementation of the Raft distributed consensus
algorithm in Go. It's accompanied by a series of blog posts:

* [Part 0: Introduction](https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/)
* [Part 1: Elections](https://eli.thegreenplace.net/2020/implementing-raft-part-1-elections/)
* [Part 2: Commands and log replication](https://eli.thegreenplace.net/2020/implementing-raft-part-2-commands-and-log-replication/)
* [Part 3: Persistence and optimizations](https://eli.thegreenplace.net/2020/implementing-raft-part-3-persistence-and-optimizations/)

Each of the `partN` directories in this repository is the complete source code
for Part N of the blog post series (except Part 0, which is introductory and has
no code). There is a lot of duplicated code between the different `partN`
directories - this is a conscious design decision. Rather than abstracting and
reusing parts of the implementation, I opted for keeping the code as simple
as possible. Each directory is completely self contained and can be read and
undestood in isolation. Using a graphical diff tool to see the deltas between
the parts can be instructional.

## Useful resources

Raft Paper Figure 2. Compare it side-by-side with the codes at `raft.go`

[Raft lecture (Raft user study) - YouTube](https://www.youtube.com/watch?v=YbZ3zDzDnrw) -- Not bad for high-level overview, refer MIT lecture below that is more geared towards implementation details.

[Lecture 5: Go, Threads, and Raft - YouTube](https://www.youtube.com/watch?v=UzzcUS2OHqo&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB&index=5) -- discusses implementation details. E.g. pitfalls, concurrent programming pattern in Go, RPC, deadlock issue in synchronous RPC, etc

[Lecture 6: Fault Tolerance: Raft (1) - YouTube](https://www.youtube.com/watch?v=64Zp3tzNbpE)

* What happen if leader half-failed, e.g., outgoing traffic (heartbeat) is fine, but incoming traffic (client req) is not fine? Can the system makes progress? Nope… because heartbeat prevents leader election process from other followers
* At most one leader per term (strong invariant). What about no leader? That's fine. Recall at-most semantic
    - At each term, each node only votes once. The node who starts the election will vote for itself
    - No one allowed to send AE unless it is the leader for that term
* @58:35  How does Raft reduce split vote? Randomized timer. Why? If all of them detected a leader is dead at the same time and starts election, then split votes is likely to happen. Recall that if you start an election, then you will vote yourself and requestVotes from others. With randomized timer, then we reduced the chance of parallel leader elections
    - @1:00:40:  How to determine the minimum and maximum of election timer before kicking off an election? For min, of course, it is at least heatbeat AE interval, but give it 2 or 3x slack since it might not be because of leader fails, but the heartbeat packet is dropped
* @1:07:00  Before leader failed, it sends AE to a subset of Followers, and then crashes before committing

[Lecture 7: Fault Tolerance: Raft (2) - YouTube](https://www.youtube.com/watch?v=4r8Mz3MMivY&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB)

* @10:00  criterion for becoming a new leader. Is the one who starts the election first always wins? Recall Lecture06 about election timer. Well, which nodes start the election is just one of the criteria to win election. Refer paper Figure 2, especially the behavior for RequestVotes RPC and Candidates
* @21:00  rollback scheme. Refer implementation at "eliben, Raft part3: Optimizing AppendEntries conflict resolution"
* @32:45  Discussion on the volatile and persistent, figure 2 top left. Why some fields can be volatile? Why some needs to be persistent?
    - Persisting is very expensive, especially on hard disk
    - How to have high perf? SSD, battery backed DRAM
* @50:00  Log compaction and snapshot
* @1:04:10  Correctness - linearizability

## How to use this repository

You can read the code, but I'd also encourage you to run tests and observe the
logs they print out. The repository contains a useful tool for visualizing
output. Here's a complete usage example:

```
$ cd part1
$ go test -v -race -run TestElectionFollowerComesBack |& tee /tmp/raftlog
... logging output
... test should PASS
$ go run ../tools/raft-testlog-viz/main.go < /tmp/raftlog
PASS TestElectionFollowerComesBack map[0:true 1:true 2:true TEST:true] ; entries: 150
... Emitted file:///tmp/TestElectionFollowerComesBack.html

PASS
```

Now open `file:///tmp/TestElectionFollowerComesBack.html` in your browser.
You should see something like this:

![Image of log browser](https://github.com/eliben/raft/blob/master/raftlog-screenshot.png)

Scroll and read the logs from the servers, noticing state changes (highlighted
with colors). Feel free to add your own `cm.dlog(...)` calls to the code to
experiment and print out more details.

## Changing and testing the code

Each `partN` directory is completely independent of the others, and is its own
Go module. The Raft code itself has no external dependencies; the only `require`
in its `go.mod` is for a package that enables goroutine leak testing - it's only
used in tests.

To work on `part2`, for example:

```
$ cd part2
... make code changes
$ go test -race ./...
```

Depending on the part and your machine, the tests can take up to a minute to
run. Feel free to enable verbose logging with ``-v``, and/or used the provided
``dotest.sh`` script to run specific tests with log visualization.

## Contributing

I'm interested in hearing your opinion or suggestions for the code in this
repository. Feel free to open an issue if something is unclear, or if you think
you found a bug. Code contributions through PRs are welcome as well.
