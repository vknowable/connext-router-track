# Connext Router Track Security Advice

Here are a few security tips if you are planning to run a Router on Connext:

## Docker Overrides Iptables (UFW)

By default, Docker overrides any iptables (UFW) rules. If you're using the Docker Router image and your VM has a direct connection to the Internet, make sure you're not accidentally exposing your Router API endpoint externally.

**Example:**  
If we have UFW on our Router set to block `ROUTER_EXTERNAL_PORT` (port **8000** by default), we would expect the following to fail from an external machine:  

`curl http://x.x.x.x:8000/config`  

Instead, we receive a response indicating that the port is open.  

`{"signerAddress":"0x26Ad85....."}`

One simple solution to edit the `docker-compose.yml` file to bind the exposed port to **localhost**, by changing:

```
...
  ports:
    - $ROUTER_EXTERNAL_PORT:8080
...
```
to:
```
...
  ports:
    - 127.0.0.1:$ROUTER_EXTERNAL_PORT:8080
...
```

Restart the Docker-Compose stack after making the change. The endpoint should now only be available from the machine running the Router -- try again from an external machine to make sure the change was successful.
***
## Admin Token Best Practices

***[To verify: is REST API still implemented in Amarok? Doc page seems to have been removed]***

Each Router has an **Admin Token**, which is a string chosen by the operator and set in the `config.json` file.

The Admin Token is used to authenticate requests made to the Router's REST API endpoint and must be kept secret.

If your Router's API endpoint is ever exposed to the public and your Admin Token is compromised/vulnerable to brute forcing, someone could use your token perform unauthorized operations with your Router.

Use a sufficiently long token (50 characters or more) to protect against brute force attacks. You can easily generate a secure token using `pwgen` with the following command:

`pwgen -s 50 1`

#### Protecting the admin token:

The Admin Token is stored in plaintext in the Router's `config.json` file. To avoid this, below is a method to load your **config.json** into **tmpfs** before starting the Router, and unmount it after the Router is started. Using this method will require you to generate and move your configuration into **tmpfs** on each Router restart. The old contents will be lost -- so don't forget to backup any important information first (eg: your wallet mnemonic!)

1. Create **tmpfs**:  
`mount -t tmpfs -o size=100m tmpfs /mnt/tmpfs`

2. Move config file to **tmpfs**:  
`mv config.json /mnt/tmpfs/config.json`

