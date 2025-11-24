# Development Environment Setup

This guide will walk you through setting up your development environment on **Linux** to build clients for MediaManager Core or contribute to the project itself.

---

## Overview

To develop for MediaManager, you'll need:

1. **Protocol Buffers compiler** - To generate code from `.proto` files
2. **Programming language tools** - Depending on your client choice
3. **Git** - To clone the repository
4. **IDE/Editor** - Your preferred development environment

---

## Operating System Requirements

### Supported Linux Distributions

MediaManager development is tested and supported on:

- **Ubuntu/Debian** (20.04 LTS or newer)
- **Fedora/RHEL** (35 or newer)
- **Arch Linux**
- **Other modern Linux distributions** with kernel 5.0+

### Verify Unix Socket Support

Most Linux distributions have Unix socket support built-in:
```bash
# Check kernel support
cat /proc/net/unix | head -5

# Verify /tmp is accessible
ls -la /tmp/ && echo "✓ /tmp accessible"
```

---

## Protocol Buffers Installation

Protocol Buffers (protobuf) is **essential** for MediaManager development.

### Debian/Ubuntu
```bash
# Update package list
sudo apt update

# Install Protocol Buffers compiler
sudo apt install -y protobuf-compiler

# Verify installation
protoc --version
# Should output: libprotoc 3.x.x or higher
```

### Fedora/RHEL
```bash
# Install Protocol Buffers compiler
sudo dnf install -y protobuf-compiler

# Verify installation
protoc --version
```

### Arch Linux
```bash
# Install Protocol Buffers
sudo pacman -S protobuf

# Verify installation
protoc --version
```

### Manual Installation (Any Distribution)

If your package manager doesn't have a recent version:
```bash
# Download latest release (check for newer versions)
PROTOC_VERSION="24.4"
wget https://github.com/protocolbuffers/protobuf/releases/download/v${PROTOC_VERSION}/protoc-${PROTOC_VERSION}-linux-x86_64.zip

# Extract to local bin directory
unzip protoc-${PROTOC_VERSION}-linux-x86_64.zip -d $HOME/.local

# Add to PATH (add this line to ~/.bashrc)
export PATH="$HOME/.local/bin:$PATH"

# Apply changes
source ~/.bashrc

# Verify installation
protoc --version
```

---

## Language-Specific Setup

Choose the language you'll use to build your MediaManager client:

### Java (17+)

Required if you're working on the Core or building a Java client.

#### Installation

**Debian/Ubuntu:**
```bash
# Install OpenJDK 17
sudo apt install -y openjdk-17-jdk openjdk-17-jre

# Verify installation
java -version
# Should output: openjdk version "17.x.x"

javac -version
# Should output: javac 17.x.x
```

**Fedora/RHEL:**
```bash
# Install OpenJDK 17
sudo dnf install -y java-17-openjdk java-17-openjdk-devel

# Verify installation
java -version
javac -version
```

**Arch Linux:**
```bash
# Install OpenJDK
sudo pacman -S jdk17-openjdk

# Verify installation
java -version
javac -version
```

#### Maven Setup

**Debian/Ubuntu:**
```bash
sudo apt install -y maven

# Verify installation
mvn -version
# Should output: Apache Maven 3.6.x or higher
```

**Fedora/RHEL:**
```bash
sudo dnf install -y maven

mvn -version
```

**Arch Linux:**
```bash
sudo pacman -S maven

mvn -version
```

#### Java Environment Variables

Add these to your `~/.bashrc`:
```bash
# Find Java installation path
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which java))))
export PATH=$JAVA_HOME/bin:$PATH
```

Apply changes:
```bash
source ~/.bashrc

# Verify
echo $JAVA_HOME
```

---

### Python (3.8+)

For building Python clients.

#### Installation

**Debian/Ubuntu:**
```bash
# Install Python and tools
sudo apt install -y python3 python3-pip python3-venv

# Verify installation
python3 --version
pip3 --version
```

**Fedora/RHEL:**
```bash
sudo dnf install -y python3 python3-pip

python3 --version
pip3 --version
```

**Arch Linux:**
```bash
sudo pacman -S python python-pip

python --version
pip --version
```

#### Install Protocol Buffers Python Library
```bash
# Create virtual environment (recommended)
python3 -m venv ~/mediamanager-env
source ~/mediamanager-env/bin/activate

# Install protobuf and grpcio
pip install protobuf grpcio grpcio-tools

# Verify installation
python -c "import google.protobuf; print(google.protobuf.__version__)"
```

Add to `~/.bashrc` for easy activation:
```bash
alias mediamanager-env='source ~/mediamanager-env/bin/activate'
```

