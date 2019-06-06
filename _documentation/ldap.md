---
title: LDAP
description: How to use an LDAP directory server with Kimai 2
toc: true
since_version: 1.0
---

Kimai supports authentication viayour companies directory server (LDAP or AD). 
LDAP users will be imported during the first login and their attributes and groups 
updated on each following login. 

## Configuration

If you want to activate LDAP authentication, you have to adjust your [local.yaml]({% link _documentation/configurations.md %}).

This is the full available configuration, most of the values are optional and their default values were carefully chosen for maximum compatibility:

```yaml
kimai:
    ldap:
        # whether LDAP authentication should be used
        active: true                            # default: false
        
        # more infos about the connection params can be found at:
        # https://docs.zendframework.com/zend-ldap/api/
        connection:
            # The default hostname of the LDAP server 
            # this option is mandatory
            host: 127.0.0.1
            # Default port for your LDAP port server
            # default: 389
            port: 389
            # Whether or not the LDAP client should use SSL encrypted transport. 
            # The useSsl and useStartTls options are mutually exclusive.
            # default: false
            useSsl: false
            # Enable TLS negotiation (should be favoured over useSsl).
            # The useSsl and useStartTls options are mutually exclusive.
            # default: false
            useStartTls: false
            # the default credentials username. Some servers require that this 
            # be in DN form. It must be given in DN form if the LDAP server 
            # requires a DN to bind and binding should be possible with simple 
            # usernames, default: empty
            username:
            # default credentials password (for username above), default: empty
            password:
            # If true, this instructs Kimai to retrieve the DN for the account, 
            # used to bind if the username is not already in DN form. 
            # default: false
            bindRequiresDn: false 
            # If set to true, this option indicates to the LDAP client that 
            # referrals should be followed, default: false
            #optReferrals: false
            # for the next options please refer to
            # see https://docs.zendframework.com/zend-ldap/api/ 
            #allowEmptyPassword: false
            #tryUsernameSplit: 
            #networkTimeout: 
            #accountCanonicalForm: 3
            #accountDomainName: HOST
            #accountDomainNameShort: HOST

        user:
            # baseDn to query for users
            # this value is mandatory
            baseDn: ou=users, dc=kimai, dc=org
            # field used to match the username with your LDAP users
            # if "bindRequiresDn: false" is set it is used in "bind", otherwise
            # it is used to search for the user and fetch its "dn" for the "bind".
            # defaults to "uid" 
            usernameAttribute: uid
            # search filter for users, used to retrieve the user (users DN) by username.
            # the result of the search filter must return 1 result only.
            # The substring %s will be replaced with the username.
            # default: (&(usernameAttribute=%s))
            # default, if usernameAttribute is not changed: (&(uid=%s))
            filter: (&(objectClass=inetOrgPerson)(uid=%s))

            # Configure the mapping between LDAP attributes and user entity
            attributes:
                # this rule is automatically prepended and can be overwritten
                # username is set to the value of the configured 
                # "usernameAttribute" field (see above)
                - { ldap_attr: "usernameAttribute", user_method: setUsername }
                # only applied if you don't configure a mapping for setEmail()
                - { ldap_attr: "usernameAttribute", user_method: setEmail }
                # an example which will set the display name in Kimai from the 
                # value of the "common name" field in your LDAP
                - { ldap_attr: cn, user_method: setAlias }

        # You can comment the following section, if you don't want to manage
        # user roles in Kimai via LDAP groups. If you want to user the group
        # sync, you have to set at least the "role.baseDn" config
        # default: deactivated as baseDn "role.baseDn" is empty by default
        role:
            # baseDn to query for groups, MUST be set to activate the 
            # "group import" feature
            # default: empty (deactivated)
            baseDn: ou=groups, dc=kimai, dc=org
            # filter to query user groups, all results will be matched against 
            # the configured "groups" mapping below.
            # default: empty
            # The filter will ALWAYS be generated like this:
            # (&%filter(userDnAttribute=valueOfUsernameAttribute)) 
            # the following example rule will be expanded to:
            # (&(&(objectClass=groupOfNames))(member=valueOfUsernameAttribute))
            filter: (&(objectClass=groupOfNames))  
            # the following field is taken from the LDAP user entry and its 
            # value is used in the filter above as "valueOfUsernameAttribute"
            # the example below uses "posix group style memberUid". 
            # default: dn
            usernameAttribute: uid
            # field that holds the group name, which will be used to map the 
            # found groups with Kimai roles (see groups mapping below)
            # default: cn
            nameAttribute: cn
            # field that holds the users dn in your LDAP group definition.
            # value of this configuration is used in the filter (see above)
            # default: member
            userDnAttribute: member
            # Convert LDAP group name (nameAttribute) to Kimai role
            #groups:
            #    - { ldap_value: group1, role: ROLE_TEAMLEAD }
            #    - { ldap_value: kimai_admin, role: ROLE_ADMIN }
``` 

