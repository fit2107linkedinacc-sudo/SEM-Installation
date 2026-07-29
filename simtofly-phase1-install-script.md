#!/bin/bash
###############################################################################
# SimToFly - Phase 1 (Simulation Mastery) Automated Installer
#
# Source guide: https://github.com/simtofly/simtofly-guide
#                docs/phase-1-simulation (1.2, 1.3, 1.5)
#
# This script automates:
#   1.2  Environment Setup     - system update, dev tools, workspace
#   1.3  ArduPilot SITL        - clone, deps, submodules, first build
#   1.5  Gazebo Harmonic       - Gazebo install + ArduPilot Gazebo plugin
#
# Tested target: Ubuntu 22.04 LTS
#
# Usage:
#   chmod +x simtofly-phase1-install.sh
#   ./simtofly-phase1-install.sh
#
# Safe to re-run: steps check for existing state and skip where possible.
###############################################################################

set -e  # exit on first error

WS="$HOME/simtofly_ws"
LOG_FILE="$HOME/simtofly_phase1_install.log"

# ---------- helpers ---------------------------------------------------------
log()  { echo -e "\n\033[1;34m==>\033[0m $1" | tee -a "$LOG_FILE"; }
ok()   { echo -e "\033[1;32m✅ $1\033[0m" | tee -a "$LOG_FILE"; }
warn() { echo -e "\033[1;33m⚠️  $1\033[0m" | tee -a "$LOG_FILE"; }
err()  { echo -e "\033[1;31m❌ $1\033[0m" | tee -a "$LOG_FILE"; }

trap 'err "Script failed at line $LINENO. Check $LOG_FILE for details."' ERR

echo "SimToFly Phase 1 installation started: $(date)" > "$LOG_FILE"

# ---------- 0. Ubuntu version check -----------------------------------------
log "Checking Ubuntu version..."
if command -v lsb_release &>/dev/null; then
    UB_VER=$(lsb_release -rs)
    if [[ "$UB_VER" != "22.04" ]]; then
        warn "This guide is verified on Ubuntu 22.04 LTS. You're running $UB_VER. Continuing anyway, but expect possible issues."
    else
        ok "Ubuntu 22.04 detected"
    fi
else
    warn "lsb_release not found yet — will be installed shortly."
fi

# ---------- 1.2 Environment Setup -------------------------------------------
log "[1.2] Updating system packages (this can take several minutes)..."
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y
sudo apt autoclean

log "[1.2] Installing essential development tools..."
sudo apt install -y \
    build-essential \
    git \
    python3 \
    python3-pip \
    python3-dev \
    curl \
    wget \
    nano \
    net-tools \
    tree \
    lsb-release \
    gnupg

ok "Development tools installed"
gcc --version | head -n1
git --version
python3 --version
pip3 --version
tree --version | head -n1

log "[1.2] Creating SimToFly workspace at $WS ..."
mkdir -p "$WS"/{logs,missions,scripts,ros2_ws}

if ! grep -q "SIMTOFLY_WS" ~/.bashrc; then
    echo 'export SIMTOFLY_WS=~/simtofly_ws' >> ~/.bashrc
fi
export SIMTOFLY_WS="$WS"
ok "Workspace structure ready: $WS"
tree -L 1 "$WS" || true

# ---------- 1.3 ArduPilot SITL -----------------------------------------------
log "[1.3] Cloning ArduPilot repository..."
if [ ! -d "$WS/ardupilot" ]; then
    cd "$WS"
    git clone https://github.com/ArduPilot/ardupilot.git
else
    warn "ArduPilot already cloned at $WS/ardupilot — skipping clone."
fi

cd "$WS/ardupilot"

log "[1.3] Installing ArduPilot prerequisites (installs compilers/libraries, ~2GB download)..."
Tools/environment_install/install-prereqs-ubuntu.sh -y

log "[1.3] Reloading profile to pick up new PATH/env vars..."
# shellcheck disable=SC1090
source ~/.profile || true

log "[1.3] Checking out stable Copter-4.5.7..."
git config --global url."https://".insteadOf git:// || true
git checkout Copter-4.5.7

log "[1.3] Initializing submodules (this can take 5-10 minutes)..."
git submodule update --init --recursive

log "[1.3] Ensuring sim_vehicle.py is on PATH..."
if ! grep -q "ardupilot/Tools/autotest" ~/.bashrc; then
    echo "export PATH=\$PATH:$WS/ardupilot/Tools/autotest" >> ~/.bashrc
fi
export PATH="$PATH:$WS/ardupilot/Tools/autotest"

log "[1.3] Ensuring pymavlink/MAVProxy Python deps are present..."
pip3 install --user pymavlink MAVProxy

ok "ArduPilot SITL source, dependencies, and submodules are ready."
warn "First SITL build/launch (sim_vehicle.py -w) is intentionally left for you to run manually,"
warn "since it opens an interactive console you need to watch for 'Received 500 parameters' and then Ctrl+C."
echo "  To do it now, run:"
echo "    cd $WS/ardupilot/ArduCopter && sim_vehicle.py -w"

# ---------- 1.5 Gazebo Harmonic -----------------------------------------------
log "[1.5] Adding Gazebo Harmonic APT repository..."
sudo apt update
sudo apt install -y curl lsb-release gnupg

sudo curl -fsSL https://packages.osrfoundation.org/gazebo.gpg \
    --output /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] https://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
    | sudo tee /etc/apt/sources.list.d/gazebo-stable.list > /dev/null

sudo apt update

log "[1.5] Installing Gazebo Harmonic (gz-harmonic)..."
sudo apt install -y gz-harmonic

