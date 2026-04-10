# 🤝 Contributing

Thank you for your interest in improving this guide. Contributions are welcome — from typo fixes to entirely new hardening topics.

---

## 📋 Before You Contribute

- **Test on a real Fedora system.** Do not submit configurations you haven't verified yourself.
- **Read the existing docs first.** Your contribution might already be covered, or conflict with an existing decision.
- **Understand the tradeoffs.** Every hardening measure has a cost (compatibility, performance, usability). Document it.

---

## ✅ What We Accept

- Bug fixes (wrong commands, outdated paths, broken verification steps)
- Updates for new Fedora releases (package names, service changes, kernel parameters)
- New hardening topics not yet covered (with full reproduction + verification)
- Improvements to existing docs (clearer explanations, better structure)
- Hardware-specific additions (AMD, ARM, laptop-specific sections)
- Translations (place in `/docs/<lang>/`, e.g. `/docs/de/`)

## ❌ What We Don't Accept

- Theoretical configurations that haven't been tested on real hardware
- Changes that break Fedora defaults without documenting the impact
- Vendor-specific recommendations (e.g. "use product X") without a generic alternative
- Automated scripts — this guide is intentionally manual so users understand every step
- Personal system details (IPs, MACs, UUIDs, usernames, hostnames)

---

## 📝 Contribution Format

Every new section or document must include:

1. **What** — what is being configured and what it does
2. **Why** — the threat it mitigates and why it matters
3. **How** — exact commands to reproduce the configuration
4. **Verify** — commands to confirm the configuration is active
5. **Tradeoffs** — what breaks, what's the performance impact, what are the alternatives

Use `<YOUR-PLACEHOLDER>` for any system-specific values (see [conventions](docs/00-overview.md#conventions-used-in-this-guide)).

---

## 🔀 Pull Request Process

1. **Fork** the repository
2. **Create a branch** from `main` (`git checkout -b topic/your-topic`)
3. **One topic per PR** — don't mix firewall changes with browser hardening
4. **Test your changes** on Fedora 43+ before submitting
5. **Include verification output** showing the configuration is active
6. **Open a PR** with a clear description of what changed and why

---

## 🐛 Reporting Issues

If you find an error, outdated information, or a security concern:

1. Open an **Issue** with a clear title
2. Include your Fedora version and kernel version
3. Show the expected vs actual behavior
4. If it's a security concern, please note this in the issue title

---

## 📜 License

By contributing, you agree that your contributions will be licensed under the same [CC BY-SA 4.0](LICENSE) license as the rest of this project.
