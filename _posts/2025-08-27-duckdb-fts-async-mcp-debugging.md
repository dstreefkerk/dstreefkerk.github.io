---
layout: post
title: "When DuckDB FTS Meets Async MCP: An AI-Assisted Debugging Nightmare"
date: 2025-08-27
categories: debugging database mcp python
tags: duckdb mcp async debugging race-conditions claude-code
author: Daniel Streefkerk
excerpt: "A cautionary tale about vibe coding utilities that combine synchronous and asynchronous code."
---

## The Deceptively Simple Setup

I was using Claude Code to build a MCP server into Detection Nexus, a security detection rule database that I'm working on. After some research, the architecture seemed straightforward:

- FastMCP framework for protocol handling, I'd used it before on previous projects
- DuckDB for fast queries with Full-Text Search (FTS)
- BM25 scoring for relevance ranking

The server worked perfectly in isolation and the core repository features worked perfectly. Then I started on the MCP server, and the basic tools looked good, until I started the implementation of the actual search tool.

## The Mystery

Once I connected the MCP Server to a client like Claude Desktop, the first search query would hang indefinitely. Not a slow response - an infinite hang. The MCP client would wait forever for a response that would never arrive. I even left it running overnight and came back to find it still waiting after 6+ hours.

This pattern was consistent:

1. Server already running → Query works immediately ✅
2. Server restart → First query hangs forever ❌
3. Cancel and immediately retry → Query works immediately ✅

The critical detail: server logs showed the query completed successfully within milliseconds. DuckDB returned results almost instantly, everything appeared fine server-side. But somehow, the response never reached the MCP client.

```bash
# Server logs showed query completed in milliseconds:
2025-08-27 09:58:24.609 - Tool call received: "malware", limit=5
2025-08-27 09:58:24.662 - FTS validation completed (52ms) ✅
2025-08-27 09:58:24.741 - Search completed, 5 rules returned (132ms total) ✅
2025-08-27 09:58:24.741 - === MCP TOOL CALL COMPLETED === ✅
# But client hung indefinitely waiting for response
```

This critical detail didn't immediately register with either me or Claude Code. It took a few days of chasing dead ends until I realised what was going on.

## The Dead Ends That Cost Me Days

### Dead End #1: The FTS Scapegoat

My first assumption was that DuckDB FTS BM25 operations were incompatible with async MCP response transmission. I disabled FTS entirely and fell back to LIKE pattern matching. Problem solved!

```python
# WRONG: Bandaid solution that masks the real problem
def _validate_fts_requirements(self) -> bool:
    # FTS permanently disabled for MCP compatibility
    return False  # This just made hangs less frequent!
```

Except it wasn't solved. "It still hangs sometimes, just not as often." I'd masked the symptom, not fixed the cause.

This is where working with Claude Code became challenging. Claude Code declared victory: "Great! The FTS incompatibility is resolved. Your server should work reliably now." But real-world testing proved otherwise. Working with Claude Code on complex race conditions requires persistence - you need to push back when it declares premature victory.

### Dead End #2: The Threading Timeout Disaster

Next theory: timeout mechanism was causing the hangs. The MCP server used `run_with_timeout()` to prevent queries from running forever:

```python
# CATASTROPHIC CODE (Removed):
def run_with_timeout(func, timeout_seconds: int, *args, **kwargs):
    def target():
        result[0] = func(*args, **kwargs)  # Uses shared database connection!

    thread = threading.Thread(target=target)
    thread.start()
    thread.join(timeout_seconds)
    # Problem: Multiple threads accessing same DuckDB connection
```

Critical issues:

- **Thread Safety Violation:** DuckDB connections are not thread-safe
- **Database Connection Contention:** Multiple threads compete for same connection
- **Deadlock Scenarios:** Threads block indefinitely waiting for database locks
- **Zombie Thread Leaks:** Timed-out threads continue running

I spent a full day removing threading timeout mechanisms. Hangs persisted.

### Dead End #3: The SearchService Red Herring

Performance testing revealed 43% overhead from an unnecessary connection pooling layer:

```
# WRONG ARCHITECTURE:
MCP Server → SearchService → Connection Pool → SearchEngine → SearchDatabase
             (43% overhead)  (unnecessary)     

# BETTER:
MCP Server → SearchEngine → SearchDatabase
             (direct)       
```

Why SearchService was wrong:

- Single-threaded application: MCP server processes one request at a time
- Pool overhead: Locking and validation for no concurrency benefit
- Complex layering: 300+ lines of pooling code that wasn't needed

Claude Code confidently suggested: "The connection pooling is definitely the issue. Remove it and your hangs will disappear." Another premature declaration of victory. While removing it improved performance, the cold start hangs remained.

### Dead End #4: The Query Optimisation Rabbit Hole

Convinced the problem was SQL performance, I optimised DuckDB queries:

```sql
-- WRONG: Correlated subquery anti-pattern
SELECT r.*,
       COALESCE(
           (SELECT MAX(fts_main_rules.match_bm25(rule_id, ?))
            FROM rules r2 WHERE r2.rule_id = r.rule_id),  -- RUNS N TIMES!
           0.0
       ) as relevance_score
FROM rules r WHERE (conditions)

-- BETTER: CTE approach
WITH fts_scores AS (
    SELECT rule_id, MAX(score) as relevance_score
    FROM (SELECT rule_id, fts_main_rules.match_bm25(rule_id, ?) AS score FROM rules) fts
    GROUP BY rule_id
)
SELECT r.*, COALESCE(fs.relevance_score, 0.0) as score
FROM rules r JOIN fts_scores fs ON r.rule_id = fs.rule_id
```

This improved performance by 40-60%, but the hanging problem persisted. Good optimisation, wrong problem.

## The Actual Root Cause

After three days of dead ends, I noticed the pattern more carefully. This required pure stubbornness, Claude Code was no help in this regard at all!

The problem was a cold start initialisation race condition:

1. MCP server starts up and declares itself "ready"
2. Client immediately sends first query
3. Database is still warming up (connection pool, indexes, cache)
4. Query hits partially initialised database state
5. **Response gets lost in the async pipeline during warmup**
6. Client waits forever for a response that will never arrive
7. Second query works because database is now fully ready

This wasn't a performance issue - DuckDB was returning results in milliseconds. The response was simply getting lost in the uninitialised async pipeline.

## The Comprehensive Fix

The server needed proper initialisation sequencing:

```python
class DetectionNexusMcpServer:
    def __init__(self):
        # ... basic setup ...

        # CRITICAL: Don't declare ready until database is warmed up
        self._database_ready = False
        self._initialization_error = None

        # 4-phase warmup sequence
        self._warmup_database()

    def _warmup_database(self) -> bool:
        try:
            # Phase 1: Health check validation
            health_status = self._search_database.health_check()

            # Phase 2: Index warming (touch all major indexes)
            self._warm_indexes()

            # Phase 3: Search engine cache warming
            self._warm_search_cache()

            # Phase 4: Performance validation
            self._validate_performance()

            self._database_ready = True
            return True
        except Exception as e:
            self._initialization_error = str(e)
            return False

    def search_rules(self, query: str, limit: int = 10):
        # Check readiness before processing ANY requests
        if not self._database_ready:
            if self._initialization_error:
                raise RuntimeError(f"Database initialization failed: {self._initialization_error}")
            raise RuntimeError("Database still initializing, please wait")

        # Now proceed with actual search
        return self._search_engine.search(query, limit)
```

## Database Health Checks That Actually Work

```python
def health_check(self) -> Dict[str, Any]:
    """Comprehensive health check covering all failure modes"""

    # Test actual query performance, not just connectivity
    start_time = time.time()

    try:
        # Verify indexes are loaded and responsive
        result = self.connection.execute("SELECT COUNT(*) FROM rules").fetchone()
        index_time = (time.time() - start_time) * 1000

        # Check WAL file status (can indicate lock issues)
        wal_check = self._check_wal_status()

        # Validate schema accessibility
        schema_check = self._validate_schema_tables()

        # Performance threshold check
        if index_time > 1000:  # More than 1 second is concerning
            return {"status": "degraded", "reason": f"Slow index response: {index_time:.1f}ms"}

        return {
            "status": "healthy",
            "index_response_time": f"{index_time:.1f}ms",
            "wal_status": wal_check,
            "schema_tables": schema_check
        }

    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}
```

## Index Warmup Strategy

```python
def _warm_indexes(self):
    """Pre-load database indexes into memory"""
    warmup_queries = [
        "SELECT COUNT(*) FROM rules",
        "SELECT COUNT(*) FROM rules WHERE provider = 'splunk'",
        "SELECT COUNT(*) FROM rules WHERE enabled = true",
        "SELECT COUNT(*) FROM mitre_techniques"
        # Add queries that touch your major indexes
    ]

    for query in warmup_queries:
        self.connection.execute(query).fetchone()
```

## Results

**Before (Broken):**

- First query after restart: Hang indefinitely ❌
- User experience: Frustrating retry-until-it-works ❌

**After (Fixed):**

- Server startup: 60-77ms with comprehensive warmup ✅
- First query after restart: 2ms response time ✅
- User experience: Reliable every time ✅

## Key Lessons

1. **Claude Code declares victory too quickly.** When Claude Code says "Problem solved!", test thoroughly. Complex race conditions often require pushing back and insisting on deeper investigation.

2. **Connection != ready.** Your database connection being established doesn't mean indexes are loaded and caches are warm.

3. **"Works on retry" = race condition.** This pattern is a dead giveaway you're dealing with an initialisation race.

4. **Performance optimisations don't fix architectural problems.** CTE queries and removing connection pooling improved performance but didn't address the core race condition.

5. **Fix the cause, not the symptom.** Disabling features to avoid problems creates technical debt.

6. **Test cold starts rigorously.** Always restart your server between tests during development.

## Architecture Anti-Patterns That Fooled Me

- **Threading with shared database connections:** Never use threading timeout mechanisms with database connections
- **Connection pooling for single-threaded applications:** Adds overhead without benefit
- **Disabling features to avoid problems:** Usually masks the real issue
- **Optimising the wrong layer:** SQL query performance won't fix initialisation race conditions

## Quick Implementation Checklist

If you're building async servers with database backends:

- [ ] Implement proper readiness checks - don't declare ready until your database actually is
- [ ] Add comprehensive health validation - test query performance, not just connectivity  
- [ ] Include index warmup - pre-load critical indexes during startup
- [ ] Test cold start scenarios - restart your server between tests
- [ ] Monitor initialisation timing - log warmup phases to identify bottlenecks
- [ ] Push back on premature victory declarations - test thoroughly even when tools say "fixed"

**Security note:** Be careful with health check endpoints - they can leak database schema information if not properly secured.

The fix eliminated the cold start race condition entirely. No more mysterious hangs, no more debugging sessions that stretch into the night. The real lesson? Understanding that "server ready" and "database ready" are two different states that need proper synchronisation - and having the persistence to find the actual root cause even when your debugging assistant declares premature victory.