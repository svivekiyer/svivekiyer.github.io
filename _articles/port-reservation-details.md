---
title: Port Reservation Details
subtitle: Two fabrics, a single view. I built one in 2011. It still isn't something you can buy.
date: 2026-10-25
summary: A gap I found as a junior SAN admin in 2011 — no tool shows you fabric A and fabric B next to each other at the moment you need to compare them — is still a gap today, on Brocade, on Cisco, and on network switches too
repo: fabric-compare
---

A storage network is built in pairs. Fabric A and fabric B. Two physically separate networks, no cables between them, each able to carry all the traffic on its own. A server gets a path through A and a path through B. If one fabric dies — a bad firmware upgrade, a failed chassis, someone pulling the wrong cable — the other carries everything and nobody notices.

That design only works if the two halves actually mirror each other. The redundancy isn't in having two fabrics. It's in the two fabrics being equivalent.

And in 2011, in my first job, there was no quick way to check whether they were. There still isn't.

## The question nobody could answer quickly

Here is the situation that came up constantly.

A new server needs storage. It needs a free port on fabric A and a free port on fabric B, and ideally in matching positions so the next person can make sense of the cabling. So you need to know, right now, what is free on both sides.

Every tool I had showed me one switch at a time. Log into a switch on fabric A, run the command, read the output. Log into the partner switch on fabric B, run the command, read that output. Then hold both in your head, or paste them into a spreadsheet and line them up by hand.

Two sources, read separately, compared manually. Every single time.

There were proper tools for this — SAN Health and others like it. They were thorough and I'm not going to pretend otherwise. But the loop was slow: run a collection, send the output off, wait, get a report back. That is a fine way to do a quarterly audit. It is a useless way to answer a question you have right now, in the middle of an allocation, or at two in the morning when something is broken and you need to know whether both sides are still equivalent.

The gap wasn't that the information didn't exist. It was that getting it took long enough that people stopped asking, and that asymmetry between A and B could sit there unnoticed for months.

## The fix was not clever

Poll both fabrics. Put them next to each other on one page. Publish that page at a URL.

That is the entire idea, and it was the right one.

The script loops over each fabric, and for each switch it reads a column in a list file that says `A` or `B`. Fabric A's output goes to one file, fabric B's to another. Then one line does the actual work:

```sh
paste -d "," fabric_a.csv fabric_b.csv > combined.csv
```

