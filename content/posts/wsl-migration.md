+++
title = 'Moving WSL to Another Drive'
date = 2023-12-12T02:42:25-05:00
draft = false
+++

Sometimes you need to move your WSL installation to another drive. Maybe you're running out of space on your current drive, or you just want to move it to a faster drive. Whatever the reason, it's simple as using several builtin [commands from WSL](https://github.com/LpCodes/Moving-WSL-Distribution-to-Another-Drive). For this example, I'll be moving my `Ubuntu-22.04` installation from my `C:` drive to my `D:` drive.

### Steps

1. List current installations in cmd or windows powershell

    ```bash
    wsl --list -v
    ```

    Output:

    ```powershell
    PS C:\Users\lim> wsl --list -v
      NAME                   STATE           VERSION
    * Ubuntu-22.04           Running         2
      docker-desktop         Stopped         2
      docker-desktop-data    Stopped         2
    ```

    Terminate the WSL instance you would like to remove. In this case, I'll be terminating `Ubuntu-22.04`.

    ```bash
    wsl -t Ubuntu-22.04
    ```

2. Export the WSL instance to a tar file using the [export command](https://learn.microsoft.com/en-us/windows/wsl/basic-commands#import-and-export-a-distribution). In this case, I'll be exporting `Ubuntu-22.04` to `D:\wsl_export\ubuntu-ex.tar`.

    Before that, create and `cd` into the directory you want to export the tar file to! I've tried using the aboslute path for the tar file, but it didn't work.

    ```powershell
    cd D:\wsl_export
    ```

    Then run the following command as a regular (non-admin) user.

    ```powershell
    wsl --export Ubuntu-22.04 .\ubuntu-ex.tar
    ```

    Output:

    ```powershell
    PS D:\wsl_export> wsl --export Ubuntu-22.04 .\ubuntu-ex.tar
    The operation completed successfully.
    ```

3. Unregister the old Ubuntu instance.

    ```bash
    wsl --unregister Ubuntu-22.04
    ```

4. Import the instance to a new location using the [import command](https://learn.microsoft.com/en-us/windows/wsl/basic-commands#import-and-export-a-distribution). In this case, I'll be importing `ubuntu-ex.tar` to `D:\wsl_import\ubuntu`. *(This time I'll be using the absolute path because it works for me, try `cd` into the directory you want to import the tar file to if it doesn't work.)*

    ```powershell
    wsl --import Ubuntu-22.04 "D:\wsl_import\ubuntu" "D:\wsl_export\ubuntu-ex.tar"
    ```

    Output:

    ```powershell
    PS D:\wsl_import> wsl --import Ubuntu-22.04 "D:\wsl_import\ubuntu"  "D:\wsl_export\ubuntu-ex.tar"
    Import in progress, this may take a few minutes.
    Windows Subsystem for Linux Distributions:
    docker-desktop (Default)
    Ubuntu-22.04
    docker-desktop-data
    ```

5. Set the new import as default for WSL

    ```powershell
    wsl --setdefault Ubuntu-22.04
    ```

6. Set default user for the new import

    I used the method 1 from [here](https://superuser.com/questions/1566022/how-to-set-default-user-for-manually-installed-wsl-distro) and modified the config in `/etc/wsl.conf` from my Ubuntu 22.04 installation.

    Start the Ubuntu 22.04 instance and run the following commands.

    ```bash
    vi /etc/wsl.conf
    ```

    Then modify the user section to:

    ```bash
    [user]
    default=<your-default-username>
    ```

    `:wq` to save and quit

    Then stop the instance from cmd or powershell.

    ```powershell
    wsl --terminate Ubuntu-22.04
    ```

    When you restart the instance, it should be using the default user you set.

    \* *Feel free to try out the [other methods](https://superuser.com/questions/1566022/how-to-set-default-user-for-manually-installed-wsl-distro) if this doesn't work for you.*

### Voila!

Now you have successfully moved your WSL installation to another drive. Happy coding!
