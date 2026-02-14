### Core Design and Architecture Questions

1. **Hierarchical Structure**: The design emphasizes a hierarchical XML tree with parent-child relationships enforced via `parent_id` and unique names within parents. How should we handle cyclic dependencies or prevent infinite loops in the hierarchy (e.g., a node indirectly parenting itself)?

2. **Content Versioning**: Content versioning is handled by adding version nodes (e.g., "Course -> 2024") to the XML tree. Should we enforce automatic versioning based on change thresholds (as mentioned in notes.md, e.g., 20% diff), or leave it manual? If automatic, what metrics (word diff, structural changes) should trigger a new version?

3. **Schema Versioning**: Schemas are versioned in the database, but how should we manage backward compatibility? For example, should older content automatically migrate to new schema versions during reads/updates, or require explicit migration commands?

4. **UUID vs. Semantic Names**: Content can be referenced by UUID (non-semantic) or hierarchical paths (semantic). In what scenarios should the system prioritize one over the other (e.g., API queries, internal traversals)? Should we support partial path resolutions for fuzzy searches?

5. **Relative References**: Notes.md mentions using relative UUHNs (like "../AUD") for versioning/copying. How should the system resolve these during tree copies or queries, and what validation rules (e.g., preventing invalid relatives) are needed?

6. **Non-XML Blobs**: The design focuses on XML, but notes.md discusses storing non-XML (e.g., binaries) externally. Should we integrate a binary reference table within the database for traceability, or keep it fully external? If internal, how to handle versioning for binaries?

### Data Management and Integrity Questions

7. **Validation**: Content is validated against schemas in XML rather than SQL. Should validation occur only on insert/update, or also on read (e.g., for drifted data)? What error handling (e.g., partial failures) should the `ContentValidatorInterface` support?

8. **Unique Constraints**: Beyond parent-name uniqueness, notes.md suggests regex constraints for names and natural unique keys per schema. How should these be enforced—at the database level, application level, or both? Should schemas define custom uniqueness rules (e.g., unique stems in MCQs)?

9. **Archiving and Deletion**: Archiving uses a date field instead of boolean, with purging after a period. What default purging period (e.g., 1 year) should we set, and should it be configurable per schema or content type? How to handle orphaned references during purging?

10. **Referential Integrity**: Deletion is only allowed if no UUID references exist. How should we detect and prevent dangling references during operations like archive or copy? Should we add a reference count column to the `content` table for efficiency?

11. **Tagging**: Notes.md suggests a Tags table for additional metadata. Is this necessary beyond hierarchical names and relationships, or redundant? If included, should tags support inheritance from parents or querying across hierarchies?

12. **Import/Export**: Subtrees can be exported/imported via files. Should we support format options (e.g., ZIP with XML files, single monolithic XML)? How to handle UUID conflicts during imports (e.g., renumbering as in notes.md)?

### Command Queue and Processing Questions

13. **Command Types**: The queue supports insert/update/delete/archive/copy/replace. Are additional types needed (e.g., move, merge, sync with external sources)? How should we prioritize commands (e.g., FIFO vs. priority based on type)?

14. **Processing**: A background worker processes the queue. Should we support transactional batches (e.g., atomic groups of commands) to handle failures without partial states? What retry logic (e.g., exponential backoff) for failed commands?

15. **Logging and Auditing**: Commands log changes for traceability. Should we link content to its creating command (e.g., via a `created_by_command_id` field)? How detailed should logs be (e.g., full diffs vs. summaries), and should we partition logs by date as suggested?

16. **Concurrency**: With a queue, how to handle concurrent modifications (e.g., multiple workers)? Should we add locking mechanisms at the content level or rely solely on queue status?

### API and Integration Questions

17. **REST API vs. PHP Library**: The design provides both. In what use cases should direct PHP access be restricted (e.g., for security)? Should the REST API support bulk operations (e.g., batch creates) to reduce overhead?

18. **Custom Queries**: Notes.md allows custom SQL SELECT for UUID/XML retrieval. Should we expose this via the API (with safeguards against injection), or limit to internal use? How to integrate with XPath for XML-specific queries?

19. **Integration with Consumers**: Consumers can pull XML directly, bypassing table rendering. What fallback mechanisms (e.g., on-demand rendering) if consumers aren't updated? Should we provide XML-to-JSON conversion in the API for broader compatibility?

20. **Binary Handling**: Binaries are referenced externally. Should the API include endpoints for uploading/retrieving binaries, or integrate with an external service (e.g., S3)? How to ensure consistency (e.g., validate references during content creation)?

### Performance and Scalability Questions

21. **Database Choice**: Notes.md prefers PostgreSQL for XML support over MySQL. Should we optimize for XML functions (e.g., XPath indexing)? What sharding strategy (e.g., by course/namespace) for large hierarchies?

22. **Tree Copying**: The algorithm copies groups after parents/relations. For large trees, should we support asynchronous copying or partial copies? How to handle date-based syncing (as in notes.md) for replication?

23. **Replication**: Notes.md suggests hourly sync to replicas. What conflict resolution (e.g., last-write-wins) for concurrent changes across replicas? Should active content be limited per replica (e.g., by course)?

24. **Read-Only Mode**: Notes.md mentions a read-only setting. Should this be per-user, per-schema, or global? How to enforce it without impacting queue processing?

### Security and Extensibility Questions

25. **Access Control**: The design lacks explicit security. Should we add role-based access (e.g., read/write per namespace) at the API level? How to secure UUIDs from enumeration attacks?

26. **Extensibility**: Supported types include courses, MCQs, etc. How should new types be added (e.g., schema registration endpoint)? Should schemas allow custom constraints like ordering or "no-copy" flags?

27. **Attributes vs. Elements**: Notes.md guidelines for attributes (unique, fixed-length). Should schemas enforce this automatically, or rely on validation? What about internationalization (e.g., non-English names)?

28. **Traceability**: Notes.md suggests linking content to commands. Should we extend this to user attribution (e.g., who issued the command) for auditing?

29. **Error Handling**: Across interfaces, what standardized error formats (e.g., XML/JSON with codes)? Should failed validations provide detailed paths to issues?

30. **Long-Term Evolution**: For long-term integration, how to deprecate old schemas without breaking consumers? Should we support A/B testing of schema versions?

