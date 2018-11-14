# wfa-for-svmtool
Wfa workflow wrapper for svmtool (https://github.com/oliviermasson/svmtool), allowing a GUI for this fantastic SVM DR script.

A set of workflows to manage Svm Dr.  Not the cdot embedded svm dr, but a script maintained svm dr.
**note** : this version follows the version for 'svmtool' (read below)

## Why the need ?
Svm dr is currently (cdot version 9.4) not supported with metrocluster and fabric pool.

## Underlying script 'svmtool'
These set of workflows are a GUI wrapper for the standalone powershell script called 'svmtool'.
https://github.com/oliviermasson/svmtool 

This script originated from Jerome Blanchet and is now being developed by Olivier Masson and with some help of the WFAGuy.
Originally the script was a single powershell script and ran interactivally.

In the last few months (today 2018/11/13), we made extensive changes, including :
* NonInteractive mode : allow to run without user interaction
* Intelligent resource selection : using regular expressions (more details below)
* Clone Dr : allowing a full svm clone on the dr site using flexclones
* Log4Net logging : allowing multithreaded logging, including eventviewer integration
* Better cifs server support after dr activate (keeping source cifs name)
* New powershell module, providing individual cmdlets (new-svmdr, invoke-svmdractivate, ...)

Other features
* backup : backup an svm to file (config)
* restore : restore an svm from file (config) -> volumes are new DP volumes (RW is possible too)
* quota sync
* svm migration
* the classic svm dr lifecycle : initialize / update / activate / resync / reverse / recover / ...

> **notes**
> if svmtool changes, you might need to wait for an update of the wfa workflows.
> if you want to manually update svmtool in your wfa server, simply copy the new svmtool code to
> %ProgramFiles%\Netapp\WFA\Posh\Modules\svmtool.  Smaller changes and bugfixes should not impact the working of the workflows

## Workflows
With the exclusion of svm dr restore, which is best to be a manual action, the full svmtool featureset is covered.
* **Svm Dr - Install module** : copy the svmtool to the wfa modules directory (svmtool module is embedded)
* **Svm Dr - Create new svm dr** : create a new relation
* **Svm Dr - Re-initialize an existing svm dr** : re-init existing relation (in case first attempt had warnings or errors)
* **Svm Dr - Update** : update existing relation (to be scheduled to detect changes and new volumes)
* **Svm Dr - Resync** : re-establisches broken snapmirrors (can also reverse snapmirrors)
* **Svm Dr - Activate** : breaks mirrors and onlines the dr side
* **Svm Dr - Recover from Dr** : break a reverse resync and sets the relation back from source to destination
* **Svm Dr - Backup** : take a backup of an svm (of the config) to json-files
* **Svm Dr - Remove Dr** : remove a dr svm and its relations
* **Svm Dr - Set mirror schedule** : add a schedule to the snapmirrors (can also handle reverse)
* **Svm Dr - Clone Svm Dr**  : create a clone of the dr svm (always on dr side)
* **Svm Dr - Remove svm dr clone** : remove a clone of the dr svm
* **Svm Dr - Migrate** : moves the source to the destination and handles a final copy + cutover
* **Svm Dr - Cleanup reverse snapmirrors** : in case of manual manipulation, old reverse mirrors can be removed
* **Svm Dr - Remove source vserver after migrate** : removes a source svm after a succesful migration

## Installation
* import the single dar file
* run the 'svm dr - install module' workflow, to copy the 'svmtool' code (or copy manually from latest github version)
* implement the svm dr config datasource (which reads the wfa.ini file - included with the svmtool script)
* add credentials (clusters, active directory, local users & ldap)

## Ip Addresses
As wfa runs interactively, we assume the dr site is identical when it comes to vlans, ipspaces & ip's.
However, if you could think of a way to deliver a pool of ip's per vlan/subnet, we could adapt the workflow to work with an asymetric setup.

## wfa.ini file

%ProgramFiles%\Netapp\WFA\Posh\Modules\svmtool\wfa.ini
```ini
[Default]
DrSuffix=-dr

[WfaCredentials]
DefaultLocalUserCreds=local_users
DefaultLdapCreds=ldap

[ResourceSelection]
AggrMatchRegex=.*
NodeMatchRegex=(.*)-[0-9]{2} 
````

The ini file contains a few global settings.
We import this ini file using the "svm dr config datasource"

The **DrSuffix** setting, defines how we will call the destination svm.  
**note** : be aware of netbios limitation (15 long)

**WfaCredentials**
During non interactive we can grab credentials for AD & clusters from wfa directly.  For AD we assume the credential-name matches the AD-domainname
But in some scenario's we also require localuser passwords and and ldap integration password.  In Wfa we cannot prompt for them.
So either we skip ldap integration and localuser copy, or we provide the passwords in a form of a "WFACredential".  

for local users, you can use the **DefaultLocalUserCreds** setting to point to the WFA credentials of which we will derive the password to create the local user on the dr side.  Obviously, after an svm dr activate, these passwords will require correction.
For example : in the ini-file above we till wfa to search for wfa credentials "local_users" with the password "Netapp12".  This means we will create all local users on the dr side, but they will all have the same password "Netapp12".

For ldap integration, you can specify the password using the **DefaultLdapCreds** setting.

**ResourceSelection**
In non interactive mode we need to select aggregates (for volumes) and nodes (for lifs).  We have add intelligent regex settings **AggrMatchRegex** and **NodeMatchRegex**.
These regular expressions are first evaluated against the source (aggr or node).  Afterwards al groupmatches are converted to wildcards (.*) and are then ready to search for the destination resource.

Example : node selection regular expressions = "(.*)-[0-9]{2}"
Let's assume that the source node = source-03
We will evaluate the regex against it, and will get "source-03" as result.  However "source" is a group match (wrapped in brackets in the regex) and is replaced by a wildcard.
Hence the final regular express to search the destination node is now : ".*-03" => so it will find the matching node on the dr-side (node 3 in this case)
The same can be done for aggregates, just write a regex to match the source and wrap the dynamic parts in brackets.
Another example : aggr_(newyork)_[0-9]* => will map source aggregate "aggr_newyork_05" to destination aggregate "aggr_boston_05"

If you use .* for the aggrMatch, we will search for the exact same aggregate on the dr side.  

**note** : there is always a fallback.  for aggregates, the one with most free space and for nodes, the first node.
**note2** : for lif-port selection, we just search on broadcast domain and vlan.


