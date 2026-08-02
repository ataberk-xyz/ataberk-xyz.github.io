---
layout: page
title: whoami
permalink: /about
sidebar_link: true
cover_hero: true
cover_split: true
kicker: "N°00 · ABOUT · ATABERK YAVUZER"
description: "Smart contract auditor & offensive security engineer — 7+ years across Web3 and traditional security. I break things, then build tools so they break less."
---

<dl class="ledger-block">
  <div class="lrow"><dt>OPERATOR</dt><span class="dots"></span><dd>Ataberk Yavuzer</dd></div>
  <div class="lrow"><dt>ROLE</dt><span class="dots"></span><dd>Smart Contract Auditor · Offensive Security</dd></div>
  <div class="lrow"><dt>STACK</dt><span class="dots"></span><dd>Solidity · Move · EVM · Active Directory</dd></div>
  <div class="lrow"><dt>CERTS</dt><span class="dots"></span><dd>OSCP · OSWE · CRTP · C-AI/MLPen</dd></div>
  <div class="lrow"><dt>TRACK</dt><span class="dots"></span><dd>ex-Hacken (Principal) · ex-Halborn (Lead)</dd></div>
  <div class="lrow"><dt>STATUS</dt><span class="dots"></span><dd class="sev">BREAKING THINGS ✓</dd></div>
</dl>

<p class="about-intro">I started in penetration testing — web apps, internal and external networks, Active Directory — and moved into blockchain security, auditing Solidity and Move. Most recently <strong>Principal Smart Contract Auditor at Hacken</strong> (2023–2026), leading the auditor team and owning audit delivery; before that Lead Offensive Security Engineer at Halborn. Lately I build tooling like <a href="https://github.com/gossipcat-ai/gossipcat-ai">gossipcat-ai</a>.</p>

<p class="about-links"><a href="https://www.linkedin.com/in/ataberkyavuzer/">LinkedIn</a> · <a href="https://github.com/ataberk-xyz">GitHub</a></p>

<h2 class="about-h">Certifications</h2>
<div class="about-index">
  <a class="ai-row" href="https://www.credly.com/badges/0bff9cfc-f08d-423b-a320-0cdb431e9f45"><span class="ai-name">Offensive Security Certified Professional</span><span class="ai-fill"></span><span class="ai-tag">OSCP</span></a>
  <a class="ai-row" href="https://www.credly.com/badges/c6ffd1d7-b4a0-40ff-9378-3fbc29d64ea1"><span class="ai-name">Offensive Security Web Expert</span><span class="ai-fill"></span><span class="ai-tag">OSWE</span></a>
  <a class="ai-row" href="https://www.credential.net/c96c3313-448d-4518-a39f-ec02cee89478"><span class="ai-name">Certified Red Team Professional</span><span class="ai-fill"></span><span class="ai-tag">CRTP</span></a>
  <a class="ai-row" href="https://pentestingexams.com/certificate-validation"><span class="ai-name">Certified AI/ML Pentester</span><span class="ai-fill"></span><span class="ai-tag">C-AI/MLPen</span></a>
</div>

<h2 class="about-h">Recognition</h2>
<div class="about-index">
  <div class="ai-row"><span class="ai-name">n-day research on a Microsoft SQL Server stack overflow — analyzed the bug, wrote a working exploit (<a href="/vulnerability-research/2021/02/06/discovering-an-undisclosed-stack-overflow-vulnerability-in-mssql-server-cve-2019-1068.html">writeup</a>)</span><span class="ai-fill"></span><span class="ai-tag">CVE-2019-1068</span></div>
  <div class="ai-row"><span class="ai-name"><a href="https://www.t-mobile.cz/bug-bounty/zed-slavy#:~:text=Ataberk%20Yavuzer%20(0xsaiyajin.github.io)%20%2D%20SQL%20Injection%2C%20Cross%2Dsite%20scripting%20(XSS)%2C%20Remote%20Code%20Execution%20(RCE)">T-Mobile Hall of Fame</a> — XSS, SQLi, RCE</span><span class="ai-fill"></span><span class="ai-tag">HOF</span></div>
  <div class="ai-row"><span class="ai-name">Mail.ru Hall of Fame — Cross-Site Scripting</span><span class="ai-fill"></span><span class="ai-tag">HOF</span></div>
  <div class="ai-row"><span class="ai-name"><a href="https://github.com/gossipcat-ai/gossipcat-ai">gossipcat-ai</a> — multi-agent code-review orchestrator (TypeScript / MCP); agents cross-verify findings against real code to filter hallucinations</span><span class="ai-fill"></span><span class="ai-tag">OSS</span></div>
