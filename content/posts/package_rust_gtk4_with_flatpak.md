---
title: "Package Rust Gtk4 apps with flatpak"
date: 2024-12-21T14:49:47+01:00
draft: true
---

I have played around with __Gtk 4__ and __Rust__ to write Desktop applications.  
But I did struggle a lot to package them with __Flatpak__.

In this article we will see __how to package__ a simple __Gtk 4__ app with __Flatpak__ for __Gnome__ and __Elementary OS__.
Will will also do an example for __[Relm4](https://relm4.org/)__ which make writing Gtk 4 for Rust a lot more pleasant.

Also, please note that we __don't need Meson__, so we won't use it. But you will have to use something else for translations.

## Let's create a simple App 🚀

First, let's make a __minimal Gtk4 app__.

### Install the dependencies

If you don't have rust. [Install it](https://www.rust-lang.org/tools/install) with
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

We also need the [Gtk 4 libraries](https://gtk-rs.org/gtk4-rs/stable/latest/book/installation_linux.html):

```shell
sudo apt install libgtk-4-dev build-essential
```

### Create the project:

As with most Rust projets. Let's use `cargo` to compile and handle the dependencies.

``` bash
cargo new demo-app
```

### Add Gtk 4 to our projet

Find [what version of Gtk 4](https://gtk-rs.org/gtk4-rs/stable/latest/book/project_setup.html) is installed on your system:

```bash
$ pkg-config --modversion gtk4
4.6.9
```

Here I have the __version 4.6__. Let's install add it to `Cargo.toml`

```bash
cd demo-app
cargo add gtk4 --rename gtk --features v4_6
```

Your `Cargo.toml` should now look something like this:

``` Toml
[package]
name = "demo-app"
version = "0.1.0"
edition = "2021"

[dependencies]
gtk = { version = "0.9.5", package = "gtk4", features = ["v4_6"] }
```

## Code our app

Edit `src/main.rs` to create our app.

__main.rs:__
``` rust
use gtk::{glib, prelude::*};

fn main() -> glib::ExitCode {
    let application = gtk::Application::builder()
        .application_id("pro.lasne.demo-app")
        .build();
    application.connect_activate(build_ui);
    application.run()
}

fn build_ui(application: &gtk::Application) {
    let window = gtk::ApplicationWindow::new(application);

    window.set_title(Some("Hello world"));
    window.set_default_size(200, 70);

    let button = gtk::Button::with_label("Click me!");
    button.connect_clicked(|_| {
        println!("You clicked me !");
    });

    window.set_child(Some(&button));
    window.present();
}
```

This code is taken from [this example](https://github.com/gtk-rs/gtk4-rs/blob/main/examples/basics/main.rs) on Github.
You can also look at the [Gtk4-rs book](https://gtk-rs.org/gtk4-rs/stable/latest/book/hello_world.html).

We have some boilerplate to create a Gtk 4 app. Flatpak use a _reverse DNS_ naming scheme for apps. You can also use something like `io.github.username.repo` for your app.

We define a _window size_ (200x70), a _title_.
And then we create a _button_, and associate a _event_ when the button is clicked.

### Run the code

We can __build the app__ with

```bash
cargo build
```

or

```bash
cargo build --release
```

This will create the binary `./target/release/demo-app`.

If everything is correctly setup, you should be able to __run your app__ with.

```bash
cargo run
```

from the `demo-app` folder. The following window should appear.

![Demo App running](/gtk4-flatpak/demo-app.png)

Now that our app is running, we need to __choose a runtime to target__. (But what is a runtime you might ask 🤔?)

## A quick Introduction to Flatpak 🧑‍🏫

[Flatpak](https://flatpak.org/) is a way to __package Linux apps__, and distribute them for all distributions at once via [flathub](https://flathub.org/). This is achieved by isolating each application, and provide them an environment to run in.

### Runtimes 📚️

To run, Flatpak apps use a [__runtime__](https://docs.flatpak.org/en/latest/available-runtimes.html) that provide a _base image_ of the system with shared libraries.

![Flatpak Runtimes](/gtk4-flatpak/runtimes.svg)


Runtimes are associated with a version of _Gnome_, _KDE_ or _Elementary_. All runtimes are __based on__ the __FreeDesktop runtime__. 

You can list the installed runtimes with: 

```bash
flatpak list --runtime
```

FreeDesktop runtimes a released a bit like Ubuntu versions with a _year.month_ scheme:

* 23.08
* 24.08

The are 2 components you might want to install:

* the platform
* the SDK.

For example, to target __Gnome 47__, we will __install__ the __Gnome Platform__, and the __Gnome SDK__. 

```bash
flatpak install --user org.gnome.Sdk//47 org.gnome.Platform//47
```

To build a __Rust program__, we also need to install the [__Rust SDK__ extension](https://github.com/flathub/org.freedesktop.Sdk.Extension.rust-stable).  
_Gnome 47_ platform is __build on top__ of the _freedesktop 24.08_ platform, so we are going to install this version of the extension.

```bash
flatpak install --user org.freedesktop.Sdk.Extension.rust-stable//24.08 org.freedesktop.Sdk.Extension.llvm18//24.08
```

## Flatpak our App 📦️

Every flatpak is defined by a __Manifest__.

A ___manifest_ defines__:
* the __runtime__ used
* instructions for __building__ the app
* __where__ the __files__ are __installed__.

A manifest can be either in _json_ or in _yaml_.

Since our __app ID__ is `pro.lasne.demo-app`, let's __create__ a `pro.lasne.demo-app.yml` __manifest__ at the root of our project.

``` YAML
app-id: pro.lasne.demo-app
runtime: org.gnome.Platform
runtime-version: '47'
sdk: org.gnome.Sdk
sdk-extensions:
  - org.freedesktop.Sdk.Extension.rust-stable
command: demo-app
finish-args:
  - --share=ipc
  - --socket=fallback-x11
  - --socket=wayland
  - --device=dri
build-options:
  append-path: /usr/lib/sdk/rust-stable/bin
modules:
  - name: demo-app
    buildsystem: simple
    build-options:
      env:
        CARGO_HOME: /run/build/demo-app/cargo
    build-commands:
      - cargo --offline fetch --manifest-path Cargo.toml --verbose
      - cargo --offline build --release --verbose
      - install -Dm755 ./target/release/demo-app -t /app/bin/
    sources:
      - type: dir
        path: .
      - cargo-sources.json
```

⚠️ Make sure that `app-id` match the _application id_ in your code.

Let's explore a bit what we have here.

#### Common Flatpak informations:

* `app-id`: the __ID__ of our app, with a reverse-DNS scheme.
* `command`: the __command executed__ to run our app.

#### Gnome Runtime:

* the __`runtime`__ and `sdk`: here we target ___Gnome 47___.
* We also need to define some `finish-args` to let our app __access__ the __graphical__ server _X_ or _Wayland_.

#### Rust:

* `sdk-extensions`: __adds__ the __Rust SDK__ to our runtime.
* `build-commands`: some commands to __build__ our app __with `cargo`__.
* `install`: is used to copy `demo-app` to `/app/bin/`.

### Integrate Cargo with Flatpak Builder 🪄

We have __one last thing__ to do before building our app.  
You might have notice is the `cargo-sources.json` in the manifest.

To build the app with `cargo` in a flatpak. We need to generate this file using the script [`flatpak-cargo-generator.py`](https://github.com/flatpak/flatpak-builder-tools/blob/master/cargo/flatpak-cargo-generator.py) from the [flatpak-builder-tools](https://github.com/flatpak/flatpak-builder-tools) repository (in the `cargo` folder).

```bash
python3 flatpak-cargo-generator.py Cargo.lock -o cargo-sources.json
```

## Build and Install 🛠️

We can finally build our flatpak.

Your source directory should look like this:

```shell
demo-app
├── Cargo.toml
├── Cargo.lock
├── cargo-sources.json
├── pro.lasne.demo-app.yml
├── src
│   └── main.rs
└── target
```

You can __build the app__ with:

```bash
flatpak-builder --user flatpak_app pro.lasne.demo-app.yml --force-clean
```

If everything works without errors. You can install the app on you system.

```bash
flatpak-builder --user flatpak_app pro.lasne.demo-app.yml --force-clean --install
```

Your app should now be listed with the installed flatpak:

```bash
$ flatpak list

demo-app        pro.lasne.demo-app              master  demo-app-origin user
```

You can run it with 
```bash
flatpak run pro.lasne.demo-app
```

![Running the app in a flatpak](/gtk4-flatpak/flatpak_run.png)


# Conclusion

In this article we have seen how to package with Flatpak a Gtk 4 app written in Rust.  
In the [next article](), we'll see how to integrate with _elementary OS_.