ok "Gazebo installed:"
gz sim --version || warn "Could not verify gz sim version — check installation."

log "[1.5] Installing ArduPilot Gazebo plugin build dependencies..."
sudo apt install -y libgz-sim8-dev rapidjson-dev
sudo apt install -y libopencv-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev \
    gstreamer1.0-plugins-bad gstreamer1.0-libav gstreamer1.0-gl
sudo apt install -y libgl1-mesa-glx libglu1-mesa

log "[1.5] Cloning ArduPilot Gazebo plugin..."
if [ ! -d "$WS/ardupilot_gazebo" ]; then
    cd "$WS"
    git clone https://github.com/ArduPilot/ardupilot_gazebo.git
else
    warn "ardupilot_gazebo already cloned — skipping clone."
fi

log "[1.5] Building ArduPilot Gazebo plugin..."
cd "$WS/ardupilot_gazebo"
mkdir -p build && cd build
cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo
make -j"$(nproc)"

ok "ArduPilot Gazebo plugin built successfully."

log "[1.5] Configuring Gazebo environment variables..."
if ! grep -q "GZ_SIM_SYSTEM_PLUGIN_PATH" ~/.bashrc; then
    echo "export GZ_SIM_SYSTEM_PLUGIN_PATH=$WS/ardupilot_gazebo/build:\${GZ_SIM_SYSTEM_PLUGIN_PATH}" >> ~/.bashrc
fi
if ! grep -q "GZ_SIM_RESOURCE_PATH" ~/.bashrc; then
    echo "export GZ_SIM_RESOURCE_PATH=$WS/ardupilot_gazebo/models:$WS/ardupilot_gazebo/worlds:\${GZ_SIM_RESOURCE_PATH}" >> ~/.bashrc
fi

ok "Environment variables added to ~/.bashrc"

# ---------- Verification script (from 1.2) ----------------------------------
log "Writing environment verification script to $WS/scripts/check_environment.sh ..."
cat > "$WS/scripts/check_environment.sh" << 'EOF'
#!/bin/bash

echo "=========================================="
echo "SimToFly Environment Check"
echo "=========================================="
echo ""

echo "Checking Ubuntu version..."
if lsb_release -d | grep -q "22.04"; then
    echo "✅ Ubuntu 22.04 detected"
else
    echo "❌ Wrong Ubuntu version. Need 22.04"
    lsb_release -d
fi
echo ""

echo "Checking disk space..."
AVAILABLE=$(df -BG / | tail -1 | awk '{print $4}' | sed 's/G//')
if [ "$AVAILABLE" -gt 30 ]; then
    echo "✅ Sufficient disk space: ${AVAILABLE}GB available"
else
    echo "⚠️ Low disk space: ${AVAILABLE}GB available (need 30GB+)"
fi
echo ""

echo "Checking RAM..."
TOTAL_RAM=$(free -g | awk '/^Mem:/{print $2}')
if [ "$TOTAL_RAM" -ge 7 ]; then
    echo "✅ Sufficient RAM: ${TOTAL_RAM}GB"
else
    echo "⚠️ Low RAM: ${TOTAL_RAM}GB (recommended 8GB+)"
fi
echo ""

echo "Checking essential tools..."
TOOLS=("gcc" "g++" "make" "git" "python3" "pip3" "tree" "gz" "cmake")
for tool in "${TOOLS[@]}"; do
    if command -v $tool &> /dev/null; then
        echo "✅ $tool installed"
    else
        echo "❌ $tool NOT installed"
    fi
done
echo ""

echo "Checking workspace structure..."
DIRS=("logs" "missions" "scripts" "ros2_ws" "ardupilot" "ardupilot_gazebo")
for dir in "${DIRS[@]}"; do
    if [ -d "$HOME/simtofly_ws/$dir" ]; then
        echo "✅ $dir directory exists"
    else
        echo "❌ $dir directory missing"
    fi
done
echo ""

echo "Checking internet connection..."
if ping -c 1 google.com &> /dev/null; then
    echo "✅ Internet connection working"
else
    echo "❌ No internet connection"
fi
echo ""

echo "=========================================="
echo "Environment check complete!"
echo "=========================================="
EOF

chmod +x "$WS/scripts/check_environment.sh"
ok "Verification script created and made executable."

# ---------- Final summary ----------------------------------------------------
log "Installation complete!"
echo ""
echo "-----------------------------------------------------------------"
echo " NEXT STEPS (manual, interactive - one-time only):"
echo ""
echo " 1) Reload your shell so new env vars/PATH take effect:"
echo "      source ~/.bashrc"
echo ""
echo " 2) First SITL build + parameter wipe (watch for 'Received 500 parameters', then Ctrl+C):"
echo "      cd $WS/ardupilot/ArduCopter"
echo "      sim_vehicle.py -w"
echo ""
echo " 3) Launch SITL normally:"
echo "      sim_vehicle.py -v ArduCopter"
echo ""
echo " 4) Test Gazebo on its own:"
echo "      gz sim -v4 shapes.sdf"
echo ""
echo " 5) Launch SITL + Gazebo together (2 terminals):"
echo "      Terminal 1: gz sim -v4 -r iris_runway.sdf"
echo "      Terminal 2: cd $WS/ardupilot/ArduCopter && sim_vehicle.py -v ArduCopter -f gazebo-iris --model JSON --map --console"
echo ""
echo " 6) Run the environment check any time:"
echo "      $WS/scripts/check_environment.sh"
echo "-----------------------------------------------------------------"
echo ""
ok "Full log saved to: $LOG_FILE"
