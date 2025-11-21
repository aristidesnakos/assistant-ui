# Spec-Driven Development (SDD) Guide

## Overview

This document establishes guidelines for implementing Spec-Driven Development (SDD) within the assistant-ui project. SDD shifts from "code is the source of truth" to "intent is the source of truth" by making specifications executable artifacts that guide development.

## Core Philosophy

**Intent is the source of truth** - Specifications become living, executable documents that capture user needs and business requirements, then guide technical implementation.

## SDD Workflow (4 Phases)

### Phase 1: Specify
**Purpose**: Define what to build and why

**Deliverable**: Product Requirements Document (PRD)

**Focus Areas**:
- High-level project goals
- User experience requirements
- Business value and outcomes
- Success criteria
- Avoid technical implementation details

### Phase 2: Plan
**Purpose**: Determine how to build it

**Deliverable**: Technical Charter

**Focus Areas**:
- Technical architecture and constraints
- Integration with existing assistant-ui patterns
- Organizational standards compliance
- Multiple implementation approaches
- Risk assessment and mitigation

### Phase 3: Tasks
**Purpose**: Break down into executable work units

**Deliverable**: Task Breakdown Structure

**Focus Areas**:
- Granular, testable tasks
- Independent, focused work items
- Clear acceptance criteria
- Alignment with specification goals

### Phase 4: Implement
**Purpose**: Execute the plan

**Deliverable**: Working Code + Documentation

**Focus Areas**:
- AI coding agents execute tasks
- Developer verification and refinement
- Continuous alignment with specification

---

## Product Requirements Document (PRD) Template

### 1. Executive Summary
- **Problem Statement**: What user problem are we solving?
- **Solution Overview**: High-level description of the proposed solution
- **Success Metrics**: How will we measure success?
- **Timeline**: Expected development phases and milestones

### 2. User Stories & Journeys
- **Primary Users**: Who will use this feature?
- **User Scenarios**: Step-by-step user workflows
- **Pain Points**: Current friction in user experience
- **Desired Outcomes**: What users should accomplish

### 3. Functional Requirements
- **Core Features**: Essential functionality
- **Feature Priorities**: Must-have vs nice-to-have
- **User Interface Requirements**: UX/UI specifications
- **Integration Points**: How it connects to existing assistant-ui components

### 4. Non-Functional Requirements
- **Performance**: Response times, throughput requirements
- **Accessibility**: a11y compliance requirements
- **Browser Compatibility**: Supported environments
- **API Compatibility**: Backward compatibility needs

### 5. Constraints & Assumptions
- **Technical Constraints**: Existing architecture limitations
- **Business Constraints**: Budget, timeline, resource limitations
- **Assumptions**: What we're assuming to be true

### 6. Success Criteria
- **User Acceptance Criteria**: When is the feature "done"?
- **Business Metrics**: KPIs and measurement plans
- **Technical Metrics**: Performance benchmarks

---

## Technical Charter Template

### 1. Architecture Overview
- **System Design**: High-level technical approach
- **Component Architecture**: How it fits within assistant-ui's structure
- **Data Flow**: Information flow through the system
- **Integration Strategy**: Connections to existing packages

### 2. Technical Requirements
- **Technology Stack**: Languages, frameworks, libraries
- **Dependencies**: New dependencies and their justification
- **API Design**: Interface specifications
- **Data Models**: Schema and type definitions

### 3. Implementation Strategy
- **Development Phases**: Technical implementation stages
- **Package Structure**: Which packages will be affected/created
- **Migration Path**: How to handle breaking changes
- **Testing Strategy**: Unit, integration, and e2e testing approaches

### 4. Technical Constraints
- **Performance Requirements**: Specific technical benchmarks
- **Security Considerations**: Security requirements and implementations
- **Scalability Needs**: How the solution should scale
- **Browser Support**: Technical compatibility matrix

### 5. Risk Assessment
- **Technical Risks**: Potential implementation challenges
- **Dependency Risks**: Third-party library concerns
- **Integration Risks**: Compatibility with existing systems
- **Mitigation Strategies**: How to address identified risks

### 6. Quality Assurance
- **Code Quality Standards**: ESLint, TypeScript strictness
- **Testing Requirements**: Coverage expectations
- **Documentation Requirements**: What needs to be documented
- **Review Process**: Code review and approval workflows

---

## Assistant-UI Specific Guidelines

### Integration with Existing Architecture
- **Runtime Pattern**: Consider how new features integrate with the runtime abstraction
- **Primitive Components**: Follow Radix UI patterns for composable primitives
- **Transport Protocol**: Leverage assistant-stream for consistent data flow
- **Package Organization**: Respect the monorepo structure and package boundaries

### Development Standards
- **TypeScript**: Strict type checking and comprehensive type definitions
- **Testing**: Vitest for unit testing, integration tests for runtime components
- **Documentation**: Update both code comments and user-facing docs
- **Performance**: Consider streaming, accessibility, and real-time update requirements

### Example Integration Points
- **New Runtime**: Extend the runtime pattern for additional backend support
- **UI Components**: Create new primitives following existing component patterns
- **Tool Integration**: Leverage tool execution framework for new capabilities
- **Cloud Features**: Consider Assistant Cloud integration for persistence/analytics

---

## Best Practices

### Specification Writing
1. **User-Centric Language**: Write from the user's perspective
2. **Clear Success Criteria**: Define measurable outcomes
3. **Iterative Refinement**: Treat specs as living documents
4. **Essential Business Logic**: Capture what matters, avoid implementation details

### Technical Planning
1. **Multiple Approaches**: Consider alternative implementation strategies
2. **Constraint Integration**: Factor in organizational and technical constraints
3. **Future-Proofing**: Consider long-term maintenance and evolution
4. **Performance First**: Design with performance and accessibility in mind

### Task Management
1. **Atomic Tasks**: Each task should be independently implementable
2. **Clear Acceptance**: Every task needs verifiable completion criteria
3. **Specification Alignment**: Ensure tasks ladder up to specification goals
4. **Incremental Value**: Tasks should deliver meaningful progress

---

## Tools & Resources

### Recommended Tools
- **Spec Kit**: Open source toolkit from GitHub for SDD workflows
- **Installation**: `uvx --from git+https://github.com/github/spec-kit.git specify init <PROJECT_NAME>`
- **AI Assistants**: Compatible with GitHub Copilot, Claude Code, Gemini CLI

### Documentation Templates
- Use this SDD guide as the foundation for all feature development
- Store PRDs and Technical Charters in `/documentation/specifications/`
- Link specifications to implementation tasks in GitHub issues
- Update specifications as requirements evolve during development

---

## Implementation Checklist

### Before Starting Development
- [ ] PRD approved by stakeholders
- [ ] Technical Charter reviewed by engineering team
- [ ] Task breakdown completed and estimated
- [ ] Risk mitigation strategies defined
- [ ] Success criteria and metrics established

### During Development
- [ ] Regular specification review and updates
- [ ] Task completion tracked against specifications
- [ ] Continuous integration with existing assistant-ui patterns
- [ ] Documentation updated in parallel with implementation

### After Development
- [ ] Success criteria validation
- [ ] Documentation review and update
- [ ] Lessons learned captured for future specifications
- [ ] Specification archived with implementation details