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
        .application_id("pro.lasne.demo")
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

If everything is correctly setup, you should be able to run your app with.

```bash
cargo run
```

from the `demo-app` folder. The following window should appear.

![Demo App running](/gtk4-flatpak/demo-app.png)

Now that our app is running, we need to __choose a runtime to target__. (But what is a runtime you might ask 🤔?)

## A quick Introduction to Flatpak 🧑‍🏫

[Flatpak](https://flatpak.org/) is a way to __package Linux apps__, and distribute them for all distributions at once via [flathub](https://flathub.org/). This is achieved by compartimenting each application.

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
_Gnome 47_ platform is build on top of the _freedesktop 24.08_ platform, so we are going to install this version of the extension.

```bash
flatpak install --user org.freedesktop.Sdk.Extension.rust-stable//24.08 org.freedesktop.Sdk.Extension.llvm18//24.08
```

## Flatpak our App 📦️

Every flatpak starts with a __Manifest__.

This file define:

* the runtimes
* the files

