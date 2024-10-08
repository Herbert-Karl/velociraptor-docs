---
title: Linux.Forensics.Journal
hidden: true
tags: [Client Artifact]
---

Parses the binary journal logs. Systemd uses a binary log format to
store logs. You can read these logs using journalctl command:

`journalctl --file /run/log/journal/*/*.journal`

This artifact uses the Velociraptor Binary parser to parse the
binary format. The format is documented
https://systemd.io/JOURNAL_FILE_FORMAT/


<pre><code class="language-yaml">
name: Linux.Forensics.Journal
description: |
  Parses the binary journal logs. Systemd uses a binary log format to
  store logs. You can read these logs using journalctl command:

  `journalctl --file /run/log/journal/*/*.journal`

  This artifact uses the Velociraptor Binary parser to parse the
  binary format. The format is documented
  https://systemd.io/JOURNAL_FILE_FORMAT/


parameters:
- name: JournalGlob
  type: glob
  description: A Glob expression for finding journal files.
  default: /{run,var}/log/journal/*/*.journal

- name: OnlyShowMessage
  type: bool
  description: If set we only show the message entry (similar to syslog).

- name: AlsoUpload
  type: bool
  description: If set we also upload the raw files.

export: |
    LET JournalProfile = '''[
    ["Header", "x=&gt;x.header_size", [
      ["Signature", 0, "String", {
          "length": 8,
      }],
      ["compatible_flags", 8, uint32],
      ["incompatible_flags", 12, Flags, {
        type: uint32,
        bitmap: {
          COMPRESSED_XZ: 0,
          COMPRESSED_LZ4: 1,
          KEYED_HASH: 2,
          COMPRESSED_ZSTD: 3,
          COMPACT: 4,
        }
      }],
      ["IsCompact", 12, BitField, {
         type: uint32,
         start_bit: 4,
         end_bit: 5,
      }],
      ["header_size", 88, "uint64"],
      ["arena_size", 96, "uint64"],
      ["n_objects", 144, uint64],
      ["n_entries", 152, uint64],
      ["Objects", "x=&gt;x.header_size", "Array", {
          "type": "ObjectHeader",
          "count": "x=&gt;x.n_objects",
          "max_count": 100000
      }]
    ]],

    ["ObjectHeader", "x=&gt;x.size", [
     ["Offset", 0, "Value", {
        "value": "x=&gt;x.StartOf",
     }],
     ["type", 0, "Enumeration",{
         "type": "uint8",
         "choices": {
          "0": OBJECT_UNUSED,
          "1": OBJECT_DATA,
          "2": OBJECT_FIELD,
          "3": OBJECT_ENTRY,
          "4": OBJECT_DATA_HASH_TABLE,
          "5": OBJECT_FIELD_HASH_TABLE,
          "6": OBJECT_ENTRY_ARRAY,
          "7": OBJECT_TAG,
         }
     }],
     ["flags", 1, "uint8"],
     ["__real_size", 8, "uint64"],
     ["__round_size", 8, "Value", {
         "value": "x=&gt;int(int=x.__real_size / 8) * 8",
     }],
     ["size", 0, "Value", {
         "value": "x=&gt;if(condition=x.__real_size = x.__round_size, then=x.__round_size, else=x.__round_size + 8)",
     }],
     ["payload", 16, Union, {
         "selector": "x=&gt;x.type",
         "choices": {
             "OBJECT_DATA": DataObject,
             "OBJECT_ENTRY": EntryObject,
         }
     }]
    ]],
    ["DataObject", 0, [
      ["payload", "x=&gt;DataOffset",  String]
    ]],

    # This is basically a single log line -
    # it is really a list of references to data Objects
    ["EntryObject", 0, [
      ["seqnum", 0, "uint64"],
      ["realtime", 8, "uint64"],
      ["monotonic", 16, "uint64"],
      ["_items", 48, Array, {
         "type": EntryItem,
         "count": 50,
         "sentinel": "x=&gt;NOT x.object",
      }],
      ["_items_compact", 48, Array, {
         "type": CompatEntryItem,
         "count": 50,
         "sentinel": "x=&gt;NOT x.object",
      }],
      ["items", 0, Value, {
         value: "x=&gt;if(condition=IsCompact, then=x._items_compact, else=x._items)",
      }]
    ]],

    ["CompatEntryItem", 4, [
       ["object", 0, uint32]
    ]],
    ["EntryItem", 16, [
       ["object", 0, "uint64"],
    ]],
    ]
    '''

    -- We make a quick pass over the file to get all the OBJECT_ENTRY
    -- objects which are all we care about. By extracting Just the
    -- offsets of the OBJECT_ENTRY Objects in the first pass we can
    -- free memory we wont need.
    LET Offsets(File) = SELECT Offset
      FROM foreach(row=parse_binary(filename=File, profile=JournalProfile,
         struct="Header").Objects)
      WHERE type = "OBJECT_ENTRY"


    -- Now parse the ObjectEntry in each offset
    LET _ParseFile(File) =
      SELECT Offset,
        parse_binary(
         filename=File, profile=JournalProfile,
         struct="ObjectHeader", offset=Offset) AS Parsed
      FROM Offsets(File=File)


    -- Extract the timestamps and all the attributes
    LET ParseFile(File) = SELECT * FROM foreach(row={
       -- If the file is compact the payload is shifted by 8 bytes.
       SELECT parse_binary(
          filename=File,
          profile=JournalProfile,
          struct="Header").IsCompact * 8 + 48 AS DataOffset,
       parse_binary(
          filename=File,
          profile=JournalProfile,
          struct="Header").IsCompact AS IsCompact
       FROM scope()

    }, query={
      SELECT File, Offset,
         timestamp(epoch=Parsed.payload.realtime) AS Timestamp,
         {
           SELECT parse_binary(
              filename=File,
              profile=JournalProfile,
              struct="ObjectHeader",
              offset=_value).payload.payload AS Line
           FROM foreach(row=Parsed.payload.items.object)
           WHERE Line
        } AS Data
      FROM _ParseFile(File=File)
    })

sources:
- query: |
    SELECT * FROM foreach(row={
      SELECT OSPath FROM glob(globs=JournalGlob)
    }, query={
      SELECT *, if(condition=OnlyShowMessage,
          then=filter(list=Data, regex="^MESSAGE=")[0], else=Data) AS Data,
          if(condition=AlsoUpload, then=upload(file=File)) AS Upload
      FROM ParseFile(File=OSPath)
    })

</code></pre>

