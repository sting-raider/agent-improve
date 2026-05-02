# ORACLE Multi-Model Orchestration Design

## Overview

This document details how ORACLE orchestrates multiple AI models across different providers, API keys, and deployment scenarios. The system is designed to maximize reliability, minimize costs, and provide optimal performance for different task types.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ORACLE Multi-Model Router                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Task Classifier                           │  │
│  │  - Analyzes task requirements                              │  │
│  │  - Determines optimal model                                 │  │
│  │  - Checks rate limits and costs                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    LiteLLM Gateway                          │  │
│  │  - Load balances across keys                               │  │
│  │  - Automatic failover                                      │  │
│  │  - Rate limit handling                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                    │
│         │                 │                 │                    │
│         ▼                 ▼                 ▼                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Anthropic   │  │    OpenAI    │  │    Google    │     │
│  │   (Claude)    │  │   (GPT)      │  │   (Gemini)   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Result Aggregator                        │  │
│  │  - Merges results from multiple models                     │  │
│  │  - Applies consensus logic                                  │  │
│  │  - Returns final answer                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Task Classification

### Task Types

```python
class TaskType(Enum):
    """Types of tasks with different model requirements."""
    
    # High-reasoning tasks (require best models)
    COMPLEX_REASONING = "complex_reasoning"
    HYPOTHESIS_GENERATION = "hypothesis_generation"
    CONTRADICTION_DETECTION = "contradiction_detection"
    SOURCE_VERIFICATION = "source_verification"
    
    # Medium-reasoning tasks (good models sufficient)
    SUMMARIZATION = "summarization"
    ENTITY_EXTRACTION = "entity_extraction"
    RELATION_EXTRACTION = "relation_extraction"
    PATTERN_DETECTION = "pattern_detection"
    
    # Fast tasks (use fastest models)
    CLASSIFICATION = "classification"
    SENTIMENT_ANALYSIS = "sentiment_analysis"
    KEYWORD_EXTRACTION = "keyword_extraction"
    
    # Specialized tasks
    VISION_ANALYSIS = "vision_analysis"
    CODE_ANALYSIS = "code_analysis"
    NUMERICAL_ANALYSIS = "numerical_analysis"
```

### Task Requirements

```python
TASK_REQUIREMENTS = {
    TaskType.COMPLEX_REASONING: {
        "min_quality": 0.9,  # 90% quality score
        "max_latency": 30000,  # 30 seconds
        "max_cost_per_1k_tokens": 0.03,
        "context_window": 200000,
        "preferred_models": ["claude-3-5-sonnet", "gpt-4o", "gemini-2.0-flash-thinking"]
    },
    
    TaskType.HYPOTHESIS_GENERATION: {
        "min_quality": 0.85,
        "max_latency": 60000,  # 1 minute
        "max_cost_per_1k_tokens": 0.05,
        "context_window": 200000,
        "preferred_models": ["claude-3-5-sonnet", "gpt-4o"]
    },
    
    TaskType.ENTITY_EXTRACTION: {
        "min_quality": 0.7,
        "max_latency": 10000,  # 10 seconds
        "max_cost_per_1k_tokens": 0.01,
        "context_window": 128000,
        "preferred_models": ["gliner2-base-v1", "gpt-4o-mini", "claude-3-haiku"]
    },
    
    TaskType.SUMMARIZATION: {
        "min_quality": 0.75,
        "max_latency": 15000,
        "max_cost_per_1k_tokens": 0.015,
        "context_window": 128000,
        "preferred_models": ["gpt-4o-mini", "claude-3-haiku", "gemini-1.5-flash"]
    },
    
    TaskType.VISION_ANALYSIS: {
        "min_quality": 0.8,
        "max_latency": 30000,
        "max_cost_per_1k_tokens": 0.02,
        "context_window": 128000,
        "preferred_models": ["gemini-2.0-flash", "claude-3-5-sonnet", "gpt-4o"]
    }
}
```

---

## Model Registry

