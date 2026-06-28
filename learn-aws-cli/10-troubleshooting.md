# 10 — Troubleshooting Common Errors

## 1. "command not found"
```
-bash: aws: command not found
```
**Fix**: AWS CLI is not installed or not in PATH.
```
# Reinstall: see 01-what-is-aws-cli.md
# Or check if installed elsewhere:
find / -name aws -type f 2>/dev/null
which aws
```

## 2. Wrong version shown after install
```bash
# If aws --version shows an old version after installing v2:
# Remove old version and reinstall
which aws   # to find the old binary
sudo rm $(which aws)
# Then reinstall
```

## 3. Incomplete parameter name
```
Unknown options: --insta
```
**Fix**: You mistyped a parameter. Use tab-completion or `--help`:
```bash
aws ec2 describe-instances --help | grep -i instance
```

## 4. Access denied
```
An error occurred (AccessDenied) when calling the ListBuckets operation: ...
```
**Fix**: Your IAM user/role doesn't have the required permissions.
- Check your IAM policy
- Verify the profile (`--profile`) is correct
- Use `aws sts get-caller-identity` to confirm who you are

## 5. Invalid credentials
```
The security token included in the request is invalid
```
**Fix**: 
- Your access keys expired (temporary STS) or are incorrect
- Run `aws configure` again with valid keys
- Check environment variables like `AWS_ACCESS_KEY_ID` / `AWS_SESSION_TOKEN`
- Time/date is wrong on your machine (use `ntp`/`chrony` to sync)

## 6. SignatureDoesNotMatch
```
The request signature we calculated does not match the signature you provided
```
**Fix**:
- Your system clock is wrong (most common cause)
- `sudo ntpdate pool.ntp.org` or enable NTP
- Check `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` are correct

## 7. SSL certificate errors
```
SSL certificate verify failed
```
**Fix**:
```bash
# Bypass SSL verification (temporary, not for production)
export AWS_CA_BUNDLE=/path/to/cert-bundle.pem

# Or update your system certificates
sudo update-ca-certificates
```

## 8. Invalid JSON errors
```
Invalid base64: "invalid"
```
**Fix**:
- Make sure your JSON is valid (use `jq` to validate)
- Check quoting — in bash, use single quotes for JSON `'{"key": "val"}'`
- For binary data, use `fileb://` prefix (e.g., `--zip-file fileb://function.zip`)
- For text files, use `file://` prefix (e.g., `--policy-document file://policy.json`)

## 9. "The config profile (profile) could not be found"
```
The config profile (myprofile) could not be found
```
**Fix**:
```bash
# Check your config files
cat ~/.aws/config
cat ~/.aws/credentials

# Verify profile name matches
# In config: [profile myprofile]
# In credentials: [myprofile]
# CLI usage: --profile myprofile
```

## 10. Debug Mode — When All Else Fails

```bash
# Run with --debug to see exactly what's happening
aws s3 ls --debug 2>&1 | less

# Look for:
# - What credentials are being used
# - What API endpoint is being called
# - The actual HTTP request/response
# - Error details
```

## Quick Troubleshooting Checklist

| Symptom | Likely Cause | Quick Fix |
|---------|-------------|-----------|
| `command not found` | Not installed | Install or fix PATH |
| `AccessDenied` | Missing IAM permissions | Check policies |
| `Invalid token` | Expired/bad credentials | Re-run `aws configure` |
| `SignatureDoesNotMatch` | Clock skew | Sync system time |
| `SSL error` | Certificate issue | Set `AWS_CA_BUNDLE` |
| Config profile not found | Wrong profile name | Check `~/.aws/config` |

```bash
# Your quick health check:
aws sts get-caller-identity
aws s3 ls
echo "Time: $(date)"
```

---

### GUI Equivalent
**AWS Console** → IAM → Users for permission issues.  
**Support Center** for account-level issues.

### Next: [11 — Bash Scripting](./11-bash-scripting.md)
