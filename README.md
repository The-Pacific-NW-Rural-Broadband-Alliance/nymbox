# Nym-gateway 

-------------------------------------------------------------------------------
## Rough 2 min summary:

1. clone OpenWRT repo
1. add this repo to the feeds list in Openwrt/ ... like this: `src-git-full nym <url.git;develop>`  
1. run `./scripts/feeds update -a` and then `./scripts/feeds install -a`
1. in Openwrt/ - make menuconfig (meh)
1. set the toolchain and targets etc for Raspi 4, (brm27xx?)
1. `make`
1. `make package/feeds/nym/package/compile V=sc`

## OpenWRT images
[raspi links](https://openwrt.org/toh/raspberry_pi_foundation/raspberry_pi)
[raspi4-aarch64 v21.02.1](https://downloads.openwrt.org/releases/21.02.1/targets/bcm27xx/bcm2711/openwrt-21.02.1-bcm27xx-bcm2711-rpi-4-squashfs-sysupgrade.img.gz)
[friendlyWRT v4.41](https://drive.google.com/file/d/1zfLZXoDbTVSzXsdRN1BRPUho8uxpjNzf/view?usp=sharing)

For a 32-bit OpenWRT compatible router - you will need to find the binaries needed. 

Our main goal here, however; is to use a heavily modified Raspberry Pi CM4 64-bit running a custom OpenWRT image.

## Building the whole thing 

### Dependencies for cross-compiling on x86 or ARM64(M1)

```
Rust & Cargo >= 1.56.0
gcc-aarch64-linux-gnu
cross
rustup >= 1.24.3
pkg-config 
build-essential
libssl-dev
```

### clone Nym and checkout the correct tag
```
git clone https://github.com/nymtech/nym.git
cd nym
git checkout tags/v0.12.1
```

### Set up a toolchain for compiling Rust with aarch64-musl target (openwrt-64bit)

#### On x86-linux (hope it works with Arch too :P tested on Pop!OS 21.04)
1. Get [Cross](https://github.com/cross-rs/cross) 
    - `cargo install cross`
    - or refer to the official docs in the link above
2. edit `nym/gateway/Cargo.toml` with the following:

```
[dependencies]
openssl-sys = { version = "0.9.72", optional = true, features = ["vendored"] }
openssl = { version = "0.10.37", optional = true, features = ["vendored"] }
```
and also add this line to `[features]`
```
vendored-openssl = ["openssl/vendored"]
```
3. Add targets to your toolchain 

For a 64-bit which we will be using in the future and do the most work on:
```
rustup target add aarch64-unknown-linux-musl
```
For a 32-bit version you will need to add another toolchain:
```
rustup target add armv7-unknown-linux-musleabihf

```

Check if you have the toolchains installed

```
rustup target list | grep installed
aarch64-unknown-linux-gnu (installed)
x86_64-unknown-linux-gnu (installed)
```

4. From nym root folder run the following command:

```
cross build --bin nym-gateway --release --target aarch64-unknown-linux-musl  --features vendored-openssl
```

5. Check for the resulting file:
```
~/test/nym/target on (HEAD detached at v0.12.1)*
$ file aarch64-unknown-linux-musl/release/nym-gateway
aarch64-unknown-linux-musl/release/nym-gateway: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

#### You can even cross-compile this on Macbook M1 or other ARM64 M1 chip
For more targets see other images [here](https://github.com/messense/rust-musl-cross)

```
docker pull messense/rust-musl-cross:aarch64-musl
```

create an alias for convienience 
```
alias rust-musl-builder='docker run --rm -it -v "$(pwd)":/home/rust/src messense/rust-musl-cross:aarch64-musl'
```
then build it like this:
```
rust-musl-builder cargo build --target=aarch64-unknown-linux-musl --features vendored-openssl --bin nym-gateway
```
If the build fails, please refer to the docs of the [repo I used for this](https://github.com/messense/rust-musl-cross)

Most likely you are missing some dependencies or something else on your Mac M1. But that is out of scope of this documentation.


### A note about potential OpenSSL issues with cross-compiling to musl distros (OpenWRT/busybox are not GNU but Musl and there is a workaround to make this build not fail) 

In the build section above we are using the code below. This is a well known issue, see it here on [Github](https://github.com/cross-rs/cross/issues/229#issuecomment-597898074) 

```
[dependencies]
openssl-sys = { version = "0.9.72", optional = true, features = ["vendored"] }
openssl = { version = "0.10.37", optional = true, features = ["vendored"] }
```
and also add this line to `[features]`
```
vendored-openssl = ["openssl/vendored"]
```
After these edits, run the `rust-musl-builder` container again but add an extra option `--features vendored-openssl`
This worked successfully on my Macbook Air M1. 

### Flash the OpenWRT image you downloaded in the first step and ssh to the box
Here’s a simple guide [how you flash OpenWRT onto an SD card:](https://www.how2shout.com/how-to/balenaetcher-how-to-create-a-bootable-usb-flash-drive-using-etcher.html)


**Note:** The current OpenWRT raspi4 64-bit image defaults to `192.168.1.1` so if that overlaps with your current LAN you will cause a duplicate IP / DHCP conflict.


### Run the gateway

1. ssh to your router and scp the binaries located in 
`nym/target/aarch64-unknown-linux-musl/release/nym-gateway`

```
./nym-gateway init --host <local-ip> --announce-host <public-ip> --id pirouter --wallet-address nymt1h9qck0e8p0eeyjz9wkuwqgx0f5svjjxfz6zarg
```


2. then you can run the gateway as following: `./nym-* run --id pirouter` 
3. Bond the gateway... 

In general, for the other steps beyond the scope of this project, please, refer to the [Nym docs](https://nymtech.net/docs)

### Pre-built NYMBox Images

Requirements:
8GB SD card
Raspberry Pi 4 / Raspberry Pi 400 / Raspberry Pi CM4
or
FriendlyELEC R2S / R2C / R4S / R5S

[NYMbox Raspberry Pi 4 - 64bit](https://nwbroadbandalliance.org/images/openwrt-22.03.0-rc4-bcm27xx-bcm2711-rpi-4-nymbox.img.tar.gz)
[NYMbox rk3328](https://nwbroadbandalliance.org/images/friendlywrt-rk3328-20220125-NYMbox.img.tar.gz)





