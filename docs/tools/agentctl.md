agentctl

---

The agentctl should provide single universal tool for managing Ligato agents.

![agentctl](../img/tools/agentctl.png)

## Goals

- Provide tool for quick & easy management of agent(s) 
- It should be practical client for users (configuring, control..),
- It should also be handy utility for developers (debug, profiling, stats..)
- Automatically provide CLI for any agent => it should work with custom agents
- Generate config for models using CLI commands => unset fields use default values
- It should replace old unused tools (which will be later deprecated):
  - agentctl (had good start, but was not finished enough to be useful)
  - vpp-agent-ctl (has hard-coded values, not very used)
- Show or watch agent status
- Change agent settings on-the-fly => optionally save them to config file
- Work with all NB accessors = > gRPC, REST or KV store

