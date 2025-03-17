# RH Ldap Test

## 0. Install Podman & Podman Compose

Make sure Podman and Podman Compose are installed:

```bash
sudo dnf install -y podman podman-compose
```

Check Podman works:

```bash
podman --version
```

Ensure rootless Podman is set up:

```bash
podman system migrate
```


## 1. Docker Compose File

This `docker-compose.yml` will spin up an OpenLDAP server with `dc=test,dc=redhat,dc=com`:

```yml
version: '3.8'

services:
  openldap:
    image: bitnami/openldap:latest
    container_name: openldap
    restart: always
    environment:
      - LDAP_ADMIN_USERNAME=admin
      - LDAP_ADMIN_PASSWORD=adminpassword
      - LDAP_ROOT=dc=test,dc=redhat,dc=com
      - LDAP_ORGANIZATION=Test Organization
      - LDAP_ALLOW_ANON_BINDING=yes
    ports:
      - "389:1389"
      - "636:1636"
    volumes:
      - ./ldif:/ldif  # Mount local directory to container for LDIF files
      - openldap_data:/bitnami/openldap
  
volumes:
  openldap_data:
```

This will create an OpenLDAP server with:

- Base DN: dc=test,dc=redhat,dc=com
- Admin DN: cn=admin,dc=test,dc=redhat,dc=com
- Password: adminpassword
- Anonymous binding enabled for easy testing

---

### 1.1. Start OpenLDAP using Podman Compose

Run the following command:

```bash
podman volume create openldap_data
podman-compose -f podman-compose.yml up -d
```

To check the running container:

```bash
podman ps
```

To stop the container:

```bash
podman-compose -f podman-compose.yml down
```

---

### 1.2. Creating a Password for Users

You'll want to update the passwords for users to something known, so you can actually login. If you have `slappasswd` installed you can use it to generate new passwords:

```bash
slappasswd -h {SSHA}
```

You will be prompted to enter a password, and will be returned a string like this:

```
{SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=
```

Use this value in your **LDIF files** for the `userPassword` attribute.

If you don’t have slappasswd, you can use Python to generate an SSHA password:

```py
import hashlib
import base64
import os

def generate_ssha_password(password):
    salt = os.urandom(4)  # Generate a 4-byte salt
    sha1_hash = hashlib.sha1(password.encode('utf-8'))
    sha1_hash.update(salt)
    return "{SSHA}" + base64.b64encode(sha1_hash.digest() + salt).decode('utf-8')

# Generate and print an SSHA password
new_password = generate_ssha_password("NewSecret123")
print(new_password)
```

This will output something like:

```
{SSHA}K4x0H2I6e7xP77lnFjwJZRt6bGZf
```

## 2. LDIF Files for Organizational Units & Users

We'll create three user groups (buckets) and assign users to them.

### 2.1 Base Structure (Base DN & OUs)

Create a file named `01_base.ldif` inside the `ldif` folder:

```
dn: dc=test,dc=redhat,dc=com
objectClass: top
objectClass: domain
dc: test

dn: ou=Users,dc=test,dc=redhat,dc=com
objectClass: organizationalUnit
ou: Users

dn: ou=Bucket1,ou=Users,dc=test,dc=redhat,dc=com
objectClass: organizationalUnit
ou: Bucket1

dn: ou=Bucket2,ou=Users,dc=test,dc=redhat,dc=com
objectClass: organizationalUnit
ou: Bucket2

dn: ou=Bucket3,ou=Users,dc=test,dc=redhat,dc=com
objectClass: organizationalUnit
ou: Bucket3
```

---

### 2.2 User Entries

Create a file named `02_users.ldif`:

```
# Bucket 1 Users
dn: uid=user1,ou=Bucket1,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User One
sn: One
uid: user1
uidNumber: 1001
gidNumber: 1001
homeDirectory: /home/user1
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=

dn: uid=user2,ou=Bucket1,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User Two
sn: Two
uid: user2
uidNumber: 1002
gidNumber: 1002
homeDirectory: /home/user2
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=

# Bucket 2 Users
dn: uid=user3,ou=Bucket2,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User Three
sn: Three
uid: user3
uidNumber: 2001
gidNumber: 2001
homeDirectory: /home/user3
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=

dn: uid=user4,ou=Bucket2,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User Four
sn: Four
uid: user4
uidNumber: 2002
gidNumber: 2002
homeDirectory: /home/user4
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=

# Bucket 3 Users
dn: uid=user5,ou=Bucket3,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User Five
sn: Five
uid: user5
uidNumber: 3001
gidNumber: 3001
homeDirectory: /home/user5
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=

dn: uid=user6,ou=Bucket3,ou=Users,dc=test,dc=redhat,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
cn: User Six
sn: Six
uid: user6
uidNumber: 3002
gidNumber: 3002
homeDirectory: /home/user6
userPassword: {SSHA}hUO/oDoFjJydEhTkXp3XK3dL6rc=
```

---

## 3. Load LDIF Files into OpenLDAP

After starting the container, run:

```bash
docker exec -it openldap ldapadd -x -D "cn=admin,dc=test,dc=redhat,dc=com" -w adminpassword -f /ldif/01_base.ldif
docker exec -it openldap ldapadd -x -D "cn=admin,dc=test,dc=redhat,dc=com" -w adminpassword -f /ldif/02_users.ldif
```

---

## 4. Verify the LDAP Entries

Run an LDAP search to verify the users:

```bash
ldapsearch -x -H ldap://localhost -b "ou=Users,dc=test,dc=redhat,dc=com" -D "cn=admin,dc=test,dc=redhat,dc=com" -w adminpassword
```

To filter by a specific bucket:

```bash
ldapsearch -x -H ldap://localhost -b "ou=Bucket1,ou=Users,dc=test,dc=redhat,dc=com" -D "cn=admin,dc=test,dc=redhat,dc=com" -w adminpassword
```

---

## 5. Django LDAP Configuration

To configure Django to authenticate using LDAP, use `django-auth-ldap`:

```py
import ldap
from django_auth_ldap.config import LDAPSearchUnion, LDAPSearch

AUTH_LDAP_SERVER_URI = "ldap://localhost"

# Define multiple search filters for different buckets
AUTH_LDAP_USER_SEARCH = LDAPSearchUnion(
    LDAPSearch("ou=Bucket1,ou=Users,dc=test,dc=redhat,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)"),
    LDAPSearch("ou=Bucket2,ou=Users,dc=test,dc=redhat,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)"),
    LDAPSearch("ou=Bucket3,ou=Users,dc=test,dc=redhat,dc=com", ldap.SCOPE_SUBTREE, "(uid=%(user)s)")
)

AUTH_LDAP_BIND_DN = "cn=admin,dc=test,dc=redhat,dc=com"
AUTH_LDAP_BIND_PASSWORD = "adminpassword"
AUTH_LDAP_USER_DN_TEMPLATE = "uid=%(user)s,ou=Users,dc=test,dc=redhat,dc=com"

# Enable user creation
AUTH_LDAP_USER_ATTR_MAP = {
    "first_name": "cn",
    "last_name": "sn",
    "email": "mail"
}

AUTHENTICATION_BACKENDS = [
    "django_auth_ldap.backend.LDAPBackend",
    "django.contrib.auth.backends.ModelBackend",
]
```

