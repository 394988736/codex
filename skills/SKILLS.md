---
name: skills-list
description: Directory of all available skills for Codex. Use this to discover available skills and their purposes.
---

# Available Skills

This directory contains specialized skills that extend Codex's capabilities with domain-specific knowledge and workflows.

## Skills

### 1. skill-creator
**Description**: Guide for creating effective skills.  
**Use when**: You want to create a new skill or update an existing skill that extends Codex's capabilities with specialized knowledge, workflows, or tool integrations.

**Key features**:
- Step-by-step skill creation process
- Best practices for skill design
- Guidance on bundling scripts, references, and assets
- Progressive disclosure design principles

### 2. codex-architecture
**Description**: Guide to Codex Rust architecture and component relationships.
**Use when**: Understanding codebase structure, module dependencies, and component interactions.

## How to Use Skills

Skills are automatically loaded when their triggers match the user's request. Each skill contains:

- **SKILL.md**: Main instructions and workflow guidance
- **scripts/**: Reusable executable code
- **references/**: Detailed documentation and schemas
- **assets/**: Templates and files for output

## Creating New Skills

To create a new skill, use the `skill-creator` skill:

```
Please help me create a new skill for [your domain/task]
```

The skill-creator will guide you through:
1. Understanding the skill requirements
2. Planning reusable contents
3. Initializing the skill structure
4. Editing and packaging the skill

## Skill Development Workflow

1. **Plan**: Identify reusable workflows and resources
2. **Initialize**: Use `scripts/init_skill.py <skill-name>`
3. **Implement**: Add scripts, references, and assets
4. **Test**: Validate the skill works as expected
5. **Package**: Use `scripts/package_skill.py <path-to-skill>`
6. **Iterate**: Refine based on usage feedback