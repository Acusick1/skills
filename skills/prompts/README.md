# Prompts

Reusable prompt text: instructions you would otherwise type before or after a request, such as "answer only, change nothing". Not part of upstream.

Each prompt is a user-invoked skill, so it works in every harness that reads Agent Skills, is never triggered by the model, and installs with `scripts/link-skills.sh`. Type `/<name> <your request>` (Codex: `$<name> <your request>`).

## Prompts

- [ask](./ask/SKILL.md): answer the request without changing anything.

## Adding a prompt

1. Create `skills/prompts/<name>/SKILL.md`. Keep the name short; it is what you type.

   ```markdown
   ---
   name: <name>
   description: <one line: what the prompt makes the agent do>
   disable-model-invocation: true
   ---

   <the prompt, written to apply to the request that accompanies it>
   ```

   Refer to "the request" rather than an argument placeholder: agents pass the text typed after the name in different ways, and this wording reads the same in all of them.
2. Add `agents/openai.yaml` with `policy: allow_implicit_invocation: false`, so Codex only runs it when named.
3. Add it to the list above, then run `scripts/link-skills.sh`.

A prompt that grows steps, branches or checks is a skill: move it to `skills/personal/`.
