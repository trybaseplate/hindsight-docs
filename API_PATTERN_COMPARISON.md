# API Pattern Comparison: When to Use What

## Quick Decision Tree

```
Does the user need...
├─ A complete answer with context?
│  └─ → Use Pattern 1: AI Agent Responses ✅
│     Examples: Glean search, Slack bot, general Q&A
│
└─ Specific data to process themselves?
   └─ → Use Pattern 2: Individual Tools ✅
      Examples: Dashboard, analytics, custom AI
```

---

## Pattern 1: AI Agent Responses (Orchestrated)

### Endpoint
```
POST /api/v1/chat/completions
```

### Request
```json
{
  "messages": [
    {"role": "user", "content": "Why are we losing to Competitor X?"}
  ],
  "org_id": "org_123"
}
```

### Response
```json
{
  "message": {
    "content": "Based on 12 recent deals, here are the main reasons...\n\n1. **Pricing** - We lost 5 deals where pricing was 20% higher [Deal A](link), [Deal B](link)...",
    "citations": [
      {
        "document_id": "doc_123",
        "excerpt": "Customer chose Competitor X due to lower pricing...",
        "url": "https://..."
      }
    ]
  },
  "tool_calls": ["select_deal", "search_deal_documents"]
}
```

### When to Use
- ✅ User wants a complete answer
- ✅ Context and synthesis needed
- ✅ Citations should be inline
- ✅ Glean, Slack, or chat interfaces
- ✅ User is non-technical

### Pros
- Complete, contextual answers
- Automatic tool orchestration
- Built-in citations
- Less client logic needed

### Cons
- Higher cost (~$0.10-0.50/req)
- Less control
- May include extra info
- Slower response time

---

## Pattern 2: Individual Tools (Direct Access)

### Endpoints
```
POST /api/v1/tools/select_deal
POST /api/v1/tools/search_deal_documents
POST /api/v1/tools/search_competitors
... (7 total tools)
```

### Request Flow
```json
// Step 1: Find relevant deals
POST /tools/select_deal
{
  "org_id": "org_123",
  "filters": {
    "status": ["Closed Lost"],
    "competitors": ["comp_x_id"]
  }
}

// Step 2: Search each deal
POST /tools/search_deal_documents
{
  "deal_ids": ["deal_1", "deal_2"],
  "query": "reason for loss"
}

// Step 3: Client synthesizes results
```

### Response
```json
{
  "results": [
    {
      "deal_id": "deal_1",
      "content": "Customer chose Competitor X due to lower pricing...",
      "similarity_score": 0.92
    }
  ]
}
```

### When to Use
- ✅ Building custom dashboard
- ✅ Need structured data
- ✅ Want granular control
- ✅ MCP tool integration
- ✅ Cost optimization important
- ✅ Custom AI/processing

### Pros
- Lower cost (~$0.01-0.05/req)
- Full control over flow
- Structured data
- Composable
- Efficient for bulk queries

### Cons
- Multiple calls needed
- Client handles synthesis
- More implementation work
- Need to understand tools

---

## Real-World Examples

### Example 1: Glean Integration

**Question:** "What are common security objections?"

**Best Approach:** Pattern 1 (AI Agent) ✅

**Why:**
- Users expect a complete answer in Glean
- Need natural language response
- Want inline citations
- Don't want raw data

**Implementation:**
```typescript
// Glean calls Hindsight API
POST /api/v1/chat/completions
{
  "messages": [{"role": "user", "content": "What are common security objections?"}]
}

// Hindsight returns formatted answer
// Glean displays in search results
```

---

### Example 2: Sales Dashboard

**Need:** Show "Top 10 deals lost to Competitor X this quarter"

**Best Approach:** Pattern 2 (Individual Tools) ✅

**Why:**
- Need structured data for charts
- Want to control formatting
- Need to combine with other data sources
- Will make many similar queries

**Implementation:**
```typescript
// Dashboard makes direct tool calls
const deals = await fetch('/tools/select_deal', {
  body: JSON.stringify({
    filters: {
      status: ['Closed Lost'],
      competitors: ['comp_x_id'],
      date_range: { start: '2024-10-01', end: '2024-12-31' }
    },
    limit: 10
  })
})

// Dashboard renders data in custom format
```

---

### Example 3: Claude Desktop via MCP

**Need:** User asks Claude "Help me prepare for a call with a prospect considering Competitor X"

**Best Approach:** Pattern 2 (Individual Tools) ✅

**Why:**
- MCP exposes individual tools
- Claude orchestrates tool calls
- Claude does the synthesis
- User gets Claude's personality/style

**Implementation:**
```json
// User's MCP config
{
  "mcpServers": {
    "hindsight": {
      "command": "npx @hindsight/mcp-server",
      "env": {
        "HINDSIGHT_API_KEY": "key_123"
      }
    }
  }
}

// Claude sees Hindsight tools
// Claude calls search_competitors("Competitor X strengths")
// Claude calls search_across_deals("Competitor X objections")
// Claude synthesizes final answer
```

