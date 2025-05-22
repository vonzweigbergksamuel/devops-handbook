# Setup SSH

- [Setup SSH](#setup-ssh)
  - [MacOS (Bash/Zsh)](#macos-bashzsh)
    - [Generate an SSH key](#generate-an-ssh-key)
    - [Add your SSH key to the ssh-agent](#add-your-ssh-key-to-the-ssh-agent)
  - [Windows (PowerShell)](#windows-powershell)
    - [Generate an SSH key](#generate-an-ssh-key-1)
    - [Add your SSH key to the ssh-agent](#add-your-ssh-key-to-the-ssh-agent-1)

<br>

## MacOS (Bash/Zsh)

### Generate an SSH key

1. Open your terminal.

<br>

2. Enter the following command:

Replace `some_comment` with a comment that will help you identify the key later. If used for GitHub, use your GitHub email address.
Replace `my_custom_keyname` with the desired name of the ssh-key.

```sh
ssh-keygen -t ed25519 -C "some_comment" -f ~/.ssh/my_custom_keyname
```

<br>

Optionally skip the `-C` flag to not add a comment and skip the `-f` flag to not add specific filename.

```sh
ssh-keygen -t ed25519
```

<br>

3. Press `Enter` to accept the default file location.

<br>

4. Enter a secure passphrase. This is optional, but recommended.

<br>

5. Your new SSH key will be generated and saved in the default location (`~/.ssh/your_key`).

<br>

### Add your SSH key to the ssh-agent

When adding your SSH key to the agent, use the default macOS `ssh-add` command, and not an application installed by macports, homebrew, or some other external source.

1. Start the ssh-agent in the background:

```sh
eval "$(ssh-agent -s)"
```

<br>

2. Make sure the ssh-directory has the correct permissions:

```sh
chmod 700 ~/.ssh
```

<br>

3. Open the config file:

```sh
nano ~/.ssh/config
```

If the file does not exist, using `nano` will create it.

<br>

4. Add the following lines to the top of the config file:

This will automatically load your SSH key into the ssh-agent when you open a terminal, so you don't have to run `ssh-add` every time.

```sh
# Global defaults
AddKeysToAgent yes
UseKeychain yes
```

<br>

5. Optionally, add a host-specific configuration to enable easy access

**For a server**

This will allow you to connect to your server by typing `ssh ubuntu-name` in the terminal. 

Replace `ubuntu-name`, `12.345.678.901`, `ubuntu`, and `~/.ssh/your-key-name` with your server's name, IP address, username, and key file location.

```sh
Host ubuntu-name
  HostName 12.345.678.901
  User ubuntu
  IdentityFile ~/.ssh/your-key-name
```

<br>

**For GitHub**

```sh
Host github.com
 IdentityFile ~/.ssh/your_key_name
```

<br>

6. Save and close the file by pressing `Ctrl + X`, then `Y`, and finally `Enter`.

<br>

7. Add your SSH key to the ssh-agent:

```sh
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

> [!NOTE]
> Replace `id_ed25519` with the name of your SSH key.
>
> The --apple-use-keychain option stores the passphrase in your keychain for you when you add an SSH key to the ssh-agent. If you chose not to add a passphrase to your key, run the command without the --apple-use-keychain option.

<br>

## Windows (PowerShell)

### Generate an SSH key

1. Open PowerShell.

<br>

2. Enter the following command:

Replace `some_comment` with a comment that will help you identify the key later. If used for GitHub, use your GitHub email address.

```sh
ssh-keygen -t ed25519 -C "some_comment"
```

<br>

Optionally skip the `-C` flag to not add a comment.

```sh
ssh-keygen -t ed25519
```

<br>

3. Press `Enter` to accept the default file location.

<br>

4. Enter a secure passphrase. This is optional, but recommended.

<br>

5. Your new SSH key will be generated and saved in the default location (`C:\Users\your_username\.ssh\your_key`).

<br>

### Add your SSH key to the ssh-agent

1. Open Powershell as an administrator.

<br>

2. Set the ssh-agent service to start automatically and start the service:

```sh
Set-Service ssh-agent -StartupType Automatic
Start-Service ssh-agent
```

<br>

3. Verify that the service is running:

```sh
Get-Service ssh-agent
```

<br>

4. Add your key to the ssh-agent:

Replace `your-key-name` with the name of your SSH key.

```sh
ssh-add C:\Users\your_username\.ssh\your-key-name
```

<br>
