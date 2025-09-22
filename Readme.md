# Archive Performance Analysis Report

## Executive Summary

This report analyzes the performance bottlenecks in the archive functionality when executing batch selection operations on files and items in the PRISM system. The investigation reveals several potential performance issues that could contribute to slow request times.

## Investigation Findings

### 1. Database Performance Analysis

#### Item Table Performance

- **Current Item Count**: 16 items in the development database
- **Query Performance**: `SELECT COUNT(*) FROM "Item"` executes in ~0.207ms (very fast)
- **Index Coverage**: The Item table has comprehensive indexing including:
  - Primary key on `id` (btree)
  - Composite indexes on `(orgId, bomTypeId, teamspaceId)`
  - Status-based partial indexes for ACTIVE items
  - JSONB indexes on `customProperties` and `customCategories` (GIN)

#### Query Analysis for Archive Operations

The `findByIds` method uses the following query pattern:

```sql
SELECT * FROM "Item"
WHERE "orgId" = ? AND "id" IN (?) AND "status" = 'ACTIVE'
```

**Performance Assessment**: This query is well-optimized with existing indexes and should perform efficiently even with larger datasets.

### 2. Archive Process Flow Analysis

#### Frontend Operations

1. **File Items Archive**: `archiveSelected()` → `archive(itemIds)` → API call
2. **Regular Items Archive**: `archiveSelected()` → `updateStatus({status: 'ARCHIVED', itemIds})` → API call
3. **Projects Archive**: Similar pattern to regular items

#### Backend Operations (API)

The archive process involves several sequential operations:

1. **Item Lookup** (`findByIds`):

   - Queries Item table with `IN` clause
   - Includes extensive relations via `getSelectInclude()`
   - Reconstructs Item entities using `ItemFactory.reconstruct()`

2. **Item Status Update**:

   - Updates item status to 'ARCHIVED'
   - Creates ItemRevision entries
   - Updates ItemHistory records

3. **Vector Database Update**:

   - Updates vector metadata via `bulkUpdateMetadata()`
   - Executes complex SQL with `unnest()` operations
   - Updates `is_archived` flag in vector store

4. **Transaction Management**:
   - All operations wrapped in database transaction
   - Extended timeout: 60 seconds (vs default 5 seconds)

### 3. Identified Performance Bottlenecks

#### 3.1 Heavy Data Loading in `findByIds`

**Issue**: The `getSelectInclude()` method loads extensive related data:

```typescript
include: {
  fileItem: { include: { jobHistory: true } },
  linkItem: { /* complex nested selects */ },
  itemPhases: { /* phase data */ },
  jItemTags: { /* tag data */ },
  itemSmartTags: { include: { smartTag: true } }
}
```

**Impact**: For batch operations with many items, this creates significant data transfer and processing overhead.

#### 3.2 Vector Database Operations

**Issue**: The `bulkUpdateMetadata()` method performs complex SQL operations:

```sql
UPDATE vector_table AS original
SET teamspace_id = updates.teamspace_id,
    category_id = updates.category_id,
    is_archived = updates.is_archived,
    updated_at = now()
FROM (
  SELECT unnest($1::text[]) as item_id,
         unnest($2::text[]) as teamspace_id,
         unnest($3::text[]) as category_id,
         unnest($4::boolean[]) as is_archived
) AS updates
WHERE original.org_id = $5 AND original.item_id = updates.item_id;
```

**Impact**: This operation can be slow with large batch sizes due to:

- Complex `unnest()` operations
- Multiple array parameter processing
- Vector database connection overhead

#### 3.3 Sequential Processing in Archive Loop

**Issue**: The archive process processes items sequentially in a loop:

```typescript
for (const itemEntity of itemEntities) {
  if (itemEntity.status !== params.status) {
    await itemEntity.updateStatus(params.status);
    // ... additional processing
  }
}
```

**Impact**: Each item update is processed individually, potentially causing delays.

#### 3.4 Frontend Network Operations

**Issue**: The frontend performs multiple operations after archiving:

1. Archive API call
2. `mutateFileItems()` - refetch data
3. `resetSelected()` - clear selection
4. Toast notification

**Impact**: Additional network round-trips and UI updates can contribute to perceived slowness.

### 4. Network Communication Analysis

#### Frontend Operations

- **Single API Call**: The archive operation makes only one API call per batch
- **No Sequential Network Calls**: No `for` or `map` functions with multiple network communications
- **Efficient Batching**: All selected items are processed in a single request

#### Backend Operations

- **Database Transaction**: All operations are wrapped in a single transaction
- **Bulk Operations**: Vector updates use bulk operations rather than individual calls
- **No External API Calls**: No additional network calls during the archive process

## Recommendations

### 1. Immediate Optimizations

#### 1.1 Optimize Data Loading

```typescript
// Create a lightweight version of getSelectInclude for archive operations
private static getArchiveSelectInclude() {
  return {
    id: true,
    status: true,
    teamspaceId: true,
    customCategory: true,
    currentRevisionId: true
    // Remove heavy relations that aren't needed for archiving
  };
}
```

#### 1.2 Add Database Index

Consider adding a composite index for the archive query pattern:

```sql
CREATE INDEX "Item_orgId_id_status_idx" ON "Item" ("orgId", "id", "status");
```

#### 1.3 Optimize Vector Database Operations

- Consider batching vector updates in smaller chunks
- Add connection pooling for vector database
- Implement retry logic for vector database failures

### 2. Medium-term Improvements

#### 2.1 Implement Bulk Status Updates

```typescript
// Instead of individual updates, use bulk update
await prisma.item.updateMany({
  where: {
    orgId,
    id: { in: itemIds },
    status: 'ACTIVE',
  },
  data: { status: 'ARCHIVED' },
});
```

#### 2.2 Add Progress Indicators

Implement progress indicators for large batch operations to improve user experience.

#### 2.3 Implement Async Processing

For very large batches, consider implementing async processing with job queues.

### 3. Long-term Architectural Improvements

#### 3.1 Separate Vector Updates

Move vector database updates to a separate async process to avoid blocking the main archive operation.

#### 3.2 Implement Caching

Add caching for frequently accessed item data to reduce database load.

#### 3.3 Database Partitioning

Consider partitioning the Item table by organization or date for better performance with large datasets.

## Performance Monitoring Recommendations

1. **Add Performance Metrics**: Implement timing logs for each step of the archive process
2. **Database Query Monitoring**: Monitor slow queries and optimize as needed
3. **Vector Database Monitoring**: Track vector database operation performance
4. **User Experience Metrics**: Monitor perceived performance vs actual performance

## Conclusion

The archive performance issues are likely caused by:

1. **Heavy data loading** in the `findByIds` operation
2. **Vector database operations** that may be slow with large batches
3. **Sequential processing** of items in the archive loop
4. **Extended transaction timeouts** indicating potential blocking operations

The most impactful improvements would be:

1. Optimizing the data loading in `findByIds`
2. Implementing bulk database operations
3. Moving vector database updates to async processing
4. Adding proper indexing for archive-specific queries

These optimizations should significantly improve the performance of batch archive operations, especially as the dataset grows.