### Model Definitions

```python
@dataclass
class ModelDefinition:
    """Definition of an AI model."""
    
    name: str
    provider: str
    api_key_name: str
    quality_score: float  # 0.0 - 1.0
    cost_per_1k_tokens: float
    context_window: int
    max_tokens: int
    supports_vision: bool = False
    supports_code: bool = False
    supports_tools: bool = True
    latency_ms: int = 1000  # Estimated latency
    
    def get_cost(self, input_tokens: int, output_tokens: int) -> float:
        """Calculate cost for this request."""
        total_tokens = input_tokens + output_tokens
        return (total_tokens / 1000) * self.cost_per_1k_tokens


MODEL_REGISTRY = {
    # Anthropic Models
    "claude-3-5-sonnet": ModelDefinition(
        name="claude-3-5-sonnet-20241022",
        provider="anthropic",
        api_key_name="ANTHROPIC_API_KEY",
        quality_score=0.95,
        cost_per_1k_tokens=0.015,
        context_window=200000,
        max_tokens=8192,
        supports_vision=True,
        supports_code=True,
        latency_ms=2000
    ),
    
    "claude-3-haiku": ModelDefinition(
        name="claude-3-haiku-20240307",
        provider="anthropic",
        api_key_name="ANTHROPIC_API_KEY",
        quality_score=0.75,
        cost_per_1k_tokens=0.00025,
        context_window=200000,
        max_tokens=4096,
        latency_ms=800
    ),
    
    # OpenAI Models
    "gpt-4o": ModelDefinition(
        name="gpt-4o-2024-05-13",
        provider="openai",
        api_key_name="OPENAI_API_KEY",
        quality_score=0.92,
        cost_per_1k_tokens=0.005,
        context_window=128000,
        max_tokens=4096,
        supports_vision=True,
        supports_code=True,
        latency_ms=1500
    ),
    
    "gpt-4o-mini": ModelDefinition(
        name="gpt-4o-mini-2024-07-18",
        provider="openai",
        api_key_name="OPENAI_API_KEY",
        quality_score=0.70,
        cost_per_1k_tokens=0.00015,
        context_window=128000,
        max_tokens=16384,
        supports_vision=True,
        supports_code=True,
        latency_ms=500
    ),
    
    # Google Models
    "gemini-2.0-flash-thinking": ModelDefinition(
        name="gemini-2.0-flash-thinking-exp-1219",
        provider="google",
        api_key_name="GOOGLE_API_KEY",
        quality_score=0.88,
        cost_per_1k_tokens=0.000075,
        context_window=1000000,
        max_tokens=8192,
        supports_vision=True,
        latency_ms=1200
    ),
    
    "gemini-1.5-flash": ModelDefinition(
        name="gemini-1.5-flash-002",
        provider="google",
        api_key_name="GOOGLE_API_KEY",
        quality_score=0.72,
        cost_per_1k_tokens=0.000075,
        context_window=1000000,
        max_tokens=8192,
        supports_vision=True,
        latency_ms=600
    ),
    
    # Local Models (via Ollama)
    "gliner2-base-v1": ModelDefinition(
        name="gliner2-base-v1",
        provider="local",
        api_key_name="",
        quality_score=0.65,
        cost_per_1k_tokens=0.0,  # Free
        context_window=32768,
        max_tokens=4096,
        latency_ms=3000
    ),
    
    "groq-llama-3.3-70b": ModelDefinition(
        name="llama-3.3-70b-versatile",
        provider="local",
        api_key_name="",
        quality_score=0.80,
        cost_per_1k_tokens=0.0,
        context_window=32768,
        max_tokens=4096,
        latency_ms=8000
    ),
    
    "llama-3.1-70b": ModelDefinition(
        name="llama3.1:70b",
        provider="local",
        api_key_name="",
        quality_score=0.78,
        cost_per_1k_tokens=0.0,
        context_window=131072,
        max_tokens=4096,
        latency_ms=10000
    )
}
```

