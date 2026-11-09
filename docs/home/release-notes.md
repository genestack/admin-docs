# Release notes

## Version 1.65

!!! tip ""
    Helm chart version 1.65.0

### Search index rebuild on instances installed before 1.51

ODM stays on Solr 9 in this release, and moves to Solr 10 in a later one. Your Helm values need no change for Solr
compared with 1.64: the `solr-exporter` section stays as it is.

Solr 10 cannot open a search index that Solr 8 created, even one that Solr 9 has rewritten since. ODM ran on Solr 8
before version 1.51, so an instance first installed before 1.51 still has such indexes. So that Solr 10 can open its
indexes later, ODM 1.65 rebuilds them on the instance's first start of 1.65. Instances first installed in 1.51 or
later are not affected: their indexes are kept as they are. If you are not sure when yours was installed, the Solr
log says on the first start, as [Follow the rebuild in the logs](#follow-the-rebuild-in-the-logs) shows.

!!! tip

    Take a snapshot of the Solr volume before you upgrade: the `odm-index` PersistentVolumeClaim for a release named
    `odm`, or the claim set in `solr.persistence.existingClaim`. See [Rolling back](#rolling-back).

#### What happens on the first start

1. Before Solr starts, its container reads which version of Lucene, Solr's search library, created each core's
   index. It empties every core whose index Lucene 8 created, and Solr starts that core as a new, empty index.

2. Core re-indexes every study, file and task from MySQL in the background, as it does on every start. This refills
   `metadata_core`, `study_core` and `tasks_core`.

3. Dictionary terms and reference genome features reach Solr only when a dictionary or reference genome is
   initialized, so step 2 does not bring back `dictionary_core` and `datafile_core`. Once it has started, core looks
   in the background for every initialized dictionary and reference genome that has nothing in Solr. It parses each
   of them again from the copy of its source that initialization stored, and indexes it. Nothing is downloaded,
   not even the ontologies that an OWL dictionary imports: ODM takes terms only from the dictionary itself. No task
   runs, nothing is written to the files, and templates do not change: the files stay initialized, as they were.

Core indexes one file at a time. The largest ontology, NCBITaxon, needs about 6.6 GB of heap. If core stops while it
indexes a file, for example because it runs out of memory, then on its next start it deletes what it had indexed of
that file and indexes it again. After 3 such stops, core leaves that file out of Solr and goes on with the others.
Its error line in core's log gives the SQL statement to run in the `genestack` database before you restart core, to
try that file again, for example once core has more heap.

To turn off indexing dictionaries and reference genomes again altogether, set:

```yaml
core:
  files:
    "/var/lib/genestack/properties/application.yaml":
      backend:
        solr:
          indexRepair:
            enabled: false
```

#### While the rebuild runs

ODM starts and is available while the rebuild runs. How long it takes depends on the instance. As an example of
scale, on one instance core re-indexed some 2.6 million studies, files and tasks, and search was complete again
about 1.5 hours after core started. At the same time, core indexed 201 dictionaries and 7 reference genomes again
in about 45 minutes: most dictionaries took about a second each, and one of 104,486 terms took about 3 minutes.

Until the rebuild finishes, on an instance that needs it:

- Search and the Study Browser show incomplete results, until core has re-indexed every study, file and task and
  logged `Harvester stopped`.

- Term lookups and suggestions return nothing for a dictionary that core has not indexed again yet.

- Validation skips a dictionary that core has not indexed again yet: a value that is not in it is not flagged, and
  that result stays until the metadata is validated again.

So hold off curation until core has logged both `Harvester stopped` and `Solr index repair: finished`. Upgrading
when curators are not working is the simplest way.

#### Follow the rebuild in the logs

On every start, the Solr log has one line for each core that has an index:

```text
solr-index-check: <core>: index created by Lucene 8, older than 9: removing <dir>, Solr starts the core empty and core re-indexes it
solr-index-check: <core>: index created by Lucene 9, kept
```

When every core is `kept` on the first start of 1.65, the instance needs no rebuild.

Core's log shows when the re-index of studies, files and tasks starts and when it ends:

```text
Harvester started
Harvester stopped, <n> objects indexed in <hh:mm:ss>
```

`Harvester stopped` is the line to wait for: from then on, search is complete again. When nothing goes wrong, core
logs nothing about the re-index between these two lines. Errors such as
`Indexing task failed, will be redeemed in …` or `Exception in Crawler, will retry …` do show, and core retries
both. But if core logged `Exception in Crawler` without "will retry" before `Harvester stopped`, the re-index ended
early, and search is only partly rebuilt. Restarting core runs the re-index again.

It also shows the dictionaries and reference genomes it indexes again, and the end of that work:

```text
Solr index repair: <n> dictionaries and <m> reference genomes are initialized but have nothing in Solr, indexing them again
Solr index repair: parsed <n> terms of dictionary <accession>
Solr index repair: indexed dictionary <accession> again in <time>
Solr index repair: indexed reference genome <accession> again in <time>
Solr index repair: finished in <time>: <n> files indexed again, <k> failed
```

For an OWL dictionary that imports other ontologies, core also logs `Not loading import <iri>` for each import. It
does not need them.

When something goes wrong, it logs:

- `Solr index repair: could not index dictionary <accession> yet, will try again at the end: <exception>`, a
  warning. Core tries that dictionary once more after the others.

- `Solr index repair: could not index dictionary <accession> again`, or `reference genome` in place of `dictionary`,
  an error with its cause.

- `Solr index repair: core stopped while indexing <accession> before, deleting what it had indexed`, when that
  file may be in Solr only in part: core stopped in the middle of it, for example out of memory, or could not delete
  what it had indexed of it after a failure. Core then indexes the file again from scratch.

- `Solr index repair: core stopped <n> times while indexing <accession>, leaving it out of Solr; to try again, for
  example with more heap, run DELETE FROM SolrIndexRepair WHERE fileId = <id> and restart core`, an error once core
  has stopped 3 times in the middle of that file.

- `Solr index repair: could not check file id <id>, skipping it`, an error with its cause, for a problem with that
  one file. Core leaves that file out on this start and goes on with the others.

- `Solr index repair: attempt <n> failed, retrying in <delay>`, a warning when Solr or MySQL cannot be reached. Core
  runs the whole repair again after a delay that grows from 30 seconds up to 10 minutes, until it gets through.

- `Solr index repair: <accession> has nothing searchable after PT1M`, a warning that a file core has indexed still
  had nothing searchable a minute later.

The `finished` line then ends with the accessions of the files that failed or that core left out:
`<k> failed: <accession>, <accession>`.

When nothing is missing, the repair logs only:

```text
Solr index repair: every initialized dictionary and reference genome is in Solr
```

Some lines come back on every start, though: core parses again a dictionary that has no terms and a dictionary whose
S3 object is gone, and logs again each file it has left out of Solr.

#### Dictionaries created from an S3 link

!!! warning

    A dictionary created from an S3 link keeps only that link, not a copy of the file. If the S3 object no longer
    exists, core cannot index the dictionary again, and the dictionary stays without terms. Core tries again on every
    start, so putting the object back where the link points and restarting core brings the terms back.

#### Rolling back

Rolling back to 1.64 after the rebuild works: Solr 9 reads the rebuilt indexes. If you roll back before core has
logged `Solr index repair: finished`, the dictionaries and reference genomes it has not reached yet stay missing from
Solr under 1.64, which does not index them again. They come back when you upgrade to 1.65 again. The snapshot taken
before the upgrade puts back the indexes as they were.

## Version 1.64

!!! danger

    This version must be installed before proceeding with the next update.

!!! tip ""
    Helm chart version 1.64.0

### Genestack audit logs migration from MySQL to Clickhouse

This release migrates MySQL `genestack_audit` schema to ClickHouse `audit.Event` table and
removes `genestack_audit` schema from MySQL upon successful completion. Migration component responsible for it
is planned for a removal in the next release. As a result, this ODM version cannot be skipped during upgrades.

### Excluding audit events from tracking

It is now possible to exclude specific audit events from tracking if they are not needed, which saves disk space.
For example, you can exclude events related to data reads.

A configuration example is available in the `examples` directory of the Helm chart.

### Audit log retention period

!!! danger

    The retention period cannot be changed after installation. Set it before you install or upgrade to this version.

ODM keeps audit events in ClickHouse for a limited number of months and deletes older events automatically.
The default retention period is 12 months. To use a different one, set it in the Helm chart values:

```yaml
core:
  files:
    "/var/lib/genestack/properties/application.yaml":
      backend:
        clickhouse:
          audit:
            retentionMonths: 24
```
