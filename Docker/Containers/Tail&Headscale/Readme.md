HEADSCALE EXIT NODE SETUP GUIDE
==============================

This guide explains how to set up a Headscale exit node on a Proxmox LXC container, allowing your iPhone to route traffic through your server while maintaining access to local services.
 
PREREQUISITES
------------
• Proxmox LXC container
• Docker and docker-compose installed
• Ports forwarded: 
  - 3478/udp (STUN)
  - 9443/tcp (DERP)
  - 50443/tcp (gRPC)

STEP-BY-STEP SETUP
-----------------

1. CONTAINER SETUP
   -------------
   When creating your LXC container, ensure it has these features:
   • features: nesting=1,keyctl=1
   • Must be privileged (cannot be changed after creation)

   Create a new privileged container:
   ```
   pct create 103 local:vztmpl/debian-12-standard_12.2-1_amd64.tar.zst \
   --hostname headscale \
   --memory 4024 \
   --swap 1024 \
   --rootfs local-lvm:10 \
   --net0 name=eth0,bridge=vmbr0,firewall=1,ip=dhcp \
   --unprivileged 0 \
   --features nesting=1,keyctl=1
   ```

2. IP FORWARDING AND NAT
   -------------------
   A. Enable IP Forwarding:
      Add to /etc/sysctl.conf:
      ```
      net.ipv4.ip_forward = 1
      net.ipv6.conf.all.forwarding = 1
      ```
      Apply with: sudo sysctl -p

   B. Set up persistent NAT:
      1. Install: sudo apt-get install iptables-persistent
      
      2. Create /etc/iptables/rules.v4:
         ```
         *filter
         :INPUT ACCEPT [0:0]
         :FORWARD ACCEPT [0:0]
         :OUTPUT ACCEPT [0:0]
         -A FORWARD -i tailscale0 -j ACCEPT
         COMMIT
         *nat
         :PREROUTING ACCEPT [0:0]
         :INPUT ACCEPT [0:0]
         :OUTPUT ACCEPT [0:0]
         :POSTROUTING ACCEPT [0:0]
         -A POSTROUTING -o eth0 -j MASQUERADE
         COMMIT
         ```

      3. Create /etc/network/if-pre-up.d/iptables:
         ```
         #!/bin/sh
         /sbin/iptables-restore < /etc/iptables/rules.v4
         ```

      4. Make executable: sudo chmod +x /etc/network/if-pre-up.d/iptables

3. HEADSCALE CONFIGURATION
   ---------------------
   Update config.yaml:
   ```yaml
   derp:
     server:
       enabled: true
       region_id: 999
       region_code: "headscale"
       region_name: "Headscale Embedded DERP"
       stun_listen_addr: "0.0.0.0:3478"
       ipv4: "YOUR_PUBLIC_IP"
       port: 9443

   acls:
     - action: accept
       src: ["*"]
       dst: ["*:*"]
     - action: accept
       src: ["*"]
       dst: ["0.0.0.0/0:*"]
     - action: accept
       src: ["*"]
       dst: ["*:*"]
       enableSubnetRouting: true
   ```

4. DOCKER CONFIGURATION
   ------------------
   Update docker-compose.yaml ports:
   ```yaml
   ports:
     - "3478:3478/udp"  # STUN
     - "50443:50443"    # gRPC
     - "8080:8080"      # HTTP API
     - "9443:9443"      # DERP
   ```

5. EXIT NODE SETUP
   -------------
   A. Start services:
      ```
      docker-compose up -d
      ```

   B. Connect Tailscale:
      ```
      sudo tailscale up --login-server=https://headscale.petestech.com \
        --advertise-exit-node \
        --advertise-routes=192.168.8.0/24,0.0.0.0/0,::/0
      ```

   C. Enable routes:
      ```
      docker exec headscale headscale routes list
      docker exec headscale headscale routes enable -r X  # For each route ID
      ```

   D. Tag as exit node:
      ```
      docker exec headscale headscale nodes tag -i NODE_ID -t tag:exit-node
      ```

TROUBLESHOOTING
--------------
1. "Unable to modify read-only option: 'unprivileged'"
   • Solution: Create new privileged container
   • Cannot be modified after creation

2. "Tailscale could not connect to DERP relay server"
   • Check port forwarding: 3478/udp, 9443/tcp, 50443/tcp
   • Verify DERP configuration in config.yaml

3. "Cannot enable route: route not available"
   • Advertise routes first with --advertise-routes
   • Then enable routes with routes enable command

4. "0.0.0.0/0 advertised without IPv6 counterpart"
   • Include both IPv4 and IPv6 routes:
   • Use: --advertise-routes=192.168.8.0/24,0.0.0.0/0,::/0

IPHONE SETUP
-----------
1. Install Tailscale app
2. Log in with custom server: https://headscale.petestech.com
3. Get auth key:
   ```
   docker exec headscale headscale --namespace default preauthkeys create
   ```
4. Enable "Use VPN" in Tailscale settings

VERIFICATION
-----------
✓ Check whatismyip.com - Should show server's IP
✓ Access local network (192.168.8.x) - Should work with VPN
✓ Verify routes: docker exec headscale headscale routes list
