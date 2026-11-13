# Research: sealing loopback so only the gate's own ports are reachable

Question (issue #21): on macOS and Linux, from inside a coding agent's local run and a CI job, can the **Gate** enforce a seal that blocks all network except loopback ports its own processes opened, so that a real database already listening on 127.0.0.1 is unreachable? Which mechanism per platform, what does it cost at start-up, and what is the fallback when it is unavailable?

Researched 2026-11-13. Sources are primary (kernel docs, man pages, vendor docs, source code). Every number marked **measured** was taken on this machine (macOS 15.6.1, build 24G90, Apple silicon); everything marked **estimated** was not measured, because there was no Linux host and no running Docker daemon to measure on.

## Direct answer

1. **Linux: yes, exactly, and cheaply.** A new network namespace has its own private loopback. Nothing outside the namespace can listen on it, so "only ports the gate's own processes opened" is true by construction, with no port list to maintain. Mechanism: `bwrap --unshare-net` (bubblewrap brings the private loopback up itself) or `unshare -rn` plus `ip link set lo up`. Estimated cost: single-digit to low tens of milliseconds.
2. **macOS: not exactly. A close approximation works and costs about 30 ms.** Seatbelt (`sandbox-exec`) filters loopback by port, but the profile is fixed when the process starts and cannot say "ports opened by this sandbox". What does work, measured: snapshot the TCP ports that are already listening, then start the seal with a profile that allows loopback and denies exactly those ports. In the tests below this blocked 30 of 30 pre-existing listeners while the sealed process's own listeners stayed reachable. It is a deny-list of a moment in time, not real isolation.
3. **Unix domain sockets are a second hole on both platforms** and must be closed separately. A network namespace does not hide path-based sockets such as `/var/run/postgresql/.s.PGSQL.5432` or `/var/run/docker.sock`. Seatbelt can deny them by path (measured); on Linux it takes a mount namespace that hides the socket directories, or a seccomp filter.
4. **The seal cannot always be established**, and the gate has to detect that at run time rather than assume:
   - macOS: a process that is already inside a Seatbelt sandbox cannot apply a second one (measured: `sandbox_apply: Operation not permitted`). So if the coding agent already sandboxes the gate's shell, the gate cannot add its own seal.
   - Linux: unprivileged user namespaces are restricted by AppArmor on Ubuntu 23.10 and later, and by Docker's default seccomp profile inside containers.
5. **Fallback: probe, then degrade loudly.** Try the strong mechanism; if it fails, fall to the next one; and if the gate ends up with loopback open, list the pre-existing listening ports and say so in the **Verdict**. Whether an unsealed run may still produce a *merge* verdict is a product decision (see Open questions).

## Recommendation per platform

| Where the gate runs | Mechanism | Property achieved | Start-up cost |
|---|---|---|---|
| Linux developer machine | `bwrap --unshare-net` plus hiding socket directories (tmpfs over `/run`, `/var/run`, `/tmp`) | Exact: private loopback | Estimated 5-30 ms |
| GitHub Actions Linux runner | Same; if unprivileged user namespaces are refused, `sudo unshare -n` and drop back to the runner user (runners have passwordless sudo) | Exact | Estimated 5-50 ms |
| Linux, inside a Docker container | Same if the container allows it. If not: the container already has its own loopback, so host databases are not on it; the gate cannot block non-loopback traffic from inside and must report a weak seal unless the job itself was started with no network | Loopback private to the container, shared with anything else in it | 0 ms extra |
| Linux, inside Claude Code's own sandbox | Nothing to add for loopback: Claude Code already puts the command in a private network namespace. But that sandbox still offers a proxy to allowed domains, so it is not a full seal | Loopback exact; outbound depends on the user's allow-list | 0 ms extra |
| macOS developer machine | `sandbox-exec` with: deny all remote IP, allow loopback, deny each port in a snapshot of current listeners, deny Unix sockets outside the gate's scratch directory. Then verify from inside the seal that each snapshot port is refused before running tests | Approximate: everything listening at seal time is blocked | **Measured** about 20-25 ms for `sandbox-exec` plus about 7 ms for the snapshot |
| macOS, inside an agent sandbox that is already Seatbelt | None available. Either the gate's command is run outside the agent's sandbox so it can apply its own, or the gate reports that it could not seal | None | - |
| macOS GitHub Actions runner | Same as macOS developer machine (not verified on a runner) | Approximate | Estimated same as local |

A Linux container or VM on macOS (Docker Desktop, Colima, Apple's `container`) would give exact isolation on a Mac, but it runs the tests on a different operating system with a different dependency build, needs a daemon the gate does not control, and costs an estimated 0.5-2 s per start. It is not recommended as the default.

## Evidence

### 1. Linux network namespaces

- A network namespace isolates "network devices, IPv4 and IPv6 protocol stacks, IP routing tables, firewall rules", port numbers and sockets, and "network namespaces isolate the UNIX domain abstract socket namespace" ([network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)). Path-based Unix sockets live in the filesystem and are not covered by that sentence, which is why they need separate handling.
- `bwrap --unshare-net` means "Create a new network namespace", and "the user namespace is required if bwrap is not run as root" ([bwrap man page source](https://github.com/containers/bubblewrap/blob/main/bwrap.xml)). The source brings loopback up for you: `if (opt_unshare_net) loopback_setup (); /* Will exit if unsuccessful */` ([bubblewrap.c](https://github.com/containers/bubblewrap/blob/main/bubblewrap.c)).
- Docker's equivalent, `--network none`: "Within the container, only the loopback device is created" ([Docker docs, none driver](https://docs.docker.com/engine/network/drivers/none/)).
- This is how Claude Code does it on Linux: "the command runs in a separate network namespace that has no connection to your network", and "a sandboxed command's `localhost` is private to that command... A direct connection to `localhost` or `127.0.0.1` doesn't reach servers on the host, and `allowLocalBinding` has no effect" ([Claude Code sandboxing docs](https://code.claude.com/docs/en/sandboxing)). The open-source runtime behind it says the same: "the sandbox has its own network namespace, whose loopback is always usable and is not the host's" ([sandbox-runtime README](https://github.com/anthropic-experimental/sandbox-runtime)).

### 2. Where Linux namespaces are not available

- **Ubuntu 23.10 and 24.04** added an AppArmor control over unprivileged user namespaces, switched by `kernel.apparmor_restrict_unprivileged_userns` ([Ubuntu blog](https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces), [Ubuntu discourse](https://discourse.ubuntu.com/t/understanding-apparmor-user-namespace-restriction/58007)). The sandbox-runtime README states that Ubuntu 24.04 and later "enable `kernel.apparmor_restrict_unprivileged_userns` by default".
- **GitHub Actions**: "The Linux and macOS virtual machines both run using passwordless `sudo`" ([GitHub-hosted runners](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)), so `sudo unshare -n` or relaxing the sysctl is always possible there. A user report of `unshare(...): Operation not permitted` on the `ubuntu-24.04` image exists ([runner-images #10443](https://github.com/actions/runner-images/issues/10443)); the maintainers did not say the image would change.
- **Inside Docker**: the default seccomp profile lists `unshare` as "Deny cloning new namespaces for processes. Also gated by CAP_SYS_ADMIN, with the exception of unshare --user", and `clone` and `setns` likewise ([Docker seccomp docs](https://docs.docker.com/engine/security/seccomp/)). Claude Code's docs describe a milder failure in practice: "In an unprivileged container, bubblewrap can't mount a fresh `/proc` filesystem", fixed by a weaker mode that reuses the container's `/proc`. So behaviour varies with the container's settings and has to be probed.
- GitHub container jobs cannot be given a network mode: "The `--network` and `--entrypoint` options are not supported" in `container.options` ([GitHub docs](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/run-jobs-in-a-container)).

### 3. Other Linux mechanisms, and why they are not the first choice

- **Landlock** needs no privileges ("For unprivileged processes, setting the no_new_privs attribute is required") and works inside containers, but its network rules are by port only, not by address or by owner: `LANDLOCK_ACCESS_NET_BIND_TCP` and `LANDLOCK_ACCESS_NET_CONNECT_TCP` arrived in ABI 4, UDP rules only in ABI 10, and abstract Unix socket scoping in ABI 6 ([kernel Landlock docs](https://docs.kernel.org/userspace-api/landlock.html)). It is an allow-list, so it cannot say "loopback only" at all - a permitted port is permitted on every address. It is useful only as an extra layer.
- **seccomp** sees syscall numbers and plain arguments, not the socket address behind a pointer, so it can forbid creating a socket family but cannot filter by port. sandbox-runtime uses it exactly that way, for Unix sockets: "`allowUnixSockets` is ignored, since seccomp can't filter by path".
- **cgroup eBPF connect hooks** (`BPF_CGROUP_INET4_CONNECT`, since Linux 4.17, [ebpf.io docs](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_CGROUP_SOCK_ADDR/)) can see the destination, but loading BPF is itself blocked in default containers ("Deny loading potentially persistent BPF programs into kernel, already gated by CAP_SYS_ADMIN", Docker seccomp docs) and needs root elsewhere. Wherever the gate has that much privilege it can simply create a network namespace.

### 4. macOS: what Seatbelt can and cannot do

- `sandbox-exec` is officially deprecated. The local man page reads "execute within a sandbox (DEPRECATED)... Developers who wish to sandbox an app should instead adopt the App Sandbox feature" (`man sandbox-exec`, dated March 9, 2017). The profile language has no public reference from Apple.
- It is nonetheless what the agents use. Claude Code: "macOS: uses Seatbelt for sandbox enforcement" ([docs](https://code.claude.com/docs/en/sandboxing)); sandbox-runtime invokes `/usr/bin/sandbox-exec -p <profile>` ([macos-sandbox-utils.ts](https://github.com/anthropic-experimental/sandbox-runtime/blob/main/src/sandbox/macos-sandbox-utils.ts)); Codex: "sandboxing works out of the box using the built-in Seatbelt framework" ([Codex sandboxing docs](https://learn.chatgpt.com/docs/sandboxing)).
- Claude Code does **not** solve the loopback hole on macOS. Its docs say that with `network.allowLocalBinding` on, "Sandboxed commands can then listen on network ports and connect to any port on localhost, which includes every other service listening there." The rule it emits is `(allow network-outbound (remote ip "localhost:*"))`. With the setting off, the profile allows only the proxy's port, so a test suite cannot start its own server at all.
- Other macOS mechanisms need what the gate may not have: the packet filter device is root-only (`ls -l /dev/pf` gives `crw------- 1 root wheel`), and a Network Extension content filter ships as an approved, signed system extension (not verified against Apple's page, which could not be read; see Not verified).

### 5. macOS measurements

All on macOS 15.6.1. "Pre-existing" means a listener owned by a process outside the sandbox, standing in for a real database.

Behaviour:

| Test | Result |
|---|---|
| `(deny network*)` only | Everything blocked, including binding an own port |
| Allow `localhost:*` (what `allowLocalBinding` does) | Pre-existing port reachable - the hole |
| Allow-list of specific ports, `(allow network-outbound (remote ip "localhost:PORT"))`, 1 to 40 ports | Exact: 0 wrongly open, 0 wrongly closed. But a port the sealed process opens itself is not on the list, so it is blocked too |
| Port range, `"localhost:3000-4000"` | Rejected: "invalid port in network address" |
| Host other than `localhost` or `*`, `"127.0.0.1:80"` | Rejected: "host must be * or localhost in network address" |
| Deny-list in the shape below, 1 to 40 denied ports, IPv4 and IPv6 | 0 leaks in every run; own listener reachable; 1.1.1.1:443 blocked |
| Same idea in a different shape (`(deny network*)` base, then allow `localhost:*`, then per-port deny) | **Leaked**: denied ports stayed reachable in most runs |
| Pre-existing Unix socket, with `(deny network-outbound (remote unix-socket (path-regex #"^/")))` | Blocked; a socket under an allowed scratch subpath still works |
| UDP to a pre-existing loopback port, deny-list built from TCP listeners only | **Delivered** - the TCP snapshot does not cover UDP |
| `sandbox-exec` run from inside a `sandbox-exec` | Fails: `sandbox_apply: Operation not permitted` |
| The command that `sandbox-exec` starts (every probe above ran as its child) | Is under the seal. Deeper descendants were not tested separately with a holding profile |
| Connecting to the machine's own LAN address under a `localhost` rule | Counts as localhost: reachable, and blocked for denied ports |

The profile shape that held:

```
(version 1)
(allow default)
(deny network-outbound (remote ip "*:*"))
(allow network-outbound (remote ip "localhost:*"))
(deny network-outbound (remote ip "localhost:5432"))   ; one line per port in the snapshot
(deny network-outbound (remote unix-socket (path-regex #"^/")))
(allow network-outbound (remote unix-socket (subpath "/private/tmp/gate-scratch")))
```

Start-up cost, **measured**, median of 50 runs of `/usr/bin/true`:

| Step | Median | p95 |
|---|---|---|
| No sandbox | 2-7 ms | 4-15 ms |
| `sandbox-exec`, 3-line deny-network profile | 10-12 ms | 13-29 ms |
| `sandbox-exec`, full profile with 59 denied ports | 18-26 ms | 35-56 ms |
| Snapshot of listeners with `netstat -anv -p tcp` | 6-7 ms | 11-15 ms |
| Snapshot of listeners with `lsof -iTCP -sTCP:LISTEN` | 60 ms | 110-186 ms |
| `sandbox-exec`, 1,000 per-port allow rules | 56 ms | - |
| `sandbox-exec`, 4,000 per-port allow rules | 741 ms | - |
| `sandbox-exec`, 65,535 per-port allow rules | Did not finish one run in several minutes | - |

The last three rows rule out the "allow every port except the snapshot" design. The ranges in the first rows are the spread between two runs on a machine that was busy with other work.

Start-up cost, **estimated**, not measured:

| Step | Estimate |
|---|---|
| `unshare -rn` on Linux | 1-5 ms |
| `bwrap --unshare-net` with a handful of mounts | 5-30 ms |
| `docker run --network none` with the image already present and the daemon running | 0.5-2 s |

Even the worst realistic case is well under 1% of the gate's 2-minute budget, so start-up cost does not decide the choice. Availability and correctness do.

## Recommended design

1. **One probe at start, in order of strength.** Linux: try `bwrap --unshare-net ... true`; then `sudo -n unshare -n true`; then check whether the gate is already in a private namespace. macOS: try `sandbox-exec -p '(version 1)(allow default)' /usr/bin/true`; a failure means the gate is already sandboxed and cannot seal.
2. **Linux seal**: private network namespace, plus a mount namespace that puts empty tmpfs directories over `/run`, `/var/run` and `/tmp` so path-based database and Docker sockets disappear.
3. **macOS seal**: snapshot listeners with `netstat` (TCP, and UDP as well), generate the profile above, start the tests under it.
4. **Self-check inside the seal before any test runs**, on both platforms: confirm that a known outside listener (a canary the gate opens just before sealing) is refused, that a remote address is refused, and that an own listener works. This takes milliseconds and is what protects the gate from the undocumented profile behaviour seen above and from a silent macOS update.
5. **Report the seal level in the Verdict**: sealed, sealed with a snapshot (macOS), or not sealed, with the list of loopback ports that were listening.

## Main risks

- **The macOS seal is a snapshot.** A service that starts listening after the seal is built is reachable. So is any UDP service unless UDP ports are snapshotted too (that variant was not tested).
- **Seatbelt's profile language is undocumented and `sandbox-exec` is deprecated.** One plausible-looking profile leaked while another held. The design leans on behaviour observed on one macOS version, hence the mandatory self-check.
- **Nesting.** If the user has Claude Code's sandbox on, on macOS the gate cannot seal from inside it. With `allowLocalBinding` on, all of localhost is open; with it off, test suites that start a server cannot run.
- **Claude Code's Linux sandbox is not a full seal** even though its loopback is private: it still proxies traffic to the user's allowed domains.
- **Linux availability drifts**: Ubuntu's AppArmor restriction, container seccomp profiles and runner images all change over time. Only a run-time probe is reliable.
- **A denied port number that a test later wants.** If a pre-existing service holds port 5432 and the suite tries to start its own server there, the bind fails with "address in use". That is correct but may read as a flaky test rather than an environment problem; the gate should name the cause.
- **Inbound is not sealed on macOS** in the profile above: another local process could connect to a server that the sealed tests started.

## Not verified

- Every Linux claim is from documentation and source, not from a run. No Linux start-up number was measured.
- Whether `unshare` or `bwrap` works unprivileged on the current GitHub `ubuntu-24.04` runner image, and inside a default GitHub container job. Evidence is one user report plus Ubuntu's documentation.
- Whether `sandbox-exec` behaves the same on GitHub's macOS runners and on macOS 26.
- Network Extension requirements: Apple's documentation page did not load, so the statement that it needs an approved system extension is from general knowledge.
- Why the second deny-list shape leaked. It was observed, not explained.
- How a process can tell from inside that it is already in Claude Code's sandbox, other than by trying to seal and failing.

## Open questions for a human

1. **What does the gate do when it cannot seal** (already inside an agent's macOS sandbox, or a Linux container that refuses namespaces)? Options: refuse to run tests and give a *block* or no verdict; run with loopback open and cap the verdict at *merge with notes*; or run with loopback fully closed and accept that suites needing a local port fail. Recommendation: run with loopback open only if the snapshot shows nothing listening on loopback, otherwise stop and tell the developer which ports are open.
2. **Is the macOS snapshot seal good enough to call "sealed"**, given that it blocks what was listening at start but not what starts later? Recommendation: yes, and name it as a snapshot seal in the Verdict.
3. **Should the merge-gate skill ask to run outside Claude Code's sandbox on macOS** so that it can apply its own, stricter seal? That means a permission prompt or an `excludedCommands` entry in the user's settings.