---

## Routing Strategy

### Primary Routing Logic

```python
class ModelRouter:
    """Routes tasks to optimal models."""
    
    def __init__(self, model_registry: dict, task_requirements: dict):
        self.model_registry = model_registry
        self.task_requirements = task_requirements
        self.rate_limit_tracker = RateLimitTracker()
        self.cost_tracker = CostTracker()
    
    def select_model(
        self,
        task_type: TaskType,
        input_tokens: int,
        estimated_output_tokens: int,
        constraints: dict | None = None
    ) -> str:
        """Select the optimal model for this task."""
        
        requirements = self.task_requirements[task_type]
        constraints = constraints or {}
        
        # Filter models by requirements
        candidates = self._filter_models(requirements, constraints)
        
        if not candidates:
            raise NoSuitableModelError(f"No models meet requirements for {task_type}")
        
        # Score each candidate
        scored_models = [
            (model_name, self._score_model(model_name, requirements, input_tokens, estimated_output_tokens))
            for model_name in candidates
        ]
        
        # Sort by score (higher is better)
        scored_models.sort(key=lambda x: x[1], reverse=True)
        
        # Return best model
        return scored_models[0][0]
    
    def _filter_models(self, requirements: dict, constraints: dict) -> list[str]:
        """Filter models that meet requirements."""
        
        candidates = []
        
        for model_name, model_def in self.model_registry.items():
            # Check quality
            if model_def.quality_score < requirements["min_quality"]:
                continue
            
            # Check latency
            if model_def.latency_ms > requirements["max_latency"]:
                continue
            
            # Check cost
            cost = model_def.get_cost(1000, 1000)  # Estimate
            if cost > requirements["max_cost_per_1k_tokens"]:
                continue
            
            # Check context window
            if model_def.context_window < requirements["context_window"]:
                continue
            
            # Check constraints
            if constraints.get("vision_required") and not model_def.supports_vision:
                continue
            
            if constraints.get("code_required") and not model_def.supports_code:
                continue
            
            # Check rate limits
            if self.rate_limit_tracker.is_rate_limited(model_name):
                continue
            
            candidates.append(model_name)
        
        return candidates
    
    def _score_model(
        self,
        model_name: str,
        requirements: dict,
        input_tokens: int,
        output_tokens: int
    ) -> float:
        """Score a model for this task (higher is better)."""
        
        model_def = self.model_registry[model_name]
        
        # Quality score (40% weight)
        quality_score = model_def.quality_score
        
        # Cost score (30% weight) - lower cost is better
        cost = model_def.get_cost(input_tokens, output_tokens)
        max_cost = requirements["max_cost_per_1k_tokens"]
        cost_score = 1.0 - (cost / max_cost)
        
        # Latency score (20% weight) - lower latency is better
        max_latency = requirements["max_latency"]
        latency_score = 1.0 - (model_def.latency_ms / max_latency)
        
        # Context window score (10% weight) - larger is better
        context_score = model_def.context_window / 200000  # Normalize to 200k
        
        # Weighted score
        total_score = (
            quality_score * 0.4 +
            cost_score * 0.3 +
            latency_score * 0.2 +
            context_score * 0.1
        )
        
        return total_score
```

### Fallback Strategy

