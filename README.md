# NAME 

Dancer2::Plugin::Auth::Extensible::Provider::DBIC - authenticate via the
[Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC) plugin

# DESCRIPTION

This class is an authentication provider designed to authenticate users against
a database, using [Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC) to access a database.

See [Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC) for how to configure a database connection
appropriately; see the ["CONFIGURATION"](#configuration) section below for how to configure this
authentication provider with database details.

See [Dancer2::Plugin::Auth::Extensible](https://metacpan.org/pod/Dancer2::Plugin::Auth::Extensible) for details on how to use the
authentication framework.

# CONFIGURATION

This provider tries to use sensible defaults, in the same manner as
[Dancer2::Plugin::Auth::Extensible::Provider::Database](https://metacpan.org/pod/Dancer2::Plugin::Auth::Extensible::Provider::Database), so you may not need
to provide much configuration if your database tables look similar to those.

The most basic configuration, assuming defaults for all options, and defining a
single authentication realm named 'users':

    plugins:
        Auth::Extensible:
            realms:
                users:
                    provider: 'DBIC'

You would still need to have provided suitable database connection details to
[Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC), of course;  see the docs for that plugin for full
details, but it could be as simple as, e.g.:

    plugins:
        Auth::Extensible:
            realms:
                users:
                    provider: 'DBIC'
                    users_resultset: 'User'
                    roles_resultset: Role
                    user_roles_resultset: UserRole
        DBIC:
            default:
                dsn: dbi:mysql:database=mydb;host=localhost
                schema_class: MyApp::Schema
                user: user
                pass: secret

A full example showing all options:

    plugins:
        Auth::Extensible:
            realms:
                users:
                    provider: 'DBIC'

                    # Should get_user_details return an inflated DBIC row
                    # object? Defaults to false which will return a hashref
                    # inflated using DBIx::Class::ResultClass::HashRefInflator
                    # instead. This also affects what `logged_in_user` returns.
                    user_as_object: 1

                    # Optionally specify the sources of the data if not the
                    # defaults (as shown).  See notes below for how these
                    # generate the resultset names.  If you use standard DBIC
                    # resultset names, then these and the column names are the
                    # only settings you might need.  The relationships between
                    # these resultsets is automatically introspected by
                    # inspection of the schema.
                    users_source: 'user'
                    roles_source: 'role'
                    user_roles_source: 'user_role'

                    # optionally set the column names
                    users_username_column: username
                    users_password_column: password
                    roles_role_column: role

                    # This plugin supports the DPAE record_lastlogin functionality.
                    # Optionally set the column name:
                    users_lastlogin_column: lastlogin

                    # Optionally set columns for user_password functionality in
                    # Dancer2::Plugin::Auth::Extensible
                    users_pwresetcode_column: pw_reset_code
                    users_pwchanged_column:   # Time of reset column. No default.

                    # Days after which passwords expire. See logged_in_user_password_expired
                    # functionality in Dancer2::Plugin::Auth::Extensible
                    password_expiry_days:       # No default

                    # Optionally set the name of the DBIC schema
                    schema_name: myschema

                    # Optionally set additional conditions when searching for the
                    # user in the database. These are the same format as required
                    # by DBIC, and are passed directly to the DBIC resultset search
                    user_valid_conditions:
                        deleted: 0
                        account_request:
                            "<": 1

                    # Optionally specify a key for the user's roles to be returned in.
                    # Roles will be returned as role_name => 1 hashref pairs
                    roles_key: roles

                    # Optionally specify the algorithm when encrypting new passwords
                    encryption_algorithm: SHA-512

                    # If you don't use standard DBIC resultset names, you might
                    # need to configure these instead:
                    users_resultset: User
                    roles_resultset: Role
                    user_roles_resultset: UserRole

                    # Optional: To validate passwords using a method called
                    # 'check_password' in users_resultset result class
                    # which takes the password to check as a single argument:
                    users_password_check: check_password

                    # Deprecated settings. The following settings were renamed for clarity
                    # to the *_source settings
                    users_table:
                    roles_table:
                    user_roles_table:

- user\_as\_object

    Defaults to false.

    By default a row object is returned as a simple hash reference using
    [DBIx::Class::ResultClass::HashRefInflator](https://metacpan.org/pod/DBIx::Class::ResultClass::HashRefInflator). Setting this to true
    causes normal row objects to be returned instead.

- user\_source

    Specifies the source name that contains the users. This will be camelized to generate
    the resultset name. The relationship to user\_roles\_source will be introspected from
    the schema.

- role\_source

    Specifies the source name that contains the roles. This will be camelized to generate
    the resultset name. The relationship to user\_roles\_source will be introspected from
    the schema.

- user\_roles\_source

    Specifies the source name that contains the user\_roles joining table. This will be
    camelized to generate the resultset name. The relationship to the user and role
    source will be introspected from the schema.

- users\_username\_column

    Specifies the column name of the username column in the users table

- users\_password\_column

    Specifies the column name of the password column in the users table

- roles\_role\_column

    Specifies the column name of the role name column in the roles table

- schema\_name

    Specfies the name of the [Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC) schema to use. If not
    specified, will default in the same manner as the DBIC plugin.

- user\_valid\_conditions

    Specifies additional search parameters when looking up a user in the users table.
    For example, you might want to exclude any account this is flagged as deleted
    or disabled.

    The value of this parameter will be passed directly to DBIC as a search condition.
    It is therefore possible to nest parameters and use different operators for the
    condition. See the example config above for an example.

- roles\_key

    Specifies a key for the returned user hash to also return the user's roles in.
    The value of this key will contain a hash ref, which will contain each
    permission with a value of 1. In your code you might then have:

        my $user = logged_in_user;
        return foo_bar($user);

        sub foo_bar
        {   my $user = shift;
            if ($user->{roles}->{beer_drinker}) {
               ...
            }
        }

    This isn't intended to replace the ["user\_has\_role" in Dancer2::Plugin::Auth::Extensible](https://metacpan.org/pod/Dancer2::Plugin::Auth::Extensible#user_has_role)
    keyword. Instead it is intended to make it easier to access a user's roles if the
    user hash is being passed around (without requiring access to the user\_has\_role
    keyword in other modules).

- users\_resultset
- roles\_resultset
- user\_roles\_resultset

    These configuration values are provided for fine-grain tuning of your DBIC
    resultset names. If you use standard DBIC naming practices, you will not need
    to configure these, and they will be generated internally automatically.

# SUGGESTED SCHEMA

If you use a schema similar to the examples provided here, you should need minimal 
configuration to get this authentication provider to work for you.  The examples 
given here should be MySQL-compatible; minimal changes should be required to use 
them with other database engines.

## user Table

You'll need a table to store user accounts in, of course. A suggestion is something 
like:

     CREATE TABLE user (
         id int(11) NOT NULL AUTO_INCREMENT,
                 username varchar(32) NOT NULL,
         password varchar(40) DEFAULT NULL,
         name varchar(128) DEFAULT NULL,
         email varchar(255) DEFAULT NULL,
         deleted tinyint(1) NOT NULL DEFAULT '0',
         lastlogin datetime DEFAULT NULL,
         pw_changed datetime DEFAULT NULL,
         pw_reset_code varchar(255) DEFAULT NULL,
         PRIMARY KEY (id)
     );

All columns from the users table will be returned by the `logged_in_user` keyword 
for your convenience.

## role Table

You'll need a table to store a list of available groups in.

         CREATE TABLE role (
         id int(11) NOT NULL AUTO_INCREMENT,
         role varchar(32) NOT NULL,
         PRIMARY KEY (id)
     );

## user\_role Table

Also requred is a table mapping the users to the roles.

     CREATE TABLE user_role (
         user_id int(11) NOT NULL,
         role_id int(11) NOT NULL,
         PRIMARY KEY (user_id, role_id),
         FOREIGN KEY (user_id) REFERENCES user(id),
         FOREIGN KEY (role_id) REFERENCES role(id)
     );

# SEE ALSO

[Dancer2::Plugin::Auth::Extensible](https://metacpan.org/pod/Dancer2::Plugin::Auth::Extensible)

[Dancer2::Plugin::DBIC](https://metacpan.org/pod/Dancer2::Plugin::DBIC)

[Dancer2::Plugin::Auth::Extensible::Provider::Database](https://metacpan.org/pod/Dancer2::Plugin::Auth::Extensible::Provider::Database)

# AUTHORS

Andrew Beverley `<a.beverley@ctrlo.com>`

Rewrite for Plugin2:

Peter Mottram, `<peter@sysnix.com>`

# CONTRIBUTORS

Ben Kaufman (whosgonna)

# LICENSE AND COPYRIGHT

Copyright 2015-2016 Andrew Beverley

This program is free software; you can redistribute it and/or modify it
under the terms of either: the GNU General Public License as published
by the Free Software Foundation; or the Artistic License.

See http://dev.perl.org/licenses/ for more information.
