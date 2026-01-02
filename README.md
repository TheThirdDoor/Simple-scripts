# Simple Scripts

**Simple Scripts** is a growing collection of command-line utilities designed to streamline daily workflows. Born from personal necessity, these tools are lightweight, practical, and built to solve recurring headaches.

Currently contains the following scripts:
* [📦 **Script Installer**](#-script-installer) - Easily install shell scripts across different terminal environments.
* [🔪 **Port Assassin**](#-port-assassin) - Instantly free up ports by killing the processes holding them.

## 📦 Script Installer
The **Script Installer** is the engine that drives this repo. It takes a name and a block of code, and turns it into a command you can run anywhere, and it handles the cross-shell compatibility so you don't have to.

### Usage
```bash
script-installer <command name> '<script_body>'
```
_Note: The script body should be enclosed in quotes to handle multi-line input correctly._

### Features
* **Shebang Injection**: Automatically detects missing shebangs and prepends #!/bin/sh to ensure execution.
* **Dynamic Command Naming**: You pick the command name. The installer automatically updates usage instructions inside the script (replacing <command>) to match the name you chose.
* **Instant Docs**: It pulls # Usage comments right out of the code and prints them during install, so you know how to use your new tool immediately.

### Installation
Run the following command in your shell. This bootstrap command is longer than the others because it sets up the environment to make future installations simple.
```bash
{
  # 1. Setup Directory
  TARGET_DIR="$HOME/.local/bin"
  mkdir -p "$TARGET_DIR"
  TARGET_FILE="$TARGET_DIR/script-installer"

  # 2. Write the POSIX compliant tool content
  cat << 'EOF' > "$TARGET_FILE"
#!/bin/sh
# script-installer: Installs script content to ~/.local/bin
# Features:
# 1. Replaces <command> with the installed filename.
# 2. Auto-prepends #!/bin/sh if missing.
# 3. Prints # Usage info if detected.

INSTALL_DIR="$HOME/.local/bin"
mkdir -p "$INSTALL_DIR"

CMD_NAME="$1"
CMD_CONTENT="$2"

if [ -z "$CMD_NAME" ] || [ -z "$CMD_CONTENT" ]; then
    echo "Usage: script-installer <command_name> <script_content>"
    exit 1
fi

DEST_FILE="$INSTALL_DIR/$CMD_NAME"

# Check for Collision
if [ -e "$DEST_FILE" ]; then
    echo "Warning: '$CMD_NAME' already exists in $INSTALL_DIR"
    printf "Overwrite? (y/N): "
    read CONFIRM
    case "$CONFIRM" in
        [Yy]*) ;; 
        *) echo "Aborted."; exit 0 ;;
    esac
fi

# -- Feature: Command Name Substitution --
CMD_CONTENT=$(printf '%s\n' "$CMD_CONTENT" | sed "s/<command>/$CMD_NAME/g")

# -- Feature: Shebang Handling --
case "$CMD_CONTENT" in
    \#!*) ;; 
    *) CMD_CONTENT="#!/bin/sh
$CMD_CONTENT" ;;
esac

# Write and allow executing
printf "%s\n" "$CMD_CONTENT" > "$DEST_FILE"
chmod +x "$DEST_FILE"
echo "✅ Installed '$CMD_NAME' to $DEST_FILE"

# -- Feature: Usage Extraction --
awk '
/^#!/ { next }
/^#/ {
    if (block == "") { block = $0 } else { block = block "\n" $0 }
    if (tolower($0) ~ /^#[[:space:]]*usage/) { is_usage = 1 }
    next
}
{
    if (is_usage && block != "") { print block }
    block = ""
    is_usage = 0
}
END {
    if (is_usage && block != "") { print block }
}' "$DEST_FILE"
EOF

  # 3. Make the tool executable
  chmod +x "$TARGET_FILE"
  echo "✅ Updated 'script-installer' at $TARGET_FILE"

  # 4. PATH Logic (POSIX version)
  # We use 'case' string matching instead of bash [[ == *foo* ]]
  case ":$PATH:" in
    *":$TARGET_DIR:"*) 
        echo "✅ $TARGET_DIR is already in your PATH."
        ;;
    *)
        echo "$TARGET_DIR is not in your PATH."
        printf "Add it to your config now? (y/N): "
        read ADD_PATH
        
        # We use 'case' instead of bash regex [[ =~ ]]
        case "$ADD_PATH" in
            [Yy]*)
                # Strip path from SHELL to get just the name (e.g. /bin/zsh -> zsh)
                SHELL_NAME="${SHELL##*/}"
                RC_FILE=""
                
                case "$SHELL_NAME" in
                    bash) RC_FILE="$HOME/.bashrc" ;;
                    zsh)  RC_FILE="$HOME/.zshrc" ;;
                    *)    RC_FILE="$HOME/.profile" ;; 
                esac

                if [ "$SHELL_NAME" = "fish" ]; then
                    fish -c "set -Ua fish_user_paths $TARGET_DIR"
                    echo "✅ Added to Fish user paths."
                else
                    echo "" >> "$RC_FILE"
                    echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$RC_FILE"
                    echo "✅ Added export line to $RC_FILE"

                    # Attempt to reload config
                    if [ -f "$RC_FILE" ]; then
                        echo "🔄 Sourcing $RC_FILE..."
                        # We use . instead of source for POSIX compliance
                        # We silence errors in case the RC file has shell-specific syntax
                        . "$RC_FILE" 2>/dev/null && echo "✅ PATH updated in current session." || echo "⚠️  Could not auto-reload (shell mismatch). Please restart your terminal."
                    fi
                fi
                ;;
            *)
                echo "Skipping PATH update."
                ;;
        esac
        ;;
  esac

  # 5. Cleanup Variables
  unset TARGET_DIR TARGET_FILE SHELL_NAME RC_FILE ADD_PATH CONFIRM
};
```

## 🔪 Port Assassin
The **Port Assassin** is the cure for "Address already in use" errors. Whether a program crashed without cleaning up, or you have dozens of script instances running and need to stop just one specific worker, this tool finds the process holding the port and kills it instantly.

### Usage
```bash
killport 5550-5600 5603 5607
```
_Note: Accepts space-separated lists, comma-separated values, and ranges._

Features
* **Smart Targeting**: Pass in single ports, ranges, or a mix of both. It handles the formatting for you.
* **Safety Checks**: Includes strict input sanitization to ensure only valid digits and ranges are passed to the kill command, preventing accidental system instability.
* **Process Resolution**: Doesn't just kill by name (which stops everything); it kills by port, ensuring you only stop exactly what you intend to.

### Installation
Run the following command after setting up the script-installer:
```bash
script-installer killport '# Usage: <command> 5555 8080-8090
# Kills processes on the specified ports.
# Supports ranges (55-60) and lists (55, 56).

# Check for missing arguments
if [ "$#" -eq 0 ]; then
echo "Usage: $(basename "$0") <ports>"
echo "Example: $(basename "$0") 5555 8080-8090"
exit 1
fi
# Normalize Input
PORTS=$(echo "$@" | tr " " "," | tr -s "," | sed "s/^,//;s/,$//")
# Validate Input
if echo "$PORTS" | grep -qv "^[0-9,-]*$"; then
echo "Error: Invalid characters detected."
echo "Allowed: digits (0-9), hyphens (-), and commas (,)."
echo "Example: $(basename "$0") 5555 8080-8090"
exit 1
fi
lsof -t -i :"$PORTS" | xargs -r kill'
```

## ❤️ Support
If you found this tool useful, consider giving the repo a star!
If you want to support my "Project a Week" challenge or buy me a coffee, you can do so here: [Become a Patron](https://www.patreon.com/cw/ccgreen)


