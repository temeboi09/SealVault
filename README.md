# SealVault

A console-only, single-binary encrypted vault for arbitrary files. No GUI,
no scripting engine, no plugin system, no network access — the entire
attack surface is: parse a menu choice, read a password, move encrypted
bytes.

## Security model

**What SealVault does:**
- Encrypts file contents, filenames, and sizes — anyone with access to
  the vault sees only random-looking blobs, no plaintext metadata.
- Authenticates everything it encrypts. Any tampering (a flipped bit,
  truncation, reordering of pieces) causes decryption to fail loudly
  rather than returning corrupted data silently.
- Treats every stored item as opaque data, always. SealVault never
  executes, opens, or inspects the contents of anything you add to it.
- Streams both encryption and decryption in small fixed-size pieces, so
  sealing or extracting a huge file uses a small, constant amount of
  memory rather than needing the whole file loaded at once.
- Writes every change to the vault atomically: a change is fully staged
  before it replaces anything on disk. A crash or power loss mid-operation
  leaves the vault in its previous consistent state, never a half-written
  one.
- Clears key material from memory when the vault is locked (and on exit),
  and takes best-effort steps to keep that key material out of swap space
  while the vault is open.

**What SealVault does not do:**
- It is not antivirus and makes no claim of "isolating" you from malware.
  While sealed, a file inside the vault can't be read or run by anything
  without your password. The moment you extract a file, it's a normal
  file again, and your system will run it if you tell it to — same as
  unzipping an archive. SealVault's job ends at "encrypted at rest,
  verified on read."
- It does not hide the *existence* or *size* of the vault, the number of
  files in it, or their approximate encrypted sizes.
- It is not a substitute for full-disk encryption, secure boot, or
  system-level hardening — it protects the files you put in it, not the
  machine it runs on.

## Cryptographic design

Every vault is created with one random master key, generated once and
never derived from your password. Your password only ever wraps and
unwraps that master key — it never touches your files directly. This
means changing your password later is cheap: it just re-wraps the master
key, without needing to re-encrypt anything you've already stored.

Each stored file, and the vault's internal file listing, gets its own key
derived from the master key, so no two items ever share key material —
compromising one derived key never exposes the master key or any other
file. Turning your password into the key that wraps the master key is
deliberately slow, tuned to cost real time and memory per guess, which is
what makes brute-forcing a stolen vault expensive even for a short
password. Everything — the wrapped master key, the file listing, and
every stored file — is protected by an authenticated form of encryption,
so any tampering is detected rather than silently accepted, and every
source of randomness comes from your operating system's secure random
generator.

The exact parameters used to slow down password guessing are stored in
each vault at creation time, so a vault always reopens correctly with the
settings it was made with, even if defaults change later. Vaults default
to a deliberately heavy setting (real-world guess cost measured in
hundreds of milliseconds to a few seconds), which can be raised further
if you want to trade slower vault-opening for stronger resistance to
brute-forcing.

## Caveats (read before trusting this with something serious)

- **Keeping keys out of swap is best-effort.** The techniques used reduce
  the chance that key material ends up in a swap file or crash dump, but
  can't guarantee it on every system configuration (for example, some
  virtualized environments, or when the running user lacks the
  permission to lock memory). Sleep/hibernation images are also outside
  this control.
- **This has not been independently audited.** It's built on well-vetted
  cryptographic building blocks, used correctly to the best of this
  implementation's own testing (see below) — but "not audited" is a
  meaningfully different statement from "audited and found sound."
- **Password strength is on you.** The slow key-derivation step makes
  brute-forcing expensive per guess, but a weak or reused password is
  still a weak or reused password.
- **No plausible deniability or hidden volumes.** A vault's existence is
  visible on disk as an ordinary folder.

## What's been tested

The reference build was actually exercised, not just compiled, before
being handed over:
- A full create → add → list → extract → delete → lock round trip,
  verified byte-for-byte against the original files.
- Boundary cases: an empty file, a file exactly the size of one internal
  chunk, and a file spanning several chunks with a partial final chunk.
- A wrong password is rejected without ever unwrapping the master key.
- A single flipped bit anywhere in a stored file's encrypted data is
  detected, and the extract is aborted before any output is written — no
  partial or corrupted file is ever left behind.
- Zero errors and zero leaked memory under memory-safety and
  leak-detection testing, across every scenario above.

That's solid evidence the implementation is *correct on the paths
tested*, not a substitute for a real third-party security review if
you're relying on this for anything you can't afford to lose or leak.

## Usage

```
SEALVAULT
=========

1. Create vault
2. Open vault
3. Add file
4. Extract file
5. List files
6. Delete file
7. Lock vault
8. Exit

Choice:
```

Once a vault is open you get an interactive prompt with shorthand
commands for adding, listing, extracting, deleting, locking, and exiting.