```python
class FallbackManager:
    """Manages fallback models when primary fails."""
    
    def __init__(self, router: ModelRouter):
        self.router = router
        self.fallback_chains = {
            "claude-3-5-sonnet": ["gpt-4o", "gemini-2.5-flash", "groq-llama-3.3-70b"],
            "gpt-4o": ["claude-3-5-sonnet", "gemini-2.5-flash", "groq-llama-3.3-70b"],
            "gemini-2.5-flash": ["claude-3-5-sonnet", "gpt-4o", "groq-llama-3.3-70b"],
            "groq-llama-3.3-70b": ["claude-3-haiku", "gpt-4o-mini", "gemini-2.5-flash"]
        }
    
    async def call_with_fallback(
        self,
        task_type: TaskType,
        messages: list,
        primary_model: str | None = None,
        max_fallbacks: int = 3
    ) -> dict:
        """Call model with automatic fallback."""
        
        if primary_model is None:
            primary_model = self.router.select_model(
                task_type,
                self._estimate_tokens(messages),
                self._estimate_output_tokens(messages)
            )
        
        models_to_try = [primary_model] + self.fallback_chains.get(primary_model, [])[:max_fallbacks]
        
        last_error = None
        
        for model_name in models_to_try:
            try:
                result = await self._call_model(model_name, messages)
                
                # Log successful call
                logger.info(
                    "model_call_success",
                    model=model_name,
                    task_type=task_type.value,
                    fallbacks_tried=models_to_try.index(model_name)
                )
                
                return result
                
            except Exception as e:
                last_error = e
                
                # Log failure
                logger.warning(
                    "model_call_failed",
                    model=model_name,
                    error=str(e),
                    error_type=type(e).__name__
                )
                
                # Update rate limit tracker
                if isinstance(e, RateLimitError):
                    self.router.rate_limit_tracker.record_rate_limit(model_name)
                
                continue
        
        # All models failed
        raise AllModelsFailedError(f"All models failed. Last error: {last_error}")
```

---

## Rate Limit Management

### Rate Limit Tracker

```python
class RateLimitTracker:
    """Tracks rate limits for each model and API key."""
    
    def __init__(self):
        self.rate_limits = {}  # {(model_name, api_key): RateLimitInfo}
        self.requests = {}  # {(model_name, api_key): [timestamp, ...]}
    
    def record_request(self, model_name: str, api_key: str):
        """Record a request."""
        key = (model_name, api_key)
        
        if key not in self.requests:
            self.requests[key] = []
        
        now = time.time()
        self.requests[key].append(now)
        
        # Clean old requests (older than 1 minute)
        self.requests[key] = [t for t in self.requests[key] if now - t < 60]
    
    def is_rate_limited(self, model_name: str, api_key: str | None = None) -> bool:
        """Check if model/key is rate limited."""
        
        # Get rate limit info
        model_def = MODEL_REGISTRY.get(model_name)
        if not model_def:
            return False
        
        # Check each API key for this model
        keys = self._get_api_keys(model_name)
        
        for key in keys:
            request_count = len(self.requests.get((model_name, key), []))
            
            if request_count >= model_def.rpm:
                return True
        
        return False
    
    def get_available_key(self, model_name: str) -> str:
        """Get an API key that's not rate limited."""
        
        keys = self._get_api_keys(model_name)
        
        for key in keys:
            if not self.is_rate_limited(model_name, key):
                return key
        
        # All keys are rate limited
        raise AllKeysRateLimitedError(f"All keys for {model_name} are rate limited")
    
    def _get_api_keys(self, model_name: str) -> list[str]:
        """Get all API keys for a model."""
        
        model_def = MODEL_REGISTRY.get(model_name)
        if not model_def:
            return []
        
        key_name = model_def.api_key_name
        keys = os.environ.get(key_name, "").split(",")
        
        return [k.strip() for k in keys if k.strip()]
```

### Rate Limit Handling in LiteLLM

LiteLLM handles rate limits automatically with:

```yaml
litellm_settings:
  num_retries: 3
  request_timeout: 10
  allowed_fails: 3
  cooldown_time: 30
```

This means:
- Retry up to 3 times on rate limit errors
- Wait 30 seconds before retrying a failed model
- Remove model from rotation after 3 failures

---

## Cost Optimization

### Budget Management

