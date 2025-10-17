# AWS Contextual Chatbot with Amazon Bedrock Knowledge Bases

## Overview

The AWS Contextual Chatbot is a production-ready, enterprise-grade Retrieval-Augmented Generation (RAG) solution built on AWS serverless infrastructure. The system enables organizations to query their document repositories using natural language, receiving accurate, contextual answers with source citations.

This solution demonstrates best practices for building secure, scalable, and cost-effective AI-powered applications using Amazon Bedrock Knowledge Bases, Lambda functions, and multi-region disaster recovery.

⚠️ **Important**
- Running this code might result in charges to your AWS account.
- We recommend that you grant your code least privilege. At most, grant only the minimum permissions required to perform the task.
- This code is not tested in every AWS Region. For more information, see [AWS Regional Services](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/).

## Prerequisites

Before deploying this solution, ensure you have the following:

- AWS CLI installed and configured with appropriate permissions
- Node.js ≥ 22.9.0 and npm
- AWS CDK CLI v2
- Docker Desktop installed and running
- Access to Amazon Bedrock foundation models in your target regions

### Required AWS Permissions

Your AWS credentials must have permissions to:
- Create and manage CloudFormation stacks
- Deploy Lambda functions and API Gateway
- Create S3 buckets and configure CloudFront
- Access Amazon Bedrock services
- Create IAM roles and policies

### Bedrock Model Access

**CRITICAL**: Before deployment, enable access to these foundation models in the Amazon Bedrock console:

1. Navigate to the [Amazon Bedrock console](https://console.aws.amazon.com/bedrock/home)
2. Click **Model access** in the bottom-left corner
3. Click **Manage model access** in the top-right
4. Enable access for:
   - **Titan Embeddings G1 - Text**: `amazon.titan-embed-text-v1` (for Knowledge Base)
   - **Anthropic Claude 3 Sonnet**: `anthropic.claude-3-sonnet-20240229-v1:0` (for answer generation)

**Note**: Enable model access in **both** us-west-2 and us-east-1 regions for disaster recovery.

## Assumptions & Design Decisions

### Business Context Assumptions

**Document Volume & Types**
- Expected document volume: 100-10,000 documents per knowledge base
- Document types: PDF, TXT, DOCX, MD, HTML (common enterprise formats)
- Average document size: 1-50 pages per document
- Update frequency: Weekly to monthly document refreshes

**Usage Patterns**
- Concurrent users: 10-100 simultaneous users
- Query volume: 100-1,000 queries per day
- Query complexity: Multi-sentence questions requiring document synthesis
- Response time requirement: < 10 seconds for complex queries

**Data & Compliance**
- Data sensitivity: Internal enterprise documents (not public data)
- Compliance requirements: SOC 2, GDPR-ready architecture
- Data retention: 7 years for audit purposes
- Access control: Role-based access (future enhancement)

### Technical Design Decisions

**Architecture Pattern: Serverless-First**
- **Decision**: Fully serverless over containerized architecture
- **Rationale**: 
  - Zero infrastructure management overhead
  - Automatic scaling without capacity planning
  - Pay-per-use cost model aligns with variable workloads
  - Faster time-to-market for MVP deployment

**RAG Implementation: Manual Two-Step Process**
- **Decision**: Separate Retrieve + InvokeModel calls vs RetrieveAndGenerate API
- **Rationale**: 
  - RetrieveAndGenerate doesn't support Bedrock Guardrails (critical for enterprise)
  - Provides finer control over the RAG pipeline
  - Enables custom prompt engineering and context manipulation
  - Better error handling and retry logic

**Document Chunking Strategy: 500 Tokens, 20% Overlap**
- **Decision**: Fixed-size chunks with significant overlap
- **Rationale**:
  - 500 tokens ≈ 1-2 paragraphs (optimal semantic unit)
  - 20% overlap prevents context loss at boundaries
  - Balances precision vs. recall in retrieval
  - Compatible with Claude 3 Sonnet's context window

**Vector Store: Bedrock-Managed OpenSearch Serverless**
- **Decision**: Managed service over self-hosted alternatives
- **Rationale**:
  - Zero operational overhead (no cluster management)
  - Automatic scaling and cost optimization
  - Tight integration with Bedrock Knowledge Bases
  - No additional infrastructure to monitor or maintain

**Disaster Recovery: Manual Backend Failover**
- **Decision**: Manual config.json update vs automatic DNS failover
- **Rationale**:
  - Simpler implementation for MVP (no custom domain required)
  - Manual control over failover timing and validation
  - Cost-effective (no Route 53 health checks needed)
  - Can be enhanced with automatic DNS failover post-MVP

**Frontend Architecture: Static SPA on S3 + CloudFront**
- **Decision**: Static hosting over Amplify or EC2
- **Rationale**:
  - Simplest deployment model with global CDN
  - Automatic HTTPS and edge caching
  - Minimal cost and operational overhead
  - Easy to implement origin failover for DR

### Operational Assumptions

**Monitoring & Alerting**
- CloudWatch native monitoring sufficient for initial deployment
- SNS email notifications for critical alerts
- No third-party monitoring tools required initially
- 24/7 monitoring not required (business hours support acceptable)

**Maintenance Windows**
- Monthly maintenance windows acceptable
- Zero-downtime deployments via blue-green approach
- Emergency patches during business hours with 2-hour advance notice

**Support Model**
- AWS Support Center for infrastructure issues
- Internal team handles application-level support
- No dedicated DevOps team required initially
- Documentation and runbooks sufficient for L1 support

**Cost Optimization**
- Pay-per-use model acceptable for variable workloads
- No upfront capacity reservations required
- Monthly cost reviews and optimization cycles
- Right-sizing recommendations based on usage patterns

### Constraints & Limitations

**Current Limitations**
- No user authentication (planned for Phase 2)
- No multi-tenant support (single knowledge base per deployment)
- Manual document upload only (no API-based ingestion)
- English language only (no multi-language support)

**Future Enhancements**
- Cognito integration for user management
- Multi-knowledge base support
- API-based document ingestion
- Multi-language document processing
- Custom domain with automatic DNS failover

## Architecture

The solution implements a fully serverless, multi-region architecture with automatic failover capabilities:

![AWS Contextual Chatbot Architecture](https://design-inspector.a2z.com?lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=contextual_chat_bot#R%3Cmxfile%20modified%3D%222025-10-16T15%3A19%3A27.112Z%22%20host%3D%22design-inspector.a2z.com%22%20agent%3D%22Mozilla%2F5.0%20(Macintosh%3B%20Intel%20Mac%20OS%20X%2010_15_7)%20AppleWebKit%2F537.36%20(KHTML%2C%20like%20Gecko)%20Chrome%2F141.0.0.0%20Safari%2F537.36%22%20etag%3D%22qI-YjhXOb1QcUFwnp_pf%22%3E%3Cdiagram%20id%3D%22rO6U7AJN-wKRN4-TpZDVF%22%20name%3D%22Page-1%22%3E7Z1rc5s4F8c%2FTWa6L8hISNxeOnaT9ull26ZttvsmI0A4bLFxMW6SfvpHwmADJvgS7Ei2djrbIgTIQpcf%2F3N0dIb6o4erhEzuPsQ%2Bjc504D%2BcocGZrkOMMfuLpzzOUyxkzBOGSejnmZYJ1%2BEfmieCPHUW%2BnRayZjGcZSGk2qiF4%2FH1EsraSRJ4vtqtiCOqk%2BdkCFdSbj2SLSaehP66d081TbAMv0NDYd3xZMhyM%2BMSJE5T5jeET%2B%2BLyWh12eon8RxOv%2FX6KFPI155Rb38cx3eznz3zfjS%2BTki37S%2Bn%2F76oOV3G9OHlOd%2F638n0awobOnGX%2Bg0niUeHdCpl4STNE7YVUmeOM8%2BS8ZnqPegsQoOh2MtHE8nrApZRtT7Ddn%2FhnQc%2BhqA76xP1%2B9w%2F%2FvH728%2B39xcfcffP4A39ucrrbWEU5qEJAr%2FkDSMx9pvmkzZ3%2FMH%2F86zkPzlJA1lzX%2FGNR2RcRp6A5KSfjxOSTimySZ3n1%2BdJuF4%2BD5MaUKieStJ6TjdoH4nSTyhSZq32rs05e2td6Zfsj%2FsJnEUDx%2FPp9SbJWH6eE5G5E88Pvfpb3Y6iGdjPysXO%2FBDMkzISPsdTmeL8rJ0ouuG4Rim5iE%2F0LCne5qLbYcdAiswbeJ6jpWV5HL%2BS95%2Bebvy%2FrYq1cpL1i%2B9eDSJx6w%2BpuzAxsQGrhFohol1DRNoaY5hGxp1A981jQC71Ou2aqaP05SOtBEfM9hbYikAG8gxHEvDuo00HJiGZgMHaISaVuD4iDrYKVcK%2B0dz%2ByjONnSC4lTe3bbpeouxINmkDb1wL4RS90KoeuHJ9cLY%2FY%2FP4DqIiMsoIvtdeSNmzcULo6%2BPk7zOWZ9gLc%2FTvDvCpv1onqvovFri9f5Lgj%2BfH2%2FCtz9%2BfEBf%2F3y70ObcUe3u0%2FSxmOepz6b9%2FDBO0rt4GI9J9HqZepHwWqP8GYAd3aUjXkTWQS%2FoQ5j%2BU%2Fr3D57l3DLyw8FDfkl28Jgf%2FEfT9DHHHTJLY5a0fOz7OJ7kN5z%2FNF66Go%2BsG4OK%2Fl9qqc0Vo1s5VpFkSNPWKswHjMQPvk1Yf5rXRom3plOa8pf0vTwQoCUpvc3zJ3%2Bb36ze%2Fz5q9%2B%2B%2BfMTa18m%2Fg%2B%2BXpddzReMRTZPHrK9FrKX%2BprUHFc2gyLdsjZ2OuNcQ3%2Fzz9V%2Fz9bubH9hEr3uXFvz6XWtrYRIMue3Fl2HM3dMbLLpCV1Uw5N9E2jRNZl46SyivFBh4wDSphmyTj%2B8UaDYCWDM9GmA9wIDkXxaH%2F%2FGoY%2FJd%2FfEmtFzKxjLNC9i8hgmAGrEMX7Mh9RHBlm4gvdPZdl15DGhYpg40C7usPBagmus7vuY6bmCYrum52H2qThZFuL%2B%2FP79H53HCp07oOA6fQflv4P2MNfDpI%2Bu7D9qYze4o5ROYwol2nGgdup6cOfY%2FbgHD8wwEPc3XMdSw4yH2O3w2ePnIsizPwLoZ7AOLePHnZHQISjL3TUlgG0pi7z15zK46N4rDH%2BVzy8uyI5npSrcVXbVNT6bcdGUqujpluip6t6IrRVeKriSjKzNipy%2F8kP0ac5hm3DBPcpN6SvYTV%2FKVkvpRPPMvE1bejS%2B5imI3e1n9wUf2%2F0HIulnozuY113yPRh70%2BKOD%2BaPXo%2BB8uH4SBTmhhKzl9NmrT7Ik5GBjcMkuuSjODcKE1eZ8Bh1zAGPngjCKStcYPQQuONtN4pD3KpZoXLA%2FbJzugzODZenzo3PdqCXUj61qAlw94veoJtSPrWoCrN8e1p4P6wUsJawcVW4Pas8HpQKyPwxXZynr4bS%2FMGtyrGXvPf5JS7UXZP%2Fx1kKmdwvqZtjCugUDXd6GP8XTMH8Fo9D3M0gvMvQiNr6xE26cpvGInSB5gsdeHu9LZYC%2Fv2ODxvWEZEPpPRvu%2BctkbSnHbAiK47yN8GumLDsfp9AA8aM7kjXF0UM2W5yT%2Byk%2BZwOLz%2BaMtx4v4sUkmf%2BjmqfUcIvGzX8CLazMW1L7DoS9is75s1FhCc5HxsJufb80Flt50l3ZTqzXWFUs2C5QTVLY7po0pTEfIRrY1GIIYmMTchrAGnEdBl7AQZBhIbJMKhqR7JtDgYspJZ6tAUwNDVNeO6bDycIwA9fBlmvtl0NbG%2FASB1pn%2BW2KpwPug3IJ2Kh2mZXNu2MdkBVszjQKBp8Dg9dsOgAXM%2B8n5SeyF0fHfgO0fUrCEeHj%2BRaANkUbgZm9LZiZoIeQtR2Y6ZYFoanATIFZC5hNkXBAtsCtAsiMVSIzmogMiE1kttxE1rH8JQ2RUYqxixyiedh1NQxYBbkOYzM9wDb1oIOo3bHdVRHZ84iseYZfTOeKxGQjsWvKatPfD4uh5aioWEyxmGKxCoth3diJxRyhUaxwlBIexSSaNxSHKQ7biMNKk%2FkLkNhhPCe7%2Brg5IIw9AVE7eKBBsHdH%2FdyZLHdCK86I7qePiuWPaz3JIMA7z%2BWn4EqWNzEJZvA15RdtwDmkt3rHK8SkciZbdHDlTaa8yZQ3mVSy1cJ1bMVQCL7QYVZ88Go21e7pNNX0v0qalbtGsUryyzdQrWArYimVab3KVFf2xqwfbycF6TUpCDSLPzo4HzKinWTPXNV9itO3%2Bdtv0L8gtl5f9GqqYl7eusiVcqBdKFwRDdKlQPU%2BOxogUJbSoHCik1m1%2F%2BkNmpPpNIhOptguWQXwSMqsXfOaNAZAGaf1vVcKDgyGXzDQAuSZrFIwqxQXANYrkGEC4jqMr8WmOamltXXIoWycgsDiJQmj%2BHc2Mv%2BdhMNw3GDe5G%2BOEvbm4F9bGTgX2lzRMTcix%2FZFAUvpjXMBSbyy3avCDCuUArL%2FminlCdjiN%2FxEUtaextlddbC8cxGsS69CmViogk39vGYha%2FAfX6SVaQWJDSty%2B4%2BjU%2FUfV7CiYEU0WFmZAysTnkIVQVBlyZStpLIqa%2B2LVNq95BWpbEUq0DpOTpHFq1qiMUdBioKUk4OU%2Buz30nJK%2B3An2nfNATFlHWJs7qhkt0shnfgp6Zs7KvGDT%2Bw3sNmVV5xgzkt4Y%2BclW9%2BZXvbvu5Rd2ksS8ljKkJtLl3f%2BxBNK7ASrFilkGuXGvzY%2FdlCts8xLsOwBi5%2Fywjhlyy372BLJPnt6g%2FiU3aqKoUd5VSmvKuVVJZX6tOJV1fvEG9bVTbP%2F1NMBvRZaFSsbfwa7zxM%2BWFsIWGQS3g5JSu%2FJ4yZkiXdwz2p00Wpy02p01Vp116pkyxyoGp5QT2xKs1YT4Wq2wudqNbEprcnBrH41bLga1q7e3L2LDV2XBrZYP91q4WbfgghePn9R4mLtYbu%2FVtOKxHUuZyQbONlBED7wcjzhg1YM0fkSRHbY6ItWbunCaZi4xuGFG33ZM6zJMUxoCbMgNkmZu2vglMbU6psoMIAeaEg3XA27AdFsM6CMUyj2dRJ4ruOIBianrfEVSFFFA2WAFAYBu1L2HH3vyt5ZOQj%2Bmhj4Ygt7G8e3tyyBhb2Xn8edogHLOY8XxZdhHt8XiZ1yfHtr7z9eaWdKO1Pa2SG0s%2Fdk5PpkV%2BXs7XiY8fEzNbMoL8QGclm7IVbJZS8jl1n2a4C3k8sGwOhD64TksryRi6aUIcOqrUvADf5%2BhrGqlRVpYjI2lsU%2BLdFEtf%2FIXaYOIYJQ8wnSuShka8TVoQaAD00TuLpJVeQuoYSyOT8UHPACAtlhvs9O2vvNcPalkW2qbS33hjwr7wy53Cjyib0hs6O6oHYAoWxjD7jC3iSkUNbssqbbdgUXkFWbRue%2FOr9q2fS3daXT7SqWoKrecSaua5zhSIIe7cUXbew8adc4hZBKqZNXqTtg1zU7%2FkB4vjgtr7z3eUYrkfT3re6Z7RH2lbqn1D2l7m23Q5JTjzqC9GNQ94pRXnjEPiU0U%2Brecap7OQYII%2B51jXgnLe6Ze1vaKo7jWtFg1utxWGTHtZ1kNFz3hMd2uc2uzY9RJb%2FAspspi8WvvfiijWpH9e0usFfd%2Fl0KlVed0uqUV534stu3SRQT%2F5C6W3v8NKW7Kd1N6W5b6W5sNKrpbou9yuXW3VQQPaW7Kd3tMLpbwQHiCG8qplzHqHedknQ2PSDqWSreiEI9hXp7RT3kLJZUSA17lizhRhTsKdiTHfYKEhAG9oref9Kw15WV1dJbses4rKz518EG4UF2n4YFtbKahbpT87J6yspaz4%2Br%2Bc%2FEtbJaRa8QHQraiy%2FaqHZIK2vHX%2FFSWVm7ntWUlVVZWZWVVTIac8pf9HvazuFYdnOwNg76Zh8d1aEapRl6%2B24O9fwmqOQXmOrybSdkpbqi%2BCdMdfsPyiYw1dmK6hTVKaqTk%2Bo6Nai%2BoSRK7w5pUFUR6ZRBVRlU97pmFR%2BF75wly%2FoUiSYqZU5V5tRGc2rBAeKYUwVd3vWSqHdB%2FST2fu7KelczkvgJCaPnO9CR2XDE52P%2FloSbUJ%2Bt3OgEpD7cu9Cd3nbUB6E5YC%2FpdKiv0tTFY78q%2BRl2A%2Fk17Ntl1MBJLPIr5Cnhye8w6po0%2B3bJKEHtvVJwYDBcg4EWIM9klYJZpbgAsF6BDBMQ19EDLLbyKDXe5sxUZp8XYFyJPhEPCridWahxK18qC3XDVt4bWKjtnYlFUAs1LPZqKHgJtkd3qec3gSx%2Bhw6WmqGcJ%2BdE8RhqXxSsd1sFclmo9%2B50qSzUykKtLNQSyJbvxvF9lAGRDi7IlB5evNSVeKnESyVedi1eQqMK19hsXAcsn3xZfNiKjt4SzV8KTpR2eara5SoACWOl7%2FobVVnpL%2FoRmfn8VV9zAuKbBZMRh4qxO51USfdA9Nu%2B%2BEfRr6JfRb87xcGpbyRsNG0kLB%2F9yrI0StGvol9Fv8LT7woPiQO%2Fgq6iPCj8dmXBt%2FdmwRfH8l4E09jA8o52Bg5BLe8GqO%2BT0h7xp57fqOY%2FE9fybsttebeV5X3%2FQW9Etrx3PKspy7uyvCvL%2B8tIkX9P6PiaksRrXeHdGnCbJmxuiej0%2BWuGpotb3ZLJJAq9rEZvEzrh0lJcbJm4BhPV1isCqpFq%2BfgGauT69i%2BaRFk30BvWUZjnZdmT5TCekWp1kdIolUa5o0a5BKwyKL2AQqmAeM8LjPa27fPxLTBywKYyZ7F6%2BXhkTr0mW1prApuv5HekCYEpS3ie9uLLgFH7eoMdR%2BKRSuYshh4lcyqZU8mcklFdXaRkRAAuZt7PzF1yJ53z1SD2ZtxZbPrX84VOtBFRLqcvpWQKo2SaoIeQtZ2SqVsWhOYpKZlIOKlS12sL9ZtCYEonVRaEJitjdwyY0kiVlGLsIodoHnZdDQNWQa5jQo0hr0096CBqd1w1AkiVUgt5JYSooICS8oSFvo%2FXLMfXeBJ6u0JfL2KV2YFhezzdCPgMBXziAd8lwtjWtwO%2Biz63rpwS8LEWLjrxWcdBfIbcxNexqiYN8QEDukAnWPMMnRGfj5DmQC%2FQgOdaCFLDNFiTV8QnEvEt%2BaHgAAV7wsBeV3ZbCPa%2Bd2Hxb262XRpxNzfcsraQPJYswPzwR%2Fnc8lbZUXEvgQy%2BG%2B9k7Tg7g8D%2BDb4vP%2FvnrVXa6X9Rfhnm%2F30R3ClvF130b2VVVVZVZVWVjLk2iU0Tz%2FwbkmbrSXYOS7P%2BOQMyvXNjkvjP1uY8XuJ7XuJbfSORTq0vUSKdnCJdpamLptbhYulIYZ%2FFx7CUxJF7KUnXqCqNWicj0qilJMetRy7ZqkQ%2FSpEUjI75FF5BQvPXLC5OaNNssmc1AiCcPMxZNT%2B%2FMFt%2F5rLzgBL2dsF7mqbZEP95RmflwObzx7RbnX9tZnV2WoG2a9hqBOQVOK6D8QoUV4F4BTbroLkCmVXcXSHiOjavsHUVv1cIuI7JKyzdCrjPBNJR6PuZMl0H0gWptjHp%2FR3r9tcTkg2w92yMrHEqqHEqv2bKsvNJCw1QM6TiczY0%2BGygzRF1kqwSKj7nLVY0Ll2NwmihVTuy2UCmi62BBEVTR2407VhIlAZNDZt9K%2Bmer1nItHiBqOYQC2vE8DHApmHZVscBmgRAU6nBrXVGVwAnGMAtZMRLEkbx7%2BxdfaHDrPzg1WyqUTJNNfjEKpBGEkvyyzexRrdHOVT0tJ6e6sQ6ZkPndojTJMU1K2%2FDJJ5Nsmc2Km7Z6dv89TdwHcTW64tejZbz8rariREN0iV4vc%2BOBgiUEREKx1IFJeXDi9MAUriJpLAuNEnlnVZalFqU%2F%2BRYSsl8SuYTjRbXYociRlGJsZ%2FEPLrP4r3ttujkC13Efnu2dXuOHtqYjGgVP39plxFAb8i3bz8vv3yHxLh9979frehZAFCS1UkZp5YOkPBJbqlQyZyDbvJZX3%2BCyFZEKrFgpm6wtO1Vmim83SswIwjLPNkCJOCYlrIrhlEMoxjmpU2VlYmwMqEpehGVXnqfeJO9utk5Qsp1Ssa%2B%2B%2Fj8%2BChkEt4OSUrvyUYhnyFoj72nfPJexifPwBYb9rcyE%2FctiODlCfnklZu6aIQLcU2vayDcJrnOFIRwW0cLCShXonlr70DnmygwgB5oSDdcDbsBYbwTUAa9FPs6CTy3a4OwAJQrNQMWOFHCghdgvzVjgGhfiy%2FJfx0EyVsExul2tcer%2FAviUJH3cs8dBZRiAaUKvSdn6D1UC2EN9QbDr3xrOxb%2BfQokJQJJFXlPNpAsccmCL0pEIA5Uwo4bzjFAZb5H865A%2BW4c30dZZBLWBMiUdsyV3SmWsyFvl9S%2FJeFmhAkVYYpHmLh3oTu97QiT4eWAvaTTIcxKWxeNNeuaJRuUj2EdcT5eKNaUijWVaV6Z5kWj6RzI6mAlpEpbDHoKqDsD6qsZSfyEhFHXEu0LorSuUFqhtELp7mXb%2Bsrno4FpXcG0gmkF0wqmu4HpJVSJidG6wuiOMbofkZlPt4lxuXSz4OTU6mEhF323h09X9K3oW9H3LsvLoH6s9C1LCHtF34q%2BFX0LT99zFptjlZj8LeieFy%2FJ339P6PiakiQLFL8Tgl%2FThI3qEZ12sD%2Fj4la3ZLJYq3ib0AmnmjjZbBUa3CF0kyLhva9Cs%2BzXAG9HwgNg9KF1QiS8vgOIhsewgNxj8ymWJZSUgmMFxwqOxYXjJWCVQUkgKBY05txLQvF7MnJ9sisQf55RPo2JqStH%2BU%2FbhKPVNuiKo%2BXk6LyViwbLul1Vkh27QUhuCFVWpInKyrJsha5YubT%2BztQhRBBqPkE6pyZbI64ONQB8aJrA1U2q1t8JRZJzKsnoQkx51VAk2S1JfptEMd%2BsQH6UNBVKKpRUKNmhW0ItlsOxoKSpUFKhpELJg6DkHC%2FEZEnzpFiS12Ecp6WsV7xFfeDtmyX%2BHw%3D%3D%3C%2Fdiagram%3E%3C%2Fmxfile%3E)

## Architecture

The solution implements a fully serverless, multi-region architecture with automatic failover capabilities:

```
┌─────────────────────────────────────────────────────────┐
│                    Global CDN Layer                     │
│  CloudFront Distribution (Origin Group Failover)        │
│  ├─ Primary Origin: S3 Frontend (us-west-2)            │
│  └─ Failover Origin: S3 Frontend (us-east-1)           │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│                  Primary Region (us-west-2)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │API Gateway  │  │Lambda       │  │Bedrock      │     │
│  │(REST API)   │─▶│Functions    │─▶│Knowledge    │     │
│  │             │  │(5 functions)│  │Base         │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                              │                         │
│                              ▼                         │
│                    ┌─────────────┐                     │
│                    │S3 Documents │                     │
│                    │(Versioned)  │                     │
│                    └─────────────┘                     │
└─────────────────────────────────────────────────────────┘
                              │
                              ▼ (Cross-Region Replication)
┌─────────────────────────────────────────────────────────┐
│                 Failover Region (us-east-1)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │API Gateway  │  │Lambda       │  │Bedrock      │     │
│  │(Standby)    │  │Functions    │  │Knowledge    │     │
│  │             │  │(Standby)    │  │Base         │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### Core Components

#### Frontend Layer
- **CloudFront Distribution**: Global CDN with origin failover (automatic < 1 second RTO)
- **S3 Frontend Buckets**: Private buckets with Origin Access Control (OAC) in both regions
- **React Application**: Modern web interface with drag-and-drop file uploads and real-time chat

#### API Layer
- **API Gateway**: RESTful API with CORS, throttling, and usage plans
- **Endpoints**:
  - `POST /docs`: Submit queries to the chatbot
  - `POST /upload`: Generate pre-signed URLs for file uploads
  - `GET /ingestion-status`: Check document processing status
  - `GET /health`: Health check for monitoring and failover

#### Compute Layer (5 Lambda Functions)
- **Query Lambda**: Core RAG orchestration (Retrieve + Generate with Claude 3 Sonnet)
- **Upload Lambda**: Generate pre-signed S3 URLs for direct file uploads
- **Ingestion Lambda**: Triggered by S3 events to start Bedrock ingestion jobs
- **Status Lambda**: Poll Bedrock for ingestion job status
- **Health Lambda**: Test Bedrock connectivity for monitoring

#### AI/ML Services
- **Bedrock Knowledge Base**: Automated document chunking (500 tokens, 20% overlap)
- **Titan Embeddings**: Vectorization of document chunks
- **Claude 3 Sonnet**: Answer generation with context from retrieved chunks
- **Bedrock Guardrails**: Content filtering for harmful content

#### Storage
- **S3 Documents Bucket**: Versioned, encrypted storage with lifecycle policies
- **OpenSearch Serverless**: Vector store for semantic search (managed by Bedrock)

#### Monitoring & Observability
- **CloudWatch Dashboard**: Real-time metrics for API Gateway, Lambda, and Bedrock
- **CloudWatch Alarms**: Automated alerts for errors and system health
- **X-Ray Tracing**: Distributed tracing across all services
- **SNS Notifications**: Alert delivery for operational teams

## Security Architecture

### Data Protection
- **Encryption at Rest**: All S3 buckets use S3-managed encryption
- **Encryption in Transit**: HTTPS enforcement via CloudFront and API Gateway
- **Versioning**: S3 versioning enabled for point-in-time recovery
- **Access Controls**: Private S3 buckets with OAC, no public access

### Content Safety
- **Bedrock Guardrails**: Multi-category content filtering (sexual, violence, hate, insults)
- **Input Validation**: API Gateway request validation and throttling
- **Least Privilege**: IAM roles with minimal required permissions

### Network Security
- **Private S3 Origins**: No direct S3 access, only via CloudFront OAC
- **CORS Configuration**: Restricted cross-origin requests
- **VPC Integration**: Ready for VPC deployment if required

## Disaster Recovery & Business Continuity

### Frontend Failover (Automatic)
- **Method**: CloudFront origin group with automatic failover
- **Detection**: Instant (5xx errors from primary S3 bucket)
- **RTO**: < 1 second (automatic)
- **User Impact**: None (same CloudFront URL)

### Backend API Failover (Manual)
- **Method**: Runtime configuration via config.json update
- **Detection**: Manual monitoring or custom alerting
- **RTO**: Manual (minutes to hours depending on response time)
- **Process**: Update config.json in both S3 frontend buckets, invalidate CloudFront cache

### Data Synchronization
- **Method**: S3 Cross-Region Replication (CRR) - manual setup required
- **RPO**: ~15 minutes (automatic background sync)
- **Scope**: Documents bucket from us-west-2 → us-east-1

## Performance & Scalability

### Auto-scaling Capabilities
- **Lambda**: Automatic scaling based on request volume
- **API Gateway**: Handles up to 10,000 requests per second
- **CloudFront**: Global edge caching for static content
- **Bedrock**: Managed service with automatic scaling

### Performance Optimizations
- **Edge Caching**: Static assets cached at 450+ CloudFront edge locations
- **Connection Pooling**: Lambda functions optimized for AWS service connections
- **Chunking Strategy**: 500-token chunks with 20% overlap for optimal retrieval

## Cost Analysis

### Monthly Cost Estimates (US East/West)
Based on moderate usage (1,000 queries/month, 10GB documents):

| Service | Primary Region | Failover Region | Monthly Cost |
|---------|---------------|-----------------|--------------|
| Lambda (Compute) | $2-5 | $1-3 | $3-8 |
| API Gateway | $1-3 | $0.50-1.50 | $1.50-4.50 |
| S3 Storage | $0.25-0.50 | $0.25-0.50 | $0.50-1.00 |
| CloudFront | $1-2 | - | $1-2 |
| Bedrock (Titan) | $0.50-1.50 | $0.50-1.50 | $1-3 |
| Bedrock (Claude) | $5-15 | $5-15 | $10-30 |
| **Total** | | | **$17-49** |

*Note: Failover region costs only apply during active failover scenarios.*

### Cost Optimization Recommendations
- Enable S3 lifecycle policies for document retention
- Monitor Bedrock usage and optimize chunk sizes
- Use CloudWatch to track and optimize Lambda cold starts
- Consider Reserved Capacity for predictable workloads

## Deployment

### Multi-Region Deployment

```bash
# Navigate to backend directory
cd backend

# Bootstrap both regions (one-time setup)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
cdk bootstrap aws://$ACCOUNT/us-west-2
cdk bootstrap aws://$ACCOUNT/us-east-1

# Deploy to both regions
cdk deploy --all
```

This creates:
- `BackendStack-Primary` in us-west-2 with CloudFront distribution
- `BackendStack-Failover` in us-east-1 with standby resources

### Single-Region Deployment (Development)

```bash
# Set target region
export AWS_DEFAULT_REGION=us-west-2

# Bootstrap (one-time)
cdk bootstrap aws://$ACCOUNT/us-west-2

# Deploy
cdk deploy BackendStack-Primary
```

## Usage

### Initial Setup
1. Navigate to the CloudFront URL provided in deployment outputs
2. Upload documents using the file upload interface
3. Wait for ingestion status to show "Complete"
4. Begin querying your documents

### API Usage Examples

#### Submit a Query
```bash
curl -X POST https://your-api-gateway-url/prod/docs \
  -H "Content-Type: application/json" \
  -d '{"query": "What are the key features of this product?"}'
```

#### Check Ingestion Status
```bash
curl https://your-api-gateway-url/prod/ingestion-status
```

#### Health Check
```bash
curl https://your-api-gateway-url/prod/health
```

## Monitoring & Operations

### CloudWatch Dashboard
Access the dashboard named `contextual-chatbot-metrics-{region}` to monitor:
- API Gateway request counts and error rates
- Lambda function performance and errors
- Bedrock service health
- Dead Letter Queue message counts

### Key Metrics to Monitor
- **API Gateway**: 4xx/5xx error rates, request latency
- **Lambda**: Error rates, duration, cold starts
- **Bedrock**: Retrieval latency, generation latency
- **S3**: Request counts, error rates

### Alerting
Configured alarms trigger SNS notifications for:
- Query Lambda errors (>5 in 5 minutes)
- Ingestion Lambda errors (>3 in 5 minutes)
- Dead Letter Queue messages (any messages)

## Troubleshooting

### Common Issues

#### Deployment Failures
- **Bedrock Model Access**: Ensure model access is enabled in both regions
- **Docker Issues**: Verify Docker Desktop is running
- **Permissions**: Check IAM permissions for CDK deployment

#### Runtime Issues
- **"Model not accessible"**: Verify Bedrock model access in target region
- **Upload failures**: Check S3 bucket permissions and CORS configuration
- **Query timeouts**: Monitor Lambda duration and Bedrock service limits

### Debugging Commands
```bash
# Check Lambda logs
aws logs tail /aws/lambda/query-bedrock-llm-{region} --follow

# Test Bedrock connectivity
aws bedrock list-foundation-models --region us-west-2

# Verify S3 bucket access
aws s3 ls s3://your-docs-bucket-name
```

## Additional Resources

- [Amazon Bedrock Developer Guide](https://docs.aws.amazon.com/bedrock/latest/userguide/)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Amazon S3 Developer Guide](https://docs.aws.amazon.com/s3/latest/userguide/)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/latest/guide/)
- [Amazon Bedrock API Reference](https://docs.aws.amazon.com/bedrock/latest/APIReference/)

## Support & Maintenance

### Operational Procedures
- Monitor CloudWatch dashboards daily
- Review SNS alerts promptly
- Test failover procedures quarterly
- Update Bedrock models as new versions become available

### Maintenance Windows
- **Scheduled**: Monthly during maintenance windows
- **Emergency**: As needed for critical issues
- **Updates**: Follow AWS service announcements for new features

### Support Contacts
- **AWS Support**: Use AWS Support Center for service issues
- **Documentation**: Refer to AWS documentation for service-specific guidance
- **Community**: AWS re:Post for community support and best practices



# AWS Architectural Diagram - Official Style Guide

## AWS Icon Library & Resources

### Official AWS Icons
- **AWS Architecture Icons**: https://aws.amazon.com/architecture/icons/
- **Download**: AWS-Architecture_Icons_AWSServicelight-bg.zip
- **Format**: SVG icons with light backgrounds
- **Usage**: Free for AWS diagrams (check license for commercial use)

### Recommended Tools for AWS-Style Diagrams

1. **Lucidchart** (Recommended)
   - Has built-in AWS icon library
   - Official AWS template available
   - Professional output quality
   - Export to PNG/PDF for presentations

2. **Draw.io (diagrams.net)**
   - Free tool with AWS icon library
   - Good for quick diagrams
   - Can import AWS icons

3. **Visio**
   - Microsoft's professional diagramming tool
   - AWS stencil available
   - Enterprise-standard output

## AWS Diagram Style Guidelines

### Color Scheme
- **AWS Services**: Light blue backgrounds (#F7F7F7 or white)
- **User/External**: Light green (#D5E8D4)
- **Data Flow**: Blue arrows (#1F77B4)
- **Failover/Standby**: Light gray (#E1E1E1)
- **Monitoring**: Light yellow (#FFF2CC)

### Icon Specifications
- **Size**: 64x64 pixels standard
- **Style**: Rounded rectangles with AWS service icons
- **Labels**: Service name below icon
- **Connections**: Clean lines with arrowheads

## Complete AWS-Style Architecture Diagram

### Diagram Title
**"AWS Contextual Chatbot with Amazon Bedrock Knowledge Bases - Architecture Overview"**

### Layout Structure (Recommended)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              USER                                       │
└─────────────────────┬───────────────────────────────────────────────────┘
                      │ HTTPS
┌─────────────────────▼───────────────────────────────────────────────────┐
│                    CLOUDFRONT                                         │
│              Global CDN Distribution                                   │
│            (Origin Group Failover)                                     │
└─────┬─────────────────────────────┬─────────────────────────────────────┘
      │ Primary Origin              │ Failover Origin
      │ (us-west-2)                 │ (us-east-1)
┌─────▼─────┐                ┌─────▼─────┐
│ S3 BUCKET │                │ S3 BUCKET │
│ Frontend  │                │ Frontend  │
│ Primary   │                │ Failover  │
└───────────┘                └───────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                      PRIMARY REGION (us-west-2)                        │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ API GATEWAY │    │   LAMBDA    │    │   LAMBDA    │                │
│  │             │───▶│   QUERY     │    │   UPLOAD    │                │
│  │ REST API    │    │             │    │             │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│       │                     │                   │                      │
│       │                     ▼                   ▼                      │
│       │              ┌─────────────┐    ┌─────────────┐                │
│       │              │   LAMBDA    │    │   LAMBDA    │                │
│       │              │  INGEST     │    │   STATUS    │                │
│       │              │             │    │             │                │
│       │              └─────────────┘    └─────────────┘                │
│       │                     │                   │                      │
│       │                     ▼                   │                      │
│       │              ┌─────────────┐            │                      │
│       │              │   LAMBDA    │            │                      │
│       │              │   HEALTH    │            │                      │
│       │              │             │            │                      │
│       │              └─────────────┘            │                      │
│       │                     │                   │                      │
│       ▼                     ▼                   ▼                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │   BEDROCK   │    │   BEDROCK   │    │   BEDROCK   │                │
│  │ KNOWLEDGE   │    │  GUARDRAILS │    │   CLAUDE    │                │
│  │    BASE     │    │             │    │   SONNET    │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│       │                                                               │
│       ▼                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ OPENSEARCH  │    │ S3 BUCKET   │    │ CLOUDWATCH  │                │
│  │ SERVERLESS  │    │ DOCUMENTS   │    │ DASHBOARD   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                              │                   │                      │
│                              ▼                   ▼                      │
│                       ┌─────────────┐    ┌─────────────┐                │
│                       │ SNS TOPIC   │    │ DEAD LETTER │                │
│                       │  ALERTS     │    │    QUEUE    │                │
│                       └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ Cross-Region Replication
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    FAILOVER REGION (us-east-1)                         │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │ API GATEWAY │    │   LAMBDA    │    │   LAMBDA    │                │
│  │  (STANDBY)  │    │   QUERY     │    │   UPLOAD    │                │
│  │             │    │ (STANDBY)   │    │ (STANDBY)   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │   BEDROCK   │    │   BEDROCK   │    │   BEDROCK   │                │
│  │ KNOWLEDGE   │    │  GUARDRAILS │    │   CLAUDE    │                │
│  │    BASE     │    │             │    │   SONNET    │                │
│  │ (STANDBY)   │    │ (STANDBY)   │    │ (STANDBY)   │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐                                   │
│  │ OPENSEARCH  │    │ S3 BUCKET   │                                   │
│  │ SERVERLESS  │    │ DOCUMENTS   │                                   │
│  │ (STANDBY)   │    │ (REPLICA)   │                                   │
│  └─────────────┘    └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## AWS Icons to Use

### User & External
- **User**: `user-icon.svg`
- **Internet**: `internet-gateway.svg`

### Frontend & CDN
- **CloudFront**: `cloudfront.svg`
- **S3**: `amazon-s3.svg`

### Compute
- **API Gateway**: `api-gateway.svg`
- **Lambda**: `aws-lambda.svg`

### AI/ML Services
- **Bedrock**: `amazon-bedrock.svg`
- **Bedrock Knowledge Base**: `bedrock-knowledge-base.svg`
- **Bedrock Guardrails**: `bedrock-guardrails.svg`

### Storage
- **S3**: `amazon-s3.svg`
- **OpenSearch**: `amazon-opensearch.svg`

### Monitoring
- **CloudWatch**: `amazon-cloudwatch.svg`
- **SNS**: `amazon-sns.svg`
- **SQS**: `amazon-sqs.svg`

## Step-by-Step Creation Instructions

### Using Lucidchart (Recommended)

1. **Create New Diagram**
   - Go to Lucidchart.com
   - Select "AWS Architecture" template
   - Choose blank canvas

2. **Add AWS Icons**
   - Click "Shapes" panel
   - Search for "AWS" to access icon library
   - Drag icons onto canvas

3. **Layout Components**
   - Arrange services according to the layout above
   - Use alignment tools for clean positioning
   - Group related services with rectangles

4. **Add Connections**
   - Use connector tool to draw arrows
   - Label connections where needed
   - Use different line styles for different flows

5. **Apply Styling**
   - Use AWS color scheme
   - Add service labels below icons
   - Apply consistent font (Arial or similar)

6. **Export**
   - Export as PNG (high resolution)
   - Or PDF for presentations

### Using Draw.io

1. **Setup**
   - Go to diagrams.net
   - Create new diagram
   - Add AWS icon library

2. **Import Icons**
   - Download AWS icons from official site
   - Import as custom shapes
   - Or use built-in AWS shapes

3. **Build Diagram**
   - Follow same layout structure
   - Use grid for alignment
   - Group components logically

## Professional Presentation Tips

### For Client Presentations
- **High Resolution**: Export at 300 DPI minimum
- **Clean Layout**: Plenty of white space
- **Consistent Styling**: Same icon sizes and colors
- **Clear Labels**: Service names and purposes
- **Flow Indicators**: Clear data flow arrows

### Color Coding Recommendations
- **Primary Services**: Light blue (#E6F3FF)
- **User/External**: Light green (#E6F7E6)
- **Failover/Standby**: Light gray (#F5F5F5)
- **Monitoring**: Light yellow (#FFFACD)
- **Data Flow**: Blue arrows (#0066CC)

## Alternative: Use AWS Architecture Center

AWS provides official reference architectures:
- **AWS Architecture Center**: https://aws.amazon.com/architecture/
- **Search for**: "RAG", "Bedrock", "Serverless"
- **Use as**: Starting templates for your diagram

## Final Output Specifications

### For README/Documentation
- **Format**: PNG or SVG
- **Size**: 1200x800 pixels minimum
- **Resolution**: 300 DPI for print quality

### For Client Presentations
- **Format**: PNG or PDF
- **Size**: 1920x1080 for full HD
- **Background**: White or light gray
- **Text**: Black, Arial, 12pt minimum


