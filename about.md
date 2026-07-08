---
layout: page
title: About
permalink: /about
sidebar_link: true
---
<style>
   img { display: unset; margin: 0px; }
   th { text-align: center; }
</style>
<div>
   <h2>whoami?</h2>
   I'm Ataberk, a smart contract auditor and offensive security engineer with 7+ years across Web3 and traditional security. I started out in penetration testing (web apps, internal/external networks, Active Directory) and moved into blockchain security, auditing Solidity and Move smart contracts.
   <br/><br/>
   Most recently I was <strong>Principal Smart Contract Auditor at Hacken</strong> (2023&ndash;2026), where I led the auditor team and owned audit delivery across Solidity and Move engagements. Before that I was Lead Offensive Security Engineer at Halborn. Lately I've been building tools like <a href="https://github.com/gossipcat-ai/gossipcat-ai">gossipcat-ai</a>.
   <br/><br/>
   More detail on my <a href="https://www.linkedin.com/in/ataberkyavuzer/">LinkedIn</a> or <a href="https://github.com/ataberk-xyz">GitHub</a>.

   <h4>Certifications</h4>
   <li><a href="https://www.credly.com/badges/0bff9cfc-f08d-423b-a320-0cdb431e9f45">Offensive Security Certified Professional (OSCP)</a></li>
   <li><a href="https://www.credly.com/badges/c6ffd1d7-b4a0-40ff-9378-3fbc29d64ea1">Offensive Security Web Expert (OSWE)</a></li>
   <li><a href="https://www.credential.net/c96c3313-448d-4518-a39f-ec02cee89478">Certified Red Team Professional (CRTP)</a></li>

   <h4>Recognition</h4>
   <li>CVE-2019-1068: n-day research on a Microsoft SQL Server stack overflow (analyzed the bug and wrote a working exploit). <a href="/vulnerability-research/2021/02/06/discovering-an-undisclosed-stack-overflow-vulnerability-in-mssql-server-cve-2019-1068.html">Writeup</a>.</li>
   <li>T-Mobile Hall of Fame: XSS, SQLi, and RCE findings.</li>
   <li>Mail.ru Hall of Fame: Cross-Site Scripting findings.</li>
   <li>HackingWars CTF #1: finished 1st of 324, hosted by Prodaft.</li>

   <h4>Security advisories</h4>
   Responsibly disclosed vulnerabilities in widely-used npm packages (credited as reporter):
   <li><a href="https://github.com/uhop/install-artifact-from-github/security/advisories/GHSA-88q3-gch3-5396">install-artifact-from-github</a> &mdash; High (7.5): a network-downloaded native addon is written and loaded with no integrity check, giving arbitrary code execution at install time (CWE-494).</li>
   <li><a href="https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x">stream-json</a> &mdash; Moderate (6.2): quadratic-complexity path filters let a small, deeply-nested JSON payload block the event loop for seconds to minutes (CWE-407). Fixed in 3.5.0.</li>
   <li><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-6hxr-mr5r-9836">re2</a> &mdash; Moderate (6.2): a global match on a zero-width pattern never advances, causing an infinite loop with unbounded native memory growth (CWE-835). Fixed in 1.25.2.</li>
   <li><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-ff84-5f28-78qj">re2</a> &mdash; Moderate (5.7): an attacker-influenced <code>lastIndex</code> on a non-ASCII subject triggers an out-of-bounds heap read and an uncatchable process crash (CWE-125).</li>
   <li><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-8hcv-x26h-mcgp">re2</a> &mdash; Moderate (6.2): <code>replace</code> with an output-amplifying template aborts the Node process when the result exceeds V8's maximum string length (CWE-617).</li>

   <h4>Open source</h4>
   <li><a href="https://github.com/gossipcat-ai/gossipcat-ai">gossipcat-ai</a>: a multi-agent code-review orchestrator (TypeScript / MCP) where agents cross-verify each other's findings against real code to filter hallucinations.</li>

   <h4>other stuff</h4>
   Past member of <a href="https://canyoupwn.me/">CanYouPwn.me</a>. On the offensive side I'm comfortable with AD attack vectors like Kerberoasting and NTLM relaying; on the building side, Solidity and Move auditing plus AI-assisted tooling (Claude Code, Cursor).
</div>
