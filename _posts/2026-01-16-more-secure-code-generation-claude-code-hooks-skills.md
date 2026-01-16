---
layout: post
title: "Trust But Verify: Using Claude Code's Hooks, Skills, and Agents to Generate Code That's Not Totally Insecure"
date: 2026-01-16
categories: [ai, security, automation]
tags: [claude-code, anthropic, secure-coding, sast, semgrep, bandit, devsecops, llm]
author: "Daniel Streefkerk"
excerpt: "How security professionals can leverage Claude Code's extensibility framework to enforce deterministic security checks on AI-generated code, treating AI coding assistants like any other developer on the team."
---

A colleague of mine shared an observation today that captures a fundamental reality of AI-assisted development. Despite him repeatedly pointing out security concerns about Claude Code inserting `eval()` statements, and despite Claude Code agreeing with them, it kept trying to insert them anyway. This perfectly illustrates why we need to treat AI coding tools exactly like any other developer on the team: capable, but requiring the same verification and quality gates we'd apply to human-written code.

Claude Code's extensibility framework (Hooks, Skills, and Subagents) provides the mechanisms to implement this "trust but verify" approach, even if you're not using a full-blown CI/CD pipeline. The goal isn't to create "secure AI" (that's a fool's errand); it's to wrap AI-generated code in the same deterministic security tooling that mature engineering teams already use.

The same principle should apply to all of your interactions with LLMs:
1. Provide them with quality guidelines and a good amount of context
2. Don't take at face value anything that they come back with

## The Core Problem

AI coding assistants can acknowledge security concerns in conversation while still producing vulnerable code. This isn't a flaw to be patched; it's a fundamental characteristic of how large language models work. **They optimise for helpfulness and pattern completion, not security enforcement**. When Claude agrees that `eval()` is dangerous but continues suggesting it, that's not a bug; it's the model doing exactly what it's designed to do.

The solution isn't better prompting. It's deterministic verification using the same SAST tools you'd run in a CI/CD workflow: [Semgrep](https://github.com/semgrep/semgrep), [Bandit](https://bandit.readthedocs.io/en/latest/), [ESLint](https://eslint.org/), etc.

## The Three Pillars

Claude Code provides three complementary features for building security verification into the development workflow.

