# rinkata-idea

Capture, list, classify, and promote ideas through rinkata MCP.

The skill backs `/idea`, `/rinkata-idea`, idea listing, and related prompts. It
preserves the user's original prose as an `idea` artifact, can list captured
ideas through `rinkata_list_ideas`, asks for confirmation, then promotes to one
of: Goal, standalone spec, anchored feature ticket, unanchored spike, or inbox
note. Standalone spec promotions are read back through `rinkata_read_spec` before
the agent reports what body, acceptance criteria, and provenance were written.
