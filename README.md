# Azure Search OpenAI Demo + Linkup Web Search Integration

This is a fork of [azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo) with **Linkup API** integrated as a smart web search fallback.

## What's Different?

This version adds **intelligent web search fallback** using the [Linkup API](https://linkup.so). When your internal documents don't have an answer, or when users ask for real-time information, the app automatically queries the web.

### How It Works

```
User Query
    ↓
┌─────────────────────────────────────┐
│ 1. Keyword Detection                │
│    Check for web search keywords    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 2. Azure AI Search                  │
│    Search internal documents        │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 3. Linkup Decision                  │
│    - If keywords found → Call Linkup│
│    - If poor results → Call Linkup  │
│    - Otherwise → Skip Linkup        │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 4. Merge Sources                    │
│    Combine docs + Linkup results    │
└─────────────────────────────────────┘
    ↓
┌─────────────────────────────────────┐
│ 5. Stream LLM Response              │
│    Generate answer with citations   │
└─────────────────────────────────────┘
```

### Dual-Trigger Logic

Linkup is called when **either** condition is met:

| Trigger | Condition | Example Queries |
|---------|-----------|-----------------|
| **Keyword** | Query contains web search keywords | "What's the current price?", "latest news" |
| **Poor Results** | Document scores < threshold | Any query with no relevant documents |

**Keywords that trigger web search:**
- Explicit: `linkup`, `search the web`, `look up`, `google`, `web search`
- Real-time: `latest`, `current`, `today`, `now`, `stock price`, `news`, `weather`, `live`

---

## Quick Start

### Prerequisites

- Azure account with permissions to create resources
- [Azure Developer CLI](https://aka.ms/azure-dev/install)
- [Python 3.10+](https://www.python.org/downloads/)
- [Node.js 20+](https://nodejs.org/download/)
- [Linkup API key](https://linkup.so) (free tier available)

### Deploy to Azure

```bash
# Clone this repo
git clone https://github.com/shauryajain21/linkup-azure-search-openai.git
cd linkup-azure-search-openai

# Login to Azure
azd auth login

# Create environment
azd env new myenv

# Set your Linkup API key
azd env set LINKUP_API_KEY "your-linkup-api-key"

# Deploy (this provisions Azure resources and deploys the app)
azd up
```

### Run Locally (after Azure deployment)

```bash
# Set environment variables
export LINKUP_API_KEY="your-linkup-api-key"

# Start the app
./app/start.sh
```

Navigate to http://localhost:50505

### Test Linkup Integration

```bash
# Test keyword trigger (should call Linkup)
curl -X POST http://localhost:50505/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "What is the current price of Bitcoin?"}]}'

# Check the response for:
# - "trigger_reason": "keyword_detected"
# - Linkup results with current Bitcoin price
```

---

## Configuration

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `LINKUP_API_KEY` | None | Your Linkup API key (required for web search) |
| `LINKUP_SCORE_THRESHOLD` | 0.5 | Minimum document score to skip Linkup fallback |

---

## Manual Implementation Guide

Want to add Linkup to your own azure-search-openai-demo fork? Follow these steps:

### Files to Modify

| File | Purpose |
|------|---------|
| `app/backend/requirements.in` | Add linkup-sdk dependency |
| `app/backend/approaches/approach.py` | Add LinkupResult dataclass and update get_sources_content |
| `app/backend/approaches/chatreadretrieveread.py` | Add keyword detection and Linkup triggering logic |
| `app/backend/app.py` | Wire up Linkup API key from environment |
| `app/backend/approaches/prompts/chat_answer_question.prompty` | Update prompt to use web search results |

### Step 1: Add Linkup SDK Dependency

**File:** `app/backend/requirements.in`

Add this line at the end:

```
linkup-sdk
```

Then regenerate requirements.txt:
```bash
cd app/backend
pip-compile requirements.in
pip install -r requirements.txt
```

### Step 2: Add LinkupResult Dataclass

**File:** `app/backend/approaches/approach.py`

Find the `SharePointResult` dataclass and add `LinkupResult` after it:

```python
@dataclass
class LinkupResult:
    """Result from Linkup API search for web-based answers."""
    id: str
    answer: str
    sources: list[dict[str, Any]]

    def serialize_for_results(self) -> dict[str, Any]:
        return {
            "type": "linkup",
            "id": self.id,
            "answer": self.answer,
            "sources": self.sources,
        }
```

### Step 3: Update get_sources_content Method

**File:** `app/backend/approaches/approach.py`

Update the method signature to include `linkup_results`:

```python
async def get_sources_content(
    self,
    results: list[Document],
    use_semantic_captions: bool,
    include_text_sources: bool,
    download_image_sources: bool,
    user_oid: Optional[str] = None,
    web_results: Optional[list[WebResult]] = None,
    sharepoint_results: Optional[list[SharePointResult]] = None,
    linkup_results: Optional[LinkupResult] = None,  # ADD THIS
) -> DataPoints:
```

Add this code inside the method, before `return DataPoints(...)`:

```python
# Handle Linkup API results
if linkup_results:
    # Add Linkup answer as a text source
    if include_text_sources and linkup_results.answer:
        text_sources.append(f"[Linkup Web Search]: {clean_source(linkup_results.answer)}")

    # Add each Linkup source as a citation and metadata
    for source in linkup_results.sources:
        source_url = source.get("url", "")
        citation = self.get_citation(source_url)
        if citation and citation not in citations:
            citations.append(citation)
        external_results_metadata.append(
            {
                "id": linkup_results.id,
                "title": source.get("title", ""),
                "url": source_url,
                "snippet": clean_source(source.get("snippet", "")),
                "type": "linkup",
            }
        )
```

### Step 4: Add Linkup Import and Client Setup

**File:** `app/backend/approaches/chatreadretrieveread.py`

Add these imports at the top:

```python
import uuid

# Import Linkup SDK (optional - only used if API key is configured)
try:
    from linkup import LinkupClient
except ImportError:
    LinkupClient = None  # type: ignore
```

Update the import from `approaches.approach` to include `LinkupResult`:

```python
from approaches.approach import (
    Approach,
    ExtraInfo,
    LinkupResult,  # ADD THIS
    ThoughtStep,
)
```

### Step 5: Add Keywords and Detection Method

**File:** `app/backend/approaches/chatreadretrieveread.py`

Add after `NO_RESPONSE = ...`:

```python
# Keywords that indicate user wants web search (triggers Linkup API)
WEB_SEARCH_KEYWORDS = [
    # Explicit search requests
    "linkup", "search the web", "look up", "find out", "check online",
    "search for", "google", "web search",
    # Real-time data indicators
    "latest", "current", "recent", "today", "now", "right now",
    "stock price", "news", "weather", "breaking", "update",
    "what is happening", "what's new", "whats new",
    "real-time", "realtime", "live",
]

def _detect_web_search_intent(self, query: str) -> bool:
    """Detect if user query indicates a web search intent based on keywords."""
    query_lower = query.lower()
    return any(keyword in query_lower for keyword in self.WEB_SEARCH_KEYWORDS)
```

### Step 6: Update __init__ Method

**File:** `app/backend/approaches/chatreadretrieveread.py`

Add these parameters to `__init__`:

```python
def __init__(
    self,
    *,
    # ... existing parameters ...
    linkup_api_key: Optional[str] = None,
    linkup_score_threshold: float = 0.5,
):
```

Add at the end of `__init__` body:

```python
# Linkup API integration for web-based fallback
self.linkup_client = None
self.linkup_score_threshold = linkup_score_threshold
if linkup_api_key and LinkupClient is not None:
    self.linkup_client = LinkupClient(api_key=linkup_api_key)
```

### Step 7: Add Linkup Query Method

**File:** `app/backend/approaches/chatreadretrieveread.py`

Add this method:

```python
async def _query_linkup(self, query: str) -> Optional[LinkupResult]:
    """Query Linkup API for web-based answers when document search is insufficient."""
    if not self.linkup_client:
        return None

    try:
        response = self.linkup_client.search(
            query=query,
            depth="deep",
            output_type="sourcedAnswer",
        )
        return LinkupResult(
            id=str(uuid.uuid4()),
            answer=response.answer,
            sources=[
                {"url": src.url, "snippet": src.snippet, "title": src.name}
                for src in response.sources
            ],
        )
    except Exception as e:
        logging.warning(f"Linkup API error: {e}")
        return None
```

### Step 8: Update run_search_approach Method

**File:** `app/backend/approaches/chatreadretrieveread.py`

After `results = await self.search(...)`, add:

```python
# Query Linkup API based on dual-trigger logic
linkup_results: Optional[LinkupResult] = None
use_linkup_fallback = overrides.get("use_linkup_fallback", True)
linkup_trigger_reason: Optional[str] = None

# Check for explicit web search intent based on keywords
user_wants_web_search = self._detect_web_search_intent(original_user_query)

if use_linkup_fallback and self.linkup_client:
    # Trigger 1: User explicitly requested web search via keywords
    if user_wants_web_search:
        linkup_trigger_reason = "keyword_detected"
        linkup_results = await self._query_linkup(query_text)
    else:
        # Trigger 2: Document search results are insufficient
        has_good_results = any(
            (doc.score or 0) >= self.linkup_score_threshold for doc in results
        )
        if not has_good_results:
            linkup_trigger_reason = "poor_search_results"
            linkup_results = await self._query_linkup(query_text)
```

Update the `get_sources_content` call:

```python
data_points = await self.get_sources_content(
    results,
    use_semantic_captions,
    include_text_sources=send_text_sources,
    download_image_sources=send_image_sources,
    user_oid=auth_claims.get("oid"),
    linkup_results=linkup_results,  # ADD THIS
)
```

Add a thought step for debugging:

```python
if use_linkup_fallback and self.linkup_client:
    thoughts.append(
        ThoughtStep(
            "Linkup web search",
            linkup_results.serialize_for_results() if linkup_results else "Not triggered",
            {
                "triggered": linkup_results is not None,
                "trigger_reason": linkup_trigger_reason,
                "keyword_detected": user_wants_web_search,
                "score_threshold": self.linkup_score_threshold,
            },
        )
    )
```

### Step 9: Wire Up in app.py

**File:** `app/backend/app.py`

Add environment variable reads:

```python
LINKUP_API_KEY = os.getenv("LINKUP_API_KEY")
LINKUP_SCORE_THRESHOLD = float(os.getenv("LINKUP_SCORE_THRESHOLD", "0.5"))
```

Update `ChatReadRetrieveReadApproach` instantiation:

```python
current_app.config[CONFIG_CHAT_APPROACH] = ChatReadRetrieveReadApproach(
    # ... existing parameters ...
    linkup_api_key=LINKUP_API_KEY,
    linkup_score_threshold=LINKUP_SCORE_THRESHOLD,
)
```

### Step 10: Update the LLM Prompt

**File:** `app/backend/approaches/prompts/chat_answer_question.prompty`

**This is critical!** Update the system prompt so the LLM knows how to use web search results:

Find:
```
Assistant helps the company employees with their questions about internal documents.
```

Replace with:
```
Assistant helps the company employees with their questions. Be brief in your answers.
Answer ONLY with the facts listed in the list of sources below. If there isn't enough information below, say you don't know. Do not generate answers that don't use the sources below.
Sources may include internal documents and web search results (marked as [Linkup Web Search]). Prefer web search results for real-time information like current prices, news, or recent events.
```

---

## Troubleshooting

### Linkup not triggering
- Check that `LINKUP_API_KEY` is set correctly
- Restart the server after code changes
- Check the "Linkup web search" thought step in the response

### LLM not using Linkup results
- Ensure you updated `chat_answer_question.prompty` (Step 10)
- The prompt must tell the LLM to prefer web search for real-time info

### Import errors
- Run `pip install linkup-sdk` in your virtual environment
- Regenerate requirements.txt with `pip-compile requirements.in`

---

## Original README

For the original azure-search-openai-demo documentation, see [Azure-Samples/azure-search-openai-demo](https://github.com/Azure-Samples/azure-search-openai-demo).

---

## License

This project is licensed under the MIT License - see the original repo for details.
