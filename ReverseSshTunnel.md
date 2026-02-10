Content is user-generated and unverified.
Reverse SSH Tunnel Implementation — Instructions for Claude

You are helping me set up a secure reverse SSH tunnel so I can access my Orange Pi (running Ubuntu ARM) from anywhere using the Terminus Android SSH app. The relay server is my existing Linode VPS (Fedora 43 Server), which already runs an Angular SSR application. We are dual-purposing this Linode.

Architecture overview:

[Terminus on Android] → SSH → [Linode VPS :22] → localhost:19222 → [Orange Pi :2222]
                                     ↑                                      |
                              reverse tunnel kept alive by ─────────────────┘
                              the Orange Pi's autossh service

The Orange Pi initiates an outbound SSH connection to the Linode and opens a reverse tunnel. The tunnel binds to localhost:19222 on the Linode. To reach the Pi, I SSH into the Linode first, then connect to localhost:19222, or I use ProxyJump from Terminus to do it in one step.

Security is the top priority. Every step must follow the hardening rules below.
PART 1: LINODE VPS CONFIGURATION

Perform all of the following on the Linode VPS. Assume I already have root or sudo access and SSH is already running for my normal admin user.
1.1 — Create a Restricted Tunnel User

Create a dedicated user account that exists solely to anchor the reverse tunnel. This user must have no shell, no home directory contents of value, and no ability to do anything other than hold a port forward.
bash

# Create the user with no login shell
sudo useradd -m -s /usr/sbin/nologin tunneluser

# Create .ssh directory for authorized_keys
sudo mkdir -p /home/tunneluser/.ssh
sudo chmod 700 /home/tunneluser/.ssh
sudo touch /home/tunneluser/.ssh/authorized_keys
sudo chmod 600 /home/tunneluser/.ssh/authorized_keys
sudo chown -R tunneluser:tunneluser /home/tunneluser/.ssh

1.2 — Harden SSHD Config on the Linode

Edit /etc/ssh/sshd_config. Preserve the existing configuration for my admin user. Append or modify these directives:

Global settings (apply to all users unless overridden by Match block):

# --- Global hardening ---
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowTcpForwarding no          # Deny by default globally

    Important: If you already have AllowUsers set, add tunneluser to it. If not, add:

    AllowUsers YOUR_ADMIN_USERNAME tunneluser

    Replace YOUR_ADMIN_USERNAME with my actual admin login.

Match block for the tunnel user (append at the END of sshd_config):

# --- Reverse tunnel user restrictions ---
Match User tunneluser
    AllowTcpForwarding remote       # Only remote (reverse) forwarding allowed
    PermitTTY no                    # No interactive terminal
    PermitTunnel no                 # No VPN-style tunneling
    GatewayPorts no                 # Tunnel binds to localhost ONLY
    X11Forwarding no
    AllowAgentForwarding no
    ForceCommand /bin/false         # Cannot execute any commands
    PermitOpen none                 # Cannot open local forwards

After editing, validate and restart:
bash

sudo sshd -t          # Test config for syntax errors — fix any before proceeding
sudo systemctl restart sshd

    CRITICAL: Do NOT close your current SSH session until you have verified you can still log in as your admin user in a separate terminal. If you lock yourself out of the Linode, recovery is painful.

1.3 — Firewall (firewalld) on the Linode

The Linode runs Fedora 43, which uses firewalld (not UFW). The firewall is already active with the FedoraServer zone. Verify and tighten:
bash

# Check existing rules
sudo firewall-cmd --list-all

# Ensure ssh, http, https are allowed (should already be)
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https

# Remove cockpit if not in use (web admin on port 9090 — unnecessary attack surface)
sudo firewall-cmd --permanent --remove-service=cockpit

# Remove dhcpv6-client if not needed
sudo firewall-cmd --permanent --remove-service=dhcpv6-client

