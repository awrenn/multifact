# Puppet Multifact

This module contains a fact terminus that allows for multiple fact termini to be combined into a single fact terminus.  

## Example

The default `routes.yaml` for a PE installation looks something like:  
```
master:
      facts:
        terminus: puppetdb
        cache: json
```  

This means that PuppetDB is the only allowed fact terminus, and local JSON files will be used as a cache, helping to reduce load on PuppetDB. However, in some cases, we want slightly different behavior. Using this module, you could modify your `routes.yaml` file to look something like:
```
master:
    facts:
        terminus: multi
        cache: json
        multi: puppetdb,splunk_hec
```

This module will attempt to use the multiple backends together as one. Since this behavior is not obvious, and can potentially be dangerous if behavior is not understood, we outline the behavior here - please read this before using!:

## find(key)

Option 1:  
When "finding" a value, the multifact terminus will call find on each terminus in succession, swallowing/logging any exception that gets raised. For example, if `puppetdb` raises an exception due to not finding the result, `multifact` will then check with the `splunk_hec` terminus. If a value is found here, then it will be returned. If all termini raise exceptions, then the final one will be re-raised. There is a pretty insidious bug lurking though - when for some reason, a `save` call fails on one terminus but not the successive ones. Then, the next read could one of two different possible states. This is not ideal. But without a across-terminus two-phase transaction call, a partial write is just going to be possible.

Option 2:   
When "finding" a value, only the first terminus will be considered. If this fails, then the whole operation fails. At this point, the multi-terminus approach only matters to `save` (and possibly `destroy`). However, we avoid a whole class of error/possible confusion. So leaving this option around as a thought provoker.

### search(key)

`search` behaves exactly like find - calling `search` on all successive fact termini, until one does not throw. It might be desirable for a combination of all termini to be called, and combined into a single response, but the de-duplication effort is likely excessive. In 99% of cases, facts that are fetched from a terminus that called save should the same result across all responses. For example, PuppetDB and yaml will return the same results if each previous save was successful. Besides the deduplication note, find also applies.

### head(key)

`head`'s logic should be identical to `find`'s, just like `search`. These are the three read operations.  

### destroy(key)

Option 1:  
Destroy is the first write operation. The number one reason for a user to hold multiple different fact termini is to save data to multiple different locations. For this reason, `destroy` should run on every single terminus. If any terminus fails and throws an exception, we should swallow it, until we have passed all the way to the end. Then, we can throw either the most or least recent exception, so the caller at least knows something when wrong. We could also consider having a special aggregate exception class, but I don't think a caller would be prepared to fix issues with two different termini at once.

Option 2:
We abort on first exception, and raise the offender back up. This makes order very important. If a customer has a flaky-but-important data store, maybe a PuppetDB server pushed to the limits, then we this write endpoint will never be dropped to due to other failing data stores, but will also often cause other data stores to drop writes they otherwise would have been able to handle. It also makes the situations where the caller attempts to fix and resolve exceptions much easier. Given that most termini are remote, and exceptions really don't attempt to get fixed often, I think this option is the least preferable. 

### save(instance)

Save really should follow the same logic as `destroy`, although I think a situation where `destroy` takes an abort-early approach, and `save` takes a tough-it-out approach very feasible. `destroy` is a deliberate attempt to get different values on the next read, so a failed destroy will be reflected in the next read if the same attempt order is used. But `save` is an attempt to persist data for a number of different reasons. Each terminus could have its own reasons for persistence, and we should give each one a shot at holding on.

## Extra notes

1. Currently, there is some functionallity where a specific terminus is required. We would really like that multifact can act as these specialized termini. However, we can't know if multifact is solving the problems that this requirement comes from. So here's some speculation about possible terminus issues that might cause this, so we can think about solutions:  
    a. Trusted facts - if getting facts from a JSON or YAML fact terminus, how can we trust trusted facts again? They aren't signed in these formats, and an attacker could modify files on disk within reason. A counter argument there - if an attacker has root write access on the file sytsem, couldn't they modify ruby code to skip the PuppetDB terminus check?

