---
# ================================================================================
#       FIXED, DO NOT MODIFY THIS FILE
# ================================================================================
weight: 12
title: "Next Steps"
layout: "learningpathall"
---

## Next steps

Now that you have a working SLM deployment on the Alif E8, here are some directions to explore:

### Try FWAuto's AI-assisted workflow

Launch [FWAuto](https://fwauto.ai/) from the project root and type natural language requests:

```bash
fwauto
```

Examples:
- "Add a new prompt preset for 'rocket' to the web dashboard"
- "Explain how the BPE tokenizer works in this codebase"
- "Change the model to generate 100 tokens instead of 50"

### Customize the firmware

The firmware source is in `alif_vscode-template/stories260k_runner/`. You can modify `main.c` to change the prompt format, adjust `max_new_tokens` to control output length, or add new sampling strategies.

After making changes, rebuild and re-flash with `/build` and `/deploy`.

### Customize the web dashboard

The web GUI is in `web_demo.html` and `web_demo_server.py`. You can add new quick-prompt buttons, change the visual theme, or integrate with other tools via the [REST API](https://restfulapi.net/).

The server exposes these API endpoints:

| Endpoint | Method | Description |
|---|---|---|
| `/api/status` | GET | Current board state and history |
| `/api/prompt` | POST | Send a prompt (`{"prompt": "text"}`) |
| `/api/reset` | POST | Reset the board connection |
| `/api/stream` | GET | SSE stream of real-time output |

### Learn more

- [llama2.c](https://github.com/karpathy/llama2.c) -- Karpathy's original project with the stories260K model
- [Alif E8 DevKit documentation](https://alifsemi.com/support/kits/ensemble-e8devkit/) -- Hardware reference
- [CMSIS documentation](https://arm-software.github.io/CMSIS_6/) -- Arm's Cortex-M software interface
- [FWAuto](https://fwauto.ai/) -- AI-assisted firmware development tool