# Reload to apply changes
sudo firewall-cmd --reload

# Verify final state
sudo firewall-cmd --list-all

No new ports need to be opened. The reverse tunnel binds to localhost:19222 on the Linode — it is not externally accessible. You will connect to the Pi by first SSH-ing into the Linode on port 22, then hopping to localhost:19222.
1.4 — Install and Configure fail2ban on the Linode
bash

sudo dnf install -y fail2ban

Create /etc/fail2ban/jail.local:
ini

[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3
backend  = systemd

[sshd]
enabled  = true
port     = 22
filter   = sshd
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 3600

    Note: On Fedora, use %(sshd_log)s which resolves to the systemd journal automatically.

bash

sudo systemctl enable fail2ban
sudo systemctl start fail2ban

1.5 — Enable Automatic Security Updates on the Linode

Fedora uses dnf-automatic instead of unattended-upgrades:
bash

sudo dnf install -y dnf-automatic

Edit /etc/dnf/automatic.conf:
ini

[commands]
apply_updates = yes
upgrade_type = security

bash

sudo systemctl enable dnf-automatic.timer
sudo systemctl start dnf-automatic.timer

Verify the timer is active:
bash

sudo systemctl status dnf-automatic.timer
PART 2: ORANGE PI CONFIGURATION

Perform all of the following on the Orange Pi running Ubuntu ARM.
2.1 — Harden SSH on the Orange Pi
2.1.1 — Generate a Client Keypair (for Terminus on Android)

On your Android device in Terminus, generate an Ed25519 keypair. Export/copy the public key. Then add it to the Orange Pi:
bash

# On the Orange Pi, as your normal user:
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "PASTE_YOUR_TERMINUS_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

    Replace PASTE_YOUR_TERMINUS_PUBLIC_KEY_HERE with the actual public key from Terminus. It will look like ssh-ed25519 AAAA... user@device.

2.1.2 — Edit SSHD Config on the Orange Pi

Edit /etc/ssh/sshd_config:

Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers YOUR_PI_USERNAME
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no

    Replace YOUR_PI_USERNAME with the actual username on the Orange Pi.

Validate and restart:
bash

sudo sshd -t
sudo systemctl restart sshd

    Test that you can log in with your key on port 2222 before closing any existing sessions.

2.1.3 — Firewall (UFW) on the Orange Pi
bash

sudo apt update && sudo apt install -y ufw

sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH only on the custom port
sudo ufw allow 2222/tcp

sudo ufw enable

2.1.4 — Install fail2ban on the Orange Pi
bash

sudo apt install -y fail2ban

Create /etc/fail2ban/jail.local:
ini

[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 3
backend  = systemd

[sshd]
enabled  = true
port     = 2222
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600

bash

sudo systemctl enable fail2ban
sudo systemctl start fail2ban

2.1.5 — Enable Unattended Security Updates on the Orange Pi
bash

sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

2.2 — Generate the Tunnel Keypair

This keypair is used only by the Orange Pi to authenticate to the Linode as tunneluser. It runs unattended, so it has no passphrase.
bash

# On the Orange Pi, as your normal user:
ssh-keygen -t ed25519 -f ~/.ssh/tunnel_key -N "" -C "orangepi-tunnel"

Now copy the public key to the Linode's tunneluser:
bash

# Display the public key
cat ~/.ssh/tunnel_key.pub

On the Linode, paste this key into /home/tunneluser/.ssh/authorized_keys:
bash

# On the Linode:
echo "PASTE_TUNNEL_PUBLIC_KEY_HERE" | sudo tee /home/tunneluser/.ssh/authorized_keys
sudo chown tunneluser:tunneluser /home/tunneluser/.ssh/authorized_keys
sudo chmod 600 /home/tunneluser/.ssh/authorized_keys

Optional but Recommended: Restrict the Tunnel Key Further

On the Linode, edit /home/tunneluser/.ssh/authorized_keys to prefix the key with restrictions:

no-pty,no-X11-forwarding,no-agent-forwarding,permitopen="none",command="/bin/false" ssh-ed25519 AAAA... orangepi-tunnel

This adds defense-in-depth on top of the sshd_config Match block.
2.3 — Test the Tunnel Connection Manually

From the Orange Pi, test that the tunnel user can connect:
bash

ssh -i ~/.ssh/tunnel_key -o StrictHostKeyChecking=accept-new -p 22 tunneluser@YOUR_LINODE_IP -N -R 19222:localhost:2222

    Replace YOUR_LINODE_IP with the Linode's public IP address.

This should hang (no shell is provided, which is correct). In a separate terminal on the Linode, verify:
bash

ss -tlnp | grep 19222

You should see port 19222 listening on 127.0.0.1. Then test the full connection from the Linode:
bash

ssh -p 19222 YOUR_PI_USERNAME@localhost

This should connect you to the Orange Pi. If it works, kill the manual tunnel (Ctrl+C on the Pi) and proceed to set up the persistent service.
2.4 — Install autossh
bash

sudo apt install -y autossh

2.5 — Create the Systemd Service

Create /etc/systemd/system/ssh-tunnel.service:
ini

[Unit]
Description=Persistent Reverse SSH Tunnel to Linode
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=YOUR_PI_USERNAME
Environment="AUTOSSH_GATETIME=0"
Environment="AUTOSSH_POLL=60"

ExecStart=/usr/bin/autossh -M 0 -N \
    -o "ServerAliveInterval=30" \
    -o "ServerAliveCountMax=3" \
    -o "ExitOnForwardFailure=yes" \
    -o "StrictHostKeyChecking=accept-new" \
    -o "IdentitiesOnly=yes" \
    -i /home/YOUR_PI_USERNAME/.ssh/tunnel_key \
    -R 19222:localhost:2222 \
    -p 22 \
    tunneluser@YOUR_LINODE_IP

Restart=always
RestartSec=15

[Install]
WantedBy=multi-user.target

    Replace YOUR_PI_USERNAME (3 occurrences) and YOUR_LINODE_IP (1 occurrence) with actual values.

Enable and start:
bash

sudo systemctl daemon-reload
sudo systemctl enable ssh-tunnel.service
sudo systemctl start ssh-tunnel.service

Verify it's running:
bash

sudo systemctl status ssh-tunnel.service

Check the tunnel is alive from the Linode:
bash

ss -tlnp | grep 19222

2.6 — Optional: Health Check Cron Job

Create a script at /home/YOUR_PI_USERNAME/check-tunnel.sh:
bash

#!/bin/bash
# Check if the tunnel port is alive on the Linode
# If not, restart the service

if ! ssh -i /home/YOUR_PI_USERNAME/.ssh/tunnel_key \
    -p 22 tunneluser@YOUR_LINODE_IP \
    -o ConnectTimeout=10 \
    -o StrictHostKeyChecking=accept-new \
    -N -f -R 19222:localhost:2222 2>/dev/null; then
    sudo systemctl restart ssh-tunnel.service
fi

    Actually, autossh and systemd's Restart=always should handle reconnections. This cron is an extra safety net. A simpler check:

bash

#!/bin/bash
if ! systemctl is-active --quiet ssh-tunnel.service; then
    systemctl restart ssh-tunnel.service
fi

bash

chmod +x /home/YOUR_PI_USERNAME/check-tunnel.sh

Add to crontab:
bash

crontab -e
# Add:
*/5 * * * * /home/YOUR_PI_USERNAME/check-tunnel.sh

PART 3: TERMINUS (ANDROID) CONFIGURATION
3.1 — Option A: Two-Hop Connection (Simple)

    Host 1 — Linode:
        Hostname: YOUR_LINODE_IP
        Port: 22
        Username: YOUR_ADMIN_USERNAME
        Auth: Your Linode admin private key
    Host 2 — Orange Pi (via Linode):
        Hostname: localhost
        Port: 19222
        Username: YOUR_PI_USERNAME
        Auth: The Terminus Ed25519 private key

Connect to Host 1 first, then from within that session, connect to Host 2.
3.2 — Option B: ProxyJump (Single-Step, Preferred)

If Terminus supports SSH config or ProxyJump/Jump Host, configure it as:

Host linode-jump
    HostName YOUR_LINODE_IP
    Port 22
    User YOUR_ADMIN_USERNAME
    IdentityFile ~/.ssh/your_linode_key

Host orangepi
    HostName localhost
    Port 19222
    User YOUR_PI_USERNAME
    IdentityFile ~/.ssh/your_terminus_ed25519_key
    ProxyJump linode-jump

With this, connecting to "orangepi" in Terminus will automatically hop through the Linode.

    Note: Terminus must have both private keys loaded — the one for the Linode admin user and the one for the Orange Pi user. These are two separate keypairs.

PART 4: SECURITY SUMMARY & CHECKLIST

Before considering this complete, verify every item:
Authentication

    Password authentication disabled on both Pi and Linode
    Root login disabled on both Pi and Linode
    Ed25519 keys used everywhere
    Terminus private key has a passphrase set
    Tunnel key (no passphrase) is used only for the tunnel, nothing else
    AllowUsers restricts which accounts can log in on both machines

Tunnel Isolation

    tunneluser has /usr/sbin/nologin as shell
    Match User tunneluser block restricts forwarding, TTY, and commands
    GatewayPorts no ensures tunnel binds to localhost only
    authorized_keys for tunneluser has no-pty,command="/bin/false" prefix
    Tunnel port (19222) is NOT exposed in firewall — it's localhost only

Network

    Firewall enabled on both machines (firewalld on Linode, UFW on Orange Pi) with default deny incoming
    fail2ban running on both machines
    Orange Pi SSH runs on non-standard port (2222)
    No ports forwarded on home router

Maintenance

    Unattended security upgrades enabled on both machines
    autossh + systemd service keeps tunnel alive with auto-restart
    Health check cron as backup

Keys Inventory
Key	Lives On	Authenticates To	Passphrase?
Terminus Ed25519	Android phone	Orange Pi (your user)	YES
Linode admin key	Android phone	Linode (admin user)	YES
Tunnel key	Orange Pi	Linode (tunneluser)	No (unattended)
PART 5: PLACEHOLDER VALUES TO REPLACE

Before running any commands, replace these placeholders throughout:
Placeholder	Replace With
YOUR_LINODE_IP	Your Linode's public IPv4 address
YOUR_ADMIN_USERNAME	Your admin user on the Linode
YOUR_PI_USERNAME	Your user on the Orange Pi
PASTE_YOUR_TERMINUS_PUBLIC_KEY_HERE	The Ed25519 public key from Terminus
PASTE_TUNNEL_PUBLIC_KEY_HERE	Contents of ~/.ssh/tunnel_key.pub from the Pi
Troubleshooting

Tunnel won't connect:
bash

# On the Pi, run manually with verbose output:
ssh -vvv -i ~/.ssh/tunnel_key -p 22 tunneluser@YOUR_LINODE_IP -N -R 19222:localhost:2222

Port 19222 not listening on Linode:
bash

# Check if something else is using it:
ss -tlnp | grep 19222
# Check tunnel service status:
journalctl -u ssh-tunnel.service -f   # on the Pi

"Connection refused" on port 19222 from Linode:

    The tunnel is down. Check systemctl status ssh-tunnel.service on the Pi.
    Ensure autossh is installed and the service is running.

Locked out of Linode after sshd_config change:

    Use the Linode console (Lish) from the Linode web dashboard to recover.
    Always test new SSH config in a separate session before closing the original.


