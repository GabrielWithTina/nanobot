# Notes
- In Cli mode, when processing a user input, the CLI refuse more user input until the previous procecessing is completed
- In Gateway mode, we can send multiple messages from QQ, but they are processed squentially.
  - For each session, the input messages are in the same queue and processed sequentially by using a global lock. (This is not applicable in production environment)
