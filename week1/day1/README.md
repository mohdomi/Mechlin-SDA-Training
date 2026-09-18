# Day 1: SDLC & GitHub Mastery

## 🎯 Learning Objectives

- Understand Software Development Life Cycle (SDLC) and Agile methodologies
- Master Git branching strategies and workflow
- Learn pull request best practices and code review process
- Handle merge conflicts and collaborative development
- Set up proper repository structure and documentation

## 📚 Theory & Concepts

### Software Development Life Cycle (SDLC)
- **Planning**: Requirements gathering, feasibility study
- **Analysis**: System analysis, requirement analysis
- **Design**: System design, database design, UI/UX design
- **Implementation**: Coding, unit testing
- **Testing**: Integration testing, system testing, user acceptance testing
- **Deployment**: Production deployment, monitoring
- **Maintenance**: Bug fixes, updates, enhancements

### Agile Methodology
- **Sprint Planning**: 2-week sprints with defined goals
- **Daily Standups**: Progress updates and blockers
- **Sprint Review**: Demo completed features
- **Retrospective**: Process improvement discussions

### Git Workflow
- **Feature Branches**: Create branches for each feature
- **Pull Requests**: Code review before merging
- **Merge Strategies**: Squash, merge, or rebase
- **Conflict Resolution**: Handle merge conflicts professionally

## 🛠️ Hands-on Tasks

### Task 1: Repository Setup
Create a new repository following these steps:

```bash
# 1. Create new repository on GitHub
# 2. Clone the repository
git clone https://github.com/YOUR_USERNAME/sda-training.git
cd sda-training

# 3. Set up initial structure
mkdir -p week1/day1/{code,docs,screenshots}
touch week1/day1/README.md
```

### Task 2: Branching Strategy
Implement a proper branching strategy:

```bash
# Create main branches
git checkout -b develop
git checkout -b feature/day1-setup
git checkout -b feature/day1-documentation

# Push branches to remote
git push -u origin develop
git push -u origin feature/day1-setup
git push -u origin feature/day1-documentation
```

### Task 3: Initial Project Structure
Create the following structure:

```
sda-training/
├── .github/
│   ├── workflows/
│   ├── ISSUE_TEMPLATE/
│   └── PULL_REQUEST_TEMPLATE.md
├── docs/
│   ├── architecture/
│   ├── api/
│   └── deployment/
├── week1/
│   └── day1/
│       ├── code/
│       ├── docs/
│       └── screenshots/
└── README.md
```

### Task 4: GitHub Templates
Create pull request and issue templates:

#### `.github/PULL_REQUEST_TEMPLATE.md`
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Screenshots
(if applicable)

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No breaking changes
```

#### `.github/ISSUE_TEMPLATE/bug_report.md`
```markdown
## Bug Description
Clear description of the bug

## Steps to Reproduce
1. Go to '...'
2. Click on '....'
3. See error

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Environment
- OS: [e.g. Windows 10]
- Browser: [e.g. Chrome 91]
- Version: [e.g. 1.0.0]
```

### Task 5: Simulate Collaboration
Practice collaborative development:

```bash
# 1. Create a conflict scenario
git checkout feature/day1-setup
echo "# Day 1 Setup" > week1/day1/README.md
git add week1/day1/README.md
git commit -m "Add day 1 setup documentation"

# 2. Switch to another branch and modify the same file
git checkout feature/day1-documentation
echo "# Day 1 Documentation" > week1/day1/README.md
git add week1/day1/README.md
git commit -m "Add day 1 documentation"

# 3. Try to merge and resolve conflicts
git checkout develop
git merge feature/day1-setup
git merge feature/day1-documentation
# Resolve conflicts in README.md
```

## 📝 Documentation Tasks

### Create Sprint Backlog
Create `week1/day1/docs/sprint-backlog.md`:

```markdown
# Sprint 1 Backlog - Week 1

## Sprint Goal
Establish development workflow and create foundation for advanced frontend development.

## User Stories

### Epic 1: Development Environment Setup
- [ ] **US-001**: As a developer, I want to set up Git workflow so that I can collaborate effectively
- [ ] **US-002**: As a developer, I want to create proper repository structure so that the project is organized
- [ ] **US-003**: As a developer, I want to establish branching strategy so that features can be developed independently

### Epic 2: Documentation Standards
- [ ] **US-004**: As a developer, I want to create PR templates so that code reviews are consistent
- [ ] **US-005**: As a developer, I want to document processes so that team members can follow best practices

## Acceptance Criteria
- Repository structure is established
- Branching strategy is implemented
- PR templates are created
- Documentation is comprehensive
- All tasks are committed with descriptive messages
```

### Create Architecture Diagram
Create `week1/day1/docs/architecture.md`:

```markdown
# Week 1 Architecture

## System Overview
This week focuses on establishing the foundation for advanced frontend development.

## Components
- **Repository Structure**: Organized codebase with proper folder hierarchy
- **Git Workflow**: Feature branch strategy with PR reviews
- **Documentation**: Comprehensive guides and templates
- **Development Environment**: VS Code setup with extensions

## Data Flow
1. Developer creates feature branch
2. Implements feature with tests
3. Creates pull request
4. Code review process
5. Merge to develop branch
6. Deploy to staging environment

## Technology Stack
- Git for version control
- GitHub for collaboration
- Markdown for documentation
- VS Code for development
```

## 🧪 Testing & Validation

### Git Workflow Validation
```bash
# Verify branch structure
git branch -a

# Check commit history
git log --oneline --graph

# Validate merge resolution
git status
```

### Documentation Review
- [x] All templates are created
- [x] Documentation is comprehensive
- [ ] Architecture diagrams are clear (prose doc present; no diagram file yet)
- [x] Sprint backlog is detailed

## 📊 Success Criteria

By the end of Day 1, you should have:

✅ **Repository Structure**: Properly organized codebase  
✅ **Git Workflow**: Feature branch strategy implemented  
✅ **Templates**: PR and issue templates created  
✅ **Documentation**: Comprehensive guides and architecture  
✅ **Collaboration**: Conflict resolution skills demonstrated  

## ✅ Completion Status

All Day 1 deliverables are complete and tested:
- `code/setup.js`, `code/.gitkeep` — environment setup script
- `docs/sprint-backlog.md` — US-001–US-005 completed
- `docs/architecture.md` — system overview, components, data flow, stack
- `.github/PULL_REQUEST_TEMPLATE.md`, `.github/ISSUE_TEMPLATE/bug_report.md`
- Known gaps: no `workflows/` CI file yet, no `screenshots/` captured

## 🔄 Next Steps

1. **Commit your work**: `git add . && git commit -m "Complete Day 1: SDLC & GitHub Mastery"`
2. **Create PR**: Submit pull request for code review
3. **Prepare for Day 2**: Review HTML5 and CSS concepts
4. **Update progress**: Document your learning in the daily summary

## 📚 Additional Resources

- [Git Documentation](https://git-scm.com/doc)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Agile Manifesto](https://agilemanifesto.org/)
- [VS Code Git Integration](https://code.visualstudio.com/docs/editor/versioncontrol)

---

**Ready for Day 2? Check out [Day 2: HTML5 & Advanced CSS](../day2/README.md)!** 🚀
