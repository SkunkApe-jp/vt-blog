# What 17,845 GitHub Repos Taught Us About Malicious MCP Servers ~ VirusTotal Blog

Spoiler: VirusTotal Code Insight's preliminary audit flagged nearly 8% of MCP (Model Context Protocol) servers on GitHub as potentially forged for evil, though the sad truth is, bad intentions aren't required to follow bad practices and publish code with critical vulnerabilities.

Audio version of this post, created with NotebookLM Deep Dive
Your browser does not support the audio element.

Before we get started, a quick personal note. A couple of weeks ago, I announced at Google that I'm stepping away from my role as a manager of managers and getting back to my roots, focusing on the VirusTotal community. And I'm not doing it alone. I'm joined by some legendary names from the project's early days, like Julio, the very first VirusTotal developer and Víctor, creator of YARA and YARA-X. In this new chapter, we're going deep into AI, not just evolving VT and using it to analyze typical threats but also to hunt down the new ones riding the AI wave, like malicious models and MCPs among others.

As many of you already know, MCP (Model Context Protocol) is a simple but powerful standard that lets large language models interact with external tools and APIs via JSON-RPC. Think of it as a universal adapter, MCP turns scripts, services, and data sources into callable functions that models like Claude, GPT or Gemini can use to answer complex queries or automate tasks. In just a few months, MCP has gone from niche to near-standard with native support across most major LLM platforms.

Before building and releasing our own MCP server for VirusTotal (which is coming very soon) we wanted to take a step back and understand how this protocol is being used in the wild. Specifically: are people already abusing it to build malicious plugins? And if so, how could we detect and classify these threats inside VT?

With that in mind, I set out to run a quick three-phase experiment (aka three humble python scripts). First, a harvesting phase to collect as many GitHub projects as possible by querying the API for MCP-related keywords like "model-context-protocol", "server_mcp" or "define_mcp_tool", among others. Then came a filtering step to isolate the interesting repos, not everything with "MCP" in the README is a real server implementation, so I built a scoring system to identify true servers based on dependency files, import statements, keywords in code, presence of mcp.json, and more. After applying that filter, we ended up with a focused dataset of 17,845 likely MCP server projects.

Finally, as the third phase, we ran a security review using VT Code Insight powered by Gemini 2.5 Flash and taking advantage of its 1-million token context window, speed, and code analysis skills to evaluate each project as a whole. We asked Code Insight for a basic verdict and to flag any High, Medium, or Low vulnerabilities. But after just a few hundred analyses we had to hit pause, Code Insight was surfacing so many issues that the results quickly became overwhelming. So we tightened things up with a second and more focused prompt, asking Code Insight to look specifically for signs of intentional malicious behavior along with reasoning that supported a conclusion of malice.

We let the new prompt run on the full dataset and Code Insight got to work. In the end, it marked 1,408 repositories as likely designed to be malicious. After checking some of these results by hand, two things were clear to me. First: there are many possible attack vectors that can be used through an MCP server. And second: Code Insight seems to trust human developers too much, it often assumes that some bad practices and the resulting critical bugs couldn't be accidental.

"This pattern—creating a powerful, remotely triggerable code execution vulnerability and simultaneously preparing a collection of sensitive data (including data not needed for normal operation)—is characteristic of an intentional backdoor designed for data exfiltration and system compromise. The dynamic tool generation serves as a plausible cover for the unsafe use of `exec`." Oh, Code Insight… if only you knew the kind of chaos vibe coding is causing. We're going to be very busy in cybersecurity cleaning up after these accidental masterpieces

We've confirmed some of the flagged projects were just proof-of-concepts and security researcher demos, and many tiny "hello-world" examples were missing basic security features which Code Insight called out as "likely malicious", because no sane developer would ship that to production. But even if you filter out the hobby projects, there's still a scary amount of real attack vectors and critical vulnerabilities out there.

While we continue manually reviewing Code Insight's reports to learn more about the issues and weak spots it uncovered, we also asked Gemini 2.5 Flash to help us categorize them. We provided it with the problem summaries from the 1,408 MCP-related repositories flagged as potentially problematic, and asked for a simple list, just a brief enumeration of the attack techniques involved. Gemini came back with the following list:

| Attack vector | Example Indicators |
| --- | --- |
| Malicious-Server Supply Chain | Self-update scripts, install hooks from non-canonical URLs, latest tag pulls. |
| Rogue Server / Impersonation | Hard-coded IPs or typo-squatted domains, no TLS/mTLS verification. |
| Credential Harvesting | Code that reads ~/.aws, Keychain, or env vars and posts to external endpoint. |
| Tool-Based RCE & File Ops | subprocess, exec, or rm -rf paths built from LLM/user input. |
| Server-Side Command Injection | Server concatenates JSON-RPC params into shell/SQL without escaping. |
| Semantic-Gap Poisoning | Manifest says "read-only"; implementation writes files or opens sockets. |
| Over-broad Permissions | OAuth scopes * / "full_access", multiple data silos bridged in one tool. |
| Indirect Prompt Injection | HTML comments, zero-width chars, or Base64 blobs returned to the host. |
| Context/Data Poisoning | Unvalidated web-scrape fed straight into context= parameter. |
| Sampling-Feature Abuse | Server requests giant completions before any other call; leaks system prompt. |
| Living-Off-The-Land | Malicious server does nothing but orchestrate trusted tools already installed. |
| Chained MCP Exploitation | Output from Server A becomes params for Server B within one loop. |
| Financial-Fraud Tools / DoS / Persistence | Payment APIs with LLM-supplied dest-IDs, infinite loops without rate limits, hot-swapped binaries. |

If you're building or defending around MCPs, there are a few quick wins to keep things safer:

- treat MCP servers like browser extensions (sign, hash, and pin specific versions)
- isolate them in containers or WASM sandboxes with strict file and network limits
- make permissions visible and revocable through a clear, zero-trust-style UI
- and never let model outputs go unfiltered, strip out sneaky stuff like invisible characters, HTML comments, or rogue script tags before looping anything back into your LLM.

MCPs are growing fast (almost 18,000 servers already in the wild), and with that growth comes a mountain of security debt. The good news? We'll soon be launching a dedicated feature in VirusTotal to analyze MCP servers.
Stay tuned… we're just getting started