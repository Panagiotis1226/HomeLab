Setting up Headscale + DERP for Tailscale
=========================================

This guide explains how to set up a Headscale exit node on a Proxmox LXC container, allowing your iPhone or any device to route traffic through your server while maintaining access to local services.

PREREQUISITES
------------
• Proxmox LXC container <br>
• Docker and docker-compose installed<br>
• A Domain Name <br>
• Ports forwarded: 
  - 3478/udp (STUN)
  - 9443/tcp (DERP)
  - 50443/tcp (gRPC)

STEP-BY-STEP SETUP
-----------------

1. CONTAINER SETUP
   -------------
   When creating your LXC container, ensure it has these features: <br>
   • features: nesting=1,keyctl=1 <br>
   • Must be privileged (cannot be changed after creation)

   Create a new privileged container (Do not auto start) <br>
   Once created go to proxmox shell and modify the new container
   ```
   nano /etc/pve/lxc/<containerID>.conf
   
   features: nesting=1,keyctl=1,mknod=1
   lxc.cgroup2.devices.allow: c 10:200 rwm
   lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
   ```

3. IP FORWARDING AND NAT
   -------------------
   A. Enable IP Forwarding: <br>
      Add to /etc/sysctl.conf:
      ```
      net.ipv4.ip_forward = 1
      net.ipv6.conf.all.forwarding = 1
      ```
      Apply with: sudo sysctl -p

   B. Set up persistent NAT:
      1. Install: sudo apt install iptables-persistent
      
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

4. HEADSCALE CONFIGURATION
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