</div>

<h2 class="about-h">Security advisories</h2>
<p class="about-note">Found and responsibly disclosed in widely-used software.</p>
<div class="adv-wrap">
<table class="adv">
  <thead>
    <tr><th>Package</th><th>Advisory</th><th>Severity</th><th>Impact</th></tr>
  </thead>
  <tbody>
    <tr>
      <td>install-artifact-from-github</td>
      <td><a href="https://github.com/uhop/install-artifact-from-github/security/advisories/GHSA-88q3-gch3-5396">GHSA-88q3-gch3-5396</a></td>
      <td><span class="sv sv-high">HIGH 7.5</span></td>
      <td><span class="cwe">CWE-494</span> native addon downloaded over the network and loaded with no integrity check → install-time RCE</td>
    </tr>
    <tr>
      <td>stream-json</td>
      <td><a href="https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x">GHSA-528h-pc64-c93x</a></td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-407</span> quadratic path filters → a small nested JSON blocks the event loop. Fixed in 3.5.0</td>
    </tr>
    <tr>
      <td>node-re2</td>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-6hxr-mr5r-9836">CVE-2026-68499</a></td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-835</span> zero-width global match never advances → infinite loop, unbounded native memory. Fixed in 1.25.2</td>
    </tr>
    <tr>
      <td>node-re2</td>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-ff84-5f28-78qj">CVE-2026-67550</a></td>
      <td><span class="sv sv-med">MEDIUM 5.7</span></td>
      <td><span class="cwe">CWE-125</span> attacker-influenced <code>lastIndex</code> on a non-ASCII subject → out-of-bounds heap read, uncatchable crash</td>
    </tr>
    <tr>
      <td>node-re2</td>
      <td><a href="https://github.com/uhop/node-re2/security/advisories/GHSA-8hcv-x26h-mcgp">GHSA-8hcv-x26h-mcgp</a></td>
      <td><span class="sv sv-med">MEDIUM 6.2</span></td>
      <td><span class="cwe">CWE-617</span> <code>replace</code> with an output-amplifying template aborts Node past V8's max string length</td>
    </tr>
    <tr>
      <td>Kentico CMS</td>
      <td><a href="https://nvd.nist.gov/vuln/detail/CVE-2019-19493">CVE-2019-19493</a></td>
      <td><span class="sv sv-med">MEDIUM 5.4</span></td>
      <td><span class="cwe">CWE-434</span> upload whose Content-Type disagrees with its extension → stored XSS (≤ 12.0.49). Fixed in 12.0.50</td>
    </tr>
  </tbody>
</table>
</div>

<h2 class="about-h">Skills</h2>
<dl class="skill-grid">
  <div class="sk-row">
    <dt>Offensive</dt>
    <dd>Network &amp; web app pentest · Active Directory (Kerberoasting, NTLM relaying, Pass-the-Hash) · phishing &amp; social engineering</dd>
  </div>
  <div class="sk-row">
    <dt>Smart contracts</dt>
    <dd>Solidity · EVM · Move (Aptos, Sui, IOTA) · DeFi security · threat modeling</dd>
  </div>
  <div class="sk-row">
    <dt>Vuln research</dt>
    <dd>n-day research · patch diffing · WinDbg · PoC development</dd>
  </div>
  <div class="sk-row">
    <dt>Build &amp; AI</dt>
    <dd>Python · TypeScript · Foundry · multi-agent orchestration (MCP) · Claude Code</dd>
  </div>
</dl>
