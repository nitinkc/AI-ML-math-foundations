# The Running Scenario: Meet the Ticket That Connects Everything

Before we go further, here's a single real-world problem that will thread through every module.
When a new concept lands, you'll see it applied to this same ticket — so nothing feels abstract.

---

## The ticket

It's Monday morning. A ticket lands in your service desk queue:

> **Subject:** Can't log in — urgent  
> **Body:** "Hi, I've been locked out of my account since this morning. Also my VPN keeps
> dropping every few minutes. Not sure if these are related but I need access ASAP for
> a client call at 10am. This happened right after the security team sent out that
> patch notice last Friday."

One ticket. Three possible things going on:

| Intent | What it would mean |
|---|---|
| `account_unlock` | User is locked out — password reset or account re-enablement needed |
| `vpn_issue` | VPN client is broken after the Friday patch |
| `security_incident` | The "patch notice" was a phishing email; account may be compromised |

**The stakes are not equal.** An account unlock takes 4 minutes and a tier-1 agent.
A security incident needs a senior engineer, an audit trail, and possibly HR — and
misrouting it as a password reset could let an attacker stay inside the network.

This is the problem your AI copilot has to solve correctly, every time, at scale.
