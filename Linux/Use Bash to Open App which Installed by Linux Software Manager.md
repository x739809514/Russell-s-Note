*   **Initial Goal:** The user wanted to open Obsidian, installed via Linux Mint's Software Manager, using the bash terminal.
*   **Common Launch Methods Explored (Initial Recommendations):**
    *   Trying the simple executable name (e.g., `obsidian`).
    *   Using `flatpak run <App ID>` if installed as a Flatpak (which is common in Mint).
    *   Using `snap run <app_name>` if installed as a Snap.
    *   Inspecting the `Exec=` line within the application's `.desktop` file for the exact command.
*   **Obsidian Specific Solution:**
    *   The user provided the exact command from Obsidian's properties: `/usr/bin/flatpak run --branch=stable --arch=x86_64 --command=obsidian.sh --file-forwarding md.obsidian.Obsidian @@u %U @@`.
    *   The direct bash command to launch Obsidian (without file arguments) was identified as: `/usr/bin/flatpak run --branch=stable --arch=x86_64 --command=obsidian.sh md.obsidian.Obsidian`.
    *   A slightly shorter, often working, alternative: `flatpak run --command=obsidian.sh md.obsidian.Obsidian`.
*   **Discord Specific Solution (Analogous):**
    *   The user applied the same logic to Discord's command: `/usr/bin/flatpak run --branch=stable --arch=x86_64 --command=com.discordapp.Discord --file-forwarding com.discordapp.Discord @@u %U @@`.
    *   It was confirmed that the same principle applies.
    *   The direct bash command to launch Discord: `/usr/bin/flatpak run --branch=stable --arch=x86_64 --command=com.discordapp.Discord com.discordapp.Discord`.
    *   A shorter, commonly working alternative: `flatpak run com.discordapp.Discord`.
*   **General Method for Creating Bash Aliases:**
    *   To simplify launching these applications, create an alias in `~/.bashrc` (or `~/.zshrc`).
    *   Add a line like `alias <short_command_name>='<full_launch_command>'` (e.g., `alias obsidian='flatpak run --command=obsidian.sh md.obsidian.Obsidian'`).
    *   Save the file.
    *   Apply the changes by running `source ~/.bashrc` in the terminal.
    *   Now, typing the `<short_command_name>` will launch the application.