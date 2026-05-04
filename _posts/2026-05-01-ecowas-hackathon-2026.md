---
title: "ECOWAS Regional Hackathon 2026: #1 in Togo across every phase"
date: 2026-05-01 00:00:00 +0000
categories: [Achievements, CTF Competitions]
tags: [ecowas, ctf, jeopardy, koth, boot2root, redteam-tg, togo, west-africa, cybersecurity]
image:
  path: /assets/ecowas-2026/ecowas-cover.png
  alt: Official logo of the 2026 ECOWAS Regional Hackathon
---

RedTeam-TG represented Togo in the ECOWAS Regional Hackathon, the official cybersecurity competition of the Economic Community of West African States. The tournament brings together 15 member nations — Benin, Burkina Faso, Cape Verde, Côte d'Ivoire, Gambia, Ghana, Guinea, Guinea-Bissau, Liberia, Mali, Niger, Nigeria, Senegal, Sierra Leone, and Togo — each hosting a national qualifier, with the top team earning a spot at the continental grand final. ECOWAS covers all travel, accommodation, and meal expenses for finalists, and the 2026 edition will be held in Ghana.

I competed as a member and captain of RedTeam-TG.

The Togolese national qualifier ran from April 1 to May 1, 2026, across three successive elimination phases. RedTeam-TG placed first at every stage, securing Togo's seat at the grand final.

---

## Phase 1: Jeopardy (April 1 - May 1, 2026)

**1,035 registered participants. 317 teams. We placed 1st across all of West Africa.**

The first phase consisted of a Jeopardy-style Capture the Flag (CTF) competition hosted on a unified global leaderboard, open simultaneously to all ECOWAS member states. A total of 1,035 participants registered and formed 317 teams, all competing on a single shared board without regional subdivisions or separate brackets. Togo was not isolated into a national category — from the outset, our teams were ranked directly alongside competitors from every participating nation.

<img src="/assets/ecowas-2026/ecowas-registered.png" alt="The ECOWAS Hackathon 2026 platform showing 1,035 registered participants across all member states" style="border-radius: 10px; width: 65%;" />

The competition encompassed 13 distinct challenge categories — including Cryptography, Discord, Forensics, Miscellaneous, Mobile, Network Forensics, OSINT, Binary Exploitation (Pwn), Reverse Engineering, Steganography, and Web — all of which our team successfully solved in their entirety.

<img src="/assets/ecowas-2026/ecowas-p1-challenges.png" alt="Challenge board showing all 13 categories completed at 100%" style="border-radius: 10px; width: 65%;" />

With a final score of **14,150 points**, RedTeam-TG secured **1st** place on the global leaderboard, finishing ahead of all 317 competing teams from across West Africa.

<img src="/assets/ecowas-2026/ecowas-p1-global-leaderboard.png" alt="Global leaderboard for Phase 1: RedTeam-TG ranked 1st out of 317 teams and 1,035 registered participants across West Africa" style="border-radius: 10px; width: 65%;" />

On the Togo national leaderboard comprising 49 teams , RedTeam-TG also claimed first place. While Government and FireTeam matched our point total, both teams were ranked behind us on the basis of solve time.

<img src="/assets/ecowas-2026/ecowas-p1-country-top3.png" alt="Togo country leaderboard for Phase 1: RedTeam-TG in 1st place out of 49 Togolese teams" style="border-radius: 10px; width: 65%;" />

The top 5 teams per country were eligible to advance to Phase 2. RedTeam-TG progressed as Togo's first-place representative.

<img src="/assets/ecowas-2026/ecowas-p1-top5-togo.png" alt="Top 5 Togolese teams qualified for Phase 2, with RedTeam-TG leading the standings" style="border-radius: 10px; width: 65%;" />

---

## Phase 2: King of the Hill (April 25, 2026)

**24 participants. 6 teams. We placed 1st again.**

