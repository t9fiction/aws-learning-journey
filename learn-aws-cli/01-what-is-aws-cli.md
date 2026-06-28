# 01 — What is AWS CLI + Installation

## What is AWS CLI?

The AWS Command Line Interface (AWS CLI) is an **open-source tool** that lets you interact with AWS services from your terminal — same capabilities as the AWS Management Console (GUI), but faster and scriptable.

```
$ aws --version
aws-cli/2.27.41 Python/3.11.6 ...
```

**Why use CLI over Console?**
- Automate repetitive tasks
- Script infrastructure (Infrastructure as Code)
- Faster than clicking through the GUI
- Combine with CI/CD pipelines

**AWS CLI v2** (this guide) is the latest major version. It comes as a **bundled installer** (includes its own Python — no dependency conflicts).

## Installation

### Linux (x86_64)
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Linux (ARM)
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### macOS
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

Or use Homebrew:
```bash
brew install awscli
```

### Windows
Download the 64-bit installer from AWS:
```
https://awscli.amazonaws.com/AWSCLIV2.msi
```

### Verify installation
```bash
aws --version
```

### Update
```bash
# Re-download and re-run the installer (same as install)
# Or check for updates:
aws --version
```

### Uninstall
**Linux**: `sudo rm -rf /usr/local/aws-cli /usr/local/bin/aws`
**macOS**: `sudo rm -rf /usr/local/aws-cli /usr/local/bin/aws` (or use the uninstall script)
**Windows**: Uninstall from "Add or Remove Programs"

---

### GUI Equivalent
In the **AWS Management Console**, you do the same tasks but via browser-based forms and dashboards. The CLI mirrors every Console action.

### Next: [02 — Configuration & Setup](./02-configuration-setup.md)