5. DOCKER CONFIGURATION
   ------------------
   Your setup requires three main configuration files:

   1. docker-compose.yaml:
      ```yaml
      volumes:
        caddy:
        headscale:
      
      services:
        caddy:
          image: "caddy"
          container_name: "caddy"
          restart: unless-stopped
          ports:
            - "80:80"
            - "443:443"
          volumes:
            - caddy:/data
            - ./Caddyfile:/etc/caddy/Caddyfile
      
        headscale:
          image: "headscale/headscale"
          container_name: "headscale"
          command: serve
          restart: unless-stopped
          ports:
            - "3478:3478/udp"  # STUN
            - "50443:50443"    # gRPC
          volumes:
            - headscale:/var/lib/headscale
            - ./config.yaml:/etc/headscale/config.yaml
      
        headscale-ui:
          image: "simcu/headscale-ui"
          container_name: "headscale-ui"
          restart: unless-stopped  
      ```

   2. config.yaml:
      ```yaml
      server_url: https://sub.domain.com

      # Address to listen to / bind to on the server
      #
      # For production:
      # listen_addr: 0.0.0.0:8080
      listen_addr: 0.0.0.0:8080
      
      # Address to listen to /metrics, you may want
      # to keep this endpoint private to your internal
      # network
      #
      metrics_listen_addr: 127.0.0.1:9090
      
      # Address to listen for gRPC.
      # gRPC is used for controlling a headscale server
      # remotely with the CLI
      # Note: Remote access _only_ works if you have
      # valid certificates.
      #
      # For production:
      # grpc_listen_addr: 0.0.0.0:50443
      grpc_listen_addr: 0.0.0.0:50443
      
      # Allow the gRPC admin interface to run in INSECURE
      # mode. This is not recommended as the traffic will
      # be unencrypted. Only enable if you know what you
      # are doing.
      grpc_allow_insecure: false
      
      # The Noise section includes specific configuration for the
      # TS2021 Noise protocol
      noise:
        # The Noise private key is used to encrypt the
        # traffic between headscale and Tailscale clients when
        # using the new Noise-based protocol.
        private_key_path: /var/lib/headscale/noise_private.key
      
      # List of IP prefixes to allocate tailaddresses from.
      # Each prefix consists of either an IPv4 or IPv6 address,
      # and the associated prefix length, delimited by a slash.
      # It must be within IP ranges supported by the Tailscale
      # client - i.e., subnets of 100.64.0.0/10 and fd7a:115c:a1e0::/48.
      # See below:
      # IPv6: https://github.com/tailscale/tailscale/blob/22ebb25e833264f58d7c3f534a8b166894a89536/net/tsaddr/tsaddr.go#LL81C52-L81C71
      # IPv4: https://github.com/tailscale/tailscale/blob/22ebb25e833264f58d7c3f534a8b166894a89536/net/tsaddr/tsaddr.go#L33
      # Any other range is NOT supported, and it will cause unexpected issues.
      prefixes:
        v6: fd7a:115c:a1e0::/48
        v4: 100.64.0.0/10
      
        # Strategy used for allocation of IPs to nodes, available options:
        # - sequential (default): assigns the next free IP from the previous given IP.
        # - random: assigns the next free IP from a pseudo-random IP generator (crypto/rand).
        allocation: sequential
      
      # DERP is a relay system that Tailscale uses when a direct
      # connection cannot be established.
      # https://tailscale.com/blog/how-tailscale-works/#encrypted-tcp-relays-derp
      #
      # headscale needs a list of DERP servers that can be presented
      # to the clients.
      derp:
        server:
          # If enabled, runs the embedded DERP server and merges it into the rest of the DERP config
          # The Headscale server_url defined above MUST be using https, DERP requires TLS to be in place
          enabled: true
      
          # Region ID to use for the embedded DERP server.
          # The local DERP prevails if the region ID collides with other region ID coming from
          # the regular DERP config.
          region_id: 999
      
          # Region code and name are displayed in the Tailscale UI to identify a DERP region
          region_code: "headscale"
          region_name: "Headscale Embedded DERP"
      
          # Listens over UDP at the configured address for STUN connections - to help with NAT traversal.
          # When the embedded DERP server is enabled stun_listen_addr MUST be defined.
          #
          # For more details on how this works, check this great article: https://tailscale.com/blog/how-tailscale-works/
          stun_listen_addr: "0.0.0.0:3478"
      
          # Private key used to encrypt the traffic between headscale DERP
          # and Tailscale clients.
          # The private key file will be autogenerated if it's missing.
          #
          private_key_path: /var/lib/headscale/derp_server_private.key
      
          # This flag can be used, so the DERP map entry for the embedded DERP server is not written automatically,
          # it enables the creation of your very own DERP map entry using a locally available file with the parameter DERP.paths
          # If you enable the DERP server and set this to false, it is required to add the DERP server to the DERP map using DERP.paths
          automatically_add_embedded_derp_region: true
      
          # For better connection stability (especially when using an Exit-Node and DNS is not working),
          # it is possible to optionally add the public IPv4 and IPv6 address to the Derp-Map using:
          ipv4: "PUBLIC_IP"  # Replace with your actual public IP
      
        # List of externally available DERP maps encoded in JSON
        urls: []
      
        # Locally available DERP map files encoded in YAML
        #
        # This option is mostly interesting for people hosting
        # their own DERP servers:
        # https://tailscale.com/kb/1118/custom-derp-servers/
        #
        # paths:
        #   - /etc/headscale/derp-example.yaml
        paths: []
      
        # If enabled, a worker will be set up to periodically
        # refresh the given sources and update the derpmap
        # will be set up.
        auto_update_enabled: true
      
        # How often should we check for DERP updates?
        update_frequency: 24h
      
      # Disables the automatic check for headscale updates on startup
      disable_check_updates: true
      
      # Time before an inactive ephemeral node is deleted?
      ephemeral_node_inactivity_timeout: 30m
      
      database:
        # Database type. Available options: sqlite, postgres
        # Please note that using Postgres is highly discouraged as it is only supported for legacy reasons.
        # All new development, testing and optimisations are done with SQLite in mind.
        type: sqlite
      
        # Enable debug mode. This setting requires the log.level to be set to "debug" or "trace".
        debug: false
      
        # GORM configuration settings.
        gorm:
          # Enable prepared statements.
          prepare_stmt: true
      
          # Enable parameterized queries.
          parameterized_queries: true
      
          # Skip logging "record not found" errors.
          skip_err_record_not_found: true
      
          # Threshold for slow queries in milliseconds.
          slow_threshold: 1000
      
        # SQLite config
        sqlite:
          path: /var/lib/headscale/db.sqlite
      
          # Enable WAL mode for SQLite. This is recommended for production environments.
          # https://www.sqlite.org/wal.html
          write_ahead_log: true
      
      ### TLS configuration
      #
      ## Let's encrypt / ACME
      #
      # headscale supports automatically requesting and setting up
      # TLS for a domain with Let's Encrypt.
      #
      # URL to ACME directory
      acme_url: https://acme-v02.api.letsencrypt.org/directory
      
      # Email to register with ACME provider
      acme_email: ""
      
      # Domain name to request a TLS certificate for:
      tls_letsencrypt_hostname: ""
      
      # Path to store certificates and metadata needed by
      # letsencrypt
      # For production:
      tls_letsencrypt_cache_dir: /var/lib/headscale/cache
      
      # Type of ACME challenge to use, currently supported types:
      # HTTP-01 or TLS-ALPN-01
      # See [docs/tls.md](docs/tls.md) for more information
      tls_letsencrypt_challenge_type: HTTP-01
      # When HTTP-01 challenge is chosen, letsencrypt must set up a
      # verification endpoint, and it will be listening on:
      # :http = port 80
      tls_letsencrypt_listen: ":http"
      
      ## Use already defined certificates:
      tls_cert_path: ""
      tls_key_path: ""
      
      log:
        # Output formatting for logs: text or json
        format: text
        level: info
      
      ## Policy
      # headscale supports Tailscale's ACL policies.
      # Please have a look to their KB to better
      # understand the concepts: https://tailscale.com/kb/1018/acls/
      policy:
        # The mode can be "file" or "database" that defines
        # where the ACL policies are stored and read from.
        mode: file
        # If the mode is set to "file", the path to a
        # HuJSON file containing ACL policies.
        path: ""
      
      ## DNS
      #
      # headscale supports Tailscale's DNS configuration and MagicDNS.
      # Please have a look to their KB to better understand the concepts:
      #
      # - https://tailscale.com/kb/1054/dns/
      # - https://tailscale.com/kb/1081/magicdns/
      # - https://tailscale.com/blog/2021-09-private-dns-with-magicdns/
      #
      # Please note that for the DNS configuration to have any effect,
      # clients must have the `--accept-dns=true` option enabled. This is the
      # default for the Tailscale client. This option is enabled by default
      # in the Tailscale client.
      #
      # Setting _any_ of the configuration and `--accept-dns=true` on the
      # clients will integrate with the DNS manager on the client or
      # overwrite /etc/resolv.conf.
      # https://tailscale.com/kb/1235/resolv-conf
      #
      # If you want stop Headscale from managing the DNS configuration
      # all the fields under `dns` should be set to empty values.
      dns:
        # Whether to use [MagicDNS](https://tailscale.com/kb/1081/magicdns/).
        # Only works if there is at least a nameserver defined.
        magic_dns: true
      
        # Defines the base domain to create the hostnames for MagicDNS.
        # This domain _must_ be different from the server_url domain.
        # `base_domain` must be a FQDN, without the trailing dot.
        # The FQDN of the hosts will be
        # `hostname.base_domain` (e.g., _myhost.example.com_).
        base_domain: null.domain.com
      
        # List of DNS servers to expose to clients.
        nameservers:
          global:
            - 1.1.1.1
            - 1.0.0.1
            - 2606:4700:4700::1111
            - 2606:4700:4700::1001
      
            # NextDNS (see https://tailscale.com/kb/1218/nextdns/).
            # "abc123" is example NextDNS ID, replace with yours.
            # - https://dns.nextdns.io/abc123
      
          # Split DNS (see https://tailscale.com/kb/1054/dns/),
          # a map of domains and which DNS server to use for each.
          split:
            {}
            # foo.bar.com:
            #   - 1.1.1.1
            # darp.headscale.net:
            #   - 1.1.1.1
            #   - 8.8.8.8
      
        # Set custom DNS search domains. With MagicDNS enabled,
        # your tailnet base_domain is always the first search domain.
        search_domains: []
      
        # Extra DNS records
        # so far only A-records are supported (on the tailscale side)
        # See https://github.com/juanfont/headscale/blob/main/docs/dns-records.md#Limitations
        extra_records:
          # Add your home server sub-domains here to resolve to the Headscale IP
          # - { name: "immich.home.easyselfhost.com", type: "A", value: "100.64.0.2" }
          # - {
          #     name: "paperless.home.easyselfhost.com",
          #     type: "A",
          #     value: "100.64.0.2",
          #   }
          # - {
          #     name: "authentik.home.easyselfhost.com",
          #     type: "A",
          #     value: "100.64.0.2",
          #   }
      
      # Unix socket used for the CLI to connect without authentication
      # Note: for production you will want to set this to something like:
      unix_socket: /var/run/headscale/headscale.sock
      unix_socket_permission: "0770"
      
      # Logtail configuration
      # Logtail is Tailscales logging and auditing infrastructure, it allows the control panel
      # to instruct tailscale nodes to log their activity to a remote server.
      logtail:
        # Enable logtail for this headscales clients.
        # As there is currently no support for overriding the log server in headscale, this is
        # disabled by default. Enabling this will make your clients send logs to Tailscale Inc.
        enabled: false
      
      # Enabling this option makes devices prefer a random port for WireGuard traffic over the
      # default static port 41641. This option is intended as a workaround for some buggy
      # firewall devices. See https://tailscale.com/kb/1181/firewalls/ for more information.
      randomize_client_port: false
      
      acls:
        # Enable ACLs
        - action: accept
          src: ["*"]
          dst: ["*:*"]
      
        # Enable exit node functionality
        - action: accept
          src: ["*"]
          dst: ["0.0.0.0/0:*"]
      
        # Enable subnet routing
        - action: accept
          src: ["*"]
          dst: ["*:*"]
          enableSubnetRouting: true
      ```

   3. Caddyfile:
      ```
      sub.domain.com {
          reverse_proxy localhost:8080
      }

      # Local access via IP address for the Web UI
      local_IP:80 {
          # UI route
          route /manager/* {
              reverse_proxy headscale-ui:80
          }
          # Default route for headscale
          route /* {
              reverse_proxy headscale:8080
          }
      }
      ```

   DIRECTORY STRUCTURE
   -----------------
   Create this directory structure:
   ```
   /root/headscale/
   ├── docker-compose.yaml
   ├── config.yaml
   ├── Caddyfile
   ├── data/           # Headscale data
   ```

   AUTOMATIC STARTUP
   ---------------
   To ensure Tailscale automatically reconnects with exit node settings after a reboot:

   1. Create systemd service:
      ```bash
      sudo nano /etc/systemd/system/tailscale-autoconnect.service
      ```

   2. Add configuration:
      ```ini
      [Unit]
      Description=Tailscale autoconnect
      After=network.target tailscaled.service
      Wants=tailscaled.service

      [Service]
      Type=oneshot
      RemainAfterExit=yes
      ExecStartPre=/bin/sleep 5
      ExecStart=/usr/bin/tailscale up --login-server=https://sub.domain.com --advertise-exit-node --advertise-routes=<your_local_subnet>/24,0.0.0.0/0,::/0
      Restart=on-failure
      RestartSec=5

      [Install]
      WantedBy=multi-user.target
      ```

   3. Enable service:
      ```bash
      sudo systemctl daemon-reload
      sudo systemctl enable tailscale-autoconnect
      sudo systemctl start tailscale-autoconnect
      ```

6. EXIT NODE SETUP
   -------------
   A. Start services:
      ```
      docker-compose up -d
      ```

   B. Connect Tailscale:
      ```
      sudo tailscale up --login-server=https://sub.domain.com \
        --advertise-exit-node \
        --advertise-routes=<your_local_subnet>/24,0.0.0.0/0,::/0
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
1. "Unable to modify read-only option: 'unprivileged'"<br>
   • Solution: Create new privileged container<br>
   • Cannot be modified after creation<br>

2. "Tailscale could not connect to DERP relay server"<br>
   • Check port forwarding: 3478/udp, 9443/tcp, 50443/tcp<br>
   • Verify DERP configuration in config.yaml<br>

3. "Cannot enable route: route not available"<br>
   • Advertise routes first with --advertise-routes<br>
   • Then enable routes with routes enable command<br>

4. "0.0.0.0/0 advertised without IPv6 counterpart" <br>
   • Include both IPv4 and IPv6 routes: <br>
   • Use: --advertise-routes=<your_local_subnet/24,0.0.0.0/0,::/0

IPHONE SETUP
-----------
1. Install Tailscale app
2. Log in with custom server: https://sub.domain.com
3. Get auth key:
   ```
   docker exec headscale headscale -u default preauthkeys create
   ```
4. Enable "Use VPN" in Tailscale settings

VERIFICATION
-----------
✓ Check whatismyip.com - Should show server's IP <br>
✓ Access local network (192.168.8.x) - Should work with VPN | Replace with your network <br>
✓ Verify routes: docker exec headscale headscale routes list