---

### TypeScript/Node.js

For building web or Electron-based clients.

#### Installation

**Debian/Ubuntu:**
```bash
# Install Node.js 18+ via NodeSource repository
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version
npm --version
```

**Fedora/RHEL:**
```bash
# Install Node.js
sudo dnf install -y nodejs npm

# Verify installation
node --version
npm --version
```

**Arch Linux:**
```bash
sudo pacman -S nodejs npm

node --version
npm --version
```

#### Install Protocol Buffers Libraries
```bash
# Create project directory
mkdir mediamanager-client
cd mediamanager-client

# Initialize npm project
npm init -y

# Install dependencies
npm install google-protobuf @grpc/grpc-js @grpc/proto-loader

# For TypeScript development
npm install -D typescript @types/google-protobuf @types/node

# Verify installation
npm list google-protobuf
```

---

### Go

For building high-performance clients.

#### Installation

**Debian/Ubuntu:**
```bash
# Download Go (check for latest version at https://go.dev/dl/)
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz

# Remove old installation and install new
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz

# Add to PATH (add to ~/.bashrc)
export PATH=$PATH:/usr/local/go/bin
export PATH=$PATH:$(go env GOPATH)/bin

# Apply changes
source ~/.bashrc

# Verify installation
go version
```

**Fedora/RHEL:**
```bash
sudo dnf install -y golang

go version
```

**Arch Linux:**
```bash
sudo pacman -S go

go version
```

#### Install Protocol Buffers Libraries
```bash
# Install protoc-gen-go plugins
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Verify installation
which protoc-gen-go
which protoc-gen-go-grpc
```

---

### Rust

For building ultra-fast native clients.

#### Installation
```bash
# Install Rust via rustup (works on all distributions)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Follow the prompts (usually option 1 for default installation)

# Load Rust environment
source $HOME/.cargo/env

# Verify installation
rustc --version
cargo --version
```

#### Install Protocol Buffers Libraries
```bash
# Create new Rust project
cargo new mediamanager-client
cd mediamanager-client

# Add dependencies to Cargo.toml
cargo add prost
cargo add prost-types
cargo add tokio --features full

# Verify
cargo build
```

---

## Git Setup

### Installation

**Debian/Ubuntu:**
```bash
sudo apt install -y git
```

**Fedora/RHEL:**
```bash
sudo dnf install -y git
```

**Arch Linux:**
```bash
sudo pacman -S git
```

### Configuration
```bash
# Set your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Verify configuration
git config --list
```

### Clone MediaManager Core
```bash
# Clone the repository
git clone https://git.gustavomiranda.xyz/GHMiranda/MediaManager-core

# Navigate to project directory
cd MediaManager-core

# Check repository status
git status

# List available branches
git branch -a
```

---

## IDE/Editor Setup

Choose your preferred development environment:

### IntelliJ IDEA (Recommended for Java)

#### Installation

**Debian/Ubuntu:**
```bash
# Install via snap (easiest method)
sudo snap install intellij-idea-community --classic
```

**Arch Linux:**
```bash
# Install from AUR
yay -S intellij-idea-community-edition
# or
sudo pacman -S intellij-idea-community-edition
```

**Manual Installation (Any Distribution):**
```bash
# Download from JetBrains website
wget https://download.jetbrains.com/idea/ideaIC-2023.3.tar.gz

# Extract
sudo tar -xzf ideaIC-2023.3.tar.gz -C /opt/

# Create symbolic link
sudo ln -s /opt/idea-IC-*/bin/idea.sh /usr/local/bin/idea

# Run
idea
```

#### Configuration

1. **Open IntelliJ IDEA**
2. **Import MediaManager-core**:
   - `File` → `Open`
   - Navigate to `MediaManager-core` directory
   - Click `OK` - IntelliJ will detect it as a Maven project

3. **Install Plugins**:
   - `File` → `Settings` → `Plugins`
   - Search and install:
     - **Protocol Buffers** (for `.proto` syntax highlighting)
     - **Maven Helper** (optional, for dependency management)

4. **Configure Maven**:
   - `File` → `Settings` → `Build, Execution, Deployment` → `Build Tools` → `Maven`
   - Verify Maven home directory: `/usr/share/maven`
   - Set JDK to version 17

5. **Generate Protocol Buffer Sources**:
   - Open Maven panel (View → Tool Windows → Maven)
   - Navigate to: `MediaManager-core` → `Lifecycle` → `generate-sources`
   - Double-click to run
   - Generated Java classes will appear in `target/generated-sources/protobuf/`

---

### Visual Studio Code

#### Installation