---

### Example 4: Custom Slack Bot

**Need:** Slack bot that answers sales questions

**Best Approach:** Pattern 1 (AI Agent) ✅

**Why:**
- Want natural language responses
- Need quick, formatted answers
- Don't want to implement orchestration
- Slack has message length limits (Pattern 1 handles this)

**Implementation:**
```typescript
app.message(/.*/, async ({ message, say }) => {
  const response = await fetch('/api/v1/chat/completions', {
    method: 'POST',
    body: JSON.stringify({
      messages: [{ role: 'user', content: message.text }]
    })
  })
  
  await say(response.message.content)
})
```

---

### Example 5: Win/Loss Analytics Pipeline

**Need:** Nightly job to analyze lost deals and send email summary

**Best Approach:** Pattern 2 (Individual Tools) ✅

**Why:**
- Batch processing
- Need structured data for analysis
- Custom aggregation logic
- Cost-sensitive (many queries)

**Implementation:**
```typescript
// Nightly cron job
async function analyzeWeeklyLosses() {
  // Get all deals lost this week
  const lostDeals = await fetch('/tools/select_deal', {
    body: JSON.stringify({
      filters: {
        status: ['Closed Lost'],
        date_range: { start: lastWeek, end: today }
      }
    })
  })
  
  // Analyze each deal
  const analyses = await Promise.all(
    lostDeals.map(deal => 
      fetch('/tools/search_deal_documents', {
        body: JSON.stringify({
          deal_ids: [deal.id],
          query: 'reason for loss'
        })
      })
    )
  )
  
  // Custom analysis and email
  const summary = customAggregation(analyses)
  await sendEmail(summary)
}
```

---

## Tool Selection Matrix

| Use Case | Pattern | Why |
|----------|---------|-----|
| Search in Glean | 1 (AI) | Natural language results needed |
| Slack Q&A bot | 1 (AI) | Quick answers with context |
| Sales dashboard | 2 (Tools) | Structured data for visualization |
| Claude Desktop | 2 (Tools) | MCP expects individual tools |
| Analytics pipeline | 2 (Tools) | Batch processing, cost optimization |
| Custom reporting | 2 (Tools) | Need raw data for custom logic |
| Mobile app search | 1 (AI) | Users want complete answers |
| API for partners | 2 (Tools) | Partners integrate in their systems |
| Internal wiki search | 1 (AI) | Employees want answers not data |
| Data science analysis | 2 (Tools) | Need to feed into ML models |

---

## Hybrid Approach

Some integrations may use BOTH patterns:

### Example: Advanced Sales Enablement App

```typescript
// Use Pattern 2 for structured queries
const deals = await selectDeal({ filters: {...} })
const competitors = await searchCompetitors({ query: '...' })

// Feed results into Pattern 1 for synthesis
const response = await chatCompletions({
  messages: [
    {
      role: "system",
      content: `Here is deal data: ${JSON.stringify(deals)}`
    },
    {
      role: "user", 
      content: "Summarize these deals and give me top 3 action items"
    }
  ]
})

// Display AI-generated summary with access to raw data
```

---

## Cost Comparison

### Scenario: "What are common pricing objections?"

**Pattern 1 (AI Agent):**
```
1 request → AI orchestrates → 3 tool calls internally → 1 response
Cost: ~$0.30
Time: ~3-5 seconds
```

**Pattern 2 (Individual Tools):**
```
Request 1: search_across_deals("pricing objections")
Request 2: search_deal_documents(deal_ids, "pricing")
Request 3: Client does synthesis
Cost: ~$0.03
Time: ~2-3 seconds + client processing
```

**Winner:** Pattern 2 is 10x cheaper, but requires more client logic

---

## Recommendations by Integration Type

### Glean
→ **Pattern 1 (AI Agent)** 
- Users expect complete answers
- Citations inline
- No custom logic needed

### MCP / Claude Desktop
→ **Pattern 2 (Individual Tools)**
- MCP design expects individual tools
- Claude does orchestration
- User gets Claude's synthesis

### Custom Dashboard
→ **Pattern 2 (Individual Tools)**
- Need structured data
- Custom UI/formatting
- Cost optimization

### Slack Bot
→ **Pattern 1 (AI Agent)**
- Quick answers needed
- Natural language responses
- No custom logic

### Partner APIs
→ **Pattern 2 (Individual Tools)**
- Partners integrate into their systems
- They control orchestration
- More flexible

---

## Decision Framework

Ask yourself:

1. **Who synthesizes the answer?**
   - Hindsight AI → Pattern 1
   - Client/External AI → Pattern 2

2. **What's the output format?**
   - Natural language → Pattern 1
   - Structured data → Pattern 2

3. **How many queries?**
   - One-off questions → Pattern 1
   - Bulk/batch processing → Pattern 2

4. **What's the cost sensitivity?**
   - Cost not critical → Pattern 1
   - Cost optimization important → Pattern 2

5. **Who's the end user?**
   - Non-technical → Pattern 1
   - Technical/Developer → Either works
