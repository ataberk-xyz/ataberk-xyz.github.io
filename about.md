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
   <li><a href="https://www.t-mobile.cz/bug-bounty/zed-slavy#:~:text=Ataberk%20Yavuzer%20(0xsaiyajin.github.io)%20%2D%20SQL%20Injection%2C%20Cross%2Dsite%20scripting%20(XSS)%2C%20Remote%20Code%20Execution%20(RCE)">T-Mobile Hall of Fame</a>: XSS, SQLi, and RCE findings.</li>
   <li>Mail.ru Hall of Fame: Cross-Site Scripting findings.</li>

   <h4>Security advisories</h4>
   Vulnerabilities I found and responsibly disclosed in widely-used software:
   <div style="overflow-x:auto;">
   <table>
     <thead>
       <tr><th>Package</th><th>Advisory / CVE</th><th>Severity</th><th>Finding type</th><th>Title</th></tr>
     </thead>
     <tbody>
       <tr>
         <td>install-artifact-from-github</td>
         <td><a href="https://github.com/uhop/install-artifact-from-github/security/advisories/GHSA-88q3-gch3-5396">GHSA-88q3-gch3-5396</a></td>
         <td>High (7.5)</td>
         <td>CWE-494 &mdash; Download of Code Without Integrity Check</td>
         <td>network-downloaded native addon loaded without integrity check &rarr; install-time RCE</td>
       </tr>
       <tr>
         <td>stream-json</td>
         <td><a href="https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x">GHSA-528h-pc64-c93x</a></td>
         <td>Moderate (6.2)</td>
         <td>CWE-407 &mdash; Inefficient Algorithmic Complexity</td>
         <td>quadratic-complexity path filters &rarr; small nested JSON blocks the event loop (DoS). Fixed in 3.5.0</td>
       </tr>
       <tr>
         <td>node-re2</td>
         <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-6hxr-mr5r-9836">GHSA-6hxr-mr5r-9836</a></td>
         <td>Moderate (6.2)</td>
         <td>CWE-835 &mdash; Infinite Loop</td>
         <td>zero-width global match never advances &rarr; infinite loop with unbounded native memory growth (DoS). Fixed in 1.25.2</td>
       </tr>
       <tr>
         <td>node-re2</td>
         <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-ff84-5f28-78qj">GHSA-ff84-5f28-78qj</a></td>
         <td>Moderate (5.7)</td>
         <td>CWE-125 &mdash; Out-of-bounds Read</td>
         <td>attacker-influenced <code>lastIndex</code> on a non-ASCII subject &rarr; out-of-bounds heap read and uncatchable crash (DoS)</td>
       </tr>
       <tr>
         <td>node-re2</td>
         <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-8hcv-x26h-mcgp">GHSA-8hcv-x26h-mcgp</a></td>
         <td>Moderate (6.2)</td>
         <td>CWE-617 &mdash; Reachable Assertion</td>
         <td><code>replace</code> with an output-amplifying template aborts the Node process past V8's max string length (DoS)</td>
       </tr>
       <tr>
         <td>Kentico CMS</td>
         <td><a href="https://nvd.nist.gov/vuln/detail/CVE-2019-19493">CVE-2019-19493</a></td>
         <td>Medium (5.4)</td>
         <td>CWE-434 &mdash; Unrestricted Upload of File with Dangerous Type</td>
         <td>file upload whose Content-Type is inconsistent with its extension &rarr; stored XSS (&le; 12.0.49). Fixed in 12.0.50</td>
       </tr>
     </tbody>
   </table>
   </div>

   <h4>Open source</h4>
   <li><a href="https://github.com/gossipcat-ai/gossipcat-ai">gossipcat-ai</a>: a multi-agent code-review orchestrator (TypeScript / MCP) where agents cross-verify each other's findings against real code to filter hallucinations.</li>

   <style>
     .skills { margin: 0.6rem 0 1.1rem; }
     .skill-group { display: flex; gap: 0.85rem; align-items: baseline; flex-wrap: wrap; margin: 0.55rem 0; }
     .skill-group > .domain {
       flex: 0 0 9rem;
       font-variant: small-caps; letter-spacing: 0.045em;
       color: var(--ink-3); font-size: 0.86rem; font-weight: 600;
     }
     .skill-tags { display: flex; flex-wrap: wrap; gap: 0.4rem; flex: 1 1 16rem; }
     .skill-tags span {
       display: inline-block; padding: 0.22rem 0.62rem;
       background: var(--accent-soft); color: var(--accent);
       border: 1px solid transparent; border-radius: 999px;
       font-size: 0.82rem; line-height: 1.35;
       transition: border-color 0.15s ease, transform 0.15s ease;
     }
     .skill-tags span:hover { border-color: var(--accent); transform: translateY(-1px); }
     @media (max-width: 34rem) {
       .skill-group { flex-direction: column; gap: 0.3rem; }
       .skill-group > .domain { flex-basis: auto; }
     }
   </style>
   <h4>Skills</h4>
   <div class="skills">
     <div class="skill-group">
       <span class="domain">Offensive</span>
       <div class="skill-tags">
         <span>Internal pentest</span><span>External pentest</span><span>Web app security</span><span>Active Directory</span><span>Kerberoasting</span><span>NTLM relaying</span><span>Pass-the-Hash</span><span>Phishing &amp; social engineering</span>
       </div>
     </div>
     <div class="skill-group">
       <span class="domain">Smart contracts</span>
       <div class="skill-tags">
         <span>Solidity</span><span>EVM</span><span>Move</span><span>Aptos</span><span>Sui</span><span>IOTA</span><span>DeFi security</span><span>Threat modeling</span>
       </div>
     </div>
     <div class="skill-group">
       <span class="domain">Reversing &amp; vuln research</span>
       <div class="skill-tags">
         <span>n-day research</span><span>Patch diffing</span><span>WinDbg</span><span>PoC development</span>
       </div>
     </div>
     <div class="skill-group">
       <span class="domain">Languages &amp; tooling</span>
       <div class="skill-tags">
         <span>Python</span><span>TypeScript</span><span>Solidity</span><span>Foundry</span>
       </div>
     </div>
     <div class="skill-group">
       <span class="domain">AI &amp; automation</span>
       <div class="skill-tags">
         <span>Multi-agent orchestration</span><span>gossipcat</span><span>MCP</span><span>Claude Code</span><span>Cursor</span>
       </div>
     </div>
   </div>
   <p>Past member of <a href="https://canyoupwn.me/">CanYouPwn.me</a>.</p>
</div>