[**Hooks**](https://code.claude.com/docs/en/hooks-guide) are event-driven scripts that execute at specific points in Claude's workflow. When Claude writes a file, a PostToolUse hook can run [Bandit](https://bandit.readthedocs.io/en/latest/) (for example) against it before the operation completes. When Claude tries to execute a shell command, a PreToolUse hook can block dangerous patterns. These aren't suggestions; they're hard gates that execute as external processes with the ability to block operations entirely.

[**Skills**](https://code.claude.com/docs/en/skills) are structured knowledge packages that give Claude domain expertise and procedural guidance. Think of them as persistent expertise that Claude loads before generating code: security requirements, common vulnerability patterns to avoid, approved libraries, and architectural patterns. Skills reduce the likelihood of insecure code generation in the first place by front-loading security knowledge into Claude's context.

[**Subagents**](https://code.claude.com/docs/en/settings#subagent-configuration) are specialised AI instances with isolated context and restricted tool access. A security audit subagent can review code with focused expertise without the context pollution that accumulates in long development sessions. This matters because during extended coding work, the primary agent's context fills with implementation details, debugging tangents, and feature requirements. A dedicated security subagent starts fresh with only security-relevant context, avoiding the "drift" where security considerations get crowded out by other concerns.

> **Skills suggest, Hooks enforce.** Don't rely on AI understanding security. Verify with deterministic tools.

## Skills: Front-Loading Security Expertise

While Hooks provide reactive enforcement, Skills provide proactive guidance. A well-constructed security skill gives Claude the context to generate better code before it even starts writing.

The community has developed some impressive security-focused skill packages. **Trail of Bits** has published a [skills marketplace](https://github.com/trailofbits/skills) specifically for security research and audit workflows. Their collection includes plugins for smart contract security across six blockchains, variant analysis for finding similar vulnerabilities across codebases, differential review for security-focused code change analysis, and a Semgrep rule creator for custom vulnerability detection. There's also a fix-review skill that verifies remediation commits actually address audit findings without introducing new issues, which is genuinely useful for security teams reviewing developer responses to findings.

**SecOpsAgentKit** provides 25+ security operations skills covering application security, DevSecOps, compliance, threat modelling, and incident response. It includes skills for [Semgrep](https://github.com/semgrep/semgrep), [Bandit](https://bandit.readthedocs.io/en/latest/), [Trivy](https://github.com/aquasecurity/trivy), [Checkov](https://github.com/bridgecrewio/checkov), Gitleaks, and even threat modelling with pytm, essentially wrapping the tools security teams already use in a format Claude can consume.

Another collection worth mentioning is [Reza Rezvani's](https://x.com/RezaRezvaniBln) [**claude-skills**](https://github.com/alirezarezvani/claude-skills?tab=readme-ov-file#-available-skills), which includes a Senior Security Engineer skill covering security architecture, penetration testing methodologies, and cryptography implementation guidance.

What makes Skills valuable isn't just the guidance text. It's the bundled resources. A good security skill includes:

- **Pattern libraries**: Approved implementations for common security functions (authentication flows, input validation, cryptographic operations)
- **Anti-patterns with examples**: Concrete "don't do this" examples showing vulnerable code alongside secure alternatives
- **Tool invocation scripts**: Pre-configured commands for security scanners that Claude can call directly
- **Framework mappings**: References to OWASP, CWE, and MITRE ATT&CK so Claude can cite specific vulnerabilities

The value proposition is straightforward: instead of repeatedly explaining your organisation's security requirements in every conversation, you encode them once as a Skill. Claude loads that expertise at session start and applies it throughout.

### Building Your Own Security Skills

You don't need to rely on community packages. The [**Claude Code Skill Factory**](`alirezarezvani/claude-code-skill-factory`), once again by Reza Rezvani, provides a comprehensive toolkit for generating production-ready skills, and it's an excellent starting point if you want to build custom security guidance for your organisation.

The Skill Factory includes interactive builders, validation tools, and templates that handle the structural boilerplate so you can focus on the security content. Run `/build skill` and answer a few guided questions, or use the `skills-guide` agent for a more conversational approach. The toolkit generates properly formatted SKILL.md files with YAML frontmatter, optional Python implementation files, sample inputs/outputs, and documentation.

For a security-focused skill, you'd want to include:

1. Your organisation's secure coding standard as the core guidance
2. Concrete examples of approved patterns (how you expect authentication to be implemented, which cryptographic libraries to use)
3. Explicit "never do this" patterns with explanations of why they're dangerous
4. Pre-configured commands for your internal security scanning tools that Claude can invoke directly

The Skill Factory also includes a Hook Factory for building the enforcement side of the equation, with templates for common hook patterns and automated installation scripts. This means you can build both the guidance (Skills) and the verification (Hooks) using the same toolkit.

The resulting structure is straightforward: a `SKILL.md` file with YAML frontmatter describing triggers and capabilities, plus optional `references/`, `scripts/`, and `assets/` directories for supporting material.

**Critical point**: Skills alone are not enforcement. Claude can deviate from skill guidance when context is complex or the skill is poorly described. This is why the combined approach matters: Skills reduce the frequency of security issues, Hooks catch what gets through.

## How Hooks Work

Hooks are configured in JSON settings files at user, project, or local level. They fire on specific events like `PreToolUse` (before Claude runs a command or writes a file) or `PostToolUse` (after an operation completes successfully).

Here's what a security-focused hook configuration looks like:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "semgrep --config=auto --config=p/owasp-top-ten . --json | jq -e '.results | length == 0' || (semgrep --config=auto . && exit 2)",
            "timeout": 60
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block-dangerous.sh"
          }
        ]
      }
    ]
  }
}
```

The **Stop hook** runs when Claude finishes responding and is about to hand control back to you. If Semgrep finds issues, it exits with code 2, Claude receives the findings, and continues working to fix them. The **PreToolUse hook** on Bash commands is a safety net for blocking dangerous shell operations before they execute.

Hooks communicate through exit codes:
- **Exit 0**: Operation proceeds normally
- **Exit 2**: Blocking error. The operation is stopped and the error message is shown to Claude

This means when Bandit finds an `eval()` statement, it doesn't matter what Claude agreed to in conversation. The hook exits with code 2, the write is blocked, and Claude receives feedback about what went wrong.

## A Practical Example

Say Claude generates Python code that uses `subprocess.call(cmd, shell=True)` with user input, a classic command injection vulnerability. Here's the workflow with a **Stop hook** in place:

1. Claude completes the code, writes the file, and signals it's finished
2. The Stop hook triggers and runs Bandit against the project
3. Bandit identifies B602 (subprocess call with shell=True) as high severity
4. Hook script exits with code 2, returning the Bandit findings
5. Claude receives the feedback and isn't actually "stopped". Instead, it continues working to fix the issue
6. Claude regenerates the code using `subprocess.run()` with `shell=False` and proper input validation
7. On the next Stop attempt, Bandit passes, and the task completes

The key insight here is **"block-at-commit" not "block-at-write"**. Running SAST on every individual file write can confuse the agent mid-operation, especially when it's making coordinated changes across multiple files. The Stop hook lets Claude complete its plan, then validates the final state. If there are issues, Claude gets the feedback and continues working rather than being interrupted mid-flow.

This happens automatically, without developer intervention. The AI gets feedback from deterministic tooling at the natural checkpoint (when it thinks it's done) and can self-correct in the same session.

## Why Hooks Over MCP Servers

You might be tempted to use MCP (Model Context Protocol) servers for security tooling. Semgrep even provides one. For security enforcement, it's probably not a great idea.

Claude Code has experienced documented issues where MCP tool outputs are silently dropped or misinterpreted. There's a [known bug](https://github.com/anthropics/claude-code/issues/6164) where Semgrep findings returned via MCP are completely ignored even when the underlying scan works correctly. Hooks execute deterministically and capture tool output directly, bypassing the MCP response handling layer entirely.

For security-critical workflows, Hooks are the reliable path. Until MCP's output handling matures, don't trust it for enforcement.

## What This Doesn't Catch

Let's be honest about the limitations. Hooks running SAST tools won't catch:

- **Semantic vulnerabilities**: Broken authorisation logic that's syntactically correct
- **Business logic flaws**: The AI doesn't understand your threat model
- **Novel attack patterns**: SAST tools need signatures for known issues
- **Architectural security problems**: System-wide implications require human review

This approach catches the common, preventable mistakes: hardcoded secrets, dangerous function calls, SQL injection patterns, path traversal vulnerabilities. The stuff that shows up in OWASP Top 10 and that developers (human or AI) introduce through inattention rather than malice.

Human review remains essential for architectural decisions, access control logic, and anything that requires understanding the broader system context.

## Summary: What Each Feature Does

| Feature | Primary Security Function | How Claude Responds |
|---------|---------------------------|---------------------|
| **Skills** | **Guidance**: Sets the rules of engagement and approved patterns | Uses skill context to inform initial code generation |
| **Hooks** | **Enforcement**: Runs external SAST/DAST tools against the code | Receives tool errors (Exit 2) and attempts to self-fix the vulnerability |
| **Subagents** | **Review**: Provides a second pair of eyes with focused expertise | Operates in separate context to audit the primary agent's work |

## The Bottom Line

AI coding assistants are powerful tools, but they require the same verification pipeline you'd apply to any developer's code. Claude Code's extensibility framework provides the mechanisms to implement this: Hooks for deterministic enforcement, Skills for guidance and context, and Subagents for specialised review tasks.

> The goal isn't to make AI "understand" security. It's to wrap AI-generated code in the same proven tooling that catches human mistakes. **Treat Claude like a productive but occasionally careless junior developer who responds well to automated feedback**, and you'll be in roughly the right mindset.

Skills suggest. Hooks enforce. Humans verify architecture.
