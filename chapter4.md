# 🔐 AWS IAM — Short Notes

### 👤 What is IAM?

> **IAM (Identity and Access Management)** controls **WHO can access WHAT** in AWS.

```text
User → IAM → Permission → AWS Resource
```

---

### 👥 IAM Main Components

**1️⃣ User**

> Represents a person/application that needs AWS access.

**2️⃣ Group**

> Collection of users with similar permissions.

```text
Developers Group
   ↓
User 1
User 2
User 3
```

**3️⃣ Policy**

> A **JSON document** that defines what actions are allowed or denied.

Example:

```text
Allow → S3 Read
Deny  → S3 Delete
```

---

### 🔑 AWS Access Methods

* 🖥️ **Management Console** → Web browser
* ☁️ **CloudShell** → AWS browser-based terminal
* ⌨️ **AWS CLI** → Use AWS from command line

---

### 🛡️ Security Best Practices

🔒 **Enable MFA on Root User**

> Password + MFA = Extra security

⚠️ **Avoid using the Root User for everyday tasks.**

---

### 📊 IAM Auditing

**Credential Report**
→ Shows information about users and their credentials.

**Last Accessed**
→ Shows which AWS services a user/role has accessed.

---

### 🧠 Remember

> **IAM = Who can do What** 🔐
> **User = Person** 👤
> **Group = Collection of Users** 👥
> **Policy = Permissions** 📜
> **MFA = Extra Security** 🛡️
> **CLI = AWS through Terminal** ⌨️
