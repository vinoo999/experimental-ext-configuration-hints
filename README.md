# Configuration Hints Extension

> **Experimental:** This extension currently has experimental status. Breaking changes are possible and community feedback is much appreciated. To learn more about the extension lifecycle, please refer to [A2A Extension and Protocol Binding Governance](https://a2a-protocol.org/latest/topics/extension-and-binding-governance/).

Staging note: this repository is staged under `vinoo999` pending TSC review and maintainer sponsorship. It is intended to move to `a2aproject/experimental-ext-configuration-hints`. The extension URI, `https://a2a-protocol.org/extensions/configuration-hints/v1`, is an identifier and does not change when the repository moves.

This repository contains the specification for the **Configuration Hints Extension** for the Agent2Agent (A2A) protocol.

## Purpose

Model APIs let the caller choose the model, the reasoning effort, and the output length. An agent delegating to another agent over A2A has no standard way to say the same things.

This extension lets an agent advertise, on its Agent Card, a JSON Schema of the harness settings a client may hint, with well-known properties such as `reasoningEffort`, `model`, output limits, and client-supplied `tools` and `agentSkills`. The client sends values in `SendMessageRequest.metadata`, keyed by the extension URI. Hints are not commands: if the agent cannot honor one, it proceeds with its defaults instead of failing.

## Specification

The full v1 specification can be found [here](./v1/spec.md).
