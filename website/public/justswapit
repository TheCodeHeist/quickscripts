#!/bin/bash
# ================================================
# Just.Swap.It - Universal Swap Setup Script
# ================================================

set -euo pipefail

SCRIPT_NAME=$(basename "$0")
RAM_GB=$(free --giga 2>/dev/null | awk '/^Mem:/ {print $2}' || echo "2")
SWAP_SIZE_DEFAULT="4G"

# ===================== BANNER =====================
show_banner() {
    clear
    echo -e "\033[1;32m"
    cat << "GRAFFITI"
    
    =[ TheCodeHeist's QuickScripts ]==============================================================

    ████████╗██╗  ██╗███████╗ ██████╗ ██████╗ ██████╗ ███████╗██╗  ██╗███████╗██╗███████╗████████╗
    ╚══██╔══╝██║  ██║██╔════╝██╔════╝██╔═══██╗██╔══██╗██╔════╝██║  ██║██╔════╝██║██╔════╝╚══██╔══╝
       ██║   ███████║█████╗  ██║     ██║   ██║██║  ██║█████╗  ███████║█████╗  ██║███████╗   ██║   
       ██║   ██╔══██║██╔══╝  ██║     ██║   ██║██║  ██║██╔══╝  ██╔══██║██╔══╝  ██║╚════██║   ██║   
       ██║   ██║  ██║███████╗╚██████╗╚██████╔╝██████╔╝███████╗██║  ██║███████╗██║███████║   ██║   
       ╚═╝   ╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚═╝  ╚═╝╚══════╝╚═╝╚══════╝   ╚═╝   
                                                                                              
             Just.Swap.It - Universal Swap Setup Utility Script   |   Just Swap It Bro
GRAFFITI
    echo -e "\033[0m"
    echo -e "\033[1;33mUniversal Swap Manager for Low-RAM / Potato Machines\033[0m"
    echo -e "Detected RAM     : \033[1;36m${RAM_GB} GB\033[0m"
    echo -e "Recommended swap : \033[1;36m${RAM_GB}–${RAM_GB}*2 GB total (zram + swapfile)\033[0m\n"
}

# ===================== HELPERS =====================
detect_package_manager() {
    if command -v apt-get >/dev/null; then echo "apt"
    elif command -v dnf >/dev/null; then echo "dnf"
    elif command -v pacman >/dev/null; then echo "pacman"
    else echo "unknown"; fi
}

install_package() {
    local pkg="$1"
    local pm=$(detect_package_manager)
    case "$pm" in
        apt)    sudo apt update && sudo apt install -y "$pkg" ;;
        dnf)    sudo dnf install -y "$pkg" ;;
        pacman) sudo pacman -S --noconfirm "$pkg" ;;
        *)      echo "❌ Could not install $pkg automatically. Please install manually." && return 1 ;;
    esac
}

detect_existing_swap() {
    if swapon --show | grep -q . || grep -qE 'swap|Swap' /etc/fstab 2>/dev/null; then
        echo -e "\033[1;33m⚠️  Swap already exists on this system.\033[0m"
        swapon --show
        echo
        read -p "Continue anyway? (y/N): " confirm
        [[ "$confirm" =~ ^[Yy]$ ]] || return 1
    fi
    return 0
}

create_swapfile() {
    local size="$1"
    local swapfile="/swapfile"

    if [[ -f "$swapfile" ]]; then
        echo "✅ Swapfile already exists. Re-activating..."
        sudo swapon "$swapfile" 2>/dev/null || true
        return
    fi

    echo "Creating ${size} swapfile..."
    sudo fallocate -l "$size" "$swapfile" 2>/dev/null || {
        echo "Using dd fallback..."
        local count=$(( $(numfmt --from=iec "$size" 2>/dev/null || echo 4194304) / 1048576 ))
        sudo dd if=/dev/zero of="$swapfile" bs=1M count=$count status=progress
    }

    sudo chmod 600 "$swapfile"
    sudo mkswap "$swapfile"

    if ! grep -qF "$swapfile" /etc/fstab; then
        echo "$swapfile none swap sw 0 0" | sudo tee -a /etc/fstab >/dev/null
    fi

    sudo swapon "$swapfile"
    echo -e "✅ \033[1;32mSwapfile (${size}) successfully created and activated!\033[0m"
}