```python
class BudgetManager:
    """Manages budgets for investigations."""
    
    def __init__(self):
        self.investigation_budgets = {}  # {investigation_id: Budget}
    
    def check_budget(self, investigation_id: str, estimated_cost: float) -> bool:
        """Check if investigation has budget for this request."""
        
        budget = self.investigation_budgets.get(investigation_id)
        if not budget:
            return True  # No budget set, allow
        
        return budget.remaining >= estimated_cost
    
    def record_cost(self, investigation_id: str, model_name: str, cost: float):
        """Record cost for investigation."""
        
        budget = self.investigation_budgets.get(investigation_id)
        if budget:
            budget.remaining -= cost
            budget.spent += cost
            budget.model_costs[model_name] = budget.model_costs.get(model_name, 0) + cost
    
    def get_cost_report(self, investigation_id: str) -> dict:
        """Get cost report for investigation."""
        
        budget = self.investigation_budgets.get(investigation_id)
        if not budget:
            return {}
        
        return {
            "total_budget": budget.total,
            "remaining": budget.remaining,
            "spent": budget.spent,
            "by_model": budget.model_costs,
            "utilization": budget.spent / budget.total if budget.total > 0 else 0
        }


@dataclass
class Budget:
    total: float
    remaining: float
    spent: float = 0.0
    model_costs: dict = field(default_factory=dict)
    duration_days: int = 30
    created_at: str = field(default_factory=lambda: datetime.now().isoformat())
```

### Cost-Aware Routing

```python
def select_model_with_budget(
    router: ModelRouter,
    budget_manager: BudgetManager,
    investigation_id: str,
    task_type: TaskType,
    input_tokens: int,
    estimated_output_tokens: int
) -> str:
    """Select model considering budget constraints."""
    
    # Get all candidates
    requirements = TASK_REQUIREMENTS[task_type]
    candidates = router._filter_models(requirements, {})
    
    # Sort by cost (cheapest first)
    candidates_with_cost = [
        (model_name, MODEL_REGISTRY[model_name].get_cost(input_tokens, estimated_output_tokens))
        for model_name in candidates
    ]
    candidates_with_cost.sort(key=lambda x: x[1])
    
    # Find cheapest model that meets budget
    for model_name, cost in candidates_with_cost:
        if budget_manager.check_budget(investigation_id, cost):
            return model_name
    
    # No model meets budget
    raise BudgetExceededError("No model available within budget")
```

---

## API Key Pooling

### Key Pool Configuration

```python
class KeyPool:
    """Manages API keys for each provider."""
    
    def __init__(self):
        self.pools = {
            "anthropic": [],
            "openai": [],
            "google": [],
            "openrouter": []
        }
        self._load_keys_from_env()
    
    def _load_keys_from_env(self):
        """Load keys from environment variables."""
        
        # Anthropic
        anthropic_keys = os.environ.get("ANTHROPIC_API_KEY", "").split(",")
        self.pools["anthropic"] = [k.strip() for k in anthropic_keys if k.strip()]
        
        # OpenAI
        openai_keys = os.environ.get("OPENAI_API_KEY", "").split(",")
        self.pools["openai"] = [k.strip() for k in openai_keys if k.strip()]
        
        # Google
        google_keys = os.environ.get("GOOGLE_API_KEY", "").split(",")
        self.pools["google"] = [k.strip() for k in google_keys if k.strip()]
        
        # OpenRouter
        openrouter_keys = os.environ.get("OPENROUTER_API_KEY", "").split(",")
        self.pools["openrouter"] = [k.strip() for k in openrouter_keys if k.strip()]
    
    def get_key(self, provider: str, model_name: str) -> str:
        """Get an available API key for this provider/model."""
        
        keys = self.pools.get(provider, [])
        
        if not keys:
            raise NoKeysAvailableError(f"No keys available for provider: {provider}")
        
        # Simple round-robin
        # In production, use rate limit tracking
        return keys[hash(model_name) % len(keys)]
    
    def add_key(self, provider: str, api_key: str):
        """Add a new API key to the pool."""
        
        if provider not in self.pools:
            self.pools[provider] = []
        
        self.pools[provider].append(api_key)
    
    def remove_key(self, provider: str, api_key: str):
        """Remove an API key from the pool."""
        
        if provider in self.pools:
            self.pools[provider] = [k for k in self.pools[provider] if k != api_key]
```

