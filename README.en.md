# amiscara-ssh

[Deutsch](README.md) · **English**

Shared SSH client of the Amiscara apps ([Amiscara Connect](https://github.com/toolwerk/amiscara-connect) and [Amiscara Agents](https://github.com/staude/amiscara-agents)), built on [russh](https://github.com/Eugeny/russh).

The crate contains everything that is only SSH: connecting and authenticating, host key verification with TOFU, keys, channels for a shell with PTY, `direct-tcpip` and `direct-streamlocal`, jump host chains with a session pool and agent forwarding, one-time codes over keyboard-interactive, and SFTP through `Session::open_sftp`. Whatever belongs to an app stays in the app.

## Using it

The apps pull the crate in as a Git dependency with a fixed tag:

```toml
amiscara-ssh = { git = "https://github.com/toolwerk/amiscara-ssh", tag = "v0.8.0" }
```

Whoever changes the crate and an app at the same time adds a local, uncommitted `.cargo/config.toml` to the app:

```toml
[patch."https://github.com/toolwerk/amiscara-ssh"]
amiscara-ssh = { path = "../amiscara-ssh" }
```

The patch key must match the URL the app uses in its dependency exactly.

## Development

- Build: `cargo build`
- Tests: `cargo test`, lint: `cargo clippy --all-targets -- -D warnings`, format: `cargo fmt`
- Tests against the local SSH container (needs Docker):

  ```sh
  scripts/ssh-testserver.sh start
  cargo test -- --ignored
  scripts/ssh-testserver.sh stop
  ```

  The server runs on `127.0.0.1:2222` (other port through `AMISCARA_SSH_TEST_PORT`), user `tester`, keys in `target/ssh-testserver/`. Tests never run against foreign or production servers.
- Tests against the jump host chain (needs Docker): `scripts/ssh-testkette.sh run` starts a bastion, a jump host and a target as containers and a separate `ssh-agent` with the test key, runs `cargo test --test chain -- --ignored` and tears everything down again, also on failure. Only the bastions are reachable from outside; the bastion with a password listens on `127.0.0.1:2223` (other port through `AMISCARA_SSH_CHAIN_PORT`). Every hop has its own user, its own credentials and its own host key, all in `target/ssh-testkette/`. Next to it runs `otp-bastion` on `127.0.0.1:2224` (other port through `AMISCARA_SSH_OTP_PORT`), a bastion that asks for a TOTP code over keyboard-interactive after the key `id_otp` (secret in `target/ssh-testkette/totp_secret`), with the same jump host and target behind it. Then `kbd-bastion` on `127.0.0.1:2225` (other port through `AMISCARA_SSH_KBD_PORT`), a bastion that accepts the password `kbd-pw` of `kbd-user` only over keyboard-interactive, like many hosting providers, again with jump host and target behind it. And `kbd-otp-bastion` on `127.0.0.1:2226` (other port through `AMISCARA_SSH_KBD_OTP_PORT`), a bastion that asks for the password `kbd-otp-pw` of `kbd-otp-user` first and then for the TOTP code, both over keyboard-interactive in one login (PAM), with jump host and target behind it as well. `start` and `stop` are also available separately.

## Example

```rust
use amiscara_ssh::{HostKeyPolicy, Identity, PtySize, Session, ShellEvent, SshTarget};

// inside an async function, `home` is the home directory
let target = SshTarget {
    host: "example.org".into(),
    port: 22,
    user: "deploy".into(),
    identity: Identity::Agent,
    host_key: HostKeyPolicy::KnownHostsFile {
        path: home.join(".ssh/known_hosts"),
        accept_new: true,
    },
    agent_forward: false,
    keyboard_interactive: None,
};
let session = Session::connect(&target).await?;
let mut shell = session.open_shell("xterm-256color", PtySize { cols: 80, rows: 24 }).await?;
shell.write(b"uptime\n").await?;
while let Some(ShellEvent::Output(bytes)) = shell.next().await {
    print!("{}", String::from_utf8_lossy(&bytes));
}
```

## Jump host chains

A `Route` consists of the hops in order and the target. Every hop is its own `SshTarget` with user, credentials and host key; the hop before it resolves the name, and the host key is stored in `known_hosts` under that name. The `SessionPool` keeps one connection per hop for as long as a `Lease` or a connection behind it uses it.

```rust
use amiscara_ssh::{HopState, PoolConfig, PtySize, Route, SessionPool, SshTarget};

// bastion, jump and target are SshTarget values as above
let pool = SessionPool::new(PoolConfig::default());
let mut events = pool.subscribe();
let route = Route { jumps: vec![bastion, jump], target, proxy_command: None };
let session = pool.connect(&route).await?; // second call: hops are reused
let shell = session.open_shell("xterm-256color", PtySize { cols: 80, rows: 24 }).await?;
while let Ok(event) = events.try_recv() {
    // e.g. "jump-user@jump:22: Connected { latency: 12ms }"
    println!("{}: {:?}", event.route[event.index], event.state);
}
```

- `connect_fresh` opens a separate connection only to the target; the hops still come from the pool.
- If a hop fails, `connect` returns `SshError::Hop` with the index and `user@host:port`; the event `HopState::Failed` names the kind of failure (`HopFailure`).
- `agent_forward: true` on the target gives shells there the local SSH agent. Hops never get it, even if their `SshTarget` asks for it.
- A `Lease` is returned by dropping it, not with `disconnect()`: that would close the shared connection for everyone. An aborted `connect` also leaves nothing behind in the pool.
- The pool key is the path of host, port and user per hop (for the target also `agent_forward`, for the first hop also `proxy_command`). Credentials and the host key rule are not part of it: a live connection is reused the way it was authenticated and verified.

## One-time codes

When a server wants to know something during login, for example a bastion asking for a one-time code after the key, the crate asks the app through keyboard-interactive: `SshTarget::keyboard_interactive` takes an `Arc<dyn KeyboardInteractive>`. Only the hop that asks is asked, and only when the server has accepted the `Identity` as a partial success and then still requires keyboard-interactive. Rejected keys stay a rejection (`SshError::AuthFailed`) without a question. A password that the server rejects through the `password` method is sent once more over keyboard-interactive when the server offers that method and no longer offers `password` (password over PAM, `PasswordAuthentication no`): the first request of an attempt with exactly one hidden question gets the stored password without asking the app; only further questions, such as a one-time code afterwards, go to `SshTarget::keyboard_interactive`, and without a prompter the login ends in `SshError::AuthFailed`. If the server keeps offering `password`, the password itself was rejected and is not sent again.

```rust
use std::sync::Arc;
use amiscara_ssh::{KeyboardAnswer, KeyboardInteractive, KeyboardPrompt, PromptFuture, Zeroizing};

#[derive(Debug)]
struct Dialog; // in the app: a question to the user interface

impl KeyboardInteractive for Dialog {
    fn ask(&self, prompt: KeyboardPrompt) -> PromptFuture {
        // prompt.route[prompt.index] is the asking hop, prompt.route the path,
        // prompt.prompts the questions (echo false: enter hidden), prompt.attempt from 1
        Box::pin(async move {
            let code = String::from("123456"); // from the user
            Some(KeyboardAnswer {
                responses: vec![Zeroizing::new(code)],
                keep_open: None, // checkbox "keep open": Some(Duration::from_secs(600))
            })
        })
    }
}

// bastion is an SshTarget as above
bastion.keyboard_interactive = Some(Arc::new(Dialog));
```

- While the question is open, the hop reports `HopState::AwaitingInput`. The waiting time counts neither toward `PoolConfig::connect_timeout` nor toward latency.
- `None` as the answer aborts: `SshError::Cancelled`, in the event `HopFailure::Cancelled`. Nothing of the asking hop stays in the pool, not even when a `connect` is dropped during the question; a hop before it that already received a code stays open as after the last `Lease`.
- A rejected code is asked again on the same connection, at most three attempts (`attempt` 1 to 3), then `SshError::AuthFailed`. If the server requires several keyboard-interactive steps in a row, `attempt` stays the same for the next step. If a server asks for the password first and then for a code in the same login through PAM, a rejected code is asked again up to three times as well. Because the server starts the whole login over after every rejection and asks for the password first again, the stored password is sent at most once per attempt, on the first hidden single question of that attempt. If the server accepted the password as a step of its own (partial success), it is not sent again; further questions, including a second password question, go to the app. If the server rejects right after the password without the app having been asked anything, the password counts as wrong: `SshError::AuthFailed` follows immediately, without a question to the app and without sending the password again. Limit: if a server still asks for the code after a wrong password (PAM rule `required` instead of `requisite`), a wrong password cannot be told apart from a wrong code; the app is then asked for the code up to three times and the password is sent up to three times, as with `ssh` by hand. The crate rejects more than 16 requests and steps in one login (`SshError::AuthFailed`).
- The waiting time is limited by the server itself (OpenSSH `LoginGraceTime`, usually 120 s); if the user answers later, the connection setup ends with `HopFailure::Closed`.
- The pool keeps a hop that asked open after the last `Lease`: `keep_open` of the answer, otherwise `PoolConfig::code_grace` (60 s). During that time a new session needs no code, not even after a later failure such as a host key confirmation. `SessionPool::close_idle` closes all unused connections at once, for example when the app is locked. If the bastion closes unused connections itself (OpenSSH `UnusedConnectionTimeout`), keeping open ends earlier. A hop is only kept open when the last `Lease` is dropped inside a Tokio runtime; outside of one (for example on a foreign thread) the hop closes at once.
- russh copies the answers into plain `String`s when sending; only the copies in `KeyboardAnswer` are wiped on drop.

## ProxyCommand

Some servers can only be reached through a helper program, such as `cloudflared` or AWS SSM. `Session::connect_command` starts a local command and speaks SSH over its stdin and stdout; in a chain, `Route::proxy_command` does that for the first hop (the first jump host or, without jump hosts, the target).

```rust
use amiscara_ssh::{Route, Session};

// target is an SshTarget as above
let session = Session::connect_command("cloudflared access ssh --hostname %h", &target).await?;

let route = Route {
    proxy_command: Some("aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters portNumber=%p".into()),
    ..Route::direct(target)
};
let session = pool.connect(&route).await?;
```

- Placeholders as in OpenSSH: `%h` host, `%p` port, `%r` user, `%%` a literal `%`. Any other placeholder or a single `%` at the end ends in `SshError::ProxyCommand` before the start.
- Host and user go into the command line unescaped. They may therefore consist only of ASCII letters, digits and `. - _ : @` and must not start with `-`, otherwise the crate refuses before the start. That way an entry in a shared document cannot smuggle in foreign commands.
- On Unix the command runs as `/bin/sh -c "exec <command>"` (like OpenSSH), on Windows as `cmd.exe /S /C "<command>"` without a console window; quotes in the command (for example around `C:\Program Files\...`) are kept.
- If the command ends before login, `SshError::ProxyCommand` names its exit status and the last 2 KiB of its error output, never the command itself (it may contain an access token): if the error output repeats it, as a shell does on a syntax error, `<command>` stands there; words of the command and parts of them (between quotes, `=`, `:`, `@` and the like) of 8 characters or more become `...`. This is a best-effort protection: if a helper prints a secret it obtained another way, the secret stays. If the command ends with an error while a process it started still holds stdout open, the login aborts at once. If a helper ends only after the connection has failed on an SSH error, and ends successfully, the SSH error stands. The event carries `HopFailure::ProxyCommand`. Host key and login errors stay what they are. `Debug` of `Route` does not show the command either.
- In the pool, setting up the first hop through a command may take up to `PoolConfig::proxy_connect_timeout` (120 s), because `cloudflared` for example waits for the browser login the first time. `Session::connect_command` has no time limit of its own, like `Session::connect`.
- When the connection ends, the crate ends the command; if it does not end by itself within one second, it is killed, on Windows together with everything running under `cmd.exe`.

## Port forwarding

`SessionPool::forward_local` listens on a local port and forwards every connection over a `direct-tcpip` channel to host and port, as the target of a `Route` sees them. The hops come from the pool and are shared with other sessions.

```rust
use amiscara_ssh::{Route, SessionPool};

// pool and route as above; port 0 picks a free port
let forward = pool
    .forward_local(route, "127.0.0.1:5432".parse()?, "db.internal".into(), 5432)
    .await?;
println!("listening on {}", forward.local_addr());
let mut status = forward.status();
status.changed().await?;
println!("active: {}, error: {:?}", status.borrow().active, status.borrow().error);
```

- The port is bound first, then the route is set up. If the port is taken, the call ends in `SshError::PortInUse` with the address before any connection exists; errors while setting up the route come unchanged as `SshError::Hop`. Taken also means: another program listens on `0.0.0.0` (or `::` for an IPv6 address) on the same port. On macOS and the BSDs, `SO_REUSEADDR` would otherwise allow `127.0.0.1` next to it, Windows does so by itself, and clients would silently end up in the tunnel instead of at the program; only Linux refuses this itself.
- The `LocalForward` holds a `Lease` on the route for as long as it lives. If the connection has ended when a client connects, it sets the route up again once for that client. This runs in the client's task, and the port keeps accepting meanwhile; clients that arrive in that time wait for this setup and use the new `Lease`; if it failed, the next one tries again itself.
- `ForwardStatus` reports the active connections and, as `HopFailure`, the reason why the last connection could not be forwarded, for example `ChannelRefused` when nothing listens at the target. The port keeps listening afterwards; the next successful connection clears the error.
- Drop ends the listening and every forwarded connection and returns the `Lease`. The forwarding runs in tasks of the Tokio runtime in which `forward_local` was called.
- Which address is bound is up to the app; `127.0.0.1` keeps the port off the network.

## SFTP

`Session::open_sftp` starts the subsystem `sftp` on a new channel, also on a `Lease` behind jump hosts, and returns a `russh_sftp::client::SftpSession`. The crate re-exports `russh_sftp` so that the apps use the same version.

```rust
let sftp = session.open_sftp().await?;
for entry in sftp.read_dir(".").await? {
    println!("{}", entry.file_name());
}
```

- The SFTP session does not hold the connection itself: the `Session` or `Lease` must live for as long as it is used.
- A server without SFTP ends in `SshError::SftpUnavailable`.

## Release

SemVer from `0.1.0`. While 0.x applies, a new minor version breaks the interface. Process: a pull request with an entry in `CHANGELOG.md`, squash merge, turn `[Unreleased]` into the new version, push the tag `v<version>`.

## License

MIT or Apache-2.0, at your option: [LICENSE-MIT](LICENSE-MIT), [LICENSE-APACHE](LICENSE-APACHE).
