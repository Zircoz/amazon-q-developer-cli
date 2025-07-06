**Knowledge Transfer Document: Autocomplete Functionality**

**Overview:**

The autocomplete system provides suggestions to the user as they type in the chat interface. It handles three main types of completions:
1.  **Commands**: Triggered when the user types `/` followed by characters.
2.  **Prompts/Contextual**: Triggered when the user types `@` followed by characters. This is likely for mentioning users, topics, or other contextual items.
3.  **File Paths**: Triggered in other cases, suggesting file and directory names.

Additionally, a "hinter" provides shadowtext suggestions based on history and known commands.

**Core Components (from the reference Rust codebase):**

*   **`InputSource` (`crates/chat-cli/src/cli/chat/input_source.rs`)**:
    *   Responsible for capturing raw user input.
    *   Uses `rustyline` library for line editing, history, and basic input handling.
    *   Sends input to other components for processing.

*   **`ChatCompleter` (`crates/chat-cli/src/cli/chat/prompt.rs`)**:
    *   The central logic unit for generating completion suggestions.
    *   Implements `rustyline::Completer` trait.
    *   Method: `complete(line: &str, pos: usize, ctx: &Context<'_>) -> Result<(usize, Vec<String>), ReadlineError>`
        *   `line`: The current input line.
        *   `pos`: The current cursor position in the line.
        *   `ctx`: `rustyline` context (can be used for history, etc., though not heavily used for completion logic itself in this case).
        *   Returns a tuple: `(start_position_of_word_to_replace, list_of_suggestions)`.
    *   **Logic Flow:**
        1.  `extract_word(line, pos, ...)`: Identifies the word currently being typed by the user.
        2.  **Command Completion**:
            *   If `word.starts_with('/')`:
                *   Calls `complete_command(word, start)` which filters a predefined `COMMANDS` list (static array of strings).
                *   `COMMANDS`: `&[&str] = ["/clear", "/help", ...]`
        3.  **Prompt Completion (`@` mentions):**
            *   If `line.starts_with('@')`: (Note: it checks the whole line, not just the current word)
                *   `search_word = line.strip_prefix('@').unwrap_or("")`
                *   Calls `self.prompt_completer.complete_prompt(search_word)`.
                *   `PromptCompleter`:
                    *   Contains an `mpsc::Sender` and `mpsc::Receiver`.
                    *   `complete_prompt` sends the `search_word` (or `None` if empty) via the sender.
                    *   It then blocks on the receiver to get a `Vec<String>` of suggestions.
                    *   The suggestions are formatted as `@{suggestion}`.
                    *   *Implication for new implementation*: This is the primary hook for dynamic, potentially asynchronous, suggestions (e.g., from an LLM, database, or API). The sender/receiver pattern allows the suggestion generation to happen in a separate thread or task.
        4.  **Path Completion (Fallback):**
            *   If neither command nor prompt completion is triggered:
                *   Calls `self.path_completer.complete_path(line, pos, ctx)`.
                *   `PathCompleter`: Wraps `rustyline::completion::FilenameCompleter`.
                *   Uses standard filesystem operations to list files/directories.
        5.  **No Completions**: If none of the above yield results, returns an empty list of suggestions.

*   **`ChatHinter` (`crates/chat-cli/src/cli/chat/prompt.rs`)**:
    *   Provides inline, non-interactive "shadowtext" hints.
    *   Implements `rustyline::hint::Hinter` trait.
    *   Method: `hint(&self, line: &str, pos: usize, _ctx: &Context<'_>) -> Option<String>`
    *   **Logic Flow:**
        1.  Only provides hints if the cursor `pos` is at the end of the `line`.
        2.  If `line.starts_with('/')`:
            *   Searches `COMMANDS` for a command that starts with `line` and is longer than `line`.
            *   Returns the remaining part of the command as a hint.
        3.  Otherwise (not a command):
            *   Searches its internal `history` (a `Vec<String>` of previous inputs) in reverse order.
            *   Finds a history entry that starts with `line` and is longer.
            *   Returns the remaining part of the history entry.
    *   `update_hinter_history(&mut self, command: &str)`: Called by `InputSource` to add successfully entered lines to the hinter's history.

