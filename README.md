# macos-dev-setup

A setup script for macOS to make local web development easier by allowing simple management of DNS
and trusted TLS without requiring a password or any interaction after the initial setup.

This means you can set up your development servers, commonly hosted at URLs like
`http://localhost:3000`, with nicer URLs like `https://example.foo`. You also avoid pesky "insecure"
warnings when trying to use untrusted self-signed certificates, and issues you might accidentally
run into where browsers treat localhost or insecure protocols specially. If you don't know what this
means, be thankful, and try to avoid it by always using HTTPS.

Services like [sslip.io](sslip.io) cover the DNS portion somewhat without needing to configure
anything, but these still require internet access and still require knowing the IP address assigned.

While this setup does take some knoweldge to use effectively, and there are no GUI tools involved,
having control over DNS and TLS can be a powerful tool. This script is just an example, you are free
to take this script and modify it to you or your team's liking. Having a common system setup like
this means your team can automate processes requiring DNS and TLS without having to reinvent the
wheel every time.

## Setup

Simply run the script:

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/jonesinator/macos-dev-setup/refs/heads/main/macos-dev-setup)"
```

You will be prompted for your password and to allow administrative actions several times. See
comments in the [macos-dev-setup](./macos-dev-setup) script for details on what it's doing.

Fork this repository if you want to make changes to the script.

## Updating DNS

The DNS server installed by the setup script is
[unbound](https://www.nlnetlabs.nl/projects/unbound/about/).

If you want to customize DNS responses, you can do so by editing unbound configuration files at
`~/.local/share/dns/*.conf`. See Unbound's documentation for more details. You can also edit the
top-level configuration file at `/opt/homebrew/etc/unbound/unbound.conf`, but that requires
superuser access and as such is more difficult to automate. Anything you can do in the toplevel file
can be delegated to an included file, though, so this is not a real limitation.

Once you have edited the config files and want to make the changes live, run the following commands
to restart unbound, and flush the DNS cache:

```sh
sudo brew services restart unbound
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

While this command does require `sudo`, it should not require entering a password, because the file
at `/etc/sudoers.d/unbound` permits it.

### Example

For example, let's say you wanted the `foo` TLD and all subdomains to point to `10.10.10.10`, while
the `bar` TLD and all subdomains point to `10.11.11.11`. You can split the following configuration
lines into files however you want, but it makes the most sense to have one configuration file for
both domains, or two configuration files, one for each domain. This example takes the latter
approach.

First, create the two unbound config files.

`~/.local/share/dns/foo.conf`

```
local-zone: "foo." redirect
local-data: "foo. IN A 10.10.10.10"
```

`~/.local/share/dns/bar.conf`

```
local-zone: "bar." redirect
local-data: "bar. IN A 10.11.11.11"
```

Once the files have been created run this command to restart the unbound server:

```sh
sudo brew services restart unbound
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

Now you should be able to run the following commands and see the following responses.

```sh
dig +short example.foo
10.10.10.10
```

```sh
dig +short wow.bar
10.11.11.11
```

To undo these changes, run these commands:

```sh
rm ~/.local/share/dns/foo.conf ~/.local/share/dns/bar.conf
sudo brew services restart unbound
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

## Generating Certificates

Once you're able to steer DNS responses to the IP addresses you desire, it's useful to be able to
use TLS to protect connections to those IPs. While [cfssl](https://github.com/cloudflare/cfssl) was
used to generate the root certificate, and was installed as part of the setup script, you don't have
to use it, and anything that works with certificates and keys can work with the files in
`~/.local/share/ca`. However, everything is already set up with cfssl, and it's fairly easy to work
with.

### Example

This is just a minimal example. See the cfssl documentation and code for more details on what it is
capable of.

The following very ugly command will generate `foo.pem` and `foo-key.pem`, the a key and certificate
for the host `example.foo`.

```sh
echo '{"CN":"example.foo","hosts":["example.foo"],"key":{"algo":"ecdsa","curve":"P-256"}}' \
  | cfssl gencert -ca ~/.local/share/ca/root.pem -ca-key ~/.local/share/ca/root-key.pem \
      -config ~/.local/share/ca/ca-config.json -profile server - | cfssljson -bare foo
```

While that is sufficient for this example, you may need to construct the full chain, like this:

```sh
cat foo.pem ~/.local/share/ca/root.pem > chain.pem
```

Now `chain.pem` and `foo-key.pem` should be compatible with any TLS server that can be configured to
identify as the host `example.foo`.

To undo these changes, remove the files:

```sh
rm chain.pem foo.pem foo-key.pem
```

## Testing

This can be tested easily using [UTM](https://mac.getutm.app/). Create a new macOS virtual machine,
open the Terminal app, and run the setup command mentioned earlier in the readme. This is pretty
useful because you can make a fresh macOS install on UTM, clone it, and then run your setup script
repeatedly until you get it the way you desire.
