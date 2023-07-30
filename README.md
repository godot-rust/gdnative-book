# The godot-rust book

The godot-rust book is a user guide for the Rust bindings. The book is still work-in-progress, and contributions are very welcome.

An online version of the book is available at [godot-rust.github.io/book][book-web].

The book is built with [mdBook] and the plugins [mdbook-toc] and [mdbook-admonish]. To install them and build the book locally, you can run:
```bash
$ cargo install mdbook mdbook-toc mdbook-admonish
$ mdbook build
```

To run a local server with automatic updates while editing the book, use:
```bash
$ mdbook serve --open
```


## Contributing

This repository is for documentation only. Please open pull requests targeting the libraries themselves in the main repos for the [Godot 3] and [Godot 4] bindings.

For contributions, see the contributing guidelines under `CONTRIBUTING.md` in the above-mentioned repositories.


## License

Any contribution intentionally submitted for inclusion in the work by you shall be licensed under the [MIT license], without any additional terms or conditions.

[book-web]: https://godot-rust.github.io/book
[mdBook]: https://github.com/rust-lang-nursery/mdBook
[mdbook-toc]: https://github.com/badboy/mdbook-toc
[mdbook-admonish]: https://github.com/tommilligan/mdbook-admonish
[Godot 3]: https://github.com/godot-rust/gdnative
[Godot 4]: https://github.com/godot-rust/gdext
[MIT license]: LICENSE.md
