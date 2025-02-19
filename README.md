# ssh-agent-lib

[![CI](https://github.com/wiktor-k/ssh-agent-lib/actions/workflows/ci.yml/badge.svg)](https://github.com/wiktor-k/ssh-agent-lib/actions/workflows/ci.yml)
[![Crates.io](https://img.shields.io/crates/v/ssh-agent-lib)](https://crates.io/crates/ssh-agent-lib)

A collection of types for writing custom SSH agents as specified by the [SSH Agent Protocol Internet Draft](https://datatracker.ietf.org/doc/html/draft-miller-ssh-agent).

This makes it possible to utilize remote keys not supported by the default OpenSSH agent.

## Example

The following example starts listening on a socket and processing requests.
On Unix it uses `ssh-agent.sock` Unix domain socket while on Windows it uses a named pipe `\\.\pipe\agent`.

```rust,no_run
#[cfg(not(windows))]
use tokio::net::UnixListener;
#[cfg(windows)]
use ssh_agent_lib::agent::NamedPipeListener;

use ssh_agent_lib::agent::{Session, Agent};
use ssh_agent_lib::proto::message::Message;

#[derive(Default)]
struct MyAgent;

#[ssh_agent_lib::async_trait]
impl Session for MyAgent {
    async fn handle(&mut self, message: Message) -> Result<Message, Box<dyn std::error::Error>> {
        match message {
            Message::SignRequest(request) => {
                // get the signature by signing `request.data`
                let signature = vec![];
                Ok(Message::SignResponse(signature))
            },
            _ => Ok(Message::Failure),
        }
    }
}

#[tokio::main]
#[cfg(not(windows))]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let socket = "ssh-agent.sock";
    let _ = std::fs::remove_file(socket); // remove the socket if exists

    MyAgent.listen(UnixListener::bind(socket)?).await?;
    Ok(())
}

#[tokio::main]
#[cfg(windows)]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    MyAgent.listen(NamedPipeListener::new(r"\\.\pipe\agent".into())?).await?;
    Ok(())
}
```

Now, point your OpenSSH client to this socket using `SSH_AUTH_SOCK` environment variable and it will transparently use the agent:

```sh
SSH_AUTH_SOCK=ssh-agent.sock ssh user@example.com
```

On Windows the path of the pipe has to be used:

```sh
SSH_AUTH_SOCK=\\.\pipe\agent ssh user@example.com
```

For more elaborate example see the `examples` directory or [crates using `ssh-agent-lib`](https://crates.io/crates/ssh-agent-lib/reverse_dependencies).

## Note

This library has been forked from
[sekey/ssh-agent.rs](https://github.com/sekey/ssh-agent.rs) as the
upstream seems not be maintained (at least as of 2022).

# License

This project is licensed under the [MIT license](https://opensource.org/licenses/MIT).

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in this crate by you shall be licensed as above, without any additional terms or conditions.