**Debian/Ubuntu:**
```bash
# Download .deb package
wget -O code.deb 'https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64'

# Install
sudo dpkg -i code.deb
sudo apt install -f  # Fix dependencies if needed
```

**Fedora/RHEL:**
```bash
# Import Microsoft GPG key
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# Add repository
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'

# Install
sudo dnf check-update
sudo dnf install code
```

**Arch Linux:**
```bash
yay -S visual-studio-code-bin
```

#### Essential Extensions

Install these extensions for MediaManager development:
```bash
# Protocol Buffers syntax highlighting
code --install-extension zxh404.vscode-proto3

# Java development (if building Java clients or working on Core)
code --install-extension vscjava.vscode-java-pack

# Python development
code --install-extension ms-python.python

# JavaScript/TypeScript
code --install-extension dbaeumer.vscode-eslint

# Rust development
code --install-extension rust-lang.rust-analyzer

# Go development
code --install-extension golang.go
```

#### Workspace Settings

Create `.vscode/settings.json` in MediaManager-core directory:
```json
{
  "protoc": {
    "path": "/usr/bin/protoc",
    "compile_on_save": false
  },
  "java.configuration.updateBuildConfiguration": "automatic",
  "java.jdt.ls.java.home": "/usr/lib/jvm/java-17-openjdk-amd64",
  "maven.executable.path": "/usr/bin/mvn",
  "files.exclude": {
    "**/target": true,
    "**/.idea": true
  }
}
```

---

### Neovim (For Terminal Enthusiasts)

#### Installation

**Debian/Ubuntu:**
```bash
sudo apt install -y neovim
```

**Fedora/RHEL:**
```bash
sudo dnf install -y neovim
```

**Arch Linux:**
```bash
sudo pacman -S neovim
```

#### Plugin Manager Setup
```bash
# Install vim-plug
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'

# Create config directory
mkdir -p ~/.config/nvim
```

#### Basic Configuration

Create `~/.config/nvim/init.vim`:
```vim
call plug#begin('~/.local/share/nvim/plugged')

" LSP support for multiple languages
Plug 'neoclide/coc.nvim', {'branch': 'release'}

" Protocol Buffers syntax
Plug 'uber/prototool', { 'rtp': 'vim/prototool' }

" File tree
Plug 'preservim/nerdtree'

" Fuzzy finder
Plug 'junegunn/fzf', { 'do': { -> fzf#install() } }
Plug 'junegunn/fzf.vim'

call plug#end()

" Basic settings
set number
set relativenumber
set tabstop=4
set shiftwidth=4
set expandtab
syntax on

" Keybindings
nnoremap <C-n> :NERDTreeToggle<CR>
nnoremap <C-p> :Files<CR>
```

Install plugins:
```bash
nvim +PlugInstall +qall
```

---

## Database Setup (Optional)

If you're running MediaManager Core locally for testing:

### SQLite (Embedded, easiest)

**Debian/Ubuntu:**
```bash
sudo apt install -y sqlite3
```

**Fedora/RHEL:**
```bash
sudo dnf install -y sqlite
```

**Arch Linux:**
```bash
sudo pacman -S sqlite
```

Verify installation:
```bash
sqlite3 --version
```

No additional configuration needed - MediaManager Core uses SQLite by default.

---

### PostgreSQL (Production-ready)

**Debian/Ubuntu:**
```bash
# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Start and enable service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Check status
sudo systemctl status postgresql
```

**Fedora/RHEL:**
```bash
sudo dnf install -y postgresql-server postgresql-contrib

# Initialize database
sudo postgresql-setup --initdb

# Start and enable
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

**Arch Linux:**
```bash
sudo pacman -S postgresql

# Initialize database cluster
sudo -u postgres initdb -D /var/lib/postgres/data

# Start and enable
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

#### Create MediaManager Database
```bash
# Switch to postgres user
sudo -u postgres psql
```

In PostgreSQL shell:
```sql
-- Create database
CREATE DATABASE mediamanager;

-- Create user
CREATE USER mediamanager WITH PASSWORD 'your_secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE mediamanager TO mediamanager;

-- Exit
\q
```

Verify connection:
```bash
psql -U mediamanager -d mediamanager -h localhost
# Enter password when prompted
```

---

## Verify Your Setup

Run this comprehensive checklist:

