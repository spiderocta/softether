# softether


## Install Ubuntu and Set Static IP
### During Ubuntu installation, configure network manually or after installation:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
## Add this configuration (adjust to your network):
```bash
network:
  version: 2
  ethernets:
    ens160:  # Your network interface name (check with: ip a)
      dhcp4: no
      addresses:
        - 192.168.1.50/24  # Your VPN server IP
      gateway4: 192.168.1.1  # Your gateway
      nameservers:
        addresses:
          - 192.168.1.1  # Your DNS or 8.8.8.8
```

## Apply configurations
```bash
sudo netplan apply
```

## Part 2: Install SoftEther VPN Server (Linux VM Only)
### Update system and install dependencies
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y build-essential wget curl gcc make libreadline-dev \
libssl-dev zlib1g-dev cmake net-tools dnsutils
```

## Download and compile SoftEther VPN Server

```bash
# Go to /opt directory
cd /opt

# Download SoftEther (check latest version at https://www.softether-download.com/)
sudo wget https://github.com/SoftEtherVPN/SoftEtherVPN_Stable/releases/download/v4.43-9799-beta/softether-vpnserver-v4.43-9799-beta-2023.08.31-linux-x64-64bit.tar.gz

# Extract
sudo tar xzf softether-vpnserver-*.tar.gz

# Go to vpnserver directory
cd vpnserver

