# DynamoDB data model (DRAFT)

Turns the GraphQL schema (`graphql/schema.graphql`) and the design spec
(`scouts-activity-tracker/docs/superpowers/specs/2026-10-28-users-and-domain-model-design.md`)
into a table design. Written from access patterns first. Nothing here is
deployed.

## 1. Decisions

| Decision | Choice | Why |
|---|---|---|
| Table layout | One table (`main`) plus one global catalog partition set | Every pattern below is a key lookup or one query. A Group is tiny (tens of Youth, hundreds of Meetings), so cost and hot partitions are not a concern; one-table keeps transactions and deletion simple |
| Tenant isolation | Every partition key starts with `G#<groupId>`; the Group id comes from the caller's identity, never from arguments | A resolver cannot build a key for another Group by accident |
| IDs | ULIDs (sortable, unguessable) | Time-ordered keys; no enumeration |
| Concurrency | Every mutable item has `version`. Writes use `ConditionExpression: version = :expected` and add 1. A failed condition becomes the CONFLICT error | Matches the spec's "block the second save" |
| Capacity | On-demand, point-in-time recovery on, deletion protection on | Small, bursty load; Youth records matter |
| Live sync | AppSync mutation-triggered subscriptions; no DynamoDB streams needed for it | Every change already goes through a mutation |
| Catalog search | Not DynamoDB. Load the catalog JSON into the Lambda that serves `searchCatalog` (a few hundred Badges); revisit OpenSearch only if it grows | DynamoDB cannot do text search; catalog is small, global and read-only |
| Progress | Computed on read from stored Credits, Sign-offs, Manual completions and the pinned Badge version; never stored | Spec: "Progress is computed, never typed in" |
| Photos | None stored | Deferred |

## 2. Keys

Attributes on every item: `PK`, `SK`, `type`, `groupId`, `version`,
`createdAt`, `updatedAt`. Indexes: `GSI1` (`GSI1PK`, `GSI1SK`) and `GSI2`
(`GSI2PK`, `GSI2SK`), both projecting ALL. Both are sparse: only items that
set the attributes appear.

Notation: `g` Group id, `s` Section id, `y` Youth id, `m` Meeting id,
`a` Activity id, `ma` Meeting activity id.

### Group partition: `PK = G#g`

| Entity | SK | Notes |
|---|---|---|
| Group | `META` | name |
| Section | `SECTION#<s>` | name, status |
| Section name lock | `SECTIONNAME#<lowercased name>` | Written in the same transaction as the Section. Enforces "unique by name". Rename = delete old lock, put new lock, update Section |
| Leader | `LEADER#<l>` | name, isAdmin, status, `sectionIds` |
| Location | `LOCATION#<id>` | name, status |
| Location name lock | `LOCATIONNAME#<lowercased name>` | Same pattern; stops duplicates in the combo box |
| Completion reason | `REASON#<id>` | name, status |
| Reason name lock | `REASONNAME#<lowercased name>` | Same pattern |
| Guardian | `GUARDIAN#<id>` | name only |

Small collections, read whole with `begins_with(SK, "SECTION#")` and so on.

### Identity and sign-up (global lookups, not in a Group partition)

| Item | PK / SK | Notes |
|---|---|---|
| Login lookup | `USER#<cognitoSub>` / `PROFILE` | `groupId`, `leaderId`. Resolvers read this first, then the Leader item, to get the Group and Section access. Archiving a Leader also disables the Cognito user |
| Sign-up link | `SIGNUP#<sha256(token)>` / `LINK` | `groupId`, `leaderId`, `expiresAt`, `ttl` = created + 72 h. Only the hash is stored. Single use: deleted on redemption. Regenerating deletes the old one. DynamoDB TTL cleans up expired links, but the resolver checks `expiresAt` itself because TTL is lazy |

### Youth partition: `PK = G#g#Y#y`

| Entity | SK | Index keys |
|---|---|---|
| Youth | `META` | `GSI2PK=G#g#YOUTH`, `GSI2SK=<lowercased name>#<y>` |
| Membership | `MEMBERSHIP#<startDate>#<s>` | `GSI1PK=G#g#S#<s>`, `GSI1SK=MEMBER#<startDate>#<y>` |
| Pickup authorization | `PICKUP#<id>` | `GSI2PK=G#g#GUARDIAN#<guardianId>`, `GSI2SK=<y>` (reverse lookup for "is this Guardian still used?") |
| Credit | `CREDIT#<ma or NONE>#<itemId>#<creditId>` | If linked to a Meeting activity: `GSI2PK=G#g#MA#<ma>`, `GSI2SK=<itemId>#<y>` |
| Sign-off | `SIGNOFF#<itemId>` | One per Item; `revokedAt` set when undone |
| Manual completion | `MANUAL#<badgeId>#<id>` | `revoked` flag |
| Awarding | `AWARD#<badgeId>` | One per Badge; written with a condition so it cannot repeat or be undone |

Reading `PK = G#g#Y#y` returns everything needed to compute that Youth's
progress in one query. Permanent deletion starts from the same query.

A suggested Credit has a deterministic id, `<ma>-<itemId>`, so a conditional
put (`attribute_not_exists`) never re-creates one that was rejected: the
"rejected is not re-suggested" rule costs nothing.

### Meeting partition: `PK = G#g#M#m`

| Entity | SK | Index keys |
|---|---|---|
| Meeting | `META` | start, end, location ids, `sectionIds` |
| Meeting-section marker | `SEC#<s>` | `GSI1PK=G#g#S#<s>`, `GSI1SK=MEETING#<startsAt>#<m>`. One per Section, so a joint Meeting shows in each Section's list |
| Meeting activity | `MA#<order padded>#<ma>` | Ordered by key. Reordering rewrites the keys in one transaction |
| Attendance | `ATT#<y>` | `GSI1PK=G#g#Y#<y>`, `GSI1SK=ATT#<startsAt>#<m>` (a Youth's history; also used by permanent delete) |

The roster for a Meeting is not stored. It is computed: for each Section,
query `GSI1` (`G#g#S#<s>`, `begins_with MEMBER#`) and keep Memberships that
cover the Meeting date. Checked-in Youth with no Membership that day cannot
exist because check-in uses the roster.

### Activity partition: `PK = G#g#A#a`

| Entity | SK | Index keys |
|---|---|---|
| Activity | `META` | `GSI2PK=G#g#ACTIVITY`, `GSI2SK=<lowercased title>#<a>`; `sectionIds` as an attribute |
| Requirement link | `LINK#<itemId>` | `GSI2PK=G#g#ITEM#<itemId>`, `GSI2SK=<a>`. Holds `proposed`, `accepted`, `rationale` |

The Activity list for a Section reads `G#g#ACTIVITY` and filters on
`sectionIds`. That is a filter over hundreds of items at most; fine here.

### Catalog (global, read-only): `PK = CATALOG`

| Entity | PK / SK |
|---|---|
| Badge | `CATALOG` / `BADGE#<badgeId>` |
| Badge version | `CATALOG#<badgeId>` / `V#<number padded>` |

A Badge version holds its Requirement groups and Items as one nested
document (they are only ever read together). Item ids are stable inside a
Badge. The catalog is written only by the loader, never by the API role.
`family`, `sectionNames` and `previousStages` live on the Badge item.

## 3. Access patterns

| # | Pattern | Operation |
|---|---|---|
| 1 | Resolve the caller | GetItem `USER#sub`, then GetItem Leader |
| 2 | Sections, Leaders, Locations, Reasons, Guardians | Query `G#g`, `begins_with` |
| 3 | Youth list with search, filter by Section, past members | Query `GSI2` `G#g#YOUTH` (name order); for a Section filter, Query `GSI1` `G#g#S#s` `MEMBER#` and BatchGet Youth; "past" = Membership with an `endDate` |
| 4 | One Youth with Memberships and pickups | Query `G#g#Y#y` |
| 5 | Meetings list by Section and date range | Query `GSI1` `G#g#S#s`, SK between `MEETING#from` and `MEETING#to` |
| 6 | One Meeting with activities | Query `G#g#M#m` (excluding `ATT#`) |
| 7 | Attendance for a Meeting | Query `G#g#M#m`, `begins_with ATT#` |
| 8 | Check in | Transaction: put `ATT#y` (condition: absent), then put suggested Credits |
| 9 | Check out / undo / remove check-in | UpdateItem `ATT#y` with version condition |
| 10 | Close-out | Query `ATT#`, batch-update those without `checkedOutAt`, 100 per transaction |
| 11 | Credits to review for a Meeting activity | Query `GSI2` `G#g#MA#ma` |
| 12 | Toggle all | Query pattern 11, update each Credit in transactions of up to 100 |
| 13 | Youth progress | Query `G#g#Y#y`, join with the pinned Badge versions, compute |
| 14 | Progress of every Youth on a Badge | Pattern 13 for each Youth in scope (tens), compute only that Badge |
| 15 | To award | Pattern 14 across Badges for the caller's Sections, keep COMPLETE with no `AWARD#`. See risk 1 |
| 16 | Awarded history | Query each Youth's `AWARD#` (begins_with) or, if slow, a `GSI2` entry on the Awarding |
| 17 | Activity with links; find Activities for an Item | Query `G#g#A#a`; Query `GSI2` `G#g#ITEM#item` |
| 18 | Suggested Credits at check-in | Query each Meeting activity's Activity `LINK#`, keep accepted links valid for the Youth's Section |
| 19 | Which Meetings are in a Section's list | pattern 5 |
| 20 | A Youth's attendance history | Query `GSI1` `G#g#Y#y`, `begins_with ATT#` |
| 21 | Delete Meeting | Query `G#g#M#m`, delete all items; for Credits on its Meeting activities (pattern 11) remove the Meeting link and keep the Credit |
| 22 | Permanently delete Youth | See section 5 |
| 23 | Duplicate Meeting | Read `META` and `MA#`; write new partition. No `ATT#` |
| 24 | Catalog Badge and versions | GetItem / Query `CATALOG` partitions; search in memory |

## 4. Resolvers and where logic lives

| Kind | Operations |
|---|---|
| APPSYNC_JS direct DynamoDB | Single-item reads and version-checked updates: `me`, `group`, list queries, `renameSection`, `renameLocation`, `setLocationStatus`, `checkOut`, `undoCheckOut`, `removeCheckIn`, `setCreditStatus`, `signOff`, `undoSignOff`, `removeLink` |
| APPSYNC_JS pipeline | Transactions: `createSection`/`renameSection` (name lock), `createLocation`, `moveUpYouth`, `createMeeting`, `updateMeeting`, `checkIn` |
| Lambda | Anything that fans out or computes: progress queries, `toAward`, `closeOutMeeting`, `setCreditsStatusForItem`, `deleteYouth`, `deleteMeeting`, `duplicateMeeting`, leader invite, archive and reactivate (these also call Cognito), `markBadgeComplete`, `awardBadge` (checks COMPLETE first), `searchCatalog`, `suggestRequirementLinks` (Bedrock) |

Authorization, in every resolver: caller is an ACTIVE Leader; the Group id
comes from the Leader item; Section access is checked against the Leader's
`sectionIds` for any action on a Youth, Meeting or Credit; Admin-only
operations also check `isAdmin`.

Rules that need a condition, not just a write:
- At least one Admin: `archiveLeader` and `updateLeader` (demote) run in a
  transaction that also checks the Group's `adminCount` on `META`, kept up to
  date by the same transactions.
- Retire versus delete a Section: `deleteSection` checks the Section's
  `youthCount`, `leaderCount` and `meetingCount` counters on its item and
  fails if any is above zero. The counters are updated in the transactions
  that change Memberships, Leader access and Meetings.
- A Credit cannot be confirmed for an Item the Youth's Section is not valid
  for.
- `awardBadge` fails if an `AWARD#badgeId` item exists.

## 5. Permanent deletion of a Youth

A Lambda runs this idempotently, so a retry after a partial failure is safe:

1. Mark the Youth `deleting` (hides it from every list and blocks writes).
2. Query `G#g#Y#y` and delete every item in batches of 25.
3. Query `GSI1` `G#g#Y#y` for `ATT#` items; delete each Attendance item in its
   Meeting partition.
4. For each Guardian the Youth referenced (read before step 2), Query `GSI2`
   `G#g#GUARDIAN#id`. If no other pickup remains, delete the Guardian.
5. Publish `onYouthDeleted`.

Because nothing outside the database holds Youth data (no photos, and Bedrock
never sees Youth data), the database and its backups are the whole scope.
Point-in-time recovery keeps deleted items restorable for its window (35 days
maximum), so the privacy notice must say so, or PITR must be reconsidered.

## 6. Risks and open questions

1. **"To award" and Badge-wide progress** compute across every Youth on each
   call. Fine at tens of Youth. If it gets slow, materialize a `COMPLETE`
   marker on a Youth's `PROGRESS#<badgeId>` item whenever a Credit, Sign-off
   or Manual completion changes, and index it for the list.
2. **Counters** (`youthCount`, `leaderCount`, `meetingCount`, `adminCount`) must
   be maintained in the same transactions as the changes they count. Easy to
   miss; each needs a test.
3. **PITR versus permanent deletion.** See section 5. Decide the retention
   window and wording.
4. **Catalog loading.** A loader job and a source file format are needed;
   content comes from Scouts Canada materials only.
5. **Activity list filter.** Reads all Activities and filters. Add a
   per-Section index if a Group grows past a few thousand.
6. **Meeting `sectionIds` changes** rewrite the `SEC#` markers and the roster
   mid-life; define what happens to existing Attendance for a removed Section
   (assumed: blocked once any Attendance exists).
7. **Subscription filtering.** AppSync subscription arguments filter by Group
   only through the resolver; confirm the enhanced filtering setup in the
   first deploy.
