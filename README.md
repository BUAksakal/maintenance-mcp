# maintenance-mcp

An MCP server that turns any AI agent (Claude, Hermes, or your own) into an industrial maintenance assistant: it detects machine faults in a virtual factory from real sensor data, diagnoses them with citations from the maintenance manual, and drafts work orders that require human approval.

## Status

🚧 Work in progress: Phase 0 (engineering foundation: project setup, tooling, CI).

## Why?

When a machine breaks down in a factory, production stops. Technicians have to check sensor charts, dig through thick maintenance manuals, and search old records to find out whether the fault has happened before. This process is slow and the knowledge is scattered. This project gives AI agents the tools to do that work, while the final decision always stays with a human.

## Note on data

The machines in the virtual factory are not connected to a real plant. They are driven by replaying real, open datasets (CWRU Bearing Data, AI4I 2020), and faults are injected on purpose for testing and demos.
