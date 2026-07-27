# ZenOS-AI Community Tips

Real-world gotchas, setup variations, and integration notes from the ZenOS community — things that work in specific configurations but don't belong in the main install docs.

**These are not official ZenOS documentation.** They are community-contributed and reflect the author's specific setup. The maintainer does not test or endorse them. Use your judgment.

---

## How to Submit a Tip

1. **Fork** the ZenOS-AI repo
2. **Copy** `tips/_template.md` to `tips/your-descriptive-name.md`
3. **Fill it in** — keep it factual, keep it scoped, include your tested versions
4. **Open a pull request** against `main` with the title `community: <your tip title>`

That's it. Your GitHub account is on the commit. You're in the contributor graph.

The maintainer reviews for accuracy and scope before merging. Tips that are wrong, dangerous, or outside ZenOS scope will be declined with a note. Tips that are solid get merged and indexed.

---

## Guidelines

- **Scope it tight.** One gotcha, one fix. If it needs three prerequisites and covers four plugins, it's too broad.
- **Version-pin it.** State what you tested on. Tips go stale — version info is how future readers know whether to trust it.
- **Don't reproduce the main docs.** If it's already in `getting_started/` or a plugin readme, it doesn't belong here.
- **No credentials, IPs, or personal info** in submitted files. Use placeholders (`<YOUR_LOCAL_HA_IP>`, `<YOUR_TOKEN>`).

---

## Discussion

[Friday's Party community thread](https://community.home-assistant.io/t/fridays-party-creating-a-private-agentic-ai-using-voice-assistant-tools/855862/) — post questions, feedback, and draft ideas there before opening a PR.
