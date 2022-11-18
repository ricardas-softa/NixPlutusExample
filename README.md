# Plutus Development with Nix
This repository contains a sample Plutus project and a guided walkthrough for converting it to a Nix project (using Flakes, `haskell.nix` and `direnv`) for enhanced reproducibility and development experience.

* **NOTE:** this walkthrough assumes the use of `VS Code` as editor and `bash` as SHELL. Other tools will require alternative workflows that are not covered here.

## Instructions:
1. Fork and clone this repository.

2. Check/install dependencies
  * Run `direnv --version` in terminal to check if `direnv` is installed.
    * If not, follow instructions to [install direnv](https://direnv.net/docs/installation.html) for your OS
    * Ubuntu example:

      ```bash
      $ sudo apt-get update
      $ sudo apt-get install direnv
      ```
  * MacOS users should also install `bash`/`grep` using Homebrew before proceeding
  * Hook `direnv` into `BASH` by adding the following line at the end of your `~/.bashrc` file:

    ```bash
    eval "$(direnv hook bash)"
    ```
  * Add the `Nix Extension Pack` VS Code extension

3. [Install Nix](https://nixos.org/download.html) package manager for your OS
  * Follow the instructions for multi-user installation at the link above
  * Then follow the prompts in your terminal

4. Install `nix-prefetch-git`
  * We'll need to calculate SHA-256 hashes for Plutus dependencies using the `nix-prefetch-git` package. This can be installed system-wide via Nix:
    ```bash
    $ nix-env -iA nixos.nix-prefetch-git
    ```
  * If you'd rather not install this, we can use `nix-prefetch-git` transiently in a nix-shell instead. Follow the instructions in **Step 8**.

5. Enable Flakes & Configure Binary Cache:
  * Edit either `~/.config/nix/nix.conf` or `/etc/nix/nix.conf` (depending on how you installed Nix in Step 3)
  * Add the following line to enable Flakes:

    ```
    experimental-features = nix-command flakes
    ```
  * Add the following lines to set up the binary cache:

    ```
    trusted-public-keys = [...] hydra.iohk.io:f/Ea+s+dFdN+3Y/G+FDgSq+a5NEWhJGzdjvKNGv0/EQ= [...]
    substituters = [...] https://cache.iog.io [...]
    ```

6. Create `flake.nix` in project root directory and add flake "scaffolding" code from [haskell.nix](https://input-output-hk.github.io/haskell.nix/tutorials/getting-started-flakes.html#scaffolding).

7. Modify `flake.nix` template:
  * In the `helloProject` overlay, set `compiler-nix-name` to `"ghc8107"` and `hlint` version in `shelltools` to `"3.4.1"` (the latest version of `hlint` is incompatible with GHC 8.x).
  * Comment out the `packages.default` line at the bottom, as this sample project is a library project with no executables.
    * For projects with executables, a default executable can be specified here in the format `"<CABAL PROJECT NAME>:exe:<EXECUTABLE NAME>"`. The `nix run` command will then run this executable.

8. Add missing SHA-256 hashes for dependencies in `cabal.project`.
  * If you attempt to load the Nix shell now with the `nix develop` command, you will receive an error.
  * Since Nix Flakes require pure inputs to guarantee reproducibility, and the content associated with a particular Git repository and tag can change, we need to hash any repositories we include in the `cabal.project` file.
  * We can calculate SHA-256 hashes for dependencies using the `nix-prefetch-git` package. This can be installed system-wide via Nix:

    ```bash
    $ nix-env -iA nixos.nix-prefetch-git
    ```

    or, alternatively we can just load a transient Nix shell with the package installed, and perform the hashing in this shell session:

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



8. Create `.envrc` file and add the following lines:
  ```bash
  # Load `flake.nix` from the current directory:
  use flake .

  # Use nix-direnv for improved performance:
  if ! has nix_direnv_version || ! nix_direnv_version 2.1.1; then
  source_url "https://raw.githubusercontent.com/nix-community/nix-direnv/2.1.1/direnvrc" "sha256-b6qJ4r34rbE23yWjMqbmu3ia2z4b2wIlZUksBke/ol0="
  fi
  ```
  * The `direnv` VS Code extension will prompt you to `Allow` the changes whenenver the `.envrc` file is modified (to prevent potential execution of malicious third-party code)