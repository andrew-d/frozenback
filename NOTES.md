# General Notes

- The compressor should work with mostly no user configuration required.
  Essentially, we're optimizing for the common use case here, at the expense of
  configuration for specialized cases.

## Important Components

- Global chunk store: records the cryptographic hash + size of a chunk, along
  with the archive UUID that it is stored in. *TODO* reference counting?
- Backup record: stores all file metadata, along with an ordered list of chunks
  that make up the file contents
- Backend: simple key-value store that allows pushing a certain amount of data,
  and returns a unique ID for that data, along with retrieving and deleting a
  chunk by ID.
- Key: cryptographic keys used for (at least)
  - Encrypting the chunk contents
  - HMAC of archive (contents + header)
  - Authenticated encryption for backup record.

## Storing

1. Walk file tree, using given ignore patterns to ignore files / directories,
   and collect a list of all files to be archived.
2. Break each files into chunks using *TODO* which algorithm? *TODO*
3. Hash each chunk using a cryptographic hash and deduplicate against our
   global chunk store.
4. Compress & encrypt each chunk that we don't have already.
5. Build a new 'archive' for the chunks we're storing - effectively, a simple
   list of chunk IDs and offsets, a header, and a MAC over the entire contents
   of the file (with the MAC segment in the header set to 0s).
    - Each archive also gets a randomly-generated UUID.
    - We cap the size of each archive at a reasonable limit - 100MiB, maybe?
6. Store each archive with the specified backend (e.g. Glacier, S3, etc. etc.)
7. After each archive is stored, we:
    - Mark the chunks contained in that archive as being stored in the global
      chunk store, along with which archive they're contained in.
    - Add a record to the backup record stating the file name, size, and the
      hashes of each chunk that makes up the file.
8. After all archives are stored, we're done.


## Retrieving

1. For each file we want, read the backup record for that file and determine
   which chunks make up the file.
2. Create a dummy file in a temporary directory for each input file with the
   size set to the expected size of the file.
3. Determine all archives needed to obtain all the necessary chunks and start
   retrieving them from the chunk store.
4. As each archive arrives, write the chunks contained in the archive into the
   correct location in the temporary files (from step #2).
5. Once we've retrieved all chunks for a file, atomically rename the file into
   the backup location.


## Deleting

*TODO* how do we do this?  Ideas:
- Reference-count the chunks used per-archive, so when you remove an archive,
  you decrement the reference count, removed at 0


## Thoughts

- Need to make sure that we don't have too many archives, or restoring can
  involve reading too many files.
- Reference count per-file, or what?
- Should we make the concept of an 'archive' a higher-level thing, so the user
  can remove old archives at a time?  How does this interact with refcounting?
