# Simple Scripts

**Simple Scripts** is a collection of scripts that will expand over time, these are scripts that I use regularly and each solve a problem I have had.

Currently contains the following scripts:
* [Script Installer](#-script-installer) - A script to install the other scripts
* [Port Assassin](#-port-assassin) - For killing processes using given ports

## 📦 Script Installer
The **Script Installer** is the heart of these scripts, it provides a simple and easy way to install scripts across terminals with explicit support for Bash, Zsh, and Fish, or defaulting to strict POSIX.

### Usage
```bash
script-installer <command name> <script content>
```

### Features
* Automatically prepends #!/bin/sh to script if no shebang is found.
* Replaces <command> with the command name so users can set the command they want when installing and comments and usage instructions in the script are updated to match.
* Prints out comment blocks with # Usage in them so usage commands and other info can be printed out when installing.

### Installation
Run the following command in a Bash, Zsh, Fish, or POSIX terminal. This is long so the others can be short.
```bash
{
  # 1. Setup Directory
  TARGET_DIR="$HOME/.local/bin"
  mkdir -p "$TARGET_DIR"
  TARGET_FILE="$TARGET_DIR/script-installer"

  # 2. Write the smarter tool content
  cat << 'EOF' > "$TARGET_FILE"
#!/bin/bash
# script-installer: Installs script content to ~/.local/bin
# Features:
# 1. Replaces <command> with the installed filename.
# 2. Auto-prepends #!/bin/sh if missing.
# 3. Prints # Usage info if detected.

INSTALL_DIR="$HOME/.local/bin"
mkdir -p "$INSTALL_DIR"

CMD_NAME="$1"
CMD_CONTENT="$2"

if [[ -z "$CMD_NAME" ]] || [[ -z "$CMD_CONTENT" ]]; then
    echo "Usage: script-installer <command_name> <script_content>"
    exit 1
fi

DEST_FILE="$INSTALL_DIR/$CMD_NAME"

# Check for Collision
if [[ -e "$DEST_FILE" ]]; then
    echo "Warning: '$CMD_NAME' already exists in $INSTALL_DIR"
    read -p "Overwrite? (y/N): " -r CONFIRM
    echo
    if [[ ! $CONFIRM =~ ^[Yy]$ ]]; then
        echo "Aborted."
        exit 0
    fi
fi

# -- Feature: Command Name Substitution --
# Replace literal "<command>" with the actual command name
CMD_CONTENT="${CMD_CONTENT//<command>/$CMD_NAME}"

# -- Feature: Shebang Handling --
if [[ "$CMD_CONTENT" != \#!* ]]; then
    CMD_CONTENT="#!/bin/sh"$'\n'"$CMD_CONTENT"
fi

# Write and allow executing
printf "%s\n" "$CMD_CONTENT" > "$DEST_FILE"
chmod +x "$DEST_FILE"
echo "✅ Installed '$CMD_NAME' to $DEST_FILE"

# -- Feature: Usage Extraction --
# Scans for a comment block containing "Usage" (case-insensitive)
# Skips the shebang line (#!/...)
awk '
/^#!/ { next }  # Ignore shebang lines
/^#/ {
    # Add line to the current block buffer
    if (block == "") { block = $0 } else { block = block "\n" $0 }
    
    # Check if this line looks like a usage header
    if (tolower($0) ~ /^#[[:space:]]*usage/) { is_usage = 1 }
    next
}
{
    # We hit a non-comment line. If the previous block was a usage block, print it.
    if (is_usage && block != "") { print block }
    block = ""
    is_usage = 0
}
END {
    # Flush if the file ends with a usage block
    if (is_usage && block != "") { print block }
}
' "$DEST_FILE"
EOF

  # 3. Make the tool executable
  chmod +x "$TARGET_FILE"
  echo "✅ Updated 'script-installer' at $TARGET_FILE"

  # 4. PATH Logic (unchanged)
  if [[ ":$PATH:" != *":$TARGET_DIR:"* ]]; then
      echo "$TARGET_DIR is not in your PATH."
      read -p "Add it to your config now? (y/N): " -r ADD_PATH
      if [[ $ADD_PATH =~ ^[Yy]$ ]]; then
          SHELL_NAME=$(basename "$SHELL")
          RC_FILE=""
          case "$SHELL_NAME" in
              bash) RC_FILE="$HOME/.bashrc" ;;
              zsh)  RC_FILE="$HOME/.zshrc" ;;
              *)    RC_FILE="$HOME/.profile" ;; 
          esac

          if [[ "$SHELL_NAME" = "fish" ]]; then
              fish -c "set -Ua fish_user_paths $TARGET_DIR"
              echo "✅ Added to Fish user paths."
          else
              echo "" >> "$RC_FILE"
              echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$RC_FILE"
              echo "✅ Added export line to $RC_FILE"

	      # Reload config file
              if [ -f "$RC_FILE" ]; then
                  echo "🔄 Sourcing $RC_FILE..."
                  . "$RC_FILE"
                  echo "✅ PATH updated in current session."
              fi
          fi
      else
          echo "Skipping PATH update."
      fi
  else
      echo "✅ $TARGET_DIR is already in your PATH."
  fi
  # 5. Cleanup Variables (Keep the terminal clean)
  unset TARGET_DIR TARGET_FILE SHELL_NAME RC_FILE ADD_PATH
};
```

## 🔪 Port Assassin
The **Port Assassin** is there for those times when a program doesn't give up it's ports when they should, or when you have 30 instances of a script running and you only want the one using a particular port to be stopped.

### Usage
```bash
killport 5550-5600 5603 5607
```
Ports can be passed in individually or as ranges, any number can be passed in, and the processes using these ports will be killed.

Features
* Accepts individual ports (e.g., 8080) or ranges (e.g., 3000-3005).
* Identifies the specific PID tied to the port and terminates it.
* Handles multiple arguments in a single command.

### Installation
Run the following command after installing the script-installer, or download the script file from this repo.
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


