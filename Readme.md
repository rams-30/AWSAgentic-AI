# Building an Agentic AI Platform — Day 1: Deploying Your First Agent on Bedrock AgentCore

*This is Part 1 of a 5-part series where I document building an agentic AI platform from scratch — from a single RAG agent to a multi-agent system with automated tenant onboarding, all deployed via CI/CD.*

---

### Why I started this

I've been following the agentic AI space for a while. Not chatbots — the kind of agents that can reason over a problem, decide which tools to call, remember past conversations, and eventually work with other agents.

At work, the opportunity came up to build something like this as a platform — something teams could get onboarded to without needing to understand every piece of the infrastructure. But before any of that, I needed to answer one question: *can I get a single agent running on Bedrock AgentCore, deployed entirely through CI/CD, with no console clicking?*

That was Day 1.

### The goal

- Deploy a single agent on AWS Bedrock AgentCore
- Use Terraform for infrastructure
- Use GitLab CI for the pipeline
- Give the agent access to a Knowledge Base (RAG)
- No manual steps. Push code → agent is live.

### The agent runtime (Terraform)

AgentCore runs your agent as a managed container. You give it an ECR image URI, an IAM role, and some environment variables. It handles scaling, networking, health checks.

Here's the core Terraform module:

```
resource "aws_bedrockagentcore_agent_runtime" "this" {
  agent_runtime_name = "${var.tenant_name}_${var.environment}_agentcore"
  role_arn           = var.agentcore_execution_role_arn

  agent_runtime_artifact {
    container_configuration {
      container_uri = var.ecr_image_uri
    }
  }

  network_configuration {
    network_mode = "PUBLIC"
  }

  environment_variables = {
    AWS_REGION  = var.aws_region
    TENANT_NAME = var.tenant_name
    MODEL_ID    = var.model_id
    KB_ID       = var.knowledge_base_id
    MEMORY_ID   = var.memory_id
  }
}
```

Nothing wild. The interesting part is that `MODEL_ID` can be either a foundation model ID or an inference profile ARN — same module works for both.

### The agent code

I used the [Strands Agents SDK](https://github.com/strands-agents/sdk-python) because it felt like the most straightforward way to build tool-using agents on Bedrock.

```
from strands import Agent, tool
from strands.models import BedrockModel
from bedrock_agentcore.runtime import BedrockAgentCoreApp

app = BedrockAgentCoreApp()

MODEL_ID = os.environ.get("MODEL_ID", "anthropic.claude-sonnet-4-6")
KB_ID = os.environ.get("KB_ID", "")

@tool
def search_knowledge_base(query: str) -> str:
    """Search the knowledge base for relevant information.

    Args:
        query: The search query to find relevant documents.

    Returns:
        Relevant information from the knowledge base.
    """
    if not KB_ID:
        return "Knowledge base not configured."
    # Uses Bedrock KB retrieval
    ...

model = BedrockModel(model_id=MODEL_ID)

agent = Agent(
    model=model,
    system_prompt="You are a helpful assistant. Use search_knowledge_base to answer questions.",
    tools=[search_knowledge_base],
)

@app.entrypoint
async def invoke(payload, context=None):
    prompt = payload.get("prompt", "")
    session_id = payload.get("sessionId", "default")
    stream = agent.stream_async(prompt)
    async for event in stream:
        if "data" in event and isinstance(event["data"], str):
            yield event["data"]

if __name__ == "__main__":
    app.run()
```

The `BedrockAgentCoreApp` gives you the runtime wrapper — it handles the HTTP server, health checks, payload parsing. You just define your entrypoint and return a response (or stream one).

### The Dockerfile

```
FROM public.ecr.aws/docker/library/python:3.12-slim
WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip install --no-cache-dir aws-opentelemetry-distro==0.10.1

RUN useradd -m -u 1000 bedrock_agentcore
USER bedrock_agentcore

COPY . .

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/ping')" || exit 1

CMD ["opentelemetry-instrument", "python", "-m", "agentcore"]
```

Small note: the OTel distro is optional but gives you X-Ray tracing out of the box. Worth it.

### The CI pipeline

The GitLab CI pipeline uses OIDC to assume an IAM role (no static credentials), builds the Docker image, pushes to ECR, and runs `terraform apply`.

```
stages:
  - setup
  - plan
  - apply

check-and-create-ecr-repo:
  stage: setup
  script:
    - # Assume role via OIDC
    - aws ecr describe-repositories --repository-names ${REPO_NAME} || \
      aws ecr create-repository --repository-name $REPO_NAME

plan:
  stage: plan
  script:
    - terraform init
    - terraform plan -var-file=environments/tfvars/${ENV}.tfvars

apply:
  stage: apply
  script:
    - terraform apply -auto-approve -var-file=environments/tfvars/${ENV}.tfvars
```

Push to `dev` branch → pipeline runs → agent is deployed. That's it.

### What I learned

- AgentCore's managed container model is nice. No Lambda cold starts, no ECS cluster management. You just give it a container image and it runs.
- The OIDC → STS AssumeRoleWithWebIdentity pattern is clean. No stored secrets in CI.
- Keeping the Terraform module generic (accepting `model_id` as just a string) means it works for inference profiles later with zero changes.

### What's next

Day 1 gave me a working agent that can search a knowledge base. But it can't actually *do* anything in the real world yet. In Day 2, I'll connect it to tools through an MCP Gateway — so it can call Lambda functions, check statuses, and take actions.

*→ Day 2: Giving the Agent Tools via MCP Gateway*
