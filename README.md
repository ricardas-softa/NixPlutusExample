# Plutus Development with Nix
This repository contains a sample Plutus project and a guided walkthrough for converting it to a Nix project (using Flakes, `haskell.nix`, and optionally `direnv`) for enhanced reproducibility and development experience.

## Requirements:
* Nix is only compatible with Unix-like operating systems. You must be using a Linux distribution, MacOS, or WSL2 (Windows Subsystem for Linux) to work with Nix.
* This guide assumes the use of `VS Code` as editor and `bash` as SHELL. Other tools will require alternative workflows that are not covered here.
* This project is storage-intensive. We suggest you have at least `30GB` of free disk space before proceeding further.
* **NOTE for MacOS users:** MacOS may ship with versions of `bash` and `grep` that are incompatible with this workflow. You should install `bash`/`grep` using Homebrew first before proceeding.

## Instructions:
1. Fork and clone this repository. Open the root folder in VS Code and a `bash` terminal.
  * When you open the project in VS Code, you should be prompted to install two recommended extensions: `direnv` and `Nix IDE`

2. [Install Nix](https://nixos.org/download.html) package manager for your OS
  * Follow the instructions for **multi-user installation** at the link above
  * Then follow the prompts in your terminal
  * When you are finished installing, close the terminal session and open a fresh one.

3. Install `nix-prefetch-git`
  * We'll need to calculate SHA-256 hashes for Plutus dependencies using the `nix-prefetch-git` package. This can be installed system-wide via Nix:
    ```bash
    $ nix-env -iA nixpkgs.nix-prefetch-git
    ```
  * If you'd rather not install this, we can use `nix-prefetch-git` transiently in a nix-shell instead. Follow the instructions in **Step 7** instead.

4. Enable Flakes & Configure Binary Cache:
  * Edit `/etc/nix/nix.conf`
    * This requires root access to edit. Use a terminal-based editor like `nano` (i.e.):
      ```bash
      $ sudo nano /etc/nix/nix.conf
      ```
  * Modify the file following the instructions below:
    ```
    # Sample /etc/nix/nix.conf

    # Leave this line alone (may appear differently in your file)
    build-users-group = nixbld

    # Step 1: Add this line to enable Flakes
    experimental-features = nix-command flakes

    # Step 2: Set up binary cache (replace existing substituters and trusted-public-keys lines if present)

    substituters = https://cache.iog.io https://cache.nixos.org https://digitallyinduced.cachix.org https://all-hies.cachix.org
    trusted-public-keys = hydra.iohk.io:f/Ea+s+dFdN+3Y/G+FDgSq+a5NEWhJGzdjvKNGv0/EQ= cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY= digitallyinduced.cachix.org-1:y+wQvrnxQ+PdEsCt91rmvv39qRCYzEgGQaldK26hCKE= all-hies.cachix.org-1:JjrzAOEUsD9ZMt8fdFbzo3jNAyEWlPAwdVuHw4RD43k=

  * **IMPORTANT!** You must restart the `nix-daemon` to apply the changes

    **Linux:**
      ```bash
      $ sudo systemctl restart nix-daemon
      ```
    **MacOS:** \
    Find the name of the `nix-daemon` service

    ```bash
    $ sudo launchctl list | grep nix
    ```
    Then stop and restart the service
    ```bash
    $ sudo launchctl stop <NAME>
    $ sudo launchctl start <NAME>
    ```

5. Create `flake.nix` in the project root directory and add flake "scaffolding" code from [haskell.nix](https://input-output-hk.github.io/haskell.nix/tutorials/getting-started-flakes.html#scaffolding).

6. Modify the `flake.nix` file:
  * In the `helloProject` overlay, set `compiler-nix-name` to `"ghc8107"` and `hlint` version in `shelltools` to `"3.4.1"` (the latest version of `hlint` is incompatible with GHC 8.x).
  * Comment out the `packages.default` line at the bottom, since this sample is a library project with no executables.
    * For projects with executables, a default executable can be specified here in the format `"<CABAL PROJECT NAME>:exe:<EXECUTABLE NAME>"`. The `nix run` command (with no additional arguments) will then run this executable.
  * **IMPORTANT!** You must stage the `flake.nix` file using the VS Code Source Control tab (or the command line if you have this method configured). If the flake is not staged you will not be able to load the Nix environment in **Step 8**.

7. Add missing SHA-256 hashes for dependencies in `cabal.project`.
  * If you attempt to load the Nix shell now with the `nix develop` command, you will receive an error.
  * Since Nix Flakes require pure inputs to guarantee reproducibility, and the content associated with a particular Git repository and tag can change, we need to hash any repositories we include in the `cabal.project` file.
  * We can calculate SHA-256 hashes for dependencies using the `nix-prefetch-git` package. If you installed this package via Nix in **Step 3**, skip to the next bullet point. If not, you can load a transient Nix shell with the package installed, and perform the hashing in this shell session:

    ```bash
    $ nix-shell -p nix-prefetch-git
    ```
  * We now need to calculate a SHA-256 hash for each `source-repository-package` listed in the `cabal.project` file, using the following command:
    ```
    $ nix-prefetch-git <LOCATION> <TAG>
    ```
    Here is an example of how we'd calculate a hash for the `plutus-apps` dependency with tag `87b647b05902a7cef37340fda9acb175f962f354`
      ```bash
      $ nix-prefetch-git https://github.com/input-output-hk/plutus-apps.git 87b647b05902a7cef37340fda9acb175f962f354

      ...

      git revision is 87b647b05902a7cef37340fda9acb175f962f354
      path is /nix/store/blhgxza7yxn0lgzylqrv7fr11gm3l467-plutus-apps-87b647b
      git human-readable version is -- none --
      Commit date is 2022-08-29 09:13:27 -0400
      hash is 06h2bx91vc4lbn63gqgb94nbih27gpajczs46jysq1g7f24fp3bj
      {
      "url": "https://github.com/input-output-hk/plutus-apps.git",
      "rev": "87b647b05902a7cef37340fda9acb175f962f354",
      "date": "2022-08-29T09:13:27-04:00",
      "path": "/nix/store/blhgxza7yxn0lgzylqrv7fr11gm3l467-plutus-apps-87b647b",
      "sha256": "06h2bx91vc4lbn63gqgb94nbih27gpajczs46jysq1g7f24fp3bj",
      "fetchLFS": false,
      "fetchSubmodules": false,
      "deepClone": false,
      "leaveDotGit": false
      }
      ```
  * The hash string must now be added as a comment prexied with `--sha256:` anywhere inside the `source-repository-package` stanza like so:
    ```
    source-repository-package
      type: git
      location: https://github.com/input-output-hk/plutus-apps.git
      tag: 87b647b05902a7cef37340fda9acb175f962f354
      --sha256: 06h2bx91vc4lbn63gqgb94nbih27gpajczs46jysq1g7f24fp3bj
    ```
  * For convenience, we've added hashes for most of the dependencies of this project, but a few at the bottom are missing. Calculate and add them to the `cabal.project` file using the procedure above.
    * This exercise is just to help you practice the procedure for adding flake-compatible dependencies in `cabal.project`. In the future, you can copy and paste the same hashed `cabal.project` dependencies into other projects and only need to recalculate hashes if you bump the tag of a particular repo.

8. From the project root directory in your `bash` shell, run `nix develop` to load the Nix environment.
  * It will take some time to set up the environment the first time
  * If you forgot to stage your `flake.nix` file in **Step 6**, this command will fail immediately. Stage the file and try again if this happens.
  * Warnings about the Git tree being dirty, or messages about "No index state specified" can be disregarded (you can remove the dirty Git tree warning by staging and committing your changes to Git)
  * Some dependencies will need to be built from source, but if it says "building" for certain packages that should be downloadable from a binary cache (particularly GHC, the Linux kernel, and other non-Haskell related dependencies) or if you see any warning such as `warning: ignoring substitute`, this means your binary cache was not set up correctly and Nix is attempting to build packages from source that it should be fetching from a cache. Exit with `CTRL+c` and repeat **Step 4**, then try again. Make sure to restart the `nix-daemon`!
  * If you see any `5XX` HTTP errors, it means the IOG binary cache is non-responsive. Wait a bit and try again later.
  * You can skip to **Step 10** and begin setting up `direnv` while you wait, but don't complete **Step 11** until this process has fully completed.

9. Test the Haskell environment.
  * Inside the Nix shell, run the following to confirm that the environment is configured correctly:
    ```bash
    $ ghc --version
    The Glorious Glasgow Haskell Compilation System, version 8.10.7
    $ hlint --version
    HLint v3.4.1, (C) Neil Mitchell 2006-2022
    ```
  * Now run `cabal repl` to compile/load the project in GHCi (this will take some time). When the GHCi prompt loads, run `test` to confirm that everything is working:
    ```
    $ cabal repl

    Prelude MathBountyV2 :: ?> test
    ```


## Optional `direnv` Integration
The following steps are optional, but provide an enhanced workflow by loading the Nix environment automatically whenever you enter the project directory.

These steps can be skipped if you face any issues with `direnv` or if you prefer not to use this workflow. In this case, you'll need to manually load the Nix development environment from a (non-VS Code) terminal session (inside the project root directory), and then open VS Code from within the environment:
  ```bash
    $ nix develop
    $ code .
  ```

If you choose not to use `direnv`, you must open your project in VS Code this way each time in order for the Haskell development environment to work correctly and all of your dependencies to be available. The VS Code integrated `bash` terminal may also behave strangely with this method, due to some bugginess around opening a nested shell session when you're already inside the Nix shell. For these reasons the `direnv` approach is recommended.

10. Install `direnv`
  * **NOTE:** if using MacOS, make sure you have installed `bash` and `grep` via Homebrew as explained in the **Requirements** section above.
  * Run `direnv --version` in your terminal to check if `direnv` is installed and at least `v2.30`
    * If not, follow instructions to [install direnv](https://direnv.net/docs/installation.html) for your OS. **NOTE:** if a version >=2.30 isn't available for your OS (this is the case for Ubuntu 20.04 at the time of writing), or if you'd rather just install a compatible version of `direnv` using Nix, run the following command:

      ```bash
      $ nix-env -iA nixpkgs.direnv
      ```
  * Hook `direnv` into `BASH` by adding the following line at the end of your `~/.bashrc` file:

    ```bash
    eval "$(direnv hook bash)"
    ```
  * Be sure to add the recommended `direnv` VS Code extension (author Martin Kühl) if you haven't done so already.


11. Create an `.envrc` file in the project root directory and add the following lines:
  ```bash
  # Load `flake.nix` from the current directory:
  use flake .

  # Use nix-direnv for improved performance:
  if ! has nix_direnv_version || ! nix_direnv_version 2.1.1; then
  source_url "https://raw.githubusercontent.com/nix-community/nix-direnv/2.1.1/direnvrc" "sha256-b6qJ4r34rbE23yWjMqbmu3ia2z4b2wIlZUksBke/ol0="
  fi
  ```
  * The `direnv` VS Code extension will prompt you to `Allow` the changes whenever the `.envrc` file is modified (to prevent potential execution of malicious third-party code)
  * You can run `direnv allow` in your terminal at any time (make sure you are inside the project directory tree) to trigger the Nix environment to load if it doesn't do so automatically
  * With this configuration, whenever you open the project folder in VS Code the Nix environment will load automatically.

## HLS Troubleshooting
If you've completed all the steps above, you should have a functional Nix + Plutus development environment with HLS support in VS Code.

Some general troubleshooting steps if you have difficulties with Haskell Language Server misbehaving:
  * **Restart HLS:** Open the command palette (`F1`), type `LSP` into the prompt, and select `Haskell: Restart Haskell LSP server` (You can also set a custom keybinding for this for convenience - click the gear icon next to the command in the palette)
  * If restarting HLS doesn't fix the issue, try closing the project (`File > Close Folder`) and reopening it (`File > Open Folder`)
  * If you are using the `direnv` workflow and the environment doesn't load automatically when you open the project folder, you can run `direnv allow` to load it manually. You may also periodically see a popup from the `direnv` extension with a button that says `Restart`. Click this when you encounter it.
  * If you have HLS installed via GHCup and have the `Haskell: Manage HLS` setting for the Haskell extension set to `GHCup`, this setting may need to be overridden to look for HLS in the `PATH` instead (we want to ensure VS Code uses the correct version of HLS installed in the Nix environment). We've done this for you in this project (see the `/.vscode/settings.json` file). For future projects you may need to create this directory and file yourself.

## Cleaning Up Nix Store
You can free up storage space at any time by running the `nix-collect-garbage` command, which will delete unnecessary packages from the Nix store - however, this might cause you to have to re-fetch/build some dependencies the next time you load your environment, so it's recommended to run garbage collection only when you're finished working on a project or really need extra space.

`nix-collect-garbage -d` is a more intensive garbage collection command, which deletes everything that isn't used by the current generations of each profile.

[Read more about Nix garbage collection here.](https://nixos.org/manual/nix/stable/command-ref/nix-collect-garbage.html)

## Uninstalling Nix
Uninstalling Nix is a complex topic and the process is different depending on the OS and installation method (multi- vs. single-user).

We don't cover it in this guide, but here are some general tips if for some reason you wish to remove Nix completely:
  * If you repeat the installation process in **Step 2** with Nix already installed, it will provide instructions for removal.
  * Multi-user installations (the recommended install method) can be particularly complex to fully remove, so we recommend Googling the process for your specific OS. MacOS users may find [this post](https://iohk.zendesk.com/hc/en-us/articles/4415830650265-Uninstall-nix-on-MacOS) helpful.