### System Check
```bash
#!/bin/bash
echo "=== MediaManager Development Environment Check ==="
echo ""

# OS Info
echo "Operating System:"
cat /etc/os-release | grep PRETTY_NAME
echo ""

# Unix Socket Support
echo "Unix Socket Support:"
ls -la /tmp/ > /dev/null 2>&1 && echo "✓ /tmp accessible" || echo "✗ /tmp not accessible"
cat /proc/net/unix | head -1 > /dev/null 2>&1 && echo "✓ Unix sockets supported" || echo "✗ Unix sockets not supported"
echo ""

# Protocol Buffers
echo "Protocol Buffers:"
if command -v protoc &> /dev/null; then
    echo "✓ protoc installed: $(protoc --version)"
else
    echo "✗ protoc not found"
fi
echo ""

# Git
echo "Git:"
if command -v git &> /dev/null; then
    echo "✓ Git installed: $(git --version)"
else
    echo "✗ Git not found"
fi
echo ""
```

Save this as `check-setup.sh`, make it executable, and run:
```bash
chmod +x check-setup.sh
./check-setup.sh
```

### Language-Specific Checks

**Java:**
```bash
java -version 2>&1 | head -1
mvn -version | head -1
echo $JAVA_HOME
```

**Python:**
```bash
python3 --version
pip3 list | grep protobuf
```

**Node.js:**
```bash
node --version
npm --version
npm list -g google-protobuf 2>/dev/null || echo "Not installed globally"
```

**Go:**
```bash
go version
which protoc-gen-go
```

**Rust:**
```bash
rustc --version
cargo --version
```

---

## Project Structure Overview

After cloning MediaManager-core:
```
MediaManager-core/
├── pom.xml                          # Maven configuration
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/mediamanager/
│   │   │       ├── service/
│   │   │       │   ├── ipc/         # Unix Socket server
│   │   │       │   │   └── IPCManager.java
│   │   │       │   └── delegate/    # Request handlers
│   │   │       │       └── DelegateActionManager.java
│   │   │       └── protocol/        # Generated protobuf classes (after build)
│   │   ├── proto/                   # Protocol Buffer definitions
│   │   │   ├── transport.proto      # Request/Response messages
│   │   │   └── test.proto           # Test commands
│   │   └── resources/
│   │       └── config.properties    # Configuration
│   └── test/
│       └── java/                    # Unit tests
├── target/                          # Build output (created after mvn compile)
│   ├── classes/
│   └── generated-sources/
│       └── protobuf/
│           └── java/                # Compiled protobuf Java classes
└── README.md
```

---

## Building MediaManager Core

Once your environment is set up, test the build:
```bash
# Navigate to project
cd MediaManager-core

# Clean previous builds
mvn clean

# Compile and generate protobuf sources
mvn generate-sources

# Check generated files
ls -la target/generated-sources/protobuf/java/com/mediamanager/protocol/

# Full build
mvn package

# Run tests
mvn test
```

If successful, you should see:
```
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```

---

## Next Steps

Now that your Linux environment is configured:

1. **Learn about Protocol Buffers** → [[Protocol Buffers Guide]]
2. **Understand Unix Socket communication** → [[Unix Socket Communication]]
3. **Start building a client** → [[Client Development Guide]]
4. **Explore examples** → [[Code Examples]]

---

## Common Issues & Troubleshooting

### "protoc: command not found"
```bash
# Check if protoc is installed
which protoc

# If not in PATH, add it
export PATH="$HOME/.local/bin:$PATH"

# Make permanent by adding to ~/.bashrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### "JAVA_HOME is not set"
```bash
# Find Java installation
dirname $(dirname $(readlink -f $(which java)))

# Set JAVA_HOME (example for Ubuntu)
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64

# Add to ~/.bashrc
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
source ~/.bashrc

# Verify
echo $JAVA_HOME
```

### Maven build fails with "Package does not exist"
```bash
# Make sure protobuf sources are generated first
mvn clean generate-sources

# Then build
mvn package
```

### "Permission denied" on /tmp/mediamanager.sock
```bash
# Check socket permissions
ls -la /tmp/mediamanager.sock

# Fix if needed (owner only)
chmod 600 /tmp/mediamanager.sock

# Or if you own the socket
sudo chown $USER:$USER /tmp/mediamanager.sock
chmod 600 /tmp/mediamanager.sock
```

### SQLite/PostgreSQL connection errors
```bash
# For SQLite - verify file permissions
ls -la mediamanager.db

# For PostgreSQL - check if service is running
sudo systemctl status postgresql

# Test connection
psql -U mediamanager -d mediamanager -h localhost
```

---

## Additional Resources

- **Protocol Buffers Documentation:** https://protobuf.dev/
- **Java NIO Guide:** https://docs.oracle.com/javase/tutorial/essential/io/
- **Maven Documentation:** https://maven.apache.org/guides/
- **Unix Socket Programming:** `man 7 unix`

---

**Environment ready?** Proceed to [[Protocol Buffers Guide]] to understand MediaManager's message definitions!