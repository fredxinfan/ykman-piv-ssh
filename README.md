# Generate SSH key with YubiKey Manager CLI using PIV

## YubiKey General information

The default PIN code is `123456`. The default PUK code is `12345678`.

The default 3DES management key (9B) is `010203040506070801020304050607080102030405060708`.

## Installation

Ubuntu

```
$ sudo apt-add-repository ppa:yubico/stable
$ sudo apt update
$ sudo apt install yubikey-manager
```

MacOS

```
$ brew install ykman
```

For more ways to install `ykman`, please refer to official documentation [here](https://developers.yubico.com/yubikey-manager/).

## Steps

1. Change YubiKey device PIN, PUK and Management Key if they're still using default ones. (Optional, but highly recommended for security reasons)

  ```
  $ ykman piv change-pin
  ```
  
  Enter current PIN first, if it's the default, enter `123456`, then enter the new PIN twice to save it.
  
  ```
  $ ykman piv change-puk
  ```
  
  Enter current PUK first, if it's the default, enter `12345678`, then enter the new PUK twice to save it.
  
  ```
  $ ykman piv change-management-key
  ```
  
  Enter current Management Key first, if it's the default, enter `010203040506070801020304050607080102030405060708`, then enter the new Management Key twice to save it.
  
  For more information about how are they being used, please refer to official documentation [here](https://developers.yubico.com/yubikey-piv-manager/PIN_and_Management_Key.html).

2. Generate a key in slot 9a: 

  ```
  $ ykman piv generate-key 9a public.pem
  ```

  For full slots reference, please refer to official documentation [here](https://developers.yubico.com/PIV/Introduction/Certificate_slots.html).

3. Create a self-signed certificate for that key: 

  ```
  $ ykman piv generate-certificate -s "/CN=SSH-key/" 9a public.pem
  ```

  If there is no error showing up, the generated certificate has already been imported to the YubiKey device automatically.
  No need to manually import certificate back like the `yubico-piv-tool` is doing [here](https://developers.yubico.com/PIV/Guides/SSH_with_PIV_and_PKCS11.html).

4. Find out where OpenSC has installed the pkcs11 module.
  - For OS X with binary installation this is typically in `/Library/OpenSC/lib/`. Homebrew users can use `export OPENSC_LIBS=$(brew --prefix opensc)/lib`.
  - For a Debian based system this is typically in `/usr/lib/x86_64-linux-gnu/`.
    After this we’ll call this location `$OPENSC_LIBS`.
    
5. Export the public key in correct format for ssh and once you got it, add it to authorized_keys on the target system.

  ```
  $ ssh-keygen -D $OPENSC_LIBS/opensc-pkcs11.so -e
  ```

6. Authenticate to the target system using the new key:

  ```
  $ ssh -I $OPENSC_LIBS/opensc-pkcs11.so user@remote.example.com
  ```
  
7. Setup YubiKey to work with ssh-agent:

  ```
  $ ssh-add -s $OPENSC_LIBS/opensc-pkcs11.so
  ```
  
  Note:
  > Since MacOS Sierra 10.12.4, `agent refused operation` error maybe showing for pkcs11 module.
  > This is because `opensc-pkcs11.so` is not installed in a "whitelisted" location for the new version of OpenSSH, e.g. `/usr/local/opt/opensc/lib`.
  > The easiest way to fix this is to copy/move `opensc-pkcs11.so` to a "whitelisted" location, e.g. `/usr/local/lib`.
  > Symbolic link will not work in this scenario, `opensc-pkcs11.so` has to be copied/moved into a "whitelisted" location.
  > The issue is discussed [here](https://github.com/OpenSC/OpenSC/issues/1007).
  
  To confirm that the ssh-agent correctly finds that key and getting the public key in correct format:
  
  ```
  $ ssh-add -L
  ```
  
8. Alternatively, selectively add the PKCS11Provider to `~/.ssh/config`:
  
  ```
  Host remote.example.com
      PKCS11Provider /usr/local/lib/opensc-pkcs11.so
      Port 22
      User user
  ```