In King of the Hill, there are no flags to capture only ground to hold. Teams fight for root access on a shared machine, plant their name in `/root/team.txt` , and defend that position for as long as possible. Every 30 seconds, the platform checks who holds the file and awards a point accordingly. When the machine resets, every team is back to zero and the battle for control starts over.

<img src="/assets/ecowas-2026/ecowas-p2-koth.png" alt="Togo KOTH challenge page showing the King of the Hill format for Phase 2, April 25, 2026" style="border-radius: 10px; width: 65%;" />

The target machine for this phase was `shambles.local`, accessible exclusively through a WireGuard VPN. Claiming control was straightforward planting our Team name was as simple as:

```bash
echo "RedTeam-TG" > /root/team.txt
```

One line. But keeping your name in that file across every reset is what the competition actually measures. Each reset triggers a race: whoever re-exploits the machine and writes their name first takes the tick. You cannot do that manually at competitive speed. You need scripts ready to fire the instant the machine comes back up.

<img src="/assets/ecowas-2026/ecowas-p2-koth-details.png" alt="KOTH instance details: target IP 10.100.10.46, team.txt path, and tick scoring structure" style="border-radius: 10px; width: 65%;" />

On top of attacking, you have to defend. Once you own the machine, you patch the vulnerability you just used, because every other team is trying the same entry point. Leave it open and someone overwrites your name. You are attacking and defending simultaneously, against live opponents, on a machine that resets every hour.

We held control consistently. Final scores: RedTeam-TG **2,400 pts**, Government **1,700 pts**, L3arn3rs **1,200 pts**.

<img src="/assets/ecowas-2026/ecowas-p2-koth-leaderboard.png" alt="Togo KOTH country leaderboard: RedTeam-TG in 1st place with 2,400 points, ahead of Government at 1,700 and L3arn3rs at 1,200" style="border-radius: 10px; width: 65%;" />

**1st in Togo.** The top 2 teams advance to Phase 3. We went through in first.

---

## Phase 3: Battleground (April 30, 2026)

**8 participants. 2 Togolese teams. We placed 1st. We were the only team to score.**

The final phase was a Boot2Root: one machine, two flags, find `user.txt` then `root.txt` as fast as possible. The format is unforgiving. Going too slow lets the other team finish first, but chasing the wrong path costs time you cannot get back.

<img src="/assets/ecowas-2026/ecowas-p3-battleground.png" alt="Togo BattleGround challenge page showing the Boot2Root format for Phase 3, April 30, 2026" style="border-radius: 10px; width: 65%;" />

The machine was named **SeaFish**, worth 500 points. We solved it. The other qualified team, Government, did not score.

<img src="/assets/ecowas-2026/ecowas-p3-battleground-leaderboard.png" alt="Togo BattleGround final leaderboard: RedTeam-TG in 1st with 500 points, the only team to complete the Boot2Root challenge" style="border-radius: 10px; width: 65%;" />

Final leaderboard: RedTeam-TG **500 pts**, Government **0 pts**.

**1st in Togo. Again.**

---

## The result

We finished first in Togo at every phase of the national qualifier. In Phase 1 we also placed first across the entire continent, out of 1,035 registered participants from every ECOWAS country.

RedTeam-TG is qualified for the ECOWAS Grand Final in Ghana.

I will be honest about what that took. The Jeopardy phase required breadth: you had to perform across cryptography, web exploitation, forensics, binary exploitation, steganography, OSINT, and mobile security in a single session. The KOTH was a completely different kind of pressure: automation, reaction speed, and the ability to flip between offense and defense without losing your footing. The Boot2Root rewarded clean methodology and composure under time pressure. These three skill sets do not overlap much. Getting through all three phases in first place is not a coincidence.

Huge respect to all the team members. This was a collective result at every stage.

To the Togolese teams who competed hard in Phase 1 (Government, FireTeam, L3arn3rs, Root Access TG): the level was high and that made the qualifier worth winning.

See you in Ghana.