# Compile (you'll need to accept license by typing '1' three times)
sudo make
```

#### When prompted, type:

- 1 (yes to agree to license)
- 1 (yes to agree to license)
- 1 (yes to agree to license)


### Set permissions
```bash
cd /opt
sudo chmod 600 vpnserver/*
sudo chmod 700 vpnserver/vpncmd
sudo chmod 700 vpnserver/vpnserver
```

### Part 3: Create Systemd Service (Auto-start on boot)

```bash
# Create service file
sudo nano /etc/systemd/system/vpnserver.service
```

### Paste this content:
```bash
[Unit]
Description=SoftEther VPN Server
After=network.target network-online.target
Wants=network-online.target

[Service]
Type=forking
User=root
ExecStart=/opt/vpnserver/vpnserver start
ExecStop=/opt/vpnserver/vpnserver stop
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
RestartSec=3s

[Install]
WantedBy=multi-user.target
```

Save and exit (Ctrl+X, then Y, then Enter)

### Start and enable the service
```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable auto-start on boot
sudo systemctl enable vpnserver

# Start VPN server
sudo systemctl start vpnserver

# Check status (should show "active (running)")
sudo systemctl status vpnserver
```

## Part 4: Configure VPN Server via Command Line (vpncmd) 
### All configuration will be done through the Linux command line tool `vpncmd`.

#### Initial Setup - Set Admin Password

```bash
cd /opt/vpnserver

# Start vpncmd
sudo ./vpncmd
```

**You'll see a menu. Follow these steps:**
```
1. Select 1 (Management of VPN Server or VPN Bridge)
2. Hostname: Press ENTER (localhost)
3. Port: Press ENTER (default)
4. Password: Press ENTER (no password set yet)
```

**Now you're in vpncmd. Set the admin password:**
```
VPN Server> ServerPasswordSet
```

Enter a strong password twice. **Remember this password!**

**Exit:**
```
VPN Server> exit
```

## Part 5: Create Virtual Hub and Configure Network
### Create the Virtual Hub

```bash
sudo ./vpncmd


**Select:**
- `1` (Management of VPN Server)
- Press ENTER for hostname
- Press ENTER for port
- **Enter the admin password you just set**

**Create hub:**

VPN Server> HubCreate CompanyVPN


Enter a password for the hub (can be the same as server password).

### Switch to the hub context

VPN Server> Hub CompanyVPN

Enter the hub password.

### Enable SecureNAT (This advertises your private network)

**This is the KEY configuration that allows VPN clients to access your entire private network!**

VPN Server/CompanyVPN> SecureNatEnable
```

### Configure SecureNAT to bridge to your private network

```bash
# Set the virtual network gateway
VPN Server/CompanyVPN> DhcpSet /START:192.168.1.100 /END:192.168.1.150 /MASK:255.255.255.0 /EXPIRE:7200 /GW:192.168.1.1 /DNS:192.168.1.1 /DNS2:8.8.8.8 /DOMAIN:company.local /LOG:yes
```

### Explanation:

- `/START:192.168.1.100` - DHCP pool starts at .100
- `/END:192.168.1.150` - DHCP pool ends at .150 (51 IPs for VPN clients)
- `/MASK:255.255.255.0` - Subnet mask
- `/GW:192.168.1.1` - Your actual network gateway
- `/DNS:192.168.1.1` - Your DNS server (or use 8.8.8.8)
- This gives VPN clients IPs in YOUR ACTUAL network range!

### Configure NAT settings

```bash
VPN Server/CompanyVPN> SecureNatHostSet /MAC:none /IP:192.168.1.50 /MASK:255.255.255.0
```
- Where 192.168.1.50 is your VPN server's IP

## Part 6: Create User Accounts for Employees
### Create users one by one
```bash
# Still in Hub CompanyVPN context
VPN Server/CompanyVPN> UserCreate employee1 /GROUP:none /REALNAME:"John Doe" /NOTE:"IT Department"

# Set password
VPN Server/CompanyVPN> UserPasswordSet employee1

Enter password twice.

**Repeat for all 20 employees.**

### OR use this automated script

Exit vpncmd first:
VPN Server/CompanyVPN> exit
exit
```

## Create a script to add all users:

```bash
sudo nano /opt/create_vpn_users.sh
```

### Paste this content (modify usernames as needed):

```bash
#!/bin/bash

# User list (add all 20 employees)
USERS=(
    "employee1:John_Doe:IT_Department"
    "employee2:Jane_Smith:Sales"
    "employee3:Bob_Johnson:HR"
    "employee4:Alice_Williams:Finance"
    "employee5:Charlie_Brown:Operations"
    # Add remaining 15 users here
    # Format: "username:Real_Name:Department"
)

# Default password (users should change this)
DEFAULT_PASS="TempPass123!"

# VPN Server settings
HUB_NAME="CompanyVPN"

cd /opt/vpnserver

echo "Creating VPN users..."

for user_info in "${USERS[@]}"; do
    IFS=':' read -r username realname department <<< "$user_info"
    
    echo "Creating user: $username ($realname)"
    
    # Create user
    sudo ./vpncmd localhost /SERVER /HUB:$HUB_NAME /ADMINHUB:$HUB_NAME /CMD UserCreate $username /GROUP:none /REALNAME:"$realname" /NOTE:"$department"
    
    # Set password
    sudo ./vpncmd localhost /SERVER /HUB:$HUB_NAME /ADMINHUB:$HUB_NAME /CMD UserPasswordSet $username /PASSWORD:$DEFAULT_PASS
done

echo ""
echo "========================================"
echo "All users created successfully!"
echo "Default password: $DEFAULT_PASS"
echo "Instruct users to change password on first login"
echo "========================================"
```

## Make executable and run:

```bash
sudo chmod +x /opt/create_vpn_users.sh
sudo /opt/create_vpn_users.sh
```

## Part 7: Enable VPN Protocols
### Enable L2TP/IPsec (for easy client connection)

```bash
cd /opt/vpnserver
sudo ./vpncmd
```

### Select 1, press ENTER twice, enter admin password

```bash
VPN Server> IPsecEnable /L2TP:yes /L2TPRAW:yes /ETHERIP:no /PSK:YourCompanySecretKey123! /DEFAULTHUB:CompanyVPN

**Replace `YourCompanySecretKey123!` with your own secure pre-shared key.**

**Exit:**

VPN Server> exit
```

## Part 8: Enable VPN Azure (No Public IP Solution)
- This is the magic that makes it work without a public IP or cloud middleware!

```bash
cd /opt/vpnserver
sudo ./vpncmd
```

- Select 1, ENTER, ENTER, enter password

```bash
VPN Server> VpnAzureSetEnable yes
```

- Get your VPN Azure hostname:

```bash
VPN Server> VpnAzureGetStatus

**You'll see output like:**


VPN Azure Hostname: vpn123456789.vpnazure.net


**Write down this hostname! This is what employees will connect to!**

**Exit:**

VPN Server> exit
```

## Part 9: Configure Firewall on Linux VM

```bash
# Allow VPN ports
sudo ufw allow 443/tcp
sudo ufw allow 992/tcp  
sudo ufw allow 1194/tcp
sudo ufw allow 1194/udp
sudo ufw allow 5555/tcp
sudo ufw allow 500/udp
sudo ufw allow 4500/udp
sudo ufw allow 1701/udp

# Enable IP forwarding
sudo nano /etc/sysctl.conf

**Find and uncomment (or add) this line:**

net.ipv4.ip_forward=1
```

Part 10: Router Configuration (Port Forwarding - Optional)
- If you have a public IP, forward these ports from your router to 192.168.1.50:

<img width="976" height="392" alt="image" src="https://github.com/user-attachments/assets/8b34758d-f476-4698-8836-61e39bd13a4c" />

- If you DON'T have a public IP, VPN Azure handles this automatically via relay!


## Part 11: Verify Everything is Working

### Check service status
```bash
sudo systemctl status vpnserver
```
- Should show: `active (running)`

### Check VPN Azure connection

```bash
cd /opt/vpnserver
sudo ./vpncmd localhost /SERVER /CMD VpnAzureGetStatus
```

- Should show: Connected and give you the hostname.

### Check listening ports

```bash
sudo netstat -tulpn | grep vpnserver
```

### List all users

```bash
sudo ./vpncmd localhost /SERVER /HUB:CompanyVPN /CMD UserList
```

## Part 12: Client Connection Instructions for Employees

### Connection Information to Give Employees

**For L2TP/IPsec (Native VPN - Easiest):**
```
Server Address: vpn123456789.vpnazure.net (your VPN Azure hostname)
VPN Type: L2TP/IPsec
Pre-shared Key: YourCompanySecretKey123!
Username: employee1
Password: [their password]
```

#### Windows 10/11 Client Setup

1- Open Settings → Network & Internet → VPN
2- Click "Add VPN"
3- Fill in:

  - VPN Provider: Windows (built-in)
  - Connection Name: Company VPN
  - Server name or address: vpn123456789.vpnazure.net
  - VPN type: L2TP/IPsec with pre-shared key
  - Pre-shared key: YourCompanySecretKey123!
  - Type of sign-in info: User name and password
  - User name: employee1
  - Password: [their password]


4- Click Save
5- Click "Connect"

### After connecting, they can access:
- Internal servers: \\192.168.1.x
- Windows Server: \\192.168.1.10
- Sophos admin: https://192.168.1.x:4444
- All internal resources!

### macOS Client Setup

1- System Preferences → Network
2- Click "+" → Interface: VPN → VPN Type: L2TP over IPsec
3- Service Name: Company VPN
4- Server Address: vpn123456789.vpnazure.net
5- Account Name: employee1
6- Click "Authentication Settings":

  - Password: [their password]
  - Shared Secret: YourCompanySecretKey123!


7- Click OK → Connect

### Linux Client Setup

```bash
# Install network-manager-l2tp
sudo apt install network-manager-l2tp network-manager-l2tp-gnome

# Then use GUI to add VPN with same settings as above
```

### Android/iOS Client Setup

1- Settings → VPN → Add VPN
2- Type: L2TP/IPsec PSK
3- Server: vpn123456789.vpnazure.net
4- IPsec pre-shared key: YourCompanySecretKey123!
  - Username: employee1
  - Password: [their password]


## Part 13: Monitoring and Maintenance

- Check active connections

```bash
cd /opt/vpnserver
sudo ./vpncmd localhost /SERVER /CMD SessionList
```

- View logs

```bash
# Server logs
sudo tail -f /opt/vpnserver/server_log/*.log

# Security logs
sudo tail -f /opt/vpnserver/security_log/*.log
```

- Backup configuration

```bash
# Create backup
sudo tar czf /root/vpnserver_backup_$(date +%Y%m%d).tar.gz /opt/vpnserver

# Restore from backup
sudo tar xzf /root/vpnserver_backup_20240207.tar.gz -C /opt/
sudo systemctl restart vpnserver
```

# FIX : Use Local Bridge (More Advanced - Direct Network Access)
- This method creates a true bridge between the VPN and your physical network. VPN clients become actual members of your LAN.

1- Disable secureNat 

```bash
cd /opt/vpnserver
sudo ./vpncmd localhost /SERVER /HUB:CompanyVPN
```

```bash
VPN Server/CompanyVPN> SecureNatDisable
VPN Server/CompanyVPN> exit
```

## Step 2: Create Local Bridge
```bash
# First, find your network interface name
VPN Server> ?

# Check your network interfaces
exit
```

```bash
ip addr show
```

- Look for your network interface (usually ens160, ens192, eth0, etc.). Note the name.

```bash
# Back to vpncmd
sudo ./vpncmd localhost /SERVER
```

```
# Back to vpncmd
sudo ./vpncmd localhost /SERVER
```

```bash
# Create bridge (replace 'ens160' with your actual interface name)
VPN Server> BridgeCreate CompanyVPN /DEVICE:ens160

# Verify bridge
VPN Server> BridgeList
```

### Step 3: Configure Network Interface
- Your Linux VM's network interface needs to be in promiscuous mode:

```bash
exit

# Replace ens160 with your interface name
sudo ip link set ens160 promisc on

# Make it permanent
sudo nano /etc/rc.local

```

- Add this line (replace ens160 with your interface):

```bash
#!/bin/bash
ip link set ens160 promisc on
exit 0
```

```bash
sudo chmod +x /etc/rc.local
```


