Kimai uses the Zend Framework LDAP module and uses the configured `connection` parameters without modification. 
Find out more about the settings in the [detailed documentation](https://docs.zendframework.com/zend-ldap/api/). 

## User synchronization

User data is synchronized on each login, fetching the latest data from your LDAP.

**How it works**

- if `bindRequiredDn` is active, a `search` is executed to find the users DN by the given username
- the authentication is checked with a `bind`
- if the `bind` was successful:
    - another `bind` using the service account (connection.username/connection.password) is executed and under that scope:
      - a `search` is executed to find and map LDAP attributes to the Kimai profile
      - if configured, another `search` is executed to sync and map the users LDAP groups to Kimai roles 

**Password handling**

Obviously Kimai does not store the users password when logged-in via LDAP and there is 
no fallback mechanism implemented, if your LDAP is not available (currently only ONE server can be configured).

{% include alert.html type="danger" alert="The default configuration allows a user to change the internal password. This manually chosen password is not overwritten by the LDAP plugin and would allow a user to login, even after you removed him from LDAP." %} 

To prevent that problem:
- disable the "[Password reset]({% link _documentation/users.md %})" function
```yaml
kimai:
    user:
        registration: false
        password_reset: false
```
- disable the "change my own password" permission for each role:
```yaml
    permissions:
        roles:
            ROLE_USER: ['!password_own_profile']
            ROLE_TEAMLEAD: ['!password_own_profile']
            ROLE_ADMIN: ['!password_own_profile']
```

Read more about `password_own_profile` and `password_other_profile` [permissions]({% link _documentation/permissions.md %}).

If you don't adjust your configuration, you have to:
- either deactivate users manually in Kimai after deleting their LDAP account
- or use a attribute mapping to set the user deactivated flag via `setEnabled()`

### User attributes

Kimai does not rely on an `objectClass`, but maps single LDAP attributes to the User entity by configuration.

An example could look like this:

```yaml
kimai:
    ldap:
        user:
            attributes:
                - { ldap_attr: uid, user_method: setUsername }
                - { ldap_attr: mail, user_method: setEmail }
                - { ldap_attr: cn, user_method: setAlias }
```
{% include alert.html type="warning" alert="You need to configure the attributes in lower-case, otherwise they won't be processed." %}

In this example we tell Kimai to sync the following fields:
- `uid` will be the username in Kimai (will fail with a 500 if not unique)
- `mail` will be the account email address (read "known limitations" below)
- `cn` will be used for the display name in Kimai

Available methods on the User entity are: `setUsername(string)`, `setEmail(string)`, `setAlias(string)`, `setAvatar(url)`, `setTitle(string)`.
Its unlikely that you will need those, but they also exist: `setEnabled(bool)`, `setSuperAdmin(bool)`, `addRole(string)`.

### Groups / Roles import

Kimai can use your LDAP groups and map them to [user roles]({% link _documentation/users.md %}).
If configured, it will execute another `search` against your LDAP after authentication and importing the user attributes.

{% include alert.html type="warning" alert="Every user automatically owns the ROLE_USER role, you don't have to create a mapping for it." %}

Assuming this `role` configuration:
```yaml
kimai:
    ldap:
        role:
            baseDn: ou=groups, dc=kimai, dc=org
            #filter: (&(objectClass=groupOfNames))  # additional group filter
            #userDnAttribute: member                # field to lookup the users
            #nameAttribute: cn                      # group name to match
            groups:
                - { ldap_value: group1, role: ROLE_TEAMLEAD }
                - { ldap_value: kimai_admin, role: ROLE_ADMIN }
                - { ldap_value: administrator, role: ROLE_SUPER_ADMIN }
```

Kimai will search the `baseDn` with `userDnAttribute=user['dn']` (e.g. `member=uid=user1,ou=users,dc=kimai,dc=org`) and extract the 
group names from the result-sets attribute `nameAttribute`. 

After finding a list of group names, they will be converted to Kimai roles:
- first step is to lookup in `groups` mapping, if there is a match in `ldap_value` and uses the `role` value without further processing 
- if no mapping was found, the group name will be UPPERCASED and prefixed with `ROLE_` => e.g. `admin` will become `ROLE_ADMIN`

These converted names will validated and [only existing roles]({% link _documentation/users.md %}) will pass to the user profile.  

## Known limitations

There are a couple of caveats that should be taken into account.

### Missing email address

Kimai requires that every user account has an email address. If you do not configure an attribute for email, 
the username will be used as fallback for the email during the initial import of the user account.

This will lead to problems, when you try to update a user profile in Kimai - you will see an error saying
that the email is not valid, even if you only tried to change the user roles.

- Bad solution: change the users email address manually, it will not be overwritten by the sync
- Good solution: sync the users email address to Kimai

### Profile changes will be overwritten

As all configured user attributes will be synchronized on every login, manual profile changes 
in the internal user database won't be permanent. 

But: fields which are not synced, won't be changed during the user login.

### Role changes will be overwritten 

If you configured the group sync, the assigned user roles in Kimai will be overwritten on login.

Roles are not merged, but replaced during authentication, so you cannot  
demote or promote a User permanently to another role in Kimai.

The rule is: either manage all roles in Kimai or in LDAP, mixing is not possible.

## Examples

{% include alert.html type="info" alert="Before you start to configure your LDAP, switch to 'dev' environment and tail 'var/log/dev.log'." %}

Another simple solution to debug the generated queries is to start your OpenLDAP with `sudo /usr/libexec/slapd -d256`.

### Minimal OpenLDAP

A minimal setup with a local OpenLDAP with roles sync.
This will only work for very basic LDAP setups, but it demonstrates the power of default values.

```yaml
kimai:
    ldap:
        active: true
        connection:
            host: 127.0.0.1
            bindRequiresDn: true
        user:
            baseDn: ou=users, dc=kimai, dc=org
        role:
            baseDn: ou=groups, dc=kimai, dc=org
```

The generated query to find the users DN looks like this:
```
SRCH base="ou=users,dc=kimai,dc=org" scope=2 deref=0 filter="(&(uid=foo))"
SRCH attr=dn
```
The query to find all user attributes looks like this:
```
SRCH base="uid=foo,ou=users,dc=kimai,dc=org" scope=2 deref=0 filter="(objectClass=*)"
SRCH attr=+ *
```
The generated query for the group-to-role mapping: 
```
SRCH base="ou=groups,dc=kimai,dc=org" scope=2 deref=0 filter="(&(member=uid=foo,ou=users,dc=kimai,dc=org))"
SRCH attr=cn + *
```

The Kimai account will have the username and email set to the `uid`, because we did not configure 
another usernameAttribute (like `cn`) or a mapping for the email.

If the role search would have returned groups with the `cn` value `admin`, `super_admin` or `teamlead`, 
the new account would have been promoted into these roles.

### OpenLDAP with group sync

A secured local OpenLDAP on port 543 with roles sync for the objectClass `inetOrgPerson` users:

```yaml
kimai:
    ldap:
        active: true
        connection:
            host: 127.0.0.1
            port: 543
            user: kimai
            password: serverToken
            bindRequiresDn: true
        user:
            baseDn: ou=users, dc=kimai, dc=org
            filter: (&(objectClass=inetOrgPerson)(uid=%s))
            usernameAttribute: uid
            attributes:
                - { ldap_attr: uid, user_method: setUsername }
                - { ldap_attr: cn, user_method: setAlias }
                - { ldap_attr: mail, user_method: setEmail }
        role:
            baseDn: ou=groups, dc=kimai, dc=org
            filter: (&(objectClass=groupOfNames)(|(cn=teamlead)(cn=manager)(cn=devops)))
            userDnAttribute: member
            usernameAttribute: uid
            groups:
                - { ldap_value: teamlead, role: ROLE_TEAMLEAD }
                - { ldap_value: manager, role: ROLE_ADMIN }
                - { ldap_value: devops, role: ROLE_SUPER_ADMIN }
```

### Connect to Active Directory

Just an example how you might be able to connect to your AD.
 
```yaml
kimai:
    ldap:
        active: true
        connection:
            host: ad.example.com
            username: user@ad.example.com
            password: secret
            accountDomainName: ad.example.com
            accountDomainNameShort: AD
        user:
            baseDn: dc=ad,dc=example,dc=com
            filter: (&(ObjectClass=Person))
            attributes:
                - { ldap_attr: samaccountname,  user_method: setUsername }
```

{% include alert.html type="warning" alert="I have never worked with AD, please contact me at GitHub if you can provide further examples." %}

The LDAP connection is based on [FR3DLdapBundle](https://github.com/Maks3w/FR3DLdapBundle), thanks to @Maks3w for sharing!