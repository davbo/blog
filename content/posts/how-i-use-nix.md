---
title: How I use Nix
date: 2019-09-30T10:00:40Z
draft: false
author: 
 - Dave
summary: Reflections on how I've been using Nix for the past few months.
---

One of the frustrations of Nix is that it's so difficult to search for. No I
didn't mean *nix, or unix-like systems. This post is about the functional
package manager, [Nix][nix-website].

Over the past 6 months or so I've been using Nix for a few different use cases
with varying degrees of success.

## NixOS ##

I'm lucky to have a spare x200 ThinkPad which has excellent compatibility with
various operating systems. Last year I ran FreeBSD on it for a while as I
learned about [jails][bsd-jails] and [bhyve][bhyve]. Earlier this year I used
that same ThinkPad to install NixOS. Using Nix on there helped give me an idea
of how Nix works and get used to the syntax.

NixOS is still installed on that ThinkPad but when I bought a new [XPS
13][xps-13-dev-blog] I chose to install Ubuntu rather than NixOS. This was
mostly because I predicted hardware support would far better with Ubuntu but
also after using NixOS for a while I felt it was a better fit for server
environments than desktops.

Plus everything I liked about NixOS (the Nix bit..) is well supported on Ubuntu.

## Managing home configuration ##

While using NixOS I discovered [home-manager][home-manager]. It provides a
friendlier way to use Nix for installing and configuring many applications.

As an example this extract from my [home.nix][home.nix] installs [Visual Studio
Code][vscode] and some fonts:

```nix
  fonts.fontconfig.enable = true;
  home.packages = with pkgs; [
    vscode
    # Fonts
    fira-code
    fira-code-symbols
    source-code-pro
  ];
```

In the past I've found installing fonts to be a real pain and something I
completely forget how to do once I'm done. So I really appreciate the support
here in Nix / home-manager for doing just that.

As another example, here is how I setup emacs / spacemacs:

```nix
  programs.emacs.enable = true;
  home.file.".emacs.d" = {
    source = builtins.fetchGit {
      url = "https://github.com/syl20bnr/spacemacs";
      ref = "develop";
    };
    recursive = true;
  };
  home.file.".spacemacs".source = ./spacemacs.el;
```

This config will [install spacemacs][spacemacs installation] cloning the
repository into .emacs.d using Nix's fetchGit builtin. Finally my spacemacs
configuration, which sits alongside the home.nix file, is put in place.

## Developer environments ##

Nix has been really handy in my job as a software developer, particularly using
[nix-shell][nix-shell] together with [dotenv][dotenv].

I'd not come across dotenv until recently, the idea makes sense though. As you
`cd` about it looks for approved `$PWD` and takes some action. In this case it
applies all the environment variables you would receive from invoking nix-shell,
without actually moving into a subshell. This is nice and the .envrc files only
ever need to contain one thing: `use_nix`. There is also a
[dotenv-mode][dotenv-mode] for emacs I recommend.

Recently I made a demo web application in Python using flask. As Python
environments (especially those with multiple versions of Python) can be a bit of
a pain to setup I found myself reaching to Nix as the tool for the job.

I ended up with two files, an `.envrc` containing `use_nix` and `shell.nix`
containing:

```nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  buildInputs = with pkgs.python3Packages; [ requests flask ];
}
```

When I `cd` into the directory python3.7 and flask become available. I didn't
have to install Python 3.7, write a Pipfile / requirements.txt or even call
`pip`.

In cases where the project is beyond a demo and has others working on it I'd
want to follow Python standards, but this worked well for me here.

Recently I also needed to build a Java project which used Maven and required a
specific version of the JDK. Here's the `shell.nix` I wrote in this case:

```nix
{ pkgs ? import <nixpkgs> {} }:
let mvn = pkgs.maven.override { jdk = pkgs.jdk11; };
in pkgs.mkShell {
  buildInputs = [ mvn ];
}
```

The [maven package in nixpkgs][nixpkgs] uses the default jdk package, which was
jdk8 at the time. The Nix code here shows how an override can be used on
existing packages to create something new.

After the initial learning curve I've been finding Nix to be a useful tool I
reach to often. That said I've not found it to be a great fit for everything,
desktop OS packages and my golang environment a couple of noteworthy examples.

In the future I'd like to investigate building Docker images using Nix as I've
seen it mentioned as a strong point but I haven't had cause to look into it yet.

Thanks to former colleagues for bringing Nix to my attention a while back.

[nix-website]: https://nixos.org/nix/
[bsd-jails]: https://en.wikipedia.org/wiki/FreeBSD_jail
[bhyve]: http://bhyve.org/
[xps-13-dev-blog]: https://www.davbo.uk/posts/xps-2019-setup/
[home-manager]: https://github.com/rycee/home-manager
[home.nix]: https://github.com/davbo/cfg/blob/master/home.nix#L9-L16
[vscode]: https://code.visualstudio.com/
[nix-shell]: https://nixos.org/nixos/nix-pills/developing-with-nix-shell.html
[dotenv]: https://github.com/direnv/direnv
[nixpkgs]: https://github.com/NixOS/nixpkgs/blob/master/pkgs/development/tools/build-managers/apache-maven/default.nix
[pep517]: https://www.python.org/dev/peps/pep-0517/
[spacemacs installation]: https://github.com/syl20bnr/spacemacs#install
[dotenv-mode]: https://github.com/preetpalS/emacs-dotenv-mode
