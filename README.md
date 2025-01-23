# easy-cloud-vpn

Ignition/Butane configuration file to deploy a firewalled Tailscale exit node as a personal VPN. 
- Designed for use with Flatcar Container OS (may also work with Fedora CoreOS). 
- Automatic Tailscale updates (via Watchtower)
- Automatic OS updates (via Flatcar)
- Using Vultr's high frequency tier of VPS, you can achieve 1Gb/s VPN speeds for $6/m with 1TB of bandwidth.

### Instructions
1. Create a free tailscale account
2. Generate an auth key in your tailscale account
3. Input the auth key into the system-config.ign
4. Create a VPS on Vultr (or other cloud platform, haven't tested others) with Flatcar stable. Paste system-config.ign into User Data when creating (or other platform).
5. Approve the device as an exit node once it shows up on the Tailscale "Machines" page.
6. Download Tailscale on the device you want to tunnel through your new VPN. Login with the same account you used to generate the auth key. You should now be connected to your Tailscale network (tailnet)
7. Open cmd/terminal and run `ping vpn-server` to make sure you can ping your server via Tailscale.
8. Run `tailscale status` and make sure the connection shows "Direct" and not relayed. In 99% of situations, it shouldn't be possible for the connection to relay. If that happens, you may be dealing with an extremely restrictive or non standard firewall on the client.
9. Once you've verified you have a working direct connection, right click on Tailscale in your device tray > Edit nodes > Select "`vpn-server`" to start tunneling your traffic through your new DIY VPN server :D


### Making Edits to an Existing Instance
1. Run `flatcar-reset`
2. Reboot (this will create a new device in Tailscale, which will require reapproving the exit node)
3. Remove the old device from the Tailscale "Machines" page.


### Other Info
- If you want to customize the configuration more, you can edit human-config.yaml and then convert it to a machine readable version with Butane (butane can be obtained via winget on windows, then use `buntane --pretty --strict human-config.yaml --output=system-config.ign`)
- If you want to be able to SSH into the machine, use system-config-ssh.ign (or human-config if you're editing other things)