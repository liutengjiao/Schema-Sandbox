# FAQ

## Is Schema Sandbox a product?

This repository publishes a public methodology and draft interoperability contract. It can support products, SDKs, API gateways, industrial middleware, or agent runtimes, but this repository itself is not a production runtime.

## Is it free to use commercially?

Yes. The public methodology content is licensed under CC BY 4.0, which allows commercial use with attribution. Example code and JSON schemas are provided under Apache-2.0 unless otherwise stated.

## Do I need to subscribe to use it?

No. The public license is not conditional on subscription. Subscription channels may be used for updates, methodology packs, case studies, and adopter communication.

## Can I build my own implementation?

Yes, subject to the licenses and attribution requirements. However, production use requires your own security review, implementation testing, compliance checks, and operational controls.

## Can I call my product Schema Sandbox?

No, not without separate permission. The public licenses do not grant trademark rights.

## How is this different from MCP?

MCP helps AI applications connect to external tools and resources. Schema Sandbox focuses on what happens after a tool or capability is available: input contracts, permission scopes, tool-call authorization, output validation, rejection feedback, and evidence records.

## How is this different from containers?

Containers isolate processes. Schema Sandbox defines agent-level execution contracts and auditability. In a mature implementation, both may be used together.

## What is SIP-Core?

SIP-Core is the minimal public draft manifest profile for describing a sandbox package: metadata, input contract, output contract, permission scope, error codes, and optional evidence requirements.