*   **`ChatHelper` (`crates/chat-cli/src/cli/chat/prompt.rs`)**:
    *   A `rustyline::Helper` struct that bundles `ChatCompleter`, `ChatHinter`, and `MultiLineValidator`.
    *   This is the primary object passed to `rustyline::Editor::set_helper()`.

*   **`rl()` function (`crates/chat-cli/src/cli/chat/prompt.rs`)**:
    *   Initializes the `rustyline::Editor`.
    *   Configures `EditMode` (Emacs/Vi), `CompletionType` (List).
    *   Creates `ChatHelper` instance (wiring up the `PromptCompleter`'s sender/receiver channels).
    *   Sets keybindings (e.g., Alt+Enter for newline, Ctrl+F to accept hint).

**Key Data Structures and Mechanisms:**

*   **`COMMANDS: &[&str]`**: A static array of predefined command strings.
*   **`mpsc` channels (in `PromptCompleter`)**: Used for asynchronous fetching of `@prompt` suggestions. The input loop sends a request for suggestions, and another part of the system (not fully detailed in these files but implied) would listen on this channel, generate suggestions, and send them back.
*   **`rustyline::Editor`**: The core of the input handling.
*   **`rustyline` Traits**: `Completer`, `Hinter`, `Helper`, `Validator`. These define the interface for customizing `rustyline`'s behavior.

**To Replicate in a Different Codebase/Language:**

1.  **Input Library Selection**:
    *   Choose a mature terminal input library for the target language that supports:
        *   Customizable tab-completion.
        *   (Optional) Hinting/autosuggestion.
        *   Line editing, history.
        *   Examples: Python's `prompt_toolkit` or `readline` (via `gnureadline`), Go's `go-prompt`, Java's `JLine`.

2.  **Core Completer Logic**:
    *   Implement a function or class equivalent to `ChatCompleter::complete`.
    *   **Word Extraction**: Need a robust way to identify the "word" the user is currently trying to complete (e.g., using whitespace or other delimiters as appropriate for the syntax).
    *   **Contextual Dispatch**:
        *   **Commands**: If the word/line starts with a specific prefix (e.g., `/`), compare against a predefined list of commands.
        *   **Mentions/Prompts**: If it starts with another prefix (e.g., `@`), this is the hook for dynamic suggestions.
            *   This might involve calling an API, querying a local database, or running a model.
            *   If suggestions are fetched asynchronously, the input library must support this or the completer needs to manage it (e.g., by returning cached results while new ones are fetched, or by having a non-blocking way to check for results).
        *   **File Paths**: Use the language's standard library or a third-party library to list files and directories relevant to the current partial path. Filter out non-relevant files/directories.
    *   **Return Format**: The input library will dictate the expected return format for completions (usually the starting position of the text to be replaced and a list of suggestion strings).

3.  **Hinter Logic (Optional but Recommended)**:
    *   If the chosen library supports it, implement hinting.
    *   Maintain a history of user inputs.
    *   Compare the current input buffer against history and potentially the command list to provide a single, unobtrusive suggestion.

4.  **Configuration and Integration**:
    *   Instantiate and configure the input library.
    *   Pass the custom completer (and hinter) instances to the library.
    *   Set up any necessary keybindings.

5.  **Dynamic Suggestion Source (`@prompts`)**:
    *   This is the most complex part if the suggestions are not static.
    *   Design how the `PromptCompleter` equivalent will get its data.
    *   If it's from an LLM:
        *   The completer will need to send the current partial input (e.g., "my_functio") to the LLM.
        *   The LLM would return a list of possible completions.
        *   Consider caching, debouncing, and cancellation of requests to avoid overwhelming the LLM or the UI.
    *   The sender/receiver pattern seen in the Rust code is a good way to decouple the input loop from the suggestion generation logic, especially if generation is slow.

**Considerations for the New Tool:**

*   **Performance**: Completion suggestions need to appear quickly to be useful. Asynchronous operations for dynamic suggestions are crucial.
*   **Context Awareness**: The more context the completer has (e.g., current project, file type, cursor position within a larger document if applicable), the better the suggestions can be. The provided example is for a chat CLI, so context is mostly the current line.
*   **Configuration**: Allow users to configure aspects of completion (e.g., enable/disable types of completion, configure external suggestion sources).
*   **Error Handling**: Gracefully handle errors from suggestion sources (e.g., network issues when calling an API).
