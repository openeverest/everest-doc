# Install everestctl

!!! warning "Not available in this release"
    `everestctl` is **not available** in the v2.0.0-dev.1 developer preview. Please use the [Helm-based installation](install_everest_helm_charts.md) instead.

You can download the latest version of `everestctl` visiting the latest release page in [this repository](https://github.com/openeverest/openeverest/releases/latest){:target="_blank"}.


=== "Homebrew (macOS and Linux)"

    The simplest way to install `everestctl` on macOS or Linux is via Homebrew. This works on both Apple Silicon and Intel CPUs.
    {.power-number}

    1. Add the OpenEverest tap:

        ```sh
        brew tap openeverest/tap
        ```

    2. Install `everestctl`:

        ```sh
        brew install everestctl
        ```

=== "Linux and WSL"

    To install `everestctl` on Linux or WSL, run the following commands:
    {.power-number}

    1. Download the latest version of `everestctl`.

        ```sh
        curl -sSL -o everestctl-linux-amd64 https://github.com/openeverest/openeverest/releases/latest/download/everestctl-linux-amd64
        ```

    2. Install `everestctl` binary to /usr/local/bin/.

        ```sh
        sudo install -m 555 everestctl-linux-amd64 /usr/local/bin/everestctl
        ```

    3. Remove `everestctl` binary to clean up.

        ```sh
        rm everestctl-linux-amd64
        ```         

=== "macOS (Apple Silicon)"

    To install `everestctl` on macOS with Apple Silicon, run the following commands:
    {.power-number}

    1. Download the latest version of `everestctl` for Apple Silicon.

        ```sh
        curl -sSL -o everestctl-darwin-arm64 https://github.com/openeverest/openeverest/releases/latest/download/everestctl-darwin-arm64
        ```

    2. Install `everestctl` binary to /usr/local/bin/.

        ```sh
        sudo install -m 555 everestctl-darwin-arm64 /usr/local/bin/everestctl
        ```
    3. Remove `everestctl` binary to clean up

        ```sh
        rm everestctl-darwin-arm64
        ```

=== "macOS (Intel CPU)"

    To install `everestctl` on macOS with an Intel CPU, run the following commands:
    {.power-number}


    1. Download the latest version of `everestctl` for Intel CPUs.

        ```sh
        curl -sSL -o everestctl-darwin-amd64 https://github.com/openeverest/openeverest/releases/latest/download/everestctl-darwin-amd64
        ```

    2. Install `everestctl` binary to /usr/local/bin/.

        ```sh
        sudo install -m 555 everestctl-darwin-amd64 /usr/local/bin/everestctl
        ```

    3. Remove `everestctl` to clean up.

        ```sh
        rm everestctl-darwin-amd64
        ```

## Next steps

Start by installing OpenEverest:

[Install OpenEverest :material-arrow-right:](../install/installEverest.md){.md-button}
