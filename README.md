# MirageVPN book

You can also read a [rendered version](https://robur-coop.github.io/miragevpn-handbook/).

This book use `mdbook` and `mdbook-graphviz`. You can install them via `rustup`
and `cargo`:

```sh
$ rustup install stable
$ cargo install mdbook
$ cargo install mdbook-graphviz
$ export PATH=$PATH:$HOME/.cargo/bin
$ git clone https://git.robur.coop/robur/miragevpn-handbook
$ cd miragevpn-handbook
$ mdbook serve
```

You can see the book on `http://localhost:3000`
