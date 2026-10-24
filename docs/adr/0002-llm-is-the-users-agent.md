# The LLM is the user's coding agent, not our client

The planning step — transcript to scene plans — is performed by whatever coding agent the user already runs, invoked through a shell-template command in project config, with a file-convention fallback for agents that cannot pipe. We ship the prompt, the JSON Schema, a validator, and a repair loop. We never ship an API key or a model client.

**Consequences.** Cost and model choice stay with the user; the repo stays agent-agnostic by construction. Our output quality then depends on an interface we do not control, and reproducibility means pinning the prompt and schema, not the model. Adding an in-repo provider client later would be a reversal, not an extension.