### LiteLLM Key Configuration

```yaml
model_list:
  # Multiple keys for same model (load balancing)
  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_1}
      rpm: 500
    model_info:
      id: claude-primary-1
  
  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_2}
      rpm: 500
    model_info:
      id: claude-primary-2
  
  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_3}
      rpm: 500
    model_info:
      id: claude-primary-3
```

---

## Model Performance Tracking

### Performance Metrics

```python
@dataclass
class ModelPerformance:
    """Tracks performance metrics for a model."""
    
    model_name: str
    total_calls: int = 0
    successful_calls: int = 0
    failed_calls: int = 0
    total_latency_ms: int = 0
    total_cost: float = 0.0
    average_latency_ms: float = 0.0
    success_rate: float = 0.0
    last_call_time: str | None = None
    
    def record_call(self, latency_ms: int, cost: float, success: bool):
        """Record a model call."""
        
        self.total_calls += 1
        self.total_latency_ms += latency_ms
        self.total_cost += cost
        
        if success:
            self.successful_calls += 1
        else:
            self.failed_calls += 1
        
        # Update averages
        self.average_latency_ms = self.total_latency_ms / self.total_calls
        self.success_rate = self.successful_calls / self.total_calls
        self.last_call_time = datetime.now().isoformat()


class PerformanceTracker:
    """Tracks performance across all models."""
    
    def __init__(self):
        self.models: dict[str, ModelPerformance] = {}
    
    def get_or_create(self, model_name: str) -> ModelPerformance:
        """Get or create performance tracker for model."""
        
        if model_name not in self.models:
            self.models[model_name] = ModelPerformance(model_name=model_name)
        
        return self.models[model_name]
    
    def record_call(
        self,
        model_name: str,
        latency_ms: int,
        cost: float,
        success: bool
    ):
        """Record a model call."""
        
        tracker = self.get_or_create(model_name)
        tracker.record_call(latency_ms, cost, success)
    
    def get_best_model(self, task_type: TaskType) -> str:
        """Get best performing model for task type."""
        
        # Filter models that support this task type
        candidates = [
            model_name
            for model_name, model_def in MODEL_REGISTRY.items()
            if self._model_supports_task(model_def, task_type)
        ]
        
        # Sort by success rate, then latency
        candidates.sort(
            key=lambda m: (
                self.models[m].success_rate,
                -self.models[m].average_latency_ms
            ),
            reverse=True
        )
        
        return candidates[0] if candidates else None
    
    def _model_supports_task(self, model_def: ModelDefinition, task_type: TaskType) -> bool:
        """Check if model supports task type."""
        
        if task_type == TaskType.VISION_ANALYSIS:
            return model_def.supports_vision
        
        if task_type == TaskType.CODE_ANALYSIS:
            return model_def.supports_code
        
        return True
```

---

## A/B Testing Models

### Experiment Framework

```python
class ModelExperiment:
    """A/B test different models for a task."""
    
    def __init__(self, experiment_id: str, task_type: TaskType):
        self.experiment_id = experiment_id
        self.task_type = task_type
        self.variants = {}  # {variant_name: {model_name, traffic_percentage}}
        self.results = {}  # {variant_name: [result, ...]}
    
    def add_variant(self, variant_name: str, model_name: str, traffic_percentage: float):
        """Add a variant to the experiment."""
        
        self.variants[variant_name] = {
            "model_name": model_name,
            "traffic_percentage": traffic_percentage
        }
        self.results[variant_name] = []
    
    def select_variant(self) -> tuple[str, str]:
        """Select a variant based on traffic percentage."""
        
        import random
        
        rand = random.random() * 100
        cumulative = 0
        
        for variant_name, variant in self.variants.items():
            cumulative += variant["traffic_percentage"]
            
            if rand <= cumulative:
                return variant_name, variant["model_name"]
        
        # Fallback to first variant
        first_variant = next(iter(self.variants.items()))
        return first_variant[0], first_variant[1]["model_name"]
    
    def record_result(self, variant_name: str, result: dict):
        """Record a result for a variant."""
        
        self.results[variant_name].append(result)
    
    def get_results(self) -> dict:
        """Get experiment results."""
        
        return {
            variant_name: {
                "model_name": self.variants[variant_name]["model_name"],
                "traffic_percentage": self.variants[variant_name]["traffic_percentage"],
                "results": results,
                "success_rate": sum(1 for r in results if r.get("success")) / len(results) if results else 0,
                "average_latency": sum(r.get("latency", 0) for r in results) / len(results) if results else 0,
                "average_cost": sum(r.get("cost", 0) for r in results) / len(results) if results else 0
            }
            for variant_name, results in self.results.items()
        }
```