3. Change the volume point in docker-compose (use type [bind](https://github.com/docker/compose/issues/2781#issuecomment-441653347)):  
`- /mnt/tmpfs/config.json:/home/node/Router/config.json`

4. Run `docker-compose`

5. Finally, unmount the **tmpfs** dir. After this step all data in **/mnt/tmpfs/** will be lost:  
`umount /mnt/tmpfs`


***
## Setting Recipient and Owner Addresses

In addition to its own signing address, each Router has a **Recipient** and **Owner** address set in the respective `Connext Handler` contracts of each chain.

- `Recipient`: Whenever liquidity is removed from the Router, the funds will be sent to this address. If an attacker were to compromise your Router, they would at best be able to withdraw your funds to the **Recipient** address (which you hopefully still control). We advise using a hardware wallet for the recipient address.

- `Owner`: Only the **Owner** has the ability to change the Recipient and Owner addresses.

The Recipient and Owner addresses can be changed by calling the corresponding write methods on the Connext Handler contract from the **Owner** address (`setRouterRecipient` and `proposeRouterOwner`/`acceptProposedRouterOwner`). This needs to be done on each chain supported.

It's **highly** recommended that you set your Recipient and Owner addresses to something different from your Router's signing address. You should also use a hardware wallet. Since the Recipient and Owner private keys are not accessed by or stored on the Router, simply compromising your Router would not be enough for an attacker to access your liquidity -- they would also need one or both of these keys.

Please be aware that each Router's Recipient and Owner addresses can be publicly queried from the Connext Handler contracts.


***
## Protecting Your Router's Private Key

**A little more research needed: any specific problems (eg: griefing, double spend) either for individual operator or network/users that result from a compromised Router colluding with a user? Or is it just best practices?**

Avoid operating your Router with your private key or mnemonic stored in plaintext. While it's possible to use a mnemonic in `config.json` or stored unencrypted in a `key.yml` file, these should be considered for testing purposes only.

Instead, use one of the supported [Web3Signer methods](https://docs.web3signer.consensys.net/en/latest/HowTo/Use-Signing-Keys/). Using an external KMS that explicitly whitelists Web3Signer will allow you to move your private key out of plaintext and off your Router server entirely.

Web3Signer supports a variety of external key vaults and HSMs. Consult the [official docs](https://docs.web3signer.consensys.net/en/latest/Reference/Key-Configuration-Files/) to get started, but if you need help getting up and running with a specific method, please feel free to reach out on our Discord... chances are we can connect you with another operator who's already gone through the setup process!


Whenever you are joining a crypto project we advise that you should use a brand new wallet each time.
You can generate a private key using the following command:  
`openssl rand -hex 32 > private_key.json`


***
## Secure Cloud Architecture

Rather than allowing a direct connection from your Router to the Internet, a much more secure setup is to place your sensitive components (eg: Router, Web3Signer) in a private subnet. Create a bastion server (aka: jump server) to access the subnet, and a NAT to allow the Router to access the Internet.

Many cloud providers have built-in support for features like private networks and Cloud NATs to make this easier to achieve, for example:  
*AWS docs:* [Private Instances](https://aws.amazon.com/vpc/), [Cloud NAT](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)  
*GCP docs:* [Private VPC](https://cloud.google.com/data-fusion/docs/how-to/create-private-ip), [Cloud NAT](https://cloud.google.com/nat/docs/overview)

#### Private network deep dive:
![ALT text](schema/nat.png)

As you can see in the diagram above, your Router doesn't expose an external IP address and can't be accessed directly. SSH access to the Router (or any other VMs in the private subnet) can only be done by connecting through the bastion. The Router accesses the Internet through NAT, using it as gateway.

Using this configuration is less prone to attacks that would compromise your Router host.

#### Connecting to the bastion securely:
How you use the bastion to connect to the private subnet is important.  

*Wrong: Connecting with an intermediate key pair*  
Don't under any circumstances do this:  
```
# from client, using first key pair
ssh [bastion external IP]

# from bastion shell, using second key pair
ssh [Router internal IP]
```
In the above example, we're storing the private SSH key to the Router on the bastion to complete the second stage of the connection. This is a very poor setup -- the bastion is our most exposed point and the last thing we want to do is store anything sensitive on it.

*Wrong: Using agent forwarding*  
You may have seen this solution used and/or recommended:
```
# from client
ssh-add [path to key]
ssh -A [bastion external IP]

# from bastion shell, now using client's ssh agent
ssh [Router internal IP]
```
In this example, we store the SSH key for Router access in the client's SSH-agent. SSH-agent is a helper process that stores keys in memory for later use by the main SSH process. Then we use the `-A` flag to pass a reference to the client SSH-agent (containing the key) to the bastion, to use as if it was its own when connecting to the Router. This is better than the first example, since the bastion is not storing or directly accessing any private keys. But there's still a potential security risk: if someone gains root access to the bastion, they can read the reference to the client SSH-agent, hijack it for their own purposes, and have full access to the Router.  

*Right: Use TCP forwarding (ProxyJump)*  
A better approach is to SSH from the client to the bastion and then establish TCP forwarding to the Router. Thankfully, recent versions of OpenSSH have built-in support to make this easy for us:
```
# from client
ssh -J [bastion external IP] [Router internal IP]
```
The client authenticates both hops to the Router, so the bastion does not need to store or have even indirect access to any private keys.

#### Hardening the bastion
Try an experiment: create a new VM on any cloud provider and leave it directly accessible to the Internet for a few hours. Check the authorization logs with `cat /var/log/auth.log` -- even in such a short time, you'll probably see dozens of unauthorized login attempts from bots. Mostly, they are targeting weak passwords and publicly known exploits, hoping to get lucky. Don't be the person who neglected these basic steps, letting yourself get burned by something that could have easily been prevented!
- To reduce the attack surface, your bastion should have one function only: forwarding you into your private subnet. Disable any unnecessary services and only allow ingress on the SSH port/egress to the subnet. You can also disallow interactive shell access on the bastion.
- Apply the general SSH hardening tips from the section below
- Use rate-limiting (eg: fail2ban) to time out an offending IP address after a certain number of failed login attempts
- Use 2-factor authentication for SSH authentication (see section below)
- By default, unattended-upgrades will not automatically reboot even if some updates require it. You can change this in the config file `/etc/apt/apt.conf.d/50unattended-upgrades` by setting **Automatic-Reboot** to true. You can also change **Automatic-Reboot-Time** to schedule the reboots for a certain time. You will also have to install the package *update-notifier-common* (if it's not already installed)

#### Teleport
As an alternative to OpenSSH, a great tool to directly access your host machine is [Teleport](https://goteleport.com/), which adds features like a web UI and certificate based authentication.

## SSH Hardening
Consider making these changes in the SSH server config file, `/etc/ssh/sshd_config`. Don't forget to restart the SSH service for the changes to take effect.  
- Change the SSH port from the default of 22 to any other port (for example 9922) -- there are a lot of bots scraping on the Internet for port 22
- Disable SSH root login
- Disable password login, and disallow empty passwords (*PermitEmptyPasswords no*)
- Explicitly whitelist users for SSH access
- Set *AllowAgentForwarding*, *AllowStreamLocalForwarding*, and *X11Forwarding* to *no*
- Check that *IgnoreRhosts* is set to *yes* and *HostbasedAuthentication* is set to *no*

You may wish to use iptables (UFW) to restrict SSH access to only allow your public IP. (Just be careful not to lock yourself out of your VM!)

## SSH Key Best Practices
Key based authentication is considered much more secure than using a password, and should be used whenever possible. SSH keys help protect you against brute force attacks, and using public-key encryption is safer than sending passwords across the network.

#### Choosing a key type:
Ed25519 keys are recommended over RSA, since they offer better security and performance.  
`ssh-keygen -t ed25519 -a 100`  

If you still want to use RSA keys (eg: for compatibility reasons), use a minimum length of 4096 bits.  
`ssh-keygen -b 4096 -o -a 100`  

*-o specifies OpenSSH format, already implied for Ed25519*  
*-a 100 specifies 100 rounds of key derivation*  

When creating the key pair, protect it with a passphrase that's at least 15-20 characters long.

## 2-Factor Auth for SSH
2-Factor Auth gives a big boost in security for relatively little effort -- you can set it up in less than 15 minutes.
- Google Authentication Module is a popular choice. It links with the Authenticator App on your mobile device, and requires a 6 digit code from the app along with your SSH key. The code is only valid once and changed every 30 seconds.
- A more secure option is U2F with a physical token, for example a Yubikey that authenticates via USB or NFC
