# Easy Tailscale VPN Server (Flatcar Container Linux Config)

**Ignition/Butane configuration file to deploy a firewalled Tailscale exit node as a personal VPN.**
- Designed for use with Flatcar Container OS (may also work with Fedora CoreOS). 
- Automatic Tailscale updates (via Watchtower)
- Automatic OS updates (via Flatcar)
- Using Vultr's high frequency tier of VPS, you can achieve ~1Gb/s VPN speeds for $6/m with 1TB of bandwidth.
- Disables most of Tailscale's logging features for security

**Configuration Tool**: https://jakelmg.github.io/tailscale-flatcar-config-tool/

## Instructions (using the [configuration tool](https://jakelmg.github.io/tailscale-flatcar-config-tool/))
1. **Create a free [Tailscale account](https://login.tailscale.com/start)**
2. **Go to [Settings>Keys](https://login.tailscale.com/admin/settings/keys) and generate an auth key.**
3. **Open the [configuration tool](https://jakelmg.github.io/tailscale-flatcar-config-tool/), paste in the auth key you just generated, and click "Generate Config" or "Copy" in the preview window.**
   - Optionally:
     - Enable SSH access and enter an SSH public key
     - Select your timezone for automatic update reboot timing / logs
     - Adjust the automatic update reboot time window
4. **Create a VPS on [Vultr](https://lmg.gg/vultr-gh)** (affiliate link)
   1. Select the location you want to deploy your VPN server to.
   2. Select "Shared CPU" as the server "Type" (the other types may also work but are much more expensive)
   3. Disable "Automatic Backups" (they cost money, and we aren't storing data on this VPS anyways)
   4. Select "High Frequency" under Plans, and then "vhf-1c-1gb"
        - This is the cheapest high frequency plan, and includes 1 shared CPU core, 1GB of RAM, and 1TB of bandwidth for $6/month. It can handle around 1Gb/s VPN speeds
        - Other tiers seemed to have worst performance - but if you have a less than 1Gb/s internet connection, that might not matter, and a cheaper plan might be a better value.
   5. On the "Step 2" page, select the "stable x64" version of Flatcar Container Linux
   6. Paste the config we generated earlier into the "Ignition Configuration" box
   7. Click "Deploy"
5. **Once `vpn-server` appears in the Tailscale "[Machines](https://login.tailscale.com/admin/machines)" page, approve the device as an exit node.**
   1. Click the "..." icon next to the "vpn-server" machine, then "Edit route settings..."
   2. Check the "Use as exit node" box.
   3. Click "Save"
6. **[Download Tailscale](https://tailscale.com/download) and install it on the device you want to tunnel through your new VPN.**
7. **Login into the Tailscale client with the same account you used when generating the auth key.**
8. **Open cmd/terminal and run `ping vpn-server` to make sure you can ping your vpn server over Tailscale.**
9. **Run `tailscale status` (mac instructions [here](https://tailscale.com/kb/1080/cli?tab=macos)) and make sure the connection shows "Direct" and not relayed.**
    - In most cases, it shouldn't be possible for the connection to relay. If that happens, you may be dealing with an extremely restrictive or non standard firewall on the client.
10. **Once you've verified you have a working direct connection, select your VPN server as your exit node. This will start tunneling your traffic through your new DIY VPN server :D**
    1. Right click on Tailscale in your device tray > Exit nodes > Select "`vpn-server`"
11. **Check to see if your external IP address has changed with a service like [ipinfo.io](https://ipinfo.io/)**
    - "org" should show "Vultr" or similar
    - city/region/count should match the location you selected when creating the VPS on Vultr
    - "ip" should be different than your non-VPN'd IP (you can disconnect from Tailscale to compare)

## Manual Instructions
1. **Create a free [Tailscale account](https://login.tailscale.com/start)**
2. **Go to [Settings>Keys](https://login.tailscale.com/admin/settings/keys) and generate an auth key.**
3. **Download the human-readable Butane config: <a id="raw-url" href="https://raw.githubusercontent.com/jakelmg/easy-tailscale-server/refs/heads/main/human-config.yaml">human-config.yaml</a>**
    1. Replace `ENTER_TAILSCALE_AUTH_KEY_HERE` with the auth key you generated in step 2.
    2. Optionally:
        - Enable SSH by adding a firewall allow rule for port 22/tcp
          ```
          - path: /var/lib/iptables/rules-save
            mode: 0644
            contents:
              inline: |
                *filter
                :INPUT DROP [0:0]
                :FORWARD DROP [0:0]
                :OUTPUT ACCEPT [0:0]
                -A INPUT -i lo -j ACCEPT
                -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
                -A INPUT -p udp -m udp --dport 41641 -j ACCEPT
                -A INPUT -p icmp -m icmp --icmp-type 0 -j ACCEPT
                -A INPUT -p icmp -m icmp --icmp-type 3 -j ACCEPT
                -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
                -A INPUT -p icmp -m icmp --icmp-type 11 -j ACCEPT
                -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT          <--- Add this line to open port 22 for SSH access
                -A FORWARD -i eth0 -o tailscale0 -j ACCEPT
                -A FORWARD -i tailscale0 -o eth0 -j ACCEPT
                COMMIT
          ```
        - Add an SSH key in Butane format for the default `core` user.
          ```
          passwd:
            users:
              - name: core
                ssh_authorized_keys:
                  - "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGdByTgSVHq......."   <--- Replace with your SSH public key
          ```
        - Enter your timezone
          ```
          links:
            - path: /etc/localtime
              overwrite: true
              target: /usr/share/zoneinfo/Etc/UTC   <--- Replace "Etc/UTC" with your desired timezone
          ```
        - Adjust the automatic update reboot time window
          ```
          - path: /etc/flatcar/update.conf
            overwrite: true
            contents:
              inline: |
                REBOOT_STRATEGY=reboot
                LOCKSMITHD_REBOOT_WINDOW_START=02:00   <--- Replace "02:00" with your desired start time for the automatic update reboot time window
                LOCKSMITHD_REBOOT_WINDOW_LENGTH=1h     <--- Replace "1h" with your desired reboot time window length
            mode: 0420
          ```

4. **Install Butane, which you'll need to convert the human-readable Butane config to a system-readable Ignition config:**
    - Windows (in cmd)
        ```
        winget install butane
        ```
    - MacOS (in terminal, requires [Homebrew](https://brew.sh/)):
        ```
        brew install butane
        ```
    - Debian/Ubuntu Linux:
        ```
        sudo apt install butane
        ```
5. **Convert the human-readable Butane config to a system-readable Ignition config:**
    ```
    butane --pretty --strict human-config.yaml --output=system-config.ign
    ```
6. **Create a VPS on [Vultr](https://lmg.gg/vultr-gh)** (affiliate link)
   1. Select the location you want to deploy your VPN server to.
   2. Select "Shared CPU" as the server "Type" (the other types may also work but are much more expensive)
   3. Disable "Automatic Backups" (they cost money, and we aren't storing data on this VPS anyways)
   4. Select "High Frequency" under Plans, and then "vhf-1c-1gb"
        - This is the cheapest high frequency plan, and includes 1 shared CPU core, 1GB of RAM, and 1TB of bandwidth for $6/month. It can handle around 1Gb/s VPN speeds
        - Other tiers seemed to have worst performance - but if you have a less than 1Gb/s internet connection, that might not matter, and a cheaper plan might be a better value.
   5. On the "Step 2" page, select the "stable x64" version of Flatcar Container Linux
   6. Paste the config we generated earlier into the "Ignition Configuration" box
   7. Click "Deploy"
7. **Once vpn-server appears in the Tailscale "[Machines](https://login.tailscale.com/admin/machines)" page, approve the device as an exit node.**
   1. Click the "..." icon next to the "vpn-server" machine, then "Edit route settings..."
   2. Check the "Use as exit node" box.
   3. Click "Save"
8. **[Download Tailscale](https://tailscale.com/download) and install it on the device you want to tunnel through your new VPN.**
9. **Login into the Tailscale client with the same account you used when generating the auth key.**
10. **Open cmd/terminal and run `ping vpn-server` to make sure you can ping your vpn server over Tailscale.**
11. **Run `tailscale status` (mac instructions [here](https://tailscale.com/kb/1080/cli?tab=macos)) and make sure the connection shows "Direct" and not relayed.**
    - In most cases, it shouldn't be possible for the connection to relay. If that happens, you may be dealing with an extremely restrictive or non standard firewall on the client.
12. **Once you've verified you have a working direct connection, select your VPN server as your exit node. This will start tunneling your traffic through your new DIY VPN server :D**
    1. Right click on Tailscale in your device tray > Exit nodes > Select "`vpn-server`"
13. **Check to see if your external IP address has changed with a service like [ipinfo.io](https://ipinfo.io/)**
    - "org" should show "Vultr" or similar
    - city/region/count should match the location you selected when creating the VPS on Vultr
    - "ip" should be different than your non-VPN'd IP (you can disconnect from Tailscale to compare)

<br>

## Deploying Tailscale/Flatcar on Other Cloud Providers
- Digital Ocean (easy & tested)
  - Upload Flatcar's official DigitalOcean image as a Custom Image
    - [Instructions](https://docs.digitalocean.com/products/custom-images/getting-started/quickstart/)
    - [Image](https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_digitalocean_image.bin.bz2)

- OVH VPS (more difficult & untested)
    - Use Flatcar's community supported OpenStack (OVH compatible) image
      - [Instructions](https://www.flatcar.org/docs/latest/installing/community-platforms/ovhcloud/)
      - [Image](https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_openstack_image.img)