[`paste`](https://man7.org/linux/man-pages/man1/paste.1.html) glues two files together side by side — line one of the first next to line one of the second, line two next to line two, and on down. It exists in Unix precisely for this, and it is the whole trick. After that the HTML table is just fourteen columns instead of seven: switch, index, slot, port, description, port address, status — and then those same seven again, for the other side.

```sh
awk 'BEGIN{
  FS=","
  print "<table border=\"1\">"
  print "<th>SWITCH</th><th>INDEX</th><th>SLOT</th><th>PORT</th><th>Description</th><th>PWWN</th><th>STATUS</th>" \
        "<th>SWITCH</th><th>INDEX</th><th>SLOT</th><th>PORT</th><th>Description</th><th>PWWN</th><th>STATUS</th>"
}
{ printf "<tr>"; for(i=1;i<=NF;i++) printf "<td>%s</td>", $i; print "</tr>" }
END{ print "</table>" }' combined.csv > fabric.html
```

The page was titled "Port Reservation Sheet", because that is what people used it for. Not reporting. Allocating. You opened it, found a row where both sides were free, and took it.

Getting the names in there mattered too. A port number on its own tells you nothing — you need to know *what* is plugged in. That meant pulling three different outputs off each switch and stitching them together: the port list, the port names, and the alias table that maps a device's hardware address to a name a human recognises. Three different shapes of text, joined with `awk`, to produce one row that no single command on the switch could give you.

And there was a small bit of care in it I'm still happy about: if a fabric only had one switch, it rendered single-width instead of breaking. The comment in the script says *"If its a one switch fabric then we are considering it as Fabric A."* Not elegant. But it degraded instead of failing, which is more than a lot of code manages.

## What was wrong with it

Quite a lot, and the worst one is subtle enough to be worth sitting with.

**`paste` lines things up by row number, not by port.** It puts line 40 of A next to line 40 of B. It does not put *port 40* of A next to *port 40* of B.

Those are the same thing right up until they aren't. Say fabric B is missing port 7 — an unlicensed port, a line filtered out of the output, a slightly different switch model on that side. Everything below that point shifts up by one:

```
  fabric A      fabric B      what the page showed
  ---------     ---------     ---------------------
  port 5        port 5        5 beside 5     correct
  port 6        port 6        6 beside 6     correct
  port 7        port 8        7 beside 8     wrong
  port 8        port 9        8 beside 9     wrong
  port 9        port 10       9 beside 10    wrong
```

From the gap down, every row compares one port against a different port. And the page still renders. It looks exactly as authoritative as it did before. It is just wrong, and nothing tells you.

**It parsed output meant for people.** The switch command prints a table for a human to read. The script removed the headers by naming them one at a time — a dozen or so strings. Any one of those changing in a firmware update means a header row gets counted as a port. It doesn't crash. It returns a slightly wrong number, which is worse.

**One column was always empty.** There are two versions of the script, one for fixed-port switches and one for director chassis. The chassis version reads a slot number, because a chassis has blades. The fixed-port version doesn't — there are no slots — but it still prints the slot variable into every row. So that column was blank on those fabrics for years. Nobody mentioned it. I only found it rereading the thing.

**An unreachable switch looked like an empty switch.** No error handling anywhere. If the login failed, the output was empty, the count came out zero, and the report published a switch with no ports as though that were a fact.

**`chmod 777` on everything it published.** World-writable — anyone with an account on that server could have edited the reports. No defence available.

**And a typo that has been harmless for years.** One `awk` block sets the field separator — the character `awk` splits each line on, which here should just be a comma — as `FS=","LMV`. A stray fragment that got pasted in at some point and never noticed. In `awk` an undefined variable is an empty string, so `","LMV` evaluates to `","` and behaves perfectly. It has been sitting in working code for the better part of a decade.

## I rebuilt it, so the bug is visible instead of theoretical

The `paste` flaw is easy to describe and hard to believe until you see it, because the broken page looks precisely as trustworthy as the correct one. So I rebuilt the tool — [`fabric-compare`](https://github.com/svivekiyer/fabric-compare), a couple of hundred lines of POSIX shell and `awk`, no dependencies, with entirely synthetic sample data — and gave it a switch that lets you choose how the two sides are joined.

```sh
./fabric-compare.sh --join port   # join on port index
./fabric-compare.sh --join row    # join on line number, as the 2011 version did
```

Same inputs both times. Both screenshots below are rendered in the original's house style — bordered table, orange heading, nothing you'd call design — so that the only thing changing between them is the join. Here is `--join port`:

![Side-by-side port report, joined on port index, with asymmetries flagged]({{ '/assets/img/report-classic-port-join.png' | relative_url }})

Sixteen ports on A, nineteen on B, and five asymmetries called out. The one that matters is `bkp01_hba0`: a host with a path on fabric A and nothing on fabric B. A single-pathed server, in a design whose entire purpose is that nothing is single-pathed. That is the thing you opened the page to find.

Now the same data, joined the old way:

![The same data joined by row number — unrelated ports presented as pairs]({{ '/assets/img/report-classic-row-join.png' | relative_url }})

Twelve wrong rows. The two sides diverge at port 7, and from there every row pairs one port with a different one. Look at the first flagged row: fabric A port 7 is `bkp01_hba0`, fabric B port 8 is `esx03_hba1`. Two unrelated machines, sitting next to each other, presented as a redundant pair. The single-pathed host has vanished — not because it was fixed, but because the misalignment papered over it.

The `MISALIGNED` and `RAN_OUT` flags are a diagnostic I added for this demonstration. The original had no such check, and that absence is the actual point. A comparison that is correct by coincidence of ordering rather than by matching keys fails silently, and silent is the worst way for a report to fail. The 2011 version would have rendered that second page with no warning at all, and someone would have allocated against it.

That is the habit the whole exercise left me with, and it generalises well beyond storage: **if you are comparing two things, join them on a key, never on position.** Position is an accident. Keys are a fact.

## The gap is still there

This is the part that surprised me when I went back to it.

I've worked on Brocade and on Cisco. I've watched the same shape of problem on network switches that have nothing to do with storage. And the thing I wanted as a junior admin in 2011 — both sides of a redundant pair, next to each other, current, without asking anyone — I still can't buy.

I know it's still open because I keep rebuilding it. The 2011 version was shell, because shell was what I had and what the switches would talk to. A later one was still shell, tidier, with collection split away from rendering. The version I maintain now is Python, inside a web application, reading from APIs instead of scraping terminal output, and showing a great deal more than ports. Three rewrites, three different sets of tools, fifteen years apart — and every one of them got written for the same reason: there was still nothing I could simply go and use.

The problem never changed. Only what I could bring to it did.

That last part is worth seeing rather than claiming, so `fabric-compare` renders both ways. Same script, same synthetic data, same comparison — `--style modern` instead of `--style classic`:

![The same comparison rendered in a present-day style — summary cards, the two fabrics colour-split]({{ '/assets/img/report-modern-port-join.png' | relative_url }})

It's better. It's obviously better: you can see the shape of the fabric before you've read a single row, and the colour split means you always know which side your eye is on. But look at what actually improved. The numbers are identical. The join is identical. The single-pathed host is found by exactly the same logic it was found by in 2011.

Fifteen years of progress went almost entirely into the part that was never hard. The rendering got easier because browsers got better, and that happened whether or not I did anything. The part that required judgement — knowing that port index is the key and line number is an accident, knowing that an unreachable switch is an error and not a zero — is the same work it always was, and no amount of better tooling has done it for me.

You can get a great view of one switch. Vendor interfaces are far better than they were. You can get dashboards full of metrics, per device, per port, per interface. What's still missing is the relational view: *these two things are supposed to mirror each other — do they?*

I think the reason is structural. Every vendor builds a view of its own equipment, and that view ends at the edge of the box. But A-versus-B isn't a fact about either box. It's a fact about the relationship between them, and that relationship only exists in the operator's head and in whatever list the operator keeps of which switch is on which side. No vendor owns that pairing, so no vendor builds the view. It gets left to whoever is holding the spreadsheet.

Which means it stays a scripting problem, and it stays worth knowing how to solve.

## What I'd build now

Read from the API, not from terminal output. Structured data, and the whole class of firmware-upgrade parsing breakage disappears. `fabric-compare` still parses text, deliberately, because it exists to mirror the original — but that is the first thing a serious version should drop, and in the Python one it is the first thing I did drop.

Join on the port index. Never on row position. If a port exists on one side and not the other, show that as a gap, in red — because that gap is the entire thing you are looking for.

Fail loudly. A switch that can't be reached is an error, not a zero.

Separate collecting from showing. The original did both in one script, which meant any second view of the same data was a rewrite.

And keep the last part exactly as it was. Put it on one page, at a URL, where anyone can look without asking permission. That decision is the difference between a script that helps one person and a tool a team uses, and it's the only part of this I'd change nothing about.

I've written the longer version of that list up in the repo, as [what an API-backed version actually changes](https://github.com/svivekiyer/fabric-compare/blob/main/docs/modern-version.md) — including the one I find most interesting, which is that a structured source lets you stop joining on position altogether and start joining on device identity. Port index is a better key than line number. It is still only a proxy for the question you're really asking.

