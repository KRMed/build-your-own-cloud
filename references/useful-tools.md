## Useful Tools

**Optional** - none of these are required, but they might make certain cluster easier or more productive

| Tool | What it does |
|---|---|
| [k9s](https://k9scli.io/) | Terminal UI: browse pods, view logs, exec into containers, watch resources in real time. The most useful tool for daily cluster work. `brew install k9s` / `winget install k9s` |
| [Lens](https://k8slens.dev/) | Desktop GUI for cluster management, log viewing, and resource browsing |
| [kubectx / kubens](https://github.com/ahmetb/kubectx) | Fast context and namespace switching: `kubectx <name>` and `kubens <ns>` instead of long `kubectl config` commands. `brew install kubectx` |