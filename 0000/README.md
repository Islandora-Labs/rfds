---
authors: Joe Corall
state: draft
---

# RFD 0000 - Enhanced Event Data for Derivative Microservices

## Required Approvers

* TAG

## Summary

This RFD proposes enhancements to Islandora's event system. Two problems and potential solutions are discussed. Namely:

1. **Incomplete event data forwarded to microservices**: Microservices currently receive limited information (file URLs, MIME types) but lack access to parent node metadata needed for advanced derivatives like cover pages and aggregated PDFs.
2. **Lack of auditing/retries/orchestration for events**: Events are currently popped off the queue and if they fail quickly retried up to some maximum number then are just dropped forever. There's no simple logging to detect what succeeded, failed, or is in flight. There's no mechanism for conditional event emission (e.g., waiting for all child pages before generating parent PDF).

**Two Proposed Solutions**:

1. **Quick win**: Pass complete event data to microservices via an `X-Islandora-Event` header in Alpaca
2. **Long-term**: Replace Alpaca and ActiveMQ with a Drupal-native queue system using Symfony Messenger


## Why

### Current Limitations

The current event flow in Islandora is:

```
Drupal (generates event) → Alpaca (routes to queue) → Microservice (processes derivative)
```

**Problem 1: Limited Event Data**

Alpaca currently sends limited information to microservices ([see code](https://github.com/Islandora/Alpaca/blob/2.x/islandora-connector-derivative/src/main/java/ca/islandora/alpaca/connector/derivative/DerivativeConnector.java#L110-L116)):
- Derivative URL
- Source file URL
- Destination URL
- MIME type
- Arguments

However, the original Drupal event contains significantly more contextual information that microservices cannot currently access.

**Problem 2: Java + ActiveMQ Stack Limits Community Contributions**

The current event lifecycle is managed by **Java-based tools** (Alpaca, ActiveMQ) that are outside the expertise of most Islandora contributors:

- **Alpaca** (Java): Handles message routing and transformation
- **ActiveMQ** (Java): Manages message queues, persistence, and delivery

This creates barriers for implementing common event management features:

| Feature | Current Difficulty | Reason |
|---------|-------------------|---------|
| **Event Retries** | Hard | Requires Java development in Alpaca |
| **Dead Letter Queues** | Hard | Requires ActiveMQ configuration + Alpaca integration |
| **Debugging Failed Events** | Medium | Requires ActiveMQ web console + Java logs |

**Problem 3: Infrastructure Complexity**

The current stack requires deploying and maintaining:
- ActiveMQ server (JVM, configuration, monitoring)
- Alpaca service (JVM, configuration, integration)
- Message persistence and management in ActiveMQ
- Coordination between Drupal, ActiveMQ, and Alpaca

This adds operational complexity and points of failure.

### Use Cases Requiring Enhanced Event Data

#### 1. Cover Page Generation

The cover page microservice needs to generate a PDF cover page with metadata from the parent node:
- Title
- Author
- Publication date
- Other bibliographic information

**Current blocker**: The microservice only receives the file URL, not the parent node's metadata, but that data is in the event Alpaca is consuming.

#### 2. Aggregated PDFs for Paged Content

Generating a single PDF from a book/paged content item requires:
- Access to the IIIF manifest to determine page order
- Parent node information to construct the manifest URL
- Metadata for the aggregated PDF (i.e. title)

**Example implementation need**: [docker-builds/scyllaridae-mergepdf/cmd.sh#L5](https://github.com/lehigh-university-libraries/docker-builds/blob/main/scyllaridae-mergepdf/cmd.sh#L5)

**Current blocker**: The microservice cannot access the parent node or manifest information.

#### 3. Event Emission Timing for Paged Content

Paged content has a special requirement: the "generate aggregated PDF" event should only fire after all child pages have their service files created.

**Current blocker**: Islandora's current event system doesn't support conditional event emission based on completion of related events.

---

## How (Proposed Solutions)


### Short-term Solution: X-Islandora-Event Header

**Goal**: Make the complete event JSON available to microservices without breaking existing functionality.

**Implementation**:

1. **In Alpaca**: Add the complete event JSON as an `X-Islandora-Event` HTTP header when calling microservices
   - Serialize the full event payload to JSON
   - Base64 encode if needed to handle special characters
   - Include as additional header alongside existing parameters

2. **In Scyllaridae**: Parse the header to access event data
   - Already implemented: [scyllaridae/pkg/api/events.go#L104](https://github.com/Islandora/scyllaridae/blob/b5b5ed36322d94864446cd4b5bb8eb01adeead9a/pkg/api/events.go#L104)
   - Microservices can optionally read parent node URLs, metadata, or other contextual information

**Benefits**:
- Non-breaking change (existing microservices continue to work)
- Enables advanced microservices to access contextual data
- Maintains backward compatibility with current Alpaca routing

**Work Required**:
- Modify Alpaca's DerivativeConnector to include X-Islandora-Event header
- Document the header format and usage for microservice developers
- Provide examples of parsing and using the event data


### Long-term Solution: Drupal-Native Queue System

#### Architectural Comparison

##### Current System
```
Drupal (PHP) → ActiveMQ (Java) → Alpaca (Java) → Microservice
               ├─ Limited data          ├─ Routing only       (Missing parent
               ├─ Event retries         ├─ No conditionals     context)
               └─ Hard to debug         └─ Requires Java
```

##### Long-term Proposal (Symfony Messenger)
```
Drupal (PHP) → Database Queue (SQL) → Message Handler (PHP) → Event Subscriber (PHP) → Microservice
               ├─ Full context         ├─ Drupal events      ├─ Full entity API    (Receives all
               ├─ SQL queries          ├─ Retry logic        ├─ Plugin system       needed data)
               ├─ Dead letter queue    ├─ Delayed messages   ├─ Conditional logic
               └─ Easy debugging       └─ PHP config         └─ PHP development
```

**Key Difference**: Long-term moves event management from **Java ecosystem** (Alpaca/ActiveMQ) to **PHP/Drupal ecosystem** (Symfony Messenger/Database), making event features accessible to the Islandora community.
**Goal**: Replace Java-based event infrastructure (Alpaca + ActiveMQ) with **PHP-based event management** in Drupal, making event lifecycle features accessible to the broader Islandora/Drupal community.

**Vision**:
- Events are queued in **Drupal's database** using **Symfony Messenger** (PHP)
- Message handlers dispatch Drupal events that subscribers can act on
- Event subscribers call microservices directly via HTTP with full context
- **All event lifecycle management in PHP** - no Java required

**Why PHP + Database Instead of Java + ActiveMQ?**

Moving event management to PHP and Drupal's database makes advanced features accessible to the community:

| Feature | Java + ActiveMQ (Current) | PHP + Database (Proposed) |
|---------|--------------------------|---------------------------|
| **Event Retries** | Requires Java code in Alpaca | Built into Symfony Messenger |
| **Dead Letter Queues** | Requires ActiveMQ + Alpaca work | Built into Symfony Messenger |
| **Event Dependencies** | Major Alpaca/ActiveMQ rework | Symfony DelayStamp |
| **Debugging** | ActiveMQ console + Java logs | Drupal DB queries + logs |
| **Event History** | ActiveMQ persistence | Database queries |

**Benefits**:

*For the Community*:
- **Drupal-native debugging**: Use familiar tools (database queries, Drush, logs)
- **Extensibility**: Add custom event logic via Drupal event subscribers and plugins
- **Lower barrier to entry**: Modify event behavior without learning Java ecosystem

*For Operations*:
- **Reduced infrastructure complexity**: No JVM, ActiveMQ, or Alpaca services to maintain
- **Simplified deployment**: Two less services to configure and monitor
- **Better observability**: Event queues are database tables that can be queried directly

*For Developers*:
- **PHP developers can contribute**: No Java expertise required for event features
- **Full Drupal API access**: Message handlers run in Drupal context with access to all APIs
- **Entity context**: No serialization/deserialization - work directly with loaded entities
- **Native support for complex workflows**: Delayed messages, deduplication, conditional logic

*Built-in Features*:
- ✅ **Automatic retry and failure handling** via Symfony Messenger
- ✅ **Dead letter queue** for permanently failed messages
- ✅ **Delayed/scheduled execution** via DelayStamp
- ✅ **Extensible plugin system** for derivative scanning
- ✅ **Deduplication** to prevent duplicate processing
- ✅ **Cron-based scanning** to find and queue missing derivatives

**Challenges**:
- Significant architectural change from current Alpaca/ActiveMQ setup
- Migration path for existing installations
  - If no events are being processed at a given time, it's a stateless move so should be pretty painless
- Community adoption and documentation needs
- Queue worker reliability considerations (cron vs. dedicated workers)
- Testing and validation across different repository sizes
- Performance benchmarking needed for high-volume repositories

## Related Work

- [Islandora Documentation Issue #1627](https://github.com/Islandora/documentation/issues/1627) - Previous discussions on event improvements

### Prototype Implementation: islandora_events Module

A working prototype implementation that demonstrates the long-term vision of Drupal-native event queuing has been started. This approach uses **Symfony Messenger** to replace Alpaca and ActiveMQ entirely. It leverages [drupal/sm](https://www.drupal.org/project/sm) as a required dependency.

#### Architecture Overview

**Event Flow**:
```
Entity Hook → Message Queue (Database) → Message Handler → Drupal Event → Event Subscribers → Microservice
```

**Core Components**:

1. **Entity Hooks** (EntityHooks.php) - Capture entity operations (insert/update/delete) and queue messages
2. **Message Bus** - Symfony Messenger handles message routing, retry logic, and delayed execution
3. **Message Handlers** - Process queued messages asynchronously and dispatch Drupal events
4. **Event Subscribers** - Listen to Drupal events and perform derivative work (including HTTP calls to microservices)
5. **Database-backed Queues** - Three Doctrine transport queues:
   - `islandora_derivatives` - Immediate processing queue
   - `islandora_scheduled` - Delayed processing with deduplication (for conditional events)
   - `failed` - Failed message handling with retry support
6. Everything managed in a central log we can create Drupal Views of (links to media/node/file)

**Key Features** (All Implemented in PHP):

1. **Plugin System for Missing Derivatives** (PHP)
   - DerivativeScanner plugins scan for missing derivatives on cron
   - Examples: Jp2ServiceFiles, OcrDerivatives, Thumbnails, FitsMetadata, AggregatedPdfs
   - Each plugin defines SQL queries to find entities missing specific derivatives
   - Automatically queues events for missing derivatives
   - **Community can add new plugins** without Java knowledge

2. **Event Retries with Exponential Backoff** (Symfony Messenger)
   - Automatic retry on failure (configurable max retries)
   - Exponential backoff between retry attempts
   - Failed messages eventually move to dead letter queue
   - **No Java code required** - all configuration in Symfony Messenger

3. **Dead Letter Queue** (Database)
   - Failed messages stored in `islandora_failed_messages` table
   - Can be inspected, debugged, and re-queued via Drush or database queries
   - **Easy troubleshooting** with SQL queries and Drupal tools

4. **Delayed/Conditional Event Emission** (Symfony DelayStamp)
   - Uses Symfony Messenger's DelayStamp to delay message processing
   - Solves the paged content problem: parent PDF generation waits for all child pages
   - Example: 5-minute delay allows child service files to complete before parent aggregation
   - **Configurable delays** in PHP code

5. **Deduplication** (Custom Transport)
   - Custom DeduplicatingDoctrineTransport prevents duplicate messages for the same entity
   - Prevents wasted processing when multiple events fire for same entity
   - Implemented in ~50 lines of PHP

6. **Full Entity Context** (Drupal API)
   - Message handlers run in Drupal, so they have complete access to:
     - Parent/child relationships via entity reference fields
     - All entity fields and metadata
     - IIIF manifest URLs constructed from node data
     - Any Drupal API (Views, Entity API, Field API, etc.)
   - **No need to pass data in headers** - just load the entity

7. **Event History and Monitoring** (Database)
   - All queue messages stored in database tables
   - Can query event history with SQL
   - Potential for Views integration, admin UI, reporting
   - **Observable with Drupal tools** community already knows

#### Extension Pattern: islandora_paged_content Module

The `islandora_paged_content` module demonstrates how derivative-specific modules extend the base system:

**Extension Points**:

1. **Event Subscribers** (PagedContentSubscriber.php):
   - Subscribe to MediaEvent::CREATE and MediaEvent::UPDATE
   - Detect when a page gets its service file
   - Queue parent entity processing with delay

2. **Custom Message Types** (PagedContentMessage.php):
   - Define domain-specific messages (parentEntityId, triggerMediaId, action)
   - Route to separate handlers via configuration

3. **Custom Message Handlers** (PagedContentHandler.php):
   - Execute actions on parent entities
   - Access parent node metadata for PDF generation
   - Call microservices with full context

4. **DerivativeScanner Plugins** (AggregatedPdfs.php):
   - Extend DerivativeScannerBase
   - Define SQL to find paged content missing aggregated PDFs
   - Automatically queue missing derivatives on cron

**Configuration**:

```yaml
# sm.routing.yml - Route message types to transports
'Drupal\islandora_events\Message\IslandoraDerivativeMessage': 'islandora_derivatives'
'Drupal\islandora_paged_content\Message\PagedContentMessage': 'islandora_scheduled'

# sm.transports.yml - Define transport backends
islandora_derivatives:
  dsn: 'doctrine://default?table_name=islandora_derivative_messages'
islandora_scheduled:
  dsn: 'deduplicating-doctrine://default?table_name=islandora_scheduled_messages'
```

**Benefits of This Approach**:

*Event Management Features (PHP-Based)*:
- ✅ **Event retries**: Implemented via Symfony Messenger - no Java required
- ✅ **Dead letter queues**: Database table with SQL query access
- ✅ **Missing event scanning**: Plugin system with 5 working scanners
- ✅ **Conditional event emission**: Delayed messages solve parent/child coordination
- ✅ **Custom event logic**: Event subscribers and plugins in PHP

*Technical Benefits*:
- ✅ **No external dependencies**: Eliminates Alpaca and ActiveMQ
- ✅ **PHP development only**: Community can contribute without Java knowledge
- ✅ **Drupal-native debugging**: SQL queries, Drush commands, familiar logs
- ✅ **Full metadata access**: Handlers run in Drupal with complete entity context
- ✅ **Extensibility**: Other modules can define custom messages, handlers, and scanners
- ✅ **Observability**: All queue operations are in Drupal database and logs


## Open Questions for Community Discussion

1. **Approach Selection**:
   - Should we pursue the short-term X-Islandora-Event header, the long-term Symfony Messenger approach, or both?
   - Does the Symfony Messenger prototype adequately address the use cases, or are there gaps?

2. **Short-term approach (X-Islandora-Event header)**:
   - Is the `X-Islandora-Event` header approach acceptable to the community?
   - What should the header format be? (JSON, base64-encoded, etc.)
   - Should we limit the event data included to avoid header size issues?

3. **Long-term architecture (Symfony Messenger)**:
   - Is there community interest in replacing Alpaca/ActiveMQ with this approach?
   - What's the migration path for existing Islandora 2.x sites?
   - Should we support both queue systems during a transition period?
   - How do we handle sites that have custom Alpaca integrations? Are there any?
   - Is database-backed queueing sufficient, or should we support Redis/other backends?

4. **Extensibility and Plugin System**:
   - Is the DerivativeScanner plugin pattern sufficient for community needs?
   - Should other modules be able to define custom Message types and Handlers?
   - How do we document and support the extension points?
   - Are there use cases not covered by the current prototype?

## Success Criteria

### Short-term Success (X-Islandora-Event Header):
1. Microservices can access parent node metadata via event header
2. Cover page and aggregated PDF use cases are unblocked
3. Existing microservices continue to function without modification
4. Documentation and examples are available for microservice developers

### Long-term Success (Symfony Messenger):
1. Event lifecycle management moved from Java (Alpaca/ActiveMQ) to PHP (Symfony Messenger)
2. **Community members can add event features** (retries, scanners, custom logic) **without Java knowledge**
3. Event retries, dead letter queues, and missing derivative scanning are working in production
4. Paged content conditional event emission works reliably
5. Migration path from Alpaca/ActiveMQ is documented and tested
6. The solution is adopted by the broader Islandora community
7. Existing microservices continue to function without modification
8. Documentation and examples show how to extend the system (plugins, event subscribers)
9. Performance is validated for high-volume repositories
10. **PHP/Drupal developers can debug events** using SQL queries and Drupal logs

## Timeline

### Short-term Work:

1. **Phase 1** (2-4 weeks): X-Islandora-Event header implementation
   - Modify Alpaca's DerivativeConnector to include event data header
   - Update Scyllaridae to parse and use the header
   - Documentation and examples for microservice developers
   - Community review and testing
2. Release a new `islandora_events` module as a MVP of this new PHP-based approach
    - Community feedback and/or sprints during implementation
    - Have paged content merged PDFs leverage this new architecture
    - Community members wanting aggregated PDFs will then have the events for aggregated PDFs using this approach

### Long-term Work:

1. **Phase 1** (Current): Prototype refinement and testing
   - Address any gaps identified by community
   - Comprehensive testing with various content types
   - Performance benchmarking
   - Documentation of architecture and extension points

2. **Phase 2** (2-3 months): Community review and adoption
   - Present to Islandora community (TAG, committers)
   - Gather feedback and iterate on design
   - Create migration guide from Alpaca/ActiveMQ
   - Develop test suite for validation

3. **Phase 3** (3-6 months): Migration tooling and gradual rollout
   - Build migration utilities for existing sites
   - Support dual-mode operation (Alpaca + Symfony Messenger)
   - Pilot deployments at early adopter sites
   - Documentation and training materials

4. **Phase 4** (6-12 months): Broader adoption
   - Include in Islandora Playbook/Starter Site
   - Deprecation plan for Alpaca/ActiveMQ (if applicable)
   - Community support and bug fixes
   - Performance optimization based on production use

## References

- [Alpaca DerivativeConnector](https://github.com/Islandora/Alpaca/blob/2.x/islandora-connector-derivative/src/main/java/ca/islandora/alpaca/connector/derivative/DerivativeConnector.java)
- [Scyllaridae Event Handling](https://github.com/Islandora/scyllaridae/blob/b5b5ed36322d94864446cd4b5bb8eb01adeead9a/pkg/api/events.go)
- [Islandora Documentation Issue #1627](https://github.com/Islandora/documentation/issues/1627)
