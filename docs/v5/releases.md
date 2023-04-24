Releases
========

HTCondor-CE 5 is distributed via RPM and are available from the following Yum repositories:

- [HTCondor stable and current channels](https://research.cs.wisc.edu/htcondor/downloads/)
- [Open Science Grid](https://opensciencegrid.org/docs/common/yum/)


Known Issues
------------

Known bugs affecting HTCondor-CEs can be found in
[Jira](https://opensciencegrid.atlassian.net/issues/?jql=project%20%3D%20HTCONDOR%20AND%20status%20not%20in%20(done%2C%20abandoned)%20and%20component%20%3D%20htcondor-ce%20and%20issuetype%20%3D%20bug)
In particular, the following bugs are of note:

-   C-style comments, e.g. `/* comment */`, in `JOB_ROUTER_ENTRIES` will prevent the JobRouter from routing jobs
    ([HTCONDOR-864](https://opensciencegrid.atlassian.net/browse/HTCONDOR-864)).
    For the time being, remove any comments if you are still using the
    [deprecated syntax](configuration/job-router-overview.md#deprecated-syntax).

Updating to HTCondor-CE 5
-------------------------

!!! tip "Finding relevant configuration changes"
    When updating HTCondor-CE RPMs, `.rpmnew` and `.rpmsave` files may be created containing new defaults that you
    should merge or new defaults that have replaced your customzations, respectively.
    To find these files for HTCondor-CE, run the following command:

        :::console
        root@host # find /etc/condor-ce/ -name '*.rpmnew' -name '*.rpmsave'

HTCondor-CE 5 is a major release that adds many features and overhauls the default configuration.
As such, upgrades from older versions of HTCondor-CE may require manual intervention:

### Support for ClassAd transforms added to the JobRouter ###

!!! danger "Transforms will override `JOB_ROUTER_ENTRIES` routes with the same name"
    Even if you do not plan on immediately using the new syntax, it's important to note that route transforms will
    override `JOB_ROUTER_ENTRIES` routes with the same name.
    In other words, the route transform names returned by `condor_ce_config_val -dump -v JOB_ROUTER_ROUTE_` should only
    appear in your list of used routes returned by `condor_ce_config_val JOB_ROUTER_ROUTE_NAMES` if you
    intend to use the new transform syntax.

HTCondor-CE now includes default [ClassAd transforms](https://htcondor.readthedocs.io/en/lts/classads/transforms.html)
equivalent to its `JOB_ROUTER_DEFAULTS`, allowing administrators to write job routes using the transform synatx.
The old syntax continues to be the default in HTCondor-CE 5.
Writing routes in the new syntax provides many benefits including:

-   Statements being evaluated in the order they are written
-   Use of variables that are not included in the resultant job ad
-   Use of simple case statements.

Additionally, it is now easier to include transforms that should be evaluated before or after your routes by including
transforms in the lists of `JOB_ROUTER_PRE_ROUTE_TRANSFORM_NAMES` and `JOB_ROUTER_PRE_ROUTE_TRANSFORM_NAMES`,
respectively.  To use the new transform syntax:

1.  Disable use of `JOB_ROUTER_ENTRIES` by setting the following in `/etc/condor-ce/config.d/`:

        :::console
        JOB_ROUTER_USE_DEPRECATED_ROUTER_ENTRIES = False

1.  Set `JOB_ROUTER_ROUTE_<ROUTE_NAME>` to a job route in the new transform syntax where `<ROUTE_NAME>` is the name of
    the route that you'd like to be reflected in logs and tool output.

1.  Add the above `<ROUTE_NAME>` to the list of routes in `JOB_ROUTER_ROUTE_NAMES`

### New `condor_mapfile` format and locations  ###

HTCondor-CE 5 separates its
[unified mapfile](https://htcondor.readthedocs.io/en/lts/admin-manual/security.html#the-unified-map-file-for-authentication)
used for authentication between multiple files across multiple directories.
Additionally, any regular expressions in the second field must be enclosed by `/`.
To update your mappings to the new format and location, perform the following actions:

1.  Upon upgrade, your existing mapfile will be moved to `/etc/condor-ce/condor_mapfile.rpmsave`.
    Remove any of the following lines provided by default in the HTCondor-CE packaging:

        GSI (.*) GSS_ASSIST_GRIDMAP
        SSL "[-.A-Za-z0-9/= ]*/CN=([-.A-Za-z0-9/= ]+)" \1@unmapped.htcondor.org
        CLAIMTOBE .* anonymous@claimtobe
        FS "^(root|condor)$" \1@daemon.htcondor.org
        FS "(.*)" \1

1.  Copy the remaining contents of `/etc/condor-ce/condor_mapfile.rpmsave` to a file ending in `*.conf` in
    `/etc/condor-ce/mapfiles.d/`.
    Note that files in this folder are parsed in lexicographic order.

1.  Update the second field of any existing mappings by enclosing any regular expressions in `/`, escaping any slashes
    with a backslash (e.g. `\/`).

    -    Consider converting any `GSI` mappings into Perl Compatible Regular Expressions (PCRE) since the authenticated
         name of incoming proxies may contain additional VOMS FQANs in addition to the Distinguished Name (DN):

            <DN>,<VOMS FQAN 1>,<VOMS FQAN 2>,...,<VOMSFQAN N>

        For example, to accept a given DN with any VOMS attributes, the mapping should look like the following:

            GSI /^\/DC=org\/DC=cilogon\/C=US\/O=University of Wisconsin-Madison\/CN=Brian Lin A226624,.*/ blin

        Alternatively, to accept any DN from the OSG VO:

            GSI /.*,\/osg\/Role=Pilot\/Capability=.*/ osg

    -   Also consider converting `SCITOKENS` mappings to PCRE since the authenticated name of incoming tokens will
        contain the token issuer (`iss`) and any token subject (`sub`) fields:

            <TOKEN ISSUER>,<TOKEN SUB>

        For example, to accept a token issued by the OSG VO with any subject, write the following mapping:

            SCITOKENS /^https:\/\/scitokens.org\/osg-connect,.*/ osg

### Specify certificate locations for token authentication ###

HTCondor-CE 5 adds improved support for accepting pilot jobs submitted with bearer tokens
(e.g., SciTokens or WLCG tokens).
As part of the bearer token authentication, HTCondor-CE uses its host certificate to perform an SSL handshake with the
client to establish trust with its token issuer.
Consult the [authentication documentation](configuration/authentication.md#configuring-certificates) to configure
certificate locations for token authentication.

### No longer set `$HOME` by default ###

Older versions of HTCondor-CE set `$HOME` in the routed job to the user's `$HOME` directory on the HTCondor-CE.
To re-enable this behavior, set `USE_CE_HOME_DIR = True` in `/etc/condor-ce/config.d/`.

HTCondor-CE 5 Version History
-----------------------------

This section contains release notes for each version of HTCondor-CE 5.
Full HTCondor-CE version history can be found on [GitHub](https://github.com/htcondor/htcondor-ce/releases).

### 5.1.6 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.6) includes the following changes:

-   HTCondor-CE now uses the C++ Collector plugin for payload job traceability
-   Fix HTCondor-CE mapfiles to be compliant with PCRE2 and HTCondor 9.10.0+
-   Add support for multiple APEL accounting scaling factors
-   Suppress spurious log message about a missing negotiator
-   Fix crash in HTCondor-CE View

### 5.1.5 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.5) includes the following changes:

-   Rename AuthToken attributes in the routed job to better support accounting
-   Prevent GSI environment from pointing the job to the wrong certificates
-   Fix issue where HTCondor-CE would need port 9618 open to start up

### 5.1.4 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.4) includes the following changes:

-   Fix whole node job glidein CPUs and GPUs expressions that caused held jobs
-   Fix bug where default CERequirements were being ignored
-   Pass whole node request from GlideinWMS to the batch system
-   Since CentOS 8 has reached end of life, we build and test on Rocky Linux 8

### 5.1.3 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.3) includes the following changes:

-   The HTCondor-CE central collector requires SSL credentials from client CEs
-   Fix BDII crash if an HTCondor Access Point is not available
-   Fix formatting of APEL records that contain huge values
-   HTCondor-CE client mapfiles are not installed on the central collector

### 5.1.2 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.2) includes the following changes:

-   Fixed the default memory and CPU requests when using job router transforms
-   Apply default MaxJobs and MaxJobsIdle when using job router transforms
-   Improved SciTokens support in submission tools
-   Fixed --debug flag in condor\_ce\_run
-   Update configuration verification script to handle job router transforms
-   Corrected ownership of the HTCondor PER\_JOBS\_HISTORY\_DIR
-   Fix bug passing maximum wall time requests to the local batch system

### 5.1.1 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.1) includes the following changes:

-   Improve restart time of HTCondor-CE View
    ([HTCONDOR-420](https://opensciencegrid.atlassian.net/browse/HTCONDOR-420))
-   Fix bug that caused HTCondor-CE to ignore incoming BatchRuntime requests (#480)
-   Fixed error that occurred during RPM installation of non-HTCondor batch systems regarding missing file `batch_gahp`
    ([HTCONDOR-504](https://opensciencegrid.atlassian.net/browse/HTCONDOR-504))

### 5.1.0 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.1.0) includes the following new features:

-   Add support for [ClassAd transforms](https://htcondor.readthedocs.io/en/lts/classads/transforms.html)
    to the JobRouter ([HTCONDOR-243](https://opensciencegrid.atlassian.net/browse/HTCONDOR-243))
-   Add mapped user and X.509 attribute to local HTCondor pool AccountingGroup mappings to Job Routers configured to use
    the ClassAd transform syntax ([HTCONDOR-187](https://opensciencegrid.atlassian.net/browse/HTCONDOR-187))
-   Split `condor_mapfile` into files that use regular expressions in `/etc/condor-ce/mapfiles.d/*.conf`
    ([HTCONDOR-244](https://opensciencegrid.atlassian.net/browse/HTCONDOR-244))
-   Accept `BatchRuntime` attributes from incoming jobs to set their maximum walltime
    ([HTCONDOR-80](https://opensciencegrid.atlassian.net/browse/HTCONDOR-80))
-   Update the HTCondor-CE registry to Python 3
    ([HTCONDOR-307](https://opensciencegrid.atlassian.net/browse/HTCONDOR-307))
-   Enable SSL authentication by default for `READ`/`WRITE` authorization levels
    ([HTCONDOR-366](https://opensciencegrid.atlassian.net/browse/HTCONDOR-366))
-   APEL reporting scripts now use history files in the local HTCondor `PER_JOB_HISTORY_DIR` to collect job data.
    ([HTCONDOR_293](https://opensciencegrid.atlassian.net/browse/HTCONDOR-293))
-   Use the `GlobalJobID` attribute as the APEL record `lrmsID`
    ([#426](https://github.com/htcondor/htcondor-ce/pull/426))
-   Downgrade errors in the configuration verification startup script to support routes written in the transform syntax
    ([#465](https://github.com/htcondor/htcondor-ce/pull/465))
-   Allow required directories to be owned by non-`condor` groups
    ([#451](https://github.com/htcondor/htcondor-ce/pull/451/files))

This release also includes the following bug-fixes:

-   Fix an issue with an overly aggressive default `SYSTEM_PERIODIC_REMOVE`
    ([HTCONDOR-350](https://opensciencegrid.atlassian.net/browse/HTCONDOR-350))
-   Fix incorrect path to Python 3 Collector plugin
    ([HTCONDOR-400](https://opensciencegrid.atlassian.net/browse/HTCONDOR-400))
-   Fix faulty validation of `JOB_ROUTER_ROUTE_NAMES` and `JOB_ROUTER_ENTRIES` in the startup script
    ([HTCONDOR-406](https://opensciencegrid.atlassian.net/browse/HTCONDOR-406))
-   Fix various Python 3 incompatibilities
    ([#460](https://github.com/htcondor/htcondor-ce/pull/460))

### 5.0.0 ###

[This release](https://github.com/htcondor/htcondor-ce/releases/tag/v5.0.0) includes the following new features:

-   Python 3 and Enterprise Linux 8 support
    ([HTCONDOR_13](https://opensciencegrid.atlassian.net/browse/HTCONDOR-13))
-   HTCondor-CE no longer sets `$HOME` in routed jobs by default
    ([HTCONDOR-176](https://opensciencegrid.atlassian.net/browse/HTCONDOR-176))
-   Whole node jobs (local HTCondor batch systems only) now make use of GPUs
    ([HTCONDOR-103](https://opensciencegrid.atlassian.net/browse/HTCONDOR-103))
-   HTCondor-CE Central Collectors now prefer GSI over SSL authentication
    ([HTCONDOR-237](https://opensciencegrid.atlassian.net/browse/HTCONDOR-237))
-   HTCondor-CE registry now validates the value of submitted client codes
    ([HTCONDOR-241](https://opensciencegrid.atlassian.net/browse/HTCONDOR-241))
-   Automatically remove CE jobs that exceed their `maxWalltime` (if defined) or the configuration value of
    `ROUTED_JOB_MAX_TIME` (default: 4320 sec/72 hrs)

This release also includes the following bug-fixes:

-   Fix a circular configuration definition in the HTCondor-CE View that resulted in 100% CPU usage by the
    `condor_gangliad` daemon ([HTCONDOR-161](https://opensciencegrid.atlassian.net/browse/HTCONDOR-161))


Getting Help
------------

If you have any questions about the release process or run into issues with an upgrade, please
[contact us](../index.md#contact-us) for assistance.
