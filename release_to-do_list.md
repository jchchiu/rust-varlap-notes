FOR RELEASES:
update release tags for
`rust-varlap/src/cli.rs`
```
#[derive(Debug, Parser)]
#[command(name = "rust-varlap")]
#[command(version = "0.1.0-alpha.5")]
```
`rust-varlap/Cargo.toml`
```
[package]
name = "rust-varlap"
version = "0.1.0-alpha.5"
```

tag the release
e.g.
`git tag v0.1.0-alpha.5 -m "rust-varlap 0.1.0-alpha.5"`

push the release
e.g.
`git push origin v0.1.0-alpha.5`