# 🧠 LLMCodingRules

Hey there! Welcome to LLMCodingRules

## What's This All About?

This is my personal source for creating coding rules in a specific format usable by any coding assistant.
I’m actually using this format in Codash (a coding assistant I created for myself), but it's designed for use with any LLM.
The format works great, and the goal here is to collect prompts with a similar structure in this repository.

---

## 📋 Guideline Collections

| Framework             | Technologies                     | Focus                                               | Status        | Link                                         |
| --------------------- | -------------------------------- |-----------------------------------------------------| ------------- |----------------------------------------------|
| Django + HTMX         | Python, Django, HTMX, Tailwind CSS | Building full-stack Django app with a service layer | ✅ Available  | [View](./python/django/django-htmx-hacksoft.md) |
| FastAPI               | Python, FastAPI                  | API development patterns                            | 🔜 Coming Soon | [View](./guidelines/coming_soon.md)          |
| *Your Framework Here* | *Your Tech Stack*                | *Your Architectural Focus*                          | ⭐ Add Yours! | *Link Here*                                  |

---

## The Core Idea

Here’s the thinking behind this project:

1.  **You Know Your Stuff:** You start by writing down guidelines based on your real-world experience with a specific tech stack – covering the patterns, the potential pitfalls (gotchas), and the reasoning ("why").
2.  **LLM Cleans It Up:** Then, you provide that draft to an LLM using a specific prompt (details below). The LLM refines it into a clean, objective format that's good for pasting into a system prompt.

The goal is to create a collection of prompts that helps:

*   **Share What Works:** Provide guideline formats that have proven effective for you.
*   **Keep Things Consistent:** Help LLMs follow the same rules across an entire project.
*   **Explain the Big Picture:** Guide LLMs on architectural decisions, not just syntax.
*   **Write Maintainable Code:** Promote patterns that prevent code from becoming a headache later on.

---

## 🤝 Contributing

Got your own set of rules for working effectively with LLMs on a particular stack? We'd love to include them!

**What you can share:**

*   Guideline sets for frameworks, languages, or architectural patterns we don't have yet.
*   Suggestions for improving the existing guidelines.
*   Thoughts on different ways to structure rules for LLMs.

### How to Add Your Guidelines

It involves combining your expertise with LLM-assisted refinement and formatting:

1.  **Write It Down:** First, draft your guidelines as if you were explaining them to a colleague. Cover the rules, the architecture, why you do things a certain way, and include concise code snippets as examples. Share your expertise.

2.  **Refine & Format with One LLM Prompt:** Now, take that draft and feed it to your favorite LLM using this single prompt. This prompt asks the LLM to optimize the text for its own use *and* format it correctly for our repository:

    ```prompt
        Please refine the following guidelines for [your framework/topic]. Your goal is to make them objective and optimized for an LLM system prompt, while also formatting the entire result correctly for embedding in a Markdown file and minimizing stylistic Markdown within the text.
        
        Here's what to do:
        1.  Make the text objective, focusing on clear rules and structure.
        2.  Use simple lists (-) for rules where appropriate.
        3.  Keep code examples concise, using '...' where appropriate, but ensure the core patterns are understandable.
        4.  Enclose the *entire* refined output within a single markdown code block (using triple backticks ```markdown).
        5.  Inside the main markdown block, use triple double quotes (e.g., """python ... """ or similar) to delimit any code examples. This prevents formatting conflicts when the main block is embedded in another Markdown file.
        6.  Within the main text content (outside of code examples and standard Markdown headings like `#`, `##`), **strictly avoid** using stylistic Markdown formatting such as bold (`**`), italics (`*`), or inline code ticks (`) for emphasis or highlighting. Use plain text for all descriptions, rules, and explanations. Standard Markdown headings (`#`, `##`, etc.) are acceptable for structure if needed.
        7.  Remember, the primary user for this refined text is an LLM that needs clear, unambiguous coding guidelines presented plainly.
        8.  BE CONCISE, OBJECTIVE. AVOID REDUNDANCY
        9.  CODE SNIPPETS MUST BE SMALL, WITHOUT COMMENTS
        Here are the guidelines to process:
        
        ==================================
        
        [Paste your manually drafted guidelines here]
        
        ==================================
    ```
    *   **The Output:** The LLM should return a single markdown code block (starting with ```markdown and ending with ```). Inside this block will be your refined guidelines, ready to be used as a system prompt, with internal code examples using `"""` delimiters.

3.  **Structure Your Contribution File:**
    *   Create a new Markdown file within the appropriate directory (e.g., `python/yourframework/yourframework-rules.md`). Follow the existing structure if possible.
    *   Your file **must** include:
        *   An **Introduction** explaining the stack and the core approach of your guidelines.
        *   A **Core Principles** section outlining the fundamental ideas.
        *   Sections for specific **Components**, patterns, or concepts relevant to the stack.
        *   A dedicated section (e.g., `## LLM System Prompt`) containing **only the final formatted markdown block** generated in Step 2.
    * You can also paste only the final result of Step 2 as it is kinda self-explanatory.

4.  **Share it!**
    *   Consider opening an **[Issue](link/to/your/issues)** first to discuss your proposed contribution.
    *   Fork the repository.
    *   Add your new `.md` file in the correct location.
    *   **Add a row to the table in *this* README.md** pointing to your new guidelines file.
    *   Open a **[Pull Request](link/to/your/pulls)** with your changes.

---

## How to Use These Guidelines

### Using Guidelines with Your AI Assistant

1.  **Find Your Match:** Pick the guidelines for your tech stack from the table above.
2.  **Copy the Prompt:** Navigate to that guideline's page. Find the section containing the final LLM prompt (usually titled `## LLM System Prompt` or similar). Copy the *entire content* within that final Markdown code block (the one starting with ```markdown).
3.  **Paste & Go:** Paste the copied text into your LLM's system prompt area or use it as the very first instruction in your chat session.
4.  **Refer Back:** During your conversation, you can remind the LLM about specific rules (e.g., "Remember the guidelines regarding service layer implementation...").

---

## License

Distributed under the [MIT License](./LICENSE.md).

---

<p align="center">⭐ If this repository helps you out, please consider giving it a star! ⭐</p>
