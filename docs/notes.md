## When to Use

Use for data that is:

-   Hierarchical
-   Closed set

Examples of data that fits the paradigm:

-   Content
-   Products

Don't use for data that is:

-   Non-hierarchical
-   Open set

Examples of data that doesn't fit the paradigm:

-   Customers
-   Orders
-   Activty Logs

## What names

-   UUID = Universal Unique ID, a random 64-bit key
-   UUHN = Universal Unique Hierarchical Name, the unique combination of parent ID and name

## Tree Copying Algorithm

-   **Content Structure**: The content is structured as a tree, represented by a hierarchy of XML
    documents.
-   **XML Document Details**:
    -   Each XML document contains **attributes**, some of which may include **UUIDs**.
    -   These UUIDs serve as pointers to other XML documents, establishing **relationships** between
        them.
-   **Objective**: Develop an algorithm to copy entire trees of XML documents along with their
    interconnecting relationships.
-   **Copying Methodology**:
    -   The copying process involves transferring the documents **one group at a time**.
    -   Each XML document encompasses a list of UUIDs that reference both its **parent document**
        and any **related documents**.
-   **Preconditions for Copying**:
    -   An XML document can only be copied **after** all its associated parent and relationship
        documents (as identified by UUIDs) have been marked as copied.
-   **Date Checking**:
    -   Check date when syncing items to see if they need updating.

## Custom SQL queries

Custom SQL SELECT queries can be passed to the content layer to retrieve a set of UUIDs and their
XML documents.

## UUIDs

UUIDs are all defined as attributes `uuid="..."`.

## Deleting Content

Content can only be deleted if there are no references to its UUID.

## Naming Constraints

Names for any schema's entities can be constrained with a regexp like `name-match="..."`.

## Object-Relational Mapping

ORM is easier because XML is rendered directly as objects.

## Database Stability

The database layer becomes stable and doesn't change with changing content definitions.

## Name-Only Nodes

It's possible to define nodes as name only containers like directories in a file system that don't
have associated XML documents. Just allow the XML to be null.

## One Table

When you split the table into multiple tables, then you can't have consistent UUID references for
all content.

## Combining Databases

It should be possible to combine two databases and renumber their UUIDs.

## Namespaced CMS

Create a CMS like WordPress but all content goes into top-level namespaces in DB and file system.
The namespaces are registered. Enables proper cleanup.

## Levels of Generality

1. Spreadsheet - lowest level. No table definitions.
2. SQL - medium level. Table definitions in one complex layer.
3. XML in SQL - high level. Separates concern of definition from manipulation.

XML in SQL is more general.

-   Layered architecture where the lower layer (SQL) described how content is maniuplated and upper
    layer (XML) describes how content is defined.Separating those concerns is beneficial because it
    allows manipulation of content separate from its definition.
-   SQL defines UUID, UUHN, schemas, schema versions, document hierarchy, and command queue/log.
-   XML defines data types and relationships which are mirrored in SQL.

## Antipatterns

There are ways to define XML in SQL that work worse. These are design anti-patterns.

-   No UUID - Isn't portable between servers. Not easy to define relationships.
-   No UUHN - No meaningful way to refer to content.
-   No schema - Content isn't validated. Must parse XML to identify type.
-   JSON instead of XML - Doesn't validate. No XPath queries.
-   No versioning of schemas - Hard to change data definition.
-   No queue/log - Lacks conflict management and auditability.
-   Storing too much data - Logs should not be stored here.
-   Deleting data - Use archiving instead.
-   Saving all versions - Just save significant changes.
-   No PHP direct interface - REST APIs are too slow.
-   Using database queries directly - Must use interface.

## Hash

Store hash of data to show it wasn't changed? Or no?

## Read-Only

Offer a read-only setting that doesn't allow updates? Or no?

## Database

-   XML-specific databases may be best.
-   PostgreSQL has better XML support than MySQL.

## Logging

Universal logging format. DB table contains:

-   Log type
-   XML of log

XML has versioned schemas just like the above. DB table is partitioned or split into YYYY, YYYYMM,
or YYYYMMDD tables.

## Non-XML

What about storing non-XML blobs? That would be more difficult. Could be stored in external binary
table which is OK because not searched.

## Auto versioning

1. Store a change count as integer.
2. Match all words in source and target with each change.
3. Do a diff (all old words not in new and all new words not in old).
4. Add diff count to change count.
5. When change count exceeds a threshold of 20%, automatic version.

## Like filesystem

The UUHN is like a filesystem. In fact, storing documents in a hierarchy is also like a filesystem.
The filesystem doesn't care what's in the document and can copy folders. That is a separation of
concerns between document storage and document definition just like this system.

## Naming elements and attributes