---

## Configuration Examples

### LiteLLM Full Configuration

```yaml
# litellm_config.yaml

model_list:
  # Primary reasoning models (with multiple keys for load balancing)
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_1}
      rpm: 500
    model_info:
      id: claude-primary-1
  
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_2}
      rpm: 500
    model_info:
      id: claude-primary-2
  
  - model_name: oracle-reasoning-primary
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: ${ANTHROPIC_API_KEY_3}
      rpm: 500
    model_info:
      id: claude-primary-3
  
  # Secondary reasoning models
  - model_name: oracle-reasoning-secondary
    litellm_params:
      model: openai/gpt-4o
      api_key: ${OPENAI_API_KEY_1}
      rpm: 500
    model_info:
      id: gpt4o-secondary-1
  
  - model_name: oracle-reasoning-secondary
    litellm_params:
      model: openai/gpt-4o
      api_key: ${OPENAI_API_KEY_2}
      rpm: 500
    model_info:
      id: gpt4o-secondary-2
  
  # Fast extraction models
  - model_name: oracle-extraction
    litellm_params:
      model: fastino/gliner2-base-v1
      api_base: http://localhost:11434
      rpm: 1000
  
  # Vision models
  - model_name: oracle-vision
    litellm_params:
      model: gemini/gemini-2.0-flash-exp
      api_key: ${GOOGLE_API_KEY}
      rpm: 500

router_settings:
  enable_pre_call_checks: true
  
  # Fallback configuration
  fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]},
    {"oracle-reasoning-secondary": ["oracle-extraction"]},
    {"oracle-vision": ["oracle-reasoning-primary"]}
  ]
  
  # Content policy fallbacks
  content_policy_fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]}
  ]
  
  # Context window fallbacks
  context_window_fallbacks: [
    {"oracle-reasoning-primary": ["oracle-reasoning-secondary"]}
  ]

litellm_settings:
  num_retries: 3
  request_timeout: 120
  allowed_fails: 3
  cooldown_time: 60
  
  # Drop params (remove sensitive data from logs)
  drop_params: ["api_key", "headers", "api_base"]
```

### Environment Variables

```bash
# .env

# Anthropic
ANTHROPIC_API_KEY_1=sk-ant-...
ANTHROPIC_API_KEY_2=sk-ant-...
ANTHROPIC_API_KEY_3=sk-ant-...

# OpenAI
OPENAI_API_KEY_1=sk-...
OPENAI_API_KEY_2=sk-...

# Google
GOOGLE_API_KEY=...

# OpenRouter (optional)
OPENROUTER_API_KEY=sk-or-...

# LiteLLM
LITELLM_MASTER_KEY=sk-litellm-...

# Budgets (per investigation)
INVESTIGATION_EPSTEIN_BUDGET=500.00
INVESTIGATION_BIBLE_BUDGET=1000.00
```

---

## Usage Examples

### Basic Usage

```python
from oracle.multi_model import ModelRouter, FallbackManager

# Initialize
router = ModelRouter(MODEL_REGISTRY, TASK_REQUIREMENTS)
fallback_manager = FallbackManager(router)

# Select model for task
task_type = TaskType.COMPLEX_REASONING
messages = [
    {"role": "system", "content": "You are a research assistant."},
    {"role": "user", "content": "Analyze the following text..."}
]

# Get model
model_name = router.select_model(
    task_type,
    input_tokens=1000,
    estimated_output_tokens=2000
)

# Call with fallback
result = await fallback_manager.call_with_fallback(
    task_type,
    messages,
    primary_model=model_name
)
```

### With Budget Constraints

```python
from oracle.multi_model import ModelRouter, BudgetManager

# Initialize
router = ModelRouter(MODEL_REGISTRY, TASK_REQUIREMENTS)
budget_manager = BudgetManager()

# Set budget
budget_manager.investigation_budgets["inv-abc123"] = Budget(
    total=100.0,
    remaining=100.0
)

# Select model with budget
model_name = select_model_with_budget(
    router,
    budget_manager,
    "inv-abc123",
    TaskType.ENTITY_EXTRACTION,
    input_tokens=5000,
    estimated_output_tokens=1000
)
```

### A/B Testing

```python
from oracle.multi_model import ModelExperiment

# Create experiment
experiment = ModelExperiment(
    experiment_id="exp-001",
    task_type=TaskType.SUMMARIZATION
)

# Add variants
experiment.add_variant("claude", "claude-3-haiku", traffic_percentage=50)
experiment.add_variant("gpt", "gpt-4o-mini", traffic_percentage=50)

# Run experiment
for _ in range(100):
    variant_name, model_name = experiment.select_variant()
    
    result = await call_model(model_name, messages)
    experiment.record_result(variant_name, result)

# Get results
results = experiment.get_results()
print(results)
```

---

## Monitoring & Debugging

### Metrics to Track

**Per-Model Metrics:**
- Total calls
- Success rate
- Average latency
- Total cost
- Rate limit hits

**Per-Provider Metrics:**
- Total API calls
- Total cost
- Error rate

**Per-Investigation Metrics:**
- Total cost
- Cost by model
- Budget utilization
- Model usage distribution

### Logging

```python
import structlog

logger = structlog.get_logger()

# Log model selection
logger.info(
    "model_selected",
    task_type=task_type.value,
    model_name=model_name,
    quality_score=model_def.quality_score,
    estimated_cost=cost,
    reasoning=reasoning
)

# Log fallback
logger.warning(
    "model_fallback",
    primary_model=primary_model,
    fallback_model=fallback_model,
    error=str(error),
    error_type=type(error).__name__
)

# Log rate limit
logger.warning(
    "rate_limit_hit",
    model=model_name,
    api_key_hash=hash(api_key),
    requests_in_last_minute=request_count
)
```

---

## Troubleshooting

### Common Issues

**Issue: All models are rate limited**

**Solution:**
1. Check API key configuration
2. Verify rate limits with providers
3. Add more API keys to the pool
4. Implement request queuing

**Issue: Costs are too high**

**Solution:**
1. Use cheaper models for less critical tasks
2. Implement budget caps
3. Use local models where possible
4. Cache results

**Issue: Latency is too high**

**Solution:**
1. Use faster models
2. Implement request batching
3. Use local models for latency-sensitive tasks
4. Implement request prioritization

**Issue: Quality is too low**

**Solution:**
1. Use higher-quality models for critical tasks
2. Implement consensus mechanisms
3. Use multiple models and aggregate results
4. Fine-tune prompts

---

## Conclusion

This multi-model orchestration system provides:

✅ **Automatic model selection** based on task requirements
✅ **Intelligent fallback** when models fail or are rate limited
✅ **Cost optimization** through budget management and cost-aware routing
✅ **Rate limit handling** across multiple API keys
✅ **Performance tracking** to identify best models
✅ **A/B testing** capabilities for model comparison

The system is designed to be reliable, cost-effective, and performant for long-running investigations.

---

**Document Version:** 1.0  
**Last Updated:** 2025-01-02  
**Status:** VALIDATED ✅
