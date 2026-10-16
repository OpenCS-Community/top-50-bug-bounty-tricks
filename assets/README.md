# 🎨 Assets

<div align="center">

```
████████╗ ██████╗ ██████╗     ███████╗ ██████╗     ██████╗ ██╗   ██╗ ██████╗
╚══██╔══╝██╔═══██╗██╔══██╗    ██╔════╝██╔═████╗    ██╔══██╗██║   ██║██╔════╝
   ██║   ██║   ██║██████╔╝    ███████╗██║██╔██║    ██████╔╝██║   ██║██║  ███╗
   ██║   ██║   ██║██╔═══╝     ╚════██║████╔╝██║    ██╔══██╗██║   ██║██║   ██║
   ██║   ╚██████╔╝██║         ███████║╚██████╔╝    ██████╔╝╚██████╔╝╚██████╔╝
   ╚═╝    ╚═════╝ ╚═╝         ╚══════╝ ╚═════╝     ╚═════╝  ╚═════╝  ╚═════╝

        🎯  B U G   B O U N T Y   H U N T I N G   T R I C K S  🎯
              50 Battle-Tested Techniques from Real Hunters
                     by Computer Science Projects
```

</div>

---

## 📂 Project Structure

```
bug-bounty-tricks/
│
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── CHANGELOG.md
│
├── docs/                            # 10 deep-dive trick categories
│   ├── 01-recon.md
│   ├── 02-ssrf-cloud.md
│   ├── 03-race-conditions.md
│   ├── 04-client-side.md
│   ├── 05-auth-session.md
│   ├── 06-business-logic.md
│   ├── 07-cache-infra.md
│   ├── 08-api-modern.md
│   ├── 09-mobile.md
│   └── 10-reporting-mindset.md
│
├── cheatsheets/                     # Quick-reference guides
│   ├── payloads.md
│   ├── dorks.md
│   ├── tools.md
│   └── projects.md
│
└── assets/                          # ← you are here
    ├── README.md
    ├── banner.svg                   # Vector repo banner
    ├── banner.png                   # PNG banner (social preview)
    ├── logo.svg                     # Project logo
    ├── diagrams/                    # Attack flows & architecture
    │   ├── ssrf-dns-rebinding.svg
    │   ├── race-condition-timeline.svg
    │   ├── cspt-chain.svg
    │   └── cache-poisoning-flow.svg
    ├── screenshots/                 # Redacted tool output
    │   ├── burp-single-packet.png
    │   ├── jwt-tool-output.png
    │   └── graphql-introspection.png
    └── icons/                       # Category icons for README
        ├── recon.svg
        ├── ssrf.svg
        ├── race.svg
        ├── client-side.svg
        ├── auth.svg
        ├── logic.svg
        ├── cache.svg
        ├── api.svg
        ├── mobile.svg
        └── reporting.svg
```

> Subfolders are created when the first file is added. Empty folders don't
> show up on GitHub.

---

## 🎓 Motivation

> *"A picture is worth a thousand words — but a good diagram is worth a
> thousand exploits."*

This repo covers **50 tricks across 10 categories**. That's a lot of concepts
to hold in your head. The visuals in this folder aren't decoration — they're
**learning tools**:

- They help beginners **see the attack** instead of just reading about it.
- They help experienced hunters **remember and re-apply** techniques.
- They help contributors **explain new tricks** clearly.
- They make the whole repo **feel like a cohesive product**, not a random
  dump of notes.

Every diagram, every icon, every redacted screenshot makes the next hunter's
journey a little smoother. That's the point.

**If you've learned something from this repo — give back by making it
clearer.** 🎯

### Words from the Builders

The greatest technologists didn't get there by playing it safe. They embraced
the hard path, the failures, and the uncertainty. Here's what they have to say:

> **"People with very high expectations have very low resilience. Resilience
> matters in success. Greatness is not intelligence. Greatness comes from
> character."**
>
> — **Jensen Huang**, NVIDIA CEO [citation:1][citation:14]

> **"In order to do something incredible, you have to suffer. It's not a walk
> in the park. If it's a walk in the park, anybody could do it."**
>
> — **Jensen Huang**, NVIDIA CEO [citation:2][citation:15]

> **"Failure is an option here. If things are not failing, you are not
> innovating enough."**
>
> — **Elon Musk**, Tesla & SpaceX CEO [citation:3][citation:10]

> **"The greatest successes come from having the freedom to fail."**
>
> — **Mark Zuckerberg**, Meta CEO [citation:12]

> **"When you have the choice to work on something hard, say yes."**
>
> — **Sundar Pichai**, Google CEO [citation:7]

> **"Move fast and break things. The idea is that if you never break
> anything, you're probably not moving fast enough."**
>
> — **Mark Zuckerberg**, Meta CEO [citation:18]

---

> **Back to:** [README](../README.md)
