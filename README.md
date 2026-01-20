# Introduction

This repository contains a set of rules and tools that can be used with Large Language Models to convert CI pipelines from various platforms into Buildkite pipeline YAML files.

If you install the [Buildkite CLI,](https://github.com/buildkite/cli) your LLM agent will use that to validate the YAML that is generated. You can also set up the [Buildkite MCP Server](https://github.com/buildkite/buildkite-mcp-server) to enable the agent to create pipelines for you in your Buildkite org with that YAML.