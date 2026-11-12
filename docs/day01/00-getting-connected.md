# 0. Getting connected

Everything practical this week happens on machines in the cloud: your own **training VM**, the
training **GitLab**, and a shared **Kubernetes** cluster managed through **Rancher**. Your
laptop is mainly the access workstation. This section gets you connected to all three, so
that the rest of the day can focus on the cloud itself.

If you already completed the pre-training instructions sent by e-mail, use this section as a
quick check and move on.

## 0.1 Your training credentials

Each participant receives an individual e-mail containing:

- a **trainee username**, such as `trainee00`;
- a **claim code**;
- the **name** of your assigned virtual machine;
- the **public IP address** of that virtual machine.

The same trainee username and claim code are used for all the training services: the
credential portal, GitLab and Rancher.

:::{danger} Keep your credentials private
Never share your claim code or your SSH private key. Do not upload them to Git, Google Drive,
OneDrive, Teams, Discord or any shared folder, and do not paste them into chats or support
messages. Section 7 explains why credentials deserve this care.
:::

## 0.2 Claim your SSH key

Your VM has no password login: you log in with an **SSH key** (section 4 shows how the cloud
put the matching public key into the VM).

1. Open the [MediTwin training credential portal](https://ssh-keys.training.meditwin-project.eu/).
2. Sign in with your trainee username and claim code.
3. Check that the displayed virtual machine name and IP address match the details in your
   e-mail.
4. Download your **SSH private key** and store the file in a private location on your laptop.

The portal also shows the details of your **S3 object storage** access, which you use from
section 5 onwards. On your training VM they are already configured for you.

## 0.3 Log in to your VM with SSH

Replace the placeholders below with the downloaded key filename and your VM's IP address. The
username on the VM is **`cloud-user`** for everyone (it is the default user of the Rocky Linux
cloud image your VM was created from; you will meet this idea again in section 4).

**Linux or macOS**, in a terminal:

```bash
mkdir -p ~/.ssh
mv ~/Downloads/<downloaded-key-file> ~/.ssh/meditwin-training-2026.pem
chmod 600 ~/.ssh/meditwin-training-2026.pem
ssh -i ~/.ssh/meditwin-training-2026.pem cloud-user@<vm-ip-address>
```

**Windows 10 or 11** (OpenSSH), in **PowerShell**:

```powershell
New-Item -ItemType Directory -Force "$HOME\.ssh" | Out-Null
Move-Item "$HOME\Downloads\<downloaded-key-file>" "$HOME\.ssh\meditwin-training-2026.pem"
ssh -i "$HOME\.ssh\meditwin-training-2026.pem" cloud-user@<vm-ip-address>
```

At the first connection, SSH asks you to confirm the server's **host key**. Check that the IP
address is the one from your e-mail, and accept. A successful login leaves you at a shell
prompt on your VM:

```text
[cloud-user@<vm-name> ~]$
```

:::{tip} `chmod 600`, and "UNPROTECTED PRIVATE KEY FILE"
SSH refuses to use a private key that other users of your laptop could read. If you see
`WARNING: UNPROTECTED PRIVATE KEY FILE!`, re-run the `chmod 600` line (on Linux/macOS).
:::

## 0.4 An SSH shortcut, and Visual Studio Code

Typing the key path and IP address every time gets old quickly. Add an entry to your SSH
configuration file (`~/.ssh/config` on Linux and macOS, `%USERPROFILE%\.ssh\config` on
Windows; create the file if it does not exist):

```text
Host meditwin-training
    HostName <vm-ip-address>
    User cloud-user
    IdentityFile ~/.ssh/meditwin-training-2026.pem
    IdentitiesOnly yes
```

From now on, this is all you need:

```bash
ssh meditwin-training
```

The rest of the material writes `ssh meditwin-training` whenever you should log in to your
VM.

To edit files on the VM comfortably, use **Visual Studio Code** with the
[Remote - SSH extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh):

1. Install [Visual Studio Code](https://code.visualstudio.com/) and the Remote - SSH extension.
2. Open the Command Palette (`Ctrl+Shift+P` on Windows/Linux, `Cmd+Shift+P` on macOS).
3. Run **Remote-SSH: Connect to Host...** and select `meditwin-training`.
4. A new window opens, connected to the VM. Its built-in terminal
   (**Terminal → New Terminal**) is a shell on the VM, and its file explorer shows the VM's
   files.

## 0.5 GitLab and Rancher

Two more services you will use later in the week. Sign in to each now, so that any access
problem surfaces today rather than on Day 3 or 4:

| Service | Sign in with | Check that |
|---|---|---|
| [MediTwin GitLab](https://gitlab.data.meditwin-project.eu/) | trainee username + claim code | you can open the `training / autumn-2026` group |
| [MediTwin training Rancher](https://rancher-training.apps.meditwin-project.eu/) | trainee username + claim code | the training cluster is visible |

The training material itself is published at
[meditwin-autumn-training-2026](https://meditwin-autumn-training-2026-b0970d.pages.data.meditwin-project.eu/).

## Readiness checklist

- [ ] I signed in to the credential portal and downloaded my SSH private key.
- [ ] I logged in to my VM as `cloud-user` over SSH.
- [ ] `ssh meditwin-training` works.
- [ ] Visual Studio Code opens a Remote - SSH session on the VM.
- [ ] I signed in to MediTwin GitLab.
- [ ] I signed in to the training Rancher.

## If something does not work

Raise your hand, or send the training team:

- your trainee username, but **not** the claim code;
- the step that failed and the exact error message;
- your operating system;
- whether the problem occurs in the browser, the terminal or Visual Studio Code.

Never send private keys, claim codes, passwords or kubeconfig files in support messages.

---

Next: [Virtualization](01-virtualization.md), where the story of the day begins.
