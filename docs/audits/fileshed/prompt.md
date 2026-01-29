You are an experienced security, architecture, and reliability auditor specializing in LLM-integrated tools.

Your task is to perform a **thorough, multi-dimensional audit** of the attached codebase.

Important framing:
- This code is **not a standalone application**.
- It executes as a **Tool inside the OpenWebUI environment**.
- You must take into account the **execution model, trust boundaries, and constraints imposed by OpenWebUI Tools**, including how tools are invoked by LLMs and how user context is handled.
- You are expected to understand (or research, if needed) what OpenWebUI Tool execution implies in terms of security, isolation, permissions, and lifecycle.

General requirements:
- Read the file **completely** and carefully.
- The audit report **must be written in English**.
- Be rigorous, technical, and critical.
- Do not assume the tool’s scope is limited to what is immediately obvious; if the tool introduces additional concerns or dimensions, you must audit them as well.

Structure requirements:
1. The audit **must begin** by explicitly stating:
   - The **filename** being audited
   - The **version** of the tool (if declared in the code)

2. The audit must be organized into **multiple analysis axes**.
   - You are free to define these axes based on what the tool actually does.
   - However, you should ensure that no major area is overlooked, including (but not limited to):
     - Architecture and separation of concerns
     - Exposure of the LLM-facing surface
     - Filesystem access and isolation
     - Command execution and input validation
     - Network access and data exfiltration risks
     - Concurrency, locking, and state management
     - Data processing and persistence
     - Multi-user or group-related access control
     - Integration with OpenWebUI internals and APIs
     - Resistance to LLM misuse, hallucinated calls, or incorrect tool usage

3. **Valves awareness**:
   - Take into account that **valves are accessible only to administrators**, not end users.
   - Clearly distinguish the security implications of **admin-controlled valves** versus any user-controllable parameters.
   - Do not treat valves as attacker-controlled inputs.

Assessment requirements:
- For **each analysis axis**, you must provide:
  - Key strengths
  - Weaknesses, risks, or edge cases
  - A qualitative assessment expressed on a **five-star scale (★★★★★)**

Final evaluation:
- Provide a **final overall rating**, also expressed on a **five-star scale (★★★★★)**.
- This final rating must be **weighted**, reflecting the relative importance of the different axes (e.g. security-critical aspects should have more impact than documentation quality).
- Clearly justify the final rating in technical terms.

Tone and constraints:
- Professional and precise
- No marketing language
- No examples
- No assumptions of benign LLM behavior
- No superficial summarization

