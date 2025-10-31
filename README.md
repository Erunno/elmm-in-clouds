# Running ELMM with Charliecloud

This repository provides the necessary files and instructions to build and run the **ELMM** atmospheric model using Charliecloud on `gpulab`. This method creates a self-contained, reproducible environment, bypassing host-level library and compiler issues.

(Similar tutorial for RegCM: [github.com/Erunno/regcm-in-clouds](https://github.com/Erunno/regcm-in-clouds) ... you can also find a couple of useful notes there [github.com/Erunno/regcm-in-clouds/blob/main/notes.md](https://github.com/Erunno/regcm-in-clouds/blob/main/notes.md))

## ⚠️ Prerequisites

All commands must be run from a **worker node** on your cluster. You can get one by running `salloc` or prefixing your commands with `srun`.

---

## 1. Build the Environment Image

This step builds the container image defined in `Dockerfile.atmos`.

1. **Build the image:**
    This command builds the `Dockerfile.atmos` and tags the resulting image as `atmos`.
    ```bash
    ch-image build -t atmos -f Dockerfile.atmos .
    ```

2. **Convert the image:**
    This converts the image into a read-only directory named `imgdir`.
    ```bash
    ch-convert -i ch-image -o dir atmos imgdir
    ```

3. **Create a working directory:**
    This `mapped` directory will be your persistent workspace, linked inside the container.
    ```bash
    mkdir mapped
    ```

> **Troubleshooting:**
>
> * If the build fails, clearing the cache might help: `rm -rf /var/tmp/YOUR_LOGIN.ch`
> * If you need more libraries, add them to the `apt-get install` list in `Dockerfile.atmos` and rebuild.

---

## 2. Compile ELMM (One-Time Setup)

Now, you will enter the container to clone and compile the model.

1. **Connect to the container:**
    This links your `mapped` folder to `/opt/build` inside the container and starts a shell.
    ```bash
    ch-run -b mapped:/opt/build imgdir -- /bin/bash
    ```
    Your terminal prompt will change. You are now "inside" the container.

2. **Clone the source code:**
    (Inside the container)
    ```bash
    cd /opt/build
    git clone [https://github.com/LadaF/PoisFFT.git](https://github.com/LadaF/PoisFFT.git)
    git clone [https://LadaF@bitbucket.org/LadaF/elmm.git](https://LadaF@bitbucket.org/LadaF/elmm.git)
    ```

3. **Compile the Code:**
    (Inside the container)
    ```bash
    # Navigate to the source directory
    cd /opt/build/elmm/src

    # 1. Unset host compiler variables
    unset CC
    unset CXX

    # 2. Run the build
    ./make_release
    ```

    The build will complete, and the final executable will be saved in `/opt/build/elmm/bin/gcc/release/`.

    *(Alternatively, you can run this entire compile step non-interactively from the host):*
    ```bash
    ch-run -b mapped:/opt/build imgdir -- /bin/bash -c "cd /opt/build/elmm/src && unset CC && unset CXX && ./make_release"
    ```

4.  **Exit the container:**
    ```bash
    exit
    ```

---

## 3. Run a Simulation

Now you can run any simulation from the host machine.

1. **Copy the example case:**
    This command copies the `channel-simple` example into your mapped workspace.
    ```bash
    cp -r channel-simple mapped/elmm/examples/channel-simple
    ```

2.  **Run the executable:**
    Use `ch-run` to execute the compiled program *within* the container environment.
    ```bash
    ch-run -b mapped:/opt/build imgdir -- /bin/bash -c "cd /opt/build/elmm/examples/channel-simple && ../../bin/gcc/release/ELMM"
    ```

## 4. Get Your Output

All simulation data will be saved in the example directory (`mapped/elmm/examples/channel-simple/`), where you can access it directly from the host.