enable_zram() {
    local percent="$1"

    echo "Setting up zram with ${percent}% of RAM..."

    if ! command -v zramctl >/dev/null && ! systemctl list-unit-files | grep -q zram; then
        echo "Installing zram support..."
        install_package "zram-tools" || install_package "zram"
    fi

    # Best effort config for zram-tools (most Debian/Ubuntu derivatives)
    if [[ -f /etc/default/zramswap ]]; then
        sudo sed -i "s/^PERCENT=.*/PERCENT=${percent}/" /etc/default/zramswap 2>/dev/null || true
        sudo sed -i 's/^ALGO=.*/ALGO=lz4/' /etc/default/zramswap 2>/dev/null || true
    fi

    sudo systemctl restart zramswap 2>/dev/null || true
    sudo systemctl enable zramswap 2>/dev/null || true

    echo -e "✅ \033[1;32mzram enabled (${percent}% of RAM with lz4 compression)\033[0m"
}

tune_swappiness() {
    if ! grep -q "vm.swappiness" /etc/sysctl.conf 2>/dev/null; then
        echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf >/dev/null
        sudo sysctl -p >/dev/null 2>&1 || true
        echo "✅ Swappiness tuned to 10 (good for mixed zram + swapfile)"
    fi
}

show_status() {
    echo -e "\033[1;36m=== Current Swap Status ===\033[0m"
    free -h
    echo
    swapon --show 2>/dev/null || echo "No active swap"
    echo
    grep -E 'swap|Swap' /etc/fstab 2>/dev/null || echo "No swap entry in fstab"
}

# ===================== CONFIGURATION MENU =====================
configure_swap() {
    show_banner

    echo -e "\033[1;34m=== Swap Configuration ===\033[0m"
    echo "Your system has ${RAM_GB} GB RAM"
    echo

    # Swapfile size
    echo "Swapfile size recommendation:"
    echo "   • 2GB RAM  → 4GB swapfile"
    echo "   • 4GB RAM  → 4-8GB swapfile"
    echo "   • 8GB+ RAM → 4GB is usually enough with zram"
    echo
    read -p "Swapfile size [${RAM_GB}G recommended, default ${SWAP_SIZE_DEFAULT}]: " swap_size
    swap_size=${swap_size:-$SWAP_SIZE_DEFAULT}

    # Zram percentage
    echo
    echo "zram percentage of RAM (compressed RAM swap):"
    echo "   • 25%  = Light usage"
    echo "   • 50%  = Balanced (recommended)"
    echo "   • 100% = Aggressive (if you have very slow disk)"
    read -p "zram percentage [50]: " zram_percent
    zram_percent=${zram_percent:-50}

    echo
    echo -e "\033[1;33mYou chose:\033[0m"
    echo "   • Swapfile size : ${swap_size}"
    echo "   • zram           : ${zram_percent}% of RAM"
    echo
    read -p "Apply this configuration? (Y/n): " confirm
    [[ "$confirm" =~ ^[Nn]$ ]] && return 1

    # Apply
    detect_existing_swap || return 1

    enable_zram "$zram_percent"
    create_swapfile "$swap_size"
    tune_swappiness

    echo
    echo -e "\033[1;32mConfiguration applied successfully!\033[0m"
    echo "Reboot recommended to test everything."
}

# ===================== MAIN MENU =====================
main_menu() {
    while true; do
        show_banner

        echo -e "\033[1;34mMain Menu\033[0m"
        options=(
            "Smart Configuration (Recommended)"
            "Quick: zram + 4GB swapfile"
            "Show current swap status"
            "Reset to no-swap state"
            "Exit"
        )

        select opt in "${options[@]}"; do
            case "$REPLY" in
                1)
                    configure_swap
                    break
                    ;;
                2)
                    detect_existing_swap && {
                        enable_zram 50
                        create_swapfile "4G"
                        tune_swappiness
                    }
                    break
                    ;;
                3)
                    show_status
                    break
                    ;;
                4)
                    echo "Resetting..."
                    sudo swapoff -a 2>/dev/null || true
                    sudo rm -f /swapfile 2>/dev/null || true
                    sudo sed -i '/swap/d' /etc/fstab 2>/dev/null || true
                    echo -e "✅ Reset complete."
                    break
                    ;;
                5)
                    echo -e "\033[1;32mGoodbye! Stay swapped.\033[0m"
                    exit 0
                    ;;
                *)
                    echo "Invalid option."
                    ;;
            esac
        done

        echo
        read -p "Press Enter to return to menu..."
    done
}

# ===================== START =====================
main_menu