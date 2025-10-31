# Hindsight API Implementation Plan

## Overview
This document outlines the implementation approach for exposing Hindsight data via API for integration with Glean, OpenAI/MCP, and other tools.

## Two Integration Patterns

### Pattern 1: AI Agent Responses (Orchestrated)
**Use Case:** External AI needs comprehensive answers with context
**Flow:** Question → Hindsight AI → Multiple tool calls → Synthesized answer with citations

**Implementation:**
- Leverage existing `/api/completions/paige-new/route.ts`
- Expose as a public API endpoint with authentication
- Returns full text responses with citations
- AI handles tool orchestration automatically

**Pros:**
- Reuses existing logic
- Provides high-quality, contextual answers
- Less client-side complexity
- Better for Glean integration (users want answers, not data)

**Cons:**
- Less control for client
- Higher compute cost per request
- May include irrelevant information

**Best for:**
- Glean search results
- Slack bot responses
- General Q&A interfaces

---

### Pattern 2: Individual Tools (Direct Access)
**Use Case:** External system needs specific data for custom processing
**Flow:** Direct call → Specific tool → Raw results

**Implementation:**
- Create wrapper endpoints for each tool in `buildPaigeChatPrompt()`
- Each tool exposed as standalone REST endpoint
- Returns structured data without AI synthesis

**Pros:**
- Granular control
- Lower cost per request
- Client can compose multiple calls
- Better for custom dashboards/analytics

**Cons:**
- Requires more client-side logic
- Multiple calls needed for complex queries
- No automatic context/synthesis

**Best for:**
- Custom dashboards
- Programmatic integrations
- Data exports
- MCP tools

---

## Core Tools to Expose

Based on `lib/prompts_zod.ts`, here are the 14 tools available:

### Search & Discovery Tools
1. **`search_knowledge_base`** - Search all competitive intelligence docs
2. **`search_across_deals`** - Semantic search across all deal documents  
3. **`search_deal_documents`** - Targeted search within specific deals
4. **`search_deals`** - Semantic search for similar deals
5. **`select_deal`** - Filtered deal search (stage, amount, competitor, etc.)
6. **`search_competitors`** - Search competitor intelligence database
7. **`get_documents`** - Find documents by keyword/name

### Utility Tools
8. **`summarize_conversation`** - Compress conversation history (internal use)
9. **`plan_research`** - Multi-step research planning (internal use)
10. **`validate_sources`** - Cross-reference findings (internal use)

### Workflow Tools (May not need in API)
11. **`draft_email`** - Start email drafting workflow
12. **`draft_rfp`** - Start RFP response workflow
13. **`draft_sq`** - Start security questionnaire workflow
14. **`revise`** - Revise selected text

---

## Recommended API Structure

```
/api/v1/
├── chat/
│   └── completions          # Pattern 1: AI Agent
│
└── tools/
    ├── search_knowledge_base
    ├── search_across_deals
    ├── search_deal_documents
    ├── search_deals
    ├── select_deal
    ├── search_competitors
    └── get_documents
```

---

## Implementation Recommendations

### Phase 1: Core Search Tools (MVP)
Start with the most valuable tools that cover 80% of use cases:
1. **`search_across_deals`** - Most versatile semantic search
2. **`select_deal`** - Essential for filtering deals
3. **`search_competitors`** - Core competitive intel
4. **Chat completions** - AI agent endpoint

This gives you:
- Glean integration (chat completions)
- MCP support (individual tools)
- Custom integrations (individual tools)

### Phase 2: Advanced Tools
Add remaining tools based on demand:
- `search_deal_documents` (targeted search)
- `search_deals` (semantic deal matching)
- `get_documents` (document retrieval)

### Phase 3: Workflow Tools (Optional)
Consider if drafting workflows make sense via API:
- These are more interactive/UI-dependent
- May be better as webhooks or async jobs

---

## Authentication & Security

### API Key Management
- Create `api_keys` table in Supabase
- Store hashed keys with org_id mapping
- Middleware to validate on each request

```typescript
// Proposed schema
api_keys {
  id: uuid
  org_id: uuid -> organizations
  key_hash: string
  name: string
  created_at: timestamp
  last_used_at: timestamp
  permissions: json // which tools they can access
  rate_limit: integer
}
```

### Rate Limiting
Use existing or add:
- Upstash Redis for rate limiting
- Per-key limits based on plan tier
- Different limits for Pattern 1 vs Pattern 2

---

## MCP Integration

### MCP Server Package
Create `@hindsight/mcp-server` npm package:
- Wraps individual tool endpoints
- Handles authentication
- Provides type-safe tool definitions

```typescript
// Example MCP tool definition
{
  name: "search_across_deals",
  description: "Search across all deal documents",
  inputSchema: {
    type: "object",
    properties: {
      query: { type: "string" },
      limit: { type: "number" }
    },
    required: ["query"]
  }
}
```

---

## Glean Integration Approach

### Option A: Direct API (Recommended)
- Expose chat completions endpoint
- Glean calls Hindsight API directly
- Returns formatted results with citations
- Simplest integration

### Option B: Glean Custom Connector
- Build Glean-specific connector
- Use their connector SDK
- More native Glean experience
- More maintenance

**Recommendation:** Start with Option A, evaluate Option B if needed.

---

## OpenAI Integration

### Function Calling
External AI (like ChatGPT) can call Hindsight tools:
1. Expose tools as OpenAI function schemas
2. AI decides when to call Hindsight
3. Hindsight returns structured data
4. AI synthesizes response

### Direct MCP
For Claude Desktop and other MCP clients:
1. User installs `@hindsight/mcp-server`
2. Claude sees Hindsight tools automatically
3. Claude can call tools during conversation

---

## Data Privacy & Permissions

### Consider:
1. **Org-level isolation** - API key tied to org_id
2. **User-level permissions** - Some orgs may want per-user keys
3. **Document permissions** - Respect existing Keto permissions
4. **Deal access** - Filter deals based on permissions
5. **Audit logging** - Track all API usage

### Proposed Permission System:
```typescript
{
  "read:deals": true,
  "read:documents": true,
  "read:competitors": true,
  "read:analytics": false,  // future
  "write:*": false  // no write access via API initially
}
```

---

## Response Formatting

### Standardized Response
```typescript
{
  "data": {...},           // Tool-specific results
  "metadata": {
    "request_id": "uuid",
    "timestamp": "ISO8601",
    "tool": "search_across_deals",
    "execution_time_ms": 234
  },
  "pagination": {          // If applicable
    "page": 1,
    "page_size": 20,
    "total_count": 45
  }
}
```

### Citations Format
```typescript
{
  "document_id": "uuid",
  "document_name": "string",
  "deal_id": "uuid | null",
  "deal_name": "string | null",
  "url": "string",
  "excerpt": "string",
  "similarity_score": 0.0-1.0,
  "metadata": {}
}
```

---

## Cost Considerations

### Per Request Costs
- **Pattern 1 (AI Agent):** ~$0.10-0.50 per request (multiple embeddings + LLM)
- **Pattern 2 (Direct Tool):** ~$0.01-0.05 per request (single embedding)

### Pricing Strategy
- Free tier: Limited requests, Pattern 2 only
- Pro tier: Higher limits, both patterns
- Enterprise: Custom limits, dedicated resources

---

## Testing Strategy

### API Testing
1. Unit tests for each tool endpoint
2. Integration tests for auth flow
3. Load tests for rate limiting
4. E2E tests for MCP server

### Example Integrations
Build example projects:
1. Simple Node.js client
2. Python client
3. MCP server example
4. Glean connector example

---

## Open Questions

1. **Do we expose all 14 tools or start with subset?**
   - Recommendation: Start with 7 core search tools

2. **Do we expose drafting workflows via API?**
   - Recommendation: No, these are too UI-dependent

3. **Should API support conversation history?**
   - Pattern 1: Yes, via messages array
   - Pattern 2: No, stateless tools

4. **How granular should permissions be?**
   - Recommendation: Start org-level, add user-level later

5. **Do we need webhook events?**
   - Recommendation: Not MVP, but plan for it

6. **Should we version the API from day 1?**
   - Recommendation: Yes, use /v1/ in paths

---

## Success Metrics

### Track:
- API requests per day/week
- Response times (p50, p95, p99)
- Error rates by endpoint
- Most used tools
- Integration adoption (Glean, MCP, custom)
- Cost per request
- User satisfaction (via feedback)

---

## Timeline Estimate

### Week 1-2: Infrastructure
- API key management
- Authentication middleware
- Rate limiting
- Logging/monitoring

### Week 3-4: Core Tools (Phase 1)
- Wrap existing tool implementations
- Create API endpoints
- Add error handling
- Write tests

### Week 5-6: MCP Server
- Build npm package
- Tool definitions
- Auth handling
- Documentation

### Week 7-8: Glean Integration
- Test with Glean
- Optimize response format
- Add caching if needed
- Performance tuning

### Week 9-10: Documentation & Launch
- API documentation
- Example code
- SDK libraries (optional)
- Beta launch

---

## Next Steps

1. **Validate approach** with team
2. **Prioritize tools** for Phase 1
3. **Design API key system** (schema, UI)
4. **Set up monitoring** (Datadog, Sentry, etc.)
5. **Create API endpoints** for core tools
6. **Build MCP server package**
7. **Test with Glean** (get account, test integration)
8. **Document everything**
9. **Launch beta**
10. **Gather feedback and iterate**
