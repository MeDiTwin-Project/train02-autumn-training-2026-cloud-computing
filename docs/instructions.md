# MediTwin Autumn Training 2026: Prerequisites and Access Instructions

Please complete the checks in this document **before the training starts**. Bring the same laptop that you will use during the practical sessions.

> **Training dates:** 2-6 November 2026  
> **Venue:** National Technical University of Athens (NTUA), Athens, Greece

## 1. Participant registration

Confirm that your name and current email address are present in the [MediTwin Autumn Training 2026 participant registration spreadsheet](https://docs.google.com/spreadsheets/d/1IPgRsQvggd7KgBsyOZTfqiKjqjRP7mYNyR5seWxhc8Q/edit?gid=208717898#gid=208717898).

The organisers will use the registration information to assign one training environment to each participant and to send individual access details.

> Do not add training usernames, claim codes, SSH private keys, passwords, or object-storage credentials to the registration spreadsheet.

## 2. Install Visual Studio Code and open these instructions

The training instructions are distributed as a Markdown file.

1. Install [Visual Studio Code](https://code.visualstudio.com/).
2. Start Visual Studio Code.
3. Open this `.md` instructions file.
4. Open the rendered Markdown preview:
   - on Windows or Linux, press `Ctrl+Shift+V`;
   - on macOS, press `Cmd+Shift+V`.
5. Keep the rendered instructions open while completing the remaining setup steps.

## 3. Individual training credentials

Each participant will receive an individual e-mail containing:

- a trainee username, such as `trainee00`;
- a claim code;
- the name of the assigned virtual machine;
- the virtual machine public IP address.

Keep the claim code private. The same trainee username and claim code are used for the training access services, including the credential portal, GitLab and Rancher.

## 4. Claim your SSH key

1. Open the [MediTwin training credential portal](https://ssh-keys.training.meditwin-project.eu/).
2. Sign in with the trainee username and claim code sent to you.
3. Verify that the displayed virtual machine name and IP address match the details in your e-mail.
4. Download your SSH private key.
5. Store the file in a private location on your laptop.

> **Security:** Never share your claim code or SSH private key. Do not upload them to Git, Google Drive, OneDrive, Teams, Discord, or any shared folder.

## 5. Test SSH access to your virtual machine

Replace the placeholders below with the downloaded key filename and your assigned VM IP address.

### Linux or macOS

Open a terminal and run:

```bash
mkdir -p ~/.ssh
mv ~/Downloads/<downloaded-key-file> ~/.ssh/meditwin-training-2026.pem
chmod 600 ~/.ssh/meditwin-training-2026.pem
ssh -i ~/.ssh/meditwin-training-2026.pem cloud-user@<vm-ip-address>
```

### Windows 10 or Windows 11 with OpenSSH

Open **PowerShell** and run:

```powershell
New-Item -ItemType Directory -Force "$HOME\.ssh" | Out-Null
Move-Item "$HOME\Downloads\<downloaded-key-file>" "$HOME\.ssh\meditwin-training-2026.pem"
ssh -i "$HOME\.ssh\meditwin-training-2026.pem" cloud-user@<vm-ip-address>
```

At the first connection, verify the IP address and accept the SSH host-key prompt.

A successful test leaves you at a shell prompt on the assigned VM.

## 6. Install the Visual Studio Code Remote - SSH extension

1. Install the [Visual Studio Code Remote - SSH extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh).

2. Add the following entry to your SSH configuration file:

```sshconfig
Host meditwin-training
    HostName <vm-ip-address>
    User cloud-user
    IdentityFile ~/.ssh/meditwin-training-2026.pem
    IdentitiesOnly yes
```

On Windows, the SSH configuration file is normally:

```text
%USERPROFILE%\.ssh\config
```

On Linux and macOS, it is normally:

```text
~/.ssh/config
```

3. Test the configuration from a terminal:

```bash
ssh meditwin-training
```

4. In Visual Studio Code, open the Command Palette:
   - on Windows or Linux, press `Ctrl+Shift+P`;
   - on macOS, press `Cmd+Shift+P`.
5. Enter `Remote-SSH: Connect to Host...` and select the command.
6. Select `meditwin-training`.
7. Confirm that Visual Studio Code successfully opens a remote session on the training VM.

## 7. Test GitLab access

1. Open [MediTwin GitLab](https://gitlab.data.meditwin-project.eu/).
2. Sign in with your trainee username and claim code.
3. Confirm that you can access the `training / autumn-2026` area.

## 8. Test Rancher and Kubernetes access

1. Open [MediTwin training Rancher](https://rancher-training.apps.meditwin-project.eu/).
2. Sign in with your trainee username and claim code.
3. Confirm that the training cluster is visible.

## 9. Laptop requirements

Bring a laptop on which you can install or use the following:

- a current web browser;
- an SSH client;
- Visual Studio Code;
- the Visual Studio Code Remote - SSH extension;
- Git;
- an internet connection suitable for SSH and HTTPS access.

The practical workloads run on the assigned training VM and Kubernetes environment. Your laptop is used mainly as the access workstation.

## 10. Final readiness checklist

Before travelling to the school, confirm that you can complete every item below:

- [ ] My name and email address are correct in the participant registration spreadsheet.
- [ ] I installed Visual Studio Code.
- [ ] I opened this `.md` file in Visual Studio Code.
- [ ] I opened the Markdown preview using `Ctrl+Shift+V` on Windows/Linux or `Cmd+Shift+V` on macOS.
- [ ] I received my trainee username, claim code, VM name and VM IP address.
- [ ] I signed in to the credential portal.
- [ ] I downloaded my SSH private key.
- [ ] I connected to my VM as `cloud-user` over SSH.
- [ ] I installed the Visual Studio Code Remote - SSH extension.
- [ ] I connected to the VM through Visual Studio Code Remote - SSH.
- [ ] I signed in to MediTwin GitLab.
- [ ] I signed in to the training Rancher interface.

## 11. Troubleshooting information to send to the training team

If a check fails, send the training team:

- your trainee username, but **not** the claim code;
- the step that failed;
- the exact error message;
- your operating system;
- whether the problem occurs in the browser, terminal, or Visual Studio Code;
- the approximate time of the failed attempt.

Do not send private keys, claim codes, passwords or kubeconfig files in support messages.