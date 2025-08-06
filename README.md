# Bastion Hardening
First line of defense for a bastion

```bash
$ sudo su -
$ wget https://raw.githubusercontent.com/just-digital/bastion/refs/heads/main/configure.sh
$ bash configure.sh <admin-user-name> <comma seperated IP whitelist>
```

# Testing Port Scan Detection and IP Whitelist

Here are comprehensive tests you can perform to verify the port scanning detection and whitelist functionality:

1. Port Scan Detection Tests

Test A: Basic Port Scan Detection

From a test machine (not whitelisted):

# Test from a remote machine - this will trigger the ban after 3 attempts
# (Remember: maxretry=2 means banned on the 3rd attempt)

# Attempt 1: Try a random closed port
nc -zv <server-ip> 8080

# Attempt 2: Try another closed port
nc -zv <server-ip> 443

# Attempt 3: This should trigger the ban
nc -zv <server-ip> 22

# Now verify you're banned (even valid SSH port won't work)
ssh -p 31221 qadmin@<server-ip>  # Should fail - connection refused

Test B: Using nmap (Aggressive Test)

# From test machine - this will immediately trigger ban
nmap -p 1-100 <server-ip>  # Scans first 100 ports

Test C: Telnet Test

# Try connecting to multiple closed ports
telnet <server-ip> 80
telnet <server-ip> 443
telnet <server-ip> 3306  # This 3rd attempt should trigger ban

2. Whitelist Functionality Tests

First, run the configure script with whitelisted IPs:
sudo ./configure.sh qadmin "192.168.1.100,10.0.0.5"

Test A: Verify Whitelist is Active

On the server:
# Check fail2ban whitelist configuration
sudo fail2ban-client get port-scan ignoreip
sudo fail2ban-client get sshd ignoreip

# Should show: 127.0.0.1/8 ::1 192.168.1.100 10.0.0.5

Test B: Port Scan from Whitelisted IP

From a whitelisted machine (e.g., 192.168.1.100):
# These should NOT result in a ban
nmap -p 1-1000 <server-ip>  # Full port scan
nc -zv <server-ip> 80
nc -zv <server-ip> 443
nc -zv <server-ip> 8080
nc -zv <server-ip> 3306

# Verify you can still connect via SSH
ssh -p 31221 qadmin@<server-ip>  # Should work

3. Monitoring Commands

Run these on the server to monitor the detection system:

Real-time Monitoring

# Watch UFW blocks in real-time
sudo tail -f /var/log/ufw.log | grep "BLOCK"

# Watch fail2ban activity
sudo tail -f /var/log/fail2ban.log

# Watch auth attempts
sudo tail -f /var/log/auth.log

Check Ban Status

# See all banned IPs in port-scan jail
sudo fail2ban-client status port-scan

# Get detailed banned list
sudo fail2ban-client get port-scan banned

# Check how many IPs are currently banned
sudo iptables -L -n | grep -c "REJECT"

Verify Specific IP Status

# Check if specific IP is banned
sudo iptables -L -n | grep "<test-ip>"

# Check fail2ban database for IP
sudo fail2ban-client get port-scan banned | grep "<test-ip>"

4. Manual Ban/Unban Testing

Manually Ban an IP

sudo fail2ban-client set port-scan banip 192.168.1.50

Manually Unban an IP

sudo fail2ban-client set port-scan unbanip 192.168.1.50

Add IP to Whitelist After Setup

sudo fail2ban-client set port-scan addignoreip 192.168.1.200
sudo fail2ban-client set sshd addignoreip 192.168.1.200

5. Verification Tests

Test A: Verify Port Scan Detection is Working

# Check the filter is correctly parsing logs
sudo fail2ban-regex /var/log/ufw.log /etc/fail2ban/filter.d/port-scan.conf

# Test with sample log line
echo "Dec 10 10:00:00 server kernel: [UFW BLOCK] IN=eth0 OUT= MAC=... SRC=192.168.1.50 DST=10.0.0.1 LEN=40 TOS=0x00 PREC=0x00 TTL=51 ID=0 DF
PROTO=TCP SPT=54321 DPT=80 WINDOW=1024 RES=0x00 SYN URGP=0" | sudo fail2ban-regex - /etc/fail2ban/filter.d/port-scan.conf

Test B: Check Jail Configuration

# Verify jail settings
sudo fail2ban-client get port-scan maxretry  # Should show: 2
sudo fail2ban-client get port-scan findtime  # Should show: 1209600
sudo fail2ban-client get port-scan bantime   # Should show: 31536000

6. Test Scenarios Summary

| Test Case                         | From IP       | Action                                                       | Expected Result       |
|-----------------------------------|---------------|--------------------------------------------------------------|-----------------------|
| Port scan - Non-whitelisted       | 192.168.1.50  | Access 3 different ports                                     | Banned for 1 year     |
| Port scan - Whitelisted           | 192.168.1.100 | Access 100 ports                                             | No ban                |
| SSH brute force - Non-whitelisted | 192.168.1.60  | 4 failed SSH attempts                                        | Banned for 2 hours    |
| Mixed ports - Non-whitelisted     | 192.168.1.70  | Access port 80, wait 1 week, access port 443, then port 3306 | Banned on 3rd attempt |
| Slow scan - Non-whitelisted       | 192.168.1.80  | Access 1 port per week for 3 weeks                           | Banned on 3rd port    |

7. Important Notes

1. Testing Safety: Always test from a machine you can afford to lose access to, or ensure you have console/physical access to the server.
2. Whitelist Yourself First: Before testing, consider whitelisting your management IP:
sudo ./configure.sh qadmin "YOUR_MANAGEMENT_IP"
3. Log Locations:
- Port scan attempts: /var/log/ufw.log
- Fail2ban actions: /var/log/fail2ban.log
- SSH attempts: /var/log/auth.log
4. Recovery: If you accidentally ban yourself:
- Use console access or a whitelisted IP
- Run: sudo fail2ban-client set port-scan unbanip YOUR_IP

