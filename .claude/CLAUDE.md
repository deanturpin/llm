# LLM Guide Project Instructions

This is a **public educational repository** - never commit sensitive information.

## Content Guidelines

- **Educational focus**: Provide comprehensive, accurate guides for self-hosting LLM applications
- **Security conscious**: Always include security best practices, but never include actual credentials, API keys, or passwords
- **Provider neutral**: Present multiple options for VPS providers, favouring independent providers over big tech
- **Privacy first**: Emphasise data sovereignty and local inference throughout all guides
- **Practical**: Include working code examples, Docker configurations and deployment scripts
- **Cost transparent**: Always provide realistic cost breakdowns for each use case

## What NEVER to Commit

- API keys, passwords or secrets (even example ones that look real)
- Personal data or user information
- Actual VPS IP addresses or domain names
- Real encryption keys or salts
- Private configuration files from implementations

## Documentation Standards

- Use British English throughout
- Code examples must be complete and runnable
- Include cost analysis for each architecture
- Always provide migration paths between providers
- Link to official documentation for external tools
- Explain the "why" not just the "how"

## Structure

- `/docs` - General guides (infrastructure, security, privacy, etc.)
- `/use-cases` - Specific application guides (personal assistant, coding partner, etc.)
- `/implementations` - Reference implementations with code

## Security Content

When writing security guides:
- Provide working examples of encryption, authentication and hardening
- Never include actual secrets (use placeholders like `<your-password-here>`)
- Explain threat models and defence strategies
- Include both "minimum viable security" and "hardened" approaches
- Reference established standards (OWASP, CIS, etc.)

## Target Audience

- Technical users comfortable with command line and Docker
- Privacy-conscious individuals wanting to self-host
- Those seeking independence from big tech infrastructure
- Developers building similar systems

## Quality Standards

- All code must be tested and working
- Links must be valid and to authoritative sources
- Cost estimates must be current and realistic
- Security advice must follow current best practices
- Performance claims must be accurate

## Maintenance

- Keep VPS provider pricing current
- Update model recommendations as new versions release
- Ensure Docker images use specific version tags where stability matters
- Test deployment scripts regularly
- Update security guidance as threats evolve