According to [W3Schools](https://www.w3schools.com/xml/xml_elements.asp) should should only use word
characters and underscores when naming attributes and elements. Avoid hyphen, dot, and colon, and
non-English letters.

This also makes valid identifiers in most languages such as PHP.

## Unique natural keys

Store a table with:

-   Schema ID
-   Key name
-   Key value

The combination must be unique. That allows multiple natural unique keys per schema.

## PHP processing

1. SimpleXML - DOM parser, reads and writes, supports XPath, doesn't validate
2. DOMDocument - DOM parser, reads and writes, supports XPath, validates
3. XMLParser - SAX parser
4. XMLReader - Pull parser

SimpleXML and DOMDocument are both part of PHP core. Use SimpleXML if you don't need validation or
else use DOMDocument.

DOMDocument validates with:

-   XML Schema 1.0 (preferred)
-   DTD
-   RELAX NG but not in compact syntax

## Relative references

When you copy trees between versions, UUIDs and absolute UUHNs don't point to the current version
anymore. The solution is use relative UUHNs just like relative paths on a hard drive.

For example, if you copy /CPA/2023/AUD to /CPA/2024/AUD, then links to /CPA/2023/AUD in
/CPA/2023/BEC won't work. But if you use the reference ../AUD and copy AUD and BEC to 2024, then the
relative link will still work.

## Attributes

Only use attributes for:

-   Unique values
-   Fixed (or maximum) length less than 64 characters
-   Matching a simple regexp or enumerated values

Here are the top 20 examples of the kinds of metadata that one would commonly store in XML
attributes:

1. ID - A unique identifier for an element

2. Type - The type or category of an element

3. Name - The name of an element or attribute

4. Namespace - A URI that identifies the source of an element

5. Language - The language of the element's content

6. Created - The date/time the element was created

7. Modified - The date/time the element was last modified

8. Author - The author or creator of the element

9. Version - The version number of the element

10. Status - The status of the element (e.g. draft, final, etc.)

11. Format - The format of the element's content (e.g. text, html, etc.)

12. Source - The source of the element's content

13. Rights - Copyright or usage rights for the element

14. Relation - How the element relates to other elements

15. Coverage - The spatial or temporal scope of the element

16. Publisher - The publisher of the element

17. Contributor - Contributors to the element's content

18. Description - A description of the element

19. Subject - The subject or topic of the element

20. Datatype - The datatype of the element's content (e.g. string, numeric, etc.)

The key principle is that metadata (data about data) should be stored as attributes, while the
actual data itself should be stored as elements. Attributes are designed to contain data related to
a specific element.

For everything else, use elements.

## Archiving and purging

Change the archive from a boolean `is_archived` to a nullable date field `archive_date`. Then delete
old records that are more than a specified period such as one month or one year

## Constraints

A question is how it is possible to replace higher level constraints like:

-   unique keys
-   ordered records

The answer is to use higher level documents to describe them. For example, if the XML documents are
questions to be put in a particular order, or if each question stem must be unique, then these fax
must be described in a higher level parent document that references the questions as children. This
document can provide constraints on order or uniqueness of names.

## Import/export

Each blob of XML in a tree is identified by its UUID. So it's possible to export a subtree to a file
then import that same subtree to a different database server.

## No copy

Allow subnodes to me marked as "no copy" so they won't be copied as part of a subtree, only
directly.

## Tagging

In addition to UUID and UUHN, consider making a Tags table with unique-named tags. Then a TagMaps
table where each piece of content can have one or more tags. Or is this redundant to "tag
documents"?

## Criticism of typical database operations

-   Everything is a special case (data is not uniform).
-   You have to define data in order to store it (no separation of concerns).
-   Complexity of triggers, etc.
-   Error-prone and irreversible.
-   No activity logging.
-   Too complex to serve as intermediary between apps.
-   Tables pretending to be hierarchies.
-   No versioning.
-   Not flexibly.
-   IDs are relative.
-   Can't do many-to-many mappings.
-   Doesn't handle ubiquitous concerns (UUIDs, transactions, etc.)
-   Can't easily relate data on different servers.
-   Incremental IDs are predictable.

## NoSQL

MongoDB does XML/JSON schemas.

## Replication

A single content server could host all content. But for efficieny, it could be synchronized to
several replication databases/hosts. Every hour, the content could be resynchronized. All active
content could be synchronized by comparing the update date.

Content could be limited to just one course.

## Traceability

Each unit of content refers to the logged command that created it. Updates would require a separate
table for many-to-many.

## References

-   https://en.wikipedia.org/wiki/XML_database
-   https://xml.coverpages.org/xmlIntro.html
-   https://unkey.dev/blog/uuid-ux
