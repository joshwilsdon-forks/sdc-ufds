---
title: UFDS Replicator
markdown2extras: tables, code-friendly
apisections: Overview, Geting Started
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# UFDS Replicator

This library provides the functionality required to replicate data living on a
remote UFDS instance into a local one.


# Overview

Data replication on LDAP can be done in several ways and in our case we take
advantage of the changelogs functionality that UFDS implements. In order to
consume the changelog objects we can either use [persistent
search](http://tools.ietf.org/id/draft-ietf-ldapext-psearch-03.txt) or polling.
In our case we take the polling approach with a configurable search interval.


## Changelogs

The idea behind changelogs is that servers should be able to notify clients
about each of the operations that have been produced in the remote LDAP server
in an ordered way. By having access to said changelog, clients can replicate
each of the changes made to an LDAP server provided that there are no
conflicting existing records in the local LDAP instance at the time of applying
these changes.

For more information, refer to the [LDAP changelogs
specification](http://tools.ietf.org/html/draft-good-ldap-changelog-04).


## Replication URLs

ufds-replicator allows replicating multiple subtrees from the same remote UFDS
instance. For the case of an SDC install, one could replicate the entire
servers and users trees separately without caring about the rest of the SDC
data such as config and groups. In order to do this, replication URLs must meet
the [LDAP url](http://www.ietf.org/rfc/rfc2255.txt) format specification. This
lets ufds-replicator know which additional criteria must the prospect records
meet in order to be replicated. For example, a replication URL might specify
that users with first name "Stan" should not be replicated. In this case,
ufds-replicator provides the functionality needed to compare local objects to
remote ones and decide when is an object eligible for replication.

The full format of a replication URL is the following:

        ldapurl = scheme "://" [hostport] ["/"
                    [dn ["?" [attributes] ["?" [scope]
                    ["?" [filter] ["?" extensions]]]]]]

If we ignore 'attributes' for now we now have:

        ldapurl = scheme "://" [hostport] ["/"
                    [dn ["??" [scope] ["?" [filter] ["?" extensions]]]]]

As an example, let's take a look at an URL for replicating all users in SDC:

        ldaps://10.99.99.14/ou=users,%20o=smartdc??sub?

And the following is the meaning of each of the fields in the URL:

| field  | value                              | meaning                                    |
| ------ | ---------------------------------- | ------------------------------------------ |
| dn     | ou=users, o=smartdc                | base subtree to replicate                  |
| scope  | sub                                | LDAP search scope, can be sub, one or base |
| filter | empty (defaults to objectclass=\*) | LDAP search filter                         |

Given this, if we wanted to replicate only sdcperson or sdckey objects under
the users tree (ignoring vms) our URL would look like:

        ldaps://10.99.99.14/ou=users,%20o=smartdc??sub?(!(objectclass=vm))


# Getting started

ufds-replicator requires two UFDS instances running at the same time. The local
UFDS should have no data in it (for our tests). We will later see how a minimal
set of bootstrap data is needed for tests to work.

If you already know how to deploy an additional UFDS instance, skip to the
"Replicating Data" section.


## Running a Second UFDS Instance

For a second UFDS instance we can create a second ufds zone with sdc-role or
just clone the ufds.git repository into a separate environment. We just need a
running UFDS server regardless the method we use.

For installing UFDS on Mac OS X follow the next steps:

        $ git clone git@github.com:joyent/sdc-ufds.git
        $ cd sdc-ufds/
        $ npm install

Now, we have to edit /etc/ufds.laptop.config.json so UFDS uses different
buckets for the smartdc and changelog trees:

        ...
        "changelog": {
            "bucket": "ufds_cn_changelog_two",
        ...
        "o=smartdc": {
            "blacklistRDN": "cn=blacklist",
            "bucket": "ufds_o_smartdc_two",
        ...

After editing the configuration file we can start the UFDS server with:

        node main.js -f ./etc/ufds.laptop.config.json -d 2 2>&1 | ./node_modules/.bin/bunyan

UFDS should be running and ready to be used.


## Replicating Data

ufds-replicator comes with a sample replicator code that can be used as a
standalone server with minor modifications. This code is located at
examples/replicator.js.

Assuming our second UFDS instance has no data in it, we need to load the
minimal SDC bootstrap schema located at data/bootstrap.ldif. This data can be
loaded with the following command (change your LDAP host variable accordingly):

        $ LOCAL_UFDS_URL="ldap://127.0.0.1:1389" \
        node examples/bootstrap-moray.js

        adding new entry "o=smartdc"

        adding new entry "ou=users, o=smartdc"

        adding new entry "ou=groups, o=smartdc"

        adding new entry "ou=config, o=smartdc"

        adding new entry "datacenter=coal, o=smartdc"

        adding new entry "ou=servers, datacenter=coal, o=smartdc"

        adding new entry "cn=replicator, datacenter=coal, o=smartdc"

ufds-replicator is ready to be used.

The sample replicator code will replicate the "ou=users, o=smartdc" tree. Also,
the local UFDS is assumed to run locally at "127.0.0.1:1389" and the remote
UFDS is assumed a running SDC UFDS in the IP address "10.99.99.14". You can
override these parameters without the need to modify the example code. Run the
example replicator with the following command:

        REMOTE_UFDS_URL='ldaps://10.99.99.14' \
        LOCAL_UFDS_URL='ldap://127.0.0.1:1389' \
        node examples/replicator.js | ./node_modules/.bin/bunyan

As soon as the replicator has finished replicating the entire changelog it will
keep polling and listening for new changelogs from the upstream UFDS.


## Recreating the Test Environment

If you are interested in running the replicator many times from a blank databse
all you have to do is run the provided cleanup-moray script. This will delete
the buckets owned by the second UFDS server. Keep in mind that in order to let
UFDS pick up the reset state of the moray buckets it needs to be restarted as
part of the cleanup command:

        // ufds-replicator working directory
        $ MORAY_IP="10.99.99.13" node examples/cleanup-moray.js

        // Restart ufds server
        ^C

        $ node main.js -f ./etc/ufds.laptop.config.json \
        -d 2 2>&1 | ./node_modules/.bin/bunyan

        // Re-bootstrap test data
        $ LOCAL_UFDS_URL="ldap://127.0.0.1:1389" \
        node examples/bootstrap-moray.js


## Running the Tests

The ufds-replicator tests are very similar to the example replicator code,
although it additionally creates some records on UFDS to verify that data is
actually being replicated.

After having a test environment with the bootstrap data loaded into the local
UFDS instance run the following command (change your LDAP host and credentials
variables accordingly):

        REMOTE_UFDS_URL='ldaps://10.99.99.14' \
        LOCAL_UFDS_URL='ldap://127.0.0.1:1389' \
        make test | ./node_modules/.bin/bunyan

Note that every time you run the tests, more of the same changelogs are created
on top of the last tests that have run. Given the nature of the replicator,
this should not affect future tests that run one after another.
