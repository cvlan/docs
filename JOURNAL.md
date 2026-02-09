# CloudVLAN Development Journal

> Notes, learnings, and observations from building CloudVLAN with AI-assisted development. Material for eventual blog post.

---

## 2026-01-19: Project Inception

### Context
Starting a part-time experiment: building a Tailscale alternative with IPsec support, using Claude Code as the primary development partner.

### Key Decisions Made Today

**Technical:**
- Rust for data plane (no GC languages)
- Dual protocol: WireGuard + IKEv2/IPsec
- Cloud-neutral, self-hosted first

**Business:**
- Target 100k clients at <$2k/month infrastructure
- No VC pressure, sustainable business model
- Success = sell, or open-source + blog post

### Observations on AI-Assisted Planning

- Claude Code effectively served as a "product manager" during planning
- Back-and-forth refined scope significantly:
  - Started with generic deep dives (key exchange, NAT traversal)
  - Refined to focus on what we actually build vs. use existing tech
  - Added USP-specific docs (IPsec, vRouter, air-gapped)
- Reality check conversation was valuable—forced explicit answers to hard questions

### What's Different About This Approach

Traditional approach: Hire team → Plan → Build → Hope
This approach: Solo + AI → Iterate fast → Ship or learn

The "no timeline, experiment" framing removes artificial pressure. Outcome is valuable either way.

---

## Template for Future Entries

```markdown
## YYYY-MM-DD: Title

### What I Worked On
- Item 1
- Item 2

### Challenges
- Challenge and how it was resolved

### AI-Assisted Development Notes
- What worked well with Claude Code
- What required human judgment

### Learnings
- Technical learnings
- Process learnings
```
