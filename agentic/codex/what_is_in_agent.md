## Logging
{condition: long-run debugging task}
On running the task, log your findings and progress in codex_logs/{date}_{taskname}_codex_log.txt, to make tracing and changelog inspection easier.

## Waiting User Answers
When using `request_user_input` (the ask-user question tool), wait indefinitely for an explicit user response unless the user explicitly requests a timeout for that question.
