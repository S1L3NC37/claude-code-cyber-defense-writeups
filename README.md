# claude-code-cyber-defense-writeups

Write-ups from working through *AI Cyber Defense Ops*, a course on using the Claude ecosystem (Claude Code, Claude Desktop, MCP) for blue-team security work.

These are learning notes, not polished tutorials. I've left the wrong turns, the dead ends, and the questions in, because working out what broke and why was most of the learning.

## Write-ups

**[Building my first tool with Claude Code](building-my-first-tool-with-claude-code.md)**
A Sysmon event parser, built as a first pass through the Claude Code workflow. Context files, sample data before code, plan mode, saving state, and a namespace trap that would have quietly returned nothing.

**[Wrapping a security CLI with MCP](wrapping-a-security-cli-with-mcp.md)**
An MCP server around Hayabusa for EVTX analysis, connected to both Claude Code and Claude Desktop. Covers MCP architecture and the wrapper pattern, plus four things that broke: a schema too loose to constrain anything, a parser that reported clean scans with zero findings, a config split the course predates, and a Windows packaging quirk that hides the Desktop config file somewhere nothing documents.

## Notes

Screenshots live in `images/`. Everything here was built on Windows, so paths and shell commands are PowerShell unless stated otherwise